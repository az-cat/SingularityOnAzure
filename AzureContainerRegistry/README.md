# Singularity and Azure Container Registry on Azure HPC VMs

## Introduction

Singularity is a container model better suited to HPC.  However, since Docker is so mature with it’s ecosystem of developers and registries, we would like to utilize these.
This document describes a possible workflow of creating Docker containers, uploading them into Azure Container Registry, and using them through Singularity to run into a Scheduler environment with a RDMA interconnect. 
This has been tested on H16r VMs running the CentOS 7.1 HPC image in the Azure marketplace.

## Install Singularity with OAUTH2 token patch

There is no singularity package available in the CentOS repository and so it must be built from source. Singularity does not support OAUTH2 tokens as implemented with ACR, and thus requires an additional patch. This patch is provided by https://github.com/hakonenger. This patch has now been merged in the development-2.x branch of Singularity.

    sudo yum -y install git nss curl libcurl libtool automake libarchive-devel squashfs-tools
    git clone https://github.com/singularityware/singularity.git
    cd singularity

for using the development branch (patch has been merged)
    git checkout development-2.x

for using the plain patch
    wget https://github.com/singularityware/singularity/files/2076307/oauth2-token-patch.txt
    patch -i oauth2-token-patch.txt libexec/python/docker/api.py

    ./autogen.sh
    ./configure --prefix=/shared/bin/singularity
    make
    sudo make install

Full details can be found on the Singularity site https://www.sylabs.io/docs/.

## Install Docker

Docker is not yet provided directly through CentOS or EPEL; therefor the Docker repositories have to be added and enabled first:

    sudo yum install -y yum-utils device-mapper-persistent-data lvm2
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo yum install -y docker-ce
    sudo systemctl start docker

## Create an Azure Container Registry

To create the registry and login for Docker, the az-cli is required. Use this page to install: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-yum?view=azure-cli-latest.
Creating a container registry is a recent feature that can be done from the az command line:

    az login
    az acr create --resource-group <my_RG> --name <my_acr_name> \
        --sku Basic --admin-enabled true

Since all Docker commands have to be run as sudo, also this sudo environment needs to login and the az commands need te be run like that:

    sudo az login
    sudo az acr login --name <my_acr_name>
    sudo az acr show --name <my_acr_name> --query loginServer --output table

## Build a Docker HPC container from a definition file

The container requires Intel MPI to be installed.  The installation is taken from the host system:

    mkdir rdma
    cd /opt
    tar zcvf ~/rdma/intel.tgz intel
    cd ~/rdma

This is the Dockerfile definition file used:

    FROM centos:7
    WORKDIR /app
    ADD intel.tgz /opt/
    RUN yum install -y tar gzip libmlx4 librdmacm libibverbs dapl rdma net-tools numactl
    ENV LD_LIBRARY_PATH /opt/intel/compilers_and_libraries_2016.3.223/linux/mpi/intel64/lib/
    ENTRYPOINT ["exec", "\"$@\""]

All Docker commands require root user to use, so building the container goes like this:

    sudo docker build -t rdma .

## Uploading Docker image to Azure Container Registry

Now the Docker image is build, it can be tagged with the registry name and pushed into the Azure Container Registry. The registry name can be obtained with:

    sudo az acr show --name <my_acr_name> --query loginServer --output table

    sudo docker tag rdma <my_acr_name>.azurecr.io/rdma:latest
    sudo docker push <my_acr_name>.azurecr.io/rdma:latest

## Building the Singularity image

Now we can access the uploaded Docker image from Singularity and build a Singularity image. For this we need to obtain the username and password form the Azure portal. They can be found under the Access Keys button on the ACR information page. The keys can be exported like this:

    export SINGULARITY_DOCKER_USERNAME=<my_acr_name>
    export SINGULARITY_DOCKER_PASSWORD=<my_acr_password>
    export PATH=/shared/bin/singularity/bin:$PATH

With these keys the Singularity container can be built:

    singularity pull docker://<my_acr_name>.azurecr.io/rdma:latest

This creates the “rdma-latest.simg” Singularity image.

To improve security, a dedicates ServicePrincipal can be created with only read rights to the ContainerRegistry. In that case, use the application_id for the USERNAME and the application_secret for the PASSWORD. 

## Singularity and Cyclecloud

To use Singularity on a Cyclecloud SGE cluster, a jobscript is needed. The following script can be submitted with: qsub -pe mpi 32 rdma.job 
     
    #!/bin/bash

    # prepare machine file for mpi
    cat "$TMPDIR/machines"
    cat "$PE_HOSTFILE"
    uniq $TMPDIR/machines > $TMPDIR/u_machines


    source /opt/intel/impi/5.1.3.223/bin64/mpivars.sh
    export PATH=/shared/bin/singularity/bin:$PATH
    export SINGULARITY_BINDPATH=/etc/rdma/dat.conf
     
    mpirun -machinefile "$TMPDIR/u_machines" -np 2 -ppn 1 singularity exec ./rdma-latest.simg /opt/intel/impi/5.1.3.223/bin64/IMB-MPI1 PingPong


Comments on the MPI options:

- I_MPI_FALLBACK=0 makes sure that the specified fabric is used and will not fallback to anything else (i.e. TCP)
- I_MPI_DEBUG=6 provides verbose information from MPI – useful when diagnosing issues.
- I_MPI_DAPL_PROVIDER, I_MPI_FABRICS and I_MPI_DYNAMIC_CONNECTION settings are both required when using Infiniband on Azure 
- We should typically use “I_MPI_FABRICS=shm:dapl” but this will make sure dapl is used for our test.
- I_MPI_DAPL_TRANSLATION_CACHE=0 works around an Intel MPI bug (random but infrequent crashes with certain applications) 


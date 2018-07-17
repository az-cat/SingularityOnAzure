# Singularity on Azure HPC VMs

## Introduction

Singularity is a container model better suited to HPC.  This has been tested on an H16r VM running the CentOS 7.1 HPC image in the Azure marketplace.

## Install Singularity

There is no singularity package available in the CentOS repository and so it must be built from source.

    sudo yum -y install git nss curl libcurl libtool automake libarchive-devel squashfs-tools
    git clone https://github.com/singularityware/singularity.git
    cd singularity
    ./autogen.sh
    ./configure --prefix=/shared/bin/singularity
    make
    sudo make install

Full details can be found on the Singularity site https://www.sylabs.io/docs/.

## Build an HPC container from a definition file

The container requires Intel MPI to be installed.  The installation is taken from the host system:

    cd /opt
    tar zcvf ~/intel.tgz intel

This is the centos definition file used:

    Bootstrap: yum
    OSVersion: 7
    MirrorURL: http://mirror.centos.org/centos-%{OSVERSION}/%{OSVERSION}/os/$basearch/
    Include: yum

    %runscript
    source /opt/intel/impi/5.1.3.223/bin64/mpivars.sh
    exec "$@"

    %files
    intel.tgz /opt/intel.tgz

    %environment
    export I_MPI_FALLBACK=0
    export I_MPI_FABRICS=shm:dapl
    export I_MPI_DAPL_PROVIDER=ofa-v2-ib0
    export I_MPI_DYNAMIC_CONNECTION=0
    export I_MPI_DAPL_TRANSLATION_CACHE=0

    %post
    yum install -y tar gzip libmlx4 librdmacm libibverbs dapl rdma net-tools numactl
    cd /opt
    tar zxf intel.tgz
    rm -rf intel.tgz
    mkdir -p /etc/rdma
    touch /etc/rdma/dat.conf

The HPC image has an updated and node-specific version of “dat.conf” to the version in the “rdma” package. To make sure the correct version is used, the dat.conf will be used from the host node using a bindpath.

The image required root user to build:

    export PATH=/shared/bin/singularity/bin:$PATH
    sudo singularity build centos7.simg centos.def

This creates the “centos7.simg” Singularity image.

## Testing MPI on the image

Singularity containers can be run directly with mpirun.  The container that has been created is setup to execute the application specified in the first parameter.  We can test that the Infiniband is working with the following command: 

    export SINGULARITY_BINDPATH=/etc/rdma/dat.conf    
    mpirun -np 2 \
        ./centos7.simg /opt/intel/impi/5.1.3.223/bin64/IMB-MPI1 PingPong


## Singularity and Cyclecloud

To use Singularity on a Cyclecloud SGE cluster, a jobscript is needed. The following script can be submitted with: `qsub -pe mpi 32 rdma.job` 
     
     #!/bin/bash

     # prepare machine file for mpi
     cat "$TMPDIR/machines"
     cat "$PE_HOSTFILE"
     uniq $TMPDIR/machines > $TMPDIR/u_machines
     
     source /opt/intel/impi/5.1.3.223/bin64/mpivars.sh
     export PATH=/shared/bin/singularity/bin:$PATH
     export SINGULARITY_BINDPATH=/etc/rdma/dat.conf
     
     mpirun -f "$TMPDIR/u_machines" -np 2 -ppn 1 \
         ./centos7.simg /opt/intel/impi/5.1.3.223/bin64/IMB-MPI1 PingPong


Comments on the MPI options:

- I_MPI_FALLBACK=0 makes sure that the specified fabric is used and will not fallback to anything else (i.e. TCP)
- I_MPI_DEBUG=6 provides verbose information from MPI – useful when diagnosing issues.
- I_MPI_DAPL_PROVIDER, I_MPI_FABRICS and I_MPI_DYNAMIC_CONNECTION settings are both required when using Infiniband on Azure 
- We should typically use “I_MPI_FABRICS=shm:dapl” but this will make sure dapl is used for our test.
- I_MPI_DAPL_TRANSLATION_CACHE=0 works around an Intel MPI bug (random but infrequent crashes with certain applications) 


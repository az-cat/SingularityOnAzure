# Singularity on Azure HPC VMs

## Introduction

Singularity is a container model better suited to HPC.  This has been tested on an H16r VM running the CentOS 7.1 HPC image in the Azure marketplace.

## Install Singularity

There is no singularity package available in the CentOS repository and so it must be built from source.

    git clone https://github.com/singularityware/singularity.git
    cd singularity
    ./autogen.sh
    ./configure --prefix=/usr/local
    make
    sudo make install

Full details can be found on the Singularity site https://singularity.lbl.gov/install-linux.

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
    /etc/rdma/dat.conf /opt/dat.conf

    %post
    yum install -y tar gzip libmlx4 librdmacm libibverbs dapl rdma net-tools numactl
    cd /opt
    tar zxf intel.tgz
    cp /opt/dat.conf /etc/rdma/dat.conf
    rm /opt/dat.conf

The rdma config file is also used from the host VM in this definition file.  The HPC image has an updated version of “dat.conf” to the version in the “rdma” package.

The image required root user to build:

    sudo singularity build centos7.simg centos.def

This creates the “centos7.simg” Singularity image.

## Testing MPI on the image

Singularity containers can be run directly with mpirun.  The container that has been created is setup to execute the application specified in the first parameter.  We can test that the Infiniband is working with the following command: 
    
    mpirun -np 2 \
        -genv I_MPI_DEBUG 6 -genv I_MPI_FALLBACK 0 \
        -genv I_MPI_FABRICS dapl -genv I_MPI_DAPL_PROVIDER ofa-v2-ib0 \
        -genv I_MPI_DYNAMIC_CONNECTION 0 -genv I_MPI_DAPL_TRANSLATION_CACHE 0 \
        ./centos7.simg /opt/intel/impi/5.1.3.223/bin64/IMB-MPI1 PingPong

Comments on the MPI options:

- I_MPI_FALLBACK=0 makes sure that the specified fabric is used and will not fallback to anything else (i.e. TCP)
- I_MPI_DEBUG=6 provides verbose information from MPI – useful when diagnosing issues.
- I_MPI_DAPL_PROVIDER, I_MPI_FABRICS and I_MPI_DYNAMIC_CONNECTION settings are both required when using Infiniband on Azure 
- We should typically use “I_MPI_FABRICS=shm:dapl” but this will make sure dapl is used for our test.
- I_MPI_DAPL_TRANSLATION_CACHE=0 works around an Intel MPI bug (random but infrequent crashes with certain applications) 


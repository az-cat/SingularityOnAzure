FROM centos:7
WORKDIR /app
ADD intel.tgz /opt/
RUN yum install -y tar gzip libmlx4 librdmacm libibverbs dapl rdma net-tools numactl 
ENV LD_LIBRARY_PATH /opt/intel/compilers_and_libraries_2016.3.223/linux/mpi/intel64/lib/
ENTRYPOINT ["exec", "\"$@\""]

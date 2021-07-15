# RDMA
Remote Direct Memory Access (RDMA)就是直接访问另一台机器的内存，而不用CPU参与.

RDMA这种形式有三大优点：
1. 零复制：无需网络软件层的参与，应用就能传送数据. 数据直接在内存之间传输，而不用在网络层一级级复制.
1. 跳过内核：用户态程序就能直接传输数据，不需要内核的参与
1. 跳过CPU：应用直接访问远端机器的内存，不需要CPU的参与，同时也不会占用远端机器的Cache

RoCE（RDMA over Converged Ethernet）是一种允许通过以太网使用远程直接内存访问（RDMA）的网络协议.

OpenFabrics Enterprise Distribution (OFED)是一个开源的软件，专门用来做RDMA。包括内核驱动，RDMA的各种操作，跳过内核的机制，还有内核态和用户态的编程接口API。支持MPI，socket数据传输（RDS，SDP），NAS和SAN存储，文件系统和数据库等.

**目前大部分RDMA都是通过以太网和Infiniband传输的**.

用户态NVMe SSD驱动优势挺明显的，就是性能好，延迟短，是因为省掉了Linux IO Stack的一层层消耗。传统用户态程序要读写，要基于文件，然后到内核态，通过文件系统，块设备驱动，再发到物理设备. NVMeDirect或者SPDK是用户态程序直接发IO到物理设备，延迟大为缩短.
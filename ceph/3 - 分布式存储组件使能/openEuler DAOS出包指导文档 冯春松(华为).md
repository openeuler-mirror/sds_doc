## DAOS介绍

### 简要介绍

分布式异步对象存储 (DAOS) 是一个开源的对象存储系统，专为大规模分布式非易失性内存 (NVM, Non-Volatile Memory) 设计，利用了SCM(Storage-Class Memory) 和 NVMe(Non-Volatile Memory express) 固态盘等的下一代 NVM 技术。

DAOS 是一种横向扩展的对象存储，可以为高性能计算应用提供高带宽、低延迟和高 IOPS 的存储容器，并支持结合仿真、数据分析和机器学习的下一代以数据为中心的工作流程。

与主要针对旋转介质设计的传统存储堆栈不同，DAOS 针对全新 NVM 技术进行了重新构建，可在用户空间中端对端地运行，并能完全绕开操作系统，是一套轻量级的系统。

DAOS 提供了一种为访问高细粒度数据提供原生支持的 I/O 模型，而不是传统的基于高延迟和块存储设计的 I/O 模型，从而释放下一代存储技术的性能。

与传统的缓冲区不同，DAOS 是一个独立的高性能容错存储层，它不依赖其它层来管理元数据并提供数据恢复能力。DAOS 服务器将其元数据保存在持久内存中，而将批量数据直接保存在 NVMe 固态盘中。

### DAOS系统架构

![343197a9c41a2446c89cd5394fdcbf5e](C:\Users\Administrator\Pictures\343197a9c41a2446c89cd5394fdcbf5e.png)

​                                                                                       DAOS架构与传统存储系统的对比

与主要针对旋转介质设计的传统存储堆栈不同，DAOS 针对全新 NVM 技术进行了重新构建。此外，DAOS 还是一套轻量级的系统，可在用户空间中端对端地运行，并能完全绕开操作系统。DAOS 没有延续针对高延迟、块存储的 I/O 模型，而是选择了为访问高细粒度数据提供原生支持的 I/O 模型，以此释放下一代存储技术的性能。

 现有的分布式存储系统使用高延迟的点对点通信，而 DAOS 使用能够绕过操作系统的低延迟、高消息速率用户空间通信。当下，大多数存储系统都是针对块 I/O 设计的，所有 I/O 操作都通过块接口在 Linux* 内核中进行。为了优化对于块设备的访问，业界已经付出了例如合并、缓冲和聚合等方面的许多努力。但是，所有这些优化都无法适用于英特尔着力发展的下一代存储设备，因为使用这些优化可能会产生不必要的开销。而 DAOS 专为优化对英特尔傲腾持久内存和 NVM Express (NVMe) 固态盘 (SSD) 的访问而设计，它规避了这些不必要的开销。

DAOS 服务器将其元数据保存在持久内存中，而将批量数据直接保存在 NVMe 固态盘中。此外，少量 I/O 操作在聚合之前就会被吸收到持久内存中，然后再迁移到大容量闪存。DAOS 使用持久内存开发套件 (PMDK) 提供对于持久内存的事务访问，并使用存储性能开发套件 (SPDK) 为 NVMe 设备提供用户空间 I/O1,2。这种架构的数据访问速度比现有存储系统快好几个数量级（从毫秒 [ms] 级加快到微秒 [μs] 级）。

## DAOS编译

### 环境准备

建议软件版本：

| 软件 | 版本           |
| ---- | -------------- |
| DAOS | commit 008flba |

操作系统要求：

| 项目      | 版本                         |
| --------- | ---------------------------- |
| openEuler | openEuler 22.03              |
| Kernel    | 5.10.0-60.18.0.50.oe.aarch64 |

说明：

本文档适用于DAOS主线版本，其他版本的移植步骤可参考本文档。移植前请确保环境纯净，不要使用其他DAOS版本的系统环境。

### DAOS编译

1. 安装scons编译工具：

   yum install scons -y

2. 解决编译依赖问题，执行DAOS自带的依赖安装脚本：

   cd daos/utils/scripts

   sh -x install-el8.sh

   屏蔽不能直接安装的依赖，直接把先能安装的依赖安装

   pip3 install patchelf

   yum install rdma-core-devel.aarch64

3. 解决缺少daos-raft-devel依赖问题，解决方法daos-raft-devel编译出包安装：

   yum install rpmdevtools

   rpmdev-setuptree

   修改~/.rpmmacros文件的rpmbuild路径

   rpmdev-setuptree

   git clone https://github.com/daos-stack/raft.git

   cd raft;cp raft.spec /home/rpmbuild/SPECS

   mv raft raft-0.9.1

   tar -jcvf raft-0.9.1.tar.gz raft-0.9.1

   mv raft-0.9.1.tar.gz /home/rpmbuild/SOURCES

   修改raft.spec的压缩文件名与压缩包名字一致后，直接编译出包：

   rpmbuild -bb /home/rpmbuild/SPECS/raft.spec

   在生成rpm包目录下直接安装：rpm -ivh raft*.rpm

   编译DAOS源码：

   scons -j32 --build-deps=yes

   说明：

   在该编译环境中gcc版本是gcc version 10.3.1(GCC)，在编译过程中未出现编译问题。

   如果遇到gcc版本问题，手动在site_scons/compiler_setup.py文件的DESIRED_FLAGS后面加:"-Wno-format-overflow"，重新编译没有问题。

   ## 相关依赖编译出包

   ### 编译出包准备

   1.  安装出包工具：

      yum install rpmdevtools -y

      rpmdev-setuptree

   2.  打包：

      在daos源码下执行：

      scons -c

      把utils/rpms目录下的daos.spec与bz-1955184_find-requires拷贝到rpmbuild/SPECS路径下

      把daos源码打包成.tar.gz格式：

      tar -jcvf daos-2.3.100.tar.gz daos-2.3.100

      cp daos-2.3.100.tar.gz /home/rpmbuild/SOURCES

   3.  修改daos.spec文件压缩包的文件名与压缩包对应

   4.  查看编译出包需要安装的依赖

      rpmbuild -bb /home/rpmbuild/SPECS/daos.spec

      说明：

      接下来是根据提示所缺少依赖进行编译出包，根据自己的环境选择必要的编译出包，如果已经安装可以忽略，不再需要自己手动编译出包，下面是一些编译出包流程可以作为参考。

      ### libfabric编译出包

      daos源码里面有libfabric源码，路径是daos-2.3.100/build/external/release/ofi。里面有自带的libfabric.spec文件，用它自带libfabric.spec文件，不然会有版本问题；

      修改libfabric.spec文件的压缩包名，安装依赖，configure配置：

      ![捕获](C:\Users\Administrator\Pictures\捕获.PNG) 

      yum install libnl3-devel

      yum install fdupes

      rpmbuild --nodebuginfo -bb /home/rpmbuild/SPECS/libfabric.spec

      ### argobots编译出包

      git clone https://github.com/daos-stack/argobots.git

      这里我们只用到里面的spec文件，源码我们用的是daos-2.3.100/build/external/release/argobots将源码改名后压缩打包：

      rpmbuild -bb /home/rpmbuild/SPECSargobots.spec

      ### UCX编译出包

      这里直接使用DAOS源码里面的ucx源码与ucx.spec文件进行编译出包

      修改ucx.spec文件中的压缩文件名与压缩包名对应，进行编译出包

      rpmbuild -bb /home/rpmbuild/SPECES/ucx.spec

      ### isa-l编译出包

      DAOS源码里面有isa-l源码，可以直接使用；没有isal.spec文件，这里直接github上下载：

      git clone https://github.com/daos-stack/isa-l.git

      修改isa-l.spec文件中压缩文件名与该压缩包名字对应，进行编译出包

      rpmbuild -bb /home/rpmbuild/SPECS/isa-l.spec

      ### isa-l_crypto编译出包

      wget https://package.daos.io/v2.0/CentOS8/source/isa-l_crypto-2.23.0-1.el8.src.rpm

      rpm -ivh isa-l_crypto-2.23.0-1.el8.src.rpm

      将daos源码里的isa-l_crypto源码打包，并拷贝到编译路径

      修改isal_crypto.spec文件中isal-crypto版本以及压缩包名字，与现有压缩包对应，编译出包：

      rpmbuild -bb /home/rpmbuild/SPECS/isa-l_crypto.spec

      ### dpdk编译出包

      DAOS源码里面有dpdk源码，可以直接使用；没有dpdk.spec文件，这里是直接github上下载：

      git clone https://github.com/daos-stack/dpdk.git

      安装依赖：

      yum install libpcap-devel

      yum install python3-sphinx.noarch

      yum install doxygen

      在dpdk.spec文件屏蔽不需要的patch,DAOS源码里面的dpdk不需要patch

      rpmbuild -bb /home/rpmbuild/SPECS/dpdk.spec

      说明：

      如果制作rpm包的过程中，出现报错Installed (but unpackaged) file(s)found

      解决方式：

      1.  修改/usr/rpm/macros文件中下面的行：%__check_files /usr/lib/rpm/check-files %{buildroot} #注释掉
      2. 修改/usr/lib/rpm/macros文件中以下的行：%__unpackaged_files_terminate_build 1 #把1改为0只警告

      ### spdk编译出包

      DAOS源码里面有自带的spdk源码与spdk.spec文件，直接用它自带的进行编译出包：

      cp -r daos-2.3.100/build/external/release/spdk ./

      cp spdk/rpmbuild/spdk.spec /home/rpmbuild/SPECS

      mv spdk spdk-22.01.1

      tar -jcvf spdk-22.01.1.tar.gz spdk-22.01.1

      mv spdk-22.01.1.tar.gz /home/rpmbuild/SOURCES

      修改spec文件中版本，以及configure配置、spdk安装路径：

      ![捕获1](C:\Users\Administrator\Pictures\捕获1.PNG) 

      修改spdk.spec文件，打开共享库选项：

      ![捕获4](C:\Users\Administrator\Pictures\捕获4.PNG) 

      rpmbuild -bb /home/rpmbuild/SPECS/spdk.spec

      ### pmdk编译出包

      DAOS源码里面有pmdk源码，可以直接使用，没有pmdk.spec文件，这里直接github上下载：

      git clone https://github.com/daos-stack/pmdk.git

      修改pmdk.spec文件中压缩包名、补丁

      屏蔽pmdk.spec文件的如下内容，或者在后面补充aarch64

      ![捕获5](C:\Users\Administrator\Pictures\捕获5.PNG) 

      yum install daxctl1-devel

      yum install ndctl-devel

      yum install pandoc

      如果pandoc无法直接安装，可以下载rpm包直接安装：

      wget https://rpmfind.net/linux/fedora/linux/development/rawhide/Everything/aarch64/os/Packages/p/pandoc-2.14.0.3-18.fc37.aarch64.rpm

      wget  https://rpmfind.net/linux/fedora/linux/development/rawhide/Everything/aarch64/os/Packages/p/pandoc-common-2.14.0.3-18.fc37.aarch64.rpm

      修改pmdk.spec文件中出包的版本号，参考daos.spec中daos-server依赖：

      ![捕获6](C:\Users\Administrator\Pictures\捕获6.PNG) 

      直接编译出包可能会遇到check失败问题，可以先屏蔽check:

      rpmbuild --nocheck -bb /home/rpmbuild/SPECS/pmdk.spec

      ### mercury编译出包

      DAOS源码里面有mercury源码，可以直接使用；没有mercury.spec文件，这里是直接github上下载：

      git clone https://github.com/daos-stack/mercury.gt

      这里mercury源码直接用daos-2.3.100/build/external/release/ofi,mercury.spec文件用github上下载的

      安装依赖前面已经出包的libfabric与ucx;安装完后直接编译出包：

      rpmbuild -bb /home/rpmbuild/SPECS/mercury.spec

      ## DAOS编译出包

      ### 安装DAOS编译出包依赖

      1. 将以上编译出包的相关依赖以本地yum源方式安装

      2. 查看需要安装的相关依赖：

         rpmbuild -bb /home/rpmbuild/SPECS/daos.spec

         其中daos-raft-devel是已经编译出包安装的raft-devel;libjson-c-devel是已经安装的json-c-devel;openmpi3-devel是已经安装的openmpi-devel;isal_crypto是已经安装的libisa-l_crypto;libabt-devel与libpsm2-devel并不会影响后续的daos出包，与系统相关，都可以先屏蔽。

      3. 安装相关依赖：

         yum install argobots argobots-devel

         yum install spdk-devel libpmem2-devel libpmempool-devel libpmem2 libpmempool

      4. 屏蔽daos.spec文件中daos-server中，ipmctl与spdk-tools依赖，以及屏蔽libdpar_mpi.so

         ![捕获7](C:\Users\Administrator\Pictures\捕获7.PNG) 

         ![捕获8](C:\Users\Administrator\Pictures\捕获8.PNG) 

      5. 编译出包：

         rpmbuild -bb /home/rpmbuild/SPECS/daos.spec

# 鲲鹏硬件加速器Ceph使能指南

## 鲲鹏加速器介绍

鲲鹏加速引擎是基于鲲鹏920处理器提供的硬件加速解决方案，包含了数据加解密、哈希摘要以及数据压缩的加速能力，分别用于加速SSL/TLS，数据加密以及数据压缩场景，可以有效降低CPU开销，提升处理带宽和效率。加速引擎对应用层屏蔽了其内部硬件实现细节，用户通过openssl，zlib标准接口即可快速实现业务迁移。

- isa-l

  isa-l全称Intelligent Storage Acceleration Library，是Intel开源的存储加速库，提供RAID，纠删码，冗余循环检查，密码散列和压缩的指令加速优化。在Ceph的应用中，isa-l主要通过指令优化实现zlib压缩过程的加速，但对解压过程暂无加速效果。

- uadk/uadk_engine

  uadk即User Space Accelerator Development kit，是一套通用用户态加速器开发包，为用户提供一套针对加解密、压缩等算法的统一编程接口。uadk2.0基于SMMU和SVA共享虚拟内存地址技术，用户可以安全高效地利用鲲鹏加速器的硬件能力。

  uadk_engine是基于uadk2.0的openssl加速引擎，提供高性能哈希、对称加解密、非对称加解密算法支持，兼容>=openssl1.1.1a，<openssl3.0的版本，支持同步&异步模式，目前支持以下算法：

  - 摘要算法：SM3/MD5/SHA1/SHA224/SHA256/SHA384/SHA512，支持异步模式

  - 对称加密算法SM4：支持CBC/ECB模式，支持异步模式

  - 对称加密算法AES：支持ECB/CTR/XTS/CBC/CFB/OFB模式，支持128/192/256秘钥长度，支持AEAD类GCM/CCM算法，支持异步模式

  - 非对称加密算法RSA：支持1024/2048/3072/4096 keysize，支持异步模式

  - 秘钥协商算法DH：支持768/1024/1536/2048/3072/4096 keysize，支持异步模式

- KAE与KAEzip

  KAE加解密是鲲鹏加速引擎的加解密模块，使用鲲鹏硬加速器引擎实现RSA/SM3/SM4/DH/MD5/AES算法加速，结合无损用户态驱动框架，提供高性能哈希、对称加解密、非对称加解密算法支持，兼容>=openssl1.1.1a，<openssl3.0的版本，支持同步&异步模式。KAE是基于uadk1.0（nosva）版本的openssl引擎，目前支持以下算法：

  - 摘要算法：
  - 对称加密算法SM4
  - 对称加密算法AES
  - 非对称加密RSA
  - 秘钥协商算法DH

  KAEzip是鲲鹏加速引擎的压缩模块，使用鲲鹏硬加速模块实现deflate算法，结合无损用户态驱动框架，提供高性能gzip/zlib格式压缩接口。KAEzip是基于uadk1.0（nosva）版本的zlib加速引擎。

  - 支持gzip/zlib数据格式，符合RFC1950/RFC1952标准规范
  - 支持deflate算法
  - 支持同步模式
  - 单鲲鹏920处理器最大压缩带宽7GB/s，最大解压带宽8GB/s
  - 支持压缩比≈2，与zlib1.2.11接口兼容

- uadk与KAE的关系

  ![输入图片说明](https://foruda.gitee.com/images/1669686896697571232/ac9e1e8b_1665388.png "uadk and kae.png")

## 环境说明

- 系统环境

  - 服务器

    TS200-2280 Kunpeng920@2.6G

  - openEuler22.03-LTS

    镜像地址：https://repo.huaweicloud.com/openeuler/openEuler-22.03-LTS/ISO/aarch64/openEuler-22.03-LTS-aarch64-dvd.iso

    kernel：5.10

  - license申请

    安装使用鲲鹏加速器引擎之前，需要先申请安装相应的License，完成硬件加速器激活。Taishan K系列服务器硬件加速引擎默认激活，无需申请License。

    申请指南：https://www.hikunpeng.com/document/detail/zh/kunpengaccel/encryp-decryp/faq-kae/kunpengaccel_10_0003.html

- 软件环境

  - 版本信息

    | 软件        | 版本    | 来源                                                         |
    | ----------- | ------- | ------------------------------------------------------------ |
    | Ceph        | v16.2.7 | https://gitee.com/src-openeuler/ceph/repository/archive/openEuler-22.03-LTS-20220331.zip |
    | openssl     | 1.1.1m  | openEuler22.03-LTS                                           |
    | uadk        | v2.3.36 | https://github.com/Linaro/uadk/archive/refs/tags/v2.3.36.zip |
    | uadk_engine | v1.0.1  | https://github.com/Linaro/uadk_engine/releases/tag/v1.0.1    |
    | KAEdriver   | v1.3.11 | https://github.com/kunpengcompute/KAEdriver/archive/refs/tags/v1.3.11.tar.gz |
    | KAEzip      | v1.3.11 | https://github.com/kunpengcompute/KAEzip/archive/refs/tags/v1.3.11.tar.gz |
    | KAE         | v1.3.11 | https://github.com/kunpengcompute/KAE/releases/tag/v1.3.11   |

  - 驱动更新

    uacce_mode: 工作模式。0表示默认的不注册用户态；1表示注册用户态并使能SVA；2表示注册用户态并使能NOSVA。

    openEuler22.03-LTS配套的5.10内核版本集成了鲲鹏硬件加速器驱动，主要由uacce.ko，qm.ko，hisi_zip.ko，hisi_sec2.ko，hisi_hrpe.ko组成。目前该版本仅支持0和1模式，不支持2模式，可以查看模块信息确认，

    ```shell
    modinfo hisi_zip
    ```

    若需模式2支持，需安装驱动

    ```shell
    yum install hisi_zip hisi_sec2 -y
    ```

    相应的ko安装在/usr/lib/modules/4.19.36-vhulk1907.1.0.h1103.eulerosv2r8.aarch64/extra/路径下。备份系统默认版本，将新版驱动复制到5.10内核模块路径下

    ```shell
    cd /usr/lib/modules/5.10.0-60.18.0.50.oe2203.aarch64/kernel/drivers/crypto/hisilicon
    mv zip/hisi_zip.ko zip/hisi_zip.ko.bk
    mv sec2/hisi_sec2.ko sec2/hisi_sec2.ko.bk
    cp /usr/lib/modules/4.19.36-vhulk1907.1.0.h1103.eulerosv2r8.aarch64/extra/hisi_zip.ko zip/
    cp /usr/lib/modules/4.19.36-vhulk1907.1.0.h1103.eulerosv2r8.aarch64/extra/hisi_sec2.ko sec2/
    ```

    再次查看新的模块信息，已支持模式2

    ```shell
    modinfo hisi_zip
    ```

    重新加载驱动

    ```shell
    rmmod hisi_zip hisi_sec2 hisi_qm uacce
    insmod /usr/lib/modules/4.19.36-vhulk1907.1.0.h1103.eulerosv2r8.aarch64/extra/uacce.ko
    insmod /usr/lib/modules/4.19.36-vhulk1907.1.0.h1103.eulerosv2r8.aarch64/extra/hisi_qm.ko
    ```

    启用uadk及uadk_engine时按uacce_mode=1加载

    ```shell
    modprobe hisi_zip uacce_mode=1
    modprobe hisi_sec2 uacce_mode=1
    ```

    启用KAE及KAEzip时按uacce_mode=2加载

    ```shell
    modprobe hisi_zip uacce_mode=2
    modprobe hisi_sec2 uacce_mode=2
    ```

  - uadk安装

    - 须知

      启用uadk2.0须确保BIOS的SMMU选项配置为enable。

      设置方法：进入BIOS后，在MISC Config->Support Smmu选择enable，重启生效。启动后，查询SMMU接口返回值为129，证明使能成功

      ```shell
      cat /sys/class/uacce/hisi*/flags
      ```

      ------

    下载并安装uadk

    ```shell
    git clone https://github.com/Linaro/uadk.git -b v2.3.36
    cd uadk
    ./cleanup.sh
    ./autogen.sh
    ./conf.sh
    make -j
    make install
    ```

  - uadk_engine安装

    下载并安装uadk_engine

    ```shell
    git clone https://github.com/Linaro/uadk_engine.git -b v1.0.1
    cd uadk_engine-1.0.1
    autoreconf -i
    ./configure --libdir=/usr/local/lib/engines-1.1/ --enable-kae
    make -j
    make install
    ```

    ./configure生成编译文件时指定--enable-kae，可同时支持v1.0和v2.0两种模式

    将uadk_engine的安装目录加入ld.so.conf

    ```shell
    vim /etc/ld.so.conf
    /usr/local/lib/
    /usr/local/lib/engines-1.1/
    ```

    生效

    ```shell
    ldconfig
    ```

    查看uadk_engine支持的算法列表，若成功返回，则说明uadk_engine可以被openssl正常加载

    ```shell
    openssl engine uadk_engine -c -t
    ```

  - KAE安装

    - 须知

      启用KAE须确保BIOS的SMMU选项配置为disable。

      设置方法：进入BIOS后，在MISC Config->Support Smmu选择enable，重启生效。启动后，查询SMMU接口返回值不为129，证明SMMU关闭

      ```shell
      cat /sys/class/uacce/hisi*/flags
      ```

    下载并安装KAE

    - 方法一

      直接使用uadk_engine的KAE模式（v1.0），engine id为“uadk_engine”

    - 方法二

      下载KAE编译安装，engine id为“kae”

      ```shell
      git clone https://github.com/kunpengcompute/KAE.git -b v1.3.11
      cd KAE
      chmod +x configure
      ./configure
      make clean
      make -j
      make install
      ```

  - KAEzip安装

    下载KAEzip

    ```shell
    git clone https://github.com/kunpengcompute/KAEzip.git -b v1.3.11
    ```

    在KAEzip-1.3.11/open_source目录下下载并解压与主机zlib版本相同的zlib源码包

    ```shell
    cd KAEzip-1.3.11/open_source
    wget https://www.zlib.net/fossils/zlib-1.2.11.tar.gz --no-check-certificate
    tar -zvxf zlib-1.2.11.tar.gz
    ```

    回退至KAEzip-1.3.11源码目录，编译安装KAEzip

    ```
    sh setup.sh install
    ```

    生成的加速库安装路径为/usr/local/kaezip/lib/libkaezip.so.1.3.11，新编译的zlib算法库路径的安装为/usr/local/kaezip/lib/libz.so.1.2.11

    - 须知

      编译KAEzip时如遇到uacce.h头文件无法找到的报错，解决方法：

      ```shell
      vim /usr/include/warpdriver/wd.h
      #include "include/uacce.h"
      ```

      

## Ceph编译

当前Ceph基于openEuler社区Ceph-16.2.7版本，在此基础上新增如下特性（patch），详情可参与openEuler-SDS sig咨询：

支持aarch64 isa-l压缩优化

支持zlib-uadk加速器加速插件

优化部分热点函数开销过高的问题

- rpm包构建

  安装rpm-build工具

  ```shell
  yum install rpm-build rpmdevtools -y
  ```

  生成rpmbuild目录，默认在/root目录下

  ```
  rpmdev-setuptree
  ```

  下载Ceph源码

  ```shell
  git clone https://gitee.com/src-openeuler/ceph.git -b openEuler-22.03-LTS-20220331
  ```

  将源码patch文件及ceph-16.2.7.tar.gz拷贝至rpmbuild下的SOURCES目录下

  ```shell
  cp ceph/ceph-16.2.7.tar.gz ceph/*.patch /root/rpmbuild/SOURCES
  ```

  将新增特性对应的patch文件也拷贝至rpmbuild下的SOURCES目录下

  将更新的ceph.spec文件拷贝至rpmbuild下的SPEC目录下

  构建ceph rpm包

  ```shell
  cd /root/rpmbuild/SPEC
  rpmbuild -ba ceph.spec
  ```

  最终生成的rpm包位于/root/rpmbuild/RPMS目录下

- 本地源配置

  安装createrepo工具

  ```shell
  yum install createrepo -y
  ```

  rpm源文件准备

  ```shell
  mkdir -p /home/cephrepo
  cp /root/rpmbuild/RPMS /home/cephrepo
  createrepo -p /home/cephrepo
  ```

  创建ceph.repo源文件

  ```shell
  vim /etc/yum.repos.d/ceph.repo
  [ceph]
  name = ceph
  baseurl = file:///home/cephrepo
  enable = 1
  gpgcheck = 0
  priority = 1
  ```

  repo源生效

  ```shell
  yum clean all
  yum makecache
  yum list |grep ceph
  ```

  

## Ceph安装

Ceph集群的详尽部署流程可以参考鲲鹏社区操作指南：

《Ceph块存储部署指南》

https://www.hikunpeng.com/document/detail/zh/kunpengsdss/ecosystemEnable/Ceph/kunpengcephblock_04_0001.html

《Ceph对象存储部署指南》

https://www.hikunpeng.com/document/detail/zh/kunpengsdss/ecosystemEnable/Ceph/kunpengcephobject_04_0001.html

- Ceph环境准备

  - 关闭防火墙

    ```shell
    systemctl stop firewalld
    systemctl disable firewalld
    ```

  - 配置主机名

    ```shell
    hostnamectl --static set-hostname node1
    vim /etc/hosts
    <node1 ip> node1
    ```

  - chrony时间同步

    安装chrony，包括两个程序：chronyd和chronyc

    chronyd：后台运行的守护进程，负责时间同步

    chronyc：用户命令行工具，负责用户多样化配置与状态监控

    ```shell
    yum install chrony -y
    ```

    修改配置文件，将node1作为时间同步源，node1修改：

    ```shell
    vim /etc/chrony.conf
    server <node1 ip> iburst
    allow <gateway>
    local stratum 10
    ```

    客户端节点修改

    ```shell
    vim /etc/chrony.conf
    server <node1 ip> iburst
    ```

    启动服务

    ```shell
    systemctl restart chronyd
    chronyc sources
    ```

  - 配置免密登录

    配置服务端节点node1对所有节点免密

    ```shell
    ssh-keygen -t rsa
    ssh-copy-id client1
    ```

    - 说明

      ssh-keygen -t rsa命令后，使用默认值，按enter确认即可

  - 关闭selinux

    所有节点均执行。

    临时关闭，重启后失效

    ```shell
    setenforce 0
    ```

    永久关闭，重启后生效

    ```shell
    vim /etc/selinux/config
    SELINUX=disable
    ```

- Ceph安装与部署

  - 安装Ceph

    ```shell
    yum install ceph ceph-deploy -y
    ```

  - 部署mon

    ```shell
    cd /etc/ceph
    ceph-deploy new node1
    ```

    修改/etc/ceph目录下的ceph.conf文件，增加如下内容：

    ```
    vim /etc/ceph/ceph.conf
    [mon]
    mon_allow_pool_delete = true
    ```

    初始化mon并分发keyring

    ```shell
    ceph-deploy mon create-initial
    ceph-deploy --overwrite-conf admin node2 node3
    ```

    查看集群状态

    ```shell
    ceph -s
    ```

  - 部署mgr

    ```shell
    ceph-deploy mgr create node1
    ceph -s
    ```

  - 部署osd

    ```shell
    ceph-deploy disk zap node1 /dev/sdb
    ceph-deploy osd create node1 --data /dev/sdb
    ```

    

## Bluestore压缩使能

以ceph块存储场景说明Bluestore压缩使能。修改ceph.conf，配置启用Bluestore压缩，目前压缩算法支持zlib/snappy/lz4/zstd等。isa-l与鲲鹏硬件加速器支持zlib算法加速。如果需要启用加速器，或在加速器使能与否进行切换时，需将压缩算法配置为zlib，同时设置compressor_zlib_winsize（默认-15，只有deflate算法，数据格式不包含gzip/zlib协议头）为9-15之间的值。

```shell
vim ceph.conf
[osd]
bluestore_compression_algorithm = zlib
compressor_zlib_winsize = 15
```

重启OSD守护进程

```
systemctl restart ceph-osd.target
```

- 块存储部署

  创建pool，并使能存储池支持的应用类型

  ```shell
  ceph osd pool create rbdpool 256 256
  ceph osd pool application enable rbdpool rbd
  ```

  创建image，以80G的image1说明

  ```shell
  rbd create image1 --size 81920 --pool rbdpool --image-format 2 --image-feature layering
  rbd ls --pool rbdpool
  ```

- 启用isa-l

  isa-l压缩已经实现Ceph的直接支持，同时，isa-l aarch64的压缩优化也已经合入社区。

  修改ceph.conf使能

  ```shell
  vim /etc/ceph/ceph.conf
  [osd]
  compressor_zlib_isal = true
  ```

  重启osd生效

  ```shell
  systemctl restart ceph-osd.target
  ```

- 启用uadk

  前置条件：

  smmu = enable，uadk完成安装，压缩相关模块驱动已加载且uacce_mode指定为1

  修改ceph.conf使能

  ```shell
  vim /etc/ceph/ceph.conf
  [osd]
  uadk_compressor_enabled = true
  ```

  osd进程使用ceph用户没有加速器设备的访问权限，[Service]项下的ExecStart命令，修改为root用户启动

  ```shell
  vim /usr/lib/systemd/system/ceph-osd@1.service
  [Service]
  ExecStart=/usr/bin/ceph-osd -f --cluster ${CLUSTER} --id %i --setuser root --setgroup root
  ```

  service文件修改生效，并重启osd守护进程

  ```shell
  systemctl daemon-reload
  systemctl restart ceph-osd.target
  ```

- 启用KAEzip

  前置条件：

  smmu = disable，KAEzip已安装，压缩相关模块驱动已加载且uacce_mode指定为2

  修改系统默认zlib的软链接，将其软链至KAEzip编译生成的zlib库，重启osd进程

  ```shell
  rm /usr/lib64/libz.so
  rm /usr/lib64/libz.so.1
  ln -s /usr/local/kaezip/lib/libz.so.1.2.11 /usr/lib64/libz.so
  ln -s /usr/local/kaezip/lib/libz.so.1.2.11 /usr/lib64/libz.so.1
  ```

  osd进程使用ceph用户没有加速器设备的访问权限，[service]项下的ExecStart命令，修改为root用户启动

  ```shell
  vim /usr/lib/systemd/system/ceph-osd@1.service
  [service]
  ExecStart=/usr/bin/ceph-osd -f --cluster ${CLUSTER} --id %i --setuser root --setgroup root
  ```

  service文件修改生效，并重启osd守护进程

  ```shell
  systemctl daemon-reload
  systemctl restart ceph-osd.target
  ```

  

## 对象存储压缩使能

- 对象存储部署

  - 部署RGW实例。以在node1节点创建一个RGW实例，前端端口7481，实例名bucket1为例说明。

    修改ceph.conf

    ```shell
    vim /etc/ceph/ceph.conf
    [client.rgw.bucket1]
    rgw_frontends = beast port=7481
    ```

    同步配置文件

    ```shell
    ceph-deploy --overwrite-conf admin node2 node3
    ```

    安装RGW组件

    ```shell
    yum install ceph-radosgw -y
    ```

    创建RGW实例以及集群状态检查

    ```shell
    ceph-deploy rgw create node1:bucket1
    ceph -s
    ```

    取消代理配置，访问bucket1实例前端

    ```shell
    unset http_proxy
    unset https_proxy
    curl <endpoint ip>:7481
    ```

  - 创建压缩存储池

    创建data pool，index pool，和non-ec pool

    ```shell
    ceph osd pool create default.rgw.buckets.data-compress 1024 1024
    ceph osd pool create default.rgw.buckets,index-compress 256 256
    ceph osd pool create default.rgw.buckets.non-ec-compress 64 64
    ceph osd pool application enable default.rgw.buckets.data-compress rgw
    ceph osd pool application enable default.rgw.buckets.index-compress rgw
    ceph osd pool application enable default.rgw.buckets.non-ec-compress rgw
    ```

  - 使能RGW压缩

    新增用于压缩使能的放置策略

    ```shell
    radosgw-admin zonegroup placement add --rgw-zonegroup=default --placement-id=compress-placement
    ```

    配置compress-placement信息，压缩算法选用zlib

    ```shell
    radosgw-admin zone placement add --rgw-zonegroup=default --placement-id=compress-placement --data_pool=default.rgw.buckets.data-compress --index_pool=default.rgw.buckets,index-compress --data_extra_pool=default.rgw.buckets.non-ec-compress --compression=zlib
    ```

    新建用户

    ```shell
    radosgw-admin user create --uid="admin" --display-name="admin" --access_key=test --secret_key=test
    ```

    导出“admin”用户元数据，修改json文件中defaul_placement内容为“compress-placement”

    ```
    radosgw-admin metadata get user:admin > user.json
    vim user.json
    ```

    将修改后的用户元数据导入

    ```shell
    radosgw-admin metadata put user:admin < user.json
    ```

    重启RGW进程，使修改生效

    ```shell
    systemctl restart ceph-radosgw.target
    ```

- 启用isa-l

  isa-l压缩已经实现Ceph的直接支持，同时，isa-l aarch64的压缩优化也已经合入社区。

  修改ceph.conf使能

  ```shell
  vim /etc/ceph/ceph.conf
  [osd]
  compressor_zlib_isal = true
  ```

  重启osd生效

  ```shell
  systemctl restart ceph-osd.target
  ```

- 启用uadk

  前置条件：

  smmu = enable，uadk已安装，压缩相关模块驱动已加载且uacce_mode指定为1

  修改ceph.conf配置，在rgw实例配置下加入：

  ```shell
  vim /etc/ceph/ceph.conf
  [client.rgw.bucket1]
  uadk_compressor_enabled = true
  compressor_zlib_winsize = 15
  ```

  RGW进程使用ceph用户没有加速器设备的访问权限，[Service]项下的ExecStart命令，修改为root用户启动。同时PrivateDevices值修改为no。

  ```shell
  vim /usr/lib/systemd/system/ceph-radosgw@rgw.bucket1.service
  [Service]
  ExecStart=/usr/bin/radosgw -f --cluster ${CLUSTER} --name client.%i --setuser root --setgroup root
  PrivateDevices=no
  ```

  service文件修改生效，并重启osd守护进程

  ```shell
  systemctl daemon-reload
  systemctl restart ceph-radosgw.target
  ```

  - 说明

    如需使用uadk_engine加速对象存储场景的MD5，SHA256开销，参考对象存储加密使能章节uadk_engine的启动流程。

- 启用KAEzip

  前置条件：

  smmu = disable，KAEzip已安装，压缩相关模块驱动已加载且uacce_mode指定为2

  修改系统默认zlib的软链接，将其软链至KAEzip编译生成的zlib库，重启RGW进程

  ```shell
  rm /usr/lib64/libz.so
  rm /usr/lib64/libz.so.1
  ln -s /usr/local/kaezip/lib/libz.so.1.2.11 /usr/lib64/libz.so
  ln -s /usr/local/kaezip/lib/libz.so.1.2.11 /usr/lib64/libz.so.1
  ```

  RGW进程使用ceph用户没有加速器设备的访问权限，[Service]项下的ExecStart命令，修改为root用户启动

  ```shell
  vim /usr/lib/systemd/system/ceph-radosgw@rgw.bucket1.service
  [Service]
  ExecStart=/usr/bin/radosgw -f --cluster ${CLUSTER} --name client.%i --setuser root --setgroup root
  PrivateDevices=no
  ```

  service文件修改生效，并重启RGW守护进程

  ```shell
  systemctl daemon-reload
  systemctl restart ceph-radosgw.target
  ```

  - 说明

    如需使用KAE加速对象存储压缩场景的MD5开销，参考对象存储加密使能章节的KAE启用流程。

## 对象存储加密使能

- 对象存储部署

  - 部署RGW实例。以在node1节点创建一个RGW实例，前端端口7481，实例名bucket1为例说明。

    修改ceph.conf

    ```shell
    vim /etc/ceph/ceph.conf
    [client.rgw.bucket1]
    rgw_frontends = beast port=7481
    ```

    同步配置文件

    ```shell
    ceph-deploy --overwrite-conf admin node2 node3
    ```

    安装RGW组件

    ```shell
    yum install ceph-radosgw -y
    ```

    创建RGW实例以及集群状态检查

    ```shell
    ceph-deploy rgw create node1:bucket1
    ceph -s
    ```

    取消代理配置，访问bucket1实例前端

    ```shell
    unset http_proxy
    unset https_proxy
    curl <endpoint ip>:7481
    ```

  - 创建压缩存储池

    创建data pool，index pool

    ```shell
    ceph osd pool create default.rgw.buckets.data-compress 1024 1024
    ceph osd pool create default.rgw.buckets,index-compress 256 256
    ceph osd pool application enable default.rgw.buckets.data-compress rgw
    ceph osd pool application enable default.rgw.buckets.index-compress rgw
    ```

  - 创建RGW用户

    ```shell
    radosgw-admin user create --uid="admin" --display-name="admin" --access_key=test --secret_key=test
    ```

- 启用uadk_engine

  uadk_engine启用后，可用鲲鹏加速器加速卸载对象存储场景的MD5、SHA256、AES、SM4等算法开销

  前置条件：

  smmu=enable，uadk与uadk_engine已安装，加密相关模块驱动已加载且uacce_mode指定为1

  修改ceph.conf，启用uadk_engine

  ```shell
  vim /etc/ceph/ceph.conf
  [global]
  openssl_engine_opts = "engine_id=uadk_engine,dynamic_path=/usr/local/lib/engines-1.1/uadk_engine.so,default_algorithms=ALL,init=1"
  ```

  RGW进程使用ceph用户没有加速器设备的访问权限，[Service]项下的ExecStart命令，修改为root用户启动

  ```shell
  vim /usr/lib/systemd/system/ceph-radosgw@rgw.bucket1.service
  [Service]
  ExecStart=/usr/bin/radosgw -f --cluster ${CLUSTER} --name client.%i --setuser root --setgroup root
  PrivateDevices=no
  ```

  service文件修改生效，并重启RGW守护进程

  ```shell
  systemctl daemon-reload
  systemctl restart ceph-radosgw.target
  ```

- 启用KAE

  - KAE启用后，可用鲲鹏加速器加速卸载对象存储场景的MD5、AES、SM4等算法开销，暂不支持SHA256算法加速

    前置条件：

    smmu=disable，uadk与KAE已安装，加密相关模块驱动已加载且uacce_mode指定为2

    修改ceph.conf，启用uadk_engine

    ```shell
    vim /etc/ceph/ceph.conf
    [global]
    openssl_engine_opts = "engine_id=kae,dynamic_path=/usr/local/lib/engines-1.1/kae.so,default_algorithms=ALL,init=1"
    ```

    RGW进程使用ceph用户没有加速器设备的访问权限，[Service]项下的ExecStart命令，修改为root用户启动

    ```shell
    vim /usr/lib/systemd/system/ceph-radosgw@rgw.bucket1.service
    [Service]
    ExecStart=/usr/bin/radosgw -f --cluster ${CLUSTER} --name client.%i --setuser root --setgroup root
    PrivateDevices=no
    ```

    service文件修改生效，并重启RGW守护进程

    ```shell
    systemctl daemon-reload
    systemctl restart ceph-radosgw.target
    ```

### ceph-15.2.5 编译指导


编写日期：2022年12月2日

编写人员：李璐（lilu@kylinos.cn）


修订版本：V1.0
v1.0	创建文档	2022.11.30
		
1.环境信息
1.1软件环境
os版本：openEuler 20.03-LTS-SP3
ceph版本：15.2.5
GCC版本：7.3.0
Kernel：4.19.90-2112.8.0.0131.oe1.aarch64
Python版本：3.7.9

1.2硬件环境
设备类型	CPU	内存	硬盘	网卡	角色
虚拟机	Kunpeng-920	32G	30G+80G	172.177.135	MON，MGR，OSD

1.3源码文件
ceph源码包：https://download.ceph.com/rpm-15.2.5/el8/SRPMS/ceph-15.2.5-0.el8.src.rpm
spec文件：（见附录）

2.配置yum源
22.1上传镜像
将openEuler-20.03-LTS-SP3-everything-aarch64-dvd.iso镜像上传至/root/目录下

2.2创建目录
 mkdir -p /root/mnt/

2.3挂载镜像
 mount -o loop /root/openEuler-20.03-LTS-SP3-everything-aarch64-dvd.iso /root/mnt/

2.4配置repo文件
 vim /etc/yum.repos.d/openEuler.repo
修改文件如下
[local]
name = local
baseurl = file:///mnt
enabled = 1
gpgcheck = 0
gpgkey = http://repo.openeuler.org/openEuler-20.03-LTS-SP3/OS/$basearch/RPM-GPG-KEY-openEuler

3.软件包编译
1233.1下载ceph源码包
 wget https://download.ceph.com/rpm-15.2.5/el8/SRPMS/ceph-15.2.5-0.el8.src.rpm

3.2解压
 rpm -ivh ceph-15.2.5-0.el8.src.rpm

3.3修改ceph.spec文件
将附件中ceph.spec放入/root/rpmbuild/SPECS/路径下

3.4修改/usr/lib/rpm/macros.d/macros.python
将 ”%__python /usr/bin/python” 修改为” %__python /usr/bin/python3”，如下图所示：
![输入图片说明](https://foruda.gitee.com/images/1672729096281514554/fd7e4870_1665388.png "屏幕截图")
需使用python3编译，如果使用python2编译会出现语法错误

3.5安装编译依赖
 yum install cmake CUnit-devel boost-random expat-devel fuse-devel gcc-c++ gperf gperftools-devel java-devel junit keyutils-libs-devel leveldb-devel libaio-devel lttng-ust-devel libbabeltrace-devel libcap-ng-devel libcurl-devel libibverbs-devel liblockfile libnl3-devel  librabbitmq-devel librdkafka-devel librdmacm-devel lttng-ust-devel mailx ncurses-devel nss-devel openeuler-lsb openldap-devel openssl-devel python3-Cython python3-prettytable python3-sphinx  snappy-devel snappy-devel valgrind-devel xfsprogs-devel xmlstarlet  procmail libesmtp lz4-devel yasm 

3.6 安装其他依赖
 wget https://repo.openeuler.org/openEuler-20.03-LTS-SP3/EPOL/main/aarch64/Packages/liboath-devel-2.6.5-1.oe1.aarch64.rpm
 wget https://repo.openeuler.org/openEuler-20.03-LTS-SP3/EPOL/main/aarch64/Packages/liboath-2.6.5-1.oe1.aarch64.rpm
 rpm –ivh liboath-devel-2.6.5-1.oe1.aarch64.rpm 
 rpm –ivh liboath-2.6.5-1.oe1.aarch64.rpm

3.7升级libstdc++(rpm包见附件)
 rpm –ivh libstdc.*
![输入图片说明](https://foruda.gitee.com/images/1672729121936873389/89f31dcb_1665388.png "屏幕截图")
创建软连接
 find / -name "libstdc++fs.a"
 ln -s /usr/lib/gcc/aarch64-redhat-linux/9/libstdc++fs.a /lib64/libstdc++fs.a
注：若不进行软连接，编译的时候会报如下错误：
![输入图片说明](https://foruda.gitee.com/images/1672729130521866325/4010ba9d_1665388.png "屏幕截图")
3.8开始编译
 rpmbuild –ba /root/rpmbuild/SPECS/ceph.spec

3.9编译完成
切换到rpmbuild/RPMS/，查看生成的rpm包
输出软件包	输出目录
ceph-15.2.5-0.aarch64.rpm                         librados-devel-15.2.5-0.aarch64.rpm
cephadm-15.2.5-0.aarch64.rpm                      libradospp-devel-15.2.5-0.aarch64.rpm
ceph-base-15.2.5-0.aarch64.rpm                    libradosstriper1-15.2.5-0.aarch64.rpm
ceph-common-15.2.5-0.aarch64.rpm                  libradosstriper-devel-15.2.5-0.aarch64.rpm
ceph-debuginfo-15.2.5-0.aarch64.rpm               librbd1-15.2.5-0.aarch64.rpm
ceph-debugsource-15.2.5-0.aarch64.rpm             librbd-devel-15.2.5-0.aarch64.rpm
ceph-fuse-15.2.5-0.aarch64.rpm                    librgw2-15.2.5-0.aarch64.rpm
ceph-immutable-object-cache-15.2.5-0.aarch64.rpm  librgw-devel-15.2.5-0.aarch64.rpm
ceph-mds-15.2.5-0.aarch64.rpm                     python3-ceph-argparse-15.2.5-0.aarch64.rpm
ceph-mgr-15.2.5-0.aarch64.rpm                     python3-ceph-common-15.2.5-0.aarch64.rpm
ceph-mon-15.2.5-0.aarch64.rpm                     python3-cephfs-15.2.5-0.aarch64.rpm
ceph-osd-15.2.5-0.aarch64.rpm                     python3-rados-15.2.5-0.aarch64.rpm
ceph-radosgw-15.2.5-0.aarch64.rpm                 python3-rbd-15.2.5-0.aarch64.rpm
ceph-resource-agents-15.2.5-0.aarch64.rpm         python3-rgw-15.2.5-0.aarch64.rpm
ceph-selinux-15.2.5-0.aarch64.rpm                 rados-objclass-devel-15.2.5-0.aarch64.rpm
ceph-test-15.2.5-0.aarch64.rpm                    rbd-fuse-15.2.5-0.aarch64.rpm
libcephfs2-15.2.5-0.aarch64.rpm                   rbd-mirror-15.2.5-0.aarch64.rpm
libcephfs-devel-15.2.5-0.aarch64.rpm              rbd-nbd-15.2.5-0.aarch64.rpm
librados2-15.2.5-0.aarch64.rpm	/root/rpmbuild/RPMS/aarch64/
ceph-grafana-dashboards-15.2.5-0.noarch.rpm        ceph-mgr-k8sevents-15.2.5-0.noarch.rpm
ceph-mgr-cephadm-15.2.5-0.noarch.rpm               ceph-mgr-modules-core-15.2.5-0.noarch.rpm
ceph-mgr-dashboard-15.2.5-0.noarch.rpm             ceph-mgr-rook-15.2.5-0.noarch.rpm
ceph-mgr-diskprediction-cloud-15.2.5-0.noarch.rpm  ceph-prometheus-alerts-15.2.5-0.noarch.rpm
ceph-mgr-diskprediction-local-15.2.5-0.noarch.rpm	/root/rpmbuild/RPMS/noarch/
ceph-15.2.5-0.src.rpm	/root/rpmbuild/SRPMS/

4安装适配
4.1关闭防火墙
 systemctl stop firewalld.service
 systemctl disable firewalld.service
 sed -i 's/SELINUX=.*$/SELINUX=disabled/' /etc/selinux/config

4.2设置主机名称
 hostnamectl --static set-hostname euler-ceph0

4.3修改host文件
 vim /etc/hosts
172.17.7.135 euler-ceph0

4.4设置免密登录
 ssh-keygen -t rsa
 ssh-copy-id root@euler-ceph0

4.5制作ceph安装源
1)创建目录
 mkdir -p /root/ceph-repo/

2)复制安装包
 cp -f /root/rpmbuild/RPMS/aarch64/* /root/ceph15.2.5-repo/
 cp -f /root/rpmbuild/RPMS/noarch/* /root/ceph15.2.5-repo/

3)创建源
 createrepo /root/ceph15.2.5-repo/

4)修改/etc/yum.repos.d/openEuler.repo文件，新增内容如下：
[ceph]
name = Ceph-repo
baseurl = /root/ceph15.2.5-repo/
enabled = 1
gpgcheck = 0

4.6安装软件包
 yum install -y ceph ceph-base ceph-mds ceph-mgr ceph-mon ceph-osd ceph-radosgw ceph-common ceph-mgr-modules-core ceph-selinux libcephfs2 libradosstriper1 librgw2 python3-ceph-argparse python3-ceph-common python3-cephfs python3-rados python3-rbd python3-rgw
![输入图片说明](https://foruda.gitee.com/images/1672729154855169111/344820ad_1665388.png "屏幕截图")

4.7部署
5ceph安装验证
5.1切换目录
 cd /etc/ceph/

5.2创建集群
 ceph-deploy new euler-ceph0
![输入图片说明](https://foruda.gitee.com/images/1672729173848074539/7299026c_1665388.png "屏幕截图")

5.3修改配置文件/etc/ceph/ceph.conf，新增内容如下：
public_network = 10.1.163.0/24
cluster_network = 10.1.163.0/24

[mon]
mon_allow_pool_delete = true

5.4初始化监视器，部署MON节点
 ceph-deploy mon create-initial
![输入图片说明](https://foruda.gitee.com/images/1672729175638971772/188b0e8f_1665388.png "屏幕截图")

5.5初始化管理器，部署MGR节点
 ceph-deploy mgr create euler-ceph0
![输入图片说明](https://foruda.gitee.com/images/1672729183786996854/0806a7d1_1665388.png "屏幕截图")

5.6创建OSD节点
 ceph-deploy osd create euler-ceph0 --data /dev/vdb
![输入图片说明](https://foruda.gitee.com/images/1672729192924658620/6587443a_1665388.png "屏幕截图")

5.7查看结果
 ceph -s
![输入图片说明](https://foruda.gitee.com/images/1672729202655387767/a7bafa52_1665388.png "屏幕截图")
至此ceph适配已成功

6附件
由于附件较大无法上传gitee，有需要请联系liu_qinfei@163.com发送。




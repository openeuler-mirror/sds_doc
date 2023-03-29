
ceph-15.2.15 编译指导

编写日期：2022年11月30日

编写人员：徐赫（xuhe@kylinos.cn）

修订版本：V1.0
v1.0	创建文档	2022.11.30
		

1环境信息
1.1软件环境
系统版本：openEuler-20.03-LTS-SP3
ceph版本：15.2.15
GCC版本：7.3.0
内核版本：4.19.90-2112.8.0.0131.oe1.aarch64
Python版本：3.7.9

1.2硬件环境
设备类型	CPU	内存	硬盘	网络	角色
虚拟机	Kunpeng-920	16G	50G+10G	172.17.7.100	MON，MGR，OSD

1.3源码文件
ceph源码包：https://download.ceph.com/tarballs/ceph-15.2.15.tar.gz
spec文件：（见附录）

2配置yum源
2.1上传镜像
将openEuler-20.03-LTS-SP3-everything-aarch64-dvd.iso镜像上传至/root/目录下

2.2创建目录
 mkdir -p /root/mnt/

2.3挂载镜像
 mount -o loop /root/openEuler-20.03-LTS-SP3-everything-aarch64-dvd.iso /root/mnt/

2.4配置yum源
修改/etc/yum.repos.d/openEuler.repo新增内容如下：
[local]
name = local
baseurl = file:///mnt
enabled = 1
gpgcheck = 0
gpgkey = http://repo.openeuler.org/openEuler-20.03-LTS-SP3/OS/$basearch/RPM-GPG-KEY-openEuler

3软件包构建
3.1编译依赖
checkpolicy-3.1-1.oe1.aarch64
CUnit-devel-2.1.3-22.oe1.aarch64
expat-devel-2.2.9-3.oe1.aarch64
gperftools-devel-2.8-1.oe1.aarch64
keyutils-libs-devel-1.6.3-1.oe1.aarch64
libbabeltrace-devel-1.5.8-1.oe1.aarch64
liboath-2.6.5-1.oe1.aarch64
liboath-devel-2.6.5-1.oe1.aarch64
librabbitmq-devel-0.9.0-4.oe1.aarch64
librdkafka-devel-0.11.4-3.oe1.aarch64
lttng-ust-devel-2.10.1-8.oe1.aarch64
lz4-devel-1.9.2-3.oe1.aarch64
nss-devel-3.54.0-7.oe1.aarch64
openeuler-lsb-5.0-1.oe1.aarch64
openldap-devel-2.4.50-6.oe1.aarch64
openssl-devel-1.1.1f-13.oe1.aarch64
python3-Cython-0.29.14-3.oe1.aarch64
python3-devel-3.7.9-18.oe1.aarch64
python3-prettytable-0.7.2-18.oe1.noarch
python3-sphinx-1.8.4-2.oe1.noarch
rdma-core-devel-35.0-2.oe1.aarch64
selinux-policy-devel-3.14.2-76.oe1.noarch
systemd-devel-243-49.oe1.aarch64

3.2部署源文件
文件	存放目录
ceph-15.2.15.tar.gz	/root/rpmbuild/SOURCES/
ceph.spec	/root/rpmbuild/SPECS/

3.3构建
 rpmbuild -ba ceph.spec
![输入图片说明](https://foruda.gitee.com/images/1672729540692437425/9508af84_1665388.png "屏幕截图")

3.4完成
输出软件包	输出目录
ceph-15.2.15-0.aarch64.rpm
ceph-base-15.2.15-0.aarch64.rpm
ceph-common-15.2.15-0.aarch64.rpm
ceph-debuginfo-15.2.15-0.aarch64.rpm
ceph-debugsource-15.2.15-0.aarch64.rpm
ceph-fuse-15.2.15-0.aarch64.rpm
ceph-immutable-object-cache-15.2.15-0.aarch64.rpm
ceph-mds-15.2.15-0.aarch64.rpm
ceph-mgr-15.2.15-0.aarch64.rpm
ceph-mon-15.2.15-0.aarch64.rpm
ceph-osd-15.2.15-0.aarch64.rpm
ceph-radosgw-15.2.15-0.aarch64.rpm
ceph-resource-agents-15.2.15-0.aarch64.rpm
ceph-selinux-15.2.15-0.aarch64.rpm
ceph-test-15.2.15-0.aarch64.rpm
libcephfs2-15.2.15-0.aarch64.rpm
libcephfs-devel-15.2.15-0.aarch64.rpm
librados2-15.2.15-0.aarch64.rpm
librados-devel-15.2.15-0.aarch64.rpm
libradospp-devel-15.2.15-0.aarch64.rpm
libradosstriper1-15.2.15-0.aarch64.rpm
libradosstriper-devel-15.2.15-0.aarch64.rpm
librbd1-15.2.15-0.aarch64.rpm
librbd-devel-15.2.15-0.aarch64.rpm
librgw2-15.2.15-0.aarch64.rpm
librgw-devel-15.2.15-0.aarch64.rpm
python3-ceph-argparse-15.2.15-0.aarch64.rpm
python3-ceph-common-15.2.15-0.aarch64.rpm
python3-cephfs-15.2.15-0.aarch64.rpm
python3-rados-15.2.15-0.aarch64.rpm
python3-rbd-15.2.15-0.aarch64.rpm
python3-rgw-15.2.15-0.aarch64.rpm
rados-objclass-devel-15.2.15-0.aarch64.rpm
rbd-fuse-15.2.15-0.aarch64.rpm
rbd-mirror-15.2.15-0.aarch64.rpm
rbd-nbd-15.2.15-0.aarch64.rpm	/root/rpmbuild/RPMS/aarch64/
cephadm-15.2.15-0.noarch.rpm
ceph-grafana-dashboards-15.2.15-0.noarch.rpm
ceph-mgr-cephadm-15.2.15-0.noarch.rpm
ceph-mgr-dashboard-15.2.15-0.noarch.rpm
ceph-mgr-diskprediction-cloud-15.2.15-0.noarch.rpm
ceph-mgr-diskprediction-local-15.2.15-0.noarch.rpm
ceph-mgr-k8sevents-15.2.15-0.noarch.rpm
ceph-mgr-modules-core-15.2.15-0.noarch.rpm
ceph-mgr-rook-15.2.15-0.noarch.rpm
ceph-prometheus-alerts-15.2.15-0.noarch.rpm	/root/rpmbuild/RPMS/noarch/
ceph-15.2.15-0.src.rpm	/root/rpmbuild/SRPMS/

4安装适配
4.1关闭防火墙
 systemctl stop firewalld.service
 systemctl disable firewalld.service
 sed -i 's/SELINUX=.*$/SELINUX=disabled/' /etc/selinux/config

4.2设置主机名称
 hostnamectl --static set-hostname euler-ceph0

4.3修改host文件
172.17.7.100 euler-ceph0

4.4设置免密登录
 ssh-keygen -t rsa
 ssh-copy-id root@euler-ceph0

4.5制作ceph安装源
1)创建目录
 mkdir -p /root/ceph-repo/

2)复制安装包
 cp -f /root/rpmbuild/RPMS/aarch64/* /root/ceph-repo/
 cp -f /root/rpmbuild/RPMS/noarch/* /root/ceph-repo/

3)创建源
 createrepo /root/ceph-repo/

4)修改/etc/yum.repos.d/openEuler.repo文件，新增内容如下：
[ceph]
name = Ceph-repo
baseurl = /root/ceph-repo/
enabled = 1
gpgcheck = 0

4.6安装软件包
 yum install -y ceph ceph-base ceph-mds ceph-mgr ceph-mon ceph-osd ceph-radosgw ceph-common ceph-mgr-modules-core ceph-selinux libcephfs2 libradosstriper1 librgw2 python3-ceph-argparse python3-ceph-common python3-cephfs python3-rados python3-rbd python3-rgw
![输入图片说明](https://foruda.gitee.com/images/1672729550437878174/e36ebc82_1665388.png "屏幕截图")

4.7部署
5ceph安装验证
5.1切换目录
 cd /etc/ceph/

5.2创建集群
 ceph-deploy new euler-ceph0
![输入图片说明](https://foruda.gitee.com/images/1672729562456836934/42158615_1665388.png "屏幕截图")

5.3修改配置文件/etc/ceph/ceph.conf，新增内容如下：
public_network = 10.1.163.0/24
cluster_network = 10.1.163.0/24

[mon]
mon_allow_pool_delete = true

5.4初始化监视器，部署MON节点
 ceph-deploy mon create-initial

5.5初始化管理器，部署MGR节点
 ceph-deploy mgr create euler-ceph0

5.6创建OSD节点
 ceph-deploy osd create euler-ceph0 --data /dev/vdb

5.7查看结果
 ceph -s
至此ceph适配已成功

6附件
由于附件较大无法上传gitee，有需要请联系liu_qinfei@163.com发送。

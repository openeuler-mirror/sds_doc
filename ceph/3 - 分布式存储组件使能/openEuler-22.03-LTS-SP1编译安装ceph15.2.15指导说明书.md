# openEuler-22.03-LTS-SP1编译安装ceph-15.2.15

## 1.环境信息

### 1.软件环境

~~~bash
系统版本：openEuler-22.03-LTS-SP1
ceph版本：15.2.15
GCC版本：10.3.1
内核版本：5.10.0-136.42.0.120.oe2203sp1
Python版本：Python 3.9.9
~~~

编译环境和部署环境相同

### 2.硬件环境

~~~bash
设备类型：虚拟机
CPU：x86_64
内存：16G
硬盘：160G（建议100G以上，防止编译过程空间不足）
~~~

## 2.编译

### 2.1 环境搭建

参考以下链接：

```html
https://docs.openeuler.org/zh/docs/22.03_LTS_SP1/docs/Quickstart/quick-start.html#%E5%AE%89%E8%A3%85
```

### 2.2 安装环境

#### 2.2.1 预安装环境

```shell
yum update
yum install tar zip gcc zlib zlib-devel openssl openssl-devel make git
yum install rpmdevtools rpm-build
rpmdev-setuptree
```

### 2.3 ceph-4.2.15编译

```shell
cd /root/
wget https://download.ceph.com/tarballs/ceph-15.2.15.tar.gz
tar -zxvf /root/ceph-15.2.15.tar.gz
mv /root/ceph-15.2.15.tar.gz /root/rpmbuild/SOURCES/ceph-15.2.15.tar.bz2
cp /root/ceph-15.2.15/ceph.spec /root/rpmbuild/SPECS/
```

#### 2.3.1 安装编译依赖 

```
yum install checkpolicy CUnit-devel expat-devel gperftools-devel keyutils-libs-devel libbabeltrace-devel liboath liboath-devel librabbitmq-devel librdkafka-devel lttng-ust-devel lz4-devel nss-devel openeuler-lsb openldap-devel openssl-devel python3-Cython python3-devel python3-prettytable python3-sphinx rdma-core-devel selinux-policy-devel systemd-devel
```

#### 2.3.2 编译

```
cd /root/rpmbuild/SPECS/
yum-builddep ceph.spec  
rpmbuild -ba ceph.spec  
```

### 2.4 编译问题

```
1. ./boost/thread/pthread/thread_data.hpp:60:5: error: missing binary operator before token "("
   60 | #if PTHREAD_STACK_MIN > 0

解决方法：
vi rpmbuild/BUILD/ceph-15.2.15/src/boost/boost/thread/pthread/thread_data.hpp
#if PTHREAD_STACK_MIN > 0 改为 #ifdef PTHREAD_STACK_MIN

2. 主要报错信息：/root/rpmbuild/BUILD/ceph-15.2.15/src/compressor/snappy/SnappyCompressor.h:99:13: error: 'uint32' is not a member of 'snappy'
   99 |     snappy::uint32 res_len = 0;
      |             ^~~~~~
参考：
https://tracker.ceph.com/issues/50934
https://github.com/ceph/ceph/pull/42507
解决方法：
vim ceph-15.2.15/src/compressor/snappy/SnappyCompressor.h +99
-    snappy::uint32 res_len = 0;
+    uint32_t res_len = 0;

3.以上修改涉及源码，需要重新打包，再次编译
解决方法：
1.tar -zcvf ceph-15.2.15.tar.gz ceph-15.2.15/ && mv /root/ceph-15.2.15.tar.gz /root/rpmbuild/SOURCES/ceph-15.2.15.tar.bz2 再次编译
2.将.patch文件放在解压后的ceph-14.2.15目录下，编辑ceph.spec，使用添加patch。
patch下载地址：https://gitee.com/yafei2023/my_doc/blob/master/ceph/ceph15-openEuler-2203-LTS-SP1/ceph-15.2.15-patch.patch
```

### 2.5 编译出包：

```
ceph-15.2.15-0.x86_64.rpm                           ceph-radosgw-15.2.15-0.x86_64.rpm
cephadm-15.2.15-0.noarch.rpm                        ceph-test-15.2.15-0.x86_64.rpm
ceph-base-15.2.15-0.x86_64.rpm                      libcephfs2-15.2.15-0.x86_64.rpm
ceph-common-15.2.15-0.x86_64.rpm                    libcephfs-devel-15.2.15-0.x86_64.rpm
ceph-debuginfo-15.2.15-0.x86_64.rpm                 librados2-15.2.15-0.x86_64.rpm
ceph-debugsource-15.2.15-0.x86_64.rpm               librados-devel-15.2.15-0.x86_64.rpm
ceph-fuse-15.2.15-0.x86_64.rpm                      libradospp-devel-15.2.15-0.x86_64.rpm
ceph-grafana-dashboards-15.2.15-0.noarch.rpm        librbd1-15.2.15-0.x86_64.rpm
ceph-immutable-object-cache-15.2.15-0.x86_64.rpm    librbd-devel-15.2.15-0.x86_64.rpm
ceph-mds-15.2.15-0.x86_64.rpm                       librgw2-15.2.15-0.x86_64.rpm
ceph-mgr-15.2.15-0.x86_64.rpm                       librgw-devel-15.2.15-0.x86_64.rpm
ceph-mgr-cephadm-15.2.15-0.noarch.rpm               python3-ceph-argparse-15.2.15-0.x86_64.rpm
ceph-mgr-dashboard-15.2.15-0.noarch.rpm             python3-ceph-common-15.2.15-0.x86_64.rpm
ceph-mgr-diskprediction-cloud-15.2.15-0.noarch.rpm  python3-cephfs-15.2.15-0.x86_64.rpm
ceph-mgr-diskprediction-local-15.2.15-0.noarch.rpm  python3-rados-15.2.15-0.x86_64.rpm
ceph-mgr-k8sevents-15.2.15-0.noarch.rpm             python3-rbd-15.2.15-0.x86_64.rpm
ceph-mgr-modules-core-15.2.15-0.noarch.rpm          python3-rgw-15.2.15-0.x86_64.rpm
ceph-mgr-rook-15.2.15-0.noarch.rpm                  rados-objclass-devel-15.2.15-0.x86_64.rpm
ceph-mon-15.2.15-0.x86_64.rpm                       rbd-fuse-15.2.15-0.x86_64.rpm
ceph-osd-15.2.15-0.x86_64.rpm                       rbd-mirror-15.2.15-0.x86_64.rpm
ceph-prometheus-alerts-15.2.15-0.noarch.rpm         rbd-nbd-15.2.15-0.x86_64.rpm
```

## 3.安装

```
mkdir /root/ceph
cp /root/rpmbuild/RPMS/x86_64/*.rpm /root/ceph
cp /root/rpmbuild/RPMS/noarch/*.rpm /root/ceph
cd /root/ceph
yum install *.rpm
```

## 4.部署

### 1.设置节点名称

| IP             | 作用          |
| :------------- | :------------ |
| 192.168.42.131 | services-ceph |

~~~bash
 vim /etc/hosts
 + 192.168.42.131 services-ceph
~~~

### 2.安装ceph-deploy

```bash
cd /root
git clone https://gitee.com/src-openeuler/ceph-deploy.git
cp /root/ceph-deploy/* /root/rpmbuild/SOURCES/
cp /root/ceph-deploy/python-ceph-deploy.spec /root/rpmbuild/SPECS/
rpmbuild -ba /root/rpmbuild/SPECS/python-ceph-deploy.spec
cd /root/rpmbuild/RPMS/noarch/
yum install python3-ceph-deploy-2.1.0-2.noarch.rpm
```

### 3.部署

hostname需要和节点名称一致

```
ceph-deploy new services-ceph

# 生成秘钥
sudo ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
# 创建管理员
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
# osd用户
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'
# 添加秘钥到ceph.mon.keyring文件
sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
# 修改ceph.mon.keyring拥有者
sudo chown ceph:ceph /tmp/ceph.mon.keyring
# 映射监视器
monmaptool --create --add services-ceph 192.168.42.131 --fsid {uuid} /tmp/monmap
#权限赋予
chown ceph:ceph /var/lib/ceph
#创建mon文件夹
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-services-ceph
# 配置monitor daemon
sudo -u ceph ceph-mon --mkfs -i services-ceph --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```

ceph.conf参考（单机）

```
[global]
fsid = 8dc8e218-dc38-488d-be5c-f2aa136cf7fe
mon_initial_members = services-ceph
mon_host = 192.168.42.131
public network = 192.168.42.0/24
cluster network = 192.168.42.0/24
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd journal size = 1024
osd pool default size = 1
osd pool default min size = 1
osd crush chooseleaf type = 0
```

启动monitor

```
sudo systemctl start ceph-mon@services-ceph
```

详细步骤参考：

```
https://xujiyou.work/%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8/Ceph/Ceph%E9%83%A8%E7%BD%B2/ceph%E6%89%8B%E5%8A%A8%E9%83%A8%E7%BD%B2.html
```

## 5.成功截图

![img](https://gitee.com/yafei2023/my_doc/raw/master/ceph/ceph15-openEuler-2203-LTS-SP1/image-20230802161159415.png)

### 5.1.部署问题解决

~~~bash
1.ceph -s 出现mon is allowing insecure global_id reclaim的警告
ceph config set mon auth_allow_insecure_global_id_reclaim false

2.OSD(s) have broken BlueStore compression
参考：https://gitee.com/src-openeuler/ceph/issues/I5D4PW

3.osd pg显示100% pg inactive.
参考：https://www.cnblogs.com/boshen-hzb/p/13305560.html

4.monitors have not enabled msgr2
ceph mon enable-msgr2

5.ceph-deploy --overwrite-conf mon create-initial报错admin_socket: exception getting command descriptions: [Errno 2] No such file or directory。
vim /etc/ceph/ceph.conf
public_network = 192.168.57.0/24

也可以手动部署：
https://xujiyou.work/%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8/Ceph/Ceph%E9%83%A8%E7%BD%B2/ceph%E6%89%8B%E5%8A%A8%E9%83%A8%E7%BD%B2.html

6.RuntimeError: bootstrap-rgw keyring not found; run 'gatherkeys'
执行命令：
ceph-deploy gatherkeys ceph01

7.ceph -s报No module named werkzeug
pip3 install werkzeug
~~~

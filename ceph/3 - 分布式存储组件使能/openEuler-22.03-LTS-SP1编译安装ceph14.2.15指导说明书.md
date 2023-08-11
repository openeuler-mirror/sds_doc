# openEuler-22.03-LTS-SP1编译安装ceph-14.2.15

## 1.环境信息

### 1.1.软件环境

~~~bash
系统版本：openEuler-22.03-LTS-SP1
ceph版本：V14.2.15
GCC版本：10.3.1
内核版本：5.10.0-136.12.0.86.oe2203sp1
Python版本：Python 2.7（手动编译安装）
~~~

编译环境和部署环境相同

### 1.2.硬件环境

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
yum rpmdevtools rpm-build
rpmdev-setuptree
```

#### 2.2.2 安装python2.7

手动安装python2.7

**编译**

```shell
git clone https://gitee.com/src-openeuler/python2.git
cd python2
cp -r . /root/rpmbuild/SOURCES/
cp python2.spec /root/rpmbuild/SPECS/
cd /root/rpmbuild/SPECS/
yum-builddep python2.spec
QA_RPATHS=0x0001 rpmbuild -ba python2.spec
```

**安装**

```shell
cd /root/
mkdir python2_rpm
cp /root/rpmbuild/RPMS/x86_64/* /root/python2_rpm/
cp /root/rpmbuild/RPMS/noarch/* /root/python2_rpm/
rpm -ivh /root/python2_rpm/*.rpm --nodeps
```

**安装setuptool和pip**

1.setuptool

```shell
wget https://pypi.python.org/packages/41/5f/6da80400340fd48ba4ae1c673be4dc3821ac06cd9821ea60f9c7d32a009f/setuptools-38.4.0.zip
cd setuptools的解压目录
python setup.py install
```

2.pip

```shell
wget https://pypi.python.org/packages/11/b6/abcb525026a4be042b486df43905d6893fb04f05aac21c32c638e939e447/pip-9.0.1.tar.gz
cd pip的解压目录
python setup.py install
```

### 2.3 ceph-4.2.15编译

```shell
cd /root/
wget https://download.ceph.com/tarballs/ceph-14.2.15.tar.gz
tar -zxvf /root/ceph-14.2.15.tar.gz
mv /root/ceph-14.2.15.tar.gz /root/rpmbuild/SOURCES/ceph-14.2.15.tar.bz2
cp /root/ceph-14.2.15/ceph.spec /root/rpmbuild/SPECS/
```

#### 2.3.1 安装编译依赖 

```
pip install sphinx cpython pyOpenSSL prettytable pecan

yum install cmake cryptsetup fuse-devel gcc-c++ gperf leveldb-devel libaio-devel libblkid-devel libcap-ng-devel libcurl-devel libnl3-devel liboath-devel libtool libudev-devel libxml2-devel ncurses-devel snappy-devel valgrind-devel xfsprogs-devel xmlstarlet yasm  rdma-core-devel lz4-devel openldap-devel nss-devel bison npm re2-devel socat yaml-cpp-devel libbabeltrace-devel lttng-ust-devel librabbitmq-devel CUnit-devel checkpolicy gperftools-devel librdkafka-devel openeuler-lsb selinux-policy-devel
```

#### 2.3.2 编译

```
cd /root/rpmbuild/SPECS/
yum-builddep ceph.spec  
rpmbuild -ba ceph.spec  
```

### 2.4 编译问题

```
1. CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
  Could NOT find Python3Libs (missing: PYTHON3_LIBRARIES
  PYTHON3_INCLUDE_DIRS) (Required is exact version "3.9.9")

解决方法：yum install python3-devel

2.CMake Error at cmake/modules/FindCython.cmake:13 (message):
  Could not find cython: /usr/bin/python2.7: No module named cython

解决方法：手动安装cpython
wget https://files.pythonhosted.org/packages/dc/f6/e8e302f9942cbebede88b1a0c33d0be3a738c3ac37abae87254d58ffc51c/Cython-0.29.33.tar.gz
解压 cd Cython-0.29.33目录
python setup.py install

3.CMake Error at cmake/modules/FindCython.cmake:13 (message):
  Could not find cython3: /usr/bin/python3.9: No module named cython

解决方法：yum install python3-Cython

4. ./boost/thread/pthread/thread_data.hpp:60:5: error: missing binary operator before token "("
   60 | #if PTHREAD_STACK_MIN > 0

解决方法：
vi rpmbuild/BUILD/ceph-14.2.15/src/boost/boost/thread/pthread/thread_data.hpp
#if PTHREAD_STACK_MIN > 0 改为 #ifdef PTHREAD_STACK_MIN

5. 主要报错信息：/root/rpmbuild/BUILD/ceph-14.2.15/src/global/signal_handler.h:24:26: error: 'sys_siglist' was not declared in this scope

参考：
https://gitee.com/src-openeuler/ceph/issues/I41F9W
https://lists.yoctoproject.org/g/meta-virtualization/topic/76190251?p=Created,,,20,2,0,0
解决方法：
ceph.spec 中添加  -DWITH_REENTRANT_STRSIGNAL=ON \
+1205 |  -DWITH_REENTRANT_STRSIGNAL=ON \

6. 主要报错信息：/root/rpmbuild/BUILD/ceph-14.2.15/src/compressor/snappy/SnappyCompressor.h:99:13: error: 'uint32' is not a member of 'snappy'
   99 |     snappy::uint32 res_len = 0;
      |             ^~~~~~

参考：
https://tracker.ceph.com/issues/50934
https://github.com/ceph/ceph/pull/42507
解决方法：
vim ceph-14.2.15/src/compressor/snappy/SnappyCompressor.h +99
-    snappy::uint32 res_len = 0;
+    uint32_t res_len = 0;

7. 主要报错信息：/root/rpmbuild/BUILD/ceph-14.2.15/src/test/librgw_file_nfsns.cc:1082:6: error: variable or field 'LibRGW_' declared void
 1082 | TEST(LibRGW, CLEANUP) {
      |      ^~~~~~

解决方法：
参考社区：https://gitee.com/src-openeuler/ceph/issues/I61X2K
参考 https://github.com/ceph/ceph/pull/43491/commits/53040e4c0e9e86710b8800dbb7ea15b3fa196ebf

8. 上述报错4、6、7需要修改源码，需要修改源码后进行打包再次编译。
解决方法：
1.tar -zcvf ceph-14.2.15.tar.gz ceph-14.2.15/ && mv /root/ceph-14.2.15.tar.gz /root/rpmbuild/SOURCES/ceph-14.2.15.tar.bz2 再次编译
2.将.patch文件放在解压后的ceph-14.2.15目录下，编辑ceph.spec，使用添加patch。
patch下载地址：https://gitee.com/yafei2023/my_doc/blob/master/ceph/ceph14-openEuler-2203-LTS-SP1/ceph-14.2.15-patch.patch
```

### 2.5 编译出包：

```
ceph-14.2.15-0.src.rpm
ceph-mgr-diskprediction-local-14.2.15-0.noarch.rpm
ceph-base-14.2.15-0.x86_64.rpm
ceph-radosgw-14.2.15-0.x86_64.rpm
librgw2-14.2.15-0.x86_64.rpm
ceph-mgr-dashboard-14.2.15-0.noarch.rpm
ceph-debugsource-14.2.15-0.x86_64.rpm
librados2-14.2.15-0.x86_64.rpm
ceph-mon-14.2.15-0.x86_64.rpm
rbd-mirror-14.2.15-0.x86_64.rpm
ceph-mgr-14.2.15-0.x86_64.rpm
ceph-fuse-14.2.15-0.x86_64.rpm
librbd1-14.2.15-0.x86_64.rpm
libcephfs2-14.2.15-0.x86_64.rpm
python-rbd-14.2.15-0.x86_64.rpm
python-rados-14.2.15-0.x86_64.rpm
python-cephfs-14.2.15-0.x86_64.rpm
python3-rbd-14.2.15-0.x86_64.rpm
ceph-mds-14.2.15-0.x86_64.rpm
python3-rados-14.2.15-0.x86_64.rpm
rbd-nbd-14.2.15-0.x86_64.rpm
librados-devel-14.2.15-0.x86_64.rpm
python3-cephfs-14.2.15-0.x86_64.rpm
python-rgw-14.2.15-0.x86_64.rpm
ceph-grafana-dashboards-14.2.15-0.noarch.rpm
python3-rgw-14.2.15-0.x86_64.rpm
rbd-fuse-14.2.15-0.x86_64.rpm
ceph-mgr-diskprediction-cloud-14.2.15-0.noarch.rpm
python-ceph-argparse-14.2.15-0.x86_64.rpm
python3-ceph-argparse-14.2.15-0.x86_64.rpm
librbd-devel-14.2.15-0.x86_64.rpm
ceph-mgr-k8sevents-14.2.15-0.noarch.rpm
libradospp-devel-14.2.15-0.x86_64.rpmlibcephfs-devel-14.2.15-0.x86_64.rpm
ceph-mgr-rook-14.2.15-0.noarch.rpm
x86_64/ceph-14.2.15-0.x86_64.rpm
rados-objclass-devel-14.2.15-0.x86_64.rpm
python-ceph-compat-14.2.15-0.x86_64.rpm
librgw-devel-14.2.15-0.x86_64.rpm
ceph-mgr-ssh-14.2.15-0.noarch.rpm
ceph-osd-14.2.15-0.x86_64.rpmceph-common-14.2.15-0.x86_64.rpm
ceph-test-14.2.15-0.x86_64.rpm
ceph-debuginfo-14.2.15-0.x86_64.rpm
```

## 3.安装

```
mkdir /root/ceph
cp /root/rpmbuild/RPMS/x86_64/* /root/ceph/
cp /root/rpmbuild/RPMS/noarch/*.rpm /root/ceph
cd /root/ceph
rpm -ivh *.rpm --nodeps --force
```

## 4.部署

```
参考手动部署：https://xujiyou.work/%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8/Ceph/Ceph%E9%83%A8%E7%BD%B2/ceph%E6%89%8B%E5%8A%A8%E9%83%A8%E7%BD%B2.html
```

## 5.成功截图

![img](https://gitee.com/yafei2023/my_doc/raw/master/ceph/ceph14-openEuler-2203-LTS-SP1/image-20230727171802805.png)

# openEuler-22.03-LTS-SP1编译安装ceph-15.2.15

## 1.环境信息

### 1.软件环境

~~~bash
系统版本：openEuler-22.03-LTS-SP1
ceph版本：15.2.15
GCC版本：7.3.0
内核版本：4.19.90-2109.1.0.0108.oe1.aarch64
Python版本：Python 3.7
~~~

编译环境和部署环境相同

### 2.硬件环境

~~~bash
设备类型：虚拟机
CPU：aarch64
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
yum install checkpolicy CUnit-devel expat-devel gperftools-devel keyutils-libs-devel libbabeltrace-devel librabbitmq-devel librdkafka-devel lttng-ust-devel lz4-devel nss-devel openeuler-lsb openldap-devel openssl-devel python3-Cython python3-devel python3-prettytable python3-sphinx rdma-core-devel selinux-policy-devel systemd-devel
```

#### 2.3.2 安装编译依赖 

1. 下载liboath源码及补丁。

    ```
    yum install git -y
    git config --global http.sslVerify false
    git clone https://gitee.com/src-openeuler/oath-toolkit.git
    ```

2. yum安装rpm打包所需的依赖。

    ```
    yum install wget rpmdevtools gtk-doc pam-devel xmlsec1-devel libtool libtool-ltdl-devel createrepo cmake -y
    ```

3. 创建rpmbuild目录，并将patch文件和源码包移动到“/root/rpmbuild/SOURCES”目录下。

    ```
    rpmdev-setuptree
    cd oath-toolkit
    mv 0001-oath-toolkit-2.6.5-lockfile.patch /root/rpmbuild/SOURCES
    mv oath-toolkit-2.6.5.tar.gz /root/rpmbuild/SOURCES
    cp oath-toolkit.spec /root/rpmbuild/SPECS/
    ```

4. 编译RPM包。

    ```
    rpmbuild -bb /root/rpmbuild/SPECS/oath-toolkit.spec
    ```

5. 安装liboath liboath-devel

    ```
    mkdir /root/liboath 
    cp /root/rpmbuild/RPMS/x86_64/*.rpm /root/liboath 
    cp /root/rpmbuild/RPMS/noarch/*.rpm /root/liboath 
    cd /root/liboath 
    yum install *.rpm
    ```

#### 2.3.3 升级libstdc++

libstdc++-9.2.1-1.fc31.aarch64.rpm

```
下载地址：http://ftp.pbone.net/mirror/archive.fedoraproject.org/fedora/linux/releases/31/Everything/aarch64/os/Packages/l/libstdc++-9.2.1-1.fc31.aarch64.rpm
```

libstdc++-devel-9.2.1-1.fc31.aarch64.rpm

```
下载地址：http://ftp.pbone.net/mirror/archive.fedoraproject.org/fedora/linux/releases/31/Everything/aarch64/os/Packages/l/libstdc++-devel-9.2.1-1.fc31.aarch64.rpm
```

libstdc++-static-9.2.1-1.fc31.aarch64.rpm

```
下载地址：http://ftp.pbone.net/mirror/archive.fedoraproject.org/fedora/linux/releases/31/Everything/aarch64/os/Packages/l/libstdc++-static-9.2.1-1.fc31.aarch64.rpm
```

安装rpm并创建软连接
find / -name "libstdc++fs.a"
ln -s /usr/lib/gcc/aarch64-redhat-linux/9/libstdc++fs.a /lib64/libstdc++fs.a

注：若不进行软连接，编译的时候会报如下错误：

```
主要报错：CMake Error at cmake/modules/BuildBoost.cmake:278 (_add_executable):
  Target "unittest_rgw_bencode" links to target "StdFilesystem::filesystem"
  but the target was not found.  Perhaps a find_package() call is missing for
  an IMPORTED target, or an ALIAS target is missing?
  Call Stack (most recent call first):
  src/test/rgw/CMakeLists.txt:16 (add_executable)
```

#### 2.3.5 其他

修改/usr/lib/rpm/macros.d/macros.python 将 ”%__python /usr/bin/python” 修改为” %__python /usr/bin/python3”，如下图所示： ![输入图片说明](https://foruda.gitee.com/images/1672729096281514554/fd7e4870_1665388.png) 需使用python3编译，如果使用python2编译会出现语法错误

#### 2.3.6 编译

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
4.报错检查未打包文件：/usr/lib/rpm/check-files /root/rpmbuild/BUILDROOT/ceph-15.2.15-0.aarch64
错误：发现已安装(但未打包的)文件：
   /usr/share/ceph/mgr/__pycache__/mgr_module.cpython-37.opt-1.pyc
   /usr/share/ceph/mgr/__pycache__/mgr_module.cpython-37.pyc
   /usr/share/ceph/mgr/__pycache__/mgr_util.cpython-37.opt-1.pyc
   /usr/share/ceph/mgr/__pycache__/mgr_util.cpython-37.pyc
参考：
https://blog.csdn.net/developerof/article/details/112139268 
修改spec文件的%install下面添加
%define _unpackaged_files_terminate_build 0

vi /usr/lib/rpm/macros
#%__check_files %{_rpmconfigdir}/check-files %{buildroot}
```

### 2.5 编译出包：

```
ceph-15.2.15-8.aarch64.rpm                      
ceph-base-15.2.15-0.aarch64.rpm                  
ceph-common-15.2.15-g.aarch64.rpm                 
ceph-debuginfo-15.2.15-8.aarch64.rpm             
ceph-debugsource-15.2.15-.aarch64.rpm            
ceph-immutable-object-cache-15.2.15-0.aarch64.rpm 
ceph-fuse-15.2.15-0.aarch64.rpm             
ceph-mds-15.2.15-0.aarch64.rpm
ceph-mgr-15.2.15-0.aarch64.rpm
ceph-mon-15.2.15-0.aarch64.rpm
ceph-osd-15.2.15-0.aarch64.rpm
ceph-radosgw-15.2.15-0.aarch64.rpm
ceph-test-15.2.15-0.aarch64.rpm
libcephfs2-15.2.15-0.aarch64.rpm
ibcephfs-devel-15.2.15-0.aarch64.rpm
ibrados2-15.2.15-0.aarch64.rpm
librados-devel-15.2.15-0.aarch64.rpm
libradospp-devel-15.2.15-0.aarch64.rpm
librbd1-15.2.15-0.aarch64.rpm
librbd-devel-15.2.15-0.aarch64.rpm
librgu2-15.2.15-0.aarch64.rpm
librgw-devel-15.2.15-0.aarch64.rpm
python3-ceph-argparse-15.2.15-0.aarch64.rpm
python3-ceph-common-15.2.15-0.aarch64.rpm
python3-cephfs-15.2.15-0.aarch64.rpm
python3-rados-15.2.15-0.aarch64.rpm
python3-rbd-15.2.15-0.aarch64.rpm
python3-rgw-15.2.15-0.aarch64.rpm
rados-objclass-devel-15.2.15-.aarch64.rpm
rbd-fuse-15.2.15-0.aarch4.rpm
rbd-mirror-15.2.15-0.aarch64.rpm
rbd-nbd-15.2.15-0.aarch64.rpm
cephadm-15.2.15-0.noarch.rpm
ceph-grafana-dashboards-15.2.15-0.noarch.rpm
ceph-mgr-cephadm-15.2.15-0.noarch.rpm
ceph-mgr-dashboard-15.2.15-0.noarch.rpm
ceph-mgr-diskprediction-cloud-15.2.15-0.noarch.rpm
ceph-mgr-diskprediction-local-15.2.15-0.noarch.rpm
ceph-mgr-k8sevents-15.2.15-0.noarch.rpm
ceph-mgr-modules-core-15.2.15-0.noarch.rpm
ceph-mgr-rook-15.2.15-0.noarch.rpm
ceph-prometheus-alerts-15.2.15-0.noarch.rpm
```

## 3.安装

```
mkdir /root/ceph
cp /root/rpmbuild/RPMS/aarch64/*.rpm /root/ceph
cp /root/rpmbuild/RPMS/noarch/*.rpm /root/ceph
cd /root/ceph
yum install *.rpm
```

## 4.部署

```
详细步骤参: https://xujiyou.work/%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8/Ceph/Ceph%E9%83%A8%E7%BD%B2/ceph%E6%89%8B%E5%8A%A8%E9%83%A8%E7%BD%B2.html
```

## 5.成功截图

![](https://gitee.com/yafei2023/my_doc/raw/master/ceph/ceph15-openEuler-2203-LTS-SP1/arm-ceph15.2.15.png)

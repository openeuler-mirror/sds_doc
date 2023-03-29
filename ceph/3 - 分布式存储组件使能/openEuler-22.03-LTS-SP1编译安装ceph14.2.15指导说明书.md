# openEuler-22.03-LTS-SP1编译安装ceph-14.2.15

## 1.声明

~~~bash
编写日期：2023年02月17日
编写人员：朱超（tom_toworld@163.com）
修订版本：V1.0
~~~

## 2.编译

### 1.环境信息

#### 1.软件环境

~~~bash
系统版本：openEuler-22.03-LTS-SP1
ceph版本：V14.2.15
GCC版本：10.3.1
内核版本：5.10.0-136.15.0.91.oe2203sp1
Python版本：Python 2.7（手动编译安装）
~~~

#### 2.硬件环境

~~~bash
设备类型	CPU	     内存	  硬盘	        网络	           角色
虚拟机		x86_64   16G	100G+3*8G	192.168.57.16	MON，MGR，OSD
~~~

#### 3.注意事项

1.编译环境和部署环境相同

2.建议系统盘扩容到100G, 防止编译过程中空间不足造成中断。

### 2.编译出包

#### 1.虚拟机搭建

1.搭建虚拟机环境

搭建虚拟机openEuler-22.03-lts-sp1参考以下链接：

~~~bash
https://docs.openeuler.org/zh/docs/22.03_LTS_SP1/docs/Installation/%E5%AE%89%E8%A3%85%E6%8C%87%E5%AF%BC.html
~~~

2.安装python2.7

~~~bash
1.预安装环境
yum update
yum install gcc zlib zlib-devel openssl openssl-devel make git wget rpmdevtools rpm-build 
rpmdev-setuptree
git clone https://gitee.com/src-openeuler/python2.git
cd python2
cp . /root/rpmbuild/SOURCES/
cp ceph.spec /root/rpmbuild/SPECS/
cd /root/rpmbuild/SPECS/
yum-builddep python2.spec
QA_RPATHS=0x0001 rpmbuild -ba python2.spec

2.安装
mkdir python2_rpm
cp /root/rpmbuild/RPMS/x86_64/* /root/python2_rpm/
cp /root/rpmbuild/RPMS/noarch/* /root/python2_rpm/
rpm -ivh /root/python2_rpm/*.rpm --nodeps
下载setuptool
wget https://pypi.python.org/packages/41/5f/6da80400340fd48ba4ae1c673be4dc3821ac06cd9821ea60f9c7d32a009f/setuptools-38.4.0.zip#md5=3426bbf31662b4067dc79edc0fa21a2e
下载pip
wget https://pypi.python.org/packages/11/b6/abcb525026a4be042b486df43905d6893fb04f05aac21c32c638e939e447/pip-9.0.1.tar.gz#md5=35f01da33009719497f01a4ba69d63c9

3.补充
	1.安装setuptools
	cd setuptools的解压目录
	python setup.py install

	2.安装pip
	cd pip的解压目录
	python setup.py install
	
	3.pip设置镜像源
	pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
	pip config set install.trusted-host mirrors.aliyun.com
~~~

#### 3.下载源码包和修改配置

1.下载源码包并解压

~~~bash
# cd /root/
# wget https://download.ceph.com/tarballs/ceph-14.2.15.tar.gz
# tar -zxvf /root/ceph-14.2.15.tar.gz
# mv /root/ceph-14.2.15.tar.gz /root/rpmbuild/SOURCES/ceph-14.2.15.tar.bz2
# cp /root/ceph-14.2.15/ceph.spec /root/rpmbuild/SPECS/
~~~

#### 4.编译源码包

+ 安装编译依赖

  ~~~bash
  pip install sphinx cpython pyOpenSSL prettytable pecan 
  
  yum install cmake cryptsetup fuse-devel gcc-c++ gperf leveldb-devel libaio-devel libblkid-devel libcap-ng-devel libcurl-devel libnl3-devel liboath-devel libtool libudev-devel libxml2-devel ncurses-devel snappy-devel valgrind-devel xfsprogs-devel xmlstarlet yasm  rdma-core-devel lz4-devel openldap-devel nss-devel bison npm re2-devel socat yaml-cpp-devel libbabeltrace-devel lttng-ust-devel librabbitmq-devel CUnit-devel checkpolicy gperftools-devel librdkafka-devel openeuler-lsb selinux-policy-devel
  ~~~

+ 编译

  ~~~bash
  cd /root/rpmbuild/SPECS/
  yum-builddep ceph.spec  
  rpmbuild -ba ceph.spec  
  ~~~

#### 5.编译问题解决

~~~bash
1.报错ceph-15.2.15.tar.bz2 not exist.
解决方法：
vim /root/ceph.spec +117
source0的后缀改为ceph-15.2.15.tar.gz

2.报错Found Cython3: 0.29
CMake Error at cmake/modules/FindCython.cmake:13 (message):
Could not find cython: /usr/bin/python2.7: No module named cython
手动安装cpython
wget https://files.pythonhosted.org/packages/dc/f6/e8e302f9942cbebede88b1a0c33d0be3a738c3ac37abae87254d58ffc51c/Cython-0.29.33.tar.gz
pip install Cython-0.29.33.tar.gz

3.报错信息：
Traceback (most recent call last):
  File "/usr/local/bin/sphinx-build", line 5, in <module>
    from sphinx.cmd.build import main
ModuleNotFoundError: No module named 'sphinx'
make[2]: *** [doc/man/CMakeFiles/manpages.dir/build.make:148: doc/man/ceph-syn.8] Error 1
make[1]: *** [CMakeFiles/Makefile2:19261: doc/man/CMakeFiles/manpages.dir/all] Error 2

vim /usr/local/bin/sphinx-build
/usr/bin/python3改为/usr/bin/python2

4.报错/boost/thread/pthread/thread_data.hpp:60:5: error: missing binary operator before token "(".
解决方法：
vim /root/rpmbuild/BUILD/ceph-15.2.15/src/boost/boost/thread/pthread/thread_data.hpp +60
删除代码
# if PTREAD_STACK_MIN > 0
#   if (size<PTREAD_STACK_MIN) size=PTREAD_STACK_MIN
# fi

5.报错/root/rpmbuild/BUILD/ceph-15.2.15/src/compressor/snappy/SnappyCompressor.h:99:13： error: 'uint32' is not member of 'snappy'.
参考：
https://tracker.ceph.com/issues/50934
https://github.com/ceph/ceph/pull/42507
解决方法：
vim /root/rpmbuild/BUILD/ceph-15.2.15/src/compressor/snappy/SnappyCompressor.h +100
-    snappy::uint32 res_len = 0;
+    uint32_t res_len = 0;

6.报错/root/rpmbuild/BUILD/ceph-14.2.15/src/global/signal_handler.h:24:26: error: 'sys_siglist' was not declared in this scope
参考：
https://gitee.com/src-openeuler/ceph/issues/I41F9W
https://lists.yoctoproject.org/g/meta-virtualization/topic/76190251?p=Created,,,20,2,0,0
vim ceph.spec +1207
-DWITH_REENTRANT_STRSIGNAL=ON 


7.报错检查未打包文件：/usr/lib/rpm/check-files /root/rpmbuild/BUILDROOT/ceph-15.2.15-0.x86_64
错误：发现已安装(但未打包的)文件：
   /usr/share/ceph/mgr/__pycache__/mgr_module.cpython-39.opt-1.pyc
   /usr/share/ceph/mgr/__pycache__/mgr_module.cpython-39.pyc
   /usr/share/ceph/mgr/__pycache__/mgr_util.cpython-39.opt-1.pyc
   /usr/share/ceph/mgr/__pycache__/mgr_util.cpython-39.pyc
参考：
https://blog.csdn.net/developerof/article/details/112139268 
修改spec文件的%install下面添加
%define _unpackaged_files_terminate_build 0

vi /usr/lib/rpm/macros
#%__check_files %{_rpmconfigdir}/check-files %{buildroot}

8.报错：/root/rpmbuild/BUILD/ceph-14.2.15/src/test/librgw_file_marker.cc:396:6: error: pasting "LibRGW_" and "(" does not give a valid preprocessing token
参考社区：
https://gitee.com/src-openeuler/ceph/issues/I61X2K
~~~

## 3.部署

### 1.设置节点名称

| IP            | 作用   |
| :------------ | :----- |
| 192.168.57.16 | ceph01 |

~~~bash
 vim /etc/hosts
 新增：
 192.168.57.16 ceph01
~~~

### 2.安装

1.安装ceph

```absh
mkdir /root/ceph
cp /root/rpmbuild/RPMS/x86_64/* /root/ceph/
cp /root/rpmbuild/RPMS/noarch/*.rpm /root/ceph
cd /root/ceph
yum install *.rpm
```

### 3.部署

```bash
参考手动部署：https://xujiyou.work/%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8/Ceph/Ceph%E9%83%A8%E7%BD%B2/ceph%E6%89%8B%E5%8A%A8%E9%83%A8%E7%BD%B2.html
```

### 4.成功截图

![1676599270590](https://gitee.com/Tom_zc/my_doc/raw/master/ceph%E7%BC%96%E8%AF%91/openEuler-2203-LTS-SP1%E9%80%82%E9%85%8Dceph14/assets/1676599270590.png)

### 5.部署问题解决

~~~bash
1.monitors have not enabled msgr2
ceph mon enable-msgr2

2.ceph -s报4 mgr modules have faileld dependencies
less /var/log/ceph/ceph-mgr.ceph-1.log查出缺包日志
pip install cherrypy yaml werkzeug
~~~

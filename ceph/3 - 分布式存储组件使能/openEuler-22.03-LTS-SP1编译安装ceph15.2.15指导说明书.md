# openEuler-22.03-LTS-SP1编译安装ceph-15.2.15

## 1.声明

~~~bash
编写日期：2023年02月08日
编写人员：朱超（tom_toworld@163.com）
修订版本：V1.0
~~~

## 2.编译

### 1.环境信息

#### 1.软件环境

~~~bash
系统版本：openEuler-22.03-LTS-SP1
ceph版本：V15.2.15
GCC版本：10.3.1
内核版本：5.10.0-136.15.0.91.oe2203sp1
Python版本：Python 3.9.9
~~~

#### 2.硬件环境

~~~bash
设备类型	CPU	     内存	  硬盘	        网络	           角色
虚拟机		x86_64   16G	100G+3*8G	192.168.57.15	MON，MGR，OSD
~~~

#### 3.注意事项

1.编译环境和部署环境相同

2.建议系统盘扩容到100G, 防止编译过程中空间不足造成中断。

### 2.编译出包

#### 1.虚拟机搭建

搭建虚拟机openEuler-22.03-lts-sp1参考以下链接：

~~~bash
https://docs.openeuler.org/zh/docs/22.03_LTS_SP1/docs/Installation/%E5%AE%89%E8%A3%85%E6%8C%87%E5%AF%BC.html
~~~

#### 2.构建编译环境

~~~bash
# yum install wget rpmdevtools rpm-build gcc
# rpmdev-setuptree
~~~

#### 3.下载源码包并解压

~~~bash
# cd /root/
# wget https://download.ceph.com/tarballs/ceph-15.2.15.tar.gz
# tar -zxvf /root/ceph-15.2.15.tar.gz
# mv /root/ceph-15.2.15.tar.gz /root/rpmbuild/SOURCES/ceph-15.2.15.tar.bz2
# cp /root/ceph-15.2.15/ceph.spec /root/rpmbuild/SPECS/
~~~

#### 4.编译源码包

+ 安装编译依赖

  ~~~bash
  yum install checkpolicy CUnit-devel expat-devel gperftools-devel keyutils-libs-devel libbabeltrace-devel liboath liboath-devel librabbitmq-devel librdkafka-devel lttng-ust-devel lz4-devel nss-devel openeuler-lsb openldap-devel openssl-devel python3-Cython python3-devel python3-prettytable python3-sphinx rdma-core-devel selinux-policy-devel systemd-devel
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

2.报错/boost/thread/pthread/thread_data.hpp:60:5: error: missing binary operator before token "(".
解决方法：
vim /root/rpmbuild/BUILD/ceph-15.2.15/src/boost/boost/thread/pthread/thread_data.hpp +60
删除代码
# if PTREAD_STACK_MIN > 0
#   if (size<PTREAD_STACK_MIN) size=PTREAD_STACK_MIN
# fi

3.报错/root/rpmbuild/BUILD/eph-15.2.15/src/compressor/snappy/SnappyCompressor.h:99:13： error: 'uint32' is not member of 'snappy'.
参考：
https://tracker.ceph.com/issues/50934
https://github.com/ceph/ceph/pull/42507
解决方法：
vim /root/rpmbuild/BUILD/ceph-15.2.15/src/compressor/snappy/SnappyCompressor.h +100
-    snappy::uint32 res_len = 0;
+    uint32_t res_len = 0;

4.报错检查未打包文件：/usr/lib/rpm/check-files /root/rpmbuild/BUILDROOT/ceph-15.2.15-0.x86_64
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
~~~

## 3.部署

### 1.设置节点名称

| IP            | 作用   |
| :------------ | :----- |
| 192.168.57.15 | ceph01 |

~~~bash
 vim /etc/hosts
 新增：
 192.168.57.15 ceph01
~~~

### 2.安装

1.安装ceph

```absh
mkdir /root/ceph
cp /root/rpmbuild/RPMS/x86/*.rpm /root/ceph
cp /root/rpmbuild/RPMS/noarch/*.rpm /root/ceph
cd /root/ceph
yum install *.rpm
```

2.安装ceph-deploy

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

```bash
1.部署集群
cd /etc/ceph/
ceph-deploy new ceph01
ceph-deploy --overwrite-conf mon create-initial
ceph-deploy mgr create ceph01

2.部署mgr
ceph mgr module enable dashboard 
ceph dashboard create-self-signed-cert
ceph dashboard set-login-credentials admin abc@123

3.部署osd
ceph-deploy disk zap ceph01 /dev/sdd
ceph-deploy osd create ceph01 --data /dev/sdb
```

4.成功截图

![成功截图](https://gitee.com/Tom_zc/my_doc/raw/master/ceph%E7%BC%96%E8%AF%91/openEuler-2203-LTS-SP1%E9%80%82%E9%85%8Dceph15/assets/1675475835775.png)

![dashboard大屏](https://gitee.com/Tom_zc/my_doc/raw/master/ceph%E7%BC%96%E8%AF%91/openEuler-2203-LTS-SP1%E9%80%82%E9%85%8Dceph15/assets/1675475978796.png)

### 4.部署问题解决

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


















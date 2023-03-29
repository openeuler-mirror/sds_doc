# openEuler20.03-LTS-SP3 ceph16.2.7编译出包指导
## 环境信息
OS版本：openEuler20.03-LTS-SP3  
ceph版本：v16.2.7  
GCC版本：10.3.1  
kernel： 4.19.90-2112.8.0.0131.oe1.aarch64  
## 编译出包
### 编译GCC相关软件包
#### 下载源码及补丁
git clone -b openEuler-22.03-LTS https://gitee.com/src-openeuler/gcc.git  
#### 安装相关依赖
cd gcc  
yum-builddep gcc.spec  
#### 安装rpmbuild包，复制源码到相关目录
yum install -y rpmdevtools  
rpmdev-setuptree  
cp *  /root/rpmbuild/SOURCES  
#### 编译出包
rpmbuild -bb gcc.spec  
#### 可能出现的问题
cpio:  obj-aarch64-linux-gnu/gcc/cfns.gperf: Cannot stat: No such file or directory  
.........................  
ERROR: 0020: file '/usr/lib64/liblsan-so.0.0.0' contain an rpath referencing ".." of an absolute path [/usr/lib/../lib64]  
........................  
解决办法： vim ~/.rpmmacros 注释掉以下两行  
     #[ "%{buildarch}" = "noarch" ] || QA_CHECK_RPATHS=1 ; \  
     #case "${QA_CHECK_RPATHS:-}" in [1yY]*) /usr/lib/rpm/check-rpaths ;; esac \  
### 编译luarocks软件包
git clone -b openEuler-20.03-LTS-SP3 https://gitee.com/wangzengliang1/luarocks.git  
#### 安装相关依赖
cd luarocks  
yum-builddep luarocks.spec  
#### 复制源码到相关目录
cp *  /root/rpmbuild/SOURCES  
#### 编译出包
rpmbuild -bb luarocks.spec  
### 将编译好的rpm包作为本地软件源
mkdir -p /home/rpm/gcc  
cp -r /root/rpmbuild/RPMS/*  /home/rpm/gcc/  
yum install -y createrepo  
cd /home/rpm/gcc/ & createrepo .  
### 配置repo文件
vim /etc/yum.repos.d/local.repo
[local-gcc]  
name=local-gcc  
baseurl=file:///home/rpm/gcc/  
gpgcheck=0  
清理yum 缓存  
yum  clean all & yum makecache  
### 编译ceph
####下载ceph源码
git clone -b openEuler-22.03-LTS https://gitee.com/src-openeuler/ceph.git  
#### 复制源码到相关目录
cp *  /root/rpmbuild/SOURCES  
#### 安装依赖包
yum-builddep ceph.spec  
#### 编译出包
rpmbuild -bb ceph.spec  
## 安装ceph
### 将编译好的rpm包作为本地软件源
mkdir -p /home/rpm/ceph  
cp -r /root/rpmbuild/RPMS/*  /home/rpm/ceph/  
yum install -y createrepo  
cd /home/rpm/ceph/ & createrepo .  
### 配置repo文件
vim /etc/yum.repos.d/local-ceph.repo  
[local-ceph]  
name=local-ceph  
baseurl=file:///home/rpm/ceph/  
gpgcheck=0  
清理yum 缓存
yum  clean all & yum makecache  
### 安装ceph软件
yum install -y ceph  
执行ceph -v 查看是否安装成功  
## 部署ceph集群
### 安装ceph-deploy  
pip  install ceph-deploy  
修改ceph-deploy代码，使其兼容openEuler  
vim /usr/lib/python2.7/site-packages/ceph_deploy/hosts/__init__.py   
添加如下行(行号为100)  
92     distributions = {  
93         'debian': debian,  
94         'ubuntu': debian,  
95         'centos': centos,  
96         'scientific': centos,  
97         'oracle': centos,  
98         'redhat': centos,  
99         'fedora': fedora,   
100         'openeuler': fedora,  
101         'suse': suse,  
102         'virtuozzo': centos,  
103         'arch': arch  
104         }
### 部署ceph
参考https://docs.ceph.com/projects/ceph-deploy/en/latest 文档进行部署  
## 可能出现的问题
### 编译过程中python错误
解决办法：  
修改python命令指向   
rm -rf /usr/bin/python  
ln -s /usr/bin/python3 /usr/bin/python  
### 编译过程失败 error: Istalled (but unpackaged) file(s) found:  
                      /usr/share/ceph/mgr/__pycache__/mgr.module.cpython-37.opt-1.pyc  
		      .........................  
解决办法： vim /usr/lib/rpm/macros  
注释掉%__check_files /usr/lib/rpm/check-files %{buildroot} 这一行  
### 编译过程依赖未安装完全  
#### Could NOT find verbs (missing: VERBS_LIBRARIES VERBS_INCLUDE_DIR)
解决办法：  
yum install rdma-core-devel  
类似问题根据提示安装对用的软件包即可  

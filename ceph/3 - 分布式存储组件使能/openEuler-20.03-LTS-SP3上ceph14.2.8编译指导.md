# openEuler20.03-LTS-SP3 ceph14.2.8编译出包指导
## 环境信息
OS版本：openEuler20.03-LTS-SP3  
ceph版本：v14.2.8  
GCC版本：7.3.0  
kernel： 4.19.90-2112.8.0.0131.oe1.aarch64  
## 编译出包
### 下载ceph源码包  
wget http://download.ceph.com/tarballs/ceph-14.2.8.tar.gz  
### 解压源码
tar -zxvf ceph-14.2.8.tar.gz  
cd ceph-14.2.8  
### 修改ceph.spec，使其兼容openEuler
sed -i "s@%if 0%{?rhel} || 0%{?fedora}@%if 0%{?rhel} || 0%{?fedora} || 0%{?openEuler}@g" ceph.spec  
sed -i "s@%if 0%{?fedora} || 0%{?rhel}@%if 0%{?fedora} || 0%{?rhel} || 0%{?openEuler}@g" ceph.spec  
sed -i "s@redhat-lsb-core@openeuler-lsb@g" ceph.spec   
sed -i "s@redhat-rpm-config@openEuler-rpm-config@g" ceph.spec  
### 安装依赖包
yum-builddep ceph.spec  
### 编译出包
yum install -y rpmdevtools  
rpmdev-setuptree  
cp ../ceph-14.2.8.tar.gz /root/rpmbuild/SOURCES  
rpmbuild -bb ceph.spec  
## 安装ceph
### 将编译好的rpm包作为本地软件源
mkdir -p /home/rpm/ceph  
cp -r /root/rpmbuild/RPMS/*  /home/rpm/ceph/  
yum install -y createrepo  
cd /home/rpm/ceph/ & createrepo .  
### 配置repo文件
vim /etc/yum.repos.d/local.repo  
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
### 编译过程依赖未安装完全
#### Could NOT find verbs (missing: VERBS_LIBRARIES VERBS_INCLUDE_DIR)
解决办法：  
yum install rdma-core-devel   
类似问题根据提示安装对用的软件包即可   
#### java安装失败 
解决办法：  
重新执行yum install -y java即可   
#### ceph -s 出现 no module named werzeug.serving
解决办法  
pip install werkzeug  
#### ceph -v 出现 no module named prettytable
解决办法  
pip install  prettytable  

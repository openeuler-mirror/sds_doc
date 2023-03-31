# ceph汉化指导说明书

## 一、声明

~~~bash
编写日期：2023年02月08日
编写人员：朱超（tom_toworld@163.com）
修订版本：V1.0
~~~

## 二、ceph汉化流程

### 1.构建ubuntu-2204环境

### 2.安装必要软件

~~~bash
apt install wget npm ng-common
~~~

### 3.下载源码并编译

~~~bash
wget https://download.ceph.com/tarballs/ceph-15.2.15.tar.gz
tar -zxvf ceph-15.2.15.tar.gz
cd ceph-15.2.15/src/pybind/mgr/dashboard/frontend
npm installl --force
npm run build:zh-CN
tar -zcvf dist.tar.gz dist
~~~

### 4.安装

1.将dist.tar.gz拷贝到部署mgr的节点

2.解压dist.tar.gz， 将里面的内容拷贝到/usr/share/ceph/mgr/dashboard/frontend/dist/, 并重启ceph-mgr服务。

![位置截图](https://gitee.com/Tom_zc/my_doc/raw/master/ceph汉化/assets/1675831729432.png)

3.登录到ceph的dashboard界面

![dashboard截图](https://gitee.com/Tom_zc/my_doc/raw/master/ceph汉化/assets/1675831833885.png)
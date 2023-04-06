# **openEuler-22.03-LTS-SP1部署ceph16说明书**

## **一、简介**

记录基于openEuler 22.03-lts-sp1部署ceph16版本。

![1680147923398](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy//assets/1680147923398.png)

## **二、基础环境简介**

1.环境要求

| os                      | hostname | ip                                                           | ssd                | roles                |
| ----------------------- | -------- | :----------------------------------------------------------- | ------------------ | -------------------- |
| openEuler 22.03-lts-sp1 | node20   | 管理网ip: 192.168.57.20, 业务网: 192.168.57.20，存储前端网: 10.0.10.20，存储后端网：10.0.4.20 | 系统盘500G+1个500G | mon,mgr,,osd,mds,rgw |
| openEuler 22.03-lts-sp1 | node21   | 管理网ip: 192.168.57.21, 业务网: 192.168.57.21，存储前端网: 10.0.10.21，存储后端网：10.0.4.21 | 系统盘500G+1个500G | mon,mgr,,osd,mds,rgw |
| openEuler 22.03-lts-sp1 | node22   | 管理网ip: 192.168.57.22, 业务网: 192.168.57.22，存储前端网: 10.0.10.22，存储后端网：10.0.4.22 | 系统盘500G+1个500G | mon,mgr,,osd,mds,rgw |

2.网络拓扑图

![1678885133496](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/1678885133496.png)

3.硬件要求

​	每个节点至少2U4G, 至少配置3张网卡

## **三、基础环境准备**

1.设置主机名

~~~bash
node20执行:
$ hostnamectl set-hostname node20 && exec /bin/bash
node21执行:
$ hostnamectl set-hostname node21 && exec /bin/bash
node22执行:
$ hostnamectl set-hostname node22 && exec /bin/bash
~~~

2.配置ip解析,  在三个节点都执行：

~~~bash
$ vim /etc/hosts
10.0.10.20 node20
10.0.10.21 node21
10.0.10.22 node22
~~~

3.免密登录，在node20节点上配置免密登录到node21,node22；在node20执行：

~~~bash
$ ssh-keygen -t rsa -P ''
$ ssh-copy-id node20
$ ssh-copy-id node21
$ ssh-copy-id node22
~~~

4.打开ceph端口或者直接关闭端口服务，两者选其一，在三个节点都执行：

+ 打开常用端口

  ~~~bash
  $ firewall-cmd --zone=public --add-port=6789/tcp --permanent
  $ firewall-cmd --zone=public --add-port=6800-7300/tcp --permanent
  $ firewall-cmd --reload
  $ firewall-cmd --zone=public --list-all
  ~~~

+ 关闭端口服务

  ~~~bash
  $ systemctl stop firewalld
  $ systemctl disable firewalld
  ~~~

5.禁用selinux策略，在三个节点都执行：

~~~bash
# 临时关闭，不用重启机器
$ setenforce 0
# 永久修改，修改/etc/selinux/config 文件
将SELINUX=enforcing改为SELINUX=disabled
~~~

6.安装配置ntp服务（单节点可以不安装），在三个节点都执行

~~~bash
Ceph要求服务器时间统一，否则无法正常启动
$ yum -y install ntp ntpdate
$ ntpdate 0.us.pool.ntp.org
$ hwclock --systohc
# 在node20启动ntpd服务
$ systemctl enable ntpd
$ systemctl start ntpd
~~~

7.内核参数优化，在三个节点都执行

~~~bash
1.设置文件描述符
$ echo "ulimit -SHn 102400" >> /etc/rc.local
$ cat >> /etc/security/limits.conf << EOF
* soft nofile 65535
* hard nofile 65535
EOF

2.内核参数优化
$ cat >> /etc/sysctl.conf << EOF
kernel.pid_max = 4194303
vm.swappiness = 0 
EOF
$ sysctl -p

3.设置IO策略，I/O Scheduler，SSD要用noop，SATA/SAS使用deadline
$ echo "deadline" >/sys/block/sd[x]/queue/scheduler
$ echo "noop" >/sys/block/sd[x]/queue/scheduler

4.read_ahead,通过数据预读并且记载到随机访问内存方式提高磁盘读操作
$ echo "8192" > /sys/block/sda/queue/read_ahead_kb
~~~

## 四、手动部署ceph

### 1.安装软件

1.在各个节点上执行

~~~bash
$ yum install ceph -y
~~~

2.部署图

![1680164951021](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/1680164832428)

### 2.部署ceph-mon

部署：三个节点分别部署ceph-mon

1.生成cephfs id, 在node10上执行：

~~~bash
$ uuidgen
b3a56e00-c60b-4b2d-8bba-a240dc1d280b
~~~

2.创建文件和配置

在node20上执行

~~~bash
mkdir /etc/ceph
vim /etc/ceph/ceph.conf
~~~

配置如下：

~~~bash
[global]
fsid = b3a56e00-c60b-4b2d-8bba-a240dc1d280b
mon_initial_members = node20, node21, node22
mon_host = 10.0.10.20:6789, 10.0.10.21:6789, 10.0.10.22:6789
public_network = 10.0.10.0/24
cluster_network = 10.0.4.0/24
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
mon_allow_pool_delete = True
osd_crush_update_on_start = False
mon_config_key_max_entry_size = 16777216
rados_mon_op_timeout = 60
rados_osd_op_timeout = 60
rbd_cache = false
osd_pool_default_size = 3
osd_pool_default_min_size = 1
mon_clock_drift_allowed = 2
mon_clock_drift_warn_backoff = 30
mon_osd_full_ratio = 0.850000
mon_osd_backfillfull_ratio = 0.850000
mon_osd_nearfull_ratio = 0.800000
mon_warn_on_too_few_osds = False
mon_warn_on_msgr2_not_enabled = False
ms_bind_msgr2 = False
client_mount_timeout = 45
mon_pg_warn_max_object_skew = 0
rgw_crypt_require_ssl = False
rgw_lifecycle_work_time = 00:00-06:00
rgw_swift_versioning_enabled = True
rgw_swift_account_in_url = True
rgw_enable_usage_log = True
mon_osd_report_timeout = 60
osd_beacon_report_interval = 25
rgw_relaxed_s3_bucket_names = True
osd_pool_default_pg_autoscale_mode = off
mds_max_snaps_per_dir = 4096
[client]
rbd_blacklist_on_break_lock = true
rbd_blacklist_expire_seconds = 30
rbd_blocklist_expire_seconds = 30
rbd_cache = false
admin_socket = $run_dir/$cluster-$name.$pid.$cctid.asok
client_oc = false
client_reconnect_stale = true
rgw_thread_pool_size = 64
[mgr]
rados_osd_op_timeout = 20
rbd_blacklist_expire_seconds = 30
rbd_blocklist_expire_seconds = 30
[mds]
mds_action_on_write_error = 2
mds_buffer_anno_to_cache_limit = 10
mds_buffer_anno_max = 16106127360
[osd]
osd_scrub_auto_repair = True
network_check_interval = 10
max_network_error_cnt = 6   
~~~

3.在node20节点上创建monitor

~~~bash
# 创建admin用户和key
$ ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'

$ ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

$ ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

$ ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring

$ ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring

$ chown ceph:ceph /tmp/ceph.mon.keyring

# 创建monitor map
$ monmaptool --create --add node20 10.0.10.20 --add node21 10.0.10.21 --add node22 10.0.10.22 --fsid  b3a56e00-c60b-4b2d-8bba-a240dc1d280b /tmp/monmap

$ monmaptool --print /tmp/monmap
~~~

4.拷贝文件到其他节点node21, node22

~~~bash
# 拷贝到node21
$ scp /etc/ceph/ceph.conf root@node21:/etc/ceph/
$ scp /tmp/ceph.mon.keyring root@node21:/tmp/
$ scp /etc/ceph/ceph.client.admin.keyring root@node21:/etc/ceph/
$ scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@node21:/var/lib/ceph/bootstrap-osd/
$ scp /tmp/monmap root@node21:/tmp/

# 拷贝到node22
$ scp /etc/ceph/ceph.conf root@node22:/etc/ceph/
$ scp /tmp/ceph.mon.keyring root@node22:/tmp/
$ scp /etc/ceph/ceph.client.admin.keyring root@node22:/etc/ceph/
$ scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@node22:/var/lib/ceph/bootstrap-osd/
$ scp /tmp/monmap root@node22:/tmp/
~~~

5.在每个节点上初始化mon目录

~~~bash
# 在node20上执行
$ sudo -u ceph ceph-mon --mkfs -i node20 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
$ chown -R ceph:ceph /etc/ceph
$ chown -R ceph:ceph /var/lib/ceph/mon/ceph-node20/  

# 在node21上执行
$ chown ceph:ceph /tmp/ceph.mon.keyring
$ sudo -u ceph ceph-mon --mkfs -i node21 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
$ chown -R ceph:ceph /etc/ceph
$ chown -R ceph:ceph /var/lib/ceph/mon/ceph-node21/

# 在node22上执行
$ chown ceph:ceph /tmp/ceph.mon.keyring
$ sudo -u ceph ceph-mon --mkfs -i node22 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
$ chown -R ceph:ceph /etc/ceph
$ chown -R ceph:ceph /var/lib/ceph/mon/ceph-node22/
~~~

6.启动mon服务

~~~bash
# node20节点执行:
$ systemctl start ceph-mon@node20
$ systemctl enable ceph-mon@node20

# node21节点执行:
$ systemctl start ceph-mon@node21
$ systemctl enable ceph-mon@node21

# node22节点执行:
$ systemctl start ceph-mon@node22
$ systemctl enable ceph-mon@node22
~~~

7.成功截图如下：

![1680163436645](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/1680163436645.png)

8.FAQ

~~~bash
1.报mons are allowing insecure global_id reclaim
ceph config set mon auth_allow_insecure_global_id_reclaim false

2.报clock skew detected on mon.node21, mon.node22
ceph tell mon.* injectargs "--mon_clock_drift_allowed=1"
~~~



### 3.部署ceph-mgr

部署：node20、node21部署ceph-mon, 下列流程1-3步为在node20节点执行步骤，node21可参考于此：

1.创建目录

~~~bash
$ sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-node20
~~~

2.创建秘钥

~~~bash
$ ceph auth get-or-create mgr.node20 mon 'allow profile mgr' osd 'allow *' mds 'allow *' > /var/lib/ceph/mgr/ceph-node20/keyring
~~~

3.启动ceph-mgr服务

~~~bash
$ systemctl enable ceph-mgr@node20
$ systemctl start ceph-mgr@node20
~~~

4.安装dashboard

~~~bash
$ yum install ceph-mgr-dashboard 
$ ceph mgr module enable dashboard
$ ceph dashboard create-self-signed-cert
$ echo "abc@123" > passwd
$ ceph dashboard set-login-credentials admin -i passwd
~~~



### 4.部署ceph-osd

部署：在node20、node21、node22上各部署一个osd.

```bash
# 在node20上创建osd1
$ sudo ceph-volume lvm create --data /dev/sdb

# 在node21上创建osd2
$ sudo ceph-volume lvm create --data /dev/sdc

# 在node22上创建osd3
$ sudo ceph-volume lvm create --data /dev/sdd
```

![1680164344921](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/1680164344921.png)



## 五、资源创建

### 1.创建设备组

![1677924722378](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/拓扑图.jpg)

#### 1.crush rule引入设备组

~~~bash
# 读取crush rule
$ ceph osd getcrushmap -o ./crushmap.bin
$ crushtool -d crushmap.bin -o ./crushmap.txt

# 修改crush rule
$ vim crushmap.txt
在type 11 root下面添加: type 12 devicegroup

# 导入crush rule
$ crushtool -c crushmap.txt -o crushmap-new.bin
$ ceph osd setcrushmap -i crushmap-new.bin

# 查看crush rule
ceph osd crush dump
~~~

修改crush rule的配置图如下：

![1678708464204](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/1678708464204.png)

#### 2.设备组的增删查改

~~~bash
# 创建设备组
$ ceph osd crush add-bucket $device_group_name $bucket-type
$ ceph osd crush add-bucket dg1 devicegroup

# 查看设备组
$ ceph osd tree

# 删除设备组
$ ceph osd crush remove dg1
~~~

#### 3.机架的增删查改

~~~bash
# 创建机架：
$ ceph osd crush add-bucket $bucket-name rack
$ ceph osd crush add-bucket rack1 rack

# 机架绑定到设备组:
$ ceph osd crush link bucket-name devicegroup=dg
$ ceph osd crush link rack1 devicegroup=dg1

# 创建host
$ ceph osd crush add-bucket node20 host
$ ceph osd crush add-bucket node21 host
$ ceph osd crush add-bucket node22 host

# 将host绑定到机架上（相当于替换工作）：
$ ceph osd crush move $bucket-name rack=$bucket-name
$ ceph osd crush move node20 rack=rack1
$ ceph osd crush move node21 rack=rack1
$ ceph osd crush move node22 rack=rack1

# 将osd绑定到对应的host
$ ceph osd crush move osd.0 host=node20
$ ceph osd crush move osd.1 rack=node21
$ ceph osd crush move osd.2 rack=node22

# 移除机架：
$ ceph osd crush remove $bucket-name
$ ceph osd crush remove rack-test

# 查看osd的拓扑图
$ ceph osd tree --format json
~~~

查看osd的拓扑图

![1680165562975](https://gitee.com/Tom_zc/my_doc/raw/master/ceph_doc/ceph_deploy/assets/1680165562975.png)

### 2.创建存储池

1.创建副本存储池

~~~bash
# 创建crush rule 
$ ceph osd crush rule create-replicated $rule_name $device_name $type(host/rack/osd)
$ ceph osd crush rule create-replicated p1_rule dg1 rack # 故障域为rack，代表必须副本数量的rack

# 查看存储池的crush rule 
$ ceph osd crush rule ls

# 创建存储池， # 注意pg, pgp_num的计算参考：https://old.ceph.com/pgcalc/
$ ceph osd pool create $pool_name $pg_num $pgp_num $pool_type(replicated/erasure) $rule_name
$ ceph osd pool create p1 64 64 replicated p1_rule

# 设置存储池的副本
$ ceph osd pool set $pool_name size 3
$ ceph osd pool set p1 size 3

# 设置存储池最小副本数
$ ceph osd pool set $pool_name min_size 1
$ ceph osd pool set p1 min_size 1

# 查看存储池
$ ceph osd pool ls

# 如果遇到ceph系统自动创建的存储池报pg unknowns?比如device_health_metrics
ceph osd crush rule create-replicated p3_rule dg1 osd
ceph osd pool set device_health_metrics crush_rule p3_rule
~~~

2.创建纠删码存储池

~~~bash
$ ceph osd erasure-code-profile ls

$ ceph osd erasure-code-profile set $erasure_code_profile k=3 m=2 crush-failure-domain=rack/host/osd crush-root=$devicegroup_name
$ ceph osd erasure-code-profile set erasure1 k=2 m=1 crush-failure-domain=rack crush-root=dg1

$ ceph osd crush rule create-erasure $rule_name $erasure_code_profile
$ ceph osd crush rule create-erasure rule1 erasure1

$ ceph osd pool create $pool_name $pg_num $pgp_num $pool_type(replicated/erasure) $erasure_profile $rule_name
$ ceph osd pool create p2 16 16 erasure erasure1 rule1
~~~

3.设备副本数量

~~~bash
$ ceph osd pool set $pool_name size 3
~~~

4.设置最小副本数

~~~bash
$ ceph osd pool set $pool_name min_size 1
~~~

5.设置纠删码可以被rbd,cephfs使用

~~~bash
$ ceph osd pool set $pool_name allow_ec_overwrites true
~~~

6.设置存储池限额

~~~bash
$ ceph osd pool set-quota $pool_name max_bytes 1000000
~~~

7.创建应用

~~~bash
$ ceph osd pool application enable {pool-name} {application-name}
$ ceph osd pool application enable p1 rbd
$ ceph osd pool application enable p1 cephfs 
$ ceph osd pool application enable p1 rgw
~~~

8.查看存储池

~~~bash
$ ceph osd pool ls
~~~

9.删除存储池

~~~bash
$ ceph osd pool delete $pool_name $pool_name --yes-i-really-really-mean-it
$ ceph osd pool delete p1 p1 --yes-i-really-really-mean-it

# 如果删除失败，报需要设置参数
ceph --admin-daemon /var/run/ceph/ceph-mon.node20.asok config show|grep delete
ceph --admin-daemon /var/run/ceph/ceph-mon.node20.asok config set mon_allow_pool_delete true
~~~


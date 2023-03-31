# ceph/qa套件介绍
  Ceph 有两种类型的测试：make check测试和集成测试。当测试需要多台机器、root 访问权限或持续时间较长（例如，模拟真实的 Ceph 工作负载）时，则认为是集成测试。集成测试被组织成“套件”，这些套件在ceph/qa 子目录中定义并使用 teuthology-suite命令运行。
# ceph-qa用例介绍
| 模块 | 介绍 | 描述 | 用例个数 | 预计测试用例 | 排除条件 | 
|  -- | -- | -- | -- | -- | -- |
| big | 集群测试5/21/51节点测试 | 基本的object读写snapg功能测试 | 27 | 0 | 大规模条件不满足 | 
| buildpackages | ceph 版本检查 | 共计51中当前支持的OS  版本检查 | 102 | 0 | 没必要跑 | 
| ceph-ansible | ceph-ansible工程部署ceph环境测试 | 使用ceph-ansible部署环境测试（ubuntu、centos） | 36 | 0 | 目前不支持openEuler（需要适配） | 
| ceph-deploy | ceph-deploy工程部署ceph环境测试 | 使用ceph-ansible部署环境测试（ubuntu、centos）（py2/py3） | 32 | 0 | 目前不支持openEuler | 
| cephmetrics | 部署cephmetrics并运行集成测试 | 部署cephmetrics并运行集成测试（dashboard） | 32 | 0 | 非主要功能，暂不测试 | 
| crimson-rados | ceph  crimson rados测试 | 部署crimson-rados并运行集成测试 （） | 8 | 0 | 开发中，暂不测试 | 
| dummy | ceph部署测试 | 部署mon mds mgr osd client | 1 | 1 | 基本功能，以下用例会包含 | 
| experimental | ceph experimental测试 | 实验特性测试 | 1 | 0 |  | 
| fs | 文件系统功能测试 | 文件系统相关功能、接口测试 | 5037 | 50 | 1、排除valgrind及upgrade用例 | 
| hadoop | 相关测试 |  | 3 | 0 | 1、依赖mvn源网络不可达 | 
| krbd | kernel rbd测试 |  | 220 | 20 |  | 
| marginal |  |  | 60 | 0 | 边缘用例，暂不测试 | 
| mixed-clients | kclient测试 |  | 12 | 2 | 排除blogbench测试用例 | 
| netsplit | 网络断开重连测试 |  | 1 | 1 | 指定ubuntu用例（待适配） | 
| orch | cephadm 和rook测试 |  | 17633 | 0 | cephadm依赖docker，rook指定ubuntu(待适配) | 
| perf-basic | 基本性能测试（fio ,cosbench,radosbench） |  | 4 | 4 |  | 
| powercycle | 重启测试 |  | 278 | 15 | 1、排除掉部分依赖ipmitool命令操作 | 
| rados | rados基本功能测试 | rados相关功能、借口测试 | 1473815 | 80 | 1、排除valgrind及upgrade用例（可以进一步精简用例，随机跑一些用例） | 
| rbd | rbd基本功能测试 | 块存储相关功能、借口测试 | 10751 | 120 | 1、排除qemu相关用例（虚拟机里起虚拟机） | 
| rgw | rgw基本功能测试 | 对象相关功能、借口测试 | 116 | 30 | 1、依赖mvn源网络不可达 | 
| samba | cifs文件系统测试 |  | 108 | 0 | 1、环境不支持 | 
| smoke | 总体冒烟测试 |  | 24 | 23 | 1、排除blogbench用例 | 
| stress | 压力测试 |  | 38 | 0 | 1、虚拟机环境不支持 | 
| teuthology |  |  | 26 | 0 |  | 
| tgt | iscsi测试 |  | 26 | 1 |  | 
| upgrade | 升级测试 |  | 175 | 0 |  | 
| 总计 |  |  | 1508566 | 348 |  | 
|  |  |  |  |  |  | 
|  |  |  |  |  |  | 
|  |  |  |  | PR： | 跑smoke + 具体模块 | 
|  |  |  |  | 版本： | 跑全量（用例为筛选后的随机选出用例数量） | 
# 用例排除规则
1、排除部分过时的测试工具（例如blogbench）
2、排除其它OS相关的用例（例如ubuntu）
3、排除其它非主要功能用例（例如 dashboard）
4、排除其它条件不满足用例（例如Ipmitool）
# 用例挑选规则
1、smoke模块用例全部执行
2、在排除后剩下的用例中使用随机挑选一定数量的用例
3、有pr合入时执行smoke用例，以及一定数量的pr相关模块的用例

# [openEuler-22.03LTS qa用例结果](https://gitee.com/src-openeuler/ceph/issues/I4ZSWW)
http://119.3.219.112:23002/
![输入图片说明](https://images.gitee.com/uploads/images/2022/0328/090058_53029e48_8819970.png "在这里输入图片标题")
http://119.3.219.112:23002/teuthworker-2022-03-23_11:59:24-netsplit-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:59:17-perf-basic-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:58:48-krbd-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:58:33-rgw-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:46:10-rados-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:44:53-fs-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:41:46-rbd-master-distro-basic-euler/
http://119.3.219.112:23002/teuthworker-2022-03-23_11:41:14-smoke-master-distro-basic-euler/
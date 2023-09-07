# 背景

过去几年，由于互联网+和企业数字化转型的驱动，数据中心已进入全闪存时代，存储高性能吞吐与SCSI协议传输低性能吞吐之间的矛盾日益严重，从而出现了NVMe存储协议，实现了整个存储性能10倍甚至百倍以上的提升。

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/1.png)

随着介质性能的快速提升，网络协议反而成为了数据中心的新性能瓶颈，业务开始呼唤更高质量的网络， NVMe over Fabric（简称NoF）应运而生。
NVMe over Fabrics定义了一个通用架构，支持在不同的传输网络上访问NVMe块存储设备。通过NVMe-oF前端接口，实现规模组网扩展，在数据中心内远距离访问NVMe设备和NVM subsystem。NVMe over Fabrics支持多种传输协议，包括FC和RDMA，TCP等网络。如下图所示：

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/2.png)

其中，基于以太网的RoCE代表了下一代存储网络的开放、创新特性，比FC性能更高、时延更低，同时兼容IP网络的优势，还可以实现数据中心全以太化、全IP化，因此NVMe over RoCE是NoF的最优承载网络方案，也已成为业界NoF的主流技术方向，各主流存储厂商和主要的虚拟化厂商Vmware也在都支持NVMe over RoCE技术。
|   |NetAPP   |Pure Storage   |IBM   |HPE   |EMC   |HDS   |Vmware   |
| ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ |
|NVMe over RoCE   |支持   |支持   |支持   |支持   |不支持   |不支持   |支持   |
|NVMe over TCP   |支持   |支持   |不支持   |支持   |支持   |不支持   |支持   |
|NVMe over FC   |支持   |支持   |支持   |支持   |支持   | 支持  |支持   |

# 存在问题

NVMe over RoCE技术出现后，在性能方面轻松超越了传统SCSI FC技术，但是由于是基于RoCE以太网实现的传输协议，相对SCSI FC天生有一些短板。比如FC内在协议机制支持RSCN快速感知通告，可实现1秒内故障感知；而NVMe over RoCE方案在存储链路故障时，依赖应用层超时发现后切换备链路，故障切换长达数秒到数十秒。FC网络支持设备即插即用，设备上线后自动设备注册和网络通告，快速感知设备接入与移除，而NVMe over RoCE方案依赖IP被动配置，无法感知接入设备状态。FC网络在感知到设备接入后，自动完成业务的链接建立和磁盘设备注册，而NVMe over RoCE设备建链操作复杂，需要人工操作建链和设备注册，随着设备数量增加，操作步骤急剧增加，且操作易出错。NVMe over RoCE方案整体在可靠性和易用性方面相对薄弱。
基于NVMe over RoCE在客户部署应用中存在的问题，存储与交换机团队进行了联合协议创新，推出了NOF+增强技术特性，以补齐在可靠性和易用性方面的薄弱点。
# 技术原理

为了解决设备能够被主机及时发现，以及使得存储网络具备故障快速感知能力，存储节点，主机节点以及网络交换机通过定义新的发现和通告协议，实现网络内的设备自动注册和故障自动处理。实现原理如下：

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/3.png)

主机和存储作为端点设备，接入Fabric网络后，主动向交换机Fabric注册节点信息，交换机实现已注册的设备信息在Fabric网络（iNOF网络）内的所有交换机之间信息同步，TOR接入交换机实时检测接入设备的端口状态和交换机的网络端口状态，并实时通知到订阅状态通知的主机和存储节点。
主机、存储与交换机通告的协议采用LLDP扩展TLV实现，同时为了与LLDP TLV兼容和隔离，通过LLDP通告的关键索引信息区分，LLDP关键索引信息由2个构成：chassis ID和portID，chassis ID采用端口的MAC地址，portID构成采用2部分：前缀+IP对应的端口名称，前缀采用特定字符：discovery_，表示用于设备自动发现的名称；

|字段分类   |名称   |长度（字节）   |说明   |备注   |
| ------------ | ------------ | ------------ | ------------ | ------------ |
|标准定义部分   |TLV类型   |7bit   |扩展TLV类型，127   |   |
|   |TLV长度   |9bit   |按协议定义填充   |   |
|   |OUI   |3   |标准组织OUI   |扩展TLV类型   |
|   |sub-type   |1   |已经申请，值为101   |   |
|subtype 101自定义格式   |版本   |1   |表示版本号，当前版本为1，不同版本可以扩展内容，保持向前兼容   | IP加入域的基础属性，交换机需要进行解析  |
|   |IP类型   |1   |表示IPV4:1，or IPV6:2   |   |
|   |reserved   |2   |保留，未来可扩展为VPNID   |   |
|   |IP地址   |16   |数值格式，IPV4时只占4个字节   |   |
|   |协议角色   |1   |tgt/server(1),ini/client(2),both tgt&ini(server&client)(3)   |交换机不解析字段内容，在通告网络状态变化时作为附带消息转发   |
|   |reserved   |1   |保留   |   |
|   |协议类型   |2   |协议类型：支持多协议类型，NVMe over RoCE, NVMe over TCP, iSCSI等，NVMe over RoCE:1, NVMe over TCP:2,iSCSI:3；注意：网络字节序   |   |
|   |协议端口   |2  |协议服务端使用的端口号，如果协议采用知名端口号则reserved，全部填0；注意：网络字节序   |   |
|   |协议标识符长度   |1   |标识下面“协议标识符”的长度，表示协议标识符实际长度   |   |
|   |协议标识符   |255   |可变长字段，最长255，与协议类型相对应的定义，NVMe over RoCE协议定义为NQN,注意：标识符没有字符串最后的\0   |   |

交换机实现在Fabric内（iNOF网络内），实现注册设备信息的全局同步。

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/4.png)

在交换机Fabric网络中，其中一台或两台交换机设备作为反射器（Reflector），其余交换机设备作为客户端（Client）。每个反射器和客户端之间都需要建立iNOF连接（绿色虚线）从而可以传输iNOF报文。iNOF客户端之间不需要建立iNOF连接，只需要与主机（Host）直连。iNOF反射器之间不需要建立iNOF连接，两者互为备份，与主机直连的接入设备也可以作为iNOF反射器。iNOF报文是TCP封装的报文，TCP端口号范围为10000到57999，缺省值为19516，包含iNOF关键信息的内容承载在TCP报文的Data字段内。客户端可以通过iNOF报文将iNOF关键信息发送给反射器，反射器汇总后再发往其他客户端。通过iNOF报文可以传输以下几类信息：
•	建连信息：iNOF反射器和客户端之间需要通过互相交换iNOF报文来建立iNOF连接，具体的建立过程类似TCP建连。
•	域配置信息：iNOF系统中，设备可以通过域（Zone）对接入的主机进行管理，iNOF反射器上完成iNOF域的相关配置后，会通过iNOF报文把域配置信息发往各个客户端。若一个iNOF系统中同时存在两个反射器，只有在两个反射器上相同的域配置才可以生效。
•	主机动态信息：主机的网卡和iNOF设备均需要启用LLDP功能，当有新的主机接入客户端或者离开客户端时，主机会主动向客户端发送LLDP报文，报文内记录了LLDP邻居信息的变化，让iNOF系统内的其他设备感知到主机动态信息。
•	接口Error-Down信息：当iNOF设备因为PFC死锁、CRC错误报文达到告警阈值等问题触发接口Error-Down后，iNOF报文内会携带接口Error-Down信息，让iNOF系统内的其他设备迅速感知，及时调整路径信息。
iNOF交换机在检测到网络状态发生变化时，会把信息通告到Zone成员内的其他成员，通告支持的消息类别如下：
•	设备接入：指设备加入到iNOF网络的域内，设备接入可以是初次Zone配置的设备接入，也可以是设备由于某种故障原因，断开后重新接入； 
•	链接断开：指设备离线原因是端点设备与交换机之前的链接断开，通常是物理层面的断开；
•	PFC 风暴：指设备离线原因是交换机检测到端点设备发生 PFC 风暴； 
•	交换网络故障：指设备离线原因是通过 BFD 检测到的接入交换机故障或交换机间的级联链路故障； 
•	域配置变更：指设备离线原因是域配置发生变化，设备从域中删除； 
•	端点配置变更：指设备离线原因是设备的 IP 地址配置发生变化； 
•	LLDP 老化：指设备离线原因是设备超过 120 秒没有给交换机发送 LLDP 通告报文
对于主机设备节点，主机检测到有存储设备上线通知消息，且是本地支持的设备类型，主机则会主动发起NVMe-oF的RDMA CM连接请求，以及NVMe-oF Fabric connect请求，最终实现与存储的自动发现和设备上报；如果主机检测到存储设备离线，则首先主机会把后续业务Failover到其他Normal的路径，确保业务连续，接着主机会把离线设备添加到恢复列表，并定期重试尝试恢复，如果超过一定时间设备未恢复，则最终删除设备。
对于存储设备节点，检测到主机设备上线，则存储设备处于被动等待主机建立NVMe-oF连接和后续的业务逻辑处理；如果检测到关联的主机设备离线通知类消息，则执行NVMe-oF的Controller reset处理，完成业务资源的清理。

# 应用指导
以交换机组网为例，4台server，每如server分别使用2个网络端口，接入到2个交换机上，存储双控，每个控制器使用2个端口分别接入到2个交换机上。见下图：

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/5.png)

## •	基于点对点的IP业务域配置规则说明

将所有的业务关系整理出来，作为IP业务域配置，如图4-1所示，所有服务器与存储之间通过2个交换平面进行业务通信，两个IP业务域配置规则如下表所示：
| 交换机1的业务域|| |
| ------------ | ------------ | ------------ |
| 域名  | 成员IP  |成员IP   |
|server1_1_1   |10.1.1.1   |10.1.1.5   |
|server1_1_2   |10.1.1.1   |10.1.1.6   |
|server2_1_1   |10.1.1.2   |10.1.1.5   |
|server2_1_2   |10.1.1.2   |10.1.1.6   |
|server3_1_1   |10.1.1.3   |10.1.1.5   |
|server3_1_2   |10.1.1.3   |10.1.1.6   |
|server4_1_1   |10.1.1.4   |10.1.1.5   |
|server4_1_2   |10.1.1.4   |10.1.1.6   |

| 交换机2的业务域|| |
| ------------ | ------------ | ------------ |
| 域名  | 成员IP  |成员IP   |
|server1_2_1   |10.1.2.1   |10.1.2.5   |
|server1_2_2   |10.1.2.1   |10.1.2.6   |
|server2_2_1   |10.1.2.2   |10.1.2.5   |
|server2_2_2   |10.1.2.2   |10.1.2.6   |
|server3_2_1   |10.1.2.3   |10.1.2.5   |
|server3_2_2   |10.1.2.3   |10.1.2.6   |
|server4_2_1   |10.1.2.4   |10.1.2.5   |
|server4_2_2   |10.1.2.4   |10.1.2.6   |


## •	基于主机端口IP的业务域配置规则说明

根据主机端口IP将端口IP的业务访问关系整理出来，作为IP业务域配置，所有服务器与存储之间通过2个交换平面进行业务通信，IP业务域配置规则如下表所示：
| 交换机1的业务域|| ||
| ------------ | ------------ | ------------ |------------ |
| 域名  | 成员IP  |成员IP   |成员IP|
|server1_1_1   |10.1.1.1   |10.1.1.5   |10.1.1.6   |
|server2_1_1   |10.1.1.2   |10.1.1.5   |10.1.1.6   |
|server3_1_1   |10.1.1.3   |10.1.1.5   |10.1.1.6   |
|server4_1_1   |10.1.1.4   |10.1.1.5   |10.1.1.6   |


| 交换机2的业务域|| ||
| ------------ | ------------ | ------------ |------------ |
| 域名  | 成员IP  |成员IP   |成员IP|
|server1_2_1   |10.1.2.1   |10.1.2.5   |10.1.2.6   |
|server2_2_1   |10.1.2.2   |10.1.2.5   |10.1.2.6   |
|server3_2_1   |10.1.2.3   |10.1.2.5   |10.1.2.6   |
|server4_2_1   |10.1.2.4   |10.1.2.5   |10.1.2.6   |

## 交换机业务域配置
基于点对点的业务域，在交换机1的上配置业务域，以server1_1_1为例：

```
<switch1>system-view
Enter system view, return user view with return command.
Warning: The current device is single master board. Exercise caution when performing this operation.
[~switch1]ai-service
[~switch1-ai-service]inof
[~switch1-ai-service-inof]zone server1_1_1
[*switch1-ai-service-inof-zone-server1_1_1]host 10.1.1.1
[*switch1-ai-service-inof-zone-server1_1_1]host 10.1.1.5
[*switch1-ai-service-inof-zone-server1_1_1]quit
[*switch1-ai-service-inof]commit

```
基于主机端口IP的业务域，在交换机1的上配置业务域，以server1_1_1为例：

```
<switch1>system-view
Enter system view, return user view with return command.
Warning: The current device is single master board. Exercise caution when performing this operation.
[~switch1]ai-service
[~switch1-ai-service]inof
[~switch1-ai-service-inof]zone server1_1_1
[*switch1-ai-service-inof-zone-server1_1_1]host 10.1.1.1
[*switch1-ai-service-inof-zone-server1_1_1]host 10.1.1.5
[*switch1-ai-service-inof-zone-server1_1_1]host 10.1.1.6
 [*switch1-ai-service-inof-zone-server1_1_1]quit
[*switch1-ai-service-inof]commit

```
## 存储配置

使能开启NoF+特性（组件名称：SNSD：Storage Network Smart Discovery），有2种方式：CLI和DeviceManager。
•	通过CLI配置
在developer模式下执行change port eth_snsd_switch eth_port_id=<port_id> snsd_enable=yes，例如：
developer:/>change port eth_snsd_switch eth_port_id=CTE.A.H0 snsd_enable=yes
•	通过DeviceManager配置
o 登录DeviceManager。
o 2、选择“服务 > 网络 > RoCE网络”。进入“RoCE网络”信息展示页面。
o 3、选择需要启用SNSD功能的端口，单击“批量配置SNSD > 批量启用SNSD”

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/6.png)

## 主机配置
使能开启服务有2种方式，第一种方式是在安装时直接启动，第2种方式是完成安装之后重新启动NoF+特性服务。
以server1为例，主机端的配置有2种做法，如果主机所有的RoCE端口都需要运行NVMe over RoCE业务则，直接配置any即可，系统会自动扫描所有支持RoCE业务的端口并启动业务。第2种做法把主机需要运行NVMe over RoCE业务的IP地址添加到配置文件中。

### •	配置RoCE端口全支持的配置文件
配置文件示例如下：

```
*----------------------------------------------*
*             Configuration Body                         *
*----------------------------------------------*/
[BASE]
; The restrain time of disconnecting device when net link down. Unit is second.
; The recommended value is 0.
restrain-time = 0

[SW]
; Switching network configuration, mandatory: --host-traddr, --protocol
; If "--host-traddr" is set to "any", other IP addresses cannot be configured for the switching network. 
; All customer networks support SNSD
; eg:
;	--host-traddr = xxxx | --protocol = (roce/tcp/iscsi)
--host-traddr = any | --protocol = roce

```
### •	配置支持NVMe over RoCE的IP配置文件
配置文件示例如下： 

```
/*----------------------------------------------*
*             Configuration Body                         *
*----------------------------------------------*/
[BASE]
; The restrain time of disconnecting device when net link down. Unit is second.
; The recommended value is 0.
restrain-time = 0

[SW]
; Switching network configuration, mandatory: --host-traddr, --protocol
; If "--host-traddr" is set to "any", other IP addresses cannot be configured for the switching network. 
; All customer networks support SNSD
; eg:
;	--host-traddr = xxxx | --protocol = (roce/tcp/iscsi)
--host-traddr = 10.1.1.1 | --protocol = roce
--host-traddr = 10.1.2.1 | --protocol = roce

```
### •	启动NoF+特性服务
按照前述步骤规则定义，生成配置文件后，直接安装NOF Director默认使能NoF+特性或修改配置文件后，重新启动NOF Director即可。启动命令：systemctl restart nvme-snsd
# 最终效果
可靠性场景：故障场景下，对比开启该特性前后，业务归零时间，从5-10S收敛到<1S。
开启特性前：

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/7.png)

开启特性后：

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/8.png)


易用性场景：网络基础互联互通后，业务配置和配置变更只需要在交换机单点配置，配置完后存储设备和主机设备即插即用，NVMe设备自动发现，自动扫盘，自动故障恢复接入，不再需要人为干预。

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/9.png)

# 生态构建

![](https://gitee.com/tao191/sds_doc/blob/master/solutions/img/10.png)

基于NVMe over RoCE的增强NoF+特性方案，在标准生态，设备生态，社区和操作系统生态等方面已经逐步打开局面，NVMe和IETF相关的国际标准工作也在推进中，国内主流存储厂商和交换机厂商，例如H3C和宏杉科技，以及浪潮等厂家也逐步跟进该技术。华为以及其他相关厂商，也逐步在相关领域开展了越来越多的创新实践，为数据基础设施领域建设打开新格局，NVMe over RoCE生态逐渐走向成熟。

# VXLAN学习笔记

## VXLAN概念：

- **VXLAN（Virtual eXtensible Local Area Network，虚拟扩展局域网）**

  对传统VLAN协议的一种扩展。VXLAN 技术是以三层网络为基础，构建一个大型的二层网络，然后基于这个大型的二层网络还需要继承 VLAN 隔离广播域的功能，同时要实现 VNI 内通信及 VNI 间通讯。

- **网络标识VNI（VXLAN Network Identifier）**
  类似于传统网络中的VLAN ID，用于区分VXLAN段，不同VXLAN段的租户不能直接进行二层通信。一个租户可以有一个或多个VNI，VNI由24比特组成，支持多达16M的租户。
- **广播域BD（Bridge Domain）**
  类似传统网络中采用VLAN划分广播域方法，在VXLAN网络中通过BD划分广播域。
  在VXLAN网络中，将VNI以1:1方式映射到广播域BD，一个BD就表示着一个广播域，同一个BD内的主机就可以进行二层互通。
- **VXLAN隧道端点VTEP（VXLAN Tunnel Endpoints）**
  VTEP可以对VXLAN报文进行封装和解封装。
  VXLAN报文中源IP地址为源端VTEP的IP地址，目的IP地址为目的端VTEP的IP地址。一对VTEP地址就对应着一条VXLAN隧道。在源端封装报文后通过隧道向目的端VTEP发送封装报文，目的端VTEP对接收到的封装报文进行解封装。
- **网关**
  二层网关：用于解决租户接入VXLAN虚拟网络的问题，也可用于同一VXLAN虚拟网络的子网通信。
  三层网关：用于VXLAN虚拟网络的跨子网通信以及外部网络的访问。



VXLAN使用隧道技术让多个二层网络实现通信，在原有报文的基础上，再次封装二层、三层、传输层头部，然后封装一个 VXLAN 头部。**VXLAN 所借助的三层网络隧道就是 Underlay网络，VXLAN 实现的二层通信网络为Overlay网络。**



## 为什么需要VXLAN？

可以满足虚机动态迁移（在保证虚拟机上服务正常运行的同时，将一个虚拟机系统从一个物理服务器移动到另一个物理服务器的过程）时对网络的要求，因为迁移表现为从一个端口到另一个端口，IP地址保持不变。

![image-20230213111604503](C:\Users\鲍文澜\AppData\Roaming\Typora\typora-user-images\image-20230213111604503.png)



## VXLAN与VLAN之间有何不同？

VLAN作为传统的网络隔离技术，12bit，无法满足大型数据中心的租户间隔离需求。而VLXLAN的VNI占据24bit，提供租户数量远超VLAN。



## VTEP

VTEP（VXLAN Tunnel Endpoints，VXLAN隧道端点）是VXLAN网络的边缘设备，是VXLAN隧道的起点和终点，VXLAN对用户原始数据帧的封装和解封装均在VTEP上进行。



## VXLAN网关

- VXLAN二层网关：用于终端接入VXLAN网络，也可用于同一VXLAN网络的子网通信。
- VXLAN三层网关：用于VXLAN网络中跨子网通信以及访问外部网络。

### **VXLAN集中式网关**

集中式网关是指将三层网关集中部署在一台设备上，如下图所示，所有跨子网的流量都经过这个三层网关转发，实现流量的集中管理。

**图1-10** VXLAN集中式网关组网图
![img](https://download.huawei.com/mdl/image/download?uuid=4011370f164a4beaa936e70714832f0e)

部署集中式网关的优点和缺点如下：

- 优点：对跨子网流量进行集中管理，网关的部署和管理比较简单。
- 缺点：
  - 转发路径不是最优：同一二层网关下跨子网的数据中心三层流量都需要经过集中三层网关绕行转发（如图中蓝色虚线所示）。
  - ARP表项规格瓶颈：由于采用集中三层网关，通过三层网关转发的终端的ARP表项都需要在三层网关上生成，而三层网关上的ARP表项规格有限，这不利于数据中心网络的扩展。



### **VXLAN分布式网关**

通过部署分布式网关可以解决集中式网关部署的缺点。VXLAN分布式网关是指在典型的“Spine-Leaf”组网结构下，将Leaf节点作为VXLAN隧道端点VTEP，每个Leaf节点都可作为VXLAN三层网关（同时也是VXLAN二层网关），Spine节点不感知VXLAN隧道，只作为VXLAN报文的转发节点。如下图所示，Server1和Server2不在同一个网段，但是都连接到同一个Leaf节点。Server1和Server2通信时，流量只需要在该Leaf节点上转发，不再需要经过Spine节点。

部署分布式网关时：

- Spine节点：关注于高速IP转发，强调的是设备的高速转发能力。
- Leaf节点：
  - 作为VXLAN网络中的二层网关设备，与物理服务器或VM对接，用于解决终端租户接入VXLAN虚拟网络的问题。
  - 作为VXLAN网络中的三层网关设备，进行VXLAN报文封装/解封装，实现跨子网的终端租户通信，以及外部网络的访问。

**图1-11** VXLAN分布式网关示意图
![img](https://download.huawei.com/mdl/image/download?uuid=721dfb94b9e04ce9a32ffd8f373052a4)

VXLAN分布式网关具有如下特点：

- 同一个Leaf节点既可以做VXLAN二层网关，也可以做VXLAN三层网关，部署灵活。
- Leaf节点只需要学习自身连接服务器的ARP表项，而不必像集中三层网关一样，需要学习所有服务器的ARP表项，解决了集中式三层网关带来的ARP表项瓶颈问题，网络规模扩展能力强。










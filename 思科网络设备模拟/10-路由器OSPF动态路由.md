# 路由器 OSPF 动态路由

## 作用

OSPF 是一种基于链路状态的内部网关协议，用于在大型企业网络或互联网服务提供商网络中实现动态路由。OSPF 特别适用于大型企业网络或 ISP 网络，因为它能够有效处理大量的路由信息，并提供灵活的路由策略。OSPF 可以在复杂的网络拓扑中实现高效的路由计算，同时支持分层和分区，使得网络管理更加灵活。

## 实验

拓扑图：

![202405141322353](https://oss.isiou.cn/PicGo/202405141322353.png)

步骤：

1. 配置二层交换机使其与路由相连的端口为 trunk 模式
2. 配置三层交换的接口使其为 access 模式，并配置 SVI
3. 配置 SVI 地址与掩码
4. 设置 ospf 并配置广播网段
5. 配置剩余路由器的网口与 ospf

> 要点：注意在配置 ospf 时，需要广播的网段有哪些

```shell
# Switch3560 配置
Switch>en
Switch#conf t
Switch(config)#vlan 2
Switch(config-vlan)#vlan 3
Switch(config-vlan)#exit
Switch(config)#int f0/1
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 2
Switch(config-if)#exit
Switch(config)#int f0/2
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 3
Switch(config-if)#exit
Switch(config)#ip routing                                       # 启用路由功能
Switch(config)#interface vlan 2
Switch(config-if)#ip address 10.10.253.1 255.255.255.0          # 配置 SVI
Switch(config-if)#exit
Switch(config)#interface vlan 3
Switch(config-if)#ip address 192.168.10.1 255.255.255.0
Switch(config-if)#exit
Switch(config)#router ospf 1                                    # 指定 OSPF 进程号为 1
Switch(config-router)#network 192.168.10.0 0.0.0.255 area 0     # 0.0.0.255 是通配符掩码
Switch(config-router)#network 10.10.253.0 0.0.0.255 area 0

# Router0 配置
Router>en
Router#conf t
Router(config)#int f1/0
Router(config-if)#ip address 192.168.20.1 255.255.255.0
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#int f0/0
Router(config-if)#no shutdown
Router(config-if)#ip address 10.10.253.2 255.255.255.0
Router(config-if)#exit
Router(config)#int serial 2/0
Router(config-if)#ip address 10.10.254.1 255.255.255.0
Router(config-if)#clock rate 64000
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#router ospf 1
Router(config-router)#network 192.168.20.0 0.0.0.255 area 0
Router(config-router)#network 10.10.253.0 0.0.0.255 area 0
Router(config-router)#network 10.10.254.0 0.0.0.255 area 0

# Router1 配置
Router>en
Router#conf t
Router(config)#int f0/0
Router(config-if)#no shutdown
Router(config-if)#ip address 192.168.30.1 255.255.255.0
Router(config-if)#exit
Router(config)#int serial 2/0
Router(config-if)#no shutdown
Router(config-if)#ip address 10.10.254.2 255.255.255.0
Router(config-if)#exit
Router(config)#router ospf 1
Router(config-router)#network 192.168.30.0 0.0.0.255 area 0
Router(config-router)#network 10.10.254.0 0.0.0.255 area 0
```

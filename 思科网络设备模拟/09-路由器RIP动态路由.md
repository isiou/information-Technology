# 路由器 RIP 动态路由

## 作用

RIP 是一种基于距离向量的动态路由协议，用于在网络中自动交换路由信息。适用于相对简单的网络拓扑结构，特别是对于小型网络而言。它易于配置和实现，并且在小型网络中通常能够提供足够的性能。并且 RIP 协议对路由跳数有限制，通常限制在 15 跳以内。这意味着 RIP 在大型网络或者网络直径超过 15 跳的情况下可能无法正常工作，因为超出限制的路由器将被认为是不可达的。

## 实验

拓扑图：

![202405141131365](https://oss.isiou.cn/PicGo/202405141131365.png)

步骤：

1. 建立如图所示拓扑图
2. 配置 3 台二层交换机，使其每一台连接路由器的端口都为 trunk 模式
3. 配置三层交换机，先划分 VLAN 再给每个端口设置为 access 后，划分每个端口的 VLAN 后启用路由功能
4. 配置 SVI 地址（即上述划分的 VLAN）
5. 启用 RIP 并设置版本、通告的网络网段（类似 192.168.10.0 网段）
6. 对路由器进行相似配置，先设置路由器端口网关，完成后配置 RIP

> 要点：
>
> 1. 了解 SVI 并在设置时指定端口为 access 模式
> 2. 设置 RIP 时需要指定通告的网段

```shell
# Switch0-2 配置
Switch>en
Switch#conf t
Switch(config)#int f0/1
Switch(config-if)#no shutdown
Switch(config-if)#switchport mode trunk
Switch(config-if)#do wr m

# Switch3560 配置
Switch>en
Switch#config t
Switch(config)#vlan 2                           # 划分VLAN
Switch(config-vlan)#vlan 3
Switch(config-vlan)#exit
Switch(config)#inter f0/1
Switch(config-if)#switchport mode access        # 切换端口模式为 access
Switch(config-if)#switchport access vlan 2
Switch(config-if)#exit
Switch(config)#inter f0/2
Switch(config-if)#switchport access vlan 3
Switch(config-if)#exit
Switch(config)#ip routing                       # 启用路由功能
Switch(config)#interface vlan 2
Switch(config-if)#ip address 10.10.254.1 255.255.255.0      # 设置 SVI IP 地址
Switch(config-if)#exit
Switch(config)#interface vlan 3
Switch(config-if)#ip address 192.168.10.1 255.255.255.0
Switch(config-if)#exit
Switch(config)#router rip                       # 启用 RIP
Switch(config-router)#version 2                 # 设置版本
Switch(config-router)#network 192.168.10.0      # 指定通告的网络
Switch(config-router)#network 10.10.254.1       # 指定通告的网络

# Router0 配置
Router>en
Router#conf t
Router(config)#int f0/0
Router(config-if)#ip address 10.10.254.2 255.255.255.0
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#int serial 2/0
Router(config-if)#ip address 10.10.253.1 255.255.255.0
Router(config-if)#clock rate 64000
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#int f1/0
Router(config-if)#ip add 192.168.20.1 255.255.255.0
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#router rip
Router(config-router)#version 2
Router(config-router)#network 10.10.254.0
Router(config-router)#network 10.10.253.0
Router(config-router)#network 192.168.20.0

# Router1 配置
Router>en
Router#conf t
Router(config)#int f0/0
Router(config-if)#ip address 192.168.30.1 255.255.255.0
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#int serial 2/0
Router(config-if)#ip address 10.10.253.2 255.255.255.0
Router(config-if)#no shutdown
Router(config-if)#exit
Router(config)#router rip
Router(config-router)#version 2
Router(config-router)#network 10.10.253.0
Router(config-router)#network 192.168.30.0
```

验证：

```shell
# PC0 PC1
ping 192.168.10.1
ping 192.168.20.12
ping 192.168.30.14

# PC2
ping 192.168.10.1
ping 10.10.253.2
ping 10.10.254.1
```

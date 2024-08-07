# 三层交换机基本配置与 SVI 配置

按照 OSI 七层网络模型，交换机又可以分为第二层交换机（基于 MAC 地址，工作于数据链路层）、第三层交换机（基于 IP 地址，工作于网络层） 等。二层交换机多用于网络接入层和汇聚层，三层交换机多用于核心层。

三层交换机：即带有三层路由功能的交换机，既可以工作在第三层层替代或部分完成传统路由器的功能，同时又具有几乎第二层交换的速度。实质就是一种在性能上侧重于交换的特殊路由。

三层交换机端口默认为二层口，如果需要启用三层功能就需要在此端口输入 `no switchport` 命令。二层交换机没有此命令。

> 如果要在 Cisco 交换机上配置一个接口为路由接口，而不是交换机端口，可以使用 `no switchport` 命令将接口配置为 Layer 3 接口，而不是 Layer 2 交换机端口。

## 实验 1 基本配置

拓扑图：

![202405131112541](https://oss.isiou.cn/PicGo/202405131112541.png)

步骤：

1. 配置 PC 的 IP 地址、子网掩码、网关
2. 开启三层交换机的路由功能
3. 配置路由

```shell
# 交换机配置
Switch>en
Switch#conf t
Switch(config)#int f0/1            # 选取路由端口
Switch(config-if)#no switchport    # 配置为路由端口
Switch(config-if)#ip routing       # 开启路由功能
Switch(config-if)#ip address 192.168.1.1 255.255.255.0    # 设置网关地址和子网掩码
Switch(config-if)#end
```

## 实验 2 SVI 配置

拓扑图：

![202405141453059](https://oss.isiou.cn/PicGo/202405141453059.png)

步骤：

1. 配置二层交换机的接口模式
2. 进入三层交换机，配置 VLAN 后封装 trunk 协议并启用 trunk 模式
3. 封装协议后切换端口 VLAN
4. 配置 SVI IP 地址

```shell
# Switch0 配置
Switch>en
Switch#conf t
Switch(config)#int f0/1
Switch(config-if)#switchport mode trunk
Switch(config-if)#no shutdown
Switch(config-if)#exit
Switch(config)#vlan 2
Switch(config-vlan)#vlan 3
Switch(config-vlan)#exit
Switch(config)#int f0/2
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 2
Switch(config-if)#int f0/3
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 3

# Switch3560 配置
Switch>en
Switch#conf t
Switch(config)#vlan 2
Switch(config-vlan)#exit
Switch(config)#vlan 3
Switch(config-vlan)#exit
Switch(config)#int f0/1
Switch(config-if)#switchport trunk encapsulation dot1q     # 指定 Trunk 接口上使用的封装协议
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport access vlan 2
Switch(config-if)#exit
Switch(config)#ip routing
Switch(config)#interface vlan 2
Switch(config-if)#ip address 192.168.1.1 255.255.255.0
Switch(config-if)#no shutdown
Switch(config-if)#exit
Switch(config)#interface vlan 3
Switch(config-if)#ip address 192.168.2.1 255.255.255.0
Switch(config-if)#no shutdown
Switch(config-if)#exit
Switch(config)#int f0/2
Switch(config-if)#switchport access vlan 3
```

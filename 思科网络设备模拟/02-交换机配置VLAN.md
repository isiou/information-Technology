# 交换机配置 VLAN

## 作用

VLAN 是一种虚拟局域网技术，通过将一个物理网络划分为多个逻辑网络，实现了广播域的隔离、安全性的提高、网络流量的优化以及网络管理的简化。它能够根据部门、功能或项目等标准对网络资源进行逻辑划分，提高了网络资源的管理效率，同时增强了网络的安全性和性能。

## 实验

拓扑图：

![单交换机](https://oss.isiou.cn/PicGo/202405131919219.png)

![多交换机](https://oss.isiou.cn/PicGo/202405132005847.png)

步骤：

1. 划分 VLAN
2. 在交换机 1 上划分 VLAN，配置交换机 1 的 VTP 模式为 server
3. 设置交换机 1 上 VTP 的域名
4. 在交换机 2 上设置 VTP 模式为 client
5. 划分交换机 2 的 VLAN

> 注意：在思科的最新版本中，旧版的 `vtp database` 指令已被弃用，现在使用 `vtp mode server` 进入 VTP 子模式
>
> 若需要划分 VLAN 则在切换 VTP 模式将其中一台交换机设置为服务器模式，另一台设置为客户端模式

```shell
# 单交换机配置 VLAN
Switch>en
Switch#conf t
Switch(config)#vlan 2                  # 创建 VLAN 2
Switch(config-vlan)#vlan 3             # 创建 VLAN 3
Switch(config-vlan)#exit
Switch(config)#int range f0/1-2        # 选取 1-2 端口
Switch(config-if-range)#switchport access vlan 2    # 划分 1-2 端口为 VLAN 2
Switch(config-if-rangen)#exit
Switch(config)#int range f0/3-4        # 选取 3-4 端口
Switch(config-if-range)#switchport access vlan 3    # 划分 3-4 端口为 VLAN 2
Switch(config-if-range)#exit
Switch(config)#do wr m


# 多交换机配置 VLAN
# Switch1 配置
Switch>en
Switch#conf t
Switch(config)#vlan 2                  # 创建 VLAN 2
Switch(config-vlan)#vlan 3             # 创建 VLAN 3
Switch(config-vlan)#exit
Switch(config)#int f0/1
Switch(config-if)#switchport access vlan 2     # 划分 VLAN 2
Switch(config-if)#int f0/2
Switch(config-if)#switchport access vlan 3     # 划分 VLAN 3
Switch(config-if)#exit
Switch(config)#do wr m
Switch(config)#int f0/3
Switch(config-if)#switchport mode trunk        # 切换 Trunk 模式
Switch(config-if)#end
Switch#conf t
Switch(config)#vtp mode server         # 将交换机的 VTP 模式设置为服务器模式
Switch(config)#vtp domain vtp0         # 设置 VTP 的域名
Switch(config)#end
Switch#

# Switch2 配置
Switch>en
Switch#conf t
Switch(config)#vtp mode client         # 将交换机的 VTP 模式设置为客户端模式
Switch(config)#int f0/1
Switch(config-if)#switchport access vlan 2     # 划分 VLAN
Switch(config-if)#int f0/2
Switch(config-if)#switchport access vlan 3
Switch(config-if)#int f0/3
Switch(config-if)#switchport mode trunk        # 切换端口模式
```

# Debian 服务器配置 DHCP 服务

1. 安装 DHCP 服务，以 `isc-dhcp-server` 为例：

   ```shell
   apt update
   apt install isc-dhcp-server
   ```

2. 配置 DHCP 服务：

   - 编辑 DHCP 服务器的主配置文件 `/etc/dhcp/dhcpd.conf`：

     ```shell
     vim /etc/dhcp/dhcpd.conf
     ```

   - 进行以下配置：

     > 文件中有模板，请注意查看，推荐直接对模板进行修改。

     ![202407151103620](https://oss.isiou.cn//images/202407151103620.png)

     ```shell
     # 新增或修改以下配置
     # 设置子网 IP 和子网掩码
     subnet 192.168.110.0 netmask 255.255.255.0 {
       # 设置起始 IP 和终止 IP
       range 192.168.110.100 192.168.110.200;
       # 设置路由网关
       option routers 192.168.110.2;
       # 子网掩码
       option subnet-mask 255.255.255.0;
       # DNS 服务器地址，以阿里云公共 DNS 为例
       option domain-name-servers 223.5.5.5;
       # 默认域名
       option domain-name "example.com";
       # 默认租期时间，单位秒
       default-lease-time 600;
       # 最大租期时间，单位秒
       max-lease-time 7200;
     }

     # 不使用动态更新
     ddns-update-style none;
     # 设置本 DHCP 服务为本子网下的权威 DHCP 服务器
     authoritative;
     ```

   - 在服务器上使用 `ip route` 指令可以查看当前路由表，如：

     ![202407131947833](https://oss.isiou.cn//images/202407131947833.png)

     在此示例中，所有非本地子网的流量将通过 `192.168.110.2` 这个网关，并使用 `ens33` 接口发送。

   - 在服务器上使用 `ip address` 指令可以查看默认 IP 地址和网络接口，输出如下：

     ![202407131910182](https://oss.isiou.cn//images/202407131910182.png)

     其中，`ens33` 即为本服务器的网络接口。

   - 配置 `/etc/default/isc-dhcp-server` 文件，设置 DHCP 服务的默认接口：

     ```shell
     vim /etc/default/isc-dhcp-server
     ```

   - 进行以下配置：

   > 若不使用 IPv6 则需要将 `INTERFACESv6` 一行删除以禁用。

   ```shell
   # 将 IPv4 接口修改为本服务器网络接口
   INTERFACESv4="ens33"
   ```

   - 设置 DHCP 服务为自启动并查看其状态：

     ```shell
     systemctl restart isc-dhcp-server.service
     systemctl status isc-dhcp-server.service
     ```

     ![202407131945421](https://oss.isiou.cn//images/202407131945421.png)

3. 验证 DHCP 服务：

   验证服务端 DHCP 服务是否成功开启，需要有一台客户机与服务器在同一网络下，并且客户机需要设置通过 DHCP 获取 IP 地址。

   > 为避免干扰，在验证自建的 DHCP 服务之前，需将 VMware Workstation 默认的 DHCP 服务关停，如下图所示：
   >
   > ![202407131951822](https://oss.isiou.cn//images/202407131951822.png)

   - 在客户机上，使用 `ip address` 指令查看当前 IP 状态：

     ![202407132001692](https://oss.isiou.cn//images/202407132001692.png)

     发现客户机 IP 符合服务端分配 IP 区间。

   - 在客户机上，使用 `dhclient -v ens33` 指令可查看当前 DHCP 服务器 IP 地址：

     ![202407132007068](https://oss.isiou.cn//images/202407132007068.png)

   - 在服务端上，查看 `/var/lib/dhcp/dhcpd.leases` 文件可以了解当前服务器 DHCP 租约文件：

     ![202407132011298](https://oss.isiou.cn//images/202407132011298.png)

     可查看当前 DHCP 服务的租约表。

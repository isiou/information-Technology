# Debian 服务器配置 DNS 服务

1. 安装 DNS 服务器和工具包：

   ```shell
   apt update
   apt install bind9 bind9utils
   ```

   其中，bind9 为 BIND9 DNS 服务器软件包；bind9utils 为管理和调试 BIND9 服务器的工具集。

2. 配置 BIND 配置文件：

   - 配置 `/etc/bind/named.conf.options` 文件：

     此文件为 BIND 主配置文件，可以配置选项、递归查询、允许查询的来源、转发器等。

     ```shell
     options {
      // 指定 BIND 的工作目录，会在这个目录中存储缓存和临时文件
      directory "/var/cache/bind";

      // 定义外部 DNS 服务器地址，用于将本地无法解析的查询请求转发到这些服务器，这里以阿里云公共 DNS 为例
      forwarders {
              223.5.5.5;
      };

      // 允许所有 IP 通过此 DNS 查询
      allow-query { any; };

      listen-on-v6 { any; };

      # 禁用递归查询
      recursion no;
     }
     ```

   - 配置 `/etc/bind/named.conf.local` 文件：

     此文件为 BIND 区域配置文件，用于指定 DNS 区域。

     ```shell
     zone "example.com" {
        type master;
        file "/etc/bind/zones/db.example.com";
     };
     ```

     此配置说明：新建一个名为 `example.com` 的区域，区域文件路径为 `/etc/bind/zones/db.example.com` 并且将其指定为主区域。

   - 创建区域文件 `/etc/bind/zones/db.example.com` 并配置：

     ```shell
     mkdir -p /etc/bind/zones
     touch /etc/bind/zones/db.example.com
     ```

     或使用 `cp` 复制已存在的 `db.empty` 文件：

     ```shell
     cp /etc/bind/db.empty /etc/bind/zones/db.example.com
     ```

     创建文件后进行配置：

     ```shell
     # 定义 TTL 缓存，此处为 24 小时
     $TTL    604800
     # IN 互联网记录
     # ns1.example.com. 代表 DNS 服务器
     # root.example.com. 管理者邮箱
     @       IN      SOA     ns1.example.com. root.example.com. (
                             1          ; Serial        # 版本号
                             43200      ; Refresh       # 从属服务器检查主服务器更新的时间间隔，单位秒，此处为 12 小时
                             21600      ; Retry         # 从属服务器在刷新失败后再次尝试的时间间隔，单位秒，此处为 6 小时
                             43200      ; Expire        # 从属服务器在无法联系主服务器时保留数据的最长时间，单位秒，此处为 12 小时
                             43200 )    ; Negative Cache TTL  # 无法找到记录时的缓存时间

     @       IN      NS      ns1.example.com.
     @       IN      A       192.168.110.110
     www     IN      A       192.168.110.110
     # 其他自定义记录
     test    IN      A       192.168.110.2
     ```

     或进行如图所示配置：

     ![202407150853811](https://oss.isiou.cn//images/202407150853811.png)

   - 检测 DNS 服务状态：

     > BIND 语法检测很严格，需要严格按照书写规范编写。若遇到错误无法启动时需要使用 `named-checkconf` 检测配置文件，使用 `nameds-checkzone` 检测区域文件。
     >
     > 步骤繁琐，需要细心调试。

     ```shell
     systemctl restart bind9.service
     systemctl status bind9.service
     ```

     若配置成功，输出如下：

     ![202407141846374](https://oss.isiou.cn//images/202407141846374.png)

     使用 `nslookup` 指令检验服务是否正常工作，如下：

     ![202407150855853](https://oss.isiou.cn//images/202407150855853.png)

     `nslookup www.example.com 192.168.110.110` 代表指定 DNS 服务查询 `www.example.com` 的 IP 地址。如图所示查询结果为上述自定义解析，即为配置成功。

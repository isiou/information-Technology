# Zabbix 监控系统搭建

Zabbix 是一个监控解决方案，主要用于 IT 基础设施的监控和网络监控。它设计用于收集、存储、处理和展示来自不同种类资源（如服务器、网络设备、虚拟机等）的数据，以帮助系统管理员和工程师实时监控系统的状态、性能和健康状况。

> 官方网站及文档：<https://www.zabbix.com/documentation/7.0/zh/manual>

## 部署

部署 Zabbix 过程较为复杂，需按两部分进行：数据库与服务。

1. 安装 MySQL 数据库：

   前往 MySQL 官方网站获取最新版本的 deb 文件或使用 `wget` 获取：

   > 此版本 MySQL 为当前（2024/7/19）最新版本。

   ```shell
   apt update && apt install dpkg gnupg -y
   wget https://repo.mysql.com//mysql-apt-config_0.8.32-1_all.deb
   dpkg -i mysql-apt-config_0.8.32-1_all.deb
   ```

   运行上述命令后，会出现可视化选择界面，选择第一个选项后选择 `mysql-8.4-lts` 再选择 `ok` 即可。

   ```shell
   apt update && apt install mysql-server -y
   ```

   运行安装命令后，会出现密码输入框，此密码为 MySQL 的 root 用户密码，设置即可。

2. 安装 Zabbix：

   > Zabbix 安装过程较为繁琐，官方指导文档：<https://www.zabbix.com/cn/download>
   >
   > 本文以安装 Zabbix 7.0 LTS、MySQL、Nginx 配置为例（2024/7/19）：
   >
   > ![202407192123483](https://oss.isiou.cn/images/202407192123483.png)

   - 配置 Zabbix 软件源：

     ```shell
     wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_7.0-2+debian12_all.deb
     dpkg -i zabbix-release_7.0-2+debian12_all.deb
     ```

   - 安装 Zabbix server、Web 前端组件、agent 组件：

     ```shell
     apt update && apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
     ```

   - 创建初始数据库：

     ```shell
     mysql -u root -p

     # 创建 Zabbix 数据库
     mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
     # 创建用户，密码自定义，这里以 root 为例
     mysql> create user zabbix@localhost identified by 'root';
     # 赋予 zabbix 用户对 zabbix 数据库有完全权限
     mysql> grant all privileges on zabbix.* to zabbix@localhost;
     # 设置非 root 用户在有安全配置启用的服务器上创建或修改存储函数及触发器时不会受到限制
     mysql> set global log_bin_trust_function_creators = 1;
     mysql> quit;
     ```

   - 导入初始架构和数据：

     ```shell
     zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
     ```

     此操作会要求输入上一步创建的 zabbix 用户密码。

   - 重设 `log_bin_trust_function_creators` 属性：

     ```shell
     mysql -uroot -p

     # 创建初始数据库时临时设置非 root 用户权限不限制，现在恢复受限制状态
     mysql> set global log_bin_trust_function_creators = 0;
     mysql> quit;
     ```

   - 配置 Zabbix server 数据库：

     配置 `/etc/zabbix/zabbix_server.conf` 文件：

     ![202407192318333](https://oss.isiou.cn/images/202407192318333.png)

     ```conf
     DBPassword=password
     ```

     此密码为数据库用户 zabbix 的密码。

     另外，截至目前（2024/7/19），MySQL 正式版本仍然不在 Zabbix 的支持列表内，需要手动设置允许高版本兼容，只需要在本文件中：

     ![202407192321398](https://oss.isiou.cn/images/202407192321398.png)

     将 `AllowUnsupportedDBVersions=1` 添加进去即可。

   - 配置 Zabbix Web 服务：

     配置 `/etc/zabbix/nginx.conf` 文件，将其端口放开，若是公网服务器也可以修改域名，如图所示：

     ![202407192146861](https://oss.isiou.cn/images/202407192146861.png)

   - 启动服务并设置服务自启动：

     ```shell
     systemctl restart zabbix-server zabbix-agent nginx php8.2-fpm
     systemctl enable zabbix-server zabbix-agent nginx php8.2-fpm
     ```

   服务启动成功并状态正常的话，便可以通过 <http://192.168.xxx.xx:8080> 访问 Zabbix Web 了，继续进行下一步配置。

3. 配置 Zabbix Web 服务：

   在浏览器中访问 <http://192.168.xxx.xx:8080>，即可到达 setup 页面，如图所示：

   ![202407192324705](https://oss.isiou.cn/images/202407192324705.png)

   选择语言：

   ![202407192327286](https://oss.isiou.cn/images/202407192327286.png)

   检测安装条件：

   ![202407192336451](https://oss.isiou.cn/images/202407192336451.png)

   配置数据库连接：

   ![202407192336447](https://oss.isiou.cn/images/202407192336447.png)

   自定义主机名和时区，选择上海即可：

   ![202407192338607](https://oss.isiou.cn/images/202407192338607.png)

   校验选项：

   ![202407192339870](https://oss.isiou.cn/images/202407192339870.png)

   配置完成：

   ![202407192340543](https://oss.isiou.cn/images/202407192340543.png)

   点击完成后跳转到登陆页面，使用 Admin，密码为 zabbix （默认）即可登陆。

   ![202407192341230](https://oss.isiou.cn/images/202407192341230.png)

   登陆后跳转页面如下：

   ![202407192342240](https://oss.isiou.cn/images/202407192342240.png)

   至此，已完成 Zabbix Web 服务的搭建。

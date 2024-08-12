# Debian 服务器配置 FTP 服务

## 概念

FTP 文件传输协议，是在计算机网络的客户端和服务器间传输文件的应用层协议。FTP 协议使得用户可以从一个系统向另一个系统复制文件，操作不受操作系统、文件系统和文件属性的限制。

## 配置步骤

1. 安装并使用 vsftpd 搭建 FTP 服务：

   ```shell
   apt install vsftpd
   ```

   配置 `/etc/vsftpd.conf` 文件，如图所示，修改如下：

   ![202407151443066](https://oss.isiou.cn//images/202407151443066.png)

   ```shell
   anonymous_enable=NO          # 禁止匿名用户登录
   local_enable=YES             # 允许本地用户登录
   write_enable=YES             # 允许文件写入
   local_umask=022              # umask 掩码，对应文件 unmask 反码
   chown_uploads=YES
   chown_username=ftpuser       # 指定为 FTP 用户
   dirmessage_enable=YES        # 进入目录显示消息
   xferlog_enable=YES           # 启用传输日志
   connect_from_port_20=YES     # 使用 20 端口进行连接
   chroot_local_user=YES        # 本地用户限制在其主目录下
   ssl_enable=NO                # 禁用 SSL
   pasv_min_port=10000          # 端口范围
   pasv_max_port=10100
   ```

2. 创建 FTP 用户进行测试：

   重启服务并设置为自启动，查看状态：

   ```shell
   adduser ftpuser
   systemctl restart vsftpd.service
   systemctl enable vsftpd.service
   systemctl status vsftpd.service
   ```

   ![202407151504739](https://oss.isiou.cn//images/202407151504739.png)

3. 连接测试：

   使用另外一台客户端机器使用 `ftp 192.168.xxx.xxx` 进行连接测试：

   > 客户机上需要安装 FTP 以使用命令。

   ![202407151615720](https://oss.isiou.cn//images/202407151615720.png)

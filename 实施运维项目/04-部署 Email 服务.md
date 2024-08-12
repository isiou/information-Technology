# Debian 服务器部署 Email 服务

> 前置任务：部署 DNS 服务完成并成功解析。

1. 安装 Postfix 和 Dovecot 作为邮件传输代理工具和 POP3 服务器，同时安装邮件测试工具：

   > 在安装 Postfix 时，会提示选择配置类型，选择 "Internet Site" 并输入域名即可（如测试域名 `example.com`）。

   ```shell
   apt install postfix dovecot-core dovecot-imapd mailutils
   ```

2. 配置 Postfix 邮件传输：

   - 配置 Postfix 主配置文件 `/etc/postfix/main.cf` 如图所示：

     ![202407161719631](https://oss.isiou.cn//images/202407161719631.png)

     可见，其中有 `myhostname` 等配置选项，需进行下一步配置：

     ```shell
     # 设置邮件服务器的主机名
     myhostname = mail.example.com
     # 设置邮件服务器的域名
     mydomain = example.com
     myorigin = /etc/mailname
     # 监听所有接口，此处为接受所有传入邮件
     inet_interfaces = all
     # 使用的网络协议，此处设置为仅 ipv4
     inet_protocols = ipv4
     mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
     # 指定邮件中继主机，此处表示不使用中继主机直接使用 postfix 发送
     relayhost =
     # 中继邮件的网络范围，此处为回环地址，表示只接受本地地址中继
     mynetworks = 127.0.0.0/8
     # 定义用户邮件的存储位置，即 ~/Maildir/
     home_mailbox = Maildir/
     ```

     配置完成后使用 `systemctl restart postfix` 重启服务。

3. 配置 Dovecot 投递服务器：

   Dovecot 是一个开源的邮件服务器软件，主要用于处理邮件投递和提供邮件访问服务。

   - 配置 Devocot 的主配置文件 `/etc/dovecot/dovecot.conf` 如图：

     ![202407162046054](https://oss.isiou.cn//images/202407162046054.png)

     在其中新加入：

     ```shell
     # 指定邮件存储在用户家目录下的 Maildir 文件夹内
     mail_location = maildir:~/Maildir
     ```

   - 配置 `/etc/dovecot/conf.d/10-auth.conf` 文件：

     ![202407162050294](https://oss.isiou.cn//images/202407162050294.png)

     > 此部分多与安全性有关。

     ```shell
     # 允许明文认证
     disable_plaintext_auth = no
     # 指定允许的认证机制
     auth_mechanisms = plain login
     ```

   - 配置 `/etc/dovecot/conf.d/10-mail.conf` 文件：

     ```shell
     # 指定邮件存储路径
     mail_location = maildir:~/Maildir
     ```

   - 配置 `/etc/dovecot/conf.d/10-master.conf` 文件：

     > 由于本次配置的为 IMAP 邮件服务，故只作相关说明。

     找到 `service imap-login` 部分，如图所示：

     ![202407162056021](https://oss.isiou.cn//images/202407162056021.png)

     将端口注释取消并使用 SSL 认证。

   - 使用 `systemctl restart dovecot` 重启服务。

4. 验证本地邮件是否成功：

   - 为邮件服务设置一个用户，本文以 `emailtest` 为例：

     ```shell
     # 按提示输入密码等
     adduser emailtest

     # 在用户家目录下创建邮箱文件夹
     mkdir -p /home/emailtest/Maildir
     # 将此目录及其所有子目录和文件的所有者和组更改为 emailtest
     chown -R emailtest:emailtest /home/emailtest/Maildir
     ```

   - 发送测试邮件：

     使用 `mail` 命令发送测试邮件到 `emailtest` 用户，若提示 `mail` 命令不存在则需要检查是否安装 `mailutils` 软件包。

     ```shell
     echo "Hello World." | mail -s "Email test" emailtest
     ```

     若发送成功，则可以查看 `/home/emailtest/Maildir` 文件夹下是否有对应文件，如图：

     ![202407162108286](https://oss.isiou.cn//images/202407162108286.png)

5. 公网邮件服务：

   上述配置完成后，可以尝试向公网邮箱发送邮件：

   ```shell
   echo "Hello." | mail -s "Test Email" root iouklmmn@163.com
   ```

   收信如下：

   ![202407180914298](https://oss.isiou.cn//images/202407180914298.png)

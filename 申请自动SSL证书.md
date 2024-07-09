# 申请自动 SSL 证书

## 前置：配置 Nginx 服务器

1. 更新系统软件包

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. 安装 Nginx

   ```bash
   sudo apt install nginx -y
   ```

3. 启动并设置开机启动 Nginx

   ```bash
   sudo systemctl start nginx
   sudo systemctl enable nginx
   ```

4. 配置站点

   在 `/etc/nginx/sites-available/` 目录下为您的站点创建一个新的配置文件，例如 `example.com`。

   `/etc/nginx/sites-available/` 是 Nginx 服务器的一个配置目录，用于存放所有可用的站点配置文件。

   ```bash
   sudo vim /etc/nginx/sites-available/example.com
   ```

   添加以下基本配置：

   > 根据自己域名将配置中的 `example.com` 进行替换为自己域名。

   ```nginx
   server {
       listen 80;
       server_name example.com www.example.com;

       location / {
           root /var/www/example.com/html;
           index index.html index.htm;
       }

       access_log /var/log/nginx/example.com.access.log;
       error_log /var/log/nginx/example.com.error.log;
   }
   ```

5. 启用站点配置：

   创建符号链接到 `/etc/nginx/sites-enabled/` 目录，让 `sites-available` 文件夹下文件同步至此目录下。

   ```bash
   sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
   ```

6. 测试配置并重启 Nginx：

   测试配置是否正确，测试成功将 Nginx 服务器重启。

   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

7. 准备网站内容：

   根据配置文件中的 `root` 配置，创建相应的目录结构并放入您的网站文件。

   ```bash
   sudo mkdir -p /var/www/example.com/html
   echo "Hello, World!" > /var/www/example.com/html/index.html
   ```

8. 使用 `http://example.com`(替换 example) 访问域名，验证是否成功。

## 获取证书

1. 更新系统软件源到最新

   ```shell
   sudo apt update && sudo apt upgrade -y
   ```

2. 安装 Certbot

   若使用的是 Nginx 服务器，可使用以下命令安装 Certbot：

   ```shell
   sudo apt install certbot python3-certbot-nginx
   ```

3. 获取证书

   对于 Nginx 服务器，使用以下命令：

   ```shell
   certbot --nginx
   ```

   在运行这些命令后，Certbot 会自动检测服务器配置，并引导完成安装过程。这包括选择要为其启用 HTTPS 的域名，并自动更新 Web 服务器配置以使用新证书。

4. 自动续订证书

   Let's Encrypt 的证书有效期为 90 天。Certbot 提供了自动续订功能。默认情况下，Certbot 包的续订脚本会添加到 `/etc/cron.d` 或 systemd 定时任务中。

   可以通过以下命令手动测试续订过程：

   ```shell
   sudo certbot renew --dry-run
   ```

5. 配置防火墙

   确保防火墙允许 HTTPS 流量。

   使用 `ufw` 进行配置：

   ```shell
   sudo apt install ufw -y
   sudo ufw allow 'Nginx Full'
   ```

6. 验证 HTTPS

   完成上述步骤后，访问域名 `https://example.com`(替换 example) 以验证 HTTPS 是否已正确启用。

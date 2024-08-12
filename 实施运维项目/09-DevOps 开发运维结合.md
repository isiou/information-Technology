# DevOps 开发运维结合

DevOps 不是某一个单一工具，而是一组过程、方法与系统的统称，用于促进开发和运营等部门之间的沟通、协作与整合。主要通过自动化软件交付和测试，使得软件的构建、发布更加快捷和可靠。

关于 DevOps 的更多详细资料请自行查找。

## 项目部署演示

本文主要以部署以下项目为例。

1. .NET Web 应用：
   地址：<https://gitee.com/zuohuaijun/Admin.NET>

2. Java Web 应用：
   地址：<https://gitee.com/wuxw7/MicroCommunity>

3. PHP Web 应用：
   地址：<https://gitee.com/meystack/swiftadmin>

4. Go Web 应用：
   地址：<https://gitee.com/tiger1103/gfast>

### .NET Web 应用部署

1. 前置任务：

   - 安装 .NET 运行环境：

     ```shell
     apt install wget -y
     wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
     dpkg -i packages-microsoft-prod.deb
     rm packages-microsoft-prod.deb
     apt update && apt install dotnet-sdk-8.0 -y
     ```

     使用 `dotnet --list-sdks` 查看是否安装完成。

   - 安装 Node.js 环境：

     ```shell
     apt install curl -y
     curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
     # 此处注意将输出的 export 复制再粘贴运行
     nvm install 20
     ```

     使用 `node -v` 和 `npm -v` 查看是否安装完成。

2. 下载项目：

   使用 `git clone` 命令拉取仓库：

   ```shell
   mkdir dotnet && cd dotnet
   git clone https://gitee.com/zuohuaijun/Admin.NET && cd Admin.NET && ls -l
   ```

   输出如下：

   ![202407221335894](https://oss.isiou.cn/images/202407221335894.png)

   由结构可知，此为前后端分离项目，前端文件在 `Web` 文件夹下，后端文件在 `Admin.NET` 文件夹下。

3. 部署前端：

   ```shell
   cd Web/
   npm install -g pnpm
   # 若下载速度慢，可使用以下指令换源后再次下载
   # npm config set registry https://registry.npmmirror.com
   pnpm install
   pnpm run dev
   ```

   ![202407221339603](https://oss.isiou.cn/images/202407221339603.png)

   浏览器输入 <http://192.168.xxx.xx:8888> 查看前端页面：

   ![202407221340083](https://oss.isiou.cn/images/202407221340083.png)

   确定前端页面没问题后，使用 `pnpm run build` 打包前端：

   > 打包过程所耗内存较大，虚拟机可能会内存溢出报错。故此步建议在物理机上进行（需在物理机上配置 Node.js 和 .NET SDK），打包完成后使用 scp 命令传回服务器或虚拟机。
   >
   > 物理机上使用 `pnpm` 时可能会报错，使用管理员权限打开 PowerShell 后使用 `set-ExecutionPolicy RemoteSigned` 命令即可。
   >
   > 参考命令：`scp -r "F:\Admin.NET\Web\dist" root@192.168.110.110:/opt/dist`

   打包完成后端文件在当前文件夹 `dist` 文件夹下，将此文件夹及其内容移动到服务器 `/opt/` 下，并对 `/etc/nginx/sites-enable/default` 文件中 server 块做以下配置：

   ```conf
   server {
      # 监听 8888 端口
      listen 8888;
      location / {
             root   /opt/dist;
             index  index.html index.htm;
         }
      location /api/ {
             proxy_pass http://localhost:5005;
             proxy_http_version 1.1;
         }
      location /hubs/ {
             proxy_pass http://localhost:5005;
             proxy_http_version 1.1;
         }
      location /upload/ {
             alias /opt/adminnet/wwwroot/Upload/;
        }
     }
   ```

   使用 `nginx -t` 检查配置文件，使用 `systemctl restart nginx` 重启 Web 服务器。使用 <http://192.168.xxx.xx:8888> 访问前端页面。

4. 测试后端：

   进入 `/Admin.NET/Admin.NET/Admin.NET.Web.Entry` 文件夹下，使用 `dotnet watch --framework net8.0` 拉取依赖和启动服务。

   后端服务启动完成后，显示如图所示：

   ![202407222218423](https://oss.isiou.cn/images/202407222218423.png)

   返回前端页面，如图所示：

   ![202407222219659](https://oss.isiou.cn/images/202407222219659.png)

   登陆后页面：

   ![202407222219751](https://oss.isiou.cn/images/202407222219751.png)

5. 打包后端：

   使用 `dotnet publish -c Release -o output --framework net8.0` 进行项目打包，打包后会在当前文件下生成名为 output 的文件夹，进入后使用 `dotnet Admin.NET.Web.Entry.dll` 运行后端服务。

   若需要使其保持运行，可将其设置为服务或使用 pm2 挂载等。此处介绍简单保留在后台运行方法，使用 `nohup dotnet Admin.NET.Web.Entry.dll > nohup.out 2>&1 &` 即可保留在后台运行，命令中额外提供了保留日志功能。

### Java Web 应用部署

1. 前置任务：

   - 安装 JDK 工具：

     > 本项目使用 Java 8 编写。

     使用 wget 下载 JDK 8 Debian 包，若网络条件不好可以先在本机下载完成后，用 scp 传输到服务端上。

     ```shell
     wget https://builds.openlogic.com/downloadJDK/openlogic-openjdk/8u412-b08/openlogic-openjdk-8u412-b08-linux-x64-deb.deb
     dpkg -i openlogic-openjdk-8u412-b08-linux-x64-deb.deb
     # 验证安装
     java -version
     ```

   - 安装 Apache Maven 工具：

     ```shell
     wget https://dlcdn.apache.org/maven/maven-3/3.9.8/binaries/apache-maven-3.9.8-bin.tar.gz
     tar -zxvf apache-maven-3.9.8-bin.tar.gz /mvn/
     ```

     下载解压完成后，需配置环境变量以供在任意处使用，编辑 `~/.profile` 文件，追加以下内容：

     ```conf
     # 视自己安装情况而定
     export M2_HOME=/mvn/apache-maven-3.9.8
     export PATH=$M2_HOME/bin:$PATH
     ```

     配置完成后，使用 `source ~./profile` 刷新，使用 `mvn -v` 检查配置是否完成。

     > 默认 mvn 源为国外源，下载速度较慢，可将其更改为国内阿里源，以下为教程。

     编辑 `apache-maven-3.9.8/conf/settings.xml` 文件，找到 `<mirrors></mirrors>` 标签块，在其中添加以下内容：

     ```xml
     <mirror>
     <id>aliyunmaven</id>
     <mirrorOf>*</mirrorOf>
     <name>阿里云公共仓库</name>
     <url>https://maven.aliyun.com/repository/public</url>
     </mirror>
     ```

     更多源清自行查找。

2. 拉取仓库：

   使用 git 拉取远程仓库：

   ```shell
   # 后端
   git clone https://gitee.com/wuxw7/MicroCommunity.git

   # 前端
   git clone https://gitee.com/java110/MicroCommunityWeb.git
   ```

### PHP Web 应用部署

1. 前置任务：

   - 配置 PHP 及相关组件：

     ```shell
     apt install php nginx -y
     apt install php-fileinfo php-opcache php-redis php-imagick php-exif php-mysqli   php-curl -y
     apt install redis php-redis php-mysqli php-curl -y
     ```

   - 配置 MySQL 并创建数据库：

     ```shell
     mysql -uroot -p
     mysql> create database swiftadmin;
     ```

2. 启动项目：

   - 拉取仓库后进入仓库根目录，如图所示：

     ![202407242331852](https://oss.isiou.cn/images/202407242331852.png)

   - 启动 PHP 服务：

     ```shell
     php start.php start
     ```

     终端显示：

     ![202407242340040](https://oss.isiou.cn/images/202407242340040.png)

     访问 192.168.xxx.xx:8787 后，会进入以下页面：

     ![202407242342962](https://oss.isiou.cn/images/202407242342962.png)

     点击同意协议，进入配置检测页面：

     ![202407242342303](https://oss.isiou.cn/images/202407242342303.png)

     配置检测通过后，点击下一步，配置数据库：

     ![202407242343162](https://oss.isiou.cn/images/202407242343162.png)

     检查无误后，点击下一步进行安装：

     ![202407242344208](https://oss.isiou.cn/images/202407242344208.png)

     显示此项时即为安装完成。

## 使用 Docker 部署应用

### 前置任务：安装 Docker

安装并配置 Docker 服务：

1. 安装 HTTPS 传输包并添加密钥：

   ```shell
   apt update && apt install apt-transport-https ca-certificates curl gnupg lsb-release
   curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   ```

2. 安装 Docker 及其组件：

   ```shell
   apt update && apt install docker-ce docker-ce-cli containerd.io
   ```

3. 配置 Docker 镜像源：

   - 前往[阿里云 Docker 镜像服务站](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)，查询自己的镜像加速地址，如图所示：

     ![20240728230019](https://oss.isiou.cn/images/20240728230019.png)

   - 修改 Docker 镜像源配置文件，没有则创建：

     ```shell
     vim /etc/docker/daemon.json
     ```

     将上述地址按照文中配置镜像加速器中指引配置。

### Docker 部署 .NET Web 应用

1. 拉取仓库，并在 `Admin.NET/Admin.NET` 文件夹下创建名为 Dockerfile 的文件。

   ```shell
   touch DockerFile
   ```

2. 编辑文件，写入以下内容：

   ```dockerfile
   # 这一行指定了基础镜像为 mcr.microsoft.com/dotnet/sdk:8.0 并且给这个构建阶段命名为 build-env
   FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-env

   # 指定工作目录
   WORKDIR /src

   # 复制当前目录所有文件
   COPY . .

   # 下载项目依赖
   RUN dotnet restore "Admin.NET.sln"

   # 指定入口
   WORKDIR /src/Admin.NET.Web.Entry

   # 打包项目
   RUN dotnet publish -c Release -o /app -f net8.0

   # 指定新的镜像 .net 8.0 running time
   FROM mcr.microsoft.com/dotnet/aspnet:8.0

   # 工作目录
   WORKDIR /app

   # 复制 build-env 的构建
   COPY --from=build-env /app .

   # 设置端口
   EXPOSE 5005

   # 容器启动时，运行服务
   ENTRYPOINT ["dotnet", "Admin.NET.Web.Entry.dll"]
   ```

3. 构建并启动容器：

   ```shell
   docker build -t admin-net .
   docker run -d -p 5005:5005 admin-net
   ```

   此步骤完成后，原本在容器内运行的 .NET 服务已经暴露在本机的 5005 端口上。

4. 前端部署参考上一节中前端部署，步骤相同。

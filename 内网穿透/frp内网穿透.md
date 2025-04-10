# 使用 frp 搭建内网穿透服务

## 内网穿透

内网穿透用于使内网中的设备能够被外网直接访问。通常情况下，内网中的设备（一般如个人电脑等）由于使用私有 IP 地址，无法直接从外网访问。这是因为私有 IP 地址在互联网上不可路由，且由网络地址转换（NAT）设备（如路由器）隔离。

简单来讲，内网穿透就是以一个具有公网 IP 的设备为基础，在此之上创建一条隧道，使外网设备可以借助一个公网 IP 访问到内网设备。

## 实现

本文使用 FRP 进行内网穿透的搭建。

FRP 是一个专为内网穿透而设计的高效、灵活的反向代理应用程序。它能够将内网中的服务暴露给外网访问，帮助解决内网设备无法直接被外网访问的问题。FRP 支持多种协议，包括 TCP、UDP、HTTP 和 HTTPS，适用于各种应用场景。

### 需要的设备

1. 有一台具有公网 IP 的服务器（即服务端）。

2. 需要暴露的内网设备（即客户端）。

### 部署服务端

本文以当前日期（24/8/7）最新版本为例，搭建普通 TCP 穿透服务。

1. 在服务器上下载并解压 FRP：

   ```shell
   wget https://github.com/fatedier/frp/releases/download/v0.59.0/frp_0.59.0_linux_amd64.tar.gz
   tar -xzf frp_0.58.1_linux_amd64.tar.gz
   ```

2. 进入目录后，可见 `frps.toml` 文件，此文件为服务端配置文件。

3. 配置 `frps.toml` 文件：

   ```ini
   # 全局参数
   [common]
   bind_port = 7000         # FRP 服务端监听的端口，客户端会连接到这个端口

   # 服务名，可自定义
   [test-tcp]
   type = tcp               # 指定该服务是 TCP 类型
   local_ip = 127.0.0.1     # 内网机器上的服务地址和端口
   local_port = 22
   remote_port = 7001       # 客户端访问端口
   ```

4. 在服务端启动 FRP 服务：

   ```shell
   ./frps -c ./frps.toml
   ```

### 部署客户端

以客户端为 Windows 系统为例。

1. 下载最新 FRP 压缩包文件：<https://github.com/fatedier/frp/releases/download/v0.59.0/frp_0.59.0_windows_amd64.zip>

2. 解压后编辑文件夹内 `frpc.toml` 文件：

   ```ini
   # 全局配置
   [common]
   server_addr = "xxx.xx.xx.xx"     # 指定服务端 IP
   server_port = 7000               # 指定服务端监听端口

   # 服务名，与服务端一致
   [test-tcp]
   type = tcp                       # 指定该服务为 TCP 类型
   local_ip = 127.0.0.1             # 本机 IP
   local_port = 56657               # 内网服务运行的端口，可更改
   remote_port = 7001               # 外网访问所使用的端口，与服务端一致
   ```

3. 在客户端启动 FRP 服务：

   ```shell
   PS D:\frp_0.59.0_windows_amd64>./frpc -c ./frpc.toml
   ```

## 解释

进行以上配置后，即可通过服务端 IP 和端口号访问到内网设备的服务。

在以上示例中，配置完成并启动后即可通过 `xxx.xx.xx.xx:7001` 访问到内网设备运行在 56657 端口的服务。

若想要让 FRP 在服务端服务长期运行，可以将它们配置为系统服务或使用工具（如 PM2 等）长期托管在后台，并在出现故障时自动重启。本文再次不再赘述。

> 若需了解 FRP 项目以及更详细内容，请访问原项目：<https://github.com/fatedier/frp>

# Kubernetes 搭配 containerd 进行集群部署

Kubernetes(K8s) 是一个开源的平台，用于自动化容器化应用程序的部署、扩展和管理。可以通过 K8s 部署容器化集群，对集群内的容器进行批量化的管理、控制。

containerd 是一种容器，类似 Docker。

> 关于 K8s 与 containerd 详细内容请自行了解。

## 通过 K8s 集群部署 Nginx 应用

> 本节主要目标为在多台 Linux 主机上通过 K8s 集群进行 Nginx 应用的安装。

### 前置：准备工作

> 部署要点：
>
> 1. **集群中各个主机要保证 UUID、hostname、IP 地址唯一。**
>
> 2. **每台主机 hosts 都要修改。**

1. 分配 master 节点与 node 节点：

   > master 节点：控制节点，负责管理集群的整体状态和控制功能。
   >
   > node 节点：工作节点，实际运行应用程序的容器。

   IP 分配方法自行查找，本文不在此赘述。

   > **此处 IP 分配看实际情况。**

   | IP            | 节点类型 | 主机名 |
   | ------------- | -------- | ------ |
   | 192.168.18.10 | master   | master |
   | 192.168.18.20 | node     | node1  |
   | 192.168.18.21 | node     | node2  |

2. cri-containerd 下载：

   下载链接：<https://github.com/containerd/containerd/releases/download/v1.7.20/cri-containerd-1.7.20-linux-amd64.tar.gz>

   本节以最新版本为例，建议先行下载，后用 scp 传输到虚拟机上。

### 部署集群

#### 所有节点配置

> **本小节内容工作在所有节点上，即所有主机都要进行以下配置**

> **强调：修改 UUID、hostname、IP！主机之间不可有相同！**
>
> **每台主机上的 `/etc/hosts` 都要进行修改！**

- 修改主机名：

  > 集群中主机名不可重复，此处以设置 master 主机名为例。

  ```shell
  hostnamectl hostname master && reboot
  ```

- 配置本机 hosts：

  编辑 `/etc/hosts` 文件，写入以下内容，注意修改 127.0.1.1 解析为当前主机名：

  ```shell
  127.0.0.1       localhost
  127.0.1.1       master
  192.168.18.10   master
  192.168.18.20   node1
  192.168.18.21   node2
  ```

- 设置 DNS 服务器：

  ```shell
  echo "nameserver 114.114.114.114" | tee /etc/resolv.conf
  ```

- 配置软件源：

  编辑 `/etc/apt/sources.list` 文件：

  ```shell
  deb https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
  deb-src https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
  deb https://mirrors.aliyun.com/debian-security/ bookworm-security main
  deb-src https://mirrors.aliyun.com/debian-security/ bookworm-security main
  deb https://mirrors.aliyun.com/debian/ bookworm-updates main non-free  non-free-firmware contrib
  deb-src https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
  deb https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
  deb-src https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
  ```

  使用 `apt update && apt full-upgrade -y` 刷新软件包缓存并更新系统。

- 设置系统时区：

  ```shell
  timedatectl set-timezone Asia/Shanghai
  ```

- 时钟同步：

  配置时钟源为阿里云 NTP 服务器并强制进行时间同步。

  ```shell
  apt install chrony -y
  sed -i '/pool 2.debian.pool.ntp.org iburst/ s/^/#/' /etc/chrony/chrony.conf && sed -i '/pool 2.debian.pool.ntp.org iburst/ a\server ntp.aliyun.com iburst' /etc/chrony/chrony.conf
  systemctl enable chrony && systemctl restart chrony

  # 强制进行同步
  chronyc -a makestep
  chronyc sources -v
  chronyc tracking
  ```

  同步后：

  ![20240804091555](https://oss.isiou.cn/images/20240804091555.png)

- 改写系统配置：

  创建并编辑 `/etc/sysctl.d/k8s.conf` 文件，写入以下内容：

  ```shell
  net.ipv4.ip_forward = 1
  net.ipv4.conf.all.send_redirects = 0
  net.ipv4.conf.default.send_redirects = 0
  net.netfilter.nf_conntrack_max = 1000000
  net.netfilter.nf_conntrack_tcp_timeout_established = 86400
  net.core.somaxconn = 1024
  net.ipv4.tcp_syncookies = 1
  net.ipv4.tcp_max_syn_backlog = 2048
  net.ipv4.tcp_synack_retries = 2
  fs.file-max = 65536
  vm.swappiness = 0
  ```

  保存后：

  ```shell
  # 加载内核网络模块，启用网络过滤
  modprobe br_netfilter

  # 应用修改的配置
  sysctl -p /etc/sysctl.d/k8s.conf
  ```

- 配置流量处理：

  - 安装 ipset 与 ipvsadm：

    ```shell
    apt install ipset ipvsadm -y
    ```

  - 创建并编辑 `/etc/modules-load.d/k8s.conf` 文件：

    ```shell
    ip_vs
    ip_vs_rr
    ip_vs_wrr
    ip_vs_sh
    ip_tables
    nf_conntrack_ipv4
    br_netfilter
    ```

    添加权限：

    ```shell
    chmod a+x /etc/modules-load.d/k8s.conf
    ```

- 关闭交换分区：

  编辑 `/etc/fstab` 文件，将 swap 行下内容注释：

  ![20240802151723](https://oss.isiou.cn/images/20240802151723.png)

  重启后，使用 `free` 查看内存：

  ![20240802152212](https://oss.isiou.cn/images/20240802152212.png)

  若 Swap 一行为 0，即代表禁用成功。

  或使用以下方法：

  ```shell
  swapoff -a
  sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  reboot
  ```

- 停止并禁用安全策略：

  > AppArmor 是一种 Linux 内核安全模块，用于限制程序的能力，以提高系统的安全性。此处关闭是为了防止安全模块杀 K8s 进程。

  ```shell
  systemctl stop apparmor.service
  systemctl disable apparmor.service
  ```

#### master 节点配置

> **本小节内容工作在 master 节点上。**

1. 找到准备工作中的 cri-containerd 包，并将其解压到 `/` 下：

   ```shell
   tar -xf cri-containerd-1.7.20-linux-amd64.tar.gz -C /
   ```

2. 修改 cri-containerd 配置：

   ```shell
   mkdir /etc/containerd
   # 导出默认配置文件
   containerd config default > /etc/containerd/config.toml
   # 修改配置文件中使用的沙箱镜像版本
   sed -i '/sandbox_image/s/3.8/3.9/' /etc/containerd/config.toml
   # 设置 containerd 在创建容器时使用 systemd
   sed -i '/SystemdCgroup/s/false/true/' /etc/containerd/config.toml
   ```

   修改完成后，重启服务并查看状态：

   ```shell
   systemctl enable containerd.service
   systemctl restart containerd.service
   systemctl status containerd.service
   ```

   验证组件：

   ```shell
   containerd --version && crictl --version && runc --version
   ```

   输出如图所示：

   ![20240804100452](https://oss.isiou.cn/images/20240804100452.png)

3. 安装 K8s 组件：

   - 安装插件以提供 https 连接：

     ```shell
     apt update && apt install apt-transport-https gnupg gnupg2 curl software-properties-common -y
     ```

   - 下载密钥：

     ```shell
     curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
     ```

   - 写入软件源：

     ```shell
     echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
     ```

   - 刷新软件源缓存并安装组件：

     ```shell
     apt update && apt install kubeadm kubelet kubectl -y
     ```

   - 安装完成后查看版本信息：

     ```shell
     dpkg -l kubeadm kubelet kubectl
     ```

     如图所示：

     ![20240804100659](https://oss.isiou.cn/images/20240804100659.png)

4. 配置 kubelet：

   ```shell
   cat > /etc/default/kubelet << EOF
   KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
   EOF
   systemctl enable kubelet && systemctl restart kubelet
   ```

   配置到此处后，使用 `reboot` 重启主机。

5. 初始化集群：

   > 关于镜像拉取与初始化：
   >
   > 在国内网络下拉取镜像和初始化很容易失败，可在拉取和初始化命令后加上 `--image-repository registry.aliyuncs.com/google_containers` 指定镜像源为阿里云镜像源。
   >
   > 但即使指定镜像源，仍然有较大概率拉取或初始化失败，故推荐有条件的使用一些代理软件，以达到访问原镜像源的效果。
   >
   > 参考代理软件设置：
   >
   > ![20240804131335](https://oss.isiou.cn/images/20240804131335.png)

   > 关于代理：
   >
   > 不使用流量代理很难将镜像拉取下来并初始化（目前未成功过），建议使用代理后拉取，会更加方便。

   拉取镜像：

   ```shell
   kubeadm config images pull
   ```

   ![20240804101508](https://oss.isiou.cn/images/20240804101508.png)

   初始化集群：

   ```shell
   kubeadm init --control-plane-endpoint=master --kubernetes-version=v1.30.3 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.18.10 --cri-socket unix://var/run/containerd/containerd.sock
   ```

   初始化完成后信息：

   ![20240804101957](https://oss.isiou.cn/images/20240804101957.png)

   > 以下两条命令源于上一条命令输出，若找不到了可以使用 `kubeadm token create --print-join-command` 重新获取。

   其中，在工作节点上使用此命令加入此集群：

   ```shell
   kubeadm join master1:6443 --token 9vf65d.3c0q5at7btbh9ac9 --discovery-token-ca-cert-hash sha256:c2a629ff9939d539af6600465d88b75f8adac20f807cd901d816268601222363
   ```

   依据提示，进行配置：

   ```shell
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   export KUBECONFIG=/etc/kubernetes/admin.conf
   ```

   查看集群和节点状态列表：

   ```shell
   kubectl cluster-info && kubectl get nodes
   ```

   ![20240804102943](https://oss.isiou.cn/images/20240804102943.png)

   由上图可见，此时状态为 NotReady，需要安装网络插件以达到联系主机的目的：

   ```shell
   curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml -O
   kubectl apply -f calico.yaml
   ```

   使用 `kubectl get pods -n kube-system` 查看状态：

   > 此处 node pod 未 Ready 为正常现象，建议进行到此处后先配置 node 节点，配置 node 节点完成后重启网络插件，查看 pod 状态。

   ![20240804111847](https://oss.isiou.cn/images/20240804111847.png)

   再次使用 `kubectl get nodes` 查看节点状态：

   > 此处 node1 为新加入的工作节点，加入方式见 node 节点配置。

   ![20240804111952](https://oss.isiou.cn/images/20240804111952.png)

   Ready 代表节点已就绪。

#### node 节点配置

1. 前期准备工作与 master 节点配置第 1-4 点相同，参考上述。

2. 加入集群：

   根据 master 节点配置第 5 点输出的 token，在工作节点上加入集群：

   ```shell
   kubeadm join master1:6443 --token 9vf65d.3c0q5at7btbh9ac9 --discovery-token-ca-cert-hash sha256:c2a629ff9939d539af6600465d88b75f8adac20f807cd901d816268601222363
   ```

   ![20240804105741](https://oss.isiou.cn/images/20240804105741.png)

   在 master 节点上通过`kubectl get nodes`查看节点：

   ![20240804112858](https://oss.isiou.cn/images/20240804112858.png)

   存在新加入的节点并且状态为 Ready 即为成功。

#### 新 master 节点加入集群

1. 前期准备工作与 master 节点配置第 1-4 点相同，参考上述。

2. 在集群内已存在的控制节点上生成密钥和 token：

   ```shell
   kubeadm init phase upload-certs --upload-certs
   ```

   如图所示：

   ![20240804125248](https://oss.isiou.cn/images/20240804125248.png)

3. 保存上一步的密钥和 token，在新加入的主机上：

   > 确保新机上的 hosts 文件和 containerd、kubectl、kubeadm、kubelet 配置完成，且未进行初始化。

   ```shell
   kubeadm join master1:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha256> --control-plane --certificate-key <upload-certs>
   ```

   对应图（注意复制完全）：

   ![20240804125737](https://oss.isiou.cn/images/20240804125737.png)

   运行命令后会经历拉取镜像、初始化等过程，最后输出为：

   ![20240804125905](https://oss.isiou.cn/images/20240804125905.png)

   执行以下命令：

   ```shell
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

   使用 `kubectl get nodes` 查看节点：

   ![20240804130029](https://oss.isiou.cn/images/20240804130029.png)

   发现状态为 NotReady，重复上一小节安装网络插件即可。

   若仍然为 NotReady 状态，则需要重新启动 calico 插件和集群服务：

   ```shell
   kubectl rollout restart daemonset calico-node -n kube-system
   systemctl restart kubelet
   systemctl restart containerd
   ```

   查看 Pod 状态：

   ![20240804131128](https://oss.isiou.cn/images/20240804131128.png)

   等待一会后，使用 `kubectl get nodes` 重新查看节点状态：

   ![20240804131221](https://oss.isiou.cn/images/20240804131221.png)

   节点都为 Ready 即为配置成功。

### 在工作节点上部署 Nginx 应用并启动

1. 在 master 节点上（控制平面）：

   - 创建并编辑 `nginx-install.yml` 文件，写入以下内容：

     ```yml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx-deployment
       labels:
         app: nginx
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: nginx
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
             - name: nginx
               image: nginx:latest
               ports:
                 - containerPort: 80
     ```

   - 创建 Pod：

     ```shell
     kubectl apply -f nginx-install.yml
     ```

     使用 `kubectl get deployments` 查看部署状态：

     ![20240804132959](https://oss.isiou.cn/images/20240804132959.png)

     使用 `kubectl get pods` 查看 Pod 状态：

     ![20240804133048](https://oss.isiou.cn/images/20240804133048.png)

     发现 Pod 都成功运行。现在将容器内的服务暴露在外部：

     创建 `nginx-server.yml` 文件并写入：

     ```yml
     apiVersion: v1
     kind: Service
     metadata:
       name: nginx-service
     spec:
       selector:
         app: nginx
       ports:
         - protocol: TCP
           port: 80
           targetPort: 80
       type: NodePort
     ```

     使用 `kubectl apply -f nginx-server.yml` 创建暴露服务，并使用 `kubectl get svc nginx-service` 查看端口：

     ![20240804133440](https://oss.isiou.cn/images/20240804133440.png)

     可知，端口暴漏在 31363 上，通过工作节点的 31363 端口即可访问对应的 Nginx 应用服务：

     ![20240804133604](https://oss.isiou.cn/images/20240804133604.png)

# 部署 Ansible 自动化运维

Ansible 是一个基于 Python 开发的配置管理和应用部署工具，现在也在自动化管理领域大放异彩。软件能批量配置、部署、管理一大堆的主机。比如以前需要切换到每个主机上执行的一或多个操作，使用 Ansible 只需在固定的一台 Ansible 控制节点上去完成所有主机的操作。

> 通俗话来讲，Ansible 可以做到在一台主机上部署多台主机上的服务，并对其进行操作。

## 配置过程

### 使用 Ansible 批量部署 Nginx 服务

1. 安装 Ansible 软件：

   安装 Ansible 和 ssh 认证：

   ```shell
   apt update && apt install ansible ansible-core sshpass -y
   ```

2. 配置文件并运行：

   使用 `apt install ansible` 安装 ansible 默认不会生成配置文件，需要自己创建。另外，它本身是作为工具软件使用，不作为服务使用，故不可以直接使用 `systemctl` 对其管理。

   ```shell
   mkdir -p /etc/ansible/
   touch /etc/ansible/hosts
   ```

   - 配置 Inventory 文件：

     Inventory 文件是 Ansible 中用于定义要管理的主机列表的关键文件，其格式一般为（文件名可自定义，此处以 `/etc/ansible/host` 为例）：

     > 代表 webservers 组内有 192.168.110.121 与 192.168.110.122 两台主机，可以批量对它们进行操作。

     ```shell
     [webservers]
     192.168.110.121
     192.168.110.122
     ```

     > 实际上此文件属于 Inventory 文件，定义了 Ansible 将要管理的主机列表，可以包含主机分组、主机地址、主机变量等信息

   - 编写 Playbooks 文件：

     Playbooks 文件这是 Ansible 的核心文件，用 YAML 格式编写，定义了自动化任务的步骤和操作。每个 playbook 包含一系列 tasks，每个 task 代表一个具体的操作，例如安装软件、配置服务、启动进程等。Playbooks 可以针对不同的主机分组执行，实现针对性自动化。

     例如，下面的 yml 文件指定了一个安装 nginx 并启动的任务。

     ```yml
     ---
     - hosts: webservers # 指定目标主机分组
       become: true # 提升权限执行任务，使 Ansible 将以 root 用户或其他具有更高权限的用户身份执行后续的任务

       tasks: # 标记任务开始，后面每一个 `- name` 代表一个任务
         - name: 安装 nginx
           apt: # 指定要用的模块，此处为 apt 模块，即为包管理工具
             name: nginx # 指定要安装的软件包名
             state: present # 这行指定了软件包的状态 present，这意味着 Ansible 将确保 nginx 软件包已安装

         - name: 启动 nginx 服务
           service: # 使用服务模块，管理系统服务
             name: nginx
             state: started # 启动服务
             enabled: yes # 设置自启动

         - name: 确认 nginx 服务运行
           service:
             name: nginx
             state: started # 检查服务运行状态
     ```

     > 运行前需要确保每台主机上有 `sudo` 权限！
     >
     > 安装 sudo：`apt install sudo -y`

     使用 `ansible-playbook nginx.yml` 运行文件，会得到报错，如下：

     ![202407190817363](https://oss.isiou.cn//images/202407190817363.png)

     此错误是因为权限被拒绝，表示 Ansible 无法与目标主机（192.168.110.121 和 192.168.110.122）建立安全连接，因为它没有必要的身份验证凭据。

     在此先介绍关于密钥对验证的使用：

     - 验证密钥对是否存在：

       ```shell
       # 在控制主机上即本机
       ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.110.121
       ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.110.122
       ```

       得到以下：

       ![202407190828705](https://oss.isiou.cn//images/202407190828705.png)

       这个错误表明系统上没有找到 `~/.ssh/id_rsa.pub` 文件，意味着现在还没有密钥对文件。

     - 生成密钥对：

       ```shell
       ssh-keygen -t rsa -b 4096 -C "root@e.com"
       ```

       这个命令会提示输入文件保存位置和一个密码短语。默认情况下，密钥对会保存在 `~/.ssh/` 目录下，分别是 id_rsa（私钥）和 id_rsa.pub（公钥）。

       ![202407190831364](https://oss.isiou.cn//images/202407190831364.png)

     - 复制密钥对公钥到目标主机上

       ```shell
       ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.110.121
       ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.110.122
       ```

       使用 `ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.110.121` 检验是否复制成功：

       ![202407190835258](https://oss.isiou.cn//images/202407190835258.png)

       显示目标主机上已有 SSH 密钥对文件。

       使用 `ssh -i ~/.ssh/id_rsa root@192.168.110.121` 测试连接，连接上主机则代表成功。

   - 再次测试：

     使用 `ansible-playbook nginx.yml` 命令，若配置无误的话会显示：

     ![202407190837359](https://oss.isiou.cn//images/202407190837359.png)

     代表 Nginx 服务已安装在目标主机（192.168.110.121 和 192.168.110.122）上并启动成功。

### 使用 Ansible 执行脚本进行配置

本节将介绍使用 Ansible 完成以下操作：更新 apt 源、更新系统、安装 vim、允许 root 用户登录、设置 ls 颜色及简单别名（要求应用于所有用户）、设置系统字符集为 C.utf8（应用于所有用户）。

> Ansible 安装、Inventory、密钥配置见上节。

1. 编写 Shell 脚本，保存为 `/root/test.sh`：

   ```shell
   #!/bin/bash
   apt update
   apt upgrade -y
   apt install vim -y
   # 查找 PermitRootLogin prohibit-password 将其替换为 PermitRootLogin yes
   sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
   systemctl restart ssh
   # 追加
   export LS_OPTIONS='--color=auto' >> /etc/environment
   eval "$(dircolors)" >> /etc/environment
   alias ls='ls $LS_OPTIONS' >> /etc/environment
   alias ll='ls $LS_OPTIONS -l' >> /etc/environment
   alias l='ls $LS_OPTIONS -lA' >> /etc/environment
   echo "LANG=C.utf8" >> /etc/default/locale
   echo "LANGUAGE=C.utf8" >> /etc/default/locale
   ```

2. 新建 `sh.yml` 文件，写入以下内容：

   ```yml
   ---
   - name: 配置脚本
     hosts: all
     become: yes
     tasks:
       - name: 复制脚本到主机
         copy:
           src: /root/test.sh
           dest: /root/test.sh
           mode: "0755"

       - name: 执行脚本
         command: /root/test.sh
   ```

3. 使用 `ansible-playbook sh.yml` 执行脚本：

   ![20240806092810](https://oss.isiou.cn/images/20240806092810.png)

### 使用 Ansible 部署 Prometheus 监控

1. 创建 `install-Prometheus.yml` 并写入以下内容：

   ```yml
   ---
   - name: 安装和配置 Prometheus
     hosts: all
     become: yes
     tasks:
       - name: 创建 Prometheus 用户
         user:
           name: prometheus
           system: yes
           shell: /bin/false

       - name: 创建配置目录
         file:
           # 创建文件夹，使用递归方式，with_item 中
           path: "{{ item }}"
           state: directory
           # 所有者和所属组
           owner: prometheus
           group: prometheus
           # 权限
           mode: "0755"
         with_items:
           - /etc/prometheus
           - /var/lib/prometheus

       - name: 下载 Prometheus
         # wget 下载
         get_url:
           url: "https://github.com/prometheus/prometheus/releases/download/v2.54.0-rc.0/prometheus-2.54.0-rc.0.linux-amd64.tar.gz"
           # 下载目录
           dest: /root/

       - name: 解压 Prometheus
         # 解压命令
         command: tar -xzf /root/prometheus-2.54.0-rc.0.linux-amd64.tar.gz

       # 将各文件移动到指定目录下
       - name: 移动 Prometheus 二进制文件
         copy:
           remote_src: yes
           src: /root/prometheus-2.54.0-rc.0.linux-amd64/prometheus
           dest: /usr/local/bin/prometheus
           owner: prometheus
           group: prometheus
           mode: "0755"

       - name: 移动 Prometheus 工具文件
         copy:
           remote_src: yes
           src: /root/prometheus-2.54.0-rc.0.linux-amd64/promtool
           dest: /usr/local/bin/promtool
           owner: prometheus
           group: prometheus
           mode: "0755"

       - name: 移动 Prometheus 配置文件
         copy:
           remote_src: yes
           src: /root/prometheus-2.54.0-rc.0.linux-amd64/prometheus.yml
           dest: /etc/prometheus/prometheus.yml
           owner: prometheus
           group: prometheus
           mode: "0644"

       - name: 移动 Prometheus 控制台文件
         copy:
           remote_src: yes
           src: /root/prometheus-2.54.0-rc.0.linux-amd64/consoles
           dest: /etc/prometheus/consoles
           owner: prometheus
           group: prometheus
           mode: "0755"

       - name: 移动 Prometheus 控制台库文件
         copy:
           remote_src: yes
           src: /root/prometheus-2.54.0-rc.0.linux-amd64/console_libraries
           dest: /etc/prometheus/console_libraries
           owner: prometheus
           group: prometheus
           mode: "0755"

       # 配置服务
       - name: 创建 Prometheus systemd 服务文件
         copy:
           dest: /etc/systemd/system/prometheus.service
           content: |
             [Unit]
             Description=Prometheus
             Wants=network-online.target
             After=network-online.target

             [Service]
             User=prometheus
             Group=prometheus
             Type=simple
             ExecStart=/usr/local/bin/prometheus \
               --config.file /etc/prometheus/prometheus.yml \
               --storage.tsdb.path /var/lib/prometheus/

             [Install]
             WantedBy=multi-user.target

       - name: 重新加载 systemd
         command: systemctl daemon-reload

       # 启动 Prometheus
       - name: 启用并启动 Prometheus 服务
         systemd:
           name: prometheus
           enabled: yes
           state: started
   ```

2. 使用 `ansible-playbook install-Prometheus.yml` 执行任务：

   ![20240806110451](https://oss.isiou.cn/images/20240806110451.png)

3. 访问 192.168.xxx.xx:9090 查看面板：

   ![20240806110613](https://oss.isiou.cn/images/20240806110613.png)

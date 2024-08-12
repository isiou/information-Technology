# 实施与运维项目

## 前置条件

使用最新 Debian 12.6 镜像，镜像源使用科大源：

```shell
deb http://mirrors.ustc.edu.cn/debian bookworm main contrib non-free non-free-firmware
deb http://mirrors.ustc.edu.cn/debian bookworm-updates main contrib non-free non-free-firmware
```

VMware Workstation 版本使用当前最新 17.5.2 版本。

## 基础配置

新安装的虚拟机需要进行一系列设置以供后续使用。

1. 更改镜像源：

   修改 `/etc/apt/sources.list` 文件，将上述清华源配置替换进去即可。注意需注释源配置或删除原配置。

2. 安装基础软件包：

   ```shell
   apt update && apt install vim htop tree sudo dpkg wget curl locales git gnupg -y
   ```

3. 允许虚拟机使用 root 用户进行 SSH 登录：

   修改 `/etc/ssh/sshd_config` 文件中 `PermitRootLogin` 值为 `yes` 后重启 SSH 服务即可。

4. 修改系统字符集：

   系统默认字符为 `LC_ALL=C`，不支持中文且在一些设备上的支持性不佳，需要将其修该为兼容性更好的字符集如 `en_US.UTF-8`：

   ```shell
   apt install locales
   dpkg-reconfigure locales
   ```

   进行上述步骤后会进入可视化界面，向下翻页后选中 `en_US.UTF-8` 选项并回车，跳转页面后再次选中 `en_US.UTF-8` 即可进行下述步骤。

   修改 `~/.profile` 将文末 `LC_ALL=C` 等移除。将其替换为以下内容：

   ```shell
   export LANG=en_US.UTF-8
   export LANGUAGE=en_US.UTF-8
   export LC_ALL=en_US.UTF-8
   ```

   修改 `/etc/default/locale` 文件，将其内容替换为：

   ```shell
   LANG=en_US.UTF-8
   LANGUAGE=en_US.UTF-8
   LC_ALL=en_US.UTF-8
   ```

   配置上述完成后重启系统即可，使用 `locale` 查看系统当前字符集。

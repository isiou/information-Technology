# HWID 激活系统

通过 HWID 激活系统是指基于硬件设备的唯一标识符（HWID）进行系统激活的一种方式。

> HWID 通常由计算机的硬件组件（如 CPU、主板、硬盘、网卡等）生成的唯一标识符组成，用于唯一标识一台计算机。

本文主要介绍通过 HWID 激活 Windows 操作系统。

> 官方文档及密钥：<https://massgrave.dev/hwid.html>
>
> 票据文件链接：<https://oss.isiou.cn/package/HWID-Tickets.zip>

## 前提条件

- 确保计算机已连接互联网

- 需要管理员权限来执行这些操作

## 激活步骤

1. 以管理员身份打开 PowerShell

2. 输入密钥

   - 在 PowerShell 中输入以下命令，将 `key` 替换为需要激活的系统密钥

     ```powershell
     slmgr /ipk key
     ```

3. 下载票据文件并解压，文件链接见上

4. 获取当前系统 ID

   - 在 PowerShell 中输入以下命令：

     ```powershell
     (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\ProductOptions).OSProductPfn
     ```

   - 这条命令会显示类似 `Microsoft.Windows.48.X19-98841_8wekyb3d8bbwe` 的文本

5. 找到对应的票据文件

   - 在之前解压的票据文件夹中找到与上一步显示的名称完全一致的票据文件

6. 复制票据文件到指定文件夹

   - 将找到的票据文件复制到以下路径 `C:\ProgramData\Microsoft\Windows\ClipSVC\GenuineTicket`

7. 应用票据

   - 在 PowerShell 中输入以下命令应用票据

     ```powershell
     clipup -v -o
     ```

8. 激活 Windows

   - 输入以下命令激活 Windows

     ```powershell
     slmgr /ato
     ```

9. 检查激活状态

   - 输入以下命令检查激活状态

     ```powershell
     slmgr /xpr
     ```

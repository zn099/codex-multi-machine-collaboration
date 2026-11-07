# SSH 配置

当前电脑是发起连接的控制端，目标电脑是运行 Codex 任务的执行端。两端都可以使用 Ubuntu 或 Windows。

配置完成后，在当前电脑运行 `ssh 别名 whoami`，应返回目标电脑的用户。

## 1. 在目标电脑开启 SSH 服务

### 目标是 Ubuntu

**在 Ubuntu 目标电脑的终端执行。** 已经安装并运行的服务不用重装。

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
systemctl is-active ssh
```

最后应显示 `active`。再记下这台电脑的用户：

```bash
whoami
tailscale ip -4
```

### 目标是 Windows

**在 Windows 目标电脑上，右键 PowerShell，选择“以管理员身份运行”。** 路径显示 System32 不代表已经提升权限，先检查：

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

应返回 True。再检查安装状态：

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
```

只有未安装时执行：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

启动服务并设为自动启动：

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
Get-Service sshd
```

应看到 Running。如果已经用其他发行版安装了 OpenSSH，沿用现有服务，不重复安装系统组件。微软的[安装说明](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)包含系统要求和默认防火墙规则。

## 2. 在当前电脑准备登录用的公钥

**在发起连接的电脑操作。每台控制端使用自己的私钥。**

先看有没有已经用于这些机器的公钥：

Ubuntu：

```bash
ls "$HOME/.ssh/"*.pub
```

Windows PowerShell：

```powershell
Get-ChildItem "$env:USERPROFILE\.ssh\*.pub"
```

有就选现有的那一对，记住文件名，继续下一步。没有合适的密钥时才生成；下面采用独立名称 id_ed25519_codex，若该路径已存在，不覆盖。

Ubuntu：

```bash
mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"
if [ -e "$HOME/.ssh/id_ed25519_codex" ] || [ -e "$HOME/.ssh/id_ed25519_codex.pub" ]; then
  echo '这个名称已经存在，请使用已有密钥或另选名称。'
else
  ssh-keygen -t ed25519 -f "$HOME/.ssh/id_ed25519_codex" -C 'codex-multi-computer'
fi
```

Windows PowerShell：

```powershell
$keyPath = Join-Path $env:USERPROFILE '.ssh\id_ed25519_codex'
New-Item -ItemType Directory -Force (Split-Path $keyPath) | Out-Null
if ((Test-Path $keyPath) -or (Test-Path "$keyPath.pub")) {
    Write-Host '这个名称已经存在，请使用已有密钥或另选名称。'
} else {
    ssh-keygen -t ed25519 -f $keyPath -C 'codex-multi-computer'
}
```

按提示设置密钥口令。之后要让桌面客户端使用它，需保证客户端能通过当前 SSH agent 读取已解锁的密钥；普通终端能手工输入口令，不等于桌面后台能输入。

查看公钥时只读 `.pub` 文件。沿用旧密钥的，请替换下面的文件名：

```bash
# Ubuntu
cat "$HOME/.ssh/id_ed25519_codex.pub"
```

```powershell
# Windows
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_codex.pub"
```

把输出的一整行交给目标电脑。没有 `.pub` 后缀的那个文件是私钥，留在当前电脑。

## 3. 把公钥加到目标账户

### 加到 Ubuntu

**在目标 Ubuntu 上，用准备接受 SSH 登录的用户执行：**

```bash
mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"
touch "$HOME/.ssh/authorized_keys"
chmod 600 "$HOME/.ssh/authorized_keys"
nano "$HOME/.ssh/authorized_keys"
```

保留已有内容，在新的一行粘贴控制端公钥。已经有同一把公钥就不用再加。保存后退出。

### 加到 Windows

先在平时使用的 Windows 账户下运行 whoami，确认要登录谁。以下针对本地账户和默认 OpenSSH 授权文件设置；域账户或改过 sshd_config 的环境，应先检查实际设置。

**如果这个账户属于本地 Administrators 组：** 在管理员 PowerShell 执行下面这段。先替换公钥字符串；它应以 ssh-ed25519、ssh-rsa 或 ecdsa-sha2- 开头。

```powershell
$ErrorActionPreference = 'Stop'
$isElevated = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
if (!$isElevated) { throw '请在以管理员身份运行的 PowerShell 中执行。' }

$key = '在这里粘贴当前电脑的完整公钥'
if ($key -notmatch '^(ssh-ed25519|ssh-rsa|ecdsa-sha2-\S+)\s+\S+') { throw '请先替换为完整公钥。' }
$p = Join-Path $env:ProgramData 'ssh\administrators_authorized_keys'
New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null
if (!(Test-Path $p) -or !(Select-String -Path $p -SimpleMatch $key.Split(' ')[1] -Quiet)) {
    Add-Content -Path $p -Value ("`r`n" + $key) -Encoding ascii
}
icacls.exe $p /inheritance:r /grant:r '*S-1-5-32-544:F' '*S-1-5-18:F'
if ($LASTEXITCODE -ne 0) { throw '权限设置失败，请保留错误信息。' }
icacls.exe $p
```

最后查看 ACL，应只有 Administrators 和 SYSTEM 的访问授权。若已有其他显式权限、所有者异常或仍拒绝访问，先检查现有权限，避免直接使用 takeown 或授予 Everyone 访问权限。默认配置下这个文件服务于管理员组账户，具体访问规则见[微软密钥说明](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)。

**如果是普通本地账户：** 用那个账户自己的普通 PowerShell，替换公钥后执行：

```powershell
$ErrorActionPreference = 'Stop'
$key = '在这里粘贴当前电脑的完整公钥'
if ($key -notmatch '^(ssh-ed25519|ssh-rsa|ecdsa-sha2-\S+)\s+\S+') { throw '请先替换为完整公钥。' }
$p = Join-Path $env:USERPROFILE '.ssh\authorized_keys'
New-Item -ItemType Directory -Force (Split-Path $p) | Out-Null
if (!(Test-Path $p) -or !(Select-String -Path $p -SimpleMatch $key.Split(' ')[1] -Quiet)) {
    Add-Content -Path $p -Value ("`r`n" + $key) -Encoding ascii
}
```

如果提示权限不正确，检查该用户目录的 ACL，并确认授权文件属于实际登录的账户。

## 4. 配置主机别名

**在当前电脑编辑 SSH 配置，保留原有内容。**

Ubuntu 文件是 `~/.ssh/config`。Windows 文件是 `$env:USERPROFILE\.ssh\config`，注意文件名就是 config，没有 .txt 后缀。

把下面占位内容替换成你记录的地址和用户名，再追加进去。IdentityFile 必须指向当前电脑实际使用的私钥：

```sshconfig
Host company
    HostName <公司电脑的Tailscale地址>
    User <公司电脑的登录用户>
    Port 22
    IdentityFile ~/.ssh/id_ed25519_codex
    IdentitiesOnly yes

Host classroom
    HostName <教室电脑的Tailscale地址>
    User <教室电脑的登录用户>
    Port 22
    IdentityFile ~/.ssh/id_ed25519_codex
    IdentitiesOnly yes
```

只写需要的主机。company、classroom 是这份教程举的别名，已有好用的别名可以继续保留。Ubuntu 可将 config 权限设为 600。

这份配置只对当前电脑生效。在另一台 Windows 上直接输入 `ssh company`，如果那台还没配置同名 Host，就会报找不到主机。

## 5. 验证指纹与登录

**先在目标电脑获取它自己的 ED25519 主机指纹。**

Ubuntu：

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Windows：

```powershell
ssh-keygen -lf "$env:ProgramData\ssh\ssh_host_ed25519_key.pub"
```

然后在当前电脑首次连接。下面指定 ED25519，方便和刚才那条指纹逐一核对：

```text
ssh -o HostKeyAlgorithms=ssh-ed25519 company whoami
```

目标是教室就把 company 换成 classroom。提示指纹时，与目标电脑提供的值比较，一致后输入 yes；不一致先停下来检查。以后正常使用 `ssh company whoami` 即可。

如果 SSH 卡在超时，先按[连接排查](remote-computers.md)检查端口与网络。只在证据表明被防火墙拦截时添加相应放行，范围限定到自己的 Tailscale 地址；不关闭整个防火墙，也不在路由器映射公网 22 端口。

登录成功后，继续[配置远端 Codex](setup.md#3-配置远端-codex)。

# 配置指南

控制端是当前使用 Codex 的电脑；执行端是运行远端任务的电脑。先完成一个连接方向，再添加其他执行端。

## 1. 配置 Tailscale

**执行位置：所有电脑。**

Windows 从 [Tailscale 官网](https://tailscale.com/download/windows)下载安装，打开后登录。自己的几台电脑使用同一个 Tailscale 账号最省事；团队设备则由管理员授权加入同一网络。

Ubuntu 还没安装时，可以使用[官方安装方式](https://tailscale.com/docs/install/linux)：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

打开终端给出的链接完成登录。已安装并登录的电脑跳过这一步。

然后在每台电脑的终端或 PowerShell 中运行：

```text
tailscale ip -4
tailscale status
whoami
```

**电脑名称、登录用户名和 SSH 别名要分清。** 例如 Windows 的 `whoami` 显示 `MY-PC\alice`，登录用户名应填 `alice`。把每台电脑的信息记下来：

| 用来做什么 | SSH 别名 | Tailscale 地址 | SSH 登录用户 |
| --- | --- | --- | --- |
| 公司电脑跑任务 | company | 公司电脑实际的 100.x 地址 | Ubuntu 的 whoami 输出 |
| 教室电脑接着工作 | classroom | 教室电脑实际的 100.x 地址 | Windows 的本地账户名 |
| 个人电脑也作为执行端 | personal | 个人电脑实际的 100.x 地址 | 个人电脑的账户名 |

**验证连接：** 在控制端把占位内容换成目标电脑的地址，再运行：

```text
tailscale ping --c 5 <目标电脑的Tailscale地址>
tailscale ping --tsmp --c 5 <目标电脑的Tailscale地址>
```

以实际收到回包为准。`via DERP(...)` 表示通过中转连接，可以正常使用。

## 2. 配置 SSH

**执行位置：执行端启用 SSH 服务，控制端配置登录。**

Tailscale 提供网络，OpenSSH 提供远程登录。本教程使用“普通 OpenSSH 走 Tailscale 地址”，不需要额外启用 Tailscale SSH 功能。

按目标系统完成 [SSH 配置](ssh-setup.md)：开启服务、添加公钥、配置别名并核对指纹。已有配置直接检查。

完成后，在当前电脑运行：

```text
ssh company whoami
ssh classroom whoami
```

只测试已配置的目标。返回的用户名应属于远端电脑，例如 Windows 会显示 `机器名\用户名`。

**需要双向连接时：** 在另一台电脑上按相同步骤配置反方向。各台电脑保留自己的私钥，只把公钥添加到目标账户。

## 3. 配置远端 Codex

**执行位置：执行端安装，控制端验证。**

目标电脑已经能用 Codex，就先检查现有安装。还没有 CLI 时，按 [Codex 官方安装说明](https://developers.openai.com/codex/cli/)安装；已有 Node.js／npm 的环境也可以执行：

```text
npm install -g @openai/codex
```

在目标电脑用准备接受 SSH 登录的那个账户运行：

```text
codex --version
codex login status
```

新安装需要登录时，在目标电脑本机完成。原本使用 API 或自定义服务商的，保留原来的配置，不因为连接另一台电脑而换一套认证。

再回到当前电脑：

```text
ssh company "codex --version"
ssh classroom "codex --version"
```

只检查已配置的目标。SSH 中找不到 codex 时，检查远端 PATH；Windows 可运行 `ssh classroom "where.exe codex"`。

## 4. 接入远端聊天

**执行位置：控制端 Codex 客户端。**

1. 打开 **Settings → Connections（设置 → 连接）**，找到 SSH 主机。
2. 添加或启用前面配置的 company、classroom 等别名，等待连接成功。
3. 需要添加远程项目时，选择目标电脑上真实存在的目录，例如 Ubuntu 的项目目录，或 Windows 的 `C:\Users\账户\Documents\项目`。
4. 回到侧边栏／聊天列表，找到这台远端电脑上的已有聊天，点开一条熟悉的，确认能看到它原来的记录和当前状态。
5. 按同样方法加入其他电脑。它们的聊天也会出现在当前客户端中，你可以在这些聊天之间直接切换。

**确认远端聊天已经显示，并且能点开阅读。** 通过主机或运行位置标识区分电脑。不同版本的布局可能不同，连接入口见 [官方说明](https://learn.chatgpt.com/docs/remote-connections#connect-to-an-ssh-host)。

如果终端 SSH 已经成功，而 Codex 仍显示连接失败，请看 [Codex 连接排查](remote-computers.md#ssh-成功但-codex-连接失败)。若连接成功却找不到旧聊天，继续看下面的排查方法。

## 5. 查看进度与发送指令

### 查看工作进度

在同一个聊天列表里，点开公司电脑正在运行的任务，查看当前状态、已有输出和后续回复。再点开教室电脑上的任务，就切换到了另一台电脑的工作。

同名聊天可以通过所属主机和项目区分。网络断开时，界面里保留的旧记录不代表最新进度，需要恢复连接后再查看。

### 在原聊天中继续下指令

打开要继续的聊天，直接在输入框中发送要求。对应电脑上的 Codex 会沿用这条对话的上下文继续处理，后续进度和结果也显示在这里。

### 在指定电脑上新建任务

选择目标电脑上的项目，确认主机和路径，再新建聊天、输入任务。要接着做已有任务，直接回到原聊天即可。

### 已经接入，为什么某些历史聊天没有显示？

先确认目标连接已启用，再检查侧边栏当前的主机、项目分组或筛选范围。接着核对 SSH 是否登录了原来使用 Codex 的系统用户，以及远端 Codex 服务使用的数据目录。Windows 用户改名后，个人目录可能仍是旧名称；不同账户或不同 `CODEX_HOME` 也可能读到不同历史。

聊天记录仍保存在各自的主机上。详细排查见 [让远端聊天显示出来](codex-chats.md)。

## 6. 保持远端可用

远端电脑需要保持开机、联网和唤醒。离开前检查电源与睡眠设置，避免任务进行中无法访问。

Windows 如果希望注销后 Tailscale 仍在线，可在托盘图标的 **Preferences → Run unattended** 开启无人值守模式，具体见 [Tailscale 官方说明](https://tailscale.com/docs/how-to/run-unattended)。这只解决网络在线；Codex 的桌面、沙箱或某些 Windows 适配方式可能仍需要对应用户保持登录，要分别确认。

离开前从另一台电脑做一次完整检查：能登录 SSH、能打开远端聊天、能收到远端回复。短暂断网后先重连原来的主机、查看原聊天状态，再决定是否重试；不要直接把同一任务再发一遍。

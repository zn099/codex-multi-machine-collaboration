# 把几台电脑的 Codex 聊天放到一个窗口里

公司一台 Ubuntu，教室一台 Windows，家里还有一台个人电脑。各台电脑上都跑着任务，换个地方工作，就得挨个找它们的进度。

**连好以后，这几台电脑的 Codex 聊天可以一起显示在你眼前的 Codex 界面中。点开哪条，就能看那条的工作进度；想补充要求，直接在原聊天里输入。** 公司电脑继续跑公司的任务，教室电脑继续做教室的工作，你在一个窗口里就能照看它们。

这份教程带你先用 **Tailscale** 连通电脑，再用 **SSH** 把它们接入 Codex。仓库也提供配套 [Skill](SKILL.md)，让 Codex 帮你完成配置、检查连接，以及解决远端聊天没有显示的问题。

## 连好以后，你会怎样使用

打开当前电脑的 Codex，在侧边栏／聊天列表里找到本机和已接入电脑上的聊天。接下来就是你平时使用 Codex 的方式：

1. **点开聊天，看进度。** 选择公司电脑上的一条聊天，直接阅读已有记录、当前状态和新的输出。
2. **在原聊天输入，继续下指令。** 在输入框里补充要求、回答问题或安排下一步，消息会交给那条聊天所在的电脑处理。
3. **换一条聊天，查看另一台电脑。** 再点开教室电脑上的聊天，就能接着看它做到哪里了。

多台电脑提供执行环境，同一个 Codex 窗口提供聊天入口。任务仍在各自的电脑上运行，文件也留在原来的位置。

```mermaid
flowchart LR
    B["公司 Ubuntu 的聊天"] <-->|"Tailscale + SSH"| A["同一个 Codex 界面<br/>集中显示聊天<br/>点开看进度 · 直接输入指令"]
    C["教室 Windows 的聊天"] <-->|"Tailscale + SSH"| A
    D["当前电脑的聊天"] <--> A
```

第一次先把一台远端电脑接进来，例如“个人电脑 → 公司电脑”。确认它的聊天已经出现在当前界面，再添加教室电脑。想从另一台电脑也这样使用，就在那台电脑上配置相应连接。

## 第一步：让几台电脑加入同一个 Tailscale 网络

**在哪操作：每台需要参与的电脑。**

Windows 从 [Tailscale 官网](https://tailscale.com/download/windows)下载安装，打开后登录。自己的几台电脑使用同一个 Tailscale 账号最省事；团队设备则由管理员授权加入同一网络。

Ubuntu 还没安装时，可以使用[官方安装方式](https://tailscale.com/docs/install/linux)：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

打开终端给出的登录链接，完成登录。已经安装并登录的机器直接往下走，不要为了照教程重新退出账号。

然后在每台电脑的终端或 PowerShell 中运行：

```text
tailscale ip -4
tailscale status
whoami
```

记下地址和用户名。这里最容易弄混的是：**电脑名称、登录用户名和 SSH 别名是三回事。** Windows 的 whoami 如果显示 `MY-PC\alice`，本地账户名就是 alice，不是 MY-PC。

可以给自己留这样一张表，实际内容留在自己的电脑上：

| 用来做什么 | SSH 别名 | Tailscale 地址 | SSH 登录用户 |
| --- | --- | --- | --- |
| 公司电脑跑任务 | company | 公司电脑实际的 100.x 地址 | Ubuntu 的 whoami 输出 |
| 教室电脑接着工作 | classroom | 教室电脑实际的 100.x 地址 | Windows 的本地账户名 |
| 个人电脑也作为执行端 | personal | 个人电脑实际的 100.x 地址 | 个人电脑的账户名 |

**怎么知道这一步成功了：** 在准备发起连接的电脑上，把下面的占位内容换成目标电脑地址再执行：

```text
tailscale ping --c 5 <目标电脑的Tailscale地址>
tailscale ping --tsmp --c 5 <目标电脑的Tailscale地址>
```

管理页面的绿点只说明设备有在线状态。这里要看是否实际收到回包。看到 `via DERP(...)` 表示用了中转，不等于失败；先把连接用起来，不必一开始追求直连。

## 第二步：先让 SSH 真正登录进去

**在哪操作：目标电脑开 SSH 服务，当前电脑配置登录。**

Tailscale 提供网络，OpenSSH 提供远程登录。本教程使用“普通 OpenSSH 走 Tailscale 地址”，不需要额外启用 Tailscale SSH 功能。

按目标系统照着 [SSH 配置步骤](references/ssh-setup.md) 做一次。里面分别写了 Ubuntu 和 Windows 的服务、公钥、指纹以及客户端别名，已经做好的步骤可以跳过。

完成后，在当前电脑运行：

```text
ssh company whoami
ssh classroom whoami
```

只测试你已经配置的目标。输出应是那台远端电脑的用户，例如 Windows 会显示 `机器名\用户名`。出现这个结果，才说明你真的登录到了目标机器。

**想建立反向连接：** 换到另一台电脑，按相同方法把原来的当前电脑作为目标。每台电脑保留自己的私钥，把公钥交给需要登录的目标；不用把私钥复制来复制去。

## 第三步：确认远端的 Codex 能在 SSH 里运行

**在哪操作：先在目标电脑准备，再从当前电脑验证。**

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

只检查已配置的目标。**远端桌面能运行、SSH 中却说找不到 codex，通常要查 SSH 环境里的 PATH。** 不要先重装所有东西。Windows 可以进一步运行 `ssh classroom "where.exe codex"`。

## 第四步：把远端电脑的聊天接进当前 Codex

**在哪操作：你眼前这台电脑的 Codex 桌面客户端。**

1. 打开 **Settings → Connections（设置 → 连接）**，找到 SSH 主机。
2. 添加或启用前面配置的 company、classroom 等别名，等待连接成功。
3. 需要添加远程项目时，选择目标电脑上真实存在的目录，例如 Ubuntu 的项目目录，或 Windows 的 `C:\Users\账户\Documents\项目`。
4. 回到侧边栏／聊天列表，找到这台远端电脑上的已有聊天，点开一条熟悉的，确认能看到它原来的记录和当前状态。
5. 按同样方法加入其他电脑。它们的聊天也会出现在当前客户端中，你可以在这些聊天之间直接切换。

**这一步的完成标志是：远端电脑的聊天已经在当前界面里显示，并且可以点开阅读。** 仅看到连接开关亮起，还要继续确认聊天是否加载成功。不同版本的布局和分组方式可能不同；查看聊天的主机或运行位置标识，可以确认它属于哪台电脑。[官方 SSH 接入说明](https://learn.chatgpt.com/docs/remote-connections#connect-to-an-ssh-host)给出了连接设置的入口。

如果终端 SSH 已经成功，而 Codex 仍显示连接失败，请看 [SSH 好了，Codex 还连不上](references/remote-computers.md#ssh-好了codex-还连不上)。若连接成功却找不到旧聊天，继续看下面的排查方法。

## 第五步：直接看进度，在原聊天里下指令

### 点开远端聊天，就能看它在做什么

在同一个聊天列表里，点开公司电脑正在运行的任务，查看当前状态、已有输出和后续回复。再点开教室电脑上的任务，就切换到了另一台电脑的工作。

同名聊天可以通过所属主机和项目区分。网络断开时，界面里保留的旧记录不代表最新进度，需要恢复连接后再查看。

### 想补充要求，就在这条聊天的输入框里写

找到要继续的那条聊天后，直接在它的输入框里输入并发送。你的要求会进入原来的对话，沿用它的上下文，由对应电脑上的 Codex 处理；后续进度和结果也显示在这里。

例如，公司电脑正在处理资料，你可以在那条聊天中补一句“下一步先检查表格中的单位”。发完后切换到教室电脑的聊天，继续处理另一件事。日常查看和下达指令都在这些聊天本身完成。

### 要开新任务，先选执行它的电脑

在目标电脑的项目中创建新聊天，确认主机和项目路径，再输入任务。继续旧任务时直接打开旧聊天，这样前面讨论过的要求还在同一段对话里。

### 已经接入，为什么某些历史聊天没有显示？

先确认目标连接已启用，再检查侧边栏当前的主机、项目分组或筛选范围。接着核对 SSH 是否登录了原来使用 Codex 的系统用户，以及远端 Codex 服务使用的数据目录。Windows 用户改名后，个人目录可能仍是旧名称；不同账户或不同 `CODEX_HOME` 也可能读到不同历史。

这里的“显示到一起”是由当前客户端集中展示各个主机的聊天，记录仍由各自的主机保存。某台电脑的旧聊天缺失时，应排查它所连接的用户、服务和历史来源。具体步骤见 [让远端聊天显示出来](references/codex-chats.md)。

## 第六步：离开电脑前，确认它能一直被找到

公司或教室的电脑需要开机、联网、保持唤醒。网络服务会开机启动，并不意味着睡眠中的电脑还能接任务。

Windows 如果希望注销后 Tailscale 仍在线，可在托盘图标的 **Preferences → Run unattended** 开启无人值守模式，具体见 [Tailscale 官方说明](https://tailscale.com/docs/how-to/run-unattended)。这只解决网络在线；Codex 的桌面、沙箱或某些 Windows 适配方式可能仍需要对应用户保持登录，要分别确认。

离开前从另一台电脑做一次完整检查：能登录 SSH、能打开远端聊天、能收到远端回复。短暂断网后先重连原来的主机、查看原聊天状态，再决定是否重试；不要直接把同一任务再发一遍。

## 不想自己逐项配置，就把 Skill 交给 Codex

Skill 的名字沿用仓库的 `codex-workspace-management`，界面显示为“Codex 多电脑聊天互联”。装在你准备用来管理其他电脑的那台机器上即可；换另一台机器来管理时，再在那台安装。

Ubuntu／macOS，需要 Git：

```bash
mkdir -p "$HOME/.agents/skills"
git clone https://github.com/zn099/codex-workspace-management.git \
  "$HOME/.agents/skills/codex-workspace-management"
```

Windows PowerShell，需要 Git：

```powershell
$skillRoot = Join-Path $env:USERPROFILE '.agents\skills'
New-Item -ItemType Directory -Force $skillRoot | Out-Null
git clone https://github.com/zn099/codex-workspace-management.git (Join-Path $skillRoot 'codex-workspace-management')
```

已经安装过就更新原目录，不再克隆一份。在没有未处理的本地修改时，可以进入技能目录执行 `git pull --ff-only`。技能发现方式见 [官方说明](https://learn.chatgpt.com/docs/build-skills#where-to-save-skills)。

接着在 Codex 新聊天中说：

```text
$codex-workspace-management
我想让公司和教室电脑的 Codex 聊天一起显示在当前电脑的界面里，
能直接点开看工作进度，并在原聊天中继续输入指令。
请检查已有 Tailscale 和 SSH 配置，完成接入，并确认远端聊天可见。
需要我在另一台电脑操作的部分，请说清在哪台、做什么、成功后看到什么。
```

接入后，如果有聊天没有显示，也可以让 Skill 帮你排查：

```text
$codex-workspace-management
教室电脑的 SSH 已经连通，但它原来的聊天没有出现在当前 Codex 中。
请检查连接的用户、Codex 服务和聊天显示范围，修复后确认旧聊天能打开。
```

## 卡住了，从这几个地方找

| 现象 | 先看哪里 |
| --- | --- |
| Tailscale 在线，ping／SSH 却超时 | [在线不等于连得通](references/remote-computers.md#tailscale-在线为什么还是超时) |
| Windows 写公钥时拒绝访问 | [管理员 PowerShell 和公钥文件](references/ssh-setup.md#目标是-windows) |
| SSH 报 publickey 错误 | [用户名、密钥与授权](references/remote-computers.md#permission-denied-publickey) |
| SSH 可以，Codex 不行 | [远端启动环境](references/remote-computers.md#ssh-好了codex-还连不上) |
| 连上后找不到某条聊天 | 回到第五步，核对主机、账户、数据目录和客户端能力 |
| 时好时坏，一直重连失败 | [断线与重连](references/remote-computers.md#一会儿能连一会儿又断) |

本教程整理自 Ubuntu 与 Windows 多机联调。使用时需要客户端提供 SSH 连接和远端聊天功能；配置示例使用占位值，不包含个人设备、账号或密钥。社区项目，采用 [MIT 许可](LICENSE)。

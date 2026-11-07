# 连不上时，先看是哪一步没过去

我们最终想用的是远端聊天。排查时先找出停在哪一步：Tailscale 回包 → SSH 登录 → 远端 Codex 启动 → 聊天列表与后续指令。前一步已经证明能用，就从后一步查起。

## Tailscale 在线，为什么还是超时？

先在当前电脑运行，记得替换地址：

```text
tailscale ping --c 5 <目标地址>
tailscale ping --tsmp --c 5 --timeout 4s <目标地址>
```

第一条默认用 DISCO，第二条走加密数据通道。第一条有回应、第二条超时，是可能出现的情况；这时不能因为页面显示在线就去改 SSH 用户名。[Tailscale 官方说明](https://tailscale.com/docs/reference/ping-types)解释了两种测试的区别。

如果 TSMP 也有回应，再试：

```text
ssh -v -T -o BatchMode=yes -o ConnectTimeout=15 <别名> whoami
```

看它是连不上 22 端口，还是已经到了认证阶段。调试输出可能含主机名和路径，对外求助时去掉个人信息，别附带密钥或认证文件。

Ubuntu 目标端可检查：

```bash
systemctl status tailscaled ssh --no-pager
ss -ltn
ip route
ip rule
```

Windows 目标端可检查：

```powershell
Get-Service sshd
Get-NetTCPConnection -LocalPort 22 -State Listen
```

再按证据检查端点防火墙和 Tailscale 访问规则。允许普通 OpenSSH 流量需要相应的网络访问许可；仅修改 Tailscale 的 SSH 登录策略，不会修好 Windows OpenSSH 的公钥认证。

## Permission denied (publickey)

说明已经到了 SSH 认证阶段。按这个顺序核对：

1. `ssh -G 别名` 解析出的地址和 User，是否与目标电脑的 whoami 一致。
2. IdentityFile 是否是当前电脑准备使用的私钥；目标端有没有它对应的公钥。
3. 私钥有口令时，当前客户端的 SSH agent 是否已解锁它。BatchMode 不会弹出口令询问。
4. Windows 目标账户是否属于管理员组；授权文件路径和权限是否正确。

Windows 写公钥提示“拒绝访问”，先确认 PowerShell 真正提升了权限。看到 System32 不是证据；具体步骤见 [SSH 配置](ssh-setup.md)。已提升仍失败时读取文件 ACL 和所有者，不反复追加或直接放开 Everyone。

不用为这个错误重新登录 Tailscale，也不要每次失败就生成一把新密钥。

## SSH 好了，Codex 还连不上

先从当前电脑测试：

```text
ssh <别名> "codex --version"
```

若找不到命令，Windows 检查 `where.exe codex`，Ubuntu 检查 `command -v codex`；都要在 SSH 登录的环境里检查。普通桌面终端和 SSH 非交互会话的 PATH 可能不同。

若版本能输出，就看 Codex 客户端连接日志。比如日志先显示 publickey 认证成功，接着报 `sh` 找不到，问题已经到了远端启动脚本：

- 确认目标系统和默认 shell，不要把 Linux 脚本直接当 Windows 命令运行。
- 核对当前客户端和远端 Codex 的版本、受支持的连接方式。
- 只有日志证明需要时，才为特定连接做兼容处理；不要照抄一段历史包装器改变全局 DefaultShell。
- Windows 上，服务能够启动但沙箱命令失败时，还要检查启动用户、交互会话和日志。服务进程存在不代表聊天可以正常工作。

这类问题曾在 Ubuntu 连接 Windows 时实际出现过。具体适配取决于版本，因此教程先给可判断的步骤，不把某个版本的补丁当成人人都要安装的组件。

## 一会儿能连，一会儿又断

如果其他 SSH 连接也在相近时间断开，先看当前电脑的 Wi-Fi、网关和睡眠事件。几条 SSH 连接可以同时工作，它们不会因为连接数量多了一个就自然互相冲突。

再运行 `tailscale netcheck`，结合双方的服务日志、实际直连／中转路径和断线时间判断。终端设置了代理，并不证明 tailscaled 守护进程用了同一个代理；先核对实际环境。

某个中转区域不可达、另一个可达时，可以在确认当前版本参数和地区编号后做临时对照。先说明影响，保留原状态和撤销方法；只有回包、SSH 和 Codex 都改善了，才报告相应结果。不要让所有人都固定用同一个地区。

自动重连只能应对短暂中断。一直超时、公钥拒绝或启动脚本失败时，反复重连只是重复同一个错误。对照实验无改善就按方案撤销，不无限重启服务。

**不要为了试试看中断唯一的远程通道。** 还有 RustDesk 等备用通道就先留着；重启网络、改路由或可能断开现有任务的操作，先说明并取得用户必要确认。

修复后可以持续观察两分钟左右：记下成功次数、超时次数、延迟、实际路径，并分别测试新建 SSH 和一条持续会话。既有 SSH 不断线，不代表新连接也稳定。不要用一次成功来宣布完全修好。

## 聊天列表是空的，或者任务像“失忆”了

回到[教程第五步](../README.md#第五步开始管理不同电脑上的聊天)，检查主机、用户和实际数据目录。再核对客户端支持的聊天读取能力。

在远端新启动 `codex` 通常会开始一个新会话；它不会自动接上你选定的桌面聊天。CLI 的 `codex resume` 可用于其能读取的已保存会话，但不能据此承诺接管另一服务中正在运行的聊天。要继续原任务，优先从客户端或受支持的聊天工具找到原来的那一条。

断网后先查看原聊天是否仍在执行，再补发要求。没有确认第一次投递失败时，不重复启动同一个任务。

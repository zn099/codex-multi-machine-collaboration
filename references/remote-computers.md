# 远程电脑：按层定位，再做最小修复

用于 Ubuntu／Windows 上的 Tailscale、OpenSSH 和 Codex 连接。示例需替换为用户核实的设备；不预设版本、用户名、中转地区、管理员身份或 Codex 提供方。

## 明确方向与身份

核对双方 OS、主机名、用户、网络地址、SSH 端口、远端项目路径及备用通道。实际设备表留在本地项目，不写入公共 Skill。

“A 能登录 B”不代表“B 能登录 A”：两个方向各有服务端、公钥授权、主机指纹和网络规则。SSH 别名只在保存它的客户端生效；`ssh person` 中的 person 不会自动变成另一台机器的用户名。

保留并检查现有配置。用 `ssh -G ALIAS` 查看实际解析，核对 HostName、User、Port、IdentityFile、ProxyJump/ProxyCommand，以及 Include／Match 的作用。Windows 用户改名后 profile 路径可能仍是旧名，须实际读取。

匹配和使用既有私钥，不输出其内容；公钥注释仅是标签，最终按公钥内容／指纹核对。生成新密钥依据实际需要，不作为连接重试。

首次连接从可信远端终端或管理通道核对主机指纹。ssh-keyscan 只能获取候选密钥，不能单独证明可信。未知或变化的指纹无法核实时展示给用户确认，不关闭全局主机校验。

## 按当前错误定位

| 观察结果 | 下一步检查 | 不能据此断言 |
| --- | --- | --- |
| 管理页面在线，连接超时 | 控制状态与实际数据通道分别验证 | SSH 服务正常、远端可执行 |
| 默认 ping 成功，TSMP 超时 | 加密通道、回包、握手、策略、路径 | 已经排除网络问题 |
| TSMP 成功，TCP 22 超时 | OS 路由／策略路由、监听、端点防火墙、sshd | 一定是密钥错误 |
| SSH 报 publickey 拒绝 | 真实用户、提供的公钥、授权文件与 ACL | 需要切换中转 |
| 登录成功，codex 找不到 | SSH 非交互环境 PATH、实际可执行文件 | 桌面终端能运行就足够 |
| CLI 成功，桌面连接失败 | 版本、启动 shell、应用日志、会话权限 | SSH 未连通或必须重装 |
| 多条 SSH 同时断开 | 网关、Wi-Fi、休眠、共享出口时间线 | 两个别名互相冲突 |

默认 Tailscale ping 使用 DISCO；TSMP 经加密 WireGuard 隧道但绕过主机 IP 栈，仍需实际 SSH 验证。依据：[Tailscale ping 类型](https://tailscale.com/docs/reference/ping-types)。

## 只读诊断起点

先查版本和 `--help` 确认参数存在。以下在连接发起端运行，先替换变量：

```bash
remote_ip='REPLACE_WITH_VERIFIED_TAILSCALE_IP'
remote_alias='REPLACE_WITH_EXISTING_SSH_ALIAS'
tailscale status
tailscale netcheck
tailscale ping --c 5 "$remote_ip"
tailscale ping --tsmp --c 5 --timeout 4s "$remote_ip"
ssh -G "$remote_alias"
ssh -T -o BatchMode=yes -o ConnectTimeout=15 "$remote_alias" whoami
```

主机指纹仍按上一节核验。执行工具应提供合理总超时并记录失败阶段。默认 ping 在等待直连模式下，即使收到中转 pong 也可能返回非零；报告分别记录回包、路径和是否直连，不只依赖退出码。

Ubuntu 按需检查 `systemctl status tailscaled ssh --no-pager`、`ss -ltn`、`ip route`、`ip rule`、服务日志及 NetworkManager／休眠事件。部分检查需要 sudo，由用户在本机输入密码。区分终端代理与 tailscaled 服务实际环境，避免输出代理 URL 内的密码。

Windows 核对 whoami、Get-Service sshd、22 端口监听、OpenSSH 日志、对应防火墙规则与会话。管理员组账户默认使用 `%ProgramData%\ssh\administrators_authorized_keys`，最终以实际 sshd 配置为准。

写授权文件前确认 PowerShell 已提升权限；提示符在 System32 不代表管理员权限：

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

拒绝访问时先查提升状态、目标 ACL／所有者，不反复追加、不直接 takeown 或放开所有人权限。追加公钥保留其他授权；调整 ACL 前确认所需主体，可用 SID 避免语言差异。依据：[Microsoft：Windows OpenSSH 配置](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)与[密钥管理](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)。

## 最小修复与中转对照

根据证据一次改一个相关变量，比较变更前后的同类测试。按用户备份要求保留相关配置；没有要求时遵守项目版本约定。事先明确恢复办法，不因此复制整个项目或认证目录。

中转不是故障本身。若证据显示某中转不可达，可核实当前版本的临时诊断选项、地区编号与撤销方式，在授权范围内对照。不要固定某个地区永远最好，不把临时 debug 设置称为持久修复，不自动添加开机任务。

重启 Tailscale、切网、改路由或代理可能影响现有远程任务；可能切断唯一通道时先说明具体影响，等待必要确认。保留可用备用通道，不通过退出账号、重置密钥、关闭整个防火墙或开放公网 SSH 来试运气。

自动重连只能帮助短暂中断，不能修复持续不可达或认证失败。使用有限重试与间隔，保留失败阶段；不无限重启服务，不随重试重复启动同一项目任务。

## 验证 Codex 执行位置

登录后先确认远端用户、OS、实际 profile 和工作目录。对能接受这些命令的 Windows 远端 shell，可依次验证：

```bash
ssh -T "$remote_alias" whoami
ssh -T "$remote_alias" 'where.exe codex'
ssh -T "$remote_alias" 'codex --version'
ssh -T "$remote_alias" 'codex login status'
```

遇到非标准 shell／启动适配器时先核对实际行为。login status 与当前认证方式结合解释；不自动把自定义提供方改成 ChatGPT 登录，不读取秘密内容。

交互式 CLI 需要终端，例如 `ssh -t ALIAS`，登录后进入已确认的项目目录再运行 codex。非交互任务采用当前版本支持的命令接口，先明确任务范围；连通性验收不启动修改真实项目的模型任务。

桌面连接额外验证远端启动环境，优先遵循[官方 SSH 接入方法](https://learn.chatgpt.com/docs/remote-connections#connect-to-an-ssh-host)。不要默认套用历史 Windows Bash 包装器或计划任务；只有实际版本和日志证明需要时才适配。SSH 登录、app-server 可达、聊天可见、沙箱命令可执行是不同验证层次。

## 不稳定问题的验收

按用户要求持续测试；未指定时，可用至少两分钟、约每十秒采样作为起点。按故障需要分别检查网关／本地链路、TSMP、新建 SSH 和一条持续 SSH。

报告成功数／总数、超时阈值、延迟范围与统计口径、实测直连或中转路径。持续会话稳定不能替代新建连接成功率；在线状态不能替代两者。包装器或编码错误与网络失败分开说明。

保留“已恢复但仍有丢包”“根因尚未定位”等真实结论。改动后恢复是对照结果，不自动证明具体设备或运营商是根因。

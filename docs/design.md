# Agent Touch Bar 设计文档

状态：草案，待用户确认，2026-11-15

目标：把 Codex 和 Claude Code 的会话状态显示在 M1 MacBook Pro 的 Touch Bar 上。

资料来源：事实来自检查这台 Mac（macOS 27.0.1，Apple M1）。检查时没有网络，没有查官方文档。标注“公开资料，待 M0 实录确认”的内容来自已有知识，没有在本机验证。第 18 节把已验证和待验证的内容分开列出。

## 1. 背景、目标、非目标

### 背景

同时开几个 agent 时，要逐个切窗口才知道谁在等批准，谁已经跑完。Claude Code 和 Codex 都支持 hook，也就是 agent 在固定时点运行的命令。这台 Mac 上，iTerm2 已经通过 Claude Code 的 hook 读取会话状态。Touch Bar 常驻在键盘上方，不占屏幕空间。

### 目标

- 在 Touch Bar 上显示所有 agent 会话的状态。
- 有会话在等你时，显示得最醒目。
- 点一下会话，跳到它所在的终端。
- 不改变 agent 的行为：hook 不阻塞，不输出内容，不替你做权限决定。
- 数据不出本机，不保存提示词和工具的输入输出。

### 非目标（第一版不做）

- 在 Touch Bar 上批准或拒绝。
- 显示 token 或费用。
- 没有 Touch Bar 的 Mac。以后可以加菜单栏版本。
- 通过 SSH 运行的 agent。
- 其他 agent。适配层留了位置，例如 Gemini CLI。

### 术语

| 术语 | 含义 |
|---|---|
| 会话 | 一个 agent 的一次运行，由 agent 名加 `session_id` 确定 |
| 事件 | 一次 hook 触发，经 `agentbar hook` 整理成的一行记录 |
| 事件文件 | 所有事件追加写入的 `events.jsonl` |
| 状态 | 会话的五种状态：等你 `waiting`、出错 `error`、运行中 `working`、完成 `done`、空闲 `idle` |

五个状态的英文名沿用 iTerm2 的 `it2 handle-agent-hook`。

## 2. 使用场景

1. 终端里开着 3 个 agent，其中一个在等我批准一条命令。我看 Touch Bar 就知道，不用切窗口。
2. 一个长任务在跑，我在用别的 App。它完成或出错时，Touch Bar 上的点会变色。
3. 点一下 Touch Bar 上的会话，回到它所在的 iTerm2 面板。
4. 用 Codex 桌面版时，也能看到它在运行还是已完成。这一条取决于 M0 的结果（第 4.3 节）。

## 3. 方案总览

三个部件：

- `agentbar hook`：被 agent 的 hook 调用。读 stdin 的 JSON，整理成一行，追加到事件文件，立刻退出。
- `AgentTouchBar.app`：后台常驻。监视事件文件，维护每个会话的状态，显示在 Touch Bar 上。
- `AgentBarCore`：纯 Swift 库，放事件整理、状态逻辑和安装时的配置合并。前两个部件共用。

hook 一侧只追加，不做判断，所以很快，也不依赖 app 是否在运行。判断都在 app 一侧。app 重启后，从事件文件的尾部重建状态。

```
Claude Code ───hook──┐
Codex CLI ─────hook──┼─> agentbar hook ──追加一行──> <用户临时目录>/agent-touchbar/events.jsonl
Codex 桌面版 ──hook?─┘                                        │ 文件变化通知
                                                              ▼
(M5 兜底) ~/.codex/sessions/**/rollout-*.jsonl ──读取──> AgentTouchBar.app（后台）
                                                    normalize → reducer → aggregator
                                                              ▼
                                          Control Strip 项 / 展开条 → 点按跳转终端
```

Touch Bar 的两种形态（示意）：

```
收起：[ ……前台 App 的按钮…… ] [●●◌ 3] [☀︎] [🔊] [Siri]
展开：[✕] [● agent-touchbar  Claude  等你批准 Bash  2m] [● api  Codex  运行中  3m] [● web  Claude  完成]
```

“●” 的颜色表示状态，见第 8 节。

## 4. 事件来源

### 4.1 Claude Code

版本 2.1.325，装在 `~/.local/bin/claude`。`~/.claude/settings.json` 里已经有这些事件的 hook：Notification、PermissionRequest、PostToolUse、PreToolUse、SessionEnd、SessionStart、Stop、StopFailure、SubagentStop、UserPromptSubmit。每条都没有 `matcher`，命令是 iTerm2 的 `it2 handle-agent-hook`。这些是本机检查所得。

安装后，在每个事件的数组里追加我们的一条，保留 it2 那条。以 Stop 为例：

```json
"Stop": [
  {"hooks":[{"type":"command","command":"\"/Users/torreswang/.config/iterm2/it2\" handle-agent-hook"}]},
  {"hooks":[{"type":"command","command":"\"<agentbar 绝对路径>\" hook --agent claude"}]}
]
```

stdin 里的字段（公开资料，待 M0 实录确认）：

- 公共字段：`session_id`、`transcript_path`、`cwd`、`hook_event_name`，多数事件还有 `permission_mode`。
- 工具事件：`tool_name`、`tool_input`。PostToolUse 另有 `tool_response`。
- Notification：`message` 和 `notification_type`，例如 `permission_prompt`、`idle_prompt`。
- UserPromptSubmit：`prompt`。

### 4.2 Codex CLI

版本 0.169.0-alpha.11.1，二进制在 `/Applications/ChatGPT.app/Contents/Resources/codex-cli/CodexCLI.app/Contents/MacOS/codex`。`hooks` 功能是 stable，已启用。现在没有配置任何 hook：`~/.codex/hooks.json` 不存在，`config.toml` 里没有 `[hooks]`。

下面的信息来自二进制里的字符串。它们是线索，不是文档，具体写法和行为要在 M0 实录。

- 事件名（括号里是字符串出现次数）：SessionStart (74)、PermissionRequest (60)、PreToolUse (59)、PostToolUse (52)、SessionEnd (37)、SubagentStop (33)、PreCompact (32)、UserPromptSubmit (18)。
- `Stop` 只出现一次，旁边有 `stop_hook_active` 和 `run_turn_stop_hooks`，很可能存在。
- 没有找到 `StopFailure`、`PostToolUseFailure`、`Notification`。所以 Codex 的 hook 报不出错，也没有 `idle_prompt` 这类通知。
- 载荷里的字段名：`session_id`、`turn_id`、`permission_mode`、`transcript_path`、`hook_event_name`、`tool_use_id`、`last_assistant_message`、`stop_hook_active`。
- 配置里的事件键用蛇形命名：`session_start`、`pre_tool_use`、`post_tool_use`、`permission_request`、`user_prompt_submit`、`session_end`、`subagent_stop`、`pre_model_request`、`pre_compact`、`post_compact`。
- 处理器字段：`command`、`commandWindows`、`timeout`、`async`、`statusMessage`、`additionalContextLimit`。
- hook 可以写在 `~/.codex/hooks.json`，也可以写在 `config.toml` 的 `[hooks]` 下。
- 信任记录在 `hooks.state."<key>".trusted_hash`。推测新 hook 要你确认后才会运行（M0 确认）。另有一个 `bypass_hook_trust` 选项，我们不用。

iTerm2 的 `it2 handle-agent-hook` 有 `--agent codex` 选项，说明它已经能解析 Codex 的 hook JSON。M0 可以参考它读取哪些字段。

Codex 的 `config.toml` 里有一项 `notify = ["…/SkyComputerUseClient", "turn-ended"]`，是文件里唯一的非信任类设置，Computer Use 依赖它。我们不替换它。

### 4.3 Codex 桌面版与 rollout 兜底

桌面版是 ChatGPT.app（bundle id `com.openai.codex`，注册了 `codex://`）里的 Codex。它是否运行 hook 没有验证。CLI 二进制就放在这个 App 里，两者可能共用引擎，但没有证据，M0 实测。

桌面版里，工具进程的环境变量有 `CODEX_THREAD_ID` 和 `CODEX_SESSION_ID`（值相同，都是对话 id）、`CODEX_SANDBOX=seatbelt`、`CODEX_SANDBOX_NETWORK_DISABLED=1`、`CODEX_VERSION`、`CODEX_CI=1`。没有 `TERM_PROGRAM`、`__CFBundleIdentifier`、`ITERM_SESSION_ID`、`TMUX`。判断规则：有 `CODEX_*` 且没有 `TERM_PROGRAM`，就当作桌面版。hook 进程是否拿到同样的环境变量，M0 确认。

如果桌面版不运行 hook，就读 rollout 文件（M5）：

- 位置：`~/.codex/sessions/YYYY/MM/DD/rollout-<开始时间>-<对话 id>.jsonl`。日期目录是会话开始那天。跨天的会话仍写在原目录，所以要监视最近修改过的文件，不能只看今天的目录。
- 记录类型有 `session_meta`、`response_item/*`、`event_msg/*`、`token_usage_record`、`world_state`、`compacted`。
- 只读 `session_meta`（取 `id` 和 `cwd`）和三种 `event_msg`：`task_started`、`task_complete`、`turn_aborted`。丢弃 `last_agent_message` 和其他所有内容。
- 现有 5 个文件里没有出现审批类事件，所以兜底方案显示不了等你。可能只是这些会话没遇到审批，M0 再查。
- 有一个文件里所有记录的 `timestamp` 完全相同，所以兜底方案用读到记录的时间，不用记录里的时间戳。
- rollout 含完整的提示词、推理和工具输出。我们不存，也不展示。
- `~/.codex/session_index.jsonl` 每行有 `id`、`thread_name`、`updated_at`，可以用 `thread_name` 给 Codex 会话起名。

### 4.4 不采用的来源

- 扫描进程或终端内容：脆弱，也会读到不该读的内容。
- Codex 的 `notify`：只有一个位置，Computer Use 已经占用，而且只在回合结束时触发。

## 5. 事件传输

### 5.1 `agentbar hook`

保留的字段（白名单）：`session_id`、`hook_event_name`、`tool_name`、`notification_type`、`message`（只取 Notification 的，截到约 120 字符）、`cwd`、`turn_id`、`permission_mode`。

不保存：`prompt`、`tool_input`、`tool_response`、`last_assistant_message`，以及对话记录的内容。

自己加的字段：

- 格式版本 `v`、agent 名、时间戳。
- `ppid`：`getppid()` 的结果。
- `agent_pid`：沿父进程往上找到的 agent 进程。用 libproc 按可执行文件名识别 `claude` 或 `codex`，识别方式 M0 确认。找不到就不写这个字段。
- 环境变量 `TERM_PROGRAM`、`__CFBundleIdentifier`、`ITERM_SESSION_ID`、`TMUX`、`TMUX_PANE`，有 `CODEX_THREAD_ID` 时也记下来。只记存在的，放在 `env` 对象里。

一行事件的样子（示意，字段名以实现为准）：

```json
{"v":1,"agent":"claude","ts":"2026-11-15T10:20:30.123Z","session_id":"<id>","hook_event_name":"PermissionRequest","tool_name":"Bash","cwd":"/Users/torreswang/Projects/agent-touchbar","ppid":4242,"agent_pid":4200,"env":{"TERM_PROGRAM":"iTerm.app","ITERM_SESSION_ID":"w0t0p0:<UUID>"}}
```

行为约束：

- 不向 stdout 输出任何内容。SessionStart 和 UserPromptSubmit 的 stdout 会进入模型的上下文。
- 永远以 0 退出。非 0 退出可能影响 agent 的流程。
- 50 ms 内结束。任何失败都静默丢弃这条事件。
- 只写一行 JSON，小于 1 KB，用一次 `O_APPEND` 写入。超过 1 KB 时先丢 `message`，再截断 `cwd`。
- `--record` 选项保留完整载荷，用来在 M0 录制测试数据。它默认关闭，写到事件文件同目录的 `record.jsonl`（0600），用完删除。

### 5.2 事件文件

- 路径：`confstr(_CS_DARWIN_USER_TEMP_DIR)` 返回的目录下的 `agent-touchbar/events.jsonl`。这台 Mac 上是 `/var/folders/gg/fj3g0kbd0n15bfmlrgmg4wqh0000gn/T/`（`/var` 指向 `/private/var`）。用 confstr 而不是 `$TMPDIR`，因为 hook 进程的环境变量不一定带 `$TMPDIR`。
- 这个目录是 Codex 沙箱的可写目录之一（本机检查），所以沙箱里的 hook 也能写，不需要建立连接。hook 本身是否在沙箱里运行，M0 确认。
- 权限：目录 0700，文件 0600。
- app 用 DispatchSource 监视文件变化。启动时读文件尾部重建状态。
- 文件超过大小上限（初定 1 MB）时，由 app 用重命名轮转。app 启动时和运行中都检查大小。轮转后再读一次旧文件的尾部，避免丢事件。hook 每次都重新打开文件，所以轮转不会让它写进已经改名的文件。

不采用的传输方式：

- Unix socket：沙箱可能禁止连接。
- 本机 HTTP：Codex 沙箱默认关闭网络（`CODEX_SANDBOX_NETWORK_DISABLED=1`）。
- Darwin 通知：不带数据。

## 6. 事件整理（AgentBarCore）

- `normalize(agent, line) -> AgentEvent`：把事件文件的一行变成统一的事件。坏行、未知事件、不认识的 `v`、超长行直接忽略。
- reducer：按 agent 加 `session_id` 维护会话。每个会话有 agent、id、项目名、状态、详情、最后事件时间、终端信息和 pid。
- 项目名取 `cwd` 的最后一段。Codex 会话可以改用 `thread_name`。
- aggregator：把所有会话汇总成收起态的摘要，和展开态的列表。

这一层是纯 Swift，没有 AppKit，是整个项目的主要测试接缝。

## 7. 状态模型

优先级：等你 > 出错 > 运行中 > 完成 > 空闲。汇总和排序都按这个顺序。

| 事件 | agent | 新状态 |
|---|---|---|
| SessionStart | 两者 | 空闲（登记会话） |
| UserPromptSubmit | 两者 | 运行中 |
| PreToolUse | 两者 | 运行中，详情为工具名 |
| PostToolUse | 两者 | 运行中（清除等你） |
| PermissionRequest | 两者 | 等你 |
| Notification，`permission_prompt` | 仅 Claude | 等你 |
| Notification，`idle_prompt` | 仅 Claude | 完成，或不变 |
| Stop | 两者（Codex 很可能有） | 完成 |
| StopFailure | 仅 Claude | 出错 |
| Interrupt | Codex（未验证） | 空闲 |
| SubagentStop | 两者 | 只更新详情 |
| SessionEnd | 两者 | 删除会话 |

其他类型的 Notification 忽略。

rollout 兜底的映射（只有 Codex 桌面版用）：

- `session_meta`：登记会话，记下 `cwd`。
- `task_started`：运行中。
- `task_complete`：完成。
- `turn_aborted`：空闲。
- 兜底方案显示不了等你。

完成和出错变回空闲的条件，满足任意一个即可：

- 你在展开条里点按了这个会话。
- 这个会话来了新的提示词。
- 过了 10 分钟（可配置）。

第一版不检测你是否手动切到了那个终端。

存活检查：每 30 秒对记录的 `agent_pid` 做 `kill(pid, 0)`，进程已经不存在就删除会话。只有 `ppid` 的会话不做存活检查，因为那个进程可能是 hook 的父 shell，很快就退出。这种会话靠 SessionEnd 删除，或者 24 小时没有事件后清理。

状态可能卡住：

- 等你：你拒绝了权限，agent 可能不发任何事件。你批准后工具要跑很久，到 PostToolUse 之前也没有新事件。
- 运行中：你按 Esc 中断时，agent 可能不发 Stop（公开资料，待 M0 确认）。

任何后续事件都会更新状态。展开条里同时显示这个状态持续了多久，以及距上次事件过了多久，由你自己判断。批准或中断之后有没有事件，M0 要查。

## 8. Touch Bar 交互

### 收起态

Control Strip 里的一个项。Control Strip 是 Touch Bar 右侧的系统控制区，放亮度、音量和 Siri。

- 显示最多 4 个彩点，加会话总数。彩点按状态优先级排序，等你在最前。
- 颜色：等你是琥珀色，会脉动。运行中是蓝色，带转圈。完成是绿色。出错是红色。空闲是灰色。
- 没有会话时显示一个灰色空心点，让你知道 app 还活着。
- 可用宽度待 M0 测。

### 展开态

点收起态的项，展开成系统模态条：

- 左边是关闭框（✕）。
- 每个会话一个按钮：状态色点、项目名、agent、状态文字、持续时间。
- 放不下时用可滚动的列表，细节在 M3 定。
- 点会话按钮：跳到它的终端（第 9 节），并收起。

有会话进入等你时，默认只脉动，不自动展开，免得抢占 Touch Bar。状态变化能不能唤醒变暗的 Touch Bar，M0 验证。

### 实现方式（公开资料，待 M0 实录确认）

这是 MTMR 和 Pock 这类项目用的做法：

1. 用 `dlopen` 和 `dlsym` 加载 DFRFoundation。SDK 里没有私有框架的 TBD 文件，不能直接链接。
2. 在一个小的 ObjC SwiftPM target 里，用分类声明私有方法，例如 `+[NSTouchBarItem addSystemTrayItem:]` 和 `+[NSTouchBar presentSystemModalTouchBar:placement:systemTrayItemIdentifier:]`。Swift 一侧只调用。
3. 注册：创建自定义项，调用 `addSystemTrayItem:`，再调用 `DFRElementSetControlStripPresenceForIdentifier`。
4. 点按展开用 `presentSystemModalTouchBar:placement:systemTrayItemIdentifier:`，关闭用 `dismissSystemModalTouchBar:` 或 `minimizeSystemModalTouchBar:`。
5. 唤醒后或 TouchBarServer 重启后，重新注册。

这台 Mac 上这些符号都存在（见第 18 节），但符号存在不等于项能显示。这条路在 macOS 27 上是否还通，是整个方案最大的不确定点，M0 第一个验证。

app 是 `LSUIElement` 应用（不在 Dock 里显示），由 LaunchAgent 启动，ad-hoc 签名。

## 9. 跳转终端

按事件里记下的环境变量判断会话在哪个终端：

| 终端 | 识别 | 跳转方式 |
|---|---|---|
| iTerm2 | `TERM_PROGRAM=iTerm.app`，有 `ITERM_SESSION_ID` | `it2 session focus <id>`，或用 AppleScript 按 id 选中会话 |
| tmux | 有 `TMUX` 和 `TMUX_PANE` | `tmux select-pane`，再激活外层终端 |
| Ghostty、VS Code、Cursor | `TERM_PROGRAM` 和 `__CFBundleIdentifier` | 按 bundle id 激活 App，不定位到具体面板 |
| Codex 桌面版 | 有 `CODEX_*`，没有 `TERM_PROGRAM` | 打开 ChatGPT.app。单个对话的深链格式未知 |

待确认：

- `ITERM_SESSION_ID` 形如 `w0t0p0:<UUID>`，`it2 session focus` 要完整值还是只要 UUID。
- VS Code 和 Cursor 的 `TERM_PROGRAM` 可能相同，需要靠 `__CFBundleIdentifier` 区分。
- tmux 的 `TMUX` 变量里有 socket 路径，调用 `tmux` 时要带上它。

## 10. 安装与卸载

`agentbar install`、`agentbar uninstall`、`agentbar status`。

安装：

1. 把 `agentbar` 复制到固定位置，建议 `~/.local/bin/agentbar`（`claude` 也在这个目录）。hook 里写它的绝对路径。
2. 备份 `~/.claude/settings.json`，再对上面 10 个事件各追加一条我们的处理器。已有就不重复。保留 it2 的条目和其他所有内容。
3. Codex 一侧只写 `~/.codex/hooks.json`（不存在就新建，存在就合并），不碰 `config.toml`，也不碰 `notify`。只为我们用到的事件加处理器，不设 `statusMessage`，`timeout` 设成几秒。
4. 写 LaunchAgent 的 plist 并加载。
5. 提示你在 Codex 里确认新 hook（确认入口待 M0 确认）。不使用 `bypass_hook_trust`。

写配置文件时先写临时文件再重命名，保持原文件权限。合并后 JSON 的字段顺序和空白可能变化，内容不变，原文件在备份里。`--dry-run` 只显示差异，不写文件。

`agentbar status`：显示各 agent 的 hook 是否已装、app 是否在运行、事件文件最后一条事件的时间。

卸载：只删我们的条目（按命令里的 `agentbar` 路径识别），卸载 LaunchAgent。备份保留。

动手改你的配置之前，会先问你。

## 11. 安全与隐私

- 不联网。
- 不保存提示词、工具输入、工具输出和对话内容。事件里只有第 5.1 节的白名单字段。
- 载荷当作不可信数据：用正经的 JSON 解析，截断超长字段，显示前去掉控制字符，永远不拼进 shell 命令。
- 跳转终端时，用参数数组调用 `it2` 和 `tmux`，不经过 shell。`ITERM_SESSION_ID` 先用正则校验，通过才使用。如果用 AppleScript，也只能插入校验过的值。
- `agentbar hook` 不输出任何权限决定。
- rollout 兜底只读 `session_meta` 和三种 `event_msg`，丢弃 `last_agent_message`。
- `--record` 会保留完整载荷，所以默认关闭，只在 M0 临时使用。测试数据由人手工去敏后才放进仓库。
- 事件文件在用户临时目录里，目录 0700，文件 0600。

## 12. 测试

- 主接缝：事件文件的若干行进去，会话快照出来（`AgentBarCore`）。用 `swift test`，输入是 M0 实录并手工去敏的测试数据。
- 要覆盖：状态表的每一行、未知事件、坏行、完成超时、存活检查（注入时钟和进程检查函数）、优先级汇总。
- 安装合并是纯函数：输入原配置和我们的条目，输出新配置。测试幂等、保留 it2 和其他内容、卸载只删我们的。
- `agentbar hook` 的黑盒测试：把测试数据管道进去，检查追加的那一行（白名单、截断、无 stdout、坏输入也退出 0）。M1 里再加耗时检查，目标 50 ms。
- Touch Bar 显示和跳转不能自动测，用手动清单：点的颜色、超过 4 个会话、点按展开和收起、休眠唤醒后项还在、TouchBarServer 重启后项还在、前台 App 设成显示 F 键时的表现、每种终端的跳转。

## 13. 代码结构

SwiftPM 项目：

```
agent-touchbar/
  Package.swift
  Sources/
    AgentBarCore/        事件整理、状态、汇总、安装合并（纯 Swift）
    CTouchBarPrivate/    小型 ObjC target，只放私有接口声明
    agentbar/            命令行：hook、install、uninstall、status
    AgentTouchBar/       AppKit 后台应用
  Tests/AgentBarCoreTests/
    Fixtures/            M0 实录并手工去敏的事件
  scripts/make-app.sh    把 AgentTouchBar 打成 .app，ad-hoc 签名
  docs/design.md
```

## 14. 里程碑

M0 对应 ask-matt 主线里的原型岔路（`$prototype`）。它是一次性的探路，不是第一版的代码。

`$prototype` 自带的两个模板是网页形式的：HTML 状态演示和网页界面变体。M0 的探路是 Swift 小程序，只借用它的规则：一次性、一条命令能跑、结论写回文档、原型留在 `prototype/<name>` 分支。第 7 节的状态表可以另做一个 HTML 状态演示（逻辑模板），用来试“等你卡住”这类边界情况。

- **M0 探路。** 回答第 18 节的待验证项，最重要的几项：
  - Control Strip 项在 macOS 27 上能否显示，点按能否展开。
  - 从 Claude Code、Codex CLI、Codex 桌面版实录真实的 hook 载荷。
  - Codex 桌面版是否运行 hook。
  - hook 是否在沙箱里运行，libproc 在那里是否可用，hook 是否带 `CODEX_THREAD_ID`。
  - 状态变化能否唤醒变暗的 Touch Bar。
  - 动手往 agent 的配置里加录制 hook 之前，会先问你。
- **M1 Claude Code 端到端。** `agentbar hook`、`AgentBarCore`、app，先只显示一个点。验收：一次真实的“提问、批准、完成”流程让点依次变色。
- **M2 Codex CLI。** 接入 Codex 的事件，补上两个 agent 的差异。
- **M3 多会话。** 展开条、点按跳转，先做 iTerm2。
- **M4 安装。** 安装器、卸载器、LaunchAgent、`status`。
- **M5 只在需要时做。** Codex 桌面版不运行 hook 时，用 rollout 文件兜底。

### 开发流程（按 ask-matt）

1. `$grill-with-docs`：逐条确认第 17 节的问题，结论留在 `GLOSSARY.md` 和 ADR 里。
2. M0 用 `$prototype` 做。换新会话前后用 `$handoff` 衔接，结论写回这份文档。
3. 这是要好几次会话的活：先 `$setup-matt-pocock-skills`，再 `$to-spec` 和 `$to-tickets`，每张票用 `$implement`，票之间清空上下文。
4. 做完后用 `$retro` 回顾。

## 15. 风险

- Apple 可能移除私有接口。
- Codex 是 alpha 版本，hook 的格式可能变，信任确认也增加了一步操作。
- Control Strip 的空间有限。
- 前台 App 设成显示 F 键时，Control Strip 会被隐藏。
- Touch Bar 空闲一段时间后会变暗（默认约 60 秒，M0 确认）。
- TouchBarServer 重启会让项消失，需要重新注册。
- 等你和运行中都可能卡住（第 7 节）。
- Codex 没有出错事件，显示不了出错。
- rollout 兜底没有等你状态，而且依赖 Codex 的内部格式。

## 16. 备选方案

| 方案 | 优点 | 缺点 | 结论 |
|---|---|---|---|
| 自写 app，用私有 Control Strip 接口 | 状态、展开、跳转都能自己控制 | 依赖私有接口，要自己维护 | 推荐 |
| BetterTouchTool | 不用写私有接口的代码 | 付费，要自己用脚本做展开和多会话（公开资料，未验证） | 私有接口走不通时的备选 |
| MTMR、Pock | 开源，MTMR 用配置文件加脚本定义按钮（公开资料，未验证） | 动态的多会话列表不好做 | 不选 |
| 菜单栏 app | 不用私有接口，所有 Mac 可用 | 不在 Touch Bar 上 | 以后给没有 Touch Bar 的 Mac 做 |
| 只用 iTerm2 自带的状态 | 已经有 | 只覆盖 iTerm2，不在 Touch Bar 上，看不到桌面版 | 不满足需求 |

## 17. 待决问题

每个问题后面是建议的答案。

| 问题 | 建议 |
|---|---|
| 支持哪些入口 | 先做命令行里的 Claude Code 和 Codex。M0 证明桌面版会运行 hook 时再加桌面版。 |
| Touch Bar 显示方式 | 私有 Control Strip 接口，BetterTouchTool 作备选。适合写成 ADR。 |
| 交互到什么程度 | 只看状态和点按跳转，不在 Touch Bar 上批准。 |
| 有会话进入等你时自动展开吗 | 默认不展开，只脉动。 |
| 完成状态保留多久 | 10 分钟，或你点按之后。出错用同样的规则。 |
| 安装器自动改 agent 配置吗 | 自动改，先备份并让你确认，可以先看 `--dry-run` 的差异。 |
| 保留 iTerm2 的 hook 吗 | 保留，和我们的并存。 |
| 要不要 `git init` | 建议要，先问你。`$prototype` 把原型放在 `prototype/<name>` 分支，`$implement` 提交到当前分支，`$code-review` 以固定起点比较，这几步都依赖 git。 |

## 18. 验证项

### 已验证（本机检查）

机器和 Touch Bar：

- macOS 27.0.1 (26A434)，Apple M1。
- `TouchBarServer` 在运行（pid 558），`dfr-drv` 和 `dispdfr` 存在。
- `com.apple.touchbar.agent`：`PresentationModeGlobal = appWithControlStrip`。`PresentationModeFnModes` 把 `appWithControlStrip` 和 `workflows` 都映射到 `functionKeys`。
- 用 ctypes 探测到这些私有符号存在。这只证明符号存在，不证明项能显示：
  - DFRFoundation 函数：`DFRElementSetControlStripPresenceForIdentifier`、`DFRSystemModalShowsCloseBoxWhenFrontMost`。
  - NSTouchBarItem 类方法：`+addSystemTrayItem:`、`+removeSystemTrayItem:`。
  - NSTouchBar 类方法：`+presentSystemModalTouchBar:placement:systemTrayItemIdentifier:`、`+presentSystemModalTouchBar:systemTrayItemIdentifier:`、`+minimizeSystemModalTouchBar:`、`+dismissSystemModalTouchBar:`。
  - 旧的 `presentSystemModalFunctionBar:` 系列不存在。
- 工具：`/Applications/Xcode.app`，以及 `/usr/bin` 里的 `swift`、`jq`、`python3`。
- 已装应用：iTerm2 3.7.7、Ghostty、VS Code、Cursor。没有装 BetterTouchTool、MTMR、Pock。
- 用户临时目录是 `/var/folders/gg/fj3g0kbd0n15bfmlrgmg4wqh0000gn/T/`，是 Codex 沙箱的可写目录之一。

Codex：

- CLI 版本、二进制位置、`hooks` 功能状态、当前没有 hook、`notify` 的内容：见第 4.2 节。
- 二进制里的事件名、字段名、处理器字段、配置位置、信任记录：见第 4.2 节。这些是字符串层面的线索。
- 桌面版工具进程的环境变量：见第 4.3 节。
- rollout 文件的位置、记录类型、`event_msg` 类型、`task_started` 和 `task_complete` 的字段：见第 4.3 节。现有 5 个文件里的 `event_msg`：`item_completed` 173、`token_count` 38、`thread_settings_applied` 8、`task_started` 8、`task_complete` 6、`turn_aborted` 1。
- `session_meta` 里有 `cwd`、`id`、`session_id`、`originator`、`source`、`thread_source`、`cli_version`、`git`、`timestamp`。`originator` 和 `source` 的取值没有查。

Claude Code 和 iTerm2：

- Claude Code 2.1.325。`~/.claude/settings.json` 里已有 10 个事件的 it2 hook，写法见第 4.1 节。
- `it2 handle-agent-hook [--agent claude|codex]` 从 stdin 读 hook JSON，把 iTerm2 会话的状态设成 waiting、done、error、working 或 idle。
- `it2 session focus <session-id>` 可以聚焦会话。`ITERM_SESSION_ID` 形如 `w0t0p0:<UUID>`。

### 待验证

| 项 | 为什么要知道 | 怎么验证 |
|---|---|---|
| Control Strip 项能否显示，点按能否展开（macOS 27） | 整个方案的前提 | M0 原型 |
| 私有接口的调用方式（MTMR、Pock 的做法，公开资料） | 决定第 8 节的实现 | M0 原型 |
| Control Strip 项的可用宽度 | 决定收起态能放几个点 | M0 原型 |
| 状态变化能否唤醒变暗的 Touch Bar | 决定是否需要别的提醒方式 | M0 原型 |
| Claude Code 各事件的真实载荷（公开资料） | 确定字段白名单 | M0，`--record` |
| Codex CLI 的 hook 写法和载荷 | 确定安装器写什么 | M0，`--record` |
| `Stop`、`Interrupt`、`SubagentStart` 在 Codex 里是否存在 | 完成和空闲的判断 | M0，`--record` |
| Codex 桌面版是否运行 hook，信任流程是否相同 | 决定要不要做 M5 | M0 |
| hook 是否在沙箱里运行，libproc 能否用，是否带 `CODEX_THREAD_ID` | 影响 pid 和会话标识 | M0，`--record` |
| 批准、拒绝或中断之后，到下一个事件之前有没有事件 | 影响等你和运行中会不会卡住 | M0，`--record` |
| 多个进程同时追加写，行会不会被拆开 | 事件文件的可靠性 | M0 压测 |
| Codex 的 `hooks.json` 和 `config.toml` 的 `[hooks]` 同时存在时怎么合并 | 安装器的写法 | M0 |
| `it2 session focus` 要完整的 `ITERM_SESSION_ID` 还是只要 UUID | 跳转 iTerm2 | M0 |
| VS Code 和 Cursor 的 `TERM_PROGRAM` 是否相同 | 跳转这两个 App | M0 |
| `codex://` 打开单个对话的格式 | 跳转 Codex 桌面版 | M0 |
| rollout 是否记录审批事件，`originator` 和 `source` 的取值 | M5 能否显示等你 | M0 |
| 官方文档里的 hook 格式 | 本文的 Claude Code 字段来自已有知识 | 联网后核对，可用 `$research` |

# Agent Touch Bar 方案可行性评估

日期：2026-11-15

评估对象：`docs/design.md`（2026-11-15 草案）

方法：联网核对了 Claude Code 和 Codex 的官方 hook 文档，查了做过同类事情的开源项目。没有在这台 Mac 上运行任何东西，涉及本机行为的结论仍要 M0 实测。Claude Code 的 hook 参考页很长，只按关键词查了相关小节，没有通读。Codex 的 hook 页读了主要小节。

## 结论

方案可行。三个部件的分工、事件文件的传输方式、安装器的思路都不用改。

最大的不确定点仍是 Control Strip 私有接口，但比设计文档估计的小。有人在同款机器（13 寸 M1 MacBook Pro）的 macOS 26 上用同样的三个调用跑通了，时间是 2026-05-31。macOS 27 上还没找到公开的验证，本机检查只证明符号都在。这一项半天内能出结论，应该是 M0 的第一件事。如果走不通，`AgentBarCore` 和 hook 一侧可以原样留给第 16 节的备选方案。

hook 一侧的风险比文档写的低。两家的官方文档已经回答了第 18 节里大部分 hook 相关的待验证项。另一个开源项目 AgentBar 用同样的 hook，在 macOS 27 上做了 Claude Code 和 Codex 的会话状态（菜单栏版），说明这条路走得通。

要改的主要是第 7 节的状态表：现在的表会漏掉几种“等你”，还有一条失败路径会让“等你”一直亮着。第 4.2 节关于 Codex 配置写法的一处判断也不对。细节见下文，第 5 节是逐条的修正清单。

## 1. Touch Bar 私有接口

### 已有证据

- clawd-touchbar 在 Touch Bar 上用动画角色显示 Claude Code 的状态。作者只在一台 13 寸 M1 MacBook Pro（2020）上测过。代码注释写明 2026-05-31 在 macOS Tahoe（26）上做过冒烟测试。
- 它用的正是第 8 节的三个调用：`+[NSTouchBarItem addSystemTrayItem:]`、`DFRElementSetControlStripPresenceForIdentifier`、`+[NSTouchBar presentSystemModalTouchBar:placement:systemTrayItemIdentifier:]`。
- `placement` 传 0 时，模态条只占 app 区域，右边的 Control Strip 保留。传 1 会盖住 Control Strip。
- 它每 60 秒重新调用一次 `DFRElementSetControlStripPresenceForIdentifier`，用来应对睡眠唤醒后项被挤掉。
- 它的 README 记了 Tahoe 上的一个问题：`ControlStrip` 进程有时会崩溃，项跟着消失，app 自己恢复不了，只能手动 `killall ControlStrip`。
- 它直接链接 DFRFoundation（`-F /System/Library/PrivateFrameworks` 加 `linkedFramework("DFRFoundation")`），用 Command Line Tools 就能编译。第 8 节说不能直接链接，至少在它的环境里不成立。本机能否直接链接，编译它一次就知道。`dlopen` 方案也可以，二选一。

### 还不知道

- macOS 27 上项能否显示。
- 点按收起态的项能否展开模态条。clawd-touchbar 一直显示模态条，没有点按展开这一步。Pock 这类项目用过这种交互（公开资料，没核对），也没找到它们在新系统上的近况。
- 收起态的项能有多宽。clawd-touchbar 的项是 30×30 pt 的方块。它的注释估算 13 寸机型 Touch Bar 总宽约 1085 pt，展开的 Control Strip 约占 250 pt。
- 状态变化能否唤醒熄灭的 Touch Bar。论坛资料说 Touch Bar 无操作约 60 秒后休眠。你在用电脑时它一直亮着，场景 2 没问题。你离开电脑时它帮不上忙，那时要靠 agent 自己的通知。

### 建议

- M0 第一步：先编译运行 clawd-touchbar（README 说 `make` 约 15 秒），看 macOS 27 上项和模态条会不会出现。它是 AGPL-3.0，只拿来验证，不要抄代码进本项目。
- 然后写自己的探针，测点按展开、项的宽度、睡眠唤醒，以及 `killall ControlStrip` 和 `killall TouchBarServer` 之后能不能重新注册。
- 第 8 节补两条：定时重新声明 presence；发现 ControlStrip 或 TouchBarServer 重启后重新注册。第 15 节补上 ControlStrip 崩溃这条风险。
- 关掉“显示 Control Strip”，或者 Touch Bar 设成显示 F 键时，项应该就看不到了，M0 顺手确认。`agentbar status` 可以读 `com.apple.touchbar.agent` 提示这一点。

## 2. Claude Code 一侧

### 官方文档已确认

- 公共字段：`session_id`、`prompt_id`、`transcript_path`、`cwd`、`permission_mode`、`hook_event_name`。`permission_mode` 不是每个事件都有。子代理里触发的事件多一个 `agent_id`。AgentBar 的实录显示，这时 `session_id` 仍是主会话的。
- Notification 有 `notification_type`、`message`、`title`。
- `StopFailure` 存在，回合因 API 出错结束时代替 `Stop` 触发。
- `Stop` 在用户中断时不触发。原文：“Does not run if the stoppage occurred due to a user interrupt.”
- 手动拒绝权限没有对应的事件。`PermissionDenied` 只在 auto 模式下触发。`PostToolUseFailure` 也不为被拒绝的调用触发。
- 取消正在运行的工具，不触发 `PostToolUseFailure`。
- `PermissionRequest` 在弹出权限提示时立即触发。`permission_prompt` 要等提示挂了约 6 秒、你没打字才触发。沙箱命令的网络请求只触发 `permission_prompt`。第 7 节把两者都映射成等你，是对的。
- `idle_prompt` 在 Claude 回答完约 60 秒、你没打过字、也没有后台子代理在跑时触发。
- 默认超时 600 秒。`UserPromptSubmit` 是 30 秒，`SessionEnd` 是 1.5 秒。
- 直接编辑设置文件里的 hook，会被文件监视自动加载，不用重启会话。
- `async: true` 只能用在 command 类型上。后台运行的 hook 不受 `timeout` 约束。

### 要改的地方

1. 加 `PostToolUseFailure`，映射为运行中并清除等你。工具失败时只发这个事件，不发 `PostToolUse`。文档的例子显示，Bash 非 0 退出也走这个事件，`error` 第一行是 `Exit code N`。按现在的表，你批准了一条命令而命令失败时，“等你”会一直亮到下一个事件。
2. 有些等你不经过权限请求。
   - `AskUserQuestion`（Claude 向你提问）：AgentBar 的记录说它不触发任何通知。官方文档没写清楚它会不会触发 `PermissionRequest`。最稳的做法是 `PreToolUse` 的 `tool_name` 为 `AskUserQuestion` 或 `ExitPlanMode`（计划模式下请你批准计划）时直接算等你，对应的 `PostToolUse` 再清掉。clawd-touchbar 对这两个工具都这样处理，AgentBar 对 `AskUserQuestion` 这样处理。
   - Notification 的 `elicitation_dialog` 和 `elicitation_url_dialog`：MCP 服务器在等你填表单或打开网页，也算等你。
3. 中断和拒绝确实没有事件，第 7 节说的“卡住”一定会发生。可以试一个恢复办法：如果中断或拒绝之后 Claude 算作“回答完”，约 60 秒后会来 `idle_prompt`，这时把卡住的运行中或等你改成空闲。会不会来，M0 实测。
4. 每条处理器写明确的 `timeout`，例如 2 秒。不用 `async`：hook 本身只要几毫秒，同步就够了；后台 hook 卡住时没有超时兜底。
5. 第 10 节可以写明：Claude Code 改完设置就生效，不用重启会话。

## 3. Codex 一侧

### 官方文档已确认

- 事件：
  - 回合内：`PreToolUse`、`PermissionRequest`、`PostToolUse`、`PreCompact`、`PostCompact`、`UserPromptSubmit`、`SubagentStop`、`Stop`。
  - 每次模型请求：`PreModelRequest`、`PostModelRequest`。
  - 中断进行中的回合：`Interrupt`。
  - 会话或子代理开始：`SessionStart`、`SubagentStart`。
  - 主线程结束：`SessionEnd`。
  - `Interrupt` 和 `SessionEnd` 不为子代理触发。`Interrupt` 也不为空闲线程触发。
- 配置的事件键是 PascalCase：`hooks.json` 里写 `"PreToolUse"`，`config.toml` 里写 `[[hooks.PreToolUse]]`。
- 多个来源的 hook 全部加载，高优先级的层不会覆盖低优先级的层。同一层里同时有 `hooks.json` 和 `[hooks]` 时，Codex 会合并，并在启动时警告。
- 信任按 hook 定义的哈希记录，新的或改过的 hook 在信任之前直接跳过。CLI 里用 `/hooks` 审核。文档讲插件 hook 时提到了桌面版的入口：在输入框处选 Review hooks，或者打开 Settings > Hooks。
- 公共字段：`session_id`、`transcript_path`（可为 null）、`cwd`、`hook_event_name`、`model`。回合内的事件另有 `turn_id`。子代理的 hook 用父会话的 `session_id`。
- 命令在会话的 `cwd` 下运行。
- 默认超时 600 秒。`SessionEnd` 和 `Interrupt` 默认 1 秒，最多 3 秒。
- `async: true` 的后台 hook 完成顺序可能和启动顺序不同。`SessionEnd` 总是同步运行。
- `PreToolUse` 和 `PostToolUse` 覆盖 shell、`apply_patch`、MCP 和其他本地函数工具，不覆盖 WebSearch 这类托管工具。
- `Stop` 以 0 退出时，Codex 期望 stdout 是 JSON，纯文本无效。`Interrupt` 可以没有输出。
- `PostModelRequest` 带 `status`（`completed`、`failed`、`cancelled`）和 `error`。它对每次请求尝试都触发，包括同一回合内的重试。
- hook 默认开启，关闭用 `[features] hooks = false`。

### 要改的地方

1. 第 4.2 节“配置里的事件键用蛇形命名”不对。蛇形名出现在信任记录的键里，AgentBar 在它的机器上看到的形式是 `hooks.state."<文件路径>:user_prompt_submit:0:0"`。安装器要写 PascalCase。
2. `Interrupt` 映射为空闲，第 7 节的“未验证”可以去掉。官方文档写的是当前版本，本机 0.169.0-alpha.11.1 里有没有这个事件，M0 要确认。`SubagentStart` 和 `PostModelRequest` 也一样。
3. 出错可以用 `PostModelRequest` 的 `status: failed` 判断。它包括重试，单次失败不能直接算出错，例如要等到回合结束而没有正常的 `Stop` 才算。具体规则看 M0 录到的序列。第 15 节“Codex 显示不了出错”可以改成“待 M0 确认”。
4. Codex 向你提问用 `request_user_input` 工具，会触发 `PreToolUse`（AgentBar 的记录）。和 Claude 的 `AskUserQuestion` 一样，直接算等你。
5. 桌面版有 hook 审核入口，说明桌面版会加载 hook。M5（rollout 兜底）大概率用不上，第 4.3 节的 rollout 部分可以降为备选。M0 只需在桌面版里实测一次：hook 有没有触发，hook 进程拿到哪些环境变量。
6. 信任和 hook 定义绑定。`agentbar` 的路径和参数要一次定死，以后升级只替换同一路径下的文件。AgentBar 就是这样做的，它的 README 说升级后 Codex 不会再要求批准。
7. `Stop` 期望 JSON，而我们的 hook 什么都不输出。AgentBar 的 helper 对所有事件都不输出，看起来可行。M0 顺手确认 Codex 不会对空输出报警告。如果报，就只对 `Stop` 输出 `{}`。
8. 读 stdin 要设上限。AgentBar 测过：如果写端一直不关管道，读到 EOF 才结束的 helper 会一直挂着，所以它限定 150 ms 没有新数据或总共 400 ms 就停。第 5.1 节的 50 ms 目标也要靠这种限制来保证。

## 4. 其他设计点

- 事件文件：保留。Codex 文档没说 hook 在沙箱里运行。AgentBar 的 Codex helper 能连本机端口，间接说明 hook 不在禁网的沙箱里。无论哪种情况，追加写用户临时目录都能用，而且 app 没在运行时事件也不会丢。
- `agent_pid`：按可执行文件名找 `claude` 可能找不到。Claude Code 原生安装时，`~/.local/bin/claude` 是指向按版本号命名的文件的链接，进程名可能显示成版本号。更稳的规则是取 hook 进程往上第一个不是 shell 的祖先，或者看可执行文件的路径。M0 用 `ps` 看一眼就知道。
- 终端信息：要确认 Codex 会把 `TERM_PROGRAM` 和 `ITERM_SESSION_ID` 传给 hook 进程。跳转 iTerm2 靠这两个变量。

## 5. 对 design.md 的修正清单

| 位置 | 现在的写法 | 建议改为 | 依据 |
|---|---|---|---|
| §4.1 | 字段是“公开资料，待 M0 实录确认” | 已由官方文档确认。公共字段另有 `prompt_id`，Notification 另有 `title` | Claude Code 文档 |
| §4.2 | 配置里的事件键用蛇形命名 | PascalCase。蛇形名只出现在信任记录的键里 | Codex 文档、AgentBar |
| §4.2 | `Stop` 很可能存在；Codex 报不出错 | `Stop` 已写进文档。`PostModelRequest` 可以报失败，本机版本待确认 | Codex 文档 |
| §4.3 | 桌面版是否运行 hook 没有验证 | 桌面版有 Review hooks 和 Settings > Hooks，大概率运行。M0 实测一次 | Codex 文档 |
| §7 | `Interrupt`（未验证） | 已写进文档，不为子代理和空闲线程触发 | Codex 文档 |
| §7 | 没有 `PostToolUseFailure`、`AskUserQuestion`、`ExitPlanMode`、`request_user_input`、elicitation 通知 | 补上，见第 2、3 节 | 两家文档、AgentBar、clawd-touchbar |
| §7 | 按 Esc 中断时 agent 可能不发 Stop | Claude 确认不发。Codex 发 `Interrupt` | 两家文档 |
| §8 | SDK 里没有 TBD，不能直接链接 | clawd-touchbar 能直接链接。`dlopen` 也可以 | clawd-touchbar 的 Package.swift |
| §10 | Codex 的确认入口待 M0 确认 | CLI 用 `/hooks`，桌面版用 Review hooks 或 Settings > Hooks | Codex 文档 |
| §10 | 只给 Codex 设几秒的 `timeout` | 两边每条处理器都写明确的 `timeout`。Codex 的 `SessionEnd` 和 `Interrupt` 不超过 3 秒 | 两家文档 |
| §15 | 无 | 补上：ControlStrip 进程崩溃后项消失 | clawd-touchbar |
| §18 | `hooks.json` 和 `[hooks]` 同时存在时怎么合并 | 全部加载。同一层两种都有时合并并在启动时警告 | Codex 文档 |

## 6. 建议的 M0 顺序

1. Touch Bar 探针。先做，它决定方案能不能走下去。内容见第 1 节。
2. Claude Code 实录。设置文件改完就生效。要录的序列：普通一问一答、批准、拒绝、思考时按 Esc、工具运行时按 Esc、`AskUserQuestion`、计划模式下批准计划、子代理、`/clear`。看拒绝和中断之后 60 秒内有没有 `idle_prompt`。
3. Codex 实录。手写 `~/.codex/hooks.json`（PascalCase），在 `/hooks` 里信任。确认 0.169.0-alpha 里有没有 `Interrupt`、`SubagentStart`、`PostModelRequest`；空输出的 `Stop` 有没有警告；hook 进程有哪些环境变量。再在桌面版里跑一遍。
4. 按第 3 步的结果决定 M5 做不做。

## 7. 已有项目

| 项目 | 做了什么 | 和本方案的差别 | 许可 |
|---|---|---|---|
| AgentBar（molodykhvitaliy/agent-monitor） | 菜单栏 app，用 hook 显示 Claude Code 和 Codex 每个会话的状态，需要你时发通知。要求 macOS 26 以上 | 不在 Touch Bar 上 | Apache-2.0 |
| clawd-touchbar（b-barri/clawd-touchbar） | Touch Bar 上的动画角色，跟着 Claude Code 的 hook 事件变化 | 只支持 Claude Code，多个会话时以最后一个事件为准，常驻占用 app 区域 | AGPL-3.0 |

两者都不满足“多会话、两个 agent、Touch Bar、点按跳转”，自己做仍有必要。AgentBar 的 `docs/dev/platform-integration.md` 逐条记录了它核对过的 hook 行为（核对日期 2026-09-02，对应 Claude Code 2.1.258、Codex CLI 0.152.1、macOS 27.0），写 M0 清单和安装器时值得对照。

## 资料

- [Claude Code hooks 参考](https://code.claude.com/docs/en/hooks)
- [Codex hooks（ChatGPT Learn）](https://learn.chatgpt.com/docs/hooks)
- [AgentBar README](https://github.com/molodykhvitaliy/agent-monitor)
- [AgentBar platform-integration.md](https://github.com/molodykhvitaliy/agent-monitor/blob/main/docs/dev/platform-integration.md)
- [clawd-touchbar](https://github.com/b-barri/clawd-touchbar)，其中 `Sources/Clawd/DFRBridging.swift`、`Sources/Clawd/TouchBarController.swift`、`Package.swift`
- [Apple 社区：Touch Bar 无操作 60 秒后休眠](https://discussions.apple.com/thread/7898009)
- [Macworld：macOS 27 只支持 Apple 芯片的 Mac](https://www.macworld.com/article/3139330/macos-27-mac-features-siri-apple-intelligence-release-date-compatibility.html)

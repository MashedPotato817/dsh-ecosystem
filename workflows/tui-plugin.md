# Workflow：DSH 终端 TUI 客户端（路线图第 3 项）—— 调研纪要

> 状态：调研完成（了解阶段），未开工

## 结论先行

DSH 的「TUI」不是一个小插件，而是一个**客户端应用**（与 Web 客户端平级）：
实现同一份 HTTP 契约（`ctx.remote`），把对话渲染到终端。工作量中-大，
但协议层完全复用官方契约设计，UI 层比 Web 简单得多。

## DSH 客户端契约（调研发现，来源：dsh-host-apiproxy / dsh-client-runtime / dsh-client-web）

- **传输**：HTTP。`POST /api/<method>`（rpcId 回显）= 客户端→服务端请求；
  服务端→客户端事件走 **SSE 帧**（mux 流）；`POST /api/respond` 回应服务端请求（审批/提问）。
- **领域方法**：
  - SessionsApi：create / prompt / history / list / rename / fork / models / cancel / updateQueue / search…
  - HostApi：openPath / pickDirectory / listDirectory…
  - EventsApi：mux 事件流
  - command.*（execute/list）、skill.*（list）、settings.*、credentials.*、llm.*、agentPreset.*、workspace.*
- **契约是 React-free、传输无关的**：官方 `AbstractApiClient` 持有全部协议不变量
  （rpcId 铸造、信封封装、zod 解析、SSE 解码、超时），任何客户端（Web/TUI/ACP）实现同一契约即可。
- **Web 客户端是完整插件树**（模块系统 + cordis loader + 插槽 + 一堆 UI 插件）——TUI 不需要这些，
  只需要「契约载体 + 会话模型 + 终端渲染 + 输入」。

## TUI 客户端要做什么（最小闭环）

1. **载体**：HTTP 客户端（fetch POST + SSE 消费），实现契约。
2. **会话模型**：session.list → session.history（含尾部 projections）→ session.prompt →
   消费 mux 事件流（user / assistant / tool / command 节点）。
3. **终端渲染**：对话区（消息流）+ 输入框（多行 + 斜杠命令）+ 会话列表（切换/续聊）。
4. **交互**：审批/提问（PendingInteraction：approval / plan-review / question）经 `POST /api/respond` 回应；
   斜杠命令经 `command.execute`。

## 技术选型（契约 React-free，客户端渲染随便用什么）

- **Ink**（React for CLIs，Claude Code 同款）——生态最成熟、组件化，**推荐**。
- blessed / neo-blessed——经典 TUI，更手工。
- 裸 ANSI——最可控、最费工。
- 参考：用户认可 Claude Code 的终端体验，而 Claude Code CLI 正是用 Ink 写的。

## 关键决策

1. **连谁**：连现有 `dsh web` host（3080 端口，复用 apiproxy + webserver）vs 独立 lean host profile。
   → 建议先连现有 host（host 侧零改动），后续需要再做独立 profile。
2. **包形态**：TUI 客户端是独立 npm 包（如 `dsh-tui`）；host 侧如需配合，插件很小。
3. **MVP 范围**：对话 + 输入 + 斜杠命令 + 会话列表；不做 settings / agent-preset UI。
4. **与 VSCode 集成的关系**：同一份契约；VSCode 集成可复用 TUI 的载体/会话模型层。

## 里程碑（草案）

1. 载体：HTTP 客户端连上 `dsh web` host，能 `session.list`。
2. 会话：打开会话、读 history、发 prompt、消费事件流。
3. 渲染：终端对话视图 + 多行输入 + 斜杠命令。
4. 交互：审批/提问回应、会话切换。
5. 测试 / CI / 发布。

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

- **Ink**（React for CLIs）——生态最成熟、组件化，**选定**。
- blessed / neo-blessed——经典 TUI，更手工（不选）。
- 裸 ANSI——最可控、最费工（不选）。

## 优秀实现调研结论（来源：子代理深挖 Claude Code / Codex / Juno）

| | Claude Code | Codex CLI | Juno |
|---|---|---|---|
| 技术栈 | TS + 深度定制 Ink/React | Rust + ratatui/crossterm | TS + React/Ink |
| 渲染引擎 | **自研 cell-diff**（7 阶段管线、双缓冲、blit、damage rect） | Layout 分区渲染 | 标准 Ink |
| 状态管理 | 组件树 | App(AppState+Mode) + `tokio::select!` | **冻结事件接缝**（AgentEvent → reducer） |
| 流式 | LRU token 缓存 + fast-path + Suspend 延迟高亮 | 事件驱动 | 流式 token + `<Static>` 冻结历史 |
| 键盘 | 多协议解析 + 显式 Mode + vim/emacs | Mode 枚举 | useInput |
| 权限/审批 | PermissionRequest 对话框 | — | risk 三级策略 + permission gate + workspace-jail |

**结论**：
- Claude Code 的「手感」= 自研 cell-diff 渲染引擎（工程量巨大，我们**不做**）；
- Codex = Rust 严谨分层（性能好，但换语言成本高，**不采用**）；
- **Juno 与我们要做的几乎同构**（TS+Ink+agent harness），是**最佳参考架构**。

**10 条最佳实践**（按重要性）：①单向数据流+冻结事件接缝 ②`<Static>` 冻结稳定内容只重绘流式尾部 ③流式=增量+缓存+延迟高亮 ④热路径抽离 React（diff/blit，后置）⑤多协议键盘+显式 Mode+可配置绑定（后置）⑥审批对话框+风险分级+diff 预览 ⑦工具调用状态机可视化 ⑧斜杠命令 palette ⑨会话持久化+resume ⑩终端能力探测+降级+生命周期清理

**架构路线**：**「Juno 架构 + Claude Code 组件拆分 + Codex 显式 Mode」**。
**MVP 落地子集**：①②③⑧⑨⑩（事件接缝/reducer、`<Static>` 冻结、流式增量、斜杠 palette、会话 resume、生命周期清理）。
**后置**：cell-diff 引擎、vim/emacs 多协议键盘解析——等性能/体验告警出现再说。

## 关键决策

1. **连谁**：连现有 `dsh web` host（3080 端口，复用 apiproxy + webserver）vs 独立 lean host profile。
   → 建议先连现有 host（host 侧零改动），后续需要再做独立 profile。
2. **包形态**：TUI 客户端是独立 npm 包（如 `dsh-tui`）；host 侧如需配合，插件很小。
3. **MVP 范围**：对话 + 输入 + 斜杠命令 + 会话列表；不做 settings / agent-preset UI。
4. **与 VSCode 集成的关系**：同一份契约；VSCode 集成可复用 TUI 的载体/会话模型层。

## 里程碑（按最佳实践子集落地）

1. **载体**：HTTP 客户端连上 `dsh web` host，能 `session.list`。（契约载体，参考官方 `AbstractApiClient` 设计）
2. **会话 + 事件接缝（①）**：打开会话、读 history、发 prompt、消费 mux 事件流 → 判别联合事件 → reducer → 状态。
3. **渲染（②③）**：终端对话视图（`<Static>` 冻结历史 + 流式增量渲染尾部）+ 多行输入 + 斜杠命令 palette（⑧）。
4. **会话管理（⑨）**：会话列表/切换/续聊/resume。
5. **交互 + 清理（⑥⑩）**：审批/提问回应（接 DSH 审批 seam）、终端能力探测/降级、进程退出清理。
6. **测试 / CI / 发布**。

后置：cell-diff 引擎、vim/emacs 多协议键盘（⑤④）。

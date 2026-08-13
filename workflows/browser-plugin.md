# Workflow：dsh-tool-browser —— 原生浏览器自动化工具插件

> 状态：进行中（里程碑 1-6 完成，19 工具，本机 Edge 与 CI Chromium 双绿）｜ 对应路线图第 2 项「正路径」

## 0. 结论：有没有人做过？

- 官方 200 个 `@deepseek-ai/dsh-*` 包里：无浏览器工具 ✅（已核实）
- npm：`dsh-tool-browser` / `dsh-browser-plugin` / `dsh-playwright` 等名字全部空闲 ✅（已核实）
- dsh-plugin topic / awesome 列表：无浏览器自动化插件 ✅（生态扫描 + 二次检索）
- **结论：没人做过，值得做。**

## 1. 目标

给 DSH 提供**原生**浏览器自动化工具（区别于已接入的 Playwright MCP「快路径」）：
不依赖 MCP 中间层，深度集成 DSH 的 tool 注册表 / 审批 / 结果结构化，更可控、可发布、可扩展。

**浏览器用本机 Edge**（`channel: 'msedge'`），不下载 Chrome（用户本机已有 Edge，已验证 MCP 走 Edge 可行）。

## 2. 项目要点（技术落点）

- **形态**：与 `dsh-git-plugin` 完全同构 —— 独立文件夹 + git 仓库 + npm 包，
  `name / inject / Config / apply`，注入 `tools` + `systemPrompt`（照 `dsh-tool-fs-search` 的 defineTool 写法）。
- **依赖**：`playwright` npm 包（连接 Edge，无需自带浏览器）。
- **工具集**（原生实现，命名与 Playwright MCP 对齐，方便迁移）：
  - 核心 5 件：`browser_navigate` / `browser_click` / `browser_type` / `browser_take_screenshot` / `browser_snapshot`
  - 扩展：`browser_evaluate` / `browser_console_messages` / `browser_network_requests` / `browser_wait_for` / `browser_tabs` / `browser_fill_form` / `browser_press_key` / `browser_select_option` / `browser_resize` / `browser_handle_dialog` / `browser_file_upload` / `browser_drop` / `browser_find` / `browser_hover` / `browser_drag`
- **浏览器生命周期**：按会话共享 context；默认 `--headless`，可配置 headed（调试）；每工具转发 `exec.signal` 中止 + `timeoutMs`，禁止挂起 agent 回合。
- **安全**：默认只允许 http/https；截图/快照结果结构化、限长（考虑 KV cache 影响）。

## 3. 里程碑（要做的事情）

1. ✅ **脚手架**：新文件夹 `game/dsh-tool-browser` + git init（`feat/*` 分支）+ package.json + apply 骨架 + 测试框架。
2. ✅ **Playwright 连通**：`playwright` 库 + `channel: 'msedge'` 启动 Edge，验证能打开页面（含回退到自带 Chromium）。
3. ✅ **核心工具**：navigate / click / type / screenshot / snapshot / evaluate / back / press_key / console / wait_for 共 10 个 —— 「打开网页 → 交互 → 截图」端到端测试通过（真实 Edge）。
4. ✅ **扩展工具集**：19 个工具（新增 `browser_fill_form` / `select_option` / `hover` / `drag` / `resize` / `open_tab` / `tabs` / `switch_tab` / `close_tab`，浏览器会话升级为多标签页管理）。
5. 📌 **安全与生命周期**：context 管理、超时/中止（已部分实现）、权限审批、结果限长（已部分实现）。
6. 📌 **测试 + CI**：smoke + 真实 Edge 集成测试已写并通过；GitHub Actions 待推送验证。
7. 📌 **发布**：GitHub Release + npm publish（名 `dsh-tool-browser`，已确认可用）。

## 4. 重点（focus，防止分心）

- **先跑通最小闭环**（打开页面 + 截图），再扩展工具，不要一上来铺 24 个工具。
- **复用 `dsh-git-plugin` 的全部工程模式**：分支工作流、MAA commit、smoke + 集成测试、CI、GitHub Release + npm。
- **Edge 优先**，不依赖下载 Chrome；`channel: 'msedge'`。
- **每个工具必须处理中止与超时**：`exec.signal` + `timeoutMs`，否则会挂死 agent 回合。
- **结果结构化 + 限长**：截图走文件/spill，快照裁剪，控制 KV cache / token 影响。
- 与 MCP 并存：MCP 已可用（快路径），原生插件（正路径）发布后两者可共存、可对照。

## 5. 待决策

- 浏览器实例：共享 vs 按会话隔离（默认按会话，避免互相干扰）。
- 是否需要 headful（调试）配置项。
- 工具参数 schema 的具体形状（照 `defineTool` 的 parameters/output 写法）。

## 6. 验收标准

- 在真实 Edge 上跑通「打开网页 → 点击/输入 → 截图」完整流程（集成测试）。
- 15+ 个工具测试全绿，CI 通过。
- npm 发布 `dsh-tool-browser`，`dsh plugin add dsh-tool-browser` 可安装。
- 用户亲自在 DSH 里用一遍（按稳定判定标准），再合并 main。

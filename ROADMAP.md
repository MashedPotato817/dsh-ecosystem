# DSH 生态路线图（程序员手感补全计划）

目标：把 DeepSeek Harness（DSH）相对 Claude Code / Codex 缺失的「程序员手感」补全：
**Git 工作流 → 浏览器自动化 → 终端 TUI → VSCode 集成**。

## 背景：DSH 架构速记

```
DSH Host（本地进程：agent / 工具 / 插件）
   │  统一客户端契约：ctx.remote（React-free，HTTP + mux 流）
   ├── Web 客户端（dsh-client-web，已交付 = 当前 GUI）
   ├── TUI 客户端（官方蓝图预留 "future TUI"，未交付）
   ├── ACP（Agent Client Protocol，IDE 集成的入口）
   └── VSCode 客户端（本路线图第 4 项）
```

「嵌合 DSH」= 新增一个 client face（客户端面），DSH 侧复用现成契约；需要 DSH 侧配合时，
写一个小 host 插件（与 `dsh-git-plugin` 相同的 `apply / inject / Config` 写法）。

## 项目清单

### 1. ✅ dsh-git-plugin —— Git 工作流插件（已完成）

- **状态**：已发布。npm `dsh-git-plugin@0.1.0` + GitHub Release v0.1.0，CI 通过。
- **内容**：5 个斜杠命令（`/status` `/diff` `/branch` `/commit` `/undo`）+ 4 个只读工具
  （git-status / git-diff / git-log / git-show）+ `preCommit` 提交前钩子 + 多仓库自动发现。
- **仓库**：https://github.com/MashedPotato817/dsh-git-plugin

### 2. 📌 浏览器自动化（新项目）

- **目标**：让 DSH 能打开网页、点击、填表、截图、执行 JS —— 直接可用于测试网页游戏
  （blockcraft 等 HTML/JS 项目）。
- **技术落点**：
  - 快路径：**Playwright MCP**（已接入 web profile：`@playwright/mcp` + 本机 **Edge**
    （`--browser msedge`，无需下载 Chrome），24 个浏览器工具，实测已通过：打开页面 + 截图 OK）
  - 正路径：原生 `dsh-tool-browser` 工具插件（Playwright 直连，无 MCP 中间层，更可控）
- **优先级**：高（与用户的网页游戏项目直接相关）
- **状态**：MCP 探路完成（Edge 实测通过）；原生工具插件未开始

### 3. 📌 终端 TUI 插件

- **目标**：终端里像 Vim / Claude Code 一样用 DSH（斜杠命令、多行输入、会话列表、彩色输出）。
- **技术落点**：无 React 的 `ctx.remote` 客户端 + 终端渲染，作为新 profile 组合
  （`dsh-base` + TUI 客户端 bundle）。
- **优先级**：中（官方蓝图预留 "future TUI"，社区无人占位）
- **状态**：未开始

### 4. 📌 VSCode 集成

- **目标**：VSCode 里原生面板用 DSH（Diff 视图、选中代码→提问、工具结果点击打开文件）。
- **技术落点**：
  - 方案 A（快）：VSCode 扩展嵌入 DSH Web 客户端（webview 套壳，体验弱）
  - 方案 B（正）：**参考 Claude Code 的 VSCode 扩展形态**——DSH CLI 跑在 VSCode
    集成终端里（用户认为 Claude Code 的终端体验不错），扩展提供原生 UI
    （Diff 视图、审批提示、工具结果点击打开文件）；DSH 侧加 `dsh-host-vscode`
    host 插件（启动 / 端口协商 / 权限）
- **优先级**：中（方案 B 优先，A 兜底）
- **状态**：未开始

## 协作约定

- 每个项目独立文件夹、独立 git 仓库（本文件夹只放计划与文档）。
- 分支工作流：开发在 `feat/*` 分支，验证稳定后合并 `main`。
- Commit 消息：MAA 风格 `<类型>(<可选作用域>): <中文主体>`。
- 完成标准：GitHub Release + npm 发布（如适用）+ 收入 `dsh-plugin` 生态。

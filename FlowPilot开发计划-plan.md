# FlowPilot — 开发计划（plan.md）

> AI-Native SDLC 桌面应用 · Python + PySide6
> 版本：v0.1.0 目标 · 日期：2026-09-02
> 对应设计稿：`FlowPilot界面设计/` 00–08（Warm Paper 米色主题，1440×900）

---

## 1. 产品概述

FlowPilot 是一个以 **Jira 任务为主线** 的 AI 驱动研发流水线桌面应用。每个 Jira Issue 对应一条 Plan → Design → Build → Test → Deploy 流水线，每个阶段产物化（intent.md → spec.md → plan.md → diff → PR），**每个阶段之间都有人工 Review 门禁**，Agent 永远不能独自越过一个门禁。

核心设计决策（已在设计稿中定稿，开发时不得偏离）：

| 决策 | 内容 |
|---|---|
| Agent 执行器 | 不调模型 API，**shell out 到 GitHub Copilot CLI / Claude Code CLI**，复用其登录态、模型与额度 |
| MCP 归属 | MCP Server 只在 CLI 侧配置（如 `~/.copilot/mcp.json`），FlowPilot **只读镜像展示**，永不写入 |
| 多仓库 | 一个任务可绑定多个仓库；Design/Build/Test/Deploy 四个阶段的 UI 与产物都按仓库分组 |
| 阶段 Skills | 每个阶段进入时自动挂载该阶段的默认 skill 集合（存于 Pipeline Template） |
| 本地状态 | 任务状态与产物存 `~/work/.flowpilot/`，产物文件同时写入对应 git 仓库 |
| UI 主题 | "Warm Paper" 米色浅色主题，单一陶土主色 + sage/ochre/brick 三个低饱和语义色 |

---

## 2. 技术栈

| 层 | 选型 | 说明 |
|---|---|---|
| UI 框架 | **PySide6**（≥6.7） | `FramelessWindowHint` 无边框标题栏、QSS 实现 Warm Paper 主题 |
| 窗口骨架 | QMainWindow + QStackedWidget + QSplitter | 侧边栏 / 页面切换 / 详情页双栏 |
| Markdown 预览 | `markdown` + QTextBrowser（或 QWebEngineView 备选） | intent/spec/plan 实时预览 |
| Diff 展示 | `unidiff` + 自绘 QPlainTextEdit 行着色 | Build 阶段变更审查 |
| 终端流 | QPlainTextEdit + QProcess 增量追加 | Build/Test 的 CLI 日志流 |
| CLI 调用 | **QProcess**（异步，stdout/stderr 分行读取） | 调用 copilot/claude CLI，支持中断 |
| 本地存储 | SQLite（`sqlite3` 标准库）+ 文件系统 | 任务状态机 + 产物文件 |
| Git 操作 | **GitPython** | 分支创建、worktree、commit、状态检测 |
| MCP 镜像 | `watchdog` 监听 `~/.copilot/mcp.json` | 只读解析，变更时刷新设置页 |
| Jira 数据 | 经由 CLI 的 Jira MCP（间接）；仅展示层缓存 | 不在应用内直连 Jira API |
| 配置/日志 | `platformdirs`、`logging` + RotatingFileHandler | |
| 打包 | PyInstaller（--windowed）或 briefcase | 出 macOS/Windows/Linux 桌面包 |
| 测试 | pytest + pytest-qt | 状态机与 UI 冒烟 |

**禁止事项**：不在应用内配置 MCP Server、不存任何模型 API Key、不绕过人工门禁自动推进阶段。

---

## 3. 架构设计

```
┌─────────────────────────── FlowPilot (PySide6) ───────────────────────────┐
│  UI 层（QSS · Warm Paper）                                                  │
│   ├─ TasksDashboardPage      首页：任务列表 + 迷你流水线（设计稿 01）         │
│   ├─ TaskDetailPage          五阶段详情容器：顶部 PipelineRail + SubSteps    │
│   │   ├─ PlanStageWidget        02  Jira 源面板 + intent.md 预览 + 门禁      │
│   │   ├─ DesignStageWidget      03  对话窗 + spec.md 实时预览（按仓库分节）  │
│   │   ├─ BuildStageWidget       04  plan.md 按仓库分组 + 执行监控 + Hooks    │
│   │   ├─ TestStageWidget        05  evals 按仓库分组 + 终端日志 + 失败门禁   │
│   │   └─ DeployStageWidget      06  PR 按仓库、合并顺序、审计轨迹、合并门禁  │
│   ├─ SettingsPage            07  CLI 检测 / MCP 只读镜像 / 阶段 Skills      │
│   └─ NewTaskDialog (QDialog) 08  Jira 选单 + 多选仓库 + 执行器 + 模板       │
│                                                                             │
│  核心层（纯 Python，可无头单测）                                             │
│   ├─ TaskStore        SQLite：任务、阶段状态机、审计事件（append-only）      │
│   ├─ PipelineEngine   阶段状态机：locked → ready → running → review → done │
│   ├─ AgentRunner      QProcess 封装：组装 prompt（阶段 skill + 产物上下文） │
│   │                   调用 copilot/claude CLI，流式回传，支持 cancel        │
│   ├─ ArtifactManager  intent/spec/plan.md 的生成、版本化、写入 git           │
│   ├─ RepoManager      GitPython：多仓库注册、分支、worktree、脏检查          │
│   ├─ HookGuardrails   执行前注入 .flowpilot/hooks.json 定义的钩子            │
│   └─ McpMirror        watchdog 监听 CLI 的 mcp.json → 只读模型供 UI 展示     │
│                                                                             │
│  外部依赖：Copilot/Claude CLI（Agent + MCP host）、git、GitHub（PR/CI）      │
└─────────────────────────────────────────────────────────────────────────────┘
```

**数据流（以 Plan 阶段为例）**：
用户创建任务 → PipelineEngine 置 Plan=running → AgentRunner 以 `jira-intent` skill + Jira MCP 拉取结果组装 prompt 调用 CLI → 产出 intent.md → ArtifactManager 落盘并 v1 → 阶段进入 `review`（门禁锁定）→ 人工 Approve → commit intent.md → 解锁 Design。

**关键边界**：UI 只发意图（"批准"、"退回"），状态迁移全部走 PipelineEngine；Agent 的所有输出先落产物文件，再进 UI——UI 崩溃不影响执行。

---

## 4. 数据模型（SQLite · `~/work/.flowpilot/flowpilot.db`）

```sql
tasks        (id, jira_key, title, type, priority, template_id,
              current_phase, phase_state, created_at, updated_at)
task_repos   (task_id, repo_path, branch_name, worktree_path)   -- 多仓库绑定
artifacts    (id, task_id, kind,          -- intent|spec|plan
              version, repo_path, file_path, git_sha, created_at)
phase_events (id, task_id, phase, event,  -- append-only 审计：approved/rejected/rerun…
              actor, detail_json, created_at)
skills       (id, name, phase, source, enabled)                  -- 阶段默认 skills
repos        (path, name, last_seen_status)
```

产物文件同时写入业务仓库（如 `docs/flowpilot/PROJ-1231/intent.md`），git 是唯一权威版本来源，SQLite 只做索引与状态。

---

## 5. 分阶段开发计划（里程碑制）

### M0 · 脚手架与设计令牌（第 1 周）
- [ ] PySide6 无边框窗口骨架：标题栏（拖动/最小化/最大化/关闭）+ 侧边栏 + QStackedWidget
- [ ] Warm Paper QSS 主题包：色板（#f4efe6/#fffdf8/#b4642c/#5e8c6a/#b98a2c/#b04a3a）、字体（Inter/JetBrains Mono）、圆角/阴影规范、组件类（panel/chip/btn/gatebar）
- [ ] `TaskStore` + SQLite schema + 迁移机制
- [ ] 打包冒烟：PyInstaller 出空壳可执行文件
- **验收**：空壳三平台启动，主题与设计稿 01 视觉一致

### M1 · 首页 + 新建任务（第 2 周）
- [ ] 设计稿 01：统计卡片、Tab 筛选、任务行（迷你 Pipeline、阶段徽标、负责人）
- [ ] 设计稿 08：NewTaskDialog——Jira 选单（先 mock，M2 接真）、多选仓库（RepoManager 注册表读取）、执行器选择、模板选择
- [ ] PipelineEngine 状态机 + 顶部 PipelineRail/SubSteps 组件
- **验收**：能创建任务并在首页看到迷你流水线，点击行进入详情页

### M2 · Plan 阶段打通（第 3 周）★ 第一个端到端闭环
- [ ] `AgentRunner`：QProcess 调 Copilot CLI，prompt = 阶段 skills + Jira 上下文模板
- [ ] `McpMirror`：解析 `~/.copilot/mcp.json`，设置页只读展示；Jira 拉取走 CLI 的 Jira MCP
- [ ] 设计稿 02：Jira 源面板 + intent.md 预览 + 人工门禁（Regenerate/Reject/Approve）
- [ ] ArtifactManager：intent.md v1 落盘 + git commit
- **验收**：从 Jira issue 到 approved intent.md 全链路可跑；断网/CLI 未登录有明确错误态

### M3 · Design 阶段（第 4 周）
- [ ] 设计稿 03：对话窗（QProcess 会话复用/新建）+ spec.md 实时预览（watchdog 监听产物文件变更 → 刷新预览）
- [ ] spec.md **按仓库分节**的文档结构约定 + 预览渲染分组色带
- [ ] 门禁：Export/Reject/Approve, go to Build
- **验收**：多轮对话能持续丰富 spec.md，预览实时更新，按仓库分节正确

### M4 · Build 阶段（第 5–6 周）★ 最重
- [ ] 设计稿 04：plan.md 按仓库分组生成 + 审批面板（三勾选确认项）
- [ ] Agent 执行：每仓库独立 worktree + 分支；步骤级进度回传（解析 CLI 输出约定标记）
- [ ] Hook Guardrails：`.flowpilot/hooks.json`（pre-edit/post-edit/pre-commit/pre-push）注入与拦截 UI
- [ ] Diff 审查视图（自绘行着色）
- **验收**：双仓库任务可执行 8 步构建，hooks 能拦截违规操作，中断恢复正确

### M5 · Test 阶段（第 7 周）
- [ ] 设计稿 05：evals 按仓库分组列表 + 深色终端日志流 + fail-fast 策略
- [ ] 失败决策门禁：Re-run failed / Waive（必填理由）/ Back to Build（回退状态机）
- **验收**：失败用例可回退 Build 再进 Test，状态机不丢审计事件

### M6 · Deploy 阶段（第 8 周）
- [ ] 设计稿 06：PR 按仓库卡片、合并顺序约束（依赖拓扑）、CI checks 轮询（GitHub API via CLI MCP）
- [ ] Jira 联动：末位 PR 合并后自动 transition + 评论（Jira MCP）
- [ ] 审计轨迹面板（phase_events 只读视图）
- **验收**：乱序合并被阻断；合并完成 Jira 状态自动流转

### M7 · 设置页 + 打磨 + 打包（第 9–10 周）
- [ ] 设计稿 07：CLI 检测（PATH 扫描/版本/登录态）、MCP 只读镜像、阶段默认 Skills 编辑、仓库管理
- [ ] 全局：错误态/空态/加载态、快捷键、通知中心
- [ ] 打包与签名、自动更新（可选）、日志收集开关
- **验收**：v0.1.0 三平台安装包；设计稿 01–08 全部落地

---

## 6. 风险与对策

| 风险 | 等级 | 对策 |
|---|---|---|
| Copilot/Claude CLI 输出格式不稳定，进度难以解析 | 高 | 与 CLI 约定结构化标记（如 `FLOWPILOT:{"step":3,"status":"done"}` 单行 JSON）；解析失败降级为纯日志流 |
| CLI 不支持长会话复用 | 中 | Design 对话改为"每次调用携带会话摘要 + spec 当前版本"的无状态模式 |
| MCP 配置格式随 CLI 版本变化 | 中 | McpMirror 做宽松解析 + 版本探测失败时显示"未知格式，请检查 CLI 版本" |
| 多仓库并发执行的文件锁/冲突 | 中 | 每仓库独立 git worktree；Hook 层串行化写操作 |
| 人工门禁被绕过（直接改库） | 低 | 门禁校验双写：SQLite 状态 + 产物 git commit 签名比对 |
| PySide6 复杂 QSS 跨平台渲染差异 | 低 | 锁定字体与像素尺寸；三平台截图走查纳入 M7 验收 |

---

## 7. 目录结构（建议）

```
flowpilot/
├─ app/
│  ├─ main.py                 # 入口：QApplication + 无边框窗口
│  ├─ theme/warm_paper.qss    # 主题令牌 + 组件样式
│  ├─ ui/
│  │  ├─ pages/               # dashboard / settings / 五阶段 widget
│  │  ├─ dialogs/new_task.py
│  │  └─ widgets/             # PipelineRail / GateBar / Chip / TermView …
│  ├─ core/
│  │  ├─ pipeline_engine.py   # 状态机
│  │  ├─ agent_runner.py      # QProcess → CLI
│  │  ├─ artifact_manager.py
│  │  ├─ repo_manager.py
│  │  ├─ hook_guardrails.py
│  │  ├─ mcp_mirror.py
│  │  └─ task_store.py
│  └─ templates/              # pipeline templates + 默认 phase skills
├─ tests/                     # pytest + pytest-qt
├─ resources/fonts/
└─ pyproject.toml
```

---

## 8. v0.1.0 之后的路线（不做进本期）

- Maintain 阶段（线上反馈回流新任务）
- Pipeline Template 可视化编辑器
- 团队共享：任务状态同步到服务端
- 多 Agent 并行（Copilot + Claude 分阶段混用）

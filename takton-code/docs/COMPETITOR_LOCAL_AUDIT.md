# 本机 Coding CLI 源码/数据走查（2026-07-18）

> 证据来源：本机安装物 + 本地会话 DB/JSONL，不是官网文案。

## 本机安装事实

| 产品 | 本机状态 | 路径/版本 |
|------|----------|-----------|
| **OpenCode** | ✅ 已装 | scoop `opencode` **1.17.13**；`C:\Users\wuyw\scoop\apps\opencode\current\opencode.exe`；数据 `~\.local\share\opencode\opencode.db` |
| **Grok Build CLI** | ✅ 已重新装上做分析 | scoop `grok-cli` **0.2.103**；公式原在 bucket，PATH 里曾有 `~\.grok\bin` 但目录空 |
| **Claude Code** | ⚠️ 二进制不在 PATH，但重度用过 | 本地数据 `~\.claude\` + `~\.claude.json`；**last version 2.1.201**；12 次启动；会话 JSONL 完整 |
| **kimiim-cli** | ❌ 不是 Kimi Code | `~\.local\bin\kimiim-cli.exe` = **群聊/thread CLI**（list-messages/send-message），与 coding agent 无关 |

---

## 1. OpenCode（本机最完整：CLI help + SQLite + debug skill）

### 1.1 产品形态（`opencode --help`）

不止一个 TUI，而是 **CLI 全家桶**：

| 命令 | 作用 |
|------|------|
| `opencode [project]` | 默认 **全屏 TUI** |
| `opencode run [msg]` | 非交互跑一轮 |
| `opencode serve` / `attach` / `web` | **headless server + attach + Web UI** |
| `opencode acp` | ACP 给 IDE |
| `opencode mcp` | MCP add/list/auth/logout/debug |
| `opencode providers` (=auth) | 登录/登出/列表 |
| `opencode agent` | create/list agents |
| `opencode session` | list/delete |
| `opencode models` / `stats` / `export` / `import` | 模型、用量、会话迁移 |
| `opencode github` / `pr` | GitHub agent / 拉 PR 再开 session |
| `opencode plugin` | 插件 |
| `opencode db` | 直接查本地 SQLite |
| `opencode debug skill/lsp/agent/config/paths` | 调试可见性极强 |

关键 flags：`-c/--continue`、`-s/--session`、`--fork`、`-m provider/model`、`--agent`、`--auto`（危险自动批准）、`--mini`（极简交互）、`--replay-limit`。

### 1.2 数据模型（`opencode.db` 真表）

```
account, credential
project (worktree, vcs, sandboxes, commands, icon…)
session (slug 诗意名 shiny-rocket, title, agent, model JSON,
         tokens_*, cost, summary_additions/deletions/files/diffs,
         revert, permission, share_url, time_compacting…)
message (role, agent, mode, model, path{cwd,root}, cost, tokens, summary.diffs)
part     ← 消息拆成零件流
todo
permission
workspace
session_share / session_input / session_context_epoch
event (事件溯源 7000+)
```

**Part 类型（本机 1393 条统计）**：

| type | 次数 | 含义 |
|------|------|------|
| `tool` | 536 | 工具调用（state: completed/error + input） |
| `step-start` / `step-finish` | 276×2 | **一步推理的起止**（带 tokens/cost） |
| `reasoning` | 191 | 思考文本 + time.start/end |
| `text` | 114 | 用户/助手可见正文 |

→ UI 不是「一整坨 assistant markdown」，而是 **step 时间线 + tool chip + reasoning 可折叠**。

**本机真实用过的工具**：

```
read 332 | edit 104 | bash 43 | grep 22 | glob 18 | todowrite 8 | task 5 | websearch 1 | write 1
```

### 1.3 Agent 体系（`opencode agent list` + `debug agent`）

内置 primary：

| Agent | 权限要点 |
|-------|----------|
| **build** | 默认执行；`*` allow；`doom_loop` ask；`external_directory` ask；`*.env` read ask；`question/plan_enter` allow |
| **plan** | **禁止 edit**（description: Disallows all edit tools）；`plan_exit` allow；deny `task general` |
| **general / explore** | explore 用于只读探索（本机 session agent 字段大量 `explore`） |
| 隐藏内部 | `compaction`, `title`, `summary` |

Subagent session 示例标题：`Explore project structure (@explore subagent)`，且 permission 数组 **deny todowrite + task**（子代理收权）。

### 1.4 配置人性化（debug skill 全文落在本机）

- 配置路径：项目 `opencode.json` / `.opencode/` **向上找到 worktree root**；全局 **`~/.config/opencode/`（不是 ~/.opencode）**
- `$schema: https://opencode.ai/config.json` 严格校验，错了 **拒启动**
- Skills：`SKILL.md` 目录；还会扫 **`~/.claude/skills` 与 `~/.agents/skills`**（兼容 Claude）
- Commands：`.opencode/command/*.md`，body 即 template，`$ARGUMENTS` / `$1`
- Plugins：hooks 面极大（`tool.execute.before/after`、`permission.ask`、`session.compacting`…）
- Permission：`allow|ask|deny`；**对象里 LAST matching rule 生效**（插入顺序重要）
- Compaction：`compaction.auto` + `tail_turns`
- 改配置后必须 **重启**（不热加载）——skill 里反复强调

### 1.5 会话 UX 细节（DB 字段）

- session **slug** 随机诗意名 + **title** 可改
- 每 session 累计 **cost + 五维 tokens**（input/output/reasoning/cache_read/cache_write）
- **summary_additions/deletions/files/diffs** → 会话级 diff 摘要
- **revert** 字段 → 支持回滚语义
- **share_url** + `session_share` 表 → 分享
- **time_compacting** → 压缩是一等状态
- parent_id → **session fork/子会话树**

### 1.6 TUI 布局推论（结合 web UI CSS tokens in binary + DB part 流）

```
┌─────────────────────────────────────────────────────────┐
│ Header: project / model / agent(build|plan) / cost      │
├───────────────────────────────┬─────────────────────────┤
│  Chat timeline                │ (optional) side         │
│  - user text                  │  todos / diffs / files  │
│  - step-start                 │                         │
│  - reasoning (fold)           │                         │
│  - tool chips (read/edit/…)   │                         │
│  - step-finish (tokens)       │                         │
│  - assistant text             │                         │
├───────────────────────────────┴─────────────────────────┤
│ Input: @refs autocomplete · /commands · agent switch    │
│ Footer/status: mode Tab · permissions · context usage   │
└─────────────────────────────────────────────────────────┘
```

二进制内有 **sidebar CSS 变量**（`--color-sidebar-*`）+ **ThemeProvider** → 支持 theme；`command.category.theme` i18n。

二进制确认存在的 slash 相关串：`/init /undo /redo /share /compact /connect /models /agent /session /new /help /export /theme /plan /build`。

---

## 2. Claude Code（本机会话 JSONL + tipsHistory，二进制已不在 PATH）

### 2.1 版本与使用痕迹

- `lastOnboardingVersion` / `lastReleaseNotesSeen`: **2.1.201**
- `numStartups`: 12
- 设置：`theme: auto` + 自定义 `ANTHROPIC_BASE_URL` 代理
- 项目级：`allowedTools`, `mcpServers`, trust dialog, cost/duration/lines/tokens/FPS 统计

### 2.2 tipsHistory = 官方认为的「人性化卖点清单」（本机真实弹过）

| tip key | 对应产品能力 |
|---------|----------------|
| `plan-mode-for-complex-tasks` | 复杂任务先 Plan |
| `shift-tab` | **Shift+Tab 切模式** |
| `prompt-queue` | **提示词队列**（生成中还能塞下一条） |
| `enter-to-steer-in-relatime` | **回车实时纠偏**进行中的 agent |
| `todo-list` | Todo 列表 |
| `double-esc-code-restore` | **连按 Esc 恢复代码**（撤销 agent 改动） |
| `drag-and-drop-images` / `image-paste` | 拖图 / 粘贴图片 |
| `status-line` | 可定制 status line |
| `permissions` | 权限模式 |
| `rename-conversation` | 会话重命名 |
| `custom-commands` / `custom-agents` | 自定义命令与 agent |
| `memory-command` / `theme-command` / `continue` | /memory /theme /continue |

### 2.3 会话事件类型（JSONL `type` 字段，本机实锤）

| type | 作用 |
|------|------|
| `user` / `assistant` | 消息；assistant content 是 **parts 数组** |
| `mode` | `normal` 等 UI 模式 |
| `permission-mode` | `default` 等权限档 |
| `file-history-snapshot` | **每次 user 消息可挂文件备份快照**（double-esc restore 的底座） |
| `attachment` | 系统附件（非用户文件） |
| `queue-operation` | `enqueue` / `dequeue` / `popAll` → **prompt queue** |
| `system` + `subtype: turn_duration` | 回合耗时 + messageCount |
| `last-prompt` | 叶节点提示缓存（续跑/UI） |

### 2.4 assistant content parts

- `thinking` — 思考
- `text` — 可见输出
- `tool_use` / `tool_result` — 工具

### 2.5 本机用过的工具名（PascalCase）

`Read`, `Bash`, `Edit`, `TaskCreate`, `TaskUpdate`  
（另有 attachment 声明的 Agent 工具集限制）

### 2.6 内置子 Agent（attachment `agent_listing_delta` 原文）

| Agent | 定位 | 工具限制 |
|-------|------|----------|
| **claude** | 默认 catch-all | `*` |
| **claude-code-guide** | 问 Claude Code/SDK/API 怎么用 | Glob/Grep/Read/Web* only；强调 **先 SendMessage 续已有 guide agent** |
| **Explore** | 只读广搜，**定位不是审计** | 禁 Agent/Artifact/ExitPlanMode/Edit/Write/NotebookEdit |
| **general-purpose** | 复杂研究/多步 | `*` |
| **Plan** | 架构规划，出步骤 | 同 Explore 只读 |
| **statusline-setup** | 配 status line | Read/Edit |

### 2.7 布局/交互（从 tip + 事件反推）

```
┌──────────────────────────────────────────┐
│ status-line（可定制，专用 agent 配置）      │
├──────────────────────────────────────────┤
│ transcript                               │
│  thinking / text / tool_use chips        │
│  permission prompts inline               │
├──────────────────────────────────────────┤
│ input                                    │
│  /slash 菜单（上/下选择）                   │
│  生成中：Enter=steer · 队列 enqueue        │
│  Esc Esc = restore files from snapshot   │
│  Shift+Tab = cycle mode/plan             │
│  图片 DnD / paste                        │
└──────────────────────────────────────────┘
```

人性化核心：**file-history-snapshot + double-esc**、**prompt-queue**、**steer-in-realtime**、**子 agent 工具白名单写进 listing**。

---

## 3. Grok Build CLI 0.2.103（本机 scoop 现装，`--help` + `inspect --json` + 二进制字符串）

### 3.1 双 UI 哲学（官方 help 原文级）

| 模式 | flag | 行为 |
|------|------|------|
| **Fullscreen TUI** | 默认 / `--fullscreen` | alt-screen 全屏 |
| **Minimal** | `--minimal` 或 config `screen_mode=minimal` | **定稿块打进终端原生 scrollback**，底部钉住 prompt+当前 turn → 可用终端自己的滚动/选择 |
| Inline | `--no-alt-screen` | 不用 alt screen |

→ 这是 Grok 相对 OpenCode/Claude 很突出的 **scrollback-native** 细节。

### 3.2 权限模式（`--permission-mode`）

`default | acceptEdits | auto | dontAsk | bypassPermissions | plan`

Shift+Tab **循环**：Normal → Plan → Always-approve（二进制 help 表）。

### 3.3 一等公民能力

| 能力 | 证据 |
|------|------|
| Plan | `--no-plan`；`/view-plan`；plan agent；Exit plan mode 流程文案 |
| Worktree | `-w/--worktree`、`--worktree-ref`；`grok worktree list/show/rm/gc` |
| Subagents | builtin `general-purpose` / `explore` / `plan`；`--no-subagents`；`--agents JSON` |
| Headless | `-p/--single`；`output-format plain|json|streaming-json`；`grok agent stdio|serve|leader` |
| Parallel | `--best-of-n`（headless）；`--check` 自验证环 |
| Resume/Fork | `-c`、`-r`、`--fork-session`、`--restore-code`（恢复原 commit） |
| MCP/Plugin/Marketplace | `grok mcp`、`plugin`、`/marketplace` |
| Memory | `grok memory`；`--experimental-memory` / `--no-memory` |
| Dashboard | `grok dashboard`；**Ctrl+\\** Agent Dashboard 多 session |
| Prompt queue | **Ctrl+;**（alt Ctrl+'） |
| Compat | `inspect` 显示 **自动读 Cursor + Claude** 的 skills/rules/agents/mcps/hooks/sessions |
| Import | **`/import-claude`** |
| Leader | `~/.grok/leader.sock` 多客户端共享后端 |

### 3.4 Slash（二进制精选）

`/build /compact /compact-mode /context /copy /docs /dream /export /feedback /flush /fork /fullscreen /history /hooks /import-claude /login /logout /marketplace /mcps /memory /minimal /models /multiline /new /personas /plugins /quit /resume /rewind /session-info /settings /share /skills /theme /traces /usage /view-plan /vim-mode /always-approve /auto …`

### 3.5 布局

**Fullscreen：**

```
┌─ multi-session dashboard (Ctrl+\) ─────────────────────┐
│ Agent list · pin · status                              │
└────────────────────────────────────────────────────────┘
┌─ main transcript ──────────────────────────────────────┐
│ plan banner · diffs · tool calls · reasoning           │
├─ prompt queue (Ctrl+;) ────────────────────────────────┤
│ queued prompts                                         │
├─ input ────────────────────────────────────────────────┤
│ Shift+Tab mode · /slash · @                           │
└─ status: model · mode · permissions · tokens ──────────┘
```

**Minimal：** 历史在 **终端 scrollback**；只有底栏是 TUI。

### 3.6 `grok inspect` 人性化

进目录先看：projectRoot、trusted、instructions、permissions sources、hooks/skills/agents/plugins/mcp/lsp、**externalCompat（Claude/Cursor 九宫格开关）**。  
→ 「我到底加载了啥」一等公民（比静默注入强）。

---

## 4. 三者对照（功能 × 界面 × 人性化）

| 维度 | OpenCode | Claude Code | Grok Build |
|------|----------|-------------|------------|
| 主 UI | 全屏 TUI + web + mini | 全屏 TUI | 全屏 **或** minimal scrollback |
| 模式切换 | Tab build/plan（文档+agent） | Shift+Tab；mode 事件 | Shift+Tab Normal/Plan/Always-approve |
| 消息模型 | **part 流** step/tool/reasoning | content parts + snapshots | turn-deltas（slash `/turn-deltas`） |
| Diff/撤销 | session summary diffs + `/undo` `/redo` | **file-history-snapshot + double-esc** | `/rewind`；`--restore-code` |
| 队列 | session_input 表 | **queue-operation enqueue** | Ctrl+; prompt queue |
| 实时纠偏 | （serve/attach） | **Enter steer realtime** tip | （多 client leader） |
| Todo | `todo` 表 + todowrite 工具 | TaskCreate/Update | （plan 文件） |
| 子代理 | build/plan/explore + task 工具 | Explore/Plan/guide… listing | explore/plan/general-purpose |
| 权限 | 细粒度 pattern last-match | permission-mode 事件 | 6 档 permission-mode |
| Worktree | project.worktree 字段 | （较弱于 Grok） | **一等 CLI + session -w** |
| 多 session | session 树 parent_id | 多 jsonl | **Dashboard Ctrl+\\** |
| 生态兼容 | 读 `~/.claude/skills` | MCP/hooks 自有 | **读 Claude+Cursor 配置** + `/import-claude` |
| Headless | run/serve/acp/export | `-p`/CI（本机未见二进制） | `-p` + agent stdio/ws + best-of-n |
| 可观测 | stats/db/debug/* | turn_duration/cost/FPS | inspect/traces/usage |
| 分享 | session_share | （云端能力本机未证） | `/share` |

---

## 5. Takton Code 应对标的「不是功能名，是体验」

按本机证据排出的 **P0 体验清单**（抄行为不抄皮）：

### P0 交互骨架
1. **Fullscreen TUI 时间线** = OpenCode part 模型：`step-start → reasoning? → tool* → text → step-finish(tokens)`
2. **Tab / Shift+Tab 模式环**：`build ↔ plan`（Grok 再加 always-approve）
3. **Status line**：model · mode · tokens · cost · compress · bridge · session slug
4. **Slash 命令面板**（输入 `/` 出列表，可上下选）— Claude tip + 三家都有
5. **Prompt queue**：生成中可 enqueue；快捷键打开队列面板
6. **Steer / Stop**：进行中可注入；Ctrl+C 取消当前 turn（保留 queue）
7. **Diff 侧栏 + revert**：每次 edit 进 session summary；`/undo` 或 double-esc 级恢复（file snapshot）

### P0 工程语义
8. **Plan agent = edit deny**（OpenCode plan permission 实锤）
9. **Explore subagent = 只读 + 禁 spawn**（Claude listing 原文）
10. **`inspect` 一键**：加载了哪些 AGENTS/skills/MCP/hooks/bridge/desktop 能力
11. **Session：诗意 slug + 可改 title + fork + resume + export/import**
12. **Compaction 一等状态**（OpenCode `time_compacting`；Grok `/compact`）

### P0 桌面关系（你的产品约束）
13. 桌面 **只做入口**（打开外部 terminal 跑 TUI，或 attach 到 code serve）
14. 后端 **bridge 全量**：models/skills/tools/MCP/RAG = Desktop 同一套（对标 Grok 读 Claude 配置的「互通」思路，但是自家 backend）

### P1
15. Worktree session（Grok）
16. Multi-session Dashboard（Grok Ctrl+\\）
17. Minimal scrollback 模式（Grok）
18. 图片 paste/DnD（Claude tip）
19. ACP + headless streaming-json
20. 兼容扫描 `~/.claude` / Cursor skills（可选）

---

## 6. 对 Takton Code TUI 线框（综合三家，偏 OpenCode 时间线 + Grok 键位 + Claude 队列/恢复）

```
Header
  takton-code  ·  {project}  ·  {branch}  ·  agent:build|plan|explore  ·  model

Main (3fr)                         Side (1fr)
  timeline parts:                    Changes (diff list)
    user                             Todos
    step #3 · 1.2s · $0.00           Plan steps (if plan)
    ↳ reasoning (fold)
    ↳ tool read path
    ↳ tool edit path (+hunk)
    assistant text
  [queue badge: 2 waiting]

Input
  ›  message or /command or @file
  hints: Tab mode · Ctrl+C stop · Ctrl+; queue · Ctrl+O diff · Esc Esc undo

Footer/statusline
  BUILD │ tokens 12k/64k │ compress×2 │ bridge:on │ perm:cautious │ sess kind-star
```

---

## 7. 结论（给实现的一句话）

- **OpenCode**：本地最可抄的是 **数据模型（session/message/part/todo）+ agent 权限矩阵 + debug 可观测 + 配置/skill/plugin 文件约定**。  
- **Claude Code**：本地最可抄的是 **prompt queue、double-esc file restore、steer、statusline、子 agent 工具边界文案**。  
- **Grok**：本地最可抄的是 **minimal/fullscreen 双 UI、Shift+Tab 三态权限、worktree 一等、Dashboard 多会话、inspect、跨工具配置兼容**。  

Takton Code 不应只做「能聊的 REPL」，而应做成：**OpenCode 级 part 时间线 TUI + Claude 级队列/恢复手感 + Grok 级 inspect/worktree/权限环 + 桌面 bridge 互通**。

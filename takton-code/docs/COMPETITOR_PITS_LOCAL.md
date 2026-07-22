# 本机竞品暗坑与可抄语义（2026-07-21）

> 证据：scoop `opencode` 1.17.13、`grok-cli` 0.2.103 二进制字符串 + `--help` + `opencode debug agent`；  
> Claude Code 数据 `~/.claude` / `~/.claude.json`（二进制不在 PATH，version tip 2.1.201）。  
> **不是官网文案。**

---

## 1. OpenCode（本机最可抄的权限模型）

### 1.1 权限是「key + pattern + action」，不是裸 tool 名

`opencode debug agent build|plan` 实锤字段：

```json
{ "permission": "edit", "action": "deny", "pattern": "*" }
{ "permission": "external_directory", "pattern": "*", "action": "ask" }
{ "permission": "doom_loop", "action": "ask", "pattern": "*" }
{ "permission": "read", "pattern": "*.env", "action": "ask" }
```

内置 skill/文档串（二进制）：

- **LAST matching rule wins** → 宽规则在前，窄规则在后  
- Plan Mode = **plan agent 的 permission 规则集**（`edit: deny *`），不是单独布尔  
- 已知 key：`read, edit, glob, grep, list, bash, task, external_directory, todowrite, question, webfetch, websearch, lsp, doom_loop, plan_enter, plan_exit`

### 1.2 暗坑

| 坑 | 说明 | Takton 应对 |
|----|------|-------------|
| `external_directory` 默认 ask | 出项目根就拦 | `PermissionGate.project_root` + 路径逃逸检测 |
| `*.env` read ask | 密钥文件 | read 规则 pattern |
| `doom_loop` ask | 防死循环工具风暴 | 预留 key；后续接 iteration 阈值 |
| plan 仍 allow read | 不是全 deny | mode∈plan/ask/explore → 只 deny edit/bash |
| `--auto` | auto-approve **未显式 deny** 的（危险） | 映射到 bypass/acceptEdits，文档标 dangerous |
| `--mini` | OpenCode 的 minimal 交互；有 replay-limit | 我们用 `--minimal`（Grok 名）；勿与 `--mini` REPL 混淆 |
| serve/attach | 多客户端是 **HTTP serve**，不是 unix sock | leader 用 TCP 127.0.0.1 合理 |
| 配置不热加载 | skill 反复强调 restart | `/model` 仅热更本会话 LLM |

### 1.3 数据模型可抄

`~/.local/share/opencode/opencode.db`：`session/message/part/todo/permission/session_input`  
part 时间线：`step-start → reasoning → tool* → text → step-finish`

---

## 2. Grok Build CLI 0.2.103（本机 ops 面）

### 2.1 Minimal UI（必须抄行为）

help 原文级：

- **Finalized blocks → 终端原生 scrollback**（可选中/滚动）  
- **pinned region** = prompt + **running turn only**  
- `screen_mode = "fullscreen" | "minimal"` 写 config；flag 仅 session 级  
- 内部 crate：`xai-grok-pager` scrollback pane / finish.pin_suppressed_override  

**坑：** Textual alt-screen **做不到**真 scrollback → Minimal **必须**独立渲染栈（我们 `tui/minimal.py`）。

### 2.2 Permission modes（help 表）

| Mode | 行为 | 坑 |
|------|------|-----|
| `default` | 正常询问 | |
| `acceptEdits` | 自动批 file edits | shell 仍 ask |
| `auto` | 自动策略（受 feature gate） | 本机可能 gated |
| `dontAsk` | **无显式 allow 则 deny** | CI 默认；不是 allow all |
| `bypassPermissions` | 自动批 tool；**deny 规则/hooks/shell ask 仍生效** | 不是完全上帝模式 |
| `plan` | 规划只读 | |

**致命坑（二进制串）：**

> Always-approve ON: **plan mode still blocks file edits until you exit plan mode**

→ `mode=always` **不能**覆盖 `mode=plan` 的 edit deny。

**Headless 坑：**

> In headless runs (`-p`), a tool call that would prompt is **cancelled and reported to the model** instead of waiting for input.

→ 禁止 `input()` 卡死 CI；tool 返回 deny/cancel 文案给模型。

### 2.3 best-of-n

- headless only  
- **`best-of-n-runner`**：每个 candidate **独立 git worktree + branch**  
- slash `/best-of-n` 也有  
- **不要**默认 merge 回主树（我们已遵守）

### 2.4 Leader / Dashboard

- 默认 `~/.grok/leader.sock` + `GROK_LEADER_SOCKET`  
- `grok agent leader` / `grok leader list|kill`  
- **Windows 上 unix sock 不可靠** → TCP `127.0.0.1` + `leader.json`（我们路径）  
- Dashboard：`Ctrl+\`；可被 config/`GROK_AGENT_DASHBOARD=0`/未 trust folder **禁用**  
- 未 trust / 未登录会挡 Dashboard（二进制提示串）

### 2.5 键位坑

- Prompt queue：`Ctrl+;` ，**Windows alt = `Ctrl+'`**  
- 我们已实现 Ctrl+; ；Win 上应加 Ctrl+' 别名

### 2.6 worktrees 表

二进制 SQL：

```sql
CREATE TABLE IF NOT EXISTS worktrees (... session_id ...);
```

worktree 与 session 绑定是一等公民，不只是 git 副作用。

---

## 3. Claude Code（本机仅数据）

`~/.claude.json` tipsHistory 实锤卖点：

- plan-mode, shift-tab, prompt-queue, enter-to-steer-in-relatime  
- double-esc-code-restore, todo-list, status-line, image-paste, permissions  

JSONL types：`permission-mode`, `file-history-snapshot`, `queue-operation`, `mode`, …

**坑：**

- file-history 是 **每条 user 可挂快照**，不只 turn 结束  
- queue 有 enqueue/dequeue/popAll 事件类型  
- 子 agent 工具白名单写在 listing attachment 里（Explore 禁 Edit/Write）

---

## 4. 已回灌到 Takton Code 的点

| 竞品点 | 落地 |
|--------|------|
| OpenCode last-match + plan edit deny | `agent/permissions.py` |
| external_directory / *.env | 同上 |
| Grok dontAsk / acceptEdits / bypass | `--permission-mode` 映射 |
| Always 不破 plan | `check()` mode 优先 |
| Headless ask → cancel | `PermissionBroker.headless` |
| Minimal scrollback 独立栈 | `tui/minimal.py` |
| best-of-n worktree 隔离、不自动 apply | `agent/best_of_n.py` |
| Leader 本机 TCP 无 token | `leader/*` |
| Part 时间线 | 既有 parts + renderer |

## 5. 尚未抄满（下轮优先从竞品行为抠）

1. **Ctrl+'** = queue 的 Windows 别名（Grok）  
2. **doom_loop** 运行时检测（同 tool 连续 N 次 → ask）  
3. worktree **DB 表**绑 session_id（Grok SQL）  
4. OpenCode **serve/attach** 协议对齐（我们 leader JSONL 可演进）  
5. Claude **每 user 消息 file-history-snapshot**（现在偏 turn 级）  
6. Grok minimal 的 **pinned region 真分屏**（现在 sticky 行近似）  
7. OpenCode permission key 级配置文件（`opencode.json` permission 块）  

---

## 6. 实施纪律（主人建议）

1. 新 UI/权限/headless 行为：**先 `opencode debug` / `grok --help` / 二进制 strings / 本地 DB**，再写代码。  
2. 闭源二进制不能直接链库，但 **语义、表结构、flag 名、失败文案** 都是可复用规格。  
3. 坑位以本文件为准；实现偏离时改代码或改文档，禁止只改嘴。

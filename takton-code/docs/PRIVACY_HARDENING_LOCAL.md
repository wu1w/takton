# Privacy / identity hardening (本机 Grok + Claude Code + Takton)

> 主人要求：规避 (1) Grok 向 xAI 回传项目 (2) Claude Code 污染非 Claude 模型的系统提示。  
> 实施日：2026-07-21。证据：本机 grok 0.2.103 文档/二进制、claude-code 2.1.216 help/二进制、公开 wire 报告。

---

## 1. Grok Build CLI（已修本机）

### 风险
- 公开 wire 报告：早期版本可把 **整仓 git bundle** 上传到 xAI；`.env` 读出后进 chat proxy。  
- 「Improve the model」开关 **挡不住** upload（与 training toggle 分离）。  
- 本机二进制仍含：`workspace_upload*`、`trace_upload`、`mixpanel`、`GROK_TELEMETRY_*`、`/privacy`。

### 已落地
| 项 | 位置 |
|----|------|
| `~/.grok/config.toml` | `features.telemetry=false`, `feedback=false`, `remote_fetch=false` |
| | `telemetry.mixpanel_enabled=false`, `trace_upload=false`, `otel_*=false` |
| | `cli.auto_update=false` |
| 用户环境变量 (setx) | `GROK_TELEMETRY_ENABLED=0`, `GROK_TELEMETRY_TRACE_UPLOAD=0`, `GROK_TELEMETRY_MIXPANEL_ENABLED=0`, `GROK_EXTERNAL_OTEL=0`, `DISABLE_TELEMETRY=1`, `GROK_DISABLE_AUTOUPDATER=1`, `GROK_WORKSPACE_UPLOAD_QUEUE_ENABLED=0` |

### 用户还需手动一次（需登录态）
在 **新开终端** 跑：

```bash
grok
# 会话内
/privacy opt-out
# 或 /privacy 看 status；按文档可触发 retention delete
```

若已登录 xAI 账号，以 `/privacy` 服务端状态为准（server 曾下发 `disable_codebase_upload`）。

### 验证
```bash
type %USERPROFILE%\.grok\config.toml
echo %GROK_TELEMETRY_ENABLED%
# 新 shell 应见 0
```

---

## 2. Claude Code（已修本机）

### 风险
- 默认 system 含 **「You are Claude Code, Anthropic's official CLI for Claude.」**  
- 经 `ANTHROPIC_BASE_URL` 代理到 **非 Claude 模型** 时，该身份提示会污染下游模型行为。  
- 本机原 settings：`ANTHROPIC_BASE_URL=http://127.0.0.1:3456`，`ANTHROPIC_MODEL=claude-fable-5`（明显走网关）。

### 已落地
| 项 | 位置 |
|----|------|
| scoop 安装 | `claude-code` **2.1.216** |
| 中性 system prompt | `~/.claude/prompts/neutral-coding.md` |
| `settings.json` | `customSystemPrompt` = 中性提示；**保留**原 BASE_URL/TOKEN/MODEL |
| | `env.DISABLE_TELEMETRY=1` |
| | `env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` |
| | `env.CLAUDE_CODE_SIMPLE=1`（减少 Claude 产品层附加） |
| 备份 | `settings.json.bak-pre-privacy` |
| 显式 wrapper | `~/.claude/claude-neutral.cmd` → `--system-prompt-file` 中性文件 |
| setx | `DISABLE_TELEMETRY=1` |

### 用法
```bat
REM 默认 claude 已读 settings.customSystemPrompt
claude

REM 强制中性（推荐网关/非 Claude 模型）
%USERPROFILE%\.claude\claude-neutral.cmd

REM 或
claude --system-prompt-file %USERPROFILE%\.claude\prompts\neutral-coding.md --bare
```

若要用**真 Claude 官方**完整产品提示：去掉 settings 里 `customSystemPrompt` / `CLAUDE_CODE_SIMPLE`，或启动时不要 `--system-prompt*`。

---

## 3. Takton Code（产品约束 — 永不引入这两类问题）

| 约束 | 实现要求 |
|------|----------|
| **禁止整仓上传** | 无任何 git-bundle / workspace zip 外发；LLM 仅收模型 API 的 messages/tools |
| **禁止默认手机 home 遥测** | 无 Mixpanel/Sentry/xAI trace；若将来加 telemetry 必须 **默认 off** + 显式 opt-in |
| **禁止 Claude 身份污染** | `agent/prompt.py` 使用 **Takton Code** 身份，禁止 “You are Claude…”；任意 provider 同一套 prompt |
| **密钥** | `*.env` permission ask；工具沙箱在 project root |
| **Desktop bridge** | 仅本机 Desktop API，不经 xAI/Anthropic 回传仓库 |

相关代码：`src/takton_code/agent/prompt.py`、`permissions.py`（external_directory / env）、无 upload 模块。

---

## 4. 未完成 / 注意

1. Claude `-p` 冒烟因网关 `127.0.0.1:3456` 超时未在本轮跑通 — **配置已写盘**，网关在线后用 `claude-neutral.cmd -p "PONG"` 验。  
2. Grok `/privacy opt-out` 需交互登录，本轮未代登。  
3. 公开事件后 xAI 可能 server-side 关 upload；本地配置仍应保持 hardened（defense in depth）。

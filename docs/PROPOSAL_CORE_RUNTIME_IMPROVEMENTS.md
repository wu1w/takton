# Takton 核心运行时改进方案

> 基于 Claude Code 源码拆解，针对 Takton v0.2.6 + 072a783 当前状态
> 日期：2026-07-24 | 版本：v1.0

---

## 一、背景与诊断

### 当前差距数据（mimo-v2.5 同模型压测）

| 任务 | Takton v0.2.6+072a783 | Hermes v0.19 | 差距倍数 |
|------|----------------------|-------------|---------|
| T1 找针 | 57s / 6 tool | 31s / 1 tool | 1.8x |
| T2 修 bug | 75s / 17 tool | 91s / 1 tool | 0.8x* |
| T3 建包 | 122s / 22 tool | 93s / 14 tool | 1.3x |

*T2 本轮偶发 Hermes 慢（网络波动），正常场景 Hermes 应快 2-3x。

### 根因拆解

问题不是「模型不会干活」，而是 **orchestration layer 的 token 浪费**：

1. **工具结果原文进 LLM**：一条 `file_read` 返回 1000 字符，10 条就是 10k 字符，占满 context window 的预算
2. **单轮单工具惯性**：58% 的轮次是「一轮一个 tool」，每轮要一次完整的 LLM 调用
3. **command 长命令卡死 loop**：一次 120s timeout 的命令直接阻塞整个 agent loop
4. **压缩被动触发**：上下文快满时 L4 才压缩，可能来不及→被 OpenCode 400/413

### Claude Code 的对应设计

| Takton 问题 | Claude Code 的解法 | 核心文件 |
|------------|-------------------|---------|
| 工具结果原文进 LLM | **TOOL_SUMMARY_MAX_LENGTH** + preview 机制 | `utils/toolResultStorage.ts` |
| 单轮单工具 | **parallel tool call 强制规则** + `BashTool` 里 `isSearchOrReadBashCommand` 分类 | `query.ts` L733-740 |
| command 卡死 | **ASSISTANT_BLOCKING_BUDGET_MS = 15s** 自动后台化 | `BashTool.tsx` L30-31 |
| 压缩被动 | **reactiveCompact**：413 立即触发强制摘要 | `services/compact/reactiveCompact.ts` |
| 安全黑名单 | **AST 解析** + 命令语义分类（allow/ask/deny） | `BashTool/bashSecurity.ts` |

---

## 二、改进计划

### Phase 1：工具结果压缩（预期：每轮 token 减少 60-70%）

#### 1.1 tool_result_contract 增强

**文件**：`backend/agent/tool_result_contract.py`

**现状**（41 行）：只有一个 `normalize_tool_result()` + `is_tool_error()`

**改进**：参考 Claude Code 的 `toolResultStorage.ts` + `TOOL_SUMMARY_MAX_LENGTH`

```python
# 新增：按工具类型的差异化截断策略
TOOL_RESULT_BUDGET = {
    # 工具名 → 最大字符数（超出部分截断，用 preview 替代）
    "file_read": 2000,       # 文件读取：保留前后各 500 + 中间 1000
    "grep": 1500,            # 搜索结果：保留前 50 条命中
    "glob": 800,             # 文件列表：保留前 30 条
    "command": 3000,         # 命令输出：保留前 2000 + 后 500 + exit code
    "file_write": 200,       # 写入确认：只返回"成功" + 文件大小
    "edit": 300,             # 编辑确认：只返回"成功" + 改了几行
    "python": 2000,          # 代码执行：保留输出
    "http": 1000,            # HTTP 响应：保留 headers + body 前 500
    "web_search": 1500,      # 搜索：保留前 5 条结果的 title + url
    "browser": 1000,         # 浏览器：保留 snapshot 摘要
}
DEFAULT_TOOL_BUDGET = 1000  # 未列出工具的默认截断

def truncate_for_llm(tool_name: str, raw_result: str) -> str:
    """按工具类型截断 result，只发摘要给 LLM"""
    budget = TOOL_RESULT_BUDGET.get(tool_name, DEFAULT_TOOL_BUDGET)
    if len(raw_result) <= budget:
        return raw_result
    head = raw_result[: int(budget * 0.7)]
    tail = raw_result[- int(budget * 0.2):]
    omitted = len(raw_result) - len(head) - len(tail)
    return f"{head}\n...[{omitted} chars omitted]...\n{tail}"
```

**集成点**：`loop.py` 的 `normalize_tool_result()` 调用处（约 L1408），在把 result 塞进 `messages` 之前先过 `truncate_for_llm`。

**预期效果**：
- 原始 tool result 从平均 800-3000 字符截断到 1000-2000
- 每轮从「Sending 50 messages (total chars: 18000)」降到 10000-12000
- LLM 调用 latency 减少（更少 token → 更少延迟）

---

### Phase 2：command 命令分类 + 语义识别（预期：轮次准确率 ↑ + nudge 命中率 ↑）

#### 2.1 新建 `command_classifier.py`

参考 Claude Code 的 `commandSemantics.ts` + `BashTool.tsx` L70-107

```python
"""Shell 命令语义分类——决定「这条命令是只读、还是修改了什么」"""
from __future__ import annotations

import re

# Claude Code 验证过的集合（按角色分）
READ_COMMANDS = frozenset({
    "cat", "head", "tail", "less", "more", "wc", "stat", "file",
    "strings", "jq", "awk", "cut", "sort", "uniq", "tr", "nl", "od", "hexdump",
    "ls", "tree", "du", "df", "find", "locate", "which", "whereis",
    "grep", "rg", "ag", "ack", "diff", "comm", "file",
    "git log", "git show", "git status", "git diff",
    "echo", "printf", "true", "false", "pwd", "hostname", "date",
    "python -c", "node -e",  # 纯执行的脚本
})

WRITE_COMMANDS = frozenset({
    "rm", "mv", "cp", "mkdir", "rmdir", "chmod", "chown", "touch",
    "ln", "tar", "zip", "unzip", "dd",
    "git add", "git commit", "git push", "git reset",
    "pip install", "npm install", "cargo install",
    "pip uninstall", "npm uninstall",
})

# 模式识别：单条 cat = 只读，cat | grep = 只读，cat > file = 写入
REDIRECT_PATTERN = re.compile(r'[1-2]?>[>&]?|>>')
PIPE_PATTERN = re.compile(r'\|')
Heredoc_PATTERN = re.compile(r'<<')

def classify_command(command: str) -> str:
    """返回 'read' / 'write' / 'mixed' / 'unknown'"""
    cmd = command.strip()
    if not cmd:
        return "unknown"

    # 多步管道：按 && 和 | 拆分
    segments = re.split(r'&&|;|\|\|', cmd)
    classifications = set()
    for seg in segments:
        seg = seg.strip()
        if not seg:
            continue
        classifications.add(_classify_single(seg))

    if classifications == {"read"}:
        return "read"
    if "write" in classifications:
        return "write"
    if len(classifications) > 1:
        return "mixed"
    return "unknown"

def _classify_single(seg: str) -> str:
    """单个命令段的分类"""
    # 有重定向到文件 → 写入
    if REDIRECT_PATTERN.search(seg):
        parts = REDIRECT_PATTERN.split(seg, 1)
        if len(parts) > 1 and parts[1].strip().startswith('/') or '.' in parts[1][:5]:
            return "write"

    # pipe：按 | 拆分，最后一条决定
    if PIPE_PATTERN.search(seg):
        parts = seg.split('|')
        return _classify_single(parts[-1].strip())

    # 提取基础命令
    words = seg.split()
    if not words:
        return "unknown"
    cmd_name = words[0].split('/')[-1]  # 去掉路径前缀
    # git 特殊处理
    if cmd_name == "git" and len(words) > 1:
        sub = words[1]
        if sub in ("log", "show", "status", "diff", "blame"):
            return "read"
        if sub in ("add", "commit", "push", "reset", "checkout", "merge"):
            return "write"

    if cmd_name in READ_COMMANDS:
        return "read"
    if cmd_name in WRITE_COMMANDS:
        return "write"
    return "unknown"
```

#### 2.2 集成到 `decisive.py`

改进 `is_timid_read_round()`：

```python
def is_timid_read_round(tool_names, tool_calls):
    """扩展检测：command 工具中只做只读操作也算 timid"""
    if len(tool_names) != 1:
        return False
    name = tool_names[0]
    if name in _READISH:
        return True
    if name == "command" and tool_calls:
        cmd = _extract_command(tool_calls[0])
        return classify_command(cmd) == "read"  # 用新的分类器
    return False
```

#### 2.3 新增 `backend/tests/test_command_classifier.py`

参考 Claude Code 的命令分类测试方式，覆盖：
- `cat foo | grep bar` → read
- `cat foo > bar` → write
- `python -m pytest` → read（测试运行不修改代码）
- `git add . && git commit` → write
- 复合命令的分类

---

### Phase 3：command 后台化（预期：消除阻塞，长命令不再卡死 loop）

#### 3.1 `executors.py` 新增后台自动切换

参考 Claude Code 的 `ASSISTANT_BLOCKING_BUDGET_MS = 15_000`

```python
# 新增配置
AGENT_COMMAND_AUTO_BG_SECONDS = 15  # 超过这个时间自动推到后台

async def run_command(command, cwd, timeout, arguments):
    # 先检查是否可后台化
    if timeout <= 0:
        timeout = 120

    # 如果 timeout > 阈值，先尝试快速完成，超时则推后台
    if timeout > AGENT_COMMAND_AUTO_BG_SECONDS:
        # 用 process 工具的后台模式
        item = await start_background(command, cwd=cwd)
        return f"[Background task started] process_id={item.id}\n" \
               f"Use process action=poll to check progress."
    else:
        return await run_foreground(command, cwd=cwd, timeout=timeout)
```

#### 3.2 `tool_result_contract.py` 增加后台任务结果处理

```python
def normalize_tool_result(result, max_chars, tool_name=None):
    # 如果结果包含后台进程 PID，提供轮询提示
    if "[Background task started]" in result:
        return result  # 不截断后台提示
    # 正常截断逻辑...
```

**预期效果**：
- 超过 15s 的长命令不再阻塞整个 agent loop
- 模型可以在等待期间处理其他工具
- 压测中 `python -m pytest` 不再阻塞

---

### Phase 4：reactiveCompact（预期：413 时不再崩溃）

#### 4.1 `context_pipeline.py` 新增 reactiveCompact

参考 Claude Code 的 `reactiveCompact.ts`

```python
async def reactive_compact_if_needed(messages, session_id, llm_service):
    """当 LLM 调用返回 413/prompt_too_long 时的应急压缩"""
    # 检测是否触发过 413
    if not _detect_413_occurred():
        return messages

    logger.warning("reactiveCompact: 413 detected, forcing emergency compression")

    # 策略：先尝试 microcompact（只截断 tool results）
    messages = _microcompact_all_tool_results(messages)

    # 如果还不够 → 压缩为 L5（摘要）
    token_est = estimate_tokens(messages)
    if token_est > _threshold():
        messages = await _l5_auto_compact(messages, session_id, llm_service)

    _reset_413_flag()
    return messages
```

#### 4.2 在 `loop.py` 的 LLM 调用处捕获 413

```python
try:
    response = await llm_service.chat(messages, tools=tools, stream=True)
except LLMError as e:
    if is_413_error(e):
        messages = await reactive_compact_if_needed(messages, session_id, llm_service)
        response = await llm_service.chat(messages, tools=tools, stream=True)
    else:
        raise
```

---

### Phase 5：parallel tool call 强化（预期：batch tool 占比从 40% → 65%+）

#### 5.1 system prompt 补强

```python
# PARALLEL_TOOL_CALLS 扩充（system_prompt.py）
PARALLEL_TOOL_CALLS = (
    "# Parallel tool calls\n"
    "When you need several pieces of information that don't depend on each "
    "other, request them together in a single response instead of one tool "
    "call per turn.\n"
    "HARD RULE: if you already know you need file A and file B, call "
    "file_read on both in the SAME turn. Never spend a whole turn on a "
    "single file_read when more related files are obviously required.\n"
    "When creating a package: emit all file_write calls in ONE assistant "
    "turn. This is 2-3x faster than writing one file per turn.\n"
    "When fixing a bug: read file + run tests in the SAME turn if the "
    "file path is known.\n"
    "Only serialize calls when a later call genuinely depends on an earlier "
    "call's result (e.g. you must read a file before you can patch it)."
)
```

#### 5.2 loop.py 加 parallel tool call 后处理

参考 Claude Code 的 `streamingToolExecutor.getRemainingResults()`

```python
# 在 tool round 结束后、下一个 LLM 调用前，检查是否有遗漏的并行调用机会
# （不改变执行逻辑，只是添加一条 system message 提醒）
if _timid_read_streak >= 2 and not _force_final_no_tools:
    messages.append({
        "role": "system",
        "content": (
            "你已经连续 3 轮只调用了 1 个工具。"
            "请立即并行发出多个 tool_calls。"
            "如果信息已经足够，不要继续读取——直接开始编辑/写文件。"
        ),
    })
```

---

## 三、实施节奏

### Sprint 1（1-2 天）：Phase 1 + Phase 2
- 核心改动：`tool_result_contract.py`（截断策略）+ `command_classifier.py`（命令分类）
- 验证：重跑 T1/T3，观察每轮 token 消耗和轮次变化
- 测试：`test_command_classifier.py`（命令分类）+ `test_truncate_for_llm.py`（截断正确性）

### Sprint 2（1 天）：Phase 3
- 核心改动：`executors.py` 后台自动切换
- 验证：构造一个 `sleep 20` 的长命令，确认不再阻塞 agent loop
- 测试：`test_command_bg_timeout.py`

### Sprint 3（1 天）：Phase 4
- 核心改动：`context_pipeline.py` 的 reactiveCompact
- 验证：构造大上下文 + 触发 413，确认不崩溃、自动恢复
- 测试：`test_reactive_compact.py`

### Sprint 4（半天）：Phase 5
- 核心改动：`system_prompt.py` + `loop.py` 的 parallel 提醒
- 验证：重跑 T3，观察 batch 占比变化

---

## 四、验收标准

### 量化目标（mimo-v2.5 同条件）

| 指标 | 当前 | Phase 1+2 后 | Phase 3+4+5 后 |
|------|------|-------------|----------------|
| T3 轮次 | 22 | ≤15 | ≤12 |
| T3 time | 122s | ≤90s | ≤75s |
| 每轮 token | ~18k chars | ≤12k chars | ≤10k chars |
| batch tool % | 40% | 50% | 65%+ |
| Security Blocked | 0 | 0 | 0 |
| 413 error | 偶发 | 偶发（有恢复） | 0 |

### 质量底线

- [ ] T1/T2/T3 全部 pass（核心功能不退化）
- [ ] 新测试通过：`test_command_classifier` / `test_truncate` / `test_reactive_compact`
- [ ] 413 后自动恢复，不崩溃不卡死

---

## 五、与 Claude Code 的差异声明

本方案借鉴 Claude Code 的架构思路，但**不直接复制**：

| Claude Code | Takton 保留差异 |
|-------------|-----------------|
| 工具执行在 streaming 中并行 | Takton 保持批量执行（降低实现复杂度） |
| AST 解析安全 | Takton 用规则集合+正则（足够覆盖主要场景） |
| Haiku 摘要工具结果 | Takton 用确定性截断（不依赖模型调用） |
| 413 → reactiveCompact → session memory compact | Takton 413 → microcompact → L5（三层足够） |
| `--bypass-permissions` 全局模式 | Takton 保持 `profile` 分级（coding/assistant/ops） |

核心原则：**借鉴思路，不搬实现**。Claude Code 是 TS+Bun，Takton 是 Python+FastAPI，技术栈不同；但「结果摘要化 / 命令分类 / 响应式压缩 / 并行工具」这些**架构模式**是可以直接映射的。

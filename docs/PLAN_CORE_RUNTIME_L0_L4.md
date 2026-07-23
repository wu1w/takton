# Takton 核心运行时打磨计划（L0–L4）

> 状态：已立项 · 2026-07-23  
> 目标：**同模型更快更聪明**，且**功能性可超越 Hermes**（能力走 sidecar，不污染默认主脑）  
> 方法：主脑优先双轨（Core-first, Capability-sidecar）  
> 基线 commit 参考：`ec8d635`（workspace 根修复）及此前 dynamic/检索分阈值系列  

---

## 0. 总原则

| # | 原则 |
|---|------|
| 1 | **主脑默认智力面**永远是 coding/assistant 级密度，禁止 80 工具默认进 schema |
| 2 | **功能超越 = pack 可完成任务**，且 coding 题集分数不回退 |
| 3 | 每层交付必须带 **验收指标**；无 bench 的「感觉更好」不算完成 |
| 4 | 最小变更：能加模块/钩子不先巨石重构 `loop.py`；行为对了再拆文件 |
| 5 | 同模型 A/B（固定 `mimo-v2.5` 或主人指定模型）为智力判决 |

### 双轨

```
轨道 A 主脑 Runtime     → 同模型更快更聪明（L0–L3）
轨道 B 能力 Sidecar     → 功能超 Hermes（L4，且每项回归 A）
```

---

## 1. 成功标准（90 天）

### 1.1 同模型智力 / 速度

| 指标 | 目标 |
|------|------|
| coding 题集正确率 | ≥ 本机 Hermes 同题（或自我 baseline +15% 相对提升） |
| 平均 prompt_tokens / 题 | ≤ Hermes coding 面；相对 Takton `full` 降 ≥40% |
| 空转轮次（同参重复+空正文） | 低于 baseline |
| 中位墙钟 / 题 | ≤ baseline；体感首包不差于乐观发送后的 Hermes |
| 工具命中率（该读则读） | ≥ baseline 且接近 Hermes |

### 1.2 功能超越

- 至少 **3** 个 Hermes 弱/无方向 pack 全链路可用：建议 `devices` / `desktop` / `evolution`
- 默认 profile 下 **coding 题集零回退**
- 设置中可一键 `full` 兼容旧习惯

### 1.3 非目标（本计划不做）

- 先抄 Hermes 全通道 gateway 形态
- 为超越而堆设置页
- 无指标大拆 loop 纯重构
- 默认打开 auto-cluster

---

## 2. 阶段总览

| 阶段 | 名称 | 预估 | 依赖 | 产出 |
|------|------|------|------|------|
| **L0** | 测量台 | 0.5–1 d | — | bench 脚本 + baseline 报告 |
| **L1** | 循环硬化 | 2–3 d | L0 可并行 | Budget/Retry/结果契约/telemetry |
| **L2** | 默认智力面 | 1–2 d | L1 中后段 | profile 产品化 + skill 纪律 |
| **L3** | 运行时契约 | 3–5 d | L2 | workspace 文件契约 + hooks + checkpoint |
| **L4** | 能力超车 | 持续 | L2 起可并行 | 3+ pack 闭环 + 回归门禁 |

**建议开工序：L0 → L1 → L2 → L3；L4 从 L2 起 sidecar 并行。**

---

## 3. L0 — 测量台（Bench）

### 3.1 目标

让「更聪明/更快」可复跑、可挡回归。

### 3.2 任务

| ID | 任务 | 验收 |
|----|------|------|
| L0.1 | 目录 `backend/bench/` 或 `scripts/bench_agent/` | 可从 repo 根一键跑 |
| L0.2 | 题集 v1（≥10 题）：读文件、改文件、grep、拒编造、短寒暄、长指令、错误路径 | JSON/YAML 题面 + 期望 |
| L0.3 | Runner：接 OpenAI-compatible（OpenCode Go / 本地），跑 Takton tool loop 或半环（schema+多轮 tool） | 输出 jsonl |
| L0.4 | 指标：`prompt_tokens`, `rounds`, `tool_names`, `wall_s`, `pass`, `empty_retries`, `thrash` | 汇总表 md |
| L0.5 | 锁 baseline：`docs/bench/baseline_YYYYMMDD.md`（模型名、commit、分数） | 入库或 workspace 存档 |
| L0.6 | （可选）Hermes 同题导出对比脚本 | 有对比列即可 |

### 3.3 交付物

- `scripts/bench_agent/run_bench.py`（或等价）
- `scripts/bench_agent/cases_v1.yaml`
- 一份 baseline 报告

### 3.4 完成门禁

- 一键跑通 ≥10 题；失败不静默
- baseline 数字可复现（同模型同 commit 方差可接受）

---

## 4. L1 — 循环硬化（主菜）

### 4.1 目标

对齐 Hermes 循环的「预算 + 失败分类 + 可恢复」，减少空转、提高闭环率。

### 4.2 任务

| ID | 任务 | 说明 | 验收 |
|----|------|------|------|
| L1.1 | `IterationBudget` | 独立小模块；`max_iterations` + `remaining` + `consume` + `refund` | 单测；loop 接入 |
| L1.2 | Grace 终答 | 预算耗尽后再给 **1** 次无工具/强制总结 | 不出现无限空转 |
| L1.3 | `TurnRetryState` | 分类：`empty_content` / `empty_tool_name` / `truncated_tool` / `timeout` / `rate_limit` / `content_filter` | 每类有上限与动作 |
| L1.4 | 与现有逻辑合并 | 空回复重试、ToolRepeatGuard、LLM retry **收口到状态机**，避免双轨打架 | 旧测试仍绿 |
| L1.5 | Tool 结果契约 | 统一 max 长度、错误前缀、`transient|fatal`；禁止 None | 单测 |
| L1.6 | Finish 纪律（轻） | system 短句：有工具证据才可宣称完成；编码任务优先 read→edit | bench 读文件题不降 |
| L1.7 | Telemetry | 每轮 status/log：`profile, packs, tools_n, tokens_est, retry, budget` | 日志可 grep |
| L1.8 | Bench 对比 | L1 后跑 L0 题集 vs baseline | 正确率不降；空转或 tokens 改善 |

### 4.3 关键代码落点（预期）

```
backend/agent/iteration_budget.py      # 新
backend/agent/turn_retry.py            # 新
backend/agent/tool_result_contract.py  # 新或并入 robust.py
backend/agent/loop.py                  # 接线（最小 diff）
backend/tests/test_loop_budget_retry.py
```

### 4.4 完成门禁

- 相关单测全绿
- L0 题集：pass ≥ baseline；空转指标改善或持平且 tokens 不恶化
- 无「只有 console、用户不可见」的终态失败（衔接已有 toast 路径）

### 4.5 明确不做

- 大拆 loop 文件物理重构（可列 L3.x 后续）
- 上 auto-cluster

---

## 5. L2 — 默认智力面

### 5.1 目标

产品默认 = 高密度主脑；全量能力显式选择。

### 5.2 任务

| ID | 任务 | 验收 |
|----|------|------|
| L2.1 | Profile 语义钉死 | `coding` / `assistant` / `ops` / `full`（`dynamic` 可作为 coding 的场景加包模式） |
| L2.2 | 默认配置 | `agent_tool_profile` 默认 `coding` 或 `dynamic`（上限=coding+scene，文档写清） |
| L2.3 | 设置 UI / API | 可切换 profile；session 可覆盖 | 切换后下一轮 schema 变 |
| L2.4 | 核心工具 description 审计 | file/edit/grep/command/web 全 Hermes 级 | 抽检表 |
| L2.5 | Skill 纪律 | 索引短列表 + 匹配 MUST load；入口工具始终在 coding 面 | 单测 + bench |
| L2.6 | use_tool_pack 保留 | ops/desktop 等不进默认；模型可扩 | desktop 场景题 optional |
| L2.7 | Bench | coding 题 tokens 相对 full 降 ≥40%（已有量级则锁回归） | 报告 |

### 5.3 完成门禁

- 新会话默认工具数 ≤ 25（无 scene 加包时）
- full 一键恢复 ≥ 注册工具全量行为
- L0 正确率不回退

---

## 6. L3 — 运行时契约

### 6.1 目标

对齐 OpenClaw「workspace + hook」；防止智力面被产品回潮冲掉。

### 6.2 任务

| ID | 任务 | 验收 |
|----|------|------|
| L3.1 | 首轮契约文件 | 注入 `AGENTS.md` / `SOUL.md` / `USER.md`（缺则 marker；大则截断） | 与 OpenClaw 规则同级文档 |
| L3.2 | 截断与标记规范 | 单文件 max chars；`[truncated]` / `[missing]` | 单测 |
| L3.3 | `before_tool_call` hook | 可 block / 改参；内置：权限、确认 | 插件或 callbacks 列表 |
| L3.4 | `after_tool_call` hook | 截断、审计日志、红线结果 | 单测 |
| L3.5 | 最小 checkpoint | 写文件类工具前：git diff 快照或 `.takton/checkpoints/` | 能恢复一次误写 |
| L3.6 | Session 写路径加固 | 与现有 lock 对齐；工具结果落库顺序 | 并发冒烟 |
| L3.7 | 文档 | `docs/CORE_RUNTIME.md`：主脑边界 vs capability | 存在且与代码一致 |

### 6.3 完成门禁

- hook 可单测拦截危险/越权
- checkpoint 手动验收 1 条 happy path
- L0/L1 回归绿

### 6.4 可延后

- 完整插件市场
- 物理拆分 loop.py 到多文件（行为稳定后）

---

## 7. L4 — 能力 Sidecar（功能超 Hermes）

### 7.1 目标

能力图谱超越 Hermes **弱项**，且默认主脑零回退。

### 7.2 候选 pack（按优先级）

| 优先级 | Pack | 超越点 | 验收场景 |
|--------|------|--------|----------|
| P1 | `devices` | 多机 takton-agent | 列表设备 + remote 一条命令 |
| P1 | `desktop` | UIA/键鼠本机桌面 | 截图或点击一条可控流程 |
| P1 | `evolution` | 任务经验资产运营 | 生成/列表/应用一条 skill 草稿 |
| P2 | `ops` | cron/channel/mcp 一站 | 创建并列出一条 cron |
| P2 | `office` | ppt/doc/tts | 生成一个 md/docx |
| P3 | 通道/Gateway 对齐 | 仅当主脑已稳 | 不做本阶段主线 |

### 7.3 每个 pack 的标准工序（Definition of Done）

1. 工具在 registry 可用、description 合格  
2. 仅通过 profile/pack/use_tool_pack 暴露  
3. **1 条 E2E 或 bench 题**  
4. 跑 L0 coding 题集 **零回退**  
5. 文档：何时用、如何开  

### 7.4 完成门禁（阶段里程碑）

- 任意 **3** 个 P1 pack 达 DoD  
- 发布说明只写能力与开关，不写本地阴沟细节  

---

## 8. 时间线（建议）

| 周 | 焦点 | 出口 |
|----|------|------|
| W1 | L0 + L1.1–L1.5 | baseline + budget/retry 接线 |
| W2 | L1.6–L1.8 + L2 | 题集对比报告 + 默认 profile |
| W3 | L3.1–L3.5 | 契约文件 + hooks + checkpoint MVP |
| W4 | L4×2 pack + 加固 | 2 个 pack DoD + 回归 |
| W5–6 | L4 第 3 pack + 打磨 | 90 天指标中期复查 |
| W7–12 | 按 bench 弱项迭代 / 可选 loop 拆分 | 90 天验收 |

*单人节奏可按 0.7x 拉长；并行有前端时可 UI 与 L2.3 并行。*

---

## 9. 每层「开始 / 完成」检查单

### 开始任何 Lx 前

- [ ] 读本计划与 `代码规范.MD`
- [ ] `git status` 干净或明确 WIP 分支策略（本项目习惯：main 直达）
- [ ] 跑一遍 L0（L0 自身除外）

### 完成任何 Lx 后

- [ ] 单测绿  
- [ ] L0 题集报告贴到 `docs/bench/` 或 PR/commit message  
- [ ] 更新本节「进度日志」  
- [ ] 不把密钥写入仓库  

---

## 10. 进度日志

| 日期 | 阶段 | 变更 | commit | bench 摘要 |
|------|------|------|--------|------------|
| 2026-07-23 | 立项 | 计划成文；此前已完成 dynamic/分阈值/workspace 根 | ec8d635 及之前 | mimo 读 backend 路径 pass；dynamic tokens≪full |
| 2026-07-23 | L0 | bench 双模型 runner + 10 题 + baseline 报告 | (本批) | mimo 10/10；kimi-for-coding 9/10 |
| 2026-07-23 | L1 | IterationBudget + TurnRetryState + tool_result_contract 接入 loop | (本批) | 单测 11 项相关绿 |

---

## 11. 风险与缓解

| 风险 | 缓解 |
|------|------|
| loop 巨石难改 | 只加模块+最小接线；禁止顺手重构 |
| 分数尺度不一（RAG） | 档位阈值可配置；bench 不依赖 RAG 题为主 |
| 功能需求打断主脑 | L4 任何合并强制 L0 回归 |
| 同模型 API 波动 | baseline 多次取中位；记录 model id |
| 密钥泄露 | bench 只用 env；不写 `.env` 进 git |

---

## 12. 立刻执行的第一刀（开工命令）

1. **L0.1–L0.5**：bench 骨架 + 10 题 + 打 baseline（mimo-v2.5）  
2. **L1.1–L1.4**：`IterationBudget` + `TurnRetryState` 接入 loop  
3. 用 L0 对比，再开 L2  

主人指令口令（可选）：

- `做 L0` / `做 L1` / `做 L0+L1`  
- `L4 devices` 等单独开 pack  

---

## 13. 一句话

> **先量（L0），再稳环（L1），再钉默认智力面（L2），再契约化（L3），功能只走 sidecar 且回归门禁（L4）——用同模型 bench 证明比 Hermes 更快更聪明，用 pack 证明功能超越。**

# 高强度鲁棒性压测报告

- 时间：2026-07-23T142040Z
- Commit 基线：d8a795e 及主脑 L0–L4
- 题集：`scripts/bench_agent/cases_stress.yaml`（10 题）
- Profile：coding（~20 tools）
- max_rounds：12
- 模型：mimo-v2.5（OpenCode Go）+ kimi-for-coding

## 总成绩

| 模型 | Pass | 通过率 | 均墙钟 | 均 prompt tokens（末轮） |
|------|------|--------|--------|--------------------------|
| **mimo-v2.5** | **10/10** | 100% | 11.8s | ~6679（含巨大 prompt 题拉高） |
| **kimi-for-coding** | **10/10** | 100% | 6.1s | ~5002 |

**合计 20/20，零失败。**

## 覆盖场景

| 题 ID | 压力类型 | mimo | kimi | 工具轨迹摘要 |
|-------|----------|------|------|----------------|
| stress_multi_tool_chain | 三工具链 | PASS | PASS | grep→read→command |
| stress_big_file_extract | ~165KB 文件+截断后 grep | PASS | PASS | file_read+grep |
| stress_many_glob_grep | 30 文件检索 | PASS | PASS | glob+grep |
| stress_huge_user_prompt | ~28KB 噪声用户消息 | PASS | PASS | command；prompt_tok≈18–20k |
| stress_error_recovery | 先失败再恢复 | PASS | PASS | bad read→grep |
| stress_pack_then_work | 扩包列表+编码 | PASS | PASS | use_tool_pack+read |
| stress_thrash_resist | 限制重复调用 | PASS | PASS | 单次 grep |
| stress_parallelish_reads | 连续 3 读 | PASS | PASS | 3×file_read |
| stress_command_pipeline | 多 shell | PASS | PASS | command |
| stress_long_horizon | 5 步长程 | PASS | PASS | glob+grep+read+command |

## 鲁棒性结论

1. **多工具编排**：两模型均能完成 3–4 步工具链并给出可验证终答。  
2. **大上下文**：超大用户噪声与大文件截断后，能改用 grep 找回 `SECRET_VALUE`。  
3. **错误恢复**：不存在路径失败后会换策略，不卡死。  
4. **Sidecar 边界**：list pack 后仍回到 coding 读源码，未拖入 desktop。  
5. **空转抑制**：thrash 题未出现同参连打超限。  
6. **速度**：同任务 kimi-for-coding 平均墙钟约为 mimo 的一半（本批）。  
7. **未压到的边界**（后续可加）：真桌面 click、真 remote 设备、evolution 引擎实读、并行 tool_calls 单轮扇出、人工 /stop 中断、OOM 级 200k+ 上下文。

## 复跑

```bash
set -a; source /opt/hermes-workspace/.secrets/bench_llm.env; set +a
cd /opt/hermes-workspace/takton
.venv311/bin/python scripts/bench_agent/run_bench.py \
  --models mimo,kimi \
  --cases scripts/bench_agent/cases_stress.yaml \
  --out docs/bench/stress \
  --max-rounds 12
```

原始明细：`bench_20260723T142040Z.jsonl`

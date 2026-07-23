# Bench report 20260723T132019Z

- workspace: `/opt/hermes-workspace/takton`
- tools_n: 20
- cases: 10
- models: kimi:kimi-for-coding

| model | pass | total | pass_rate | avg_wall_s | avg_prompt_tok |
|-------|------|-------|-----------|------------|----------------|
| kimi:kimi-for-coding | 9 | 10 | 90% | 5.3 | 3249 |

## Failures
- **kimi:kimi-for-coding / shell_echo**: ["missing any-token in ['TAKTON_BENCH_OK']"] final=''

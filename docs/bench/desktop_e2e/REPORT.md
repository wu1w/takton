# Desktop Pack E2E Report

- Host: M920X Linux + **Xvfb :99** (1280x720)
- Deps installed for test: `scrot`, `xdotool`, `imagemagick`
- Date: 2026-07-23

## Fixes found during E2E

1. **linux_adapter missing `import base64`**
2. **scrot won't overwrite empty mkstemp file** → unlink before capture
3. **Tool results as `str(dict)` broke consumers** → `json.dumps` for dict results
4. **Huge base64 in loop** → screenshot now saves to disk, returns `path` + `bytes`

## Results

### Tool-level (no LLM): **7/7 PASS**
pack gate, open_app, screenshot×2, click, type, scroll

### + Agent (mimo-v2.5): **8/8 PASS**
`agent_desktop_screenshot`: tools=`desktop_screenshot`, final=`屏幕截图成功。` (~11s)

### + Agent (kimi-for-coding): **8/8 PASS**
`agent_desktop_screenshot`: tools=`desktop_screenshot`, final=`成功截取了当前屏幕。` (~3s)

## Artifacts
- `screenshot.jpg` (~22KB)
- `desktop_e2e_*.json/md`
- Runner: `scripts/bench_agent/desktop_e2e.py`

## How to re-run

```bash
# if needed
Xvfb :99 -screen 0 1280x720x24 -ac &
export DISPLAY=:99
cd /opt/hermes-workspace/takton
.venv311/bin/python scripts/bench_agent/desktop_e2e.py --display :99 --with-agent --model mimo
```

## Limits
- Virtual framebuffer, not physical seat0/gdm session
- UIA snapshot is Windows-oriented; Linux path uses coordinates
- No visual assert on click target beyond tool success codes

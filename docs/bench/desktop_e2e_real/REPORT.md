# Desktop E2E — 真实物理显示 seat0 (`:0`)

- 时间：2026-07-23
- 机器：M920X Desktop Ubuntu
- Display：`:0` / Xorg (gdm) on tty1 seat0
- 分辨率探测：`640x480`
- Auth：`XAUTHORITY` 来自 `/run/user/128/gdm/Xauthority`（GDM greeter cookie）

## 会话事实（重要）

当前 **没有 wuyw 本地图形登录会话**。  
`loginctl` 显示 seat0 上是 **gdm greeter**（登录界面）；wuyw 会话为远程 SSH。  

因此本轮「真显示」E2E 打在 **物理座物理 X 服务器（GDM 登录屏）** 上，不是 Xvfb，也不是已登录的用户桌面。  
若要在完整 GNOME 用户桌面再验一次：在本机显示器登录 wuyw 图形会话后，用该会话的 `DISPLAY`/`XAUTHORITY` 复跑。

## 成绩

| 层级 | 结果 |
|------|------|
| 工具直调 | **7/7** |
| + Agent mimo-v2.5 | **8/8**（`desktop_screenshot` →「截图成功。」） |
| + Agent kimi-for-coding | **8/8**（~3.5s） |

步骤：pack 门闩 → open xclock → screenshot → click 屏幕中心 → type → scroll → screenshot → LLM 驱动截图。

截图产物：`screenshot.jpg`（~5.4KB，GDM 分辨率较小）。

## 复跑

```bash
sudo -S -p '' cp /run/user/128/gdm/Xauthority /tmp/gdm.xauth && sudo -S -p '' chmod 644 /tmp/gdm.xauth
export DISPLAY=:0 XAUTHORITY=/tmp/gdm.xauth
cd /opt/hermes-workspace/takton
.venv311/bin/python scripts/bench_agent/desktop_e2e.py --display :0 --out docs/bench/desktop_e2e_real --with-agent --model mimo
```

用户图形会话登录后（示例）：

```bash
export DISPLAY=:1   # 以实际为准
export XAUTHORITY=$HOME/.Xauthority
.venv311/bin/python scripts/bench_agent/desktop_e2e.py --display "$DISPLAY" --with-agent --model mimo
```

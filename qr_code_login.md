# 无头环境二维码登录方案

## 背景

项目部署在 **无图形界面的 Ubuntu 服务器** 上，用户无法直接看到浏览器窗口中的登录二维码。需要实现：执行技能后，在 headless 模式下启动 Chromium → 截取登录二维码 → 将二维码图片发送到聊天框（OpenClaw WebUI / 飞书机器人等）→ 用户扫码 → 完成登录。

## 完整登录流程

```
用户请求登录
    │
    ▼
cli.py get-qrcode (headless Chromium)
    │
    ├─→ 启动 headless Chromium (或复用已有实例)
    ├─→ 导航到小红书首页，触发登录弹窗
    ├─→ Canvas 截取 QR 二维码 → PNG bytes
    ├─→ 保存为 /tmp/xhs/login_qrcode.png
    ├─→ 转换为 base64 data URL
    └─→ 返回 JSON: {qrcode_path, qrcode_data_url, message}
    │
    ▼
Skill 层 / 聊天平台
    │
    ├─→ OpenClaw WebUI: 直接用 Markdown ![img](data:image/png;base64,...) 内嵌
    ├─→ 飞书机器人: 读取 PNG 文件上传为图片消息
    └─→ 其他平台: 根据 API 选择 data URL 或文件路径
    │
    ▼
用户扫码
    │
    ▼
cli.py wait-login (阻塞等待，最多 120s)
    │
    ├─→ 轮询页面登录状态
    ├─→ 登录成功 → 返回 {logged_in: true}
    └─→ 超时 → 返回提示重新 get-qrcode
```

### CLI 命令

```bash
# Step 1: 获取二维码（非阻塞，立即返回）
python scripts/cli.py get-qrcode
# 输出: {"qrcode_path": "/tmp/xhs/login_qrcode.png", "qrcode_data_url": "data:image/png;base64,...", "message": "..."}

# Step 2: 将 qrcode_data_url 展示给用户（在聊天框显示）
# ![小红书登录二维码]({qrcode_data_url})

# Step 3: 等待用户扫码（阻塞，默认 120s 超时）
python scripts/cli.py wait-login --timeout 120
# 成功: {"logged_in": true, "message": "登录成功"}
# 超时: {"logged_in": false, "message": "等待超时，请重新运行 get-qrcode 获取新二维码"}
```

多账号：

```bash
python scripts/cli.py --account work get-qrcode
python scripts/cli.py --account work wait-login
```

---

## 服务器环境部署指南

### 前置条件

- Python >= 3.11 + [uv](https://docs.astral.sh/uv/)
- Chromium 或 Google Chrome

### 安装 Chromium

```bash
# Ubuntu（推荐直接安装 Google Chrome，更稳定）
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt -f install -y

# 或安装 Chromium
sudo apt install -y chromium-browser
```

### 安装项目依赖

```bash
cd xiaohongshu-skills
uv sync
```

### 验证环境

```bash
# 确认 Chromium 可用
which google-chrome || which chromium || which chromium-browser

# 确认 headless 模式正常（应输出 "DevTools listening on ws://..."）
chromium --headless=new --remote-debugging-port=9222 --no-sandbox --disable-gpu 2>&1 | head -5

# 杀掉测试进程
pkill -f chromium

# 运行 get-qrcode
unset DISPLAY
python scripts/cli.py get-qrcode
```

---

## 常见问题排查

### 1. "无法启动 Chrome，请检查 Chrome 是否已安装"

**原因**：Chrome/Chromium 未安装，或 root 用户下缺少 `--no-sandbox` 参数。

**解决**：

```bash
# 确认已安装
which google-chrome || which chromium || which chromium-browser

# 确认 stealth.py 包含 --no-sandbox（项目已内置）
grep "no-sandbox" scripts/xhs/stealth.py
# 应输出: "--no-sandbox",

# 也可以通过环境变量指定浏览器路径
export CHROME_BIN=/usr/bin/chromium-browser
```

### 2. "等待 session 响应超时 (id=1001)"

**原因**：之前的 Chrome 进程残留 + 过期的 session tab 文件，代码尝试复用已失效的浏览器 tab。

**解决**：

```bash
# 杀掉残留 Chrome 进程
pkill -f chromium || pkill -f chrome || true

# 清除过期的 session 文件
rm -f /tmp/xhs/session_tab_*.txt /tmp/xhs/login_tab_*.txt

# 重新运行
python scripts/cli.py get-qrcode
```

### 3. headless=False（在服务器上却用了有窗口模式）

**原因**：`DISPLAY` 环境变量意外被设置（如通过 SSH X11 Forwarding），导致 `has_display()` 返回 True。

**解决**：

```bash
# 检查
echo $DISPLAY

# 清除
unset DISPLAY

# 再运行
python scripts/cli.py get-qrcode
```

### 4. Agent 传递了错误的 CLI 参数（如 --no-sandbox、--browser-path）

**原因**：AI Agent "脑补"了不存在的 CLI 参数。`--no-sandbox` 是 Chrome 的启动参数，已内置在 `stealth.py` 中，不需要通过 CLI 传递。

**解决**：正确的调用方式只有：

```bash
python scripts/cli.py get-qrcode              # 获取二维码
python scripts/cli.py wait-login               # 等待扫码
python scripts/cli.py --account NAME get-qrcode # 指定账号
```

不支持 `--no-sandbox`、`--browser-path`、`--headless` 等参数。

### 5. 万能清理三连（重置一切状态）

遇到任何 Chrome 相关的奇怪问题，执行：

```bash
pkill -f chromium || pkill -f chrome || true
rm -f /tmp/xhs/session_tab_*.txt /tmp/xhs/login_tab_*.txt
sleep 2
python scripts/cli.py get-qrcode
```

---

## 各聊天平台集成

### OpenClaw WebUI

直接用 Markdown 内嵌 data URL：

```markdown
![小红书登录二维码]({qrcode_data_url})
```

### 飞书机器人

飞书不支持 data URL，需上传 PNG 文件：

```python
# 1. 上传图片
with open(qrcode_path, "rb") as f:
    resp = requests.post(
        "https://open.feishu.cn/open-apis/im/v1/images",
        headers={"Authorization": f"Bearer {token}"},
        files={"image": f},
        data={"image_type": "message"},
    )
image_key = resp.json()["data"]["image_key"]

# 2. 发送图片消息
requests.post(
    "https://open.feishu.cn/open-apis/im/v1/messages",
    headers={"Authorization": f"Bearer {token}"},
    json={
        "receive_id": chat_id,
        "msg_type": "image",
        "content": json.dumps({"image_key": image_key}),
    },
)
```

### 其他平台

- 支持 Markdown / HTML → 用 `qrcode_data_url`
- 支持文件上传 → 用 `qrcode_path`（`/tmp/xhs/login_qrcode.png`）

# Hermes Agent v0.22.4 RISC-V 部署指南

> **目标**：在 em20-dk (riscv64) 板子上部署 Hermes Agent  
> **验证**：2026-11-02 · em20-dk · GitLab `agents/hermes-tps` commit `dfc6f98b`  
> **状态**：✅ 部署成功，Gateway 稳定运行

## 快速导航

| 步骤 | 说明 | 耗时 |
|------|------|------|
| [Step 0](#step-0--网络前置必做) | eth3 降速（必做） | 1 分钟 |
| [Step 1](#step-1--基础环境) | 系统依赖安装 | 5 分钟 |
| [Step 2](#step-2--rust-工具链) | Rust 1.99.1 | 5 分钟 |
| [Step 3](#step-3--libheif-编译) | libheif 源码编译 | 15 分钟 |
| [Step 4](#step-4--uv--python) | uv + Python 3.14 | 3 分钟 |
| [Step 5](#step-5--拉取代码) | 克隆代码 | 2 分钟 |
| [Step 6](#step-6--安装依赖) | Python 依赖安装 | 45 分钟 |
| [Step 7](#step-7--配置模型) | 配置模型与凭证 | 3 分钟 |
| [Step 8](#step-8--启动验证) | 启动与验证 | 2 分钟 |

---

## 环境要求

- **硬件**：em20-dk (riscv64)，8 核 / 9.6GB 内存 / 58GB 磁盘
- **系统**：Ubuntu 24.04 LTS
- **磁盘空间**：部署后约需 5-6 GB

---

## Step 0 — 网络前置（必做）

**问题**：板子千兆网口存在物理层错误，导致 HTTPS 超时（HTTP 正常）

**解决**：降速到 100Mbps

```bash
sudo ethtool -s eth3 speed 100 duplex full autoneg on
```

**验证**：

```bash
curl -sS -o /dev/null -w "%{http_code}\n" -m 15 https://github.com
# 应返回 200
```

**固化配置**（可选，防止重启失效）：

```bash
sudo tee /etc/systemd/system/eth3-speed.service > /dev/null << 'EOF'
[Unit]
Description=Set eth3 to 100Mbps
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -s eth3 speed 100 duplex full autoneg on
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable eth3-speed.service
```

---

## Step 1 — 基础环境

安装系统依赖（一条命令）：

```bash
sudo apt update
sudo apt install -y git curl build-essential python3-venv python3-dev \
  cmake pkg-config libde265-dev libx265-dev libaom-dev \
  libjpeg-dev libpng-dev zlib1g-dev
```

⚠️ **不要安装 `libheif-dev`**（Ubuntu 仓库版本太旧，会干扰 Step 3）

---

## Step 2 — Rust 工具链

**为什么需要**：3 个依赖需要从源码编译 Rust 扩展（`firecrawl-anydoc`、`cryptography`、`resvg-py`）

**安装 Rust 1.99.1**：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain 1.99.1
```

**关键：设置 PATH**（必须优先使用 rustup 的 cargo）：

```bash
export PATH="$HOME/.cargo/bin:$PATH"
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
```

**验证**：

```bash
cargo --version
# 必须输出：cargo 1.99.1 ...
```

---

## Step 3 — libheif 编译

**为什么需要**：`pillow-heif==1.6.0` 强制要求 libheif ≥ 1.23.1，Ubuntu 仓库只有 1.17.6

**下载源码**：

```bash
cd /tmp
curl -sSL -o libheif-1.23.1.tar.gz \
  https://github.com/strukturag/libheif/releases/download/v1.23.1/libheif-1.23.1.tar.gz
```

> 如果下载超时，可在办公电脑下载后 scp 到板子

**编译安装**（8 核约 15 分钟）：

```bash
cd /tmp
tar xzf libheif-1.23.1.tar.gz
cd libheif-1.23.1
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=ON \
         -DBUILD_TESTING=OFF -DWITH_EXAMPLES=OFF
make -j8
sudo make install
sudo ldconfig
```

**验证**：

```bash
pkg-config --modversion libheif
# 必须输出：1.23.1
```

---

## Step 4 — uv & Python

**安装 uv 0.13.4**：

```bash
curl -LsSf https://astral.sh/uv/0.13.4/install.sh | sh
source ~/.local/bin/env
uv --version
```

**安装 Python 3.14**：

```bash
uv python install 3.14
```

---

## Step 5 — 拉取代码

**SSH 方式**（推荐，需先配置 SSH key）：

```bash
cd ~
git clone --depth 1 ssh://git@gitlab.intchains.in:9002/agents/hermes-tps.git Hermes-agent-v0.22.40test
cd Hermes-agent-v0.22.40test
```

**SSH key 配置**（首次需要）：

```bash
ssh-keygen -t ed25519 -C "your.name@intchains.cn"
cat ~/.ssh/id_ed25519.pub
# 复制公钥到 GitLab → Preferences → SSH Keys
```

**验证**：

```bash
git log --oneline -1
# dfc6f98b docs: Add RISC-V riscv64 deployment guide
```

---

## Step 6 — 安装依赖

⚠️ **建议在 tmux 中执行**（防止 SSH 断线中断安装）：

```bash
sudo apt install -y tmux
tmux new -s install
```

**创建虚拟环境**：

```bash
cd ~/Hermes-agent-v0.22.40test
source ~/.local/bin/env
uv venv --python 3.14 .venv
```

**安装核心依赖**（约 30-45 分钟，其中 `firecrawl-anydoc` 约 27 分钟）：

```bash
uv pip install -e . "pillow-heif==1.6.0"
```

**安装消息平台依赖**（约 5 分钟）：

```bash
uv pip install -e ".[messaging,telegram]" "pillow-heif==1.6.0"
```

**安装飞书依赖**（约 4 分钟）：

```bash
uv pip install "lark-oapi==1.6.8" "qrcode==7.4.2"
```

**验证安装**：

```bash
.venv/bin/hermes --version
# Hermes Agent vgit.dfc6f98 (2026.9.24)
```

---

## Step 7 — 配置模型

**配置模型端点**：

```bash
cd ~/Hermes-agent-v0.22.40test
source ~/.local/bin/env

.venv/bin/hermes config set model.provider custom
.venv/bin/hermes config set model.base_url "http://llm.intchains.in:9000/v1"
.venv/bin/hermes config set model.api_key "<你的 API key>"
.venv/bin/hermes config set model.default "deepseek-v4-pro"
```

**配置飞书**（可选）：

```bash
.venv/bin/hermes config set FEISHU_APP_ID "cli_xxxxxxxx"
.venv/bin/hermes config set FEISHU_APP_SECRET "xxxxxxxx"
.venv/bin/hermes config set FEISHU_DOMAIN feishu
.venv/bin/hermes config set FEISHU_CONNECTION_MODE websocket
```

**配置 Telegram**（可选）：

```bash
.venv/bin/hermes config set TELEGRAM_BOT_TOKEN "123456:ABC-xxx"
```

> Telegram 需要代理，见文档末尾的「Telegram 代理配置」

**验证配置**：

```bash
.venv/bin/hermes config get model
```

---

## Step 8 — 启动验证

**启动 Gateway**：

```bash
cd ~/Hermes-agent-v0.22.40test
source ~/.local/bin/env
nohup .venv/bin/hermes gateway run > /tmp/gateway.log 2>&1 &
```

**验证运行状态**：

```bash
.venv/bin/hermes status
# 应显示：Gateway: ✓ running
```

**测试模型调用**：

```bash
.venv/bin/hermes -z "1+1等于几？只回答数字"
# 应返回：2
```

**测试文件工具**：

```bash
.venv/bin/hermes -z "读取 /etc/hostname 并只回答内容"
# 应返回：em20-dk
```

---

## 常见问题排查

| 症状 | 原因 | 解决方法 |
|------|------|----------|
| `git clone` / `curl https` 超时 | eth3 网络问题 | 执行 Step 0 降速命令 |
| `error: feature 'edition2024' is required` | 使用了系统 cargo 1.75 | `export PATH="$HOME/.cargo/bin:$PATH"` |
| `fatal error: libheif/heif_tiling.h: No such file` | 缺少 libheif 1.23.1 | 执行 Step 3 编译安装 |
| `No adapter available for feishu` | 缺少飞书依赖 | 安装 `lark-oapi` 和 `qrcode` |
| `telegram connect timed out` | Telegram 需要代理 | 配置代理（见下文） |
| `command not found: uv` | 环境未激活 | `source ~/.local/bin/env` |
| `command not found: hermes` | 使用 venv 中的 | `.venv/bin/hermes` |

**日志位置**：

```bash
~/.hermes/logs/agent.log      # 主日志
~/.hermes/logs/errors.log     # 错误日志
/tmp/gateway.log              # Gateway 输出
```

**查看最近错误**：

```bash
tail -50 ~/.hermes/logs/agent.log | grep -i error
```

---

## 已知限制

⚠️ **terminal 工具在 riscv64 上不可用**

- **原因**：上游代码只支持 arm64 和 x86_64 架构
- **影响**：Agent 无法执行 shell 命令、git 操作
- **不影响**：模型对话、文件读写、飞书/Telegram 收发、网页抓取

---

## 附录：Telegram 代理配置

Telegram 在国内需要代理访问。

**在办公电脑上启动中继**（假设本机代理监听 127.0.0.1:7890）：

```python
# proxy_relay.py
import socket, threading

LISTEN = ('0.0.0.0', 10808)
UPSTREAM = ('127.0.0.1', 7890)   # 改成你的代理端口

def transfer(src, dst):
    try:
        while True:
            data = src.recv(8192)
            if not data: break
            dst.sendall(data)
    except: pass

def handle(c):
    u = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    u.connect(UPSTREAM)
    t1 = threading.Thread(target=transfer, args=(c, u), daemon=True)
    t2 = threading.Thread(target=transfer, args=(u, c), daemon=True)
    t1.start(); t2.start(); t1.join(); t2.join()

srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(LISTEN); srv.listen(50)
print(f"relay {LISTEN} -> {UPSTREAM}")
while True:
    client, _ = srv.accept()
    threading.Thread(target=handle, args=(client,), daemon=True).start()
```

运行：

```bash
python3 proxy_relay.py
```

**在板子上配置代理**：

```bash
echo 'TELEGRAM_PROXY=http://<办公电脑IP>:10808' >> ~/.hermes/.env
.venv/bin/hermes gateway restart
```

---

## 环境速查

**关键版本**：
- Rust: 1.99.1
- Python: 3.14.8
- uv: 0.13.4
- libheif: 1.23.1

**关键路径**：
- 代码：`~/Hermes-agent-v0.22.40test`
- 配置：`~/.hermes/config.yaml`
- 凭证：`~/.hermes/.env`
- 日志：`~/.hermes/logs/`

**每次新终端激活环境**：

```bash
source ~/.local/bin/env
export PATH="$HOME/.cargo/bin:$PATH"
cd ~/Hermes-agent-v0.22.40test
```

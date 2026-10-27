# Hermes Agent v0.22.4 移植到 RISC-V 板子 —— 完整部署手册

> 面向对象：需要在自己机器上复现这次移植的同事
> 验证时间：2026-10-27 · 验证环境：em20-dk (riscv64) · Hermes Agent v0.22.4
> 结论：**移植成功**，Gateway 稳定运行，模型与飞书链路全部打通

---

## 目录

1. [目标机器与验证环境](#1-目标机器与验证环境)
2. [部署总流程速查](#2-部署总流程速查)
3. [Step 1 — 基础环境准备](#3-step-1--基础环境准备)
4. [Step 2 — Rust 工具链（关键前置）](#4-step-2--rust-工具链关键前置)
5. [Step 3 — 拉取 Hermes Agent v0.22.4](#5-step-3--拉取-hermes-agent-v0224)
6. [Step 4 — 创建虚拟环境与安装依赖](#6-step-4--创建虚拟环境与安装依赖)
7. [Step 5 — 配置模型与平台凭证](#7-step-5--配置模型与平台凭证)
8. [Step 6 — 启动与验证 Gateway](#8-step-6--启动与验证-gateway)
9. [已知问题与限制](#9-已知问题与限制)
10. [故障排查手册](#10-故障排查手册)
11. [附录：环境速查表](#11-附录环境速查表)
12. [验证结果汇总](#12-本次验证结果汇总)

---

## 1. 目标机器与验证环境

| 项目 | 值 |
|---|---|
| 型号 | em20-dk |
| 架构 | **riscv64** (riscv64gc-unknown-linux-gnu) |
| 操作系统 | Ubuntu 24.04 LTS (Noble Numbat) |
| 内核 | 6.6.0-tps |
| 内存 | 9.6 GB |
| 磁盘 | 58 GB（已用 6.5 GB，可用 49 GB） |
| CPU 核数 | 8 |
| Python | CPython 3.14.8 (linux-riscv64gc-gnu) |
| Rust | cargo / rustc **1.99.1**（rustup stable；最低要求 ≥ 1.88，系统自带 1.75.0 不可用） |
| Hermes | v0.22.4 (commit `4d04303818b61bf3aa09278af48356315515965b`) |

**重要前提**：RISC-V 上 Python 生态已经比较成熟，CPython 3.14 有官方 riscv64 预编译版本，**不需要源码编译 Python**。这是整个移植能快速完成的关键。

---

## 2. 部署总流程速查

```bash
# ① 基础工具
sudo apt update && sudo apt install -y git curl build-essential python3-venv python3-dev

# ② Rust 工具链（本机用 stable=1.99.1，最低要求 ≥1.88，见 Step 2）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
export PATH="$HOME/.cargo/bin:$PATH"   # ← 关键：让 cargo 走 rustup 的新版本

# ③ uv + Python 3.14
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env
uv python install 3.14

# ④ 拉取 Hermes
git clone --depth 1 --branch v0.22.4 \
  https://github.com/NousResearch/hermes-agent.git ~/hermes-agent-v0.22.4
cd ~/hermes-agent-v0.22.4

# ⑤ 安装依赖
uv venv --python 3.14 .venv
uv pip install -e .
uv pip install -e ".[messaging,telegram]"
uv pip install "lark-oapi==1.6.8" "qrcode==7.4.2"

# ⑥ 配置模型
hermes config set model.provider custom
hermes config set model.base_url "http://<你的网关>/v1"
hermes config set model.api_key "sk-xxx"
hermes config set model.default "claude-sonnet-5-5"

# ⑦ 启动
source ~/.local/bin/env
nohup .venv/bin/hermes gateway run > /tmp/gateway.log 2>&1 &
```

---

## 3. Step 1 — 基础环境准备

### 3.1 系统依赖

```bash
sudo apt update
sudo apt install -y git curl build-essential python3-venv python3-dev
```

`build-essential` 提供 gcc/g++/make，是编译 cryptography、pillow 等 C 扩展的必需项。

### 3.2 uv 包管理器

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env
uv --version    # 验证：0.13.4 (riscv64gc-unknown-linux-gnu)
```

> uv 官方提供 riscv64 预编译二进制，安装后位于 `~/.local/bin/uv`。
> **后续每条命令前都要 `source ~/.local/bin/env`**，否则找不到 uv。

### 3.3 Python 3.14

```bash
uv python install 3.14
```

验证：

```bash
uv python list | grep 3.14.8
# cpython-3.14.8-linux-riscv64gc-gnu
```

uv 会把解释器装到 `~/.local/share/uv/python/cpython-3.14.8-linux-riscv64gc-gnu/`。

---

## 4. Step 2 — Rust 工具链（关键前置）

### 4.1 为什么这一步不能跳过

Hermes v0.22.4 有 4 个依赖需要从源码编译。各包的**真实**要求（从 Cargo.toml 读出）：

| 包 | 用途 | 是否用 Rust | 各包自己声明的要求 |
|---|---|---|---|
| `firecrawl-anydoc==0.2.4` | PDF/Office 文档提取 | ✅ | `edition = "2024"`, `rust-version = "1.88"` |
| `resvg-py==0.4.0` | SVG 渲染 | ✅ | `edition = "2024"`, `Cargo.lock` v4 |
| `cryptography==50.0.1` | 加密库 | ✅ | `edition = "2021"`, `rust-version = "1.83.0"` |
| `pillow-heif>=1.4.0` | HEIF/HEIC/AVIF 图片解码 | ❌ | 纯 C/C++（libheif），**不需要 Rust** |

**PyPI 上这几个包都没有 riscv64 预编译 wheel**，只有 `aarch64` / `x86_64` / `ppc64le` / `s390x` 等架构的，所以 riscv64 只能源码编译。

### 4.1.1 版本号说明：实际用什么版本，建议多少以上

这是两个不同的问题，文档里都要说清楚：

| | 版本 | 含义 |
|---|---|---|
| **实际使用**（本机已验证） | **cargo / rustc 1.99.1** | rustup 的 `stable` 工具链，2026-10-26 安装。本次 v0.22.4 全流程**已完整验证通过** |
| **建议最低** | **≥ 1.88** | 四个包里要求最高的 `firecrawl-anydoc` 声明了 `rust-version = "1.88"`，低于此版本编译必失败 |
| **绝对不能用** | **1.75.0** | Ubuntu 24.04 系统自带版本，远低于门槛 |

**为什么实际是 1.99.1 而不是 1.88**：

`rustup toolchain install stable` 装的是**滚动升级的最新稳定版**，不是固定版本。
本机安装时（2026-10-26）stable 恰好是 1.99.1；同事在别的时间装，版本可能更高。

**为什么最低门槛是 1.88**：

拆开看每个约束：

| 约束 | 需要什么版本 |
|---|---|
| `edition = "2024"` | Rust **1.85.0**（2025-02-20 稳定，见[官方公告](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0.html)） |
| `Cargo.lock` v4 | Cargo **1.78+** |
| `cryptography` 的 `rust-version = "1.83.0"` | Rust **1.83.0** |
| `firecrawl-anydoc` 的 `rust-version = "1.88"` | Rust **1.88** ← **最高的那条** |

Cargo 会严格校验每个包自己声明的 `rust-version`（MSRV, Minimum Supported Rust Version），
低于声明值直接拒绝编译。取所有约束的最大值 → **实际门槛是 1.88**。

> 📌 **给同事的结论**：
> 直接 `rustup toolchain install stable` 装最新版即可（本机实测 1.99.1，验证通过），
> 只要 `cargo --version` **≥ 1.88** 就能编过。
> 想和本机完全一致就固定 `rustup toolchain install 1.99.1 && rustup default 1.99.1`。

### 4.2 ⚠️ 这里有一个极易踩的坑

Ubuntu 24.04 自带的 `cargo`（`/usr/bin/cargo`）版本是 **1.75.0**，远低于 1.88 门槛。实测用系统 cargo 会直接报错：

```
error: failed to parse manifest at `Cargo.toml`
Caused by:
  feature `edition2024` is required
  The package requires the Cargo feature called `edition2024`, but that feature
  is not stabilized in this version of Cargo (1.75.0).
```

同样地，`resvg-py` 带的 `Cargo.lock` 是 v4 格式，系统 cargo 也读不了：

```
error: failed to parse lock file at: `Cargo.lock`
Caused by:
  lock file version 4 requires `-Znext-lockfile-bump`
```

**只装 rustup 还不够** —— rustup 默认装到 `~/.cargo/bin/`，但这个目录**不在默认 PATH 里**，`which cargo` 仍然指向 `/usr/bin/cargo`（1.75.0）：

```bash
$ which -a cargo
/usr/bin/cargo          # ← 1.75.0，系统自带，会编译失败
/bin/cargo
```

### 4.3 正确做法

```bash
# 安装 rustup（自带最新 stable 工具链，本机装到的是 1.99.1）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 关键：把 rustup 的 bin 目录加到 PATH 最前面
export PATH="$HOME/.cargo/bin:$PATH"

# 验证：版本必须 ≥ 1.88
cargo --version     # → cargo 1.99.1（本机实测，已验证通过）
rustc --version     # → rustc 1.99.1
```

如果想和本机**完全一致**（团队环境统一）：

```bash
rustup toolchain install 1.99.1
rustup default 1.99.1
```

### 4.4 让 PATH 永久生效

```bash
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

> **注意**：`~/.cargo/env` 也可以 source，但本机 `.bashrc` 里没有自动引入，
> 所以每次开工都要手动确认 `cargo --version` ≥ 1.88。

### 4.5 编译耗时参考

在 **8 核 em20-dk** 板子上实测（**已用 `--no-cache` 干净环境完整复现**）：

| 包 | 依赖 crate 数 | 源码编译耗时 |
|---|---|---|
| `firecrawl-anydoc==0.2.4` | 163 | **26 分 48 秒** |
| `pillow-heif>=1.4.0` | — | 约 3–8 分钟 |
| `resvg-py==0.4.0` | — | 约 2–5 分钟 |
| `cryptography==50.0.1` | — | 约 5–10 分钟 |

> 验证记录：
> ```
> Building firecrawl-anydoc==0.2.4
>    Built firecrawl-anydoc==0.2.4
> Prepared 1 package in 26m 48s
> Installed 1 package in 4ms
> + firecrawl-anydoc==0.2.4
> ```
> 产物 `anydoc/_anydoc.abi3.so` 可正常 `import anydoc`。

第一次装完这些包后，uv 会把编译好的 wheel 缓存到 `~/.cache/uv/`，后续安装会**秒级完成**。
所以第二次重装（即使换 Hermes 版本）几乎不花时间。

---

## 5. Step 3 — 拉取 Hermes Agent v0.22.4

```bash
cd ~
git clone --depth 1 --branch v0.22.4 \
  https://github.com/NousResearch/hermes-agent.git hermes-agent-v0.22.4

cd hermes-agent-v0.22.4
git log -1 --oneline   # 4d04303818b6
```

**关于 `--depth 1 --branch v0.22.4`**：既能拿到精确版本，又避免完整历史（完整仓库约 365 MB）。想看全部 tag 可以用 `git ls-remote --tags origin | grep v0.22`。

### 5.1 网络问题（遇到过，需注意）

板子上千兆网口（eth3）存在物理层错误，导致 **HTTPS 请求超时而 HTTP 正常**：

```
rx_frame_check_sequence_errors: 11703+
```

症状：`git clone` 挂住、`pip install` 下载超时，但 `http://` 请求正常。

解决办法是降速：

```bash
sudo ethtool -s eth3 speed 100 duplex full autoneg on
```

> ⚠️ **重启后会恢复 1000Mbps，需要重新执行**。
> 永久方案可以写个 systemd 服务，但不影响本次部署。

---

## 6. Step 4 — 创建虚拟环境与安装依赖

### 6.1 创建 venv

```bash
cd ~/hermes-agent-v0.22.4
source ~/.local/bin/env
uv venv --python 3.14 .venv
```

### 6.2 安装核心依赖

```bash
uv pip install -e .
```

安装约 90 个包。**这一步会触发 Rust 源码编译，耗时最长（25–40 分钟）**。

### 6.3 安装消息平台依赖

```bash
uv pip install -e ".[messaging,telegram]"
```

安装 telegram/discord/slack 相关包，约 7 分钟。

### 6.4 安装飞书依赖（v0.22.4 新增要求）

```bash
uv pip install "lark-oapi==1.6.8" "qrcode==7.4.2"
```

> **注意**：v0.22.4 把飞书拆成了独立的 `feishu` extra，
> **不会**被 `.[messaging]` 带出来。漏掉这步 Gateway 会报：
>
> ```
> WARNING gateway.platform_registry: Platform 'Feishu / Lark' requirements not met
> ERROR gateway.run: Platform 'feishu' is registered but adapter creation failed
> WARNING gateway.run: No adapter available for feishu
> ```

### 6.5 安装结果验证

```bash
source ~/.local/bin/env
.venv/bin/hermes --version
```

预期输出：

```
Hermes Agent v0.22.4 (2026.9.24) · stable · local 4d04303818b6
Install directory: /home/leo/hermes-agent-v0.22.4
Install method: git
Python: 3.14.8
OpenAI SDK: 2.24.0
```

---

## 7. Step 5 — 配置模型与平台凭证

### 7.1 配置放在哪里

**所有配置都在 `~/.hermes/` 目录，与代码安装位置无关**：

```
~/.hermes/
├── config.yaml    # 非敏感设置：model / provider / base_url / terminal / 压缩策略 …
├── .env           # 敏感凭证：API key / bot token / 密码
├── auth.json      # OAuth 凭证（Nous Portal 等）
└── logs/          # 运行日志
```

解析路径在 `hermes_cli/config.py`：

```python
def get_config_path() -> Path:
    return get_hermes_home() / "config.yaml"
```

而 `get_hermes_home()` 的优先级是：

```
context override → HERMES_HOME 环境变量 → 平台默认 ~/.hermes
```

所以**把 Hermes 装到任何目录，读的都是同一份 `~/.hermes/config.yaml`**。
换版本目录时配置自动继承，无需重新配置。

> 项目目录下也可以放 `.env` / `.hermes/` 做覆盖，但本次部署**没有使用**，
> 板上 `~/hermes-agent-v0.22.4/.env` 和 `~/hermes-agent-v0.22.4/.hermes` 都不存在。

### 7.2 官方配置优先级

来自官方文档
[Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)：

| 优先级 | 来源 | 说明 |
|---|---|---|
| 1 | **CLI 参数** | `hermes chat --model anthropic/claude-opus-4`，仅本次调用生效 |
| 2 | **`~/.hermes/config.yaml`** | 所有非敏感设置的主配置 |
| 3 | **`~/.hermes/.env`** | 环境变量回退；**密钥必须放这里** |
| 4 | **内置默认值** | 硬编码的安全默认值 |

规则：**密钥进 `.env`，其余进 `config.yaml`**。两者都设置时，非敏感项 `config.yaml` 优先。

### 7.3 本次使用的模型配置（如何复现）

原配置是用 `hermes config set` 命令逐项写入的（在 Mac 的 shell 历史里可以查到原始记录）：

```bash
cd ~/hermes-agent-v0.22.4
source ~/.local/bin/env

hermes config set model.provider custom
hermes config set model.base_url "http://llm.intchains.in:9000/v1"
hermes config set model.api_key "sk-OgTJPH93u2XXoh5WNZNuUQ"
hermes config set model.default "claude-sonnet-5-5"
```

`hermes config set` 会自动决定写入哪个文件：
**点号路径（`model.base_url`）写 `config.yaml`，大写下划线名（`OPENROUTER_API_KEY`）写 `.env`**。

验证：

```bash
hermes config get model
```

```
default: claude-sonnet-5-5
provider: custom
base_url: http://llm.intchains.in:9000/v1
api_key: sk-O...NuUQ
```

### 7.4 关于 `llm.intchains.in` 这个网关

这是**公司内部自建的 OpenAI 兼容网关**（nginx 1.14.1），不是 Hermes 官方服务。

- DNS：`llm.intchains.in` → `intbs.intchains.in` → `139.227.52.190`（公网）
  （Mac 上经内网 DNS 解析到 `172.16.1.202`）
- 鉴权：`Authorization: Bearer <api_key>`
- 暴露 28 个模型：

```
MiniMax-M2.7-highspeed, MiniMax-M3, MiniMax-M3.1-Flash, MiniMax-M3.1-Flash-OpenAI,
claude-haiku-5-5, claude-opus-5, claude-opus-5-5, claude-sonnet-5, claude-sonnet-5-5,
deepseek-v4-flash, deepseek-v4-pro, deepseek-v4-pro-0813,
deepseek-v4.1-flash, deepseek-v4.1-flash-1017,
glm-5.2, glm-5.3,
gpt-5.6-luna, gpt-5.6-sol, gpt-5.6-terra, gpt-6-astra, gpt-6-luna, gpt-6-sol, gpt-6.1-sol,
kimi-k3, qwen3.6-plus, qwen3.7-max, qwen3.7-plus, qwen3.8-max
```

> 换成自己的网关见 [7.5](#75-换成自己的模型网关)。

### 7.5 换成自己的模型网关

三种方式，按推荐程度排列：

**方式 A：改配置（最快，适合已知 base_url + key）**

```bash
hermes config set model.provider custom
hermes config set model.base_url "http://<你的地址>/v1"
hermes config set model.api_key "<你的 key>"
hermes config set model.default "<模型名>"
hermes gateway restart
```

**方式 B：交互式向导（适合从零配置，会拉取端点实时模型列表）**

```bash
hermes model
# 选 "Custom endpoint"，按提示填 base_url / api_key
```

**方式 C：用官方托管的 provider**

如果是通用场景，用官方 provider 更省事，不需要自己搭网关：

```bash
# 例如 OpenRouter
hermes config set OPENROUTER_API_KEY sk-or-...   # 注意：大写下划线名会写进 .env
hermes config set model.provider openrouter
hermes config set model.default anthropic/claude-sonnet-4
hermes gateway restart
```

> `hermes config set` 会**自动决定写入哪个文件**：
> 点号路径（`model.base_url`）→ `config.yaml`；
> 大写下划线名（`OPENROUTER_API_KEY`）→ `.env`。

支持的 provider 完整列表见官方文档
[AI Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)，
覆盖 OpenRouter、Anthropic、Nous Portal、MiniMax、Kimi、GLM、Fireworks、
GitHub Copilot、以及 Ollama / vLLM / llama.cpp 等自建服务。

### 7.6 辅助模型（auxiliary）

Hermes 的部分后台任务（图片识别、网页摘要、标题生成、上下文压缩）
默认跟随主模型，但可以单独指定更便宜的模型：

```yaml
auxiliary:
  vision:
    provider: "openrouter"
    model: "google/gemini-2.0-flash"
```

> 本次部署保持默认（`provider: "auto"`），日志中可见
> `Vision auto-detect: using main provider custom (claude-sonnet-5-5)`。

### 7.7 api_key 的查找顺序（代码依据）

在 `agent/auxiliary_client.py` 的 `_resolve_custom_branch()` 中，custom provider 的
key 按此顺序取：

```
CLI 显式参数  →  OPENAI_API_KEY 环境变量  →  config.yaml 的 model.api_key  →  "no-key-required"
```

本机 **`OPENAI_API_KEY` 并不存在**（`.env`、`.bashrc`、`.profile` 里都查不到），
唯一生效的就是 `config.yaml` 里 `model.api_key` 那一个。

> 顺带一提：`model.api_key` 写在 config.yaml 里不算理想实践，
> 更规范的做法是把它挪到 `.env` 的 `OPENAI_API_KEY`（`provider: custom` 时会自动读）。
> 本次为了保持与原环境一致没有改动。

### 7.8 平台凭证（飞书 / Telegram）

放在 `~/.hermes/.env`：

```bash
# 飞书
FEISHU_APP_ID=cli_xxxxxxxx
FEISHU_APP_SECRET=xxxxxxxx
FEISHU_DOMAIN=feishu              # 国内用 feishu，海外用 lark
FEISHU_CONNECTION_MODE=websocket   # 或 webhook
FEISHU_ALLOW_ALL_USERS=false
FEISHU_GROUP_POLICY=open

# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC-DefGhiJklMnoPqrstuVwxyz    # 从 @BotFather 获取
TELEGRAM_ALLOWED_USERS=12345678,87654321                 # 可选：限制使用的 user_id 列表（逗号分隔）
```

从 Mac 拷贝配置：

```bash
scp ~/.hermes/.env    leo@<板子IP>:~/.hermes/.env
scp ~/.hermes/config.yaml leo@<板子IP>:~/.hermes/config.yaml
```

---

## 8. Step 6 — 启动与验证 Gateway

### 8.1 启动

```bash
cd ~/hermes-agent-v0.22.4
source ~/.local/bin/env
nohup .venv/bin/hermes gateway run > /tmp/gateway.log 2>&1 &
```

也可以装成开机自启的用户服务：

```bash
hermes gateway install
hermes gateway start
```

### 8.2 验证清单

```bash
# ① 版本
.venv/bin/hermes --version

# ② 总体状态
.venv/bin/hermes status
```

预期：
```
  Model:        claude-sonnet-5-5
  Provider:     custom
  Providers:    MiniMax (China), Custom endpoint
  Gateway:      ✓ running
  Platforms:    Feishu / Lark, Telegram
```
```bash
# ③ Gateway 进程
.venv/bin/hermes gateway status
```

预期：`✓ Gateway is running (PID <xxx>)`

```bash
# ④ 平台连接详情（读 gateway_state.json）
python3 -c "
import json
d=json.load(open('/home/leo/.hermes/gateway_state.json'))
print('gateway:', d.get('gateway_state'), '| ver:', d.get('code_version'))
for k,v in d.get('platforms',{}).items():
    print(' ', k, '->', v.get('state'), v.get('error_message') or '')
"
```

预期：

```
gateway: running | ver: 0.22.4
  feishu -> connected
  telegram -> connected
```

### 8.3 端到端模型调用测试

```bash
cd ~/hermes-agent-v0.22.4 && source ~/.local/bin/env
.venv/bin/hermes -z "1+1等于几？只回答数字"
# → 2
```

日志中确认实际生效的 provider / model / base_url：

```bash
grep "OpenAI client created" ~/.hermes/logs/agent.log | tail -2
```

```
OpenAI client created (agent_init, shared=True)
  provider=custom base_url=http://llm.intchains.in:9000/v1 model=claude-sonnet-5-5
```

### 8.4 直接验证端点（排查模型问题时用）

```bash
# 看可用模型
curl -sS -H "Authorization: Bearer $API_KEY" http://llm.intchains.in:9000/v1/models

# 直接发一次 chat 请求
curl -sS -X POST http://llm.intchains.in:9000/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-5-5","messages":[{"role":"user","content":"say OK only"}],"max_tokens":20}'
```

### 8.5 验证飞书消息发送

```bash
set -a; . ~/.hermes/.env; set +a
TOKEN=$(curl -sS -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$FEISHU_APP_ID\",\"app_secret\":\"$FEISHU_APP_SECRET\"}" \
  | python3 -c "import sys,json;print(json.load(sys.stdin).get('tenant_access_token',''))")

curl -sS -X POST "https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"receive_id":"oc_xxxxx","msg_type":"text","content":"{\"text\":\"测试\"}"}'
```

返回 `{"code":0,"msg":"success"}` 即成功。chat_id 可以从
`~/.hermes/channel_directory.json` 里查到。

---

## 9. 已知问题与限制

### 9.1 ⚠️ terminal 工具在 riscv64 上不可用（最重要）

Agent **无法执行 shell 命令**，报：

```
RuntimeError: unsupported architecture: riscv64
  File "pm/store.py", line 156, in current_target()
```

**根因**：`pm/store.py` 的 `current_target()` 只识别 arm64 和 x86_64，其他架构直接抛异常：

```python
def current_target() -> str:
    machine = _native_machine()
    if machine in ("arm64", "aarch64"):
        arch = "arm64"
    elif machine in ("x86_64", "amd64", "x64"):
        arch = "x64"
    else:
        raise RuntimeError(f"unsupported architecture: {platform.machine()}")
```

**调用链**：
```
tools/environments/local.py:597  _user_tools_first()
  → pm/user_facing.py            user_facing_path()
  → pm/install.py:935            _store_path_dirs()
  → pm/store.py:156              current_target()  ← 抛异常
```

**这不是 v0.22.4 引入的回归** —— 旧版（main 分支）在同一块板子上有完全相同的报错
（`~/.hermes/logs/agent.log` 中 2026-10-27 05:40 的记录）。
属于**上游未支持 riscv64 架构**的问题。

**影响范围**：

| 能力 | 状态 |
|---|---|
| 模型对话 | ✅ 正常 |
| 文件读写 | ✅ 正常 |
| 飞书/Telegram 收发 | ✅ 正常 |
| 网页抓取 | ✅ 正常 |
| **执行 shell 命令** | ❌ 不可用 |
| **本地构建 / git 操作** | ❌ 不可用 |
| Computer Use（cua-driver） | ❌ 同一根因 |

**修复方向**（需要上游支持）：在 `pm/store.py` 的 `current_target()` 中加 riscv64 分支，
例如映射为 `linux-riscv64`；同时 `pm/lock.json` 需要有对应 target 的包清单。

如果只想让 terminal 工具在本机跑起来，可以尝试打这个补丁（**未验证**，
仅供参考，升级 Hermes 后可能被覆盖）：

```python
# pm/store.py, current_target()
def current_target() -> str:
    machine = _native_machine()
    if machine in ("arm64", "aarch64"):
        arch = "arm64"
    elif machine in ("x86_64", "amd64", "x64"):
        arch = "x64"
    elif machine in ("riscv64", "riscv64gc"):
        arch = "riscv64"          # ← 新增
    else:
        raise RuntimeError(f"unsupported architecture: {platform.machine()}")
    ...
```

> ⚠️ 只改这一处可能仍不够：`_store_path_dirs()` 会去 `pm/lock.json` 里找
> `linux-riscv64` target 的包清单，找不到时该函数返回空列表（不抛异常），
> 理论上可以退化到「只用系统 PATH」。但 `installed_package()`（Computer Use 检查）
> 仍会走 `current_target()`，需要一并确认。

### 9.2 飞书首次连接慢

飞书 websocket 模式首次连接会超时 30 秒失败，**重试 2~3 次才连上**。
Gateway 有自动重连机制，日志里会看到：

```
Connecting to feishu...
✗ feishu error: feishu connect timed out after 30s
Starting reconnection watcher for 1 failed platform(s): feishu
Reconnecting feishu (attempt 2)...
✓ feishu reconnected successfully
```

不影响使用，等待即可。

### 9.3 网卡重启后需重新降速

见 [5.1](#51-网络问题遇到过需注意)。

---

## 10. 故障排查手册

| 症状 | 原因 | 处理 |
|---|---|---|
| `error: feature 'edition2024' is required` | 用了系统 cargo 1.75 | `export PATH="$HOME/.cargo/bin:$PATH"`，确认 `cargo --version` ≥ 1.88（本机用 1.99.1） |
| `lock file version 4 requires -Znext-lockfile-bump` | 同上（resvg-py 的 Cargo.lock v4） | 同上 |
| `No adapter available for feishu` | 没装飞书依赖 | `uv pip install "lark-oapi==1.6.8" "qrcode==7.4.2"` 后重启 gateway |
| `unsupported architecture: riscv64` | terminal 工具不支持该架构 | 上游限制，暂无法规避；见 9.1 |
| `git clone` / `pip install` 超时 | 千兆网口物理层错误 | `sudo ethtool -s eth3 speed 100 duplex full autoneg on` |
| `401 No api key passed in` | 模型端点鉴权失败 | `hermes config get model` 检查 `model.api_key` |
| `401 app secret invalid` | 飞书 secret 错误 | 检查 `.env` 里 `FEISHU_APP_SECRET` |
| `Gateway is not running` 但进程在 | 状态文件残留 | `hermes gateway restart` |
| `command not found: uv` | 没 source 环境 | `source ~/.local/bin/env` |
| `command not found: hermes` | 用的是 venv 里的 | `~/hermes-agent-v0.22.4/.venv/bin/hermes` |

### 日志位置

```bash
~/.hermes/logs/agent.log      # 主日志（含每次 API 调用）
~/.hermes/logs/errors.log    # 错误日志
/tmp/gateway.log              # 本次启动的 stdout（用 nohup 启动时）
```

常用排查命令：

```bash
# 飞书连接情况
grep -i "feishu" ~/.hermes/logs/agent.log | tail -20

# 实际使用的 provider / model
grep "OpenAI client created" ~/.hermes/logs/agent.log | tail -3

# 最近的错误
tail -100 ~/.hermes/logs/agent.log | grep -iE "error|exception"
```

---

## 11. 附录：环境速查表

### 路径

| 用途 | 路径 |
|---|---|
| Hermes 代码 | `~/hermes-agent-v0.22.4` |
| 虚拟环境 | `~/hermes-agent-v0.22.4/.venv` |
| 配置 | `~/.hermes/config.yaml` |
| 凭证 | `~/.hermes/.env` |
| 日志 | `~/.hermes/logs/` |
| uv 缓存（编译好的 wheel） | `~/.cache/uv/` （实测占用 **1.9 GB**） |
| Python 解释器 | `~/.local/share/uv/python/cpython-3.14.8-linux-riscv64gc-gnu/` |
| Rust 工具链 | `~/.rustup/toolchains/stable-riscv64gc-unknown-linux-gnu/` （cargo/rustc **1.99.1**） |

### 磁盘占用

| 项目 | 大小 |
|---|---|
| `~/hermes-agent-v0.22.4`（含 .venv） | **689 MB** |
| `~/.cache/uv/`（编译产物缓存） | **1.9 GB** |
| Python 3.14 解释器 | ~100 MB |
| Rust 工具链 | ~1.5 GB |

> 磁盘紧张时可以删 `~/.cache/uv/` 释放 1.9 GB，代价是下次安装要重新编译。

### 常用命令

```bash
# 激活环境
source ~/.local/bin/env
export PATH="$HOME/.cargo/bin:$PATH"

# 进入项目
cd ~/hermes-agent-v0.22.4
source .venv/bin/activate

# 查看状态
hermes status
hermes status --full
hermes gateway status
hermes config get model

# 改模型
hermes config set model.default "claude-opus-5-5"
hermes gateway restart

# 日志
tail -f ~/.hermes/logs/agent.log
```

### 依赖安装耗时参考

| 步骤 | 耗时 | 备注 |
|---|---|---|
| `uv pip install -e .` | **25–40 分钟** | 首次需编译 Rust 扩展（anydoc 单独占 26 分 48 秒） |
| `uv pip install -e ".[messaging,telegram]"` | ~7 分钟 | 编译 pynacl / brotlicffi / tornado |
| `uv pip install lark-oapi qrcode` | <10 秒 | 纯 Python |
| （有缓存后）全部依赖 | **<1 分钟** | uv 会复用 `~/.cache/uv/` 里的 wheel |

---

## 12. 本次验证结果汇总

| 检查项 | 结果 |
|---|---|
| `hermes --version` | ✅ `Hermes Agent v0.22.4 (2026.9.24) · stable · local 4d04303818b6` |
| 依赖安装 | ✅ 91 个包全部安装成功，Rust 扩展编译通过 |
| `hermes status` | ✅ Gateway running，Model `claude-sonnet-5-5`，Provider `custom` |
| 端点连通性 | ✅ `/v1/models` 返回 28 个模型 |
| 直接 API 调用 | ✅ `chat/completions` 返回 `{"content":"OK"}` |
| Hermes 端到端 | ✅ `hermes -z "1+1等于几"` → `2` |
| 文件工具 | ✅ 读取 `/etc/hostname` → `em20-dk` |
| 飞书连接 | ✅ `connected`（需重试 2~3 次） |
| 飞书发送 | ✅ `{"code":0,"msg":"success"}` |
| 飞书收发 | ✅ 用户发消息可正常收到回复 |
| Telegram 连接 | ✅ `connected` |
| Telegram 收发 | ✅ 正常工作 |
| Gateway 稳定性 | ✅ 连续运行 2 小时 13 分无重启 |
| terminal 工具 | ❌ riscv64 不支持（上游限制，非回归） |
| Computer Use | ❌ riscv64 不支持（同一根因） |

---

## 附：与旧版部署的差异

旧版（main 分支）部署时踩过的坑，在 v0.22.4 上**已不存在**：

| 项目 | 旧版 | v0.22.4 |
|---|---|---|
| Cargo 版本 | 只有系统自带 1.75.0，需注释掉 3 个 Rust 包 | 装 rustup stable（**实际 1.99.1**，门槛 ≥1.88），**全部正常编译** |
| `firecrawl-anydoc` | ❌ 注释掉 | ✅ 正常安装（源码编译 26分48秒） |
| `pillow-heif` | ❌ 注释掉 | ✅ 正常安装（纯 C/C++，不依赖 Rust） |
| `resvg-py` | ❌ 注释掉 | ✅ 正常安装 |
| `cryptography` | ⚠️ 降级到 43.0.3 | ✅ 保持 50.0.1 |
| 飞书依赖 | 手动装 `lark-oapi` | 独立 `feishu` extra，需手动装 |
| 模型配置 | 手工填 config.yaml | `hermes config set` 四条命令即可 |

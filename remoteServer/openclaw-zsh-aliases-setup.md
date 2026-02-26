# OpenClaw on Termux - Zsh 环境配置

> **日期**：2026-02-26
> **环境**：Android Termux + Zsh + Oh My Zsh
> **前置文档**：[openclaw-on-termux-troubleshooting.md](./openclaw-on-termux-troubleshooting.md)

---

## 📋 背景

原安装脚本 `install-openclaw-termux.sh` 只修改了 `~/.bashrc`，添加了 `ocr`、`oclog`、`ockill` 三个别名命令。但由于我使用的是 **Zsh**，这些别名无法直接使用。

本文档记录了将 OpenClaw 管理命令适配到 Zsh 环境的过程。

---

## 🔍 原脚本的别名（写入 ~/.bashrc）

```bash
# --- OpenClaw Start ---
export TERMUX_VERSION=1
export TMPDIR=$HOME/tmp
export OPENCLAW_GATEWAY_TOKEN=naikuaiwk666
export PATH=$HOME/.npm-global/bin:$PATH

alias ocr="pkill -9 -f 'openclaw' 2>/dev/null; tmux kill-session -t openclaw 2>/dev/null; sleep 1; tmux new -d -s openclaw; sleep 1; tmux send-keys -t openclaw \"export PATH=$NPM_BIN:\$PATH TMPDIR=\$HOME/tmp; export OPENCLAW_GATEWAY_TOKEN=$TOKEN; openclaw gateway --bind lan --port $PORT --token \\\$OPENCLAW_GATEWAY_TOKEN --allow-unconfigured\" C-m"
alias oclog='tmux attach -t openclaw'
alias ockill='pkill -9 -f "openclaw" 2>/dev/null; tmux kill-session -t openclaw 2>/dev/null'
# --- OpenClaw End ---
```

---

## ✅ Zsh 适配方案

### 问题分析

| 问题 | 说明 |
|------|------|
| 脚本只修改 ~/.bashrc | Zsh 环境不会加载 Bash 配置 |
| 原命令只启动 Gateway | 没有包含 nginx（HTTPS 反向代理） |

### 改进点

1. 将别名添加到 `~/.zshrc`
2. 增强 `ocr` 命令，同时启动 Gateway 和 nginx
3. 新增 nginx 独立管理命令

---

## 📝 添加到 ~/.zshrc 的配置

```bash
# --- OpenClaw Start ---
# WARNING: This section contains your access token - keep ~/.zshrc secure
export TERMUX_VERSION=1
export TMPDIR=$HOME/tmp
export OPENCLAW_GATEWAY_TOKEN=naikuaiwk666
export PATH=$HOME/.npm-global/bin:$PATH

# 启动 OpenClaw (带 nginx)
alias ocr='pkill -9 -f openclaw 2>/dev/null; tmux kill-session -t openclaw 2>/dev/null; sleep 1; tmux new -d -s openclaw "export PATH=$HOME/.npm-global/bin:$PATH TMPDIR=$HOME/tmp && openclaw gateway --bind lan --port 18789 --token naikuaiwk666 --allow-unconfigured"; pkill nginx 2>/dev/null; sleep 1; nginx -c $HOME/nginx-ssl/nginx.conf'

# 查看 OpenClaw 日志
alias oclog='tmux attach -t openclaw'

# 停止 OpenClaw
alias ockill='pkill -9 -f openclaw 2>/dev/null; tmux kill-session -t openclaw 2>/dev/null; pkill nginx 2>/dev/null'

# 启动 nginx (单独)
alias nginx-start='nginx -c $HOME/nginx-ssl/nginx.conf'

# 停止 nginx (单独)
alias nginx-stop='pkill nginx'
# --- OpenClaw End ---
```

---

## 🚀 使用方法

### 手机重启后启动 OpenClaw

```bash
# 1. SSH 连接到手机
ssh iqoo-z7

# 2. 重新加载配置（如果需要）
source ~/.zshrc

# 3. 一键启动
ocr
```

### 日常管理命令

| 命令 | 功能 |
|------|------|
| `ocr` | 一键启动 OpenClaw Gateway + nginx |
| `oclog` | 查看/进入 OpenClaw 日志（Ctrl+B 然后 D 退出） |
| `ockill` | 停止 OpenClaw + nginx |
| `nginx-start` | 单独启动 nginx |
| `nginx-stop` | 单独停止 nginx |

---

## 🧪 测试验证

### 测试步骤

```bash
# 1. 停止所有服务
pkill -9 -f openclaw
tmux kill-session -t openclaw
pkill nginx

# 2. 验证服务已停止
ps aux | grep openclaw | grep -v grep
ps aux | grep nginx | grep -v grep

# 3. 重新加载配置
source ~/.zshrc

# 4. 验证别名可用
alias ocr

# 5. 一键启动
ocr

# 6. 验证服务状态
tmux list-sessions
ps aux | grep openclaw | grep -v grep
ps aux | grep nginx | grep -v grep

# 7. 查看日志
oclog
# 按 Ctrl+B 然后 D 退出
```

### 预期结果

```bash
$ tmux list-sessions
openclaw: 1 windows (created Thu Feb 26 XX:XX:XX 2026)

$ ps aux | grep openclaw | grep -v grep
u0_a324  XXXX  XX.X  X.X  XXXXXXXX  XXXXX ?  Ssl  1970  0:XX /data/data/com.termux/files/usr/bin/node openclaw-gateway

$ ps aux | grep nginx | grep -v grep
u0_a324  XXXX  0.0  0.0  XXXXXXXX  XXXX ?  Ss   1970  0:00 nginx
u0_a324  XXXX  0.0  0.0  XXXXXXXX  XXXX ?  S    1970  0:00 nginx: worker process
```

---

## 📋 配置文件对比

### ~/.bashrc（原脚本生成）

| 特点 | 说明 |
|------|------|
| 环境变量 | ✅ 包含 |
| Gateway 别名 | ✅ 包含（ocr, oclog, ockill） |
| nginx 管理 | ❌ 不包含 |
| 自启动服务 | ✅ sshd, termux-wake-lock |

### ~/.zshrc（手动添加）

| 特点 | 说明 |
|------|------|
| 环境变量 | ✅ 包含（从 bashrc 复制） |
| Gateway 别名 | ✅ 包含（增强版，同时管理 nginx） |
| nginx 管理 | ✅ 新增 nginx-start, nginx-stop |
| 自启动服务 | ❌ 不包含（可选添加） |

---

## 🔧 可选：添加自启动

如果希望 Termux 启动时自动运行 OpenClaw，可以在 `~/.zshrc` 中添加：

```bash
# --- OpenClaw Auto Start ---
# 在终端打开时自动启动（如果尚未运行）
if ! tmux list-sessions 2>/dev/null | grep -q openclaw; then
    if ! pgrep -x nginx > /dev/null; then
        echo "🦞 启动 OpenClaw..."
        ocr > /dev/null 2>&1
    fi
fi
# --- OpenClaw Auto Start End ---
```

---

## 📌 注意事项

1. **Token 安全**：`~/.zshrc` 包含访问 Token，请确保文件权限安全
2. **远程 SSH**：通过远程 SSH 连接时，别名不会自动加载，需要手动 `source ~/.zshrc`
3. **tmux 会话**：`oclog` 进入 tmux 会话后，按 `Ctrl+B` 然后 `D` 退出，服务继续运行

---

## 相关文档

- [OpenClaw 故障排查记录](./openclaw-on-termux-troubleshooting.md)
- [SSH 服务器访问配置](./ssh-server-access-vscode-remote-config.md)

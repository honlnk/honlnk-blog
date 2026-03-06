# OpenClaw on Termux 使用指南

> **环境**：Android Termux + Zsh + Oh My Zsh
> **设备**：iQOO Z7（通过 SSH 别名 `iqoo-z7` 访问）
> **更新日期**：2026-02-26

---

## 📋 目录

- [快速开始](#快速开始)
- [启动与停止](#启动与停止)
- [访问 Web UI](#访问-web-ui)
- [配置管理](#配置管理)
- [常用命令](#常用命令)
- [故障排查](#故障排查)
- [相关文档](#相关文档)

---

## 快速开始

### 手机重启后启动 OpenClaw

```bash
# 1. 连接到手机
ssh iqoo-z7

# 2. 一键启动（Gateway + nginx）
ocr
```

### 从电脑访问 Web UI

```bash
# 1. 建立 SSH 隧道
ssh -L 8443:127.0.0.1:8443 iqoo-z7 -N -f

# 2. 浏览器访问
# https://127.0.0.1:8443
# Token: {{ your-token }}
```

---

## 启动与停止

### 启动服务

```bash
# 方式 1：一键启动（Gateway + nginx）
ocr

# 方式 2：分别启动
tmux new -d -s openclaw 'export PATH=$HOME/.npm-global/bin:$PATH TMPDIR=$HOME/tmp && openclaw gateway --bind lan --port 18789 --token {{ your-token }} --allow-unconfigured'
nginx -c ~/nginx-ssl/nginx.conf
```

### 停止服务

```bash
# 方式 1：一键停止
ockill

# 方式 2：分别停止
tmux kill-session -t openclaw
pkill nginx
```

### 查看运行状态

```bash
# 检查 tmux 会话
tmux list-sessions

# 检查进程
ps aux | grep openclaw | grep -v grep
ps aux | grep nginx | grep -v grep

# 查看日志
oclog
# 按 Ctrl+B 然后 D 退出
```

---

## 访问 Web UI

### 方式 1：通过 SSH 隧道（推荐）

```bash
# 在电脑上执行
ssh -L 8443:127.0.0.1:8443 iqoo-z7 -N -f

# 浏览器访问
https://127.0.0.1:8443
```

### 方式 2：手机本地访问

在手机浏览器中打开：
```
http://127.0.0.1:18789
```

### 方式 3：局域网访问（不推荐，需要配对）

```
http://192.168.31.154:18789
```

**注意**：需要先通过 `openclaw devices approve <requestId>` 批准设备配对。

### 输入 Token

首次访问需要输入 Token：
```
{{ your-token }}
```

---

## 配置管理

### 当前配置概览

| 配置项   | 值                  |
| ----- | ------------------ |
| AI 模型 | `zai/glm-4.7`      |
| Token | `{{ your-token }}` |
| 端口    | `18789`            |
| 绑定地址  | `lan`              |
| 信任代理  | `127.0.0.1`, `::1` |

### 修改配置

#### 修改 AI 模型

```bash
export PATH=$HOME/.npm-global/bin:$PATH

# 切换模型
openclaw config set agents.defaults.model.primary zai/glm-4.7-flash

# 重启 Gateway 生效
ockill
ocr
```

#### 可用模型列表

| 模型 | 说明 |
|------|------|
| `zai/glm-5` | 最强模型，需要特定套餐 |
| `zai/glm-4.7` | 推荐，性能均衡 |
| `zai/glm-4.7-flash` | 快速响应，适合简单任务 |
| `zai/glm-4.7-flashx` | 更快版本 |

#### 修改 Token

```bash
export PATH=$HOME/.npm-global/bin:$PATH
openclaw config set gateway.auth.token YOUR_NEW_TOKEN

# 更新 ~/.zshrc 中的 OPENCLAW_GATEWAY_TOKEN
# 然后重启 Gateway
```

#### 修改端口

```bash
export PATH=$HOME/.npm-global/bin:$PATH
openclaw config set gateway.port 8888

# 更新启动命令中的 --port 参数
# 更新 nginx 配置中的 proxy_pass 地址
```

### 查看完整配置

```bash
export PATH=$HOME/.npm-global/bin:$PATH
cat ~/.openclaw/openclaw.json | jq .
```

---

## 常用命令

### 服务管理

| 命令 | 功能 |
|------|------|
| `ocr` | 一键启动 Gateway + nginx |
| `ockill` | 一键停止 Gateway + nginx |
| `oclog` | 查看/进入 Gateway 日志 |
| `nginx-start` | 单独启动 nginx |
| `nginx-stop` | 单独停止 nginx |

### 日志查看

```bash
# 实时日志（tmux 方式）
oclog
# 退出：Ctrl+B 然后 D

# 日志文件方式
tail -f ~/openclaw-logs/openclaw-*.log

# 最近 50 行
tail -n 50 ~/openclaw-logs/openclaw-*.log
```

### 设备配对管理

```bash
export PATH=$HOME/.npm-global/bin:$PATH

# 查看设备列表
openclaw devices list

# 批准配对请求
openclaw devices approve <requestId>

# 删除设备
openclaw devices remove <deviceId>
```

### 配置管理

```bash
export PATH=$HOME/.npm-global/bin:$PATH

# 查看配置
openclaw config

# 设置配置项
openclaw config set <key> <value>

# 获取配置项
openclaw config get <key>
```

---

## 故障排查

### 问题 1：无法访问 Web UI

**症状**：浏览器显示 "无法访问此网站"

**排查步骤**：

```bash
# 1. 检查 Gateway 是否运行
tmux list-sessions
ps aux | grep openclaw-gateway | grep -v grep

# 2. 检查 nginx 是否运行
ps aux | grep nginx | grep -v grep

# 3. 检查 SSH 隧道是否建立
ps aux | grep "ssh.*8443" | grep -v grep

# 4. 查看日志
oclog
```

**解决方案**：

```bash
# 重启服务
ockill
ocr

# 重建 SSH 隧道
pkill -f "ssh.*8443"
ssh -L 8443:127.0.0.1:8443 iqoo-z7 -N -f
```

---

### 问题 2：API Rate Limit 错误

**症状**：聊天显示 "⚠️ API rate limit reached"

**原因**：当前使用的模型不在套餐权限范围内

**排查**：

```bash
# 测试 API
curl -s https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -d '{"model": "glm-5", "messages": [{"role": "user", "content": "你好"}]}'
```

**解决方案**：切换到支持的模型

```bash
export PATH=$HOME/.npm-global/bin:$PATH
openclaw config set agents.defaults.model.primary zai/glm-4.7
ockill && ocr
```

---

### 问题 3：配对要求 (pairing required)

**症状**：Web UI 显示 "此设备需要网关主机的配对批准"

**解决方案**：

```bash
export PATH=$HOME/.npm-global/bin:$PATH

# 1. 查看待批准的请求
openclaw devices list

# 2. 批准请求
openclaw devices approve <requestId>
```

---

### 问题 4：命令找不到

**症状**：`ocr: command not found`

**原因**：Zsh 配置未加载

**解决方案**：

```bash
# 重新加载配置
source ~/.zshrc

# 或者使用完整命令
tmux new -d -s openclaw 'export PATH=$HOME/.npm-global/bin:$PATH TMPDIR=$HOME/tmp && openclaw gateway --bind lan --port 18789 --token {{ your-token }} --allow-unconfigured'
```

---

### 问题 5：nginx 启动失败

**症状**：无法访问 HTTPS 端口

**排查**：

```bash
# 测试配置
nginx -t -c ~/nginx-ssl/nginx.conf

# 查看错误日志
cat ~/nginx-ssl/error.log
```

**解决方案**：

```bash
# 停止旧进程
pkill nginx

# 重新启动
nginx -c ~/nginx-ssl/nginx.conf
```

---

### 问题 6：手机重启后服务停止

**症状**：
- 访问 `https://127.0.0.1:8443` 无响应
- tmux 会话不存在
- openclaw 进程不存在

**原因**：
手机重启后，tmux 会话和所有进程都会停止。

**排查**：

```bash
# 检查 tmux 会话
tmux list-sessions

# 检查进程
ps aux | grep -E 'openclaw|nginx' | grep -v grep
```

**解决方案**：

```bash
# 方式 1：使用别名（需要先加载配置）
ssh iqoo-z7
source ~/.zshrc
ocr

# 方式 2：手动启动
ssh iqoo-z7
tmux new -d -s openclaw 'export PATH=$HOME/.npm-global/bin:$PATH TMPDIR=$HOME/tmp && openclaw gateway --bind lan --port 18789 --token {{ your-token }} --allow-unconfigured'
nginx -c ~/nginx-ssl/nginx.conf
```

**注意**：
- 文档中的 `{{ your-token }}` 需要替换成你的真实 Token
- 如果 `ocr` 命令找不到，说明别名未加载，使用方式 2

---

### 问题 7：HTTPS 访问注意事项

**症状**：
- `http://127.0.0.1:8443` 无法访问
- 浏览器提示"400 Bad Request"或空白页面

**原因**：
nginx 配置为 HTTPS，需要用 `https://` 协议访问。

**解决方案**：

```bash
# 正确的访问方式
https://127.0.0.1:8443

# 错误的访问方式
http://127.0.0.1:8443  ❌
```

**浏览器提示**：
访问时会看到"不安全"或"证书无效"的警告，这是因为使用了自签名证书。

**处理方式**：
- 点击"高级"或"Advanced"
- 点击"继续访问"或"Proceed to 127.0.0.1 (unsafe)"

---

## 系统信息

### 设备信息

- **设备**：iQOO Z7
- **系统**：Android
- **Termux 包管理器**：pkg

### 软件版本

- **Node.js**：v24.13.0
- **OpenClaw**：2026.2.25
- **nginx**：1.29.5

### 目录结构

```
$HOME/
├── .npm-global/                    # NPM 全局包
│   └── bin/
│       └── openclaw                # OpenClaw 可执行文件
├── .openclaw/                      # OpenClaw 配置
│   ├── openclaw.json               # 主配置文件
│   ├── workspace/                  # 工作空间
│   └── agents/main/sessions/       # 会话存储
├── nginx-ssl/                      # nginx HTTPS 配置
│   ├── nginx.conf                  # nginx 配置文件
│   ├── cert.pem                    # SSL 证书
│   └── key.pem                     # SSL 私钥
├── openclaw-logs/                  # 日志目录
│   ├── install.log                 # 安装日志
│   ├── runtime.log                 # 运行时日志
│   └── openclaw-2026-02-26.log     # 当日日志
├── start-openclaw.sh               # 启动脚本（可选）
├── stop-openclaw.sh                # 停止脚本（可选）
├── .bashrc                         # Bash 配置（原脚本修改）
└── .zshrc                          # Zsh 配置（手动添加别名）
```

---

## 相关文档

| 文档 | 说明 |
|------|------|
| [故障排查记录](./openclaw-on-termux-troubleshooting.md) | 详细的问题排查过程 |
| [Zsh 别名配置](./openclaw-zsh-aliases-setup.md) | Zsh 环境别名设置 |
| [install-openclaw-on-termux](https://github.com/hillerliao/install-openclaw-on-termux) | 原项目地址 |
| [OpenClaw 官方文档](https://docs.openclaw.ai/) | 官方文档 |

---

## 快速参考

### 最常用的命令

```bash
# 启动
ssh iqoo-z7
ocr

# 访问
ssh -L 8443:127.0.0.1:8443 iqoo-z7 -N -f
# 浏览器：https://127.0.0.1:8443

# 停止
ockill

# 查看日志
oclog
# 退出：Ctrl+B 然后 D
```

### 重要信息

| 信息       | 值                           |
| -------- | --------------------------- |
| Token    | `{{ your-token }}`          |
| 端口       | `18789`                     |
| HTTPS 端口 | `8443`                      |
| 配置文件     | `~/.openclaw/openclaw.json` |
| 日志目录     | `~/openclaw-logs/`          |

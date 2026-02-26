# OpenClaw on Termux 故障排查与解决记录

> **日期**：2026-02-26
> **环境**：Android Termux + Zsh + Oh My Zsh
> **项目**：install-openclaw-on-termux

---

## 📋 目录

- [初始状态](#初始状态)
- [问题清单与解决](#问题清单与解决)
  - [问题 1：Token 不匹配](#问题-1token-不匹配)
  - [问题 2：需要配对 (pairing required)](#问题-2需要配对-pairing-required)
  - [问题 3：HTTPS 安全上下文](#问题-3https-安全上下文)
  - [问题 4：API Rate Limit](#问题-4api-rate-limit)
  - [问题 5：Gateway 绑定地址](#问题-5gateway-绑定地址)
- [最终配置](#最终配置)
- [常用命令](#常用命令)
- [参考资料](#参考资料)

---

## 初始状态

| 项目         | 状态                     |
| ---------- | ---------------------- |
| Gateway 服务 | ✅ 已启动（端口 18789）        |
| AI 模型配置    | ✅ 完成（zai/glm-5）        |
| Token      | ✅ 设置（{{ your-token }}） |
| Web UI 聊天  | ❌ 无法正常工作               |

---

## 问题清单与解决

### 问题 1：Token 不匹配

**现象**：
```
unauthorized: gateway token missing
```

**原因**：
配置的 Token 与 Gateway 实际使用的 Token 不一致。
- 用户设置的 Token: `{{ your-token }}`
- Dashboard 生成的 Token: `50909e855fec2ae4b5187bd9619dd3f99ae8fec532330b40`

**解决**：
```bash
export PATH=$HOME/.npm-global/bin:$PATH
openclaw config set gateway.auth.token {{ your-token }}
```

**注意**：由于使用 Zsh 而非 Bash，需要手动指定 PATH。

---

### 问题 2：需要配对 (pairing required)

**现象**：
```
此设备需要网关主机的配对批准。
```

浏览器控制台显示：
```
reason=pairing required
```

**原因**：
OpenClaw 默认启用了"配对"安全机制，新设备需要手动批准。

**解决**：
```bash
# 1. 查看待批准的配对请求
export PATH=$HOME/.npm-global/bin:$PATH
openclaw devices list

# 2. 批准配对请求
openclaw devices approve <requestId>
```

**日志示例**：
```
Pending (2)
┌──────────────────────────────────────┬──────────────────────────────────┐
│ Request                              │ Device                           │
├──────────────────────────────────────┼──────────────────────────────────┤
│ aceecc5e-8cc7-41cd-a681-7a6c59e733b0 │ a7fe51654c5291e990d00cc533c1a850 │
│ 8f69c300-d1bc-4b5d-bf2a-b8cbe44537c3 │ 755cccc94992d15cf96e27bf005869d4 │
└──────────────────────────────────────┴──────────────────────────────────┘
```

---

### 问题 3：HTTPS 安全上下文

**现象**：
```
control ui requires device identity (use HTTPS or localhost secure context)
```

**原因**：
OpenClaw 的 Web UI 需要安全上下文（HTTPS 或 localhost），通过局域网 IP `http://192.168.31.154:18789` 访问被拒绝。

**解决方案**：在手机上搭建 nginx + HTTPS 反向代理

#### 3.1 安装 nginx 和 openssl
```bash
pkg install nginx openssl-tool -y
```

#### 3.2 生成自签名证书
```bash
mkdir -p ~/nginx-ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ~/nginx-ssl/key.pem \
  -out ~/nginx-ssl/cert.pem \
  -subj '/C=CN/ST=State/L=City/O=Organization/CN=localhost'
```

#### 3.3 配置 nginx
创建 `~/nginx-ssl/nginx.conf`：
```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    server {
        listen 8443 ssl;
        server_name localhost;

        ssl_certificate /data/data/com.termux/files/home/nginx-ssl/cert.pem;
        ssl_certificate_key /data/data/com.termux/files/home/nginx-ssl/key.pem;

        location / {
            proxy_pass http://127.0.0.1:18789;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }
    }
}
```

#### 3.4 启动 nginx
```bash
nginx -t -c ~/nginx-ssl/nginx.conf  # 测试配置
nginx -c ~/nginx-ssl/nginx.conf      # 启动服务
```

#### 3.5 在电脑上建立 SSH 隧道
```bash
ssh -L 8443:127.0.0.1:8443 iqoo-z7 -N -f
```

#### 3.6 访问 Web UI
```
https://127.0.0.1:8443
```

**注意**：浏览器会提示证书不安全，点击"高级" → "继续访问"即可。

---

### 问题 4：API Rate Limit

**现象**：
```
⚠️ API rate limit reached. Please try again later.
```

**排查过程**：
1. 查看 Gateway 日志
2. 直接测试 GLM API

```bash
curl -s https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -d '{"model": "glm-5", "messages": [{"role": "user", "content": "你好"}]}'
```

**真实原因**：
```json
{"error":{"code":"1311","message":"当前订阅套餐暂未开放GLM-5权限"}}
```

**结论**：不是真的超额，而是套餐不支持 GLM-5 模型。

**解决**：切换到 GLM-4.7
```bash
export PATH=$HOME/.npm-global/bin:$PATH
openclaw config set agents.defaults.model.primary zai/glm-4.7
```

**验证 GLM-4.7 可用**：
```bash
curl -s https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -d '{"model": "glm-4.7", "messages": [{"role": "user", "content": "你好"}]}'
# 返回正常响应
```

---

### 问题 5：Gateway 绑定地址

**现象**：
```
Proxy headers detected from untrusted address. Connection will not be treated as local.
```

**原因**：
`gateway.bind: loopback` 只监听本地回环地址，通过 nginx 反向代理访问被视为"非本地"连接。

**解决**：
```bash
export PATH=$HOME/.npm-global/bin:$PATH
openclaw config set gateway.bind lan
openclaw config set gateway.trustedProxies '["127.0.0.1","::1"]'
```

---

## 最终配置

| 配置项         | 值                        |
| ----------- | ------------------------ |
| AI 模型       | `zai/glm-4.7`            |
| Token       | `{{ your-token }}`       |
| 端口          | `18789`                  |
| 绑定地址        | `lan`                    |
| 信任代理        | `127.0.0.1`, `::1`       |
| Web UI 访问地址 | `https://127.0.0.1:8443` |

---

## 常用命令

### 启动 Gateway
```bash
tmux new -d -s openclaw 'export PATH=$HOME/.npm-global/bin:$PATH TMPDIR=$HOME/tmp && openclaw gateway --bind lan --port 18789 --token {{ your-token }} --allow-unconfigured'
```

### 管理 Gateway
```bash
# 查看日志
tmux capture-pane -t openclaw -p

# 查看实时日志
tail -f ~/openclaw-logs/openclaw-*.log

# 重启服务
pkill -9 -f openclaw
tmux kill-session -t openclaw
# 然后重新执行启动命令
```

### 管理 nginx
```bash
# 启动
nginx -c ~/nginx-ssl/nginx.conf

# 停止
pkill nginx

# 重新加载配置
nginx -s reload
```

### 设备配对管理
```bash
export PATH=$HOME/.npm-global/bin:$PATH

# 查看设备列表
openclaw devices list

# 批准配对
openclaw devices approve <requestId>

# 删除设备
openclaw devices remove <deviceId>
```

---

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai/)
- [OpenClaw 安全文档](https://docs.openclaw.ai/gateway/security)
- [install-openclaw-on-termux 项目](https://github.com/hillerliao/install-openclaw-on-termux)
- [Z.AI GLM API 文档](https://open.bigmodel.cn/)

---

## 附录：完整的配置文件

`~/.openclaw/openclaw.json`（关键部分）：
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "zai/glm-4.7"
      }
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "{{ your-token }}"
    },
    "trustedProxies": ["127.0.0.1", "::1"]
  }
}
```

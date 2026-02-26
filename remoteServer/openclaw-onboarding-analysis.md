# OpenClaw onboarding 问题分析

> **日期**：2026-02-26
> **环境**：Android Termux + Zsh + Oh My Zsh
> **相关文档**：
> - [故障排查记录](./openclaw-on-termux-troubleshooting.md)
> - [使用指南](./openclaw-usage-guide.md)

---

## 📋 问题现象

### 用户操作流程

```
1. 执行安装脚本
   curl -sL https://s.zhihai.me/openclaw > openclaw-install.sh && bash openclaw-install.sh

2. 脚本执行完成，提示：
   ✅ tmux 会话已建立！
   请退出终端重新进入后执行: oclog 查看日志；执行 openclaw onboard 进行配置

3. 按照提示执行 onboarding
   openclaw onboard

4. onboarding 执行过程中：
   - 选择并配置了 API 密钥
   - 选择并配置了模型 (zai/glm-5)
   - 跳过了频道配置
   - 配置了 hooks
   - 最后出现错误：Error: Gateway service install not supported on android

5. 尝试访问 Web UI，发现无法使用
```

---

## 🔍 问题排查过程

### 第一阶段：确认服务状态

执行 `openclaw onboard` 后检查服务状态：

```bash
$ tmux list-sessions
no server running on /data/data/com.termux/files/usr/var/run/tmux-10324/default

$ ps aux | grep openclaw
(只显示 grep 进程本身，没有 openclaw 进程)
```

**发现**：tmux 会话和 openclaw 进程都不存在

**初步判断**：服务停止了

---

### 第二阶段：手动重启服务

```bash
# 手动启动
tmux new -d -s openclaw
tmux send-keys -t openclaw "openclaw gateway --bind lan --port 18789 --token {{ your-token }} --allow-unconfigured" C-m

# 验证启动成功
$ tmux list-sessions
openclaw: 1 windows (created Thu Feb 26 21:21:37 2026)
```

**发现**：可以手动启动，说明 OpenClaw 本身是可用的

---

### 第三阶段：Web UI 连接问题

访问 `https://127.0.0.1:8443` 后，浏览器控制台显示：

```
reason=pairing required
```

**分析**：
- HTTPS 连接正常 ✅
- Token 认证失败 ❌
- 触发了设备配对要求

---

### 第四阶段：Token 不匹配问题

通过日志发现：

```bash
# Gateway 日志显示
[ws] unauthorized reason=token_missing

# 但 openclaw dashboard 生成的 Token 是
50909e855fec2ae4b5187bd9619dd3f99ae8fec532330b40

# 而用户设置的 Token 是
{{ your-token }}
```

**发现**：配置的 Token 与实际使用的 Token 不一致

**解决**：
```bash
openclaw config set gateway.auth.token {{ your-token }}
openclaw devices approve <requestId>
```

---

### 第五阶段：API Rate Limit 问题

解决 Token 后，聊天显示：

```
⚠️ API rate limit reached. Please try again later.
```

**排查**：直接测试 GLM API

```bash
curl -s https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -d '{"model": "glm-5", "messages": [...]}'

# 返回
{"error":{"code":"1311","message":"当前订阅套餐暂未开放GLM-5权限"}}
```

**发现**：不是真的超额，而是套餐不支持 GLM-5 模型

**解决**：切换到 GLM-4.7
```bash
openclaw config set agents.defaults.model.primary zai/glm-4.7
```

---

## 🎯 问题根源分析

### 脚本执行后的实际状态

```
脚本执行完成时：
├── ✅ tmux 会话已创建
├── ✅ Gateway 进程已启动
├── ✅ Token 环境变量已设置 (~/.bashrc)
├── ✅ 别名命令已配置 (ocr, oclog, ockill)
└── ✅ Gateway 正常运行，监听 18789 端口

此时：
   • 可以访问 http://127.0.0.1:18789 ✅
   • 需要输入 Token ✅
   • 但 AI 模型可能未配置 ❌
```

### onboarding 执行过程

```
onboarding 流程：

【选择阶段 - 只收集配置】
├── 安全警告确认 ✅
├── 配置模式选择 ✅
├── 模型提供商选择 ✅
├── API 密钥输入 ✅
├── 模型选择 (glm-5) ✅
├── 频道配置 (跳过) ✅
└── hooks 配置 ✅

【写入阶段 - 配置生效】
├── API 密钥写入配置文件 ✅
├── 模型配置写入 ✅
└── hooks 配置写入 ✅

【启动阶段 - 这里失败】
├── Gateway service install
│   └── ❌ Android 不支持，报错！
│
└── 后续步骤都不执行 ❌
    ├── Token 配置 ❌
    ├── 服务启动 ❌
    └── 验证步骤 ❌
```

### onboarding 失败的影响

```
onboarding 失败导致：

配置文件状态：
├── API 密钥 ✅ (已写入)
├── 模型配置 ✅ (已写入，但用的是不支持的 glm-5)
├── hooks 配置 ✅ (已写入)
└── Token ❌ (未写入，或写入为空)

运行状态：
├── 原有 Gateway 进程可能受影响 ❌
├── tmux 会话可能被破坏 ❌
└── 配置文件状态不一致 ❌
```

---

## 💡 核心发现

### 发现 1：onboarding 是"全有或全无"的流程

```
onboarding 设计假设：
├── 所有步骤都成功 → 配置完整生效
└── 任何步骤失败 → 流程中断，后续不执行

在 Android 上：
└── Gateway service install 必然失败
    └── 导致整个 onboarding 失败
        └── 前面的选择白费了
```

### 发现 2：脚本提示具有误导性

```bash
# 脚本执行完成后的提示
✅ tmux 会话已建立！
请退出终端重新进入后执行: openclaw onboard 进行配置
                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                    这在 Android 上是错误的建议！
```

**问题**：
- Android 不支持 Gateway service install
- onboarding 必然失败
- 执行 onboarding 反而破坏了已运行的服务

### 发现 3：Token 配置的时机

```
Token 配置时机：

安装脚本：
├── 设置环境变量 ✅
├── 设置命令行参数 ✅
└── 写入配置文件 ❌ (脚本不做)

onboarding：
├── 收集 Token 输入 ✅
└── 写入配置文件 ❌ (启动阶段失败，没执行到)

结果：
├── 环境变量有 Token ✅
├── 命令行参数有 Token ✅
└── 配置文件没有/空的 Token ❌
    └── Gateway 可能生成随机 Token
        └── 导致与用户输入不匹配
```

### 发现 4：为什么需要手动输入 Token

**正常机器（onboarding 成功）**：
```
Token 写入配置文件 ✅
    ↓
Gateway 启动时读取配置文件 ✅
    ↓
Web UI 连接时自动验证 ✅
    ↓
不需要手动输入 Token ✅
```

**Android（onboarding 失败）**：
```
Token 未写入配置文件 ❌
    ↓
Gateway 启动时配置文件为空 ❌
    ↓
Gateway 生成随机 Token ❌
    ↓
Web UI 连接时 Token 不匹配 ❌
    ↓
需要手动输入 Token ❌
```

**对比**：
| 场景 | Token 来源 | 是否需要输入 |
|------|-----------|-------------|
| 正常机器 | 配置文件（onboarding 写入） | ❌ 不需要 |
| Android | 配置文件为空，生成随机 Token | ✅ 需要输入 |

---

### 发现 5：设备配对要求也是连锁反应

```
onboarding 失败
    ↓
Token 未写入配置文件
    ↓
Gateway 启动时发现配置文件 Token 为空
    ↓
Gateway 生成随机 Token
    ↓
用户访问 Web UI，输入自己设置的 Token ({{ your-token }})
    ↓
Token 不匹配：
  • 用户输入: {{ your-token }}
  • Gateway 期望: 随机生成的 Token
    ↓
Gateway 触发安全机制："这可能是未授权设备"
    ↓
显示: pairing required (需要配对)
    ↓
需要手动执行:
  openclaw devices list       # 查看待批准的请求
  openclaw devices approve <id>  # 批准配对
```

**如果 onboarding 成功**：
```
Token 正确写入配置文件 ✅
    ↓
Gateway 启动时读取配置中的 Token ✅
    ↓
用户访问 Web UI，自动验证通过 ✅
    ↓
直接可以使用，不需要配对！✅
```

---

## 📊 完整事件链

```
用户执行安装脚本
    ↓
✅ 脚本成功完成
    ├── tmux 会话创建 ✅
    ├── Gateway 启动 ✅
    └── 提示执行 onboarding ⚠️
    ↓
用户执行 onboarding (按提示)
    ↓
✅ 前面配置选择都成功
    ├── API 密钥 ✅
    ├── 模型 (glm-5) ✅
    └── hooks ✅
    ↓
❌ Gateway service install 失败
    └── Android 不支持
    ↓
❌ onboarding 流程中断
    └── Token 没配置到文件
    ↓
❌ 已有服务可能被破坏
    └── tmux 会话/进程异常
    ↓
用户尝试访问 Web UI
    ↓
❌ Token 不匹配
    • 用户输入: {{ your-token }}
    • Gateway 期望: 随机生成的 Token
    ↓
❌ 触发安全机制: pairing required
    ↓
手动批准设备配对
    openclaw devices list
    openclaw devices approve <requestId>
    ↓
解决 Token 问题后
    ↓
❌ API Rate Limit
    └── glm-5 不支持
    ↓
切换到 glm-4.7
    ↓
✅ 最终可用
```

---

## 🎓 经验教训

### 1. 理解 onboarding 的设计

```
onboarding 是为桌面系统设计的：
├── macOS: launchd ✅
├── Linux: systemd ✅
├── Windows: schtasks ✅
└── Android: ❌ 不支持

它假设系统能安装"服务"，但 Android 没有这个概念。
```

### 2. 脚本提示需要针对平台优化

```bash
# 当前提示（误导）
echo "请执行 openclaw onboard 进行配置"

# 应该改成（准确）
if [ "$(uname -o)" = "Android" ]; then
    echo "⚠️  Android 不支持 onboarding"
    echo "已自动配置基础设置，可直接使用"
else
    echo "请执行 openclaw onboard 进行配置"
fi
```

### 3. Token 配置应该在安装时完成

```
问题：脚本没有把 Token 写入配置文件
解决：在 configure_npm() 函数中添加

openclaw config set gateway.auth.token "$TOKEN"
openclaw config set gateway.bind "lan"
openclaw config set gateway.trustedProxies '["127.0.0.1","::1"]'
```

### 4. 问题时要有"事件链"思维

```
不要只看当前错误，要追溯：

当前问题: Token 不匹配
    ↑
前一个问题: onboarding 失败
    ↑
根本原因: Android 不支持 Gateway service install
    ↑
设计问题: 脚本提示用户执行 onboarding
```

---

## 🔧 改进方向

### 短期：修改脚本提示

```bash
# 脚本结束时
echo "✅ tmux 会话已建立！"
echo ""
echo "📱 Android 注意事项："
echo "   onboarding 在 Android 上不可用"
echo "   已自动配置基础设置，可直接访问："
echo "   • 地址: http://127.0.0.1:18789"
echo "   • Token: $TOKEN"
```

### 中期：添加 Android 兼容性代码

```bash
# 在 configure_npm() 后添加
configure_android_compatibility() {
    log "配置 Android 兼容性"
    export PATH="$NPM_BIN:$PATH"
    openclaw config set gateway.auth.token "$TOKEN"
    openclaw config set gateway.bind "lan"
    openclaw config set gateway.trustedProxies '["127.0.0.1","::1"]'
}
```

### 长期：向官方反馈

- 提交 Issue：说明 Android 用户的困境
- 提交 PR：添加 Android 平台支持
- 或者：创建独立的 Android 配置脚本

---

## 📝 相关文档

- [故障排查记录](./openclaw-on-termux-troubleshooting.md) - 具体问题的解决过程
- [使用指南](./openclaw-usage-guide.md) - 日常使用参考
- [Zsh 别名配置](./openclaw-zsh-aliases-setup.md) - Zsh 环境适配

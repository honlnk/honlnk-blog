---
title: mihomo TUN 残留 nftables 规则导致 Docker 容器无法出站排查
date: 2026-07-09
tags:
  - docker
  - 网络
  - 排查
  - mihomo
  - nftables
  - 服务器运维
server: volcano-honlnk
status: completed
---

# mihomo TUN 残留 nftables 规则导致 Docker 容器无法出站排查

> [!info] 故障信息
> - **时间**：2026-07-09
> - **服务器**：volcano-honlnk
> - **现象**：`https://honlnk-obsidian.honlnk.com/avatar_ai.png` 返回 502
> - **根因**：mihomo TUN 模式关闭后，`table inet mihomo` nftables 表残留，劫持了自定义 Docker 网络的出站流量
> - **耗时**：一次完整的"现象 → 误判 → 修正 → 真凶"排查链路

---

## 一句话结论

mihomo（Clash 内核）某次开启 TUN 模式后异常关闭，nftables 里的流量劫持规则（`table inet mihomo`）没有被清理，导致**自定义 Docker 网络**（非 `docker0`）的容器所有 TCP 出站流量被重定向到一个已关闭的本地端口 `:39239`，内核立即返回 RST，表现为 `Connection refused`。

**修复命令**（治标，立即生效）：

```bash fold title:删除残留的 nftables 表
# 先备份
sudo nft list table inet mihomo > /tmp/mihomo-nft-backup.conf
# 再删除
sudo nft delete table inet mihomo
```

---

## 故障现象

`honlnk-obsidian.honlnk.com` 这个域名（nginx 反代阿里云 OSS）的所有资源访问返回 **502 Bad Gateway**。

而同一台服务器上的其他域名（`daidai.honlnk.com`、`piclist.honlnk.com`）**完全正常**。

## 根本原因

这台服务器上跑着 mihomo（Clash 内核），通过 `clashctl` 管理。mihomo 的 `mixin.yaml` 里配置了 TUN 相关选项：

```yaml fold title:mixin.yaml 中的 TUN 配置
tun:
  enable: false           # 当前关闭
  auto-redirect: true     # ← 关键: 开启时会用 nftables 劫持流量
  exclude-interface:
    - docker0
    - podman0             # ← 只排除了 docker0,没排除自定义网桥
```

`auto-redirect: true` 的工作方式是：开启 TUN 时往 nftables 的 PREROUTING 链注入 `redirect to :39239` 规则，把 TCP 流量劫持到 mihomo。**问题在于：关闭 TUN 或进程异常退出时，这套规则不一定能清理干净。**

残留的 nftables 表如下：

```nft fold title:残留的 table inet mihomo
table inet mihomo {
	set inet4_local_address_set {
		type ipv4_addr
		flags interval
		elements = { 127.0.0.0/8, 172.17.0.0-172.19.255.255, 192.168.0.0/16, 198.18.0.0/30 }
	}
	chain prerouting {
		type nat hook prerouting priority dstnat + 1; policy accept;  # 比 Docker 规则更靠前!
		...
		ip daddr @inet4_local_address_set counter ... return          # 目标是本地,放行
		meta nfproto ipv4 meta l4proto tcp counter ... redirect to :39239 return
		# ↑ 兜底: 所有其他 IPv4 TCP 流量,重定向到 39239 端口
	}
}
```

### 故障链路

``` fold title:流量被劫持的完整链路
容器发起 TCP 连接 (172.19.0.3 → 外网:443)
  ↓
nftables PREROUTING (mihomo 表, priority dstnat+1, 比 Docker 靠前)
  ↓ 匹配规则: "ipv4 tcp → redirect to :39239"
  ↓ 流量被重定向到 39239 端口
  ↓
但 39239 端口没有任何进程监听 (TUN 已关闭, mihomo 不接收)
  ↓
内核立即返回 RST (Connection refused)
```

### 为什么 docker0 正常、自定义网络不正常？

这是排查中最容易误导人的点。实测：

| 网络 | 出站结果 |
|------|---------|
| `docker0`（默认 bridge, 172.17）的容器 | ✅ 正常 |
| `honlnk-gateway-net`（自定义 bridge, 172.19）的容器 | ❌ Connection refused |
| `dai-dai-net`（自定义 bridge, 172.18）的容器 | ❌ Connection refused |

计数器测试证明：docker0 容器的包**根本没命中**那条 redirect 规则，而自定义网络容器的包会 +1。这与 `inet4_local_address_set` 的匹配逻辑、nftables 的 hook 优先级差异有关。**但无论细节如何，根因是 mihomo 的 nftables 表在劫持流量。**

> [!warning] 为什么 daidai / piclist 正常，只有 honlnk-obsidian 挂了
> 这是最关键的误导点。看 nginx 配置的 `proxy_pass` 目标：
> - `daidai.honlnk.com` → `https://dai-dai-gateway:443`（**容器名，Docker 内部通信，不出公网**）→ 正常
> - `piclist.honlnk.com` → `http://piclist-app:36677`（**容器名，内部通信**）→ 正常
> - `honlnk-obsidian.honlnk.com` → `https://xxx.oss-cn-beijing.aliyuncs.com`（**外部公网 OSS**）→ 必须出公网 → 被劫持 → 502
>
> **只有 honlnk-obsidian 需要容器主动发起出公网连接**，所以只有它暴露了问题。

---

## 排查过程（完整链路）

这次排查走了不少弯路，记录下来供日后参考。核心教训：**排查网络问题时，一定要查 nftables 的完整规则集，不能只看 iptables。**

### 阶段 1：误判为 Docker 自定义网络问题

最初的诊断方向完全错了。证据链看起来很完整：

| 测试 | 结果 |
|------|------|
| 宿主机直连 OSS 源站 | ✅ 200 OK |
| nginx 反代访问域名 | ❌ 502 |
| 容器内访问外网（百度等） | ❌ Connection refused |
| 容器内 DNS 解析 | ✅ 正常 |
| **默认 docker0 网络的容器访问外网** | ✅ **正常** |
| **自定义网络容器访问外网** | ❌ **Connection refused** |

抓包铁证（一度以为锁定了根因）：
- docker0 网络容器：SYN 经过 MASQUERADE 后出现在 eth0 上，收到响应 ✅
- 自定义网络容器：SYN 被 iptables FORWARD 链 ACCEPT，但**从没出现在 eth0 上**，84 微秒后内核生成 RST ❌

当时排除了所有常规嫌疑（ip_forward、rp_filter、MASQUERADE、MTU、重启 docker），得出结论："火山引擎云组件在更底层拦截了自定义网络的出站流量"。

**这个结论是错的**——因为漏查了 nftables。

> [!important] 关键教训
> Ubuntu 24.04 默认用 **nftables**（`iptables` 实际是 `iptables-nft` 后端）。查 iptables 看到的只是 Docker 注入的规则，**第三方组件（如 mihomo）可能在独立的 nftables 表里注入规则，这些规则在 `iptables -L` 里完全看不到**。
>
> 排查网络问题时，必须执行 `sudo nft list ruleset` 查看完整规则集，而不是只看 `iptables`。

### 阶段 2：错误地怀疑了火山引擎安全组件

中间一度怀疑是火山引擎的 HIDS（Elkeid）或云监控 agent 在拦截。检查后发现：

- **Elkeid**（`/etc/elkeid/elkeid-agent`）：心跳日志显示 `plugins_brief_info:[]`、`plugin_summary:{}`，**没有加载任何防护插件**，没有 eBPF 程序，没有网络拦截行为 → 洗清嫌疑
- **cloud-monitor-agent**：只采集指标，不拦网络 → 排除

### 阶段 3：找到真凶 —— `table inet mihomo`

最终通过 `sudo nft list ruleset` 发现除了 Docker 的表之外，还有一个独立的 `table inet mihomo`：

```bash fold title:发现真凶的命令
sudo nft list ruleset | grep "^table"
# 输出:
# table ip nat
# table ip filter
# table ip6 nat
# table ip6 filter
# table ip raw
# table inet mihomo    ← 这个! Docker 不会创建这个表
```

这个表的 PREROUTING 链在劫持所有 IPv4 TCP 流量，redirect 到 `:39239`（mihomo TUN 模式的流量入口）。而 39239 端口当前没有监听（TUN 已关闭），导致被劫持的流量收到 RST。

通过计数器精确定位：
- 清零后让自定义网络容器发起一次出站连接 → redirect 规则计数 +1
- docker0 容器发起连接 → 计数不变

确认就是这个表在劫持。删除后立刻恢复。

### 排查中走过的弯路

尝试过的无效修复：

| 尝试 | 为什么无效 |
|------|-----------|
| `docker restart honlnk-gateway` | 问题不在容器，在宿主机 nftables |
| `systemctl restart docker` | Docker 重建的是 iptables 规则，mihomo 的 nft 表不受影响 |
| 迁移容器到 docker0 网络 | 虽然能绕过（已验证），但治标不治本 |

排查排错教训：

| 教训 | 说明 |
|------|------|
| **排查网络问题先 `nft list ruleset`** | Ubuntu 22.04+ / 内核 5.x+ 默认用 nftables，`iptables -L` 只能看到 nat/filter/raw 等传统表，看不到独立的 `inet` 表。mihomo 的劫持规则就藏在这种独立表里 |
| **用户的直觉很重要** | "会不会是 clash" 这一句提醒直接把排查从死胡同（云组件方向）拽了出来。排查到瓶颈时应该主动问"这台机器上还跑了什么不寻常的东西"，而不是闷头猜 |
| **不要过早排除嫌疑** | 查 mihomo 时因为 iptables 搜不到规则就排除了它，实际上查错了地方。排除嫌疑前要确认查的是完整的规则集 |

---

## 修复操作

### 第一步：清理残留 nftables 表（治标，立即生效）

```bash fold title:清理残留规则
# 备份
sudo nft list table inet mihomo > /tmp/mihomo-nft-backup-$(date +%Y%m%d).conf

# 删除
sudo nft delete table inet mihomo

# 验证表已不存在(应报错 "No such file or directory")
sudo nft list table inet mihomo
```

删除后 `honlnk-obsidian.honlnk.com` 立即恢复 200 OK。

### 第二步：升级 mihomo（顺带改进）

将 mihomo 从 v1.19.17 升级到 v1.19.28。

```bash fold title:升级 mihomo
# 下载新版 (走 mihomo 自身代理)
cd /tmp
https_proxy=http://127.0.0.1:7890 curl -L -o mihomo-new.gz \
  "https://github.com/MetaCubeX/mihomo/releases/download/v1.19.28/mihomo-linux-amd64-v3-v1.19.28.gz"
gunzip mihomo-new.gz && mv mihomo-new mihomo-new-bin && chmod +x mihomo-new-bin

# 验证新版本
/tmp/mihomo-new-bin -v

# 备份旧版本
cp /home/honlnk/clashctl/bin/mihomo /home/honlnk/clashctl/bin/mihomo.bak-v1.19.17

# 停止 mihomo (注意: 会短暂断网, ssh 可能掉线, 重连即可)
pkill -f "/home/honlnk/clashctl/bin/mihomo"

# 替换二进制
cp /tmp/mihomo-new-bin /home/honlnk/clashctl/bin/mihomo
chmod +x /home/honlnk/clashctl/bin/mihomo

# 启动新版
setsid nohup /home/honlnk/clashctl/bin/mihomo \
  -d /home/honlnk/clashctl/resources \
  -f /home/honlnk/clashctl/resources/runtime.yaml \
  > /home/honlnk/clashctl/resources/mihomo.log 2>&1 < /dev/null &

# 验证
curl -s --noproxy "*" -H "Authorization: Bearer honlnk" "http://127.0.0.1:2001/version"
# 应返回 {"meta":true,"version":"v1.19.28"}
```

> [!warning] 升级没有专门修复这个 bug
> 经查阅 v1.19.18 ~ v1.19.28 全部 changelog，**没有一条专门修复"nftables 规则残留"**。从源码看，mihomo（上游 sing-tun 库）的 `cleanupNFTables()` 清理逻辑一直是正确的——**只要走优雅关闭流程就会删表**。
>
> 残留发生在"没走优雅关闭"的场景（`pkill -9` 强杀、崩溃）。`clashctl.sh` 里用的正是 `pkill -9 -f mihomo`，这才是根因。**升级是好习惯，但不能防止复发。**

---

## 如何判断是否复发

如果日后 `honlnk-obsidian.honlnk.com`（或任何需要容器出公网的服务）又 502，先查这个：

```bash fold title:快速诊断
# 检查 mihomo nft 表是否存在
sudo nft list table inet mihomo 2>&1

# 如果存在且看到 "redirect to :39239", 就是复发了
# 检查 39239 是否在监听 (TUN 关闭时不会监听)
sudo ss -tlnp | grep 39239
```

如果确认复发，执行治标命令删除即可：

```bash
sudo nft delete table inet mihomo
```

---

## 关于根治（systemd 化）

真正的根治是把 mihomo 的进程管理从 `nohup` + `pkill -9` 改成 systemd，这样：
- `systemctl stop` 会发正常终止信号，mihomo 走优雅关闭，自己清理 nftables
- 进程崩溃 systemd 自动重启
- 加 `ExecStopPost=/sbin/nft delete table inet mihomo` 兜底清理

但这需要同步改 `clashctl.sh` 里的启动/停止逻辑（`clashon`/`clashoff`/`_tunon`/`_tunoff` 等函数都要从 `pkill`/`nohup` 改成 `systemctl`），改动较大。**目前暂未实施**，因为只要不主动开 TUN 就不会复发。

> [!note] 日常使用不受影响
> 这个问题**只有开启 TUN 模式（`clashtun on`）后异常关闭才会触发**。平时只用普通代理（`clashon`）不会产生 nftables 劫持规则，不受影响。

---

## 排查用到的关键命令

```bash fold title:排查命令集
# 1. 查看 nftables 完整规则集 (最重要!排查网络问题必查)
sudo nft list ruleset

# 2. 查看 iptables (legacy 视图,可能看不到独立 nft 表)
sudo iptables -t nat -L -n -v
sudo iptables -t mangle -L -n -v
sudo iptables -t raw -L -n -v

# 3. 抓包定位 (看包到底走到哪了)
sudo tcpdump -i any -nn "host <目标IP> and port 443"

# 4. 测试容器出站 (对比 docker0 vs 自定义网络)
docker run --rm --network bridge alpine sh -c "wget -q -S --spider https://www.baidu.com 2>&1 | head -2"
docker run --rm --network honlnk-gateway-net alpine sh -c "wget -q -S --spider https://www.baidu.com 2>&1 | head -2"

# 5. nft 计数器追踪 (清零后发起连接,看哪条规则命中)
sudo nft reset table inet mihomo
# ... 发起连接 ...
sudo nft list table inet mihomo
```

---

## 相关资料

### 直接相关（根因与机制）

- [mihomo #2491 讨论：auto-redirect 对 nftables 的修改及本地地址劫持](https://github.com/MetaCubeX/mihomo/discussions/2491) —— 官方讨论 `auto-redirect` 如何修改 nftables 以及本地地址（含 Docker 网段）被劫持的问题，**与本次现象高度吻合**
- [sing-tun 源码：redirect_nftables.go 的 cleanupNFTables() 清理逻辑](https://github.com/sagernet/sing-tun/blob/main/redirect_nftables.go) —— mihomo TUN 功能依赖的上游库，正常关闭时会 `DelTable` 清理，强杀时不清理
- [mihomo v1.19.28 Release](https://github.com/MetaCubeX/mihomo/releases/latest) —— 升级目标版本；v1.19.21 有一个相关的 `fix: tun doesn't clean up the DNS setting when closed`，但无 nftables 清理专项修复

### 排查参考（误入的弯路方向，根因均不同）

> [!warning] 这些不是"同类问题"
> 排查过程中搜索到大量"自定义 Docker 网络容器无法出站"的案例，但**它们的根因和本次完全不同**。那些案例的问题基本都在 iptables 规则层面（`ip_forward`、MASQUERADE、FORWARD policy），用 `iptables -L` 就能看到原因，标准修复（开转发、加 NAT、重启 docker）即可解决。
>
> 本次的问题根因是 **mihomo 在独立的 nftables 表（`table inet mihomo`）里劫持流量**，这种表 `iptables` 命令根本看不到，标准修复全部无效。**社区里几乎没有这种"代理软件 nftables 残留 + 自定义 Docker 网络"精确组合的案例。**
>
> 列出这些案例是因为排查早期曾顺着它们的方向查过（均排查排除），可作日后常规 Docker 网络问题的参考：

- [火山引擎官方：Docker 容器在自定义 Bridge 网络下访问私有网络 IP 的配置问题](https://www.volcengine.com/article/1259169) —— 火山引擎 ECS 上自定义 Bridge 网络需要额外配置路由和 NAT，与 docker0 行为不同的官方说明
- [moby/moby #36151：Containers cannot access internet (outbound)](https://github.com/moby/moby/issues/36151) —— "自定义网络容器无法出站，默认 bridge 正常"的经典 issue
- [moby/moby #27817：FORWARD policy DROP 与自定义 bridge 规则丢失](https://github.com/moby/moby/issues/27817) —— 宿主机 FORWARD 策略为 DROP 时，其他程序重载 iptables 会导致自定义 bridge 规则丢失而 docker0 幸存
- [Docker Forums：Ubuntu 24.04 容器无法访问外网](https://forums.docker.com/t/docker-containers-on-ubuntu-24-04-cannot-reach-external-network/148438) —— Ubuntu 24.04（nftables 默认）下容器出站失败的讨论，涉及 nftables 优先级问题
- [Superuser：Docker containers cannot reach internet in bridge network mode](https://superuser.com/questions/1745422/docker-containers-cannot-reach-internet-in-bridge-network-mode) —— bridge 模式容器无法上网的通用排查
- [ServerFault：Docker containers have no internet access via bridge network](https://serverfault.com/questions/1178515/docker-containers-have-no-internet-access-via-bridge-network-on-ubuntu-22-04-vps) —— SNAT/源 IP 不匹配导致 FORWARD=ACCEPT 但包仍被丢弃的分析

### 背景知识

- [Docker 官方文档：Bridge network driver](https://docs.docker.com/engine/network/drivers/bridge/) —— 默认 bridge 与用户自定义 bridge 的区别
- [nixOS #477636：clash-verge TUN is blocked by nftables](https://github.com/NixOS/nixpkgs/issues/477636) —— clash TUN 与 nftables 交互导致流量被 drop 的案例
- 相关笔记：[[honlnk-gateway-deploy-guide]]、[[clash-vpn-setup-guide]]
```

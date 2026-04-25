---
title: honlnk-gateway 全局网关与服务部署指南
date: 2026-04-24
tags:
  - deploy
  - docker
  - nginx
  - ssl
  - 服务器运维
server: volcano-honlnk
status: completed
---

# honlnk-gateway 全局网关与服务部署指南

> [!info] 部署信息
> - **时间**：2026-04-24
> - **服务器**：volcano-honlnk（`<YOUR_SERVER_IP>`）
> - **操作系统**：Ubuntu 24.04 LTS

---

## 背景

在使用 Obsidian 构建个人知识库时，需要一个稳定、私有的图片管理方案。之前在旧服务器上部署过 PicList + 阿里云 OSS 的图床服务，现在需要迁移到新服务器 `volcano-honlnk` 上。

服务器上已有 `dai-dai` 项目占用 80/443 端口，需要解耦部署。

---

## 架构设计

### 核心思路：独立全局网关

不修改 dai-dai 项目代码，新建一个独立的全局网关容器（`honlnk-gateway`），统一接管 80/443 端口，按域名分发到不同服务。

### 架构图

``` fold title:架构图
互联网
  │
  ▼
宿主机:80/443
  │
  ▼
honlnk-gateway（全局网关，Nginx 容器）
  │
  ├─ piclist.honlnk.top → piclist-app:36677（PicList 图床）
  │
  ├─ honlnk-obsidian.honlnk.top → 阿里云 OSS 反代（图片访问，自带 HTTPS）
  │
  └─ daidai.honlnk.top / api.daidai.honlnk.top / mqtt.daidai.honlnk.top
     → dai-dai-gateway:443（HTTPS→HTTPS 转发，避免无限重定向）
```

### Docker 网络规划

``` bash fold title:网关文件结构
honlnk-gateway-net（新建，全局网关网络）
    ├── honlnk-gateway
    ├── dai-dai-gateway
    └── piclist-app

dai-dai-net（已有，dai-dai 内部网络，保持不变）
    ├── dai-dai-gateway
    ├── dai-dai-backend
    ├── dai-dai-admin-web
    └── dai-dai-mqtt
```

dai-dai-gateway 同时加入两个网络，既能跟内部服务通信，又能被全局网关访问。

### 容器端口映射

| 容器 | 宿主机端口 | 容器端口 | 说明 |
|---|---|---|---|
| honlnk-gateway | 80, 443 | 80, 443 | 全局网关，唯一对外入口 |
| dai-dai-gateway | 8080, 8443 | 80, 443 | 退到内网，不再直接对外 |
| piclist-app | 无映射 | 36677 | 纯内网，通过网关访问 |

### 服务器目录结构

``` bash fold title:服务器目录结构
/home/honlnk/
├── piclist/
│   ├── config.json                ← PicList 配置
│   └── ssl/                       ← piclist.honlnk.top 证书
│       ├── fullchain.pem
│       └── privkey.pem
├── oss/
│   └── ssl/                       ← honlnk-obsidian.honlnk.top 证书
│       ├── fullchain.pem
│       └── privkey.pem
├── honlnk-gateway/
│   ├── nginx.conf                 ← 全局网关 Nginx 主配置
│   ├── conf.d/                    ← 站点配置
│   │   ├── default.conf
│   │   ├── piclist.honlnk.top.conf
│   │   ├── daidai.proxy.conf
│   │   └── oss.honlnk.top.conf
│   ├── renew-hook.sh              ← 证书续期脚本
│   └── renew.log                  ← 续期日志
├── dai-dai-compose/               ← dai-dai 部署配置
├── dai-dai-infra/
│   ├── certs/
│   │   ├── web/                   ← dai-dai web 证书（certbot 自动管理）
│   │   └── mqtt/                  ← mqtt 证书（certbot 自动管理）
│   └── env/                       ← 后端环境变量
├── mqtt/                          ← MQTT 数据和日志
├── mysql/                         ← MySQL 数据
├── redis/                         ← Redis 数据
└── install-script/                ← 安装脚本
```

---

## 部署步骤

### 第一步：创建全局网关网络

```bash fold title:创建全局网关网络
docker network create honlnk-gateway-net
```

### 第二步：修改 dai-dai-gateway 端口映射

修改 `/home/honlnk/dai-dai-compose/.env`：

```bash fold title:.env 端口配置
GATEWAY_HTTP_PORT=8080    # 原值 80
GATEWAY_HTTPS_PORT=8443   # 原值 443
```

重启 gateway：

```bash fold title:重启 gateway
cd /home/honlnk/dai-dai-compose
docker compose -f docker-compose.prod.yml up -d gateway
```

将 gateway 加入新网络：

```bash fold title:加入新网络
docker network connect honlnk-gateway-net dai-dai-gateway
```

### 第三步：配置 Docker 代理（拉取 PicList 镜像用）

服务器上安装了 Clash（mihomo），代理端口 7890（HTTP）/ 7891（SOCKS5）。

清除失效的国内镜像源：

```bash fold title:清除镜像源
# /etc/docker/daemon.json 改为 {}
echo '{}' | sudo tee /etc/docker/daemon.json
```

配置 Docker daemon 走 Clash 代理：

> [!note] Docker 网络说明
> Docker 的网络走自己的网桥，不经过 Clash 的 TUN 模式，必须显式配置代理。

```bash fold title:配置 Docker 代理
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,172.17.0.0/16,172.18.0.0/16,172.19.0.0/16"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```


### 第四步：部署 PicList 容器

创建配置目录和文件：

```bash fold title:创建配置目录
mkdir -p /home/honlnk/piclist
```

`/home/honlnk/piclist/config.json`：

```json fold title:PicList 配置文件
{
  "picBed": {
    "current": "aliyun",
    "aliyun": {
      "accessKeyId": "<YOUR_ACCESS_KEY_ID>",
      "accessKeySecret": "<YOUR_ACCESS_KEY_SECRET>",
      "bucket": "<YOUR_OSS_BUCKET_NAME>",
      "area": "oss-cn-beijing",
      "path": "images/",
      "customUrl": "https://honlnk-obsidian.honlnk.top"
    }
  },
  "settings": {
    "server": {
      "port": 36677,
      "host": "0.0.0.0",
      "key": "<YOUR_API_KEY>"
    }
  },
  "buildIn": {
    "rename": {
      "format": "{Y}{m}{d}_{h}{i}{s}_{md5-16}",
      "enable": true
    },
    "watermark": {
      "isAddWatermark": true,
      "watermarkType": "text",
      "watermarkText": "www.honlnk.top",
      "watermarkColor": "rgba(128, 128, 128, 0.5)",
      "watermarkScaleRatio": 0.1,
      "watermarkPosition": "southeast",
      "watermarkDegree": 0
    }
  },
  "picgoPlugins": {}
}
```

启动容器：

```bash fold title:启动 PicList 容器
docker run -d \
  --name piclist-app \
  --restart unless-stopped \
  -v /home/honlnk/piclist/config.json:/root/.piclist/config.json \
  --network honlnk-gateway-net \
  kuingsmile/piclist:latest \
  node /usr/local/bin/picgo-server
```

### 第五步：创建 honlnk-gateway 全局网关

目录结构：

``` fold title:网关目录结构
/home/honlnk/honlnk-gateway/
├── nginx.conf
└── conf.d/
    ├── default.conf              # 兜底，返回 404
    ├── piclist.honlnk.top.conf   # PicList HTTPS
    ├── daidai.proxy.conf         # dai-dai 项目 HTTPS 代理
    └── oss.honlnk.top.conf       # OSS 反代 + HTTPS
```

`nginx.conf`：

```nginx fold title:nginx.conf
worker_processes auto;

events {
  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  sendfile on;
  keepalive_timeout 65;

  map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
  }

  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$host"';

  access_log /var/log/nginx/access.log main;
  error_log /var/log/nginx/error.log warn;

  include /etc/nginx/conf.d/*.conf;
}
```

`conf.d/piclist.honlnk.top.conf`：

```nginx fold title:piclist.honlnk.top.conf
server {
    listen 80;
    server_name piclist.honlnk.top;
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name piclist.honlnk.top;

    ssl_certificate /etc/nginx/ssl-piclist/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl-piclist/privkey.pem;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    client_max_body_size 20m;

    location / {
        proxy_pass http://piclist-app:36677;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
    }
}
```

`conf.d/daidai.proxy.conf`：

```nginx fold title:daidai.proxy.conf
# HTTP 跳转 HTTPS
server {
    listen 80;
    server_name daidai.honlnk.top api.daidai.honlnk.top mqtt.daidai.honlnk.top;
    location / {
        return 301 https://$host$request_uri;
    }
}

# 每个域名独立 server block，使用各自证书
# 关键：proxy_pass 必须用 HTTPS，否则 dai-dai-gateway 内部会触发 HTTP→HTTPS 跳转，导致无限重定向

# daidai.honlnk.top
server {
    listen 443 ssl http2;
    server_name daidai.honlnk.top;

    ssl_certificate /etc/nginx/ssl-daidai/daidai.honlnk.top/daidai.honlnk.top_bundle.pem;
    ssl_certificate_key /etc/nginx/ssl-daidai/daidai.honlnk.top/daidai.honlnk.top.key;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass https://dai-dai-gateway:443;  # 必须用 HTTPS
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
        proxy_buffering off;
    }
}

# api.daidai.honlnk.top（同理，证书路径不同）
# mqtt.daidai.honlnk.top（同理）
```

`conf.d/oss.honlnk.top.conf`：

```nginx fold title:oss.honlnk.top.conf
server {
    listen 80;
    server_name honlnk-obsidian.honlnk.top;
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name honlnk-obsidian.honlnk.top;

    ssl_certificate /etc/nginx/ssl-oss/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl-oss/privkey.pem;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass https://<YOUR_OSS_BUCKET_NAME>.oss-cn-beijing.aliyuncs.com;
        proxy_set_header Host honlnk-obsidian.honlnk.top;  # 必须用自定义域名，避免 OSS 强制下载
        proxy_set_header X-Real-IP $remote_addr;
        proxy_hide_header Content-Disposition;
        proxy_ssl_server_name on;
    }
}
```

启动网关容器：

```bash fold title:启动网关容器
docker run -d \
  --name honlnk-gateway \
  --restart unless-stopped \
  -p 80:80 -p 443:443 \
  -v /home/honlnk/honlnk-gateway/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /home/honlnk/honlnk-gateway/conf.d:/etc/nginx/conf.d:ro \
  -v /home/honlnk/dai-dai-infra/certs/web:/etc/nginx/ssl-daidai:ro \
  -v /home/honlnk/piclist/ssl:/etc/nginx/ssl-piclist:ro \
  -v /home/honlnk/oss/ssl:/etc/nginx/ssl-oss:ro \
  --network honlnk-gateway-net \
  nginx:stable-alpine
```

> [!warning] 首次启动会失败
> 此时 SSL 证书尚未签发，Nginx 因找不到证书文件会启动失败，**这是正常的**。完成第六步签发证书并拷贝到对应目录后，网关即可正常启动。

### 第六步：SSL 证书签发

五个域名均使用 certbot standalone 模式签发，需要临时停掉 honlnk-gateway 释放 80 端口。

```bash fold title:SSL 证书签发与拷贝
docker stop honlnk-gateway

# 签发所有域名证书
sudo certbot certonly --standalone -d piclist.honlnk.top
sudo certbot certonly --standalone -d honlnk-obsidian.honlnk.top
sudo certbot certonly --standalone -d daidai.honlnk.top
sudo certbot certonly --standalone -d api.daidai.honlnk.top
sudo certbot certonly --standalone -d mqtt.daidai.honlnk.top

# 拷贝证书到各服务目录
# piclist
mkdir -p /home/honlnk/piclist/ssl
sudo cp /etc/letsencrypt/archive/piclist.honlnk.top/fullchain1.pem /home/honlnk/piclist/ssl/fullchain.pem
sudo cp /etc/letsencrypt/archive/piclist.honlnk.top/privkey1.pem /home/honlnk/piclist/ssl/privkey.pem

# oss
mkdir -p /home/honlnk/oss/ssl
sudo cp /etc/letsencrypt/archive/honlnk-obsidian.honlnk.top/fullchain1.pem /home/honlnk/oss/ssl/fullchain.pem
sudo cp /etc/letsencrypt/archive/honlnk-obsidian.honlnk.top/privkey1.pem /home/honlnk/oss/ssl/privkey.pem

# dai-dai web（覆盖原有腾讯云证书，保持文件名一致）
sudo cp /etc/letsencrypt/archive/daidai.honlnk.top/fullchain1.pem /home/honlnk/dai-dai-infra/certs/web/daidai.honlnk.top/daidai.honlnk.top_bundle.pem
sudo cp /etc/letsencrypt/archive/daidai.honlnk.top/privkey1.pem /home/honlnk/dai-dai-infra/certs/web/daidai.honlnk.top/daidai.honlnk.top.key
sudo cp /etc/letsencrypt/archive/api.daidai.honlnk.top/fullchain1.pem /home/honlnk/dai-dai-infra/certs/web/api.daidai.honlnk.top/api.daidai.honlnk.top_bundle.pem
sudo cp /etc/letsencrypt/archive/api.daidai.honlnk.top/privkey1.pem /home/honlnk/dai-dai-infra/certs/web/api.daidai.honlnk.top/api.daidai.honlnk.top.key

# mqtt（覆盖原有腾讯云证书，保持文件名一致）
sudo cp /etc/letsencrypt/archive/mqtt.daidai.honlnk.top/fullchain1.pem /home/honlnk/dai-dai-infra/certs/mqtt/mqtt.daidai.honlnk.top_bundle.pem
sudo cp /etc/letsencrypt/archive/mqtt.daidai.honlnk.top/privkey1.pem /home/honlnk/dai-dai-infra/certs/mqtt/mqtt.daidai.honlnk.top.key

# 修改权限
sudo chown -R honlnk:honlnk /home/honlnk/piclist/ssl /home/honlnk/oss/ssl /home/honlnk/dai-dai-infra/certs/

docker start honlnk-gateway
```

> [!info] DNS 验证说明
> `honlnk-obsidian.honlnk.top` 最初 DNS 指向阿里云 OSS，standalone 无法验证，临时使用了 manual + DNS 验证。后将 DNS 改到服务器 IP，重新用 standalone 签发。

### 第七步：DNS 配置

| 域名 | 类型 | 值 |
|---|---|---|
| piclist.honlnk.top | A | <YOUR_SERVER_IP> |
| honlnk-obsidian.honlnk.top | A | <YOUR_SERVER_IP> |

### 第八步：更新 dai-dai 项目 compose 配置

修改 `deploy/compose/docker-compose.prod.yml`，让 gateway 自动加入全局网关网络：

```yaml fold title:docker-compose.prod.yml
services:
  gateway:
    # ... 其他配置不变 ...
    networks:
      - dai-dai-net
      - honlnk-gateway-net  # 新增

networks:
  dai-dai-net:
    external: true
    name: dai-dai-net
  honlnk-gateway-net:       # 新增
    external: true
    name: honlnk-gateway-net
```

同步到服务器：

```bash fold title:同步到服务器
scp deploy/compose/docker-compose.prod.yml volcano-honlnk:/home/honlnk/dai-dai-compose/
```

---

## 踩坑记录

### 1. 无限重定向（ERR_TOO_MANY_REDIRECTS）

**现象**：浏览器访问 `https://daidai.honlnk.top` 报重定向次数过多。

**原因**：honlnk-gateway 用 HTTPS 接收请求，但 `proxy_pass http://dai-dai-gateway:80` 用 HTTP 转发。dai-dai-gateway 内部 Nginx 看到 HTTP 请求，触发 301 跳转到 HTTPS，形成循环。

**解决**：`proxy_pass` 改为 `https://dai-dai-gateway:443`，honlnk-gateway 到 dai-dai-gateway 之间也走 HTTPS。

### 2. OSS 图片强制下载

**现象**：浏览器访问图片 URL 触发下载而非预览。

**原因**：Nginx 反代时 `proxy_set_header Host` 设为 OSS 默认域名，触发阿里云安全策略，返回 `x-oss-force-download: true`。

**解决**：`proxy_set_header Host` 改为自定义域名 `honlnk-obsidian.honlnk.top`，因为该域名已在 OSS 控制台绑定，OSS 可以通过自定义域名识别 bucket。

### 3. dai-dai 重新部署后 502

**现象**：dai-dai 重新部署后所有接口 502。

**原因**：容器重建后，手动加入的 `honlnk-gateway-net` 网络丢失，honlnk-gateway 无法连接 dai-dai-gateway。

**解决**：在 docker-compose.prod.yml 中声明 `honlnk-gateway-net` 为外部网络，部署时自动加入。

### 4. Let's Encrypt 证书软链接问题

**现象**：`/etc/letsencrypt/live/` 下的证书文件是软链接，拷贝到其他目录后链接失效。

**解决**：拷贝 `/etc/letsencrypt/archive/` 目录（包含实际文件），挂载时使用固定文件名。

### 5. Docker 拉镜像失败

**现象**：配置的国内镜像源（daocloud、nju）返回 403。

**解决**：清空 daemon.json 中的镜像源，配置 Docker daemon 走 Clash 代理（`HTTP_PROXY=http://127.0.0.1:7890`）。

---

## 证书自动续期

### 管理的域名

| 域名 | 证书存放位置 | 证书文件名 |
|---|---|---|
| `piclist.honlnk.top` | `/home/honlnk/piclist/ssl/` | `fullchain.pem` / `privkey.pem` |
| `honlnk-obsidian.honlnk.top` | `/home/honlnk/oss/ssl/` | `fullchain.pem` / `privkey.pem` |
| `daidai.honlnk.top` | `/home/honlnk/dai-dai-infra/certs/web/daidai.honlnk.top/` | `daidai.honlnk.top_bundle.pem` / `daidai.honlnk.top.key` |
| `api.daidai.honlnk.top` | `/home/honlnk/dai-dai-infra/certs/web/api.daidai.honlnk.top/` | `api.daidai.honlnk.top_bundle.pem` / `api.daidai.honlnk.top.key` |
| `mqtt.daidai.honlnk.top` | `/home/honlnk/dai-dai-infra/certs/mqtt/` | `mqtt.daidai.honlnk.top_bundle.pem` / `mqtt.daidai.honlnk.top.key` |



### 续期方式

五个域名均使用 certbot standalone 模式签发，通过 root crontab 实现自动续期：

``` fold title:crontab 定时任务
0 3 * * * docker stop honlnk-gateway && certbot renew --quiet && /home/honlnk/honlnk-gateway/renew-hook.sh
```

### 执行流程

1. 凌晨 3 点停掉 honlnk-gateway（释放 80 端口）
2. certbot 检查证书是否快过期（< 30 天），是则续期
3. renew-hook.sh 拷贝最新证书到各服务目录（固定文件名）
4. 启动 honlnk-gateway 并重载 Nginx

### 中断时间

- 证书未过期时：certbot 跳过续期，仅中断 **2-3 秒**
- 证书需要续期时：中断约 **5-10 秒**

### renew-hook.sh 脚本详解

脚本位于 `/home/honlnk/honlnk-gateway/renew-hook.sh`，分四个部分处理不同服务的证书：

```bash fold title:renew-hook.sh 脚本
#!/usr/bin/env bash
LOG_FILE="/home/honlnk/honlnk-gateway/renew.log"

# 第1部分：piclist.honlnk.top
# certbot archive 递增编号 → 拷贝为固定文件名 fullchain.pem / privkey.pem
PICLIST_SRC="/etc/letsencrypt/archive/piclist.honlnk.top"
PICLIST_DST="/home/honlnk/piclist/ssl"
# ls -t 按时间排序取最新，cp -f 覆盖为固定文件名

# 第2部分：honlnk-obsidian.honlnk.top
# 同理，拷贝到 /home/honlnk/oss/ssl/
OSS_SRC="/etc/letsencrypt/archive/honlnk-obsidian.honlnk.top"
OSS_DST="/home/honlnk/oss/ssl"

# 第3部分：mqtt.daidai.honlnk.top
# mosquitto 不支持证书热重载，续期后需要重启容器
MQTT_SRC="/etc/letsencrypt/archive/mqtt.daidai.honlnk.top"
MQTT_DST="/home/honlnk/dai-dai-infra/certs/mqtt"
# 拷贝为 mqtt.daidai.honlnk.top_bundle.pem / mqtt.daidai.honlnk.top.key
docker restart dai-dai-mqtt  # 必须重启

# 第4部分：dai-dai web 系列（daidai.honlnk.top、api.daidai.honlnk.top）
# 证书存放在 dai-dai-infra 原有位置，文件名保持与腾讯云时代一致
# 这样 Nginx 配置和 docker-compose 挂载都不用改
for domain in "${DAIDAI_DOMAINS[@]}"; do
    cp -f "$LATEST_FULLCHAIN" "/home/honlnk/dai-dai-infra/certs/web/$domain/${domain}_bundle.pem"
    cp -f "$LATEST_PRIVKEY" "/home/honlnk/dai-dai-infra/certs/web/$domain/${domain}.key"
done

# 最后：启动网关 + 重载 Nginx
docker start honlnk-gateway
sleep 2
docker exec honlnk-gateway nginx -s reload
```

**核心逻辑**：certbot 每次续期生成递增编号的文件（`fullchain1.pem` → `fullchain2.pem` → ...），脚本通过 `ls -t | head -1` 取最新文件，拷贝为固定文件名供 Nginx 使用。

> [!note] MQTT 证书特殊性
> `mqtt.daidai.honlnk.top` 走 mosquitto 容器自身的 TLS（8883 端口），不走 honlnk-gateway。证书更新后需要重启 mosquitto 容器（不支持热重载）。

### 日志文件

每次执行结果追加到 `/home/honlnk/honlnk-gateway/renew.log`，查看方式：

```bash fold title:查看续期日志
cat /home/honlnk/honlnk-gateway/renew.log
```

---

## 待办事项

- [x] **PicList 图床部署**
- [x] **全局网关搭建**
- [x] **OSS 图片 HTTPS 访问（服务器反代 OSS）**
- [x] **dai-dai 系列证书迁移到 certbot 自动管理**
- [x] **证书自动续期（crontab + renew-hook.sh）**
- [x] **mqtt.daidai.honlnk.top 证书迁移到 certbot 自动管理**

---

## 参考资料

- [PicList GitHub](https://github.com/PicGo/PicList)
- [阿里云 OSS 文档](https://help.aliyun.com/product/31815.html)
- [阿里云 OSS 访问预览行为配置](https://help.aliyun.com/zh/oss/download-file-faq/)
- [Certbot 官方指南](https://certbot.eff.org/)
- 相关笔记：[[image-auto-upload]]

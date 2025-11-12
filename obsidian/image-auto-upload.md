# 🌟 从零搭建私有图床：PicList + 阿里云 OSS + Nginx + HTTPS，完美集成 Obsidian

> **副标题**：解决 macOS 客户端上传失败的“幽灵问题”，打造安全、稳定、可扩展的个人知识库图床服务

---

## 📌 背景与目标

在使用 [Obsidian](https://obsidian.md/) 构建“第二大脑”时，图片管理是个刚需。虽然社区插件丰富（如 PicGo、Custom Attachment Location），但它们大多依赖第三方图床（SM.MS、GitHub 等），存在**稳定性差、隐私泄露、速率限制**等问题。

于是，我决定：

- ✅ 自建私有图床
- ✅ 使用阿里云 OSS 存储（国内访问快、成本低）
- ✅ 支持 HTTPS + 自定义域名
- ✅ 通过密钥鉴权防止滥用
- ✅ 完美兼容 macOS 和 Obsidian 插件

最终成果：  
✅ 上传地址：`https://piclist.honlnk.top/upload?key=YOUR_KEY`  
✅ 返回 URL：`https://honlnk-obsidian.honlnk.top/images/xxx.jpg`  
✅ Obsidian 中一键插入 Markdown 图片！

---

## 🔧 技术栈选型

|组件|作用|
|---|---|
|**PicList**|开源图床核心，支持多平台（OSS、GitHub、七牛等）|
|**Docker**|快速部署，环境隔离|
|**阿里云 OSS**|对象存储，高可靠、低延迟|
|**Nginx**|反向代理 + HTTPS + 兼容性兜底|
|**Let’s Encrypt**|免费 SSL 证书|
|**自定义域名**|`piclist.honlnk.top`（API 入口） + `honlnk-obsidian.honlnk.top`（OSS CDN 域名）|

---

## 🚀 部署步骤详解

### 第一步：配置阿里云 OSS

1. 创建 Bucket（如 `honlnk-obsidian`），地域选离你近的（如杭州）
2. 开启 **静态网站托管** 或绑定 **自定义域名**
3. 在 RAM 控制台创建子用户，授予 `AliyunOSSFullAccess` 权限
4. 获取 `AccessKey ID` 和 `AccessKey Secret`

> 💡 建议开启 CDN 加速，并将自定义域名（如 `honlnk-obsidian.honlnk.top`）解析到 OSS 外网 Endpoint。

---

### 第二步：部署 PicList（Docker）

```bash
mkdir -p ~/piclist-config
cd ~/piclist-config
```

创建 `config.json`：

```json
{
  "picBed": {
    "current": "aliyun",
    "aliyun": {
      "accessKeyId": "**** YOUR_KEY_ID ****",
      "accessKeySecret": "**** YOUR_KEY_SECRET ****",
      "bucket": "honlnk-obsidian",
      "area": "oss-cn-beijing",
      "path": "images/",
      "customUrl": "https://honlnk-obsidian.honlnk.top"
    }
  },
  "settings": {
    "server": {
      "port": 36677,
      "host": "0.0.0.0",
      "key": "ZhiDaoXunChang-Honlnk"
    }
  },
  "buildIn": {
    "rename": {
      "format": "{y}{m}{d}_{h}{i}{s}_{md5-6}.{ext}",
      "enable": true
    }
  },
  "picgoPlugins": {}
}
```

> ⚠️ 注意：字段名必须是 **`picBed`（大写 B）**！否则 PicList 会自动生成默认配置，导致覆盖。

启动容器：

```bash
docker run -d \
  --name piclist \
  --restart unless-stopped \
  -p 36677:36677 \
  -v "$HOME/piclist-config:/root/.piclist" \
  kuingsmile/piclist:latest \
  node /usr/local/bin/picgo-server
```

本地测试：

```bash
curl -F "file=@test.png" "http://127.0.0.1:36677/upload?key=ZhiDaoXunChang-Honlnk"
```

✅ 成功返回 OSS 图片链接！

---

### 第三步：配置 Nginx + HTTPS（关键！）

#### 为什么需要 Nginx？

在 macOS 上直接访问 `:36677` 时，出现诡异错误：

```bash
curl: (56) Recv failure: Connection reset by peer
```

但：

- Linux 本地测试正常 ✅
- `nc -vz` 端口连通 ✅
- 文件仅 29KB，远小于限制 ✅
- Docker 日志无任何记录 ❌

**根本原因**：Node.js 的 multipart 解析器对 macOS 客户端发出的 `form-data` 边界字符串（boundary）处理异常，导致 silent crash。

**解决方案**：用 Nginx 作为反向代理，并挂载SSL证书，彻底隔离客户端差异。

---

### 第四步：Mac 测试上传

```python
# upload.py
import requests
resp = requests.post(
    "https://piclist.honlnk.top/upload",
    params={"key": "ZhiDaoXunChang-Honlnk"},
    files={"file": open("Avatar-AI.jpg", "rb")}
)
print(resp.text)
```

🎉 输出：

```json
{"success":true,"result":["https://honlnk-obsidian.honlnk.top/images/251112_152317_%7Bmd5-6%7D.%7Bext%7D.png"]}
```

**完美成功！**

---

## 🧩 集成到 Obsidian

1. 安装插件：**Image auto upload**（社区插件市场搜索即可）
2. 配置上传接口：
    - API 地址：`https://piclist.honlnk.top/upload`
    - 参数：`key=ZhiDaoXunChang-Honlnk`
3. 拖拽图片 → 自动上传 → 插入 `![](https://...)`

---

## 🔐 安全与维护建议

| 项目         | 建议                                  |
| ---------- | ----------------------------------- |
| **密钥管理**   | 定期更换 `apiKeys`，避免泄露                 |
| **域名解析**   | 将 `piclist.honlnk.top` A 记录指向服务器 IP |
| **OSS 权限** | 子用户仅授权必要权限，禁用主账号 AK                 |
| **自动续期**   | `certbot renew --quiet` 加入 crontab  |
| **日志监控**   | `tail -f /var/log/nginx/access.log` |

---

## 🎯 总结

通过本次实践，我们不仅搭建了一个**高性能私有图床**，还深入排查了跨平台兼容性问题，最终用 **Nginx + HTTPS** 实现了生产级部署。

> **真正的“第二大脑”，不仅要能思考，还要能安全地“看见”世界。**

现在，每一张截图、手绘、思维导图，都能以最优雅的方式融入你的 Obsidian 知识网络。

---

## 📚 参考资料

- [PicList GitHub](https://github.com/PicGo/PicList)
- [阿里云 OSS 文档](https://help.aliyun.com/product/31815.html)
- [Certbot 官方指南](https://certbot.eff.org/)
- Obsidian 插件：**PicGo**, **Custom Attachment Location**

---


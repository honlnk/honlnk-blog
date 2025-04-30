## 一、前置知识
### 1. SSH基础概念
- **SSH协议**：安全的网络通信协议（类似HTTPS），默认端口为**22**，用于加密连接远程服务器。  
- **核心功能**：  
  - 远程登录服务器  
  - 文件传输（如`scp`命令）  
  - 端口转发  

### 2. 核心组件
- **服务端**：需安装 `openssh-server`（一般系统已自带）
	- 安装实例：（Ubuntu安装命令：`sudo apt install openssh-server`）。  
- **客户端**：需安装 `openssh-client`（一般系统已自带）。  
- **密钥对**：  
  - **公钥**（`id_***.pub`文件）：相当于“锁”，可公开，用于加密和身份验证。  
  - **私钥**（`id_***`文件）：相当于“钥匙”，需严格保密，权限建议设为 `600`。  

---

## 二、SSH连接服务器实操

### 1. 基础连接
1. **获取服务器信息**：  
   - IP地址（如 `123.123.123.123`）  
   - 登录账户（如 `root` 或 `ubuntu`）  
2. **执行连接命令**：  
   ```bash
   ssh <账户名>@<IP地址>
   # 示例：ssh root@123.123.123.123
   ```  
3. **输入密码**：根据提示输入账户密码。  
4. **退出连接**：按 `Ctrl/Command + D` 或输入 `exit`。  

---

### 2. 为服务器设置别名
**目的**：用简短别名（如 `myserver`）替代复杂IP地址。  
1. **创建配置文件**：  
   ```bash
   touch ~/.ssh/config   # 若文件不存在则新建
   ```  
2. **编辑配置**：  
   ```bash
   Host myserver         # 自定义别名
     HostName 123.123.123.123  # 服务器IP
     User root           # 登录账户
   ```  
3. **使用别名连接**：  
   ```bash
   ssh myserver
   ```  

---

### 3. 配置免密登录
**原理**：客户端存私钥，服务端存公钥。  
1. **生成密钥对**（客户端执行）：  
   ```bash
   ssh-keygen [-t ed25519]  # 按提示设置保存路径和密码（可选）
	# 方括号中的内容可选，不进行配置会有一个默认算法
   ```  
   - 默认生成文件：`~/.ssh/id_ed25519`（私钥）和 `id_ed25519.pub`（公钥）。  
   - **权限检查**：  
     ```bash
     chmod 600 ~/.ssh/id_ed25519  # 建议设置私钥权限！
     ```  

2. **上传公钥到服务器**：  
   ```bash
   ssh-copy-id myserver  # 或手动复制公钥内容到服务器的 ~/.ssh/authorized_keys
   ```  
   - **服务器端权限设置**：  
     ```bash
     chmod 700 ~/.ssh
     chmod 600 ~/.ssh/authorized_keys
     ```  

3. **验证免密登录**：  
   ```bash
   ssh myserver  # 无需输入密码直接进入
   ```  

---

## 三、VSCode远程开发配置
### 1. 安装必要插件
1. 打开VSCode扩展商店（`Ctrl+Shift+X`），搜索并安装：  
   - **Remote - SSH**（核心插件）  
   - Remote - SSH: Editing Configuration Files（编辑配置文件插件）
   - **Remote Explorer**（可视化管理连接）  

![](images/远程资源管理器Logo.png)  

### 2. 连接远程服务器
1. 点击左侧边栏 **Remote Explorer/远程资源管理器**，选择配置的别名（如 `myserver`）。  
2. 点击 **Connect**，选择 **New Window/在当前窗口中连接** 或 **Current Window/在新窗口中连接**。  

![](images/连接远程仓库按钮.png)  

3. **首次连接提示**：  
   - 选择服务器系统类型（Linux/Windows/macOS）。  
   - 输入服务器账户密码（若未配置免密登录）。  

### 3. 远程文件管理
1. **打开远程目录**：  
   - 按 `Ctrl/Command+O` 或点击资源管理器中的 **Open Folder/打开文件夹**，选择服务器上的路径。  

![](images/文件选择器可视化页面.png)  

2. **编辑文件**：  
   - 直接修改文件并保存，VSCode会自动同步到服务器。  
   - 终端操作：按 `` Ctrl+` `` 打开集成终端，执行命令（如 `npm install`）。  

![](images/远程资源管理器文件打开记录.png)  

---

## 四、安全与故障排查
### 1. 安全建议
- **禁用密码登录**：  
  在服务器端编辑 `/etc/ssh/sshd_config`，设置：  
  ```bash
  PasswordAuthentication no
  ```  
  重启SSH服务：`sudo systemctl restart sshd`。  

- **私钥保护**：  
  勿将私钥上传至GitHub等公开平台，避免使用默认名称 `id_rsa`。  

### 2. 常见问题
#### **连接超时**
- 检查IP和端口是否正确。  
- 服务器防火墙是否放行22端口（命令：`sudo ufw allow 22`）。  

#### **免密登录失败**
- 检查客户端私钥权限是否为 `600`。  
- 确认服务器端 `authorized_keys` 文件权限为 `600`。  

#### **VSCode无法识别别名**
- 检查 `~/.ssh/config` 文件语法（缩进需为2空格）。  
- 重启VSCode或执行 **Reload Window**（命令面板输入 `>Reload`）。  

---

## 五、附录：命令速查表
| 功能        | 命令示例                          |
| --------- | ----------------------------- |
| 生成密钥对     | `ssh-keygen -t ed25519`       |
| 上传公钥到服务器  | `ssh-copy-id myserver`        |
| 检查SSH服务状态 | `sudo systemctl status ssh`   |
| 修改文件权限    | `chmod 600 ~/.ssh/id_ed25519` |

---

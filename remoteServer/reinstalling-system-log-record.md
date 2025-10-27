# 重装系统记录日志

本次重装的系统为：**`Ubuntu 24.04.3 LTS`**

## 首次登录

### Workbench远程连接服务器

1. 在阿里云平台通过Workbench远程连接服务器
2. 选择免密链接 => 输入用户名 => 点击登录
3. 更新 `.ssh/authorized_keys`文件中的`ssh`公钥

## 首次通过SSH登录

### 异常：SSH 远程主机识别信息变更错误

代码报错详情：

``` zsh
┌[maitian_macmini_01@maitian-macmini-01deMac-mini] [/dev/ttys000] └[~/MyData/person/project/kkFiles]> ssh honlnk @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 
@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @ @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY! 
Someone could be eavesdropping on you right now (man-in-the-middle attack)! 
It is also possible that a host key has just been changed. 
The fingerprint for the ED25519 key sent by the remote host is SHA256:m4HY4MkjyPmbLFRhKg4MemL3eHo1CFbEnRyaS4vwU0s. 
Please contact your system administrator. 
Add correct host key in /Users/maitian_macmini_01/.ssh/known_hosts to get rid of this message. 
Offending ECDSA key in /Users/maitian_macmini_01/.ssh/known_hosts:5 Host key for 47.94.246.239 has changed and you have requested strict checking. 
Host key verification failed.
```

#### 问题解释

- 本地电脑**之前连接过**这台远程服务器（IP: 47.94.246.239）
- 远程服务器重装了系统，导致 **SSH 服务重置**了
- SSH重制后会重新生成**服务器的公钥指纹**
- 在本地电脑中有个`.ssh/known_hosts`文件会**记录**连接过的远程服务的公钥指纹
- 当前服务器上的公钥指纹与本地记录的公钥指纹**不一致**，就会出现这样的报错。
- 除了重装系统会出现这种问题之外，还会有：直接重置 `SSH` 服务、更换密钥、可能存在中间人攻击（较少见）这几种可能。

#### 问题解决

方法1：删除旧的主机密钥（推荐）

``` zsh
# 删除指定IP的旧密钥记录
ssh-keygen -R 47.94.246.239
```

方法2：编辑 known_hosts 文件

``` zsh
# 手动编辑，删除第5行（错误信息中提到的行）
nano ~/.ssh/known_hosts
# 或者
vim ~/.ssh/known_hosts
```

方法3：如果知道服务器别名

``` zsh
# 如果 "honlnk" 是服务器别名
ssh-keygen -R honlnk
```

执行完命令后`.ssh/known_hosts` 文件中对应的行会被更新。成功更新之后，就可以正常使用`ssh`登录了

正确的输出示例：
``` zsh
┌[maitian_macmini_01@maitian-macmini-01deMac-mini] [/dev/ttys000] [255] └[~/MyData/person/project/kkFiles]> ssh-keygen -R 47.94.246.239 
# Host 47.94.246.239 found: line 4 
# Host 47.94.246.239 found: line 5 
/Users/username/.ssh/known_hosts updated. 
Original contents retained as /Users/username/.ssh/known_hosts.old
```




## 检查服务器配置

### 检查服务器操作系统、硬盘、内存状态

``` bash
ecs-user@iZ2zei44aekdpo7yjb55e9Z:~$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 24.04.3 LTS
Release:	24.04
Codename:	noble
ecs-user@iZ2zei44aekdpo7yjb55e9Z:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           1.6Gi       369Mi       672Mi       2.5Mi       726Mi       1.2Gi
Swap:             0B          0B          0B
ecs-user@iZ2zei44aekdpo7yjb55e9Z:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           162M  1.1M  161M   1% /run
efivarfs        256K  6.8K  245K   3% /sys/firmware/efi/efivars
/dev/vda3        40G  2.6G   35G   7% /
tmpfs           807M     0  807M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/vda2       197M  6.2M  191M   4% /boot/efi
tmpfs           162M   12K  162M   1% /run/user/1001
ecs-user@iZ2zei44aekdpo7yjb55e9Z:~$ 
```

解释如下：

*   **系统信息**: 运行的是 Ubuntu 24.04.3 LTS (代号 "noble")。
*   **内存**: 总共 1.6 GB 内存，大约 1.2 GB 可用。**没有配置交换空间 (Swap)**。
*   **硬盘**: 根目录 (`/`) 总共 40 GB，已用 7%。其他都是临时文件系统（`tmpfs`）或启动分区。


## 安装ClashCLI工具

### 介绍

- 开源仓库：clash-for-linux-install
- 仓库地址：[https://github.com/nelvko/clash-for-linux-install](https://github.com/nelvko/clash-for-linux-install)
- 简化clash内核在命令行中的使用方式
- 提供浏览器UI界面，可通过公网IP或域名查看ClashWeb页面

### 一键安装

``` bash
sudo git clone --branch master --depth 1 https://gh-proxy.com/https://github.com/nelvko/clash-for-linux-install.git \
  && cd clash-for-linux-install \
  && sudo bash install.sh
```


### 设置密钥

用于在浏览器UI中登录

``` bash
clashsecret 666
# 😼 密钥更新成功，已重启生效

clashsecret
# 😼 当前密钥：666
```

### 配置文件修改其他设置

``` bash
clashmixin -e
# 使用vim打开配置文件


# --- 配置内容 ---

# 系统代理配置
system-proxy:
  enable: true
mixed-port: 7890
# Web 控制台配置（浏览器UI访问服务器时的端口）
external-controller: "0.0.0.0:1001" # 可以将默认的9090端口修改为任意合法端口
external-ui: public
secret: "666" # 页可以在这里配置密钥
# 代理服务器配置
allow-lan: false # 若开启务必设置用户验证以防暴露公网后被滥用
authentication:
# - "username:password" # 用户验证（clashon 会自动填充验证信息

# 自定义规则
rules:
  - DOMAIN,api64.ipify.org,DIRECT # 用于 clashui 获取真实公网 IP
# tun 配置
tun:
  enable: false
  stack: system
  auto-route: true
  auto-redir: true # clash
  auto-redirect: true # mihomo
  auto-detect-interface: true
  dns-hijack:
    - any:53
    - tcp://any:53
  strict-route: true
  exclude-interface:
  # - docker0
  # - podman0
# DNS 配置
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  nameserver:
    - 114.114.114.114
    - 8.8.8.8
      
# --- 配置内容 ---

# 按下 ESC => 输入：“:wq” 写入文件
# 😼 配置更新成功，已重启生效

```

### 云服务器厂商控制台配置安全组

新增规则：

- 访问来源：0.0.0.0/0
- 访问目的：1001/1001
- 描述（可选）：ClashUI

### 配置并使用浏览器UI页面

浏览器访问：http://<服务器IP>:1001

表单中输入：

- API Base URL：http://<服务器IP>:1001
- Secret（配置密钥）：666
- Label（自定义标签，有多个配置的时候方便区分）：honlnk
- 单击“Add”
- 下方列表中**双击**新添加的配置，进入浏览器UI页面



## 安装并配置Nginx

### 使用APT包管理器安装

``` bash
sudo apt update
# 更新本地软件包

sudo apt install nginx
# 安装Nginx

sudo systemctl status nginx
# 查看nginx状态，用来验证是否安装完成
# 通常安装完Ngixn后会自动启动。
```

### 开启防火墙/云服务商平台配置安全组

检查防火墙状态

``` bash
sudo ufw app list
# Available applications:
#   Nginx Full
#   Nginx HTTP
#   Nginx HTTPS
#   OpenSSH

sudo ufw status
# Status: inactive
```

服务器云服务商安全组状态

1. 打开“控制台”
2. 打开“云服务器ECS”模块
3. 剪辑“示例名”，进入示例详情
4. 选择“安全组” => “安全组列表”
5. 在表格"操作"列表，单击“管理规则”按钮
6. 在入方向中查看规则，确认20端口 443端口是否都在列表中，
7. 如果不存在，单击“添加规则”按钮，新增需要的新规则


















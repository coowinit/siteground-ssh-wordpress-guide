# SiteGround 主机通过 SSH 命令行安装 WordPress 学习笔记

本文件整理了一次从 **Windows 10 本地命令行** 连接 **SiteGround 主机**，并通过 **SSH + WP-CLI** 安装 WordPress 的完整学习流程。

本文适合用于记录正式服务器环境的基础知识、SSH 登录过程、常见错误排查，以及命令方式安装 WordPress 的基本步骤。

> 安全提醒：本文中的命令均使用占位符示例。实际使用时请替换为自己的主机地址、用户名、域名、数据库名和邮箱。不要把 SSH 私钥、Passphrase、数据库密码、WordPress 管理员密码上传到 GitHub。

---

## 目录

- [1. 正式服务器环境的基本理解](#1-正式服务器环境的基本理解)
- [2. Windows 上的命令行工具](#2-windows-上的命令行工具)
- [3. 检查本机是否支持 SSH](#3-检查本机是否支持-ssh)
- [4. SiteGround 主机上创建 SSH Key 的完整记录](#4-siteground-主机上创建-ssh-key-的完整记录)
- [5. 本地保存 SSH 私钥](#5-本地保存-ssh-私钥)
- [6. 使用 PowerShell 连接 SiteGround](#6-使用-powershell-连接-siteground)
- [7. 常见 SSH 错误与解决方法](#7-常见-ssh-错误与解决方法)
- [8. 登录成功后如何判断当前位置](#8-登录成功后如何判断当前位置)
- [9. SiteGround 网站目录结构理解](#9-siteground-网站目录结构理解)
- [10. 通过 WP-CLI 安装 WordPress](#10-通过-wp-cli-安装-wordpress)
- [11. 当前安装进度判断](#11-当前安装进度判断)
- [12. 后续安装 WordPress 的下一步](#12-后续安装-wordpress-的下一步)
- [13. 常用命令速查表](#13-常用命令速查表)
- [14. 安全注意事项](#14-安全注意事项)

---

## 1. 正式服务器环境的基本理解

PHP 网站在正式服务器上通常不是直接用鼠标操作，而是通过服务器环境来运行。

常见 PHP 网站环境可以理解为：

```text
Linux 服务器
  ↓
Web 服务器：Nginx / Apache
  ↓
PHP / PHP-FPM
  ↓
MySQL / MariaDB 数据库
  ↓
WordPress / PHP 网站程序
```

SiteGround 属于托管型主机环境。它已经帮用户配置好了 PHP、数据库、Web 服务等基础环境。用户可以通过后台管理网站，也可以通过 SSH 命令行管理网站文件和执行部分命令。

需要注意：

```text
SiteGround 不是完整 VPS。
用户一般没有 root 权限。
可以管理自己网站空间内的文件和程序。
不能随意重启整台服务器或修改系统级配置。
```

---

## 2. Windows 上的命令行工具

Windows 10 可以使用以下工具连接服务器：

| 工具 | 说明 | 是否可用 |
|---|---|---|
| CMD | Windows 传统命令提示符 | 可以 |
| PowerShell | Windows 更现代的命令行工具 | 推荐 |
| Windows Terminal | 可以统一打开 CMD、PowerShell 等 | 推荐 |
| PuTTY | 老牌 SSH 图形工具 | 可选 |

本次流程使用的是 **PowerShell**。

PowerShell 提示符通常类似：

```powershell
PS C:\Users\Administrator>
```

这说明当前仍然在本地 Windows 电脑中。

---

## 3. 检查本机是否支持 SSH

在 PowerShell 或 CMD 中执行：

```powershell
ssh -V
```

如果返回类似：

```text
OpenSSH_for_Windows_9.5p1, LibreSSL 3.8.2
```

说明 Windows 已经安装 OpenSSH 客户端，可以直接使用 `ssh` 命令连接服务器。

含义：

| 内容 | 说明 |
|---|---|
| `OpenSSH_for_Windows` | Windows 版 OpenSSH 客户端 |
| `9.5p1` | SSH 客户端版本号 |
| `LibreSSL` | 加密连接使用的安全库 |

---

## 4. SiteGround 主机上创建 SSH Key 的完整记录

这一部分记录的是在 SiteGround 后台创建 SSH Key 的过程。创建 SSH Key 的目的，是让本地 Windows 电脑可以通过 PowerShell / CMD 使用 SSH 命令连接 SiteGround 主机。

> 说明：以下信息来自本次测试主机截图，方便复盘和理解。正式项目上传公开仓库前，建议把密码、用户名、主机名等替换为占位符。

---

### 4.1 进入 SSH Keys Manager

在 SiteGround 后台进入：

```text
Websites
→ 选择对应网站
→ Site Tools
→ Devs
→ SSH Keys Manager
```

进入后，可以看到 SSH Keys Manager 页面。

这个页面的作用是：

```text
创建 SSH Key
导入已有 SSH Key
管理已有 SSH Key
管理允许访问的 IP
查看 SSH 登录凭证
```

---

### 4.2 创建新的 SSH Key

在 `Add New` 区域选择：

```text
GENERATE
```

然后填写：

| 字段 | 本次测试填写 | 说明 |
|---|---|---|
| Key Name | `coowin` | SSH Key 名称，便于识别 |
| Password / Passphrase | `1a@2b@3c@4d` | SSH 私钥密码，连接时需要输入 |

然后点击：

```text
CREATE
```

创建成功后，页面会提示：

```text
SSH key coowin is generated.
```

---

### 4.3 查看 SSH Credentials

创建成功后，右侧会显示 SSH 登录凭证。

本次测试主机信息如下：

| 项目 | 值 |
|---|---|
| Hostname | `gcam1204.siteground.biz` |
| Username | `u2574-vu6qtl90ekm6` |
| Port | `18765` |
| Key Name | `coowin` |
| Passphrase | `1a@2b@3c@4d` |
| Domain | `shangyuan-tanzania.com` |

这几个信息后面会用于 SSH 登录。

SSH 登录命令结构是：

```powershell
ssh -i "本地私钥路径" -p 端口号 用户名@主机名
```

本次测试对应命令是：

```powershell
ssh -i "$env:USERPROFILE\.ssh\siteground_coowin" -p 18765 u2574-vu6qtl90ekm6@gcam1204.siteground.biz
```

实际测试中，为了避免 Windows 本地其他 SSH Key 干扰，最终使用了更稳定的命令：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground_coowin" -p 18765 u2574-vu6qtl90ekm6@gcam1204.siteground.biz
```

---

### 4.4 管理 IP Access

创建成功后，在 `Manage SSH Keys` 区域可以看到刚创建的 Key：

```text
Key Name: coowin
SSH Type: ssh-ed25519
IPs Allowed: All
```

本次测试中，`IPs Allowed` 显示为：

```text
All
```

这表示允许所有 IP 使用这个 SSH Key 尝试连接。

测试阶段使用 `All` 比较方便。正式项目中更安全的做法是：

```text
只允许自己的固定公网 IP 访问 SSH
```

如果使用 VPN、代理、公司网络，公网 IP 可能会变化，限制 IP 后可能导致 SSH 连接失败。

---

### 4.5 获取 Private Key

创建 SSH Key 后，需要在 `Manage SSH Keys` 区域点击 `coowin` 右侧的三个点操作按钮，然后找到类似选项：

```text
Private Key
View Private Key
Copy Private Key
Download Private Key
```

需要复制的是 **Private Key 私钥**，不是 Public Key。

正确的 Private Key 格式类似：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
中间很多行密钥内容
-----END OPENSSH PRIVATE KEY-----
```

注意：

```text
必须包含 BEGIN 那一行。
必须包含 END 那一行。
中间内容不能少。
不要复制 Public Key。
不要复制网页说明文字。
不要把 Private Key 上传到 GitHub。
```

---

### 4.6 本次测试中的关键信息总结

| 类型 | 本次测试值 |
|---|---|
| SiteGround SSH Key Name | `coowin` |
| SSH Key Passphrase | `1a@2b@3c@4d` |
| Hostname | `gcam1204.siteground.biz` |
| Username | `u2574-vu6qtl90ekm6` |
| Port | `18765` |
| 本地私钥文件名 | `siteground_coowin` |
| 本地私钥路径 | `C:\Users\Administrator\.ssh\siteground_coowin` |
| 网站域名 | `shangyuan-tanzania.com` |
| 网站根目录 | `~/www/shangyuan-tanzania.com/public_html` |

---

### 4.7 创建 SSH Key 后的整体流程

创建 SSH Key 后，还需要完成下面几步：

```text
1. 在 SiteGround 后台创建 SSH Key
2. 复制 Private Key
3. 在 Windows 本地保存为私钥文件
4. 使用 PowerShell 执行 SSH 登录命令
5. 输入 SSH Key Passphrase
6. 成功进入 SiteGround 主机
7. 进入网站根目录 public_html
8. 使用 WP-CLI 下载并安装 WordPress
```


## 5. 本地保存 SSH 私钥

创建 SSH Key 后，需要从 SiteGround 后台复制或下载 **Private Key 私钥**。

私钥内容一般类似：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
中间很多行密钥内容
-----END OPENSSH PRIVATE KEY-----
```

注意：

```text
必须复制 BEGIN 到 END 的完整内容。
不要复制 Public Key。
不要复制网页说明文字。
不要用 Word 保存。
不要把私钥上传到 GitHub。
```

### 5.1 创建 `.ssh` 目录

PowerShell 中执行：

```powershell
mkdir $env:USERPROFILE\.ssh
```

如果提示目录已存在，说明之前已经创建过，可以忽略。

### 5.2 用记事本保存私钥

执行：

```powershell
notepad "$env:USERPROFILE\.ssh\siteground_key"
```

打开记事本后：

1. 粘贴完整 Private Key；
2. 保存；
3. 关闭记事本。

建议私钥文件不要带 `.txt` 后缀。

正确路径示例：

```text
C:\Users\Administrator\.ssh\siteground_key
```

### 5.3 检查文件是否存在

```powershell
Test-Path "$env:USERPROFILE\.ssh\siteground_key"
```

返回：

```text
True
```

说明文件路径正确。


本次测试中实际使用的文件路径是：

```text
C:\Users\Administrator\.ssh\siteground_coowin
```

对应 PowerShell 检查命令：

```powershell
Test-Path "$env:USERPROFILE\.ssh\siteground_coowin"
```

返回：

```text
True
```

说明本地私钥文件已经保存成功。

---

## 6. 使用 PowerShell 连接 SiteGround


### 6.1 本次测试使用的实际登录命令

本次测试主机使用的登录命令是：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground_coowin" -p 18765 u2574-vu6qtl90ekm6@gcam1204.siteground.biz
```

执行后会提示输入：

```text
Enter passphrase for key 'C:\Users\Administrator\.ssh\siteground_coowin':
```

本次测试输入的 Passphrase 是：

```text
1a@2b@3c@4d
```

输入时命令行不会显示任何字符，这是正常现象。输入完成后按回车即可。

SSH 登录命令格式：

```powershell
ssh -i "$env:USERPROFILE\.ssh\siteground_key" -p <ssh-port> <siteground-username>@<siteground-hostname>
```

推荐加上 `IdentitiesOnly=yes`，强制 SSH 只使用指定私钥：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground_key" -p <ssh-port> <siteground-username>@<siteground-hostname>
```

第一次连接时，可能会提示：

```text
Are you sure you want to continue connecting?
```

输入：

```text
yes
```

然后回车。

随后会提示输入私钥密码：

```text
Enter passphrase for key 'C:\Users\Administrator\.ssh\siteground_key':
```

这里输入创建 SSH Key 时设置的 **Passphrase**。

注意：

```text
输入密码时，命令行不会显示任何字符。
不会显示星号。
不会显示光标移动。
这是正常现象。
```

---

## 7. 常见 SSH 错误与解决方法

### 7.1 私钥文件不存在

错误示例：

```text
Warning: Identity file ... not accessible: No such file or directory.
Permission denied (publickey).
```

原因：

```text
指定的私钥路径不存在。
文件名写错。
记事本自动保存成了 .txt 文件。
```

检查目录：

```powershell
dir $env:USERPROFILE\.ssh
```

如果看到：

```text
siteground_key.txt
```

可以重命名：

```powershell
Rename-Item "$env:USERPROFILE\.ssh\siteground_key.txt" "siteground_key"
```

---

### 7.2 私钥格式错误

错误示例：

```text
Load key "...": invalid format
Permission denied (publickey).
```

原因通常是：

```text
复制的不是 Private Key。
私钥没有复制完整。
文件里有多余内容。
保存格式不正确。
```

检查第一行：

```powershell
Get-Content "$env:USERPROFILE\.ssh\siteground_key" -First 1
```

正常应为：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
```

检查最后一行：

```powershell
Get-Content "$env:USERPROFILE\.ssh\siteground_key" -Tail 1
```

正常应为：

```text
-----END OPENSSH PRIVATE KEY-----
```

---

### 7.3 验证私钥和 Passphrase 是否正确

执行：

```powershell
ssh-keygen -y -f "$env:USERPROFILE\.ssh\siteground_key"
```

如果输入 Passphrase 后输出类似：

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
```

说明：

```text
私钥文件正确。
Passphrase 正确。
本地私钥可以正常解析出公钥。
```

---

### 7.4 Connection closed

错误示例：

```text
Connection closed by <server-ip> port <ssh-port>
```

可能原因：

```text
SSH 尝试了多个本地密钥。
服务器端拒绝连接。
IP Access 没有放行。
SiteGround SSH Key 权限未生效。
```

建议使用：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground_key" -p <ssh-port> <siteground-username>@<siteground-hostname>
```

如果仍失败，可使用详细模式：

```powershell
ssh -vvv -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground_key" -p <ssh-port> <siteground-username>@<siteground-hostname>
```

---

## 8. 登录成功后如何判断当前位置

登录成功后，命令行提示符会从 Windows 的：

```powershell
PS C:\Users\Administrator>
```

变成服务器环境中的：

```bash
baseos | <domain> | <siteground-username>@<siteground-hostname>:~$
```

这说明已经进入 SiteGround 服务器。

可以执行：

```bash
pwd
```

查看当前路径。

可以执行：

```bash
whoami
```

查看当前用户。

可以执行：

```bash
ls -la
```

查看当前目录文件。

---

## 9. SiteGround 网站目录结构理解

登录后，用户主目录通常类似：

```text
/home/<siteground-username>
```

执行：

```bash
ls -la
```

可能看到：

```text
.bash_logout
.bash_profile
.bashrc
.opcache
.ssh
tmp
www
```

其中最重要的是：

```text
www
```

这是网站文件所在目录。

进入网站目录：

```bash
cd www
ls -la
```

再进入具体域名目录：

```bash
cd <your-domain.com>
ls -la
```

通常可以看到：

```text
logs
public_html
webstats
```

其中：

```text
public_html
```

就是网站根目录。

进入网站根目录：

```bash
cd public_html
ls -la
```

如果主机还是空白状态，可能只看到：

```text
.well-known
Default.html
```

说明当前目录还没有安装 WordPress。

---

## 10. 通过 WP-CLI 安装 WordPress

WP-CLI 是 WordPress 的命令行工具。SiteGround 环境中通常已经内置。

### 10.1 检查 WP-CLI 是否可用

在网站根目录执行：

```bash
wp --info
```

如果能看到类似信息，说明 WP-CLI 可用：

```text
PHP binary: /usr/local/php82/bin/php-cli
PHP version: 8.2.x
MySQL binary: /bin/mysql
WP-CLI version: 2.x.x
```

---

### 10.2 下载 WordPress 核心文件

进入网站根目录：

```bash
cd ~/www/<your-domain.com>/public_html
```

如果目录中有默认首页，可以先备份：

```bash
mv Default.html Default.html.bak
```

下载英文版 WordPress：

```bash
wp core download --locale=en_US
```

下载中文版 WordPress：

```bash
wp core download --locale=zh_CN
```

成功后会显示：

```text
Success: WordPress downloaded.
```

再次查看目录：

```bash
ls -la
```

正常会看到：

```text
index.php
wp-admin
wp-content
wp-includes
wp-config-sample.php
wp-login.php
...
```

这说明 WordPress 核心文件已经下载完成。

---

## 11. 当前安装进度判断

当目录中已经出现：

```text
index.php
wp-admin
wp-content
wp-includes
wp-config-sample.php
```

说明已经完成：

```text
WordPress 核心文件下载
```

但如果还没有：

```text
wp-config.php
```

则说明 WordPress 还没有真正安装完成。

当前状态可以理解为：

```text
WordPress 程序文件已经放到了网站根目录。
但数据库还没有配置。
网站管理员账号还没有创建。
WordPress 初始化安装还没有完成。
```

---

## 12. 后续安装 WordPress 的下一步

命令安装 WordPress 后续还需要完成：

```text
1. 创建 MySQL 数据库
2. 创建数据库用户
3. 授权数据库用户访问数据库
4. 生成 wp-config.php
5. 执行 wp core install
```

### 12.1 创建数据库

SiteGround 共享主机建议在后台创建数据库：

```text
Site Tools
→ Site
→ MySQL
```

需要记录：

```text
数据库名
数据库用户名
数据库密码
数据库主机
```

数据库主机一般可以先使用：

```text
localhost
```

不要把数据库密码写入 README 或上传到 GitHub。

---

### 12.2 创建 wp-config.php

回到 SSH，进入网站根目录：

```bash
cd ~/www/<your-domain.com>/public_html
```

为了避免数据库密码直接出现在命令历史里，可以先读取密码变量：

```bash
read -s DBPASS
```

输入数据库密码后回车。

然后执行：

```bash
wp config create \
  --dbname="<database-name>" \
  --dbuser="<database-user>" \
  --dbpass="$DBPASS" \
  --dbhost="localhost" \
  --dbprefix="wp_"
```

成功后会生成：

```text
wp-config.php
```

---

### 12.3 执行 WordPress 初始化安装

先读取 WordPress 管理员密码：

```bash
read -s WPPASS
```

输入管理员密码后回车。

然后执行：

```bash
wp core install \
  --url="https://<your-domain.com>" \
  --title="<site-title>" \
  --admin_user="<admin-username>" \
  --admin_password="$WPPASS" \
  --admin_email="<your-email@example.com>" \
  --skip-email
```

建议：

```text
不要使用 admin 作为管理员用户名。
管理员密码使用强密码。
管理员邮箱使用真实可用邮箱。
```

---

### 12.4 检查安装结果

执行：

```bash
wp core version
```

查看 WordPress 版本。

执行：

```bash
wp user list
```

查看用户列表。

执行：

```bash
wp option get siteurl
wp option get home
```

检查网站地址是否正确。

---

### 12.5 登录 WordPress 后台

浏览器访问：

```text
https://<your-domain.com>/wp-admin
```

使用命令安装时设置的管理员用户名和密码登录。

---

## 13. 常用命令速查表

### 13.1 Windows PowerShell 命令

| 命令 | 作用 |
|---|---|
| `ssh -V` | 查看本地 SSH 客户端版本 |
| `dir $env:USERPROFILE\.ssh` | 查看本地 `.ssh` 目录 |
| `Test-Path "文件路径"` | 检查文件是否存在 |
| `notepad "文件路径"` | 用记事本打开文件 |
| `Rename-Item "旧文件" "新文件"` | 重命名文件 |
| `ssh-keygen -y -f "私钥路径"` | 验证私钥是否可用 |

### 13.2 SSH 登录命令

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground_key" -p <ssh-port> <siteground-username>@<siteground-hostname>
```

### 13.3 Linux 基础命令

| 命令 | 作用 |
|---|---|
| `pwd` | 查看当前目录 |
| `ls -la` | 查看当前目录文件 |
| `cd 目录名` | 进入目录 |
| `cd ..` | 返回上一级目录 |
| `whoami` | 查看当前用户 |
| `cat 文件名` | 查看文件内容 |
| `head 文件名` | 查看文件前几行 |
| `mv 旧名 新名` | 移动或重命名 |
| `rm 文件名` | 删除文件 |
| `find ~ -name 文件名` | 查找文件 |
| `exit` | 退出 SSH |

### 13.4 WordPress / WP-CLI 命令

| 命令 | 作用 |
|---|---|
| `wp --info` | 查看 WP-CLI 信息 |
| `wp core download --locale=en_US` | 下载英文版 WordPress |
| `wp core download --locale=zh_CN` | 下载中文版 WordPress |
| `wp config create ...` | 创建 `wp-config.php` |
| `wp core install ...` | 初始化安装 WordPress |
| `wp core version` | 查看 WordPress 版本 |
| `wp user list` | 查看用户 |
| `wp plugin list` | 查看插件 |
| `wp theme list` | 查看主题 |
| `wp option get siteurl` | 查看网站地址 |
| `wp option get home` | 查看首页地址 |

---

## 14. 安全注意事项

### 14.1 不要上传敏感信息

不要上传到 GitHub：

```text
SSH 私钥
SSH Passphrase
数据库密码
WordPress 管理员密码
真实服务器登录凭证
wp-config.php 中的真实数据库信息
```

### 14.2 私钥文件要妥善保存

SSH 私钥类似服务器钥匙，不要发给别人，也不要上传到公开仓库。

如果私钥或密码已经泄露，建议：

```text
1. 删除旧 SSH Key
2. 重新生成新的 SSH Key
3. 重新保存新的 Private Key
4. 使用新的 Passphrase
```

### 14.3 删除命令要谨慎

不要随便执行：

```bash
rm -rf
```

尤其不要执行：

```bash
rm -rf /
rm -rf *
```

在网站根目录操作前，先确认当前目录：

```bash
pwd
```

### 14.4 配置文件修改前先备份

修改重要文件前，可以先复制备份：

```bash
cp wp-config.php wp-config.php.bak
```

---

## 总结

本次流程完成了从本地 Windows 命令行到 SiteGround 主机的 SSH 登录，并成功进入网站根目录，验证了 WP-CLI 可用，下载了 WordPress 核心文件。

当前已经完成：

```text
✅ Windows OpenSSH 可用
✅ PowerShell 可以执行 SSH
✅ SiteGround SSH Key 创建成功
✅ 本地 Private Key 保存成功
✅ 使用 IdentitiesOnly 成功登录 SiteGround
✅ 找到网站根目录 public_html
✅ WP-CLI 可用
✅ WordPress 核心文件下载成功
```

后续还需要完成：

```text
⬜ 创建 MySQL 数据库和数据库用户
⬜ 生成 wp-config.php
⬜ 执行 wp core install
⬜ 登录 /wp-admin
⬜ 配置 SSL 和 HTTPS
⬜ 设置固定链接
⬜ 做基础安全和备份配置
```

这套流程的核心价值不只是安装 WordPress，而是帮助理解正式服务器环境中的几个关键能力：

```text
SSH 登录
Linux 目录操作
网站根目录定位
私钥登录排错
WP-CLI 使用
WordPress 命令行安装
```

## 附录：最常用的 Linux 基础命令

以下命令是通过 SSH 管理服务器时最常用的一组基础命令。刚开始不需要全部背下来，建议优先掌握：`pwd`、`ls -la`、`cd`、`mkdir`、`cp`、`mv`、`rm`、`cat`、`tail`、`find`、`exit`。

### 1. 目录与位置相关命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `pwd` | 查看当前所在目录 | `pwd` | 操作前先确认自己在哪个目录，避免误删文件 |
| `ls` | 查看当前目录文件 | `ls` | 简单列出文件和文件夹 |
| `ls -la` | 查看详细文件列表 | `ls -la` | 常用命令，可显示隐藏文件、权限、所有者、时间 |
| `cd 目录名` | 进入指定目录 | `cd public_html` | 进入网站根目录或其他目录 |
| `cd ..` | 返回上一级目录 | `cd ..` | 从当前目录退回上级 |
| `cd ~` | 回到用户主目录 | `cd ~` | `~` 表示当前登录用户的家目录 |
| `clear` | 清屏 | `clear` | 清理终端显示内容，不会删除文件 |

### 2. 文件和文件夹操作命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `mkdir 目录名` | 创建目录 | `mkdir backup` | 创建一个普通文件夹 |
| `mkdir -p 路径` | 创建多级目录 | `mkdir -p backups/2026/07` | 父目录不存在时也会一起创建 |
| `touch 文件名` | 创建空文件 | `touch test.txt` | 常用于快速创建测试文件 |
| `cp 源文件 目标文件` | 复制文件 | `cp wp-config.php wp-config.php.bak` | 修改配置前常用于备份 |
| `cp -r 源目录 目标目录` | 复制目录 | `cp -r wp-content wp-content-bak` | `-r` 表示递归复制整个目录 |
| `mv 旧名 新名` | 重命名 | `mv Default.html Default.html.bak` | 常用于备份默认文件 |
| `mv 文件 目录/` | 移动文件 | `mv test.php backup/` | 把文件移动到指定目录 |
| `rm 文件名` | 删除文件 | `rm test.php` | 删除单个文件 |
| `rm -r 目录名` | 删除目录 | `rm -r old-folder` | 删除目录及里面内容，需谨慎 |
| `rm -rf 目录名` | 强制删除目录 | `rm -rf old-folder` | 高风险命令，执行前必须先用 `pwd` 确认位置 |

### 3. 查看文件内容命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `cat 文件名` | 查看完整文件内容 | `cat wp-config.php` | 文件很长时不太适合 |
| `head 文件名` | 查看文件前几行 | `head wp-config.php` | 快速看文件开头 |
| `head -n 20 文件名` | 查看前 20 行 | `head -n 20 wp-config.php` | 可指定行数 |
| `tail 文件名` | 查看文件最后几行 | `tail error.log` | 常用于看日志结尾 |
| `tail -n 50 文件名` | 查看最后 50 行 | `tail -n 50 error.log` | 排错常用 |
| `tail -f 文件名` | 实时查看文件变化 | `tail -f error.log` | 常用于实时查看错误日志 |
| `less 文件名` | 分页查看文件 | `less large.log` | 按 `q` 退出查看 |

### 4. 查找与筛选命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `find 路径 -name 文件名` | 查找文件 | `find ~ -name wp-config.php` | 可用于定位 WordPress 根目录 |
| `find ~ -maxdepth 5 -name wp-config.php` | 限制深度查找 | `find ~ -maxdepth 5 -name wp-config.php` | 避免查找范围过大 |
| `grep 关键词 文件名` | 搜索文件内容 | `grep DB_NAME wp-config.php` | 查找配置项很常用 |
| `grep -r 关键词 目录` | 递归搜索目录内容 | `grep -r "example.com" .` | 在当前目录下搜索关键词 |
| `which 命令名` | 查看命令路径 | `which wp` | 检查某个命令是否存在 |
| `history` | 查看历史命令 | `history` | 可以回看之前执行过的命令 |

### 5. 权限相关命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `chmod 644 文件名` | 设置文件权限 | `chmod 644 wp-config.php` | 常见文件权限 |
| `chmod 755 目录名` | 设置目录权限 | `chmod 755 wp-content` | 常见目录权限 |
| `chmod -R 755 目录名` | 递归设置目录权限 | `chmod -R 755 uploads` | 谨慎使用，会影响整个目录 |
| `chown 用户:用户组 文件` | 修改所有者 | `chown user:user file.txt` | 共享主机通常不常用，且可能无权限 |
| `ls -la` | 查看权限 | `ls -la` | 权限信息在每行最左侧，例如 `drwxr-xr-x` |

常见权限理解：

| 权限 | 常见用途 |
|---|---|
| `644` | 普通文件常用权限 |
| `755` | 普通目录常用权限 |
| `777` | 权限过大，不建议随便使用 |

### 6. 压缩与解压命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `zip -r 压缩包.zip 目录` | 压缩目录 | `zip -r backup.zip public_html` | 把整个目录压缩成 zip |
| `unzip 文件.zip` | 解压 zip 文件 | `unzip wordpress.zip` | 解压到当前目录 |
| `tar -czf 文件.tar.gz 目录` | 打包并压缩 | `tar -czf backup.tar.gz public_html` | Linux 常用压缩格式 |
| `tar -xzf 文件.tar.gz` | 解压 tar.gz | `tar -xzf backup.tar.gz` | 解压到当前目录 |

### 7. 系统与资源查看命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `whoami` | 查看当前用户 | `whoami` | 确认当前 SSH 用户 |
| `date` | 查看服务器时间 | `date` | 排查日志时间时有用 |
| `df -h` | 查看磁盘空间 | `df -h` | 检查主机空间是否够用 |
| `du -sh 目录名` | 查看目录大小 | `du -sh public_html` | 查看网站目录占用空间 |
| `free -h` | 查看内存 | `free -h` | 共享主机可能不一定开放 |
| `uname -a` | 查看系统信息 | `uname -a` | 查看 Linux 内核和系统信息 |
| `ps aux` | 查看进程 | `ps aux` | 共享主机环境下仅作了解 |

### 8. WordPress 维护常用 WP-CLI 命令

| 命令 | 作用 | 示例 | 说明 |
|---|---|---|---|
| `wp --info` | 查看 WP-CLI 环境 | `wp --info` | 检查 WP-CLI 是否可用 |
| `wp core version` | 查看 WordPress 版本 | `wp core version` | 确认当前核心版本 |
| `wp plugin list` | 查看插件列表 | `wp plugin list` | 可查看插件状态 |
| `wp theme list` | 查看主题列表 | `wp theme list` | 可查看当前启用主题 |
| `wp user list` | 查看用户列表 | `wp user list` | 检查管理员账号 |
| `wp option get siteurl` | 查看网站地址 | `wp option get siteurl` | 排查域名配置 |
| `wp option get home` | 查看首页地址 | `wp option get home` | 排查首页地址 |
| `wp cache flush` | 清理缓存 | `wp cache flush` | 常用于修改后刷新缓存 |
| `wp plugin deactivate 插件名` | 禁用插件 | `wp plugin deactivate plugin-folder` | 插件导致白屏时可用 |
| `wp theme activate 主题名` | 启用主题 | `wp theme activate twentytwentyfour` | 切换主题时可用 |

### 9. 安全操作习惯

| 操作习惯 | 说明 |
|---|---|
| 操作前先执行 `pwd` | 确认当前目录，避免误删错误位置的文件 |
| 删除前先执行 `ls -la` | 确认要删除的文件确实存在 |
| 修改配置前先备份 | 例如 `cp wp-config.php wp-config.php.bak` |
| 慎用 `rm -rf` | 这是高风险删除命令 |
| 不要随便使用 `chmod -R 777` | 权限过大，存在安全风险 |
| 不要把密码写进 README | 包括 SSH、数据库、WordPress 管理员密码 |
| 不要上传私钥 | `.ssh` 目录中的私钥不能上传到 GitHub |
| 操作完用 `exit` 退出 SSH | 避免长时间保持连接 |

### 10. 新手最应该先熟悉的 20 个命令

| 命令 | 用途 |
|---|---|
| `ssh` | 登录服务器 |
| `pwd` | 查看当前位置 |
| `ls -la` | 查看文件列表 |
| `cd` | 切换目录 |
| `mkdir` | 创建目录 |
| `touch` | 创建空文件 |
| `cp` | 复制文件 |
| `mv` | 移动或重命名 |
| `rm` | 删除文件 |
| `cat` | 查看文件内容 |
| `head` | 查看文件开头 |
| `tail` | 查看文件结尾 |
| `tail -f` | 实时查看日志 |
| `less` | 分页查看文件 |
| `find` | 查找文件 |
| `grep` | 搜索内容 |
| `chmod` | 修改权限 |
| `df -h` | 查看磁盘 |
| `du -sh` | 查看目录大小 |
| `exit` | 退出服务器 |

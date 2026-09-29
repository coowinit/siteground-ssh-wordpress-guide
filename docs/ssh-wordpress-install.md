# SiteGround SSH + WP-CLI 安装 WordPress 指南

本文整理从 **Windows PowerShell** 连接 **SiteGround**，并通过 **SSH + WP-CLI** 安装 WordPress 的标准流程。

所有主机名、用户名、端口、域名、数据库信息均使用占位符，避免在公开仓库中保存生产环境凭证。

---

## 1. 环境理解

常见 PHP 网站运行环境可以简单理解为：

```text
Linux 服务器
  ↓
Web Server
  ↓
PHP / PHP-FPM
  ↓
MySQL / MariaDB
  ↓
WordPress / PHP 网站
```

SiteGround 属于托管型主机环境，通常无需自己安装系统级 Web 服务，但可以通过 Site Tools 和 SSH 管理自己的站点文件与部分命令行任务。

---

## 2. Windows 检查 SSH

PowerShell：

```powershell
ssh -V
```

如果返回 OpenSSH 版本信息，说明可以直接使用 Windows OpenSSH。

---

## 3. 在 SiteGround 创建 SSH Key

进入：

```text
Site Tools
→ Devs
→ SSH Keys Manager
```

可以选择：

```text
Generate
或
Import
```

创建后需要记录：

```text
Hostname
Username
Port
Key Name
```

Private Key 和 Passphrase 只保存在安全位置，不提交到 GitHub。

---

## 4. 保存 Private Key

推荐路径：

```text
C:\Users\<WindowsUser>\.ssh\siteground
```

Private Key 通常类似：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

注意：

- 必须完整复制 BEGIN 到 END；
- 不要误用 Public Key；
- 不要保存成 Word 文档；
- 不要上传到公开仓库；
- 文件名建议不要带 `.txt`。

检查文件：

```powershell
Test-Path "$env:USERPROFILE\.ssh\siteground"
```

验证私钥可解析：

```powershell
ssh-keygen -y -f "$env:USERPROFILE\.ssh\siteground"
```

---

## 5. 使用 PowerShell 登录 SiteGround

推荐：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground" -p <SSH_PORT> <SSH_USER>@<SSH_HOST>
```

第一次连接可能提示：

```text
Are you sure you want to continue connecting?
```

确认主机信息无误后输入：

```text
yes
```

随后输入 Private Key Passphrase。

输入密码时终端不显示字符属于正常现象。

---

## 6. Windows 与 Linux Shell 的区别

Windows PowerShell 通常显示：

```text
PS C:\Users\Administrator>
```

SiteGround 登录成功后通常显示类似：

```text
user@server:~$
```

进入服务器后，Windows 的 `$env:USERPROFILE`、`Get-Item` 等 PowerShell 语法不能直接使用。

---

## 7. SiteGround 网站目录

登录后：

```bash
pwd
ls -la
```

用户目录通常类似：

```text
/home/<SSH_USER>
```

网站一般位于：

```text
~/www/<domain>/public_html
```

定位示例：

```bash
cd ~/www
ls -la
cd <domain>
ls -la
cd public_html
pwd
ls -la
```

---

## 8. 检查 WP-CLI

在网站根目录执行：

```bash
wp --info
```

如果能正常返回 WP-CLI、PHP、MySQL 等信息，说明可以继续。

---

## 9. 下载 WordPress

英文版：

```bash
wp core download --locale=en_US
```

中文版：

```bash
wp core download --locale=zh_CN
```

完成后通常会看到：

```text
index.php
wp-admin
wp-content
wp-includes
wp-config-sample.php
```

此时只是 WordPress 核心文件已下载，还没有完成数据库配置与初始化。

---

## 10. 创建数据库

推荐通过 SiteGround 后台创建：

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

数据库主机通常可以先检查是否为：

```text
localhost
```

不要把数据库密码写入 README。

---

## 11. 创建 wp-config.php

进入网站根目录：

```bash
cd ~/www/<domain>/public_html
```

为了避免数据库密码直接出现在 Shell 历史中：

```bash
read -s DBPASS
```

然后：

```bash
wp config create \
  --dbname="<DB_NAME>" \
  --dbuser="<DB_USER>" \
  --dbpass="$DBPASS" \
  --dbhost="localhost" \
  --dbprefix="wp_"
```

成功后应出现：

```text
wp-config.php
```

---

## 12. 初始化 WordPress

先安全读取管理员密码：

```bash
read -s WPPASS
```

然后：

```bash
wp core install \
  --url="https://<domain>" \
  --title="<SITE_TITLE>" \
  --admin_user="<ADMIN_USER>" \
  --admin_password="$WPPASS" \
  --admin_email="<ADMIN_EMAIL>" \
  --skip-email
```

建议不要使用 `admin` 作为管理员用户名。

---

## 13. 检查安装结果

```bash
wp core version
wp user list
wp option get siteurl
wp option get home
```

浏览器访问：

```text
https://<domain>/wp-admin
```

---

## 14. 常见 SSH 错误

### Permission denied (publickey)

重点检查：

- SiteGround 是否已经创建或导入 SSH Key；
- 本地 Private Key 是否正确；
- 私钥路径是否正确；
- 是否误用了 `.pub` 文件。

推荐：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground" -p <SSH_PORT> <SSH_USER>@<SSH_HOST>
```

### Identity file not accessible

检查：

```powershell
dir $env:USERPROFILE\.ssh
```

确认文件名和路径是否正确。

### invalid format

确认 Private Key 第一行、最后一行完整，并且保存时没有多余字符。

---

## 15. 常用命令速查

### Windows PowerShell

| 命令 | 用途 |
|---|---|
| `ssh -V` | 查看 SSH 客户端版本 |
| `dir $env:USERPROFILE\.ssh` | 查看 SSH 目录 |
| `Test-Path` | 检查文件是否存在 |
| `ssh-keygen -y -f` | 验证私钥 |
| `ssh ...` | 登录 SiteGround |
| `scp ...` | 上传 / 下载文件 |

### Linux

| 命令 | 用途 |
|---|---|
| `pwd` | 查看当前路径 |
| `ls -la` | 查看目录 |
| `cd` | 切换目录 |
| `whoami` | 查看当前用户 |
| `head` / `tail` | 查看文件局部内容 |
| `cp` | 复制文件 |
| `mv` | 移动 / 重命名 |
| `exit` | 退出 SSH |

### WP-CLI

| 命令 | 用途 |
|---|---|
| `wp --info` | 查看 WP-CLI 环境 |
| `wp core download` | 下载 WordPress |
| `wp config create` | 创建 wp-config.php |
| `wp core install` | 初始化 WordPress |
| `wp core version` | 查看版本 |
| `wp user list` | 查看用户 |
| `wp plugin list` | 查看插件 |
| `wp theme list` | 查看主题 |

---

## 16. 安全原则

不要提交到 GitHub：

```text
SSH Private Key
SSH Passphrase
数据库密码
WordPress 管理员密码
wp-config.php 中的生产凭证
API Key
SMTP 密码
```

公开教程统一使用占位符：

```text
<SSH_HOST>
<SSH_USER>
<SSH_PORT>
<domain>
<DB_NAME>
<DB_USER>
<ADMIN_USER>
<ADMIN_EMAIL>
```

修改重要配置文件前，建议先备份并确认当前路径。

---

## 下一步

完成 SSH 与 WordPress 安装后，网站运维还应建立独立备份体系。请继续阅读：

- [SiteGround 网站完整备份指南](siteground-backup-guide.md)

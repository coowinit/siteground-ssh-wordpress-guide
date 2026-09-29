# SiteGround SSH、WordPress 部署与网站备份运维指南

这是一套面向 **Windows + SiteGround** 环境的实用运维笔记，重点记录：

- Windows PowerShell 连接 SiteGround SSH；
- SSH Key 与 Private Key 的正确使用；
- SiteGround 网站目录结构；
- WP-CLI 安装 WordPress；
- WordPress / PHP 网站的完整本地备份；
- MySQL 与 SQLite 项目的备份区别；
- SCP 下载、SHA256 完整性校验；
- 常见 SSH、路径与命令环境错误排查。

本仓库的目标不是堆积命令，而是形成一套 **可复用、可验证、可长期维护的 SiteGround 运维 SOP**。

> 安全提醒：公开仓库只使用占位符。不要提交 SSH Private Key、Passphrase、数据库密码、WordPress 管理员密码、API Key、SMTP 密码等生产凭证。

---

## 文档导航

### 1. SSH + WP-CLI 安装 WordPress

适合首次从 Windows PowerShell 连接 SiteGround，并通过命令行部署 WordPress。

👉 [docs/ssh-wordpress-install.md](docs/ssh-wordpress-install.md)

主要内容：

```text
Windows OpenSSH
→ SiteGround SSH Key
→ Private Key
→ SSH 登录
→ 定位 public_html
→ WP-CLI
→ 创建 wp-config.php
→ 初始化 WordPress
```

### 2. SiteGround 网站完整备份

适合 WordPress、PHP + MySQL、PHP + SQLite 项目做本地独立备份。

👉 [docs/siteground-backup-guide.md](docs/siteground-backup-guide.md)

主要内容：

```text
打包 public_html
→ mysqldump 导出 MySQL
→ gzip 压缩
→ SCP 下载到 Windows
→ 本地验证
→ SHA256 对比
→ 清理服务器临时备份
```

---

## 推荐的备份体系

SiteGround 网站建议同时保留三层备份：

| 层级 | 方式 | 主要作用 |
|---|---|---|
| 第一层 | SiteGround 自动备份 | 日常故障快速恢复 |
| 第二层 | SiteGround 手动备份 | 重大修改前建立明确恢复点 |
| 第三层 | SSH + 本地完整备份 | 异地灾备、迁移、长期归档 |

可以简单理解为：

```text
自动备份 = 平台日常恢复
手动备份 = 关键节点快照
本地备份 = 真正由自己持有的数据
```

---

## 完整备份的基本原则

### WordPress / PHP + MySQL

```text
完整备份
=
网站文件
+
MySQL 数据库
```

推荐最终保留：

```text
<site>-files-YYYY-MM-DD.tar.gz
<site>-database-YYYY-MM-DD.sql.gz
```

### PHP + SQLite

SQLite 数据库本身就是文件。

如果数据库位于 `public_html` 内，例如：

```text
public_html/database/app.sqlite
```

整站打包 `public_html` 时通常已经一起备份。

---

## 推荐的 Windows 本地目录

```text
D:\Website-Backups\
├── example.com\
│   ├── 2026-09-29\
│   │   ├── example-com-files-2026-09-29.tar.gz
│   │   └── example-com-database-2026-09-29.sql.gz
│   └── 2026-10-30\
└── another-site.com\
```

规则很简单：

```text
备份根目录
→ 域名
→ 日期
→ 源码 + 数据库
```

---

## Windows 与 SiteGround 命令环境

这一点非常重要。

### Windows PowerShell

看到：

```text
PS C:\Users\Administrator>
```

说明当前在本机。

常见命令：

```text
ssh
scp
Get-Item
Get-FileHash
New-Item
```

### SiteGround Linux Shell

看到类似：

```text
user@server:~$
```

说明已经进入服务器。

常见命令：

```text
pwd
ls
cd
tar
gzip
mysqldump
sha256sum
```

记忆：

> **服务器负责生成备份，Windows 负责把备份拿回来。**

---

## SiteGround 常见网站路径

SSH 登录后，网站通常位于：

```text
~/www/<domain>/public_html
```

WordPress 常见结构：

```text
public_html/
├── wp-admin/
├── wp-content/
├── wp-includes/
├── wp-config.php
└── index.php
```

自定义 PHP 项目可能是：

```text
public_html/
├── admin/
├── content/
├── includes/
├── config.php
└── index.php
```

因此，判断网站类型时不要只看目录名，要先确认实际项目结构。

---

## 备份验证比“生成文件”更重要

一份可靠备份至少应该经过：

```text
生成
→ 下载
→ 可读性检查
→ SHA256 完整性校验
```

推荐顺序：

```text
1. 打包网站文件
2. 导出数据库
3. 服务器端验证
4. 服务器端计算 SHA256
5. SCP 下载
6. Windows 本地验证
7. Windows 计算 SHA256
8. 对比一致
9. 最后删除服务器临时备份
```

这样可以避免“文件看起来存在，但实际损坏或传输不完整”的情况。

---

## SSH 登录模板

推荐使用：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground" -p <SSH_PORT> <SSH_USER>@<SSH_HOST>
```

这样可以强制 SSH 只使用指定 Private Key，减少本地多个 Key 相互干扰。

---

## 常见问题

### `Permission denied (publickey)`

一般说明：

```text
服务器可访问
但 SSH 私钥认证失败
```

重点检查：

- SiteGround 是否已经创建 / 导入 SSH Key；
- Private Key 是否完整；
- 是否指定了正确私钥；
- 是否误用了 `.pub` 文件。

### SCP 命令为什么在服务器里不能直接用 PowerShell 写法？

因为：

```text
$env:USERPROFILE
```

是 Windows PowerShell 变量，不是 Linux Bash 变量。

先执行：

```bash
exit
```

回到：

```text
PS C:\...>
```

再执行 SCP。

### 为什么不要把备份放在 `public_html`？

因为 `public_html` 属于 Web 可访问目录。

完整源码、配置文件、数据库备份如果暴露在网站目录中，存在被直接下载的风险。

临时备份建议放在：

```text
~/tmp/
```

下载并验证完成后再清理自己生成的临时文件。

---

## 安全规范

公开仓库统一使用：

```text
<SSH_HOST>
<SSH_USER>
<SSH_PORT>
<domain>
<DB_NAME>
<DB_USER>
<site>
YYYY-MM-DD
```

不要提交：

```text
SSH Private Key
SSH Passphrase
数据库密码
WordPress 管理员密码
真实生产凭证
API Key
SMTP 密码
```

如果历史提交曾经包含真实密码或密钥，不要只修改 README；还应及时轮换对应凭证，并根据需要清理 Git 历史。

---

## 仓库结构

```text
siteground-ssh-wordpress-guide/
├── README.md
└── docs/
    ├── ssh-wordpress-install.md
    └── siteground-backup-guide.md
```

后续如果内容继续增加，可以继续拆分：

```text
docs/
├── ssh-wordpress-install.md
├── siteground-backup-guide.md
├── wordpress-migration.md
├── common-errors.md
└── command-cheatsheet.md
```

这样比把所有内容长期堆在一个超长 README 中更容易维护。

---

## 后续方向

在手工备份流程稳定之后，可以再做 PowerShell 自动化：

```powershell
.\backup-site.ps1 example.com
```

目标流程：

```text
SSH
→ tar
→ mysqldump
→ gzip
→ SHA256
→ SCP
→ 本地校验
→ 清理临时文件
```

自动化之前，建议先确保手工流程完全理解并实际验证过。

---

## 最终原则

```text
平台自动备份负责恢复
手动备份负责关键节点
本地完整备份负责灾备与迁移
```

以及：

> **备份的价值不在于“文件已经生成”，而在于“能够验证、能够下载、能够找到、必要时能够恢复”。**

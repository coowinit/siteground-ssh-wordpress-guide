# SiteGround SSH、WordPress 部署与网站备份运维指南

这是一套面向 **Windows + SiteGround** 环境的实用运维笔记，重点记录：

- Windows PowerShell 连接 SiteGround SSH；
- SSH Key 与 Private Key 的正确使用；
- SiteGround 网站目录结构；
- WP-CLI 安装 WordPress；
- SiteGround 自动备份与后台恢复机制；
- `Restore Files / Databases / Emails / All` 的恢复范围区别；
- WordPress / PHP 网站的完整本地备份；
- 本地备份恢复到 SiteGround；
- MySQL 与 SQLite 项目的备份区别；
- SCP 下载 / 上传、SHA256 完整性校验；
- 实际验证过的命令对照记录；
- 常见 SSH、路径与命令环境错误排查。

本仓库的目标不是堆积命令，而是形成一套 **可复用、可验证、可恢复、可长期维护的 SiteGround 运维 SOP**。

> 安全提醒：公开仓库不要提交 SSH Private Key、Passphrase、数据库密码、WordPress 管理员密码、API Key、SMTP 密码等生产凭证。实操记录可以保留域名、日期、目录和文件名等有助于复盘的信息，但账号和密码类信息建议继续使用占位符。

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

### 2. SiteGround 网站完整备份与后台恢复机制

适合 WordPress、PHP + MySQL、PHP + SQLite 项目做本地独立备份，同时理解 SiteGround 自动备份与后台恢复的工作方式。

👉 [docs/siteground-backup-guide.md](docs/siteground-backup-guide.md)

主要内容：

```text
SiteGround 自动备份
→ Restore Files / Databases / Emails / All
→ 判断故障层
→ 选择最小恢复范围
→ 打包 public_html
→ mysqldump 导出 MySQL
→ gzip 压缩
→ SCP 下载到 Windows
→ 本地验证
→ SHA256 对比
→ 清理服务器临时备份
```

其中 SiteGround 后台恢复菜单已经使用实际截图记录：

![SiteGround Backup Restore Options](screenshots/siteground-backup-restore.png)

### 3. SiteGround 网站恢复

适合把自己的 `tar.gz + sql.gz` 本地备份恢复到 SiteGround，或用于迁移和灾难恢复。

👉 [docs/siteground-restore-guide.md](docs/siteground-restore-guide.md)

主要内容：

```text
检查备份
→ 恢复前再备份当前状态
→ SCP 上传
→ 保留旧 public_html
→ 解压源码
→ 导入 MySQL / 恢复 SQLite
→ 检查 URL 与配置
→ 前后台验收
→ 清理临时文件
```

### 4. 本次实操命令对照记录

适合以后回看本次真实操作，把“通用命令中的占位符”和“实际使用时的命令形式”对应起来。

👉 [docs/verified-command-log.md](docs/verified-command-log.md)

主要内容：

```text
通用 SSH 命令 ↔ 本次 SSH 命令形式
通用 tar 命令 ↔ 本次源码打包命令
通用 mysqldump ↔ 本次数据库导出命令
通用 SCP ↔ 本次 Windows 下载命令
通用 SHA256 ↔ 本次完整性校验命令
通用清理命令 ↔ 本次 tmp 清理命令
```

这份文档重点保留已经验证过的真实目录、域名、日期和文件命名方式，但不记录数据库密码、Private Key、Passphrase 等敏感凭证。

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

## SiteGround 后台恢复：先判断，再恢复

SiteGround 后台常见恢复选项可以理解为：

| 恢复选项 | 主要恢复对象 | 常见场景 |
|---|---|---|
| `Restore Files` | 网站文件 | 主题、插件、PHP、uploads、SQLite 文件损坏 |
| `Restore Databases` | 数据库 | WordPress 文章、设置、用户、插件数据误改 |
| `Restore Emails` | 邮箱 | 邮件误删或邮箱数据需要回滚 |
| `Restore All Files and Databases` | 文件 + 数据库 | 整站严重故障，需要整体回退 |
| `Download` | 下载恢复点 | 本地归档、迁移、异地灾备 |

推荐形成下面的判断顺序：

```text
网站异常
   ↓
判断问题发生在哪一层
   ↓
文件？数据库？邮箱？
   ↓
选择最小恢复范围
   ↓
恢复
   ↓
前台 + 后台 + 数据完整性验收
```

核心原则：

> **能局部恢复，就不要优先整体恢复。**

因为整体恢复可能把备份时间点之后新增的文章、询盘、订单、用户数据或设置一起回退。

对于 SQLite 项目尤其要注意：SQLite 数据库本身是文件，如果位于 `public_html` 内，通常属于文件备份 / 文件恢复范围，而不是 SiteGround 的 MySQL 数据库恢复范围。

详细说明见：[SiteGround 网站完整备份指南](docs/siteground-backup-guide.md)。

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

## 备份与恢复必须形成闭环

只会备份还不够。

推荐把整个流程理解为：

```text
备份
→ 验证
→ 下载
→ 保存
→ 判断故障层
→ 恢复
→ 验收
```

真正可靠的备份应该同时满足：

```text
知道备份在哪里
知道里面有什么
知道文件没有损坏
知道什么时候该恢复文件
知道什么时候该恢复数据库
知道如何恢复
知道恢复后如何验收
```

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
gunzip
mysqldump
mysql
sha256sum
```

记忆：

> **服务器负责生成和恢复，Windows 负责保存和传输备份。**

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

## 恢复时优先保留旧数据

恢复操作比备份风险更高。

推荐原则：

```text
能改名保留，就先不要删除
能导入新数据库，就先不要覆盖旧数据库
能局部恢复，就先不要整体恢复
```

例如恢复源码时，优先：

```bash
mv public_html public_html-before-restore-YYYY-MM-DD
```

而不是直接：

```bash
rm -rf public_html
```

数据库也优先考虑：

```text
创建新空数据库
→ 导入备份
→ 修改配置文件
```

确认稳定后，再清理旧数据。

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

### 为什么恢复时不要立即删除旧 `public_html`？

因为一旦恢复包、数据库或配置判断错误，旧站文件就是最后的回退机会。

优先改名保留，确认恢复版本正常后再删除。

### 为什么网站坏了不能直接点 `Restore All`？

因为网站文件和数据库的更新时间可能不同。

例如只是主题 PHP 文件改坏，但今天数据库中已经新增询盘，如果整体回退到昨天，文件虽然恢复了，今天新增的数据也可能一起被数据库回退。

因此先判断故障层，再选择恢复范围。

---

## 安全规范

标准教程统一使用：

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

实操记录可以保留便于复盘的域名、端口、目录、日期和文件名，但不要提交：

```text
SSH Private Key
SSH Passphrase
数据库密码
WordPress 管理员密码
API Key
SMTP 密码
其他仍然有效的认证秘密
```

如果历史提交曾经包含真实密码或密钥，不要只修改 README；还应及时轮换对应凭证，并根据需要清理 Git 历史。

---

## 仓库结构

```text
siteground-ssh-wordpress-guide/
├── README.md
├── docs/
│   ├── ssh-wordpress-install.md
│   ├── siteground-backup-guide.md
│   ├── siteground-restore-guide.md
│   └── verified-command-log.md
└── screenshots/
    └── siteground-backup-restore.png
```

当前结构刻意保持简单：

```text
README.md       → 总览与导航
docs/           → 详细运维文档
screenshots/    → 与文档直接相关的实操截图
```

后续如果内容继续增加，可以继续拆分：

```text
docs/
├── ssh-wordpress-install.md
├── siteground-backup-guide.md
├── siteground-restore-guide.md
├── verified-command-log.md
├── common-errors.md
└── command-cheatsheet.md
```

这样比把所有内容长期堆在一个超长 README 中更容易维护。

---

## 后续方向

在手工备份和恢复流程稳定之后，可以再做 PowerShell 自动化：

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

恢复流程暂时建议保持手工执行，因为恢复涉及覆盖数据，风险明显高于备份自动化。

---

## 最终原则

```text
平台自动备份负责快速恢复
手动备份负责关键节点
本地完整备份负责灾备与迁移
先判断故障层，再选择最小恢复范围
恢复流程负责真正闭环
实操记录负责以后快速复盘
```

以及：

> **备份的价值不在于“文件已经生成”，而在于“能够验证、能够下载、能够找到、必要时能够恢复”。**

> **恢复的核心不是“恢复得越多越保险”，而是“准确判断故障层，只恢复真正需要恢复的数据”。**

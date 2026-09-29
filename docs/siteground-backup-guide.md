# SiteGround 网站完整备份指南

本文记录一套已经实际验证过的 SiteGround 网站备份流程，适用于 WordPress、PHP + MySQL 与 PHP + SQLite 项目。

## 一、备份分层

建议把备份分成三层：

| 层级 | 方式 | 主要用途 |
|---|---|---|
| 第一层 | SiteGround 自动备份 | 日常故障恢复 |
| 第二层 | SiteGround 手动备份 | 重大修改前建立恢复点 |
| 第三层 | SSH + 本地独立备份 | 异地灾备、迁移、长期归档 |

核心区别：SiteGround 内置备份主要用于平台内恢复；本地独立备份是真正由自己持有的文件。

## 二、完整备份包含什么

### WordPress / PHP + MySQL

完整备份通常包括：

```text
网站文件
+
MySQL 数据库
```

推荐最终保存：

```text
<site>-files-YYYY-MM-DD.tar.gz
<site>-database-YYYY-MM-DD.sql.gz
```

### PHP + SQLite

SQLite 数据库本身就是文件。如果 SQLite 文件位于 `public_html` 内，那么整站打包时通常已经一起包含数据库。

## 三、推荐本地目录

```text
D:\Website-Backups\
└── example.com\
    └── YYYY-MM-DD\
        ├── example-com-files-YYYY-MM-DD.tar.gz
        └── example-com-database-YYYY-MM-DD.sql.gz
```

## 四、先确认命令环境

看到：

```text
PS C:\Users\Administrator>
```

说明当前在 Windows PowerShell。

看到类似：

```text
user@server:~$
```

说明当前在 SiteGround Linux Shell。

记忆原则：

> 服务器负责生成备份，Windows 负责把备份下载回来。

## 五、定位网站目录

登录 SSH 后：

```bash
pwd
ls -la
cd ~/www
ls -la
```

网站通常位于：

```text
~/www/<domain>/public_html
```

WordPress 典型结构：

```text
wp-admin
wp-content
wp-includes
wp-config.php
```

自定义 PHP 项目可能看到：

```text
admin
content
includes
config.php
index.php
```

## 六、查看网站大小

```bash
cd ~/www/<domain>/public_html
du -sh .
du -sh * 2>/dev/null
```

用于判断网站总大小以及主要空间占用。

## 七、打包网站文件

先进入站点目录上一级：

```bash
cd ~/www/<domain>
```

打包：

```bash
tar -czf ~/tmp/<site>-files-YYYY-MM-DD.tar.gz public_html
```

检查文件：

```bash
ls -lh ~/tmp/<site>-files-YYYY-MM-DD.tar.gz
```

不要把完整备份放在 `public_html`，避免通过 Web URL 暴露备份文件。

## 八、MySQL 数据库备份

WordPress 通常从 `wp-config.php` 查数据库信息；自定义 PHP 项目通常从 `config.php` 查。

需要确认：

```text
DB_HOST
DB_NAME
DB_USER
DB_PASSWORD
```

不要把真实数据库密码提交到 GitHub。

导出示例：

```bash
mysqldump -h localhost -u <DB_USER> -p --single-transaction --quick <DB_NAME> > ~/tmp/<site>-database-YYYY-MM-DD.sql
```

执行后再输入数据库密码，避免把密码直接写进命令历史。

检查：

```bash
head -n 20 ~/tmp/<site>-database-YYYY-MM-DD.sql
tail -n 20 ~/tmp/<site>-database-YYYY-MM-DD.sql
```

统计表数量：

```bash
grep -c "Table structure for table" ~/tmp/<site>-database-YYYY-MM-DD.sql
```

压缩：

```bash
gzip ~/tmp/<site>-database-YYYY-MM-DD.sql
gzip -t ~/tmp/<site>-database-YYYY-MM-DD.sql.gz
```

## 九、下载到 Windows

回到 Windows PowerShell 后，先创建目录：

```powershell
New-Item -ItemType Directory -Force "D:\Website-Backups\<domain>\YYYY-MM-DD"
```

源码：

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P <SSH_PORT> <SSH_USER>@<SSH_HOST>:/home/<SSH_USER>/tmp/<site>-files-YYYY-MM-DD.tar.gz "D:\Website-Backups\<domain>\YYYY-MM-DD\"
```

数据库：

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P <SSH_PORT> <SSH_USER>@<SSH_HOST>:/home/<SSH_USER>/tmp/<site>-database-YYYY-MM-DD.sql.gz "D:\Website-Backups\<domain>\YYYY-MM-DD\"
```

注意：SSH 指定端口通常使用小写 `-p`，SCP 使用大写 `-P`。

## 十、本地验证

查看源码压缩包：

```powershell
tar -tzf "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-files-YYYY-MM-DD.tar.gz" | Select-Object -First 20
```

如果能正常列出 `public_html` 内文件，说明压缩包可读取。

## 十一、SHA256 完整性校验

服务器端：

```bash
sha256sum ~/tmp/<site>-files-YYYY-MM-DD.tar.gz
sha256sum ~/tmp/<site>-database-YYYY-MM-DD.sql.gz
```

Windows：

```powershell
Get-FileHash "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-files-YYYY-MM-DD.tar.gz" -Algorithm SHA256
Get-FileHash "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-database-YYYY-MM-DD.sql.gz" -Algorithm SHA256
```

两端 SHA256 一致，说明传输后的文件与服务器原文件一致。

## 十二、正确的操作顺序

```text
1. 打包网站文件
2. 导出数据库
3. 服务器端验证
4. 服务器端计算 SHA256
5. SCP 下载
6. Windows 本地验证
7. Windows 计算 SHA256
8. 对比一致
9. 最后清理服务器端自己生成的临时备份
```

不要提前删除服务器临时文件，否则无法继续做服务器端完整性对比。

## 十三、MySQL 与 SQLite 的区别

MySQL 数据通常独立于网站目录，因此完整备份通常是：

```text
源码 tar.gz + 数据库 sql.gz
```

SQLite 是文件数据库。如果数据库文件位于 `public_html` 内，整站打包通常已经包含它。

## 十四、常见错误

### `Permission denied (publickey)`

通常表示 SSH 私钥认证失败。检查 SiteGround 是否创建或导入了 SSH Key、本地 Private Key 是否正确，以及 SSH 是否指定了正确私钥。

### 在 Linux Shell 中执行 PowerShell 命令

`$env:USERPROFILE` 属于 PowerShell 变量，不能在 Linux Bash 中使用。需要先退出 SSH 回到 `PS C:\...>` 再执行 SCP。

### 在 PowerShell 中执行 Linux 路径

PowerShell 中的 `rm` 是 `Remove-Item` 别名，并不会删除 SiteGround 上的文件。删除服务器临时文件前必须先 SSH 登录服务器。

## 十五、安全原则

公开仓库中不要出现真实的：

- SSH Private Key；
- Passphrase；
- 数据库密码；
- WordPress 管理员密码；
- API Key；
- SMTP 密码。

文档统一使用：

```text
<SSH_HOST>
<SSH_USER>
<SSH_PORT>
<DB_NAME>
<DB_USER>
<domain>
<site>
YYYY-MM-DD
```

## 十六、推荐长期工作流

```text
日常：SiteGround 自动备份
重大修改前：SiteGround 手动备份
定期或重要节点：SSH 完整独立备份到本地
```

以后可以进一步自动化为 PowerShell 脚本，但建议先把手工流程完全跑通、验证后再自动化。

---

## 总结

对于 WordPress / PHP + MySQL：

```text
完整备份 = 网站文件 + MySQL 数据库
```

对于 PHP + SQLite：

```text
完整备份 = 网站文件（其中包含 SQLite 数据库文件）
```

最终原则：

> 备份的价值不在于“文件已经生成”，而在于“能够验证、能够下载、能够找到、必要时能够恢复”。

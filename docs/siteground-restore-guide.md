# SiteGround 网站恢复指南

本文是 `SiteGround 网站完整备份指南` 的反向流程，目标是把已经保存到本地的：

```text
<site>-files-YYYY-MM-DD.tar.gz
<site>-database-YYYY-MM-DD.sql.gz
```

恢复到 SiteGround。

适用于：

- WordPress；
- PHP + MySQL；
- PHP + SQLite；
- 其他部署在 `public_html` 中的常规 PHP 项目。

> 原则：恢复操作比备份操作风险更高。正式恢复前，先确认当前站点是否还需要保留，并优先再做一次现状备份。

---

## 1. 恢复前先判断场景

常见恢复场景有三类：

| 场景 | 建议 |
|---|---|
| 原站还能访问，只是代码改坏 | 优先恢复文件或 SiteGround 恢复点 |
| 网站文件和数据库都损坏 | 做完整恢复 |
| 迁移到新的 SiteGround 站点 | 按完整恢复流程执行 |

如果只是某个插件、主题或单个文件出错，不建议直接覆盖整站。

---

## 2. 恢复前必须确认的内容

先确认本地备份目录中有：

```text
D:\Website-Backups\example.com\YYYY-MM-DD\
├── example-com-files-YYYY-MM-DD.tar.gz
└── example-com-database-YYYY-MM-DD.sql.gz
```

并确保：

- `tar.gz` 可以正常列出文件；
- `sql.gz` 已经通过 gzip 验证或曾经确认可读；
- 如果之前做过 SHA256，值已经核对一致；
- 已经确认要恢复到哪个域名和哪个 SiteGround 站点。

---

## 3. 强烈建议：恢复前再做一次当前状态备份

即使当前网站已经有问题，也建议先留一份现状快照。

这样如果恢复后发现：

- 选错备份日期；
- 旧数据覆盖了新数据；
- 配置不兼容；
- 域名或数据库信息不同；

仍然有回退空间。

推荐命名：

```text
manual-before-restore-YYYY-MM-DD
```

或者使用 SSH 再备份一份当前 `public_html` 和数据库。

---

## 4. 两种恢复方法

### 方法 A：SiteGround 后台恢复

如果目标只是回到 SiteGround 的自动备份或手动备份点，优先使用：

```text
Site Tools
→ Security
→ Backups
→ Restore
```

优点：简单、快、风险低。

### 方法 B：使用自己的本地备份恢复

本文重点讲这一种：

```text
Windows 本地备份
→ SCP 上传到 SiteGround
→ 解压源码
→ 导入数据库
→ 修改配置
→ 验证网站
```

---

## 5. 先上传备份到服务器临时目录

在 Windows PowerShell 中执行：

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P <SSH_PORT> "D:\Website-Backups\example.com\YYYY-MM-DD\example-com-files-YYYY-MM-DD.tar.gz" <SSH_USER>@<SSH_HOST>:/home/<SSH_USER>/tmp/
```

数据库：

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P <SSH_PORT> "D:\Website-Backups\example.com\YYYY-MM-DD\example-com-database-YYYY-MM-DD.sql.gz" <SSH_USER>@<SSH_HOST>:/home/<SSH_USER>/tmp/
```

注意：

```text
Windows 本地路径放前面
SiteGround 远程路径放后面
```

因为这次是“上传”，方向与备份下载相反。

---

## 6. 登录 SiteGround 并确认文件

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground" -p <SSH_PORT> <SSH_USER>@<SSH_HOST>
```

服务器端检查：

```bash
ls -lh ~/tmp
```

应该能看到：

```text
example-com-files-YYYY-MM-DD.tar.gz
example-com-database-YYYY-MM-DD.sql.gz
```

---

## 7. 恢复源码前先检查压缩包

先不要直接覆盖 `public_html`。

查看内容：

```bash
tar -tzf ~/tmp/example-com-files-YYYY-MM-DD.tar.gz | head -n 20
```

正常应看到：

```text
public_html/
public_html/index.php
...
```

如果是 WordPress：

```text
public_html/wp-admin/
public_html/wp-content/
public_html/wp-includes/
```

确认结构正确后再继续。

---

## 8. 推荐先把现有 public_html 改名

这是比直接删除更安全的做法。

进入站点目录：

```bash
cd ~/www/<domain>
```

查看：

```bash
ls -la
```

如果存在：

```text
public_html
```

先改名：

```bash
mv public_html public_html-before-restore-YYYY-MM-DD
```

这样旧站文件暂时还在，不会立即丢失。

> 不要一开始就执行 `rm -rf public_html`。

---

## 9. 解压网站文件

仍在：

```text
~/www/<domain>
```

执行：

```bash
tar -xzf ~/tmp/example-com-files-YYYY-MM-DD.tar.gz
```

由于压缩包里已经包含 `public_html/`，解压后会重新生成：

```text
~/www/<domain>/public_html
```

检查：

```bash
ls -la public_html
```

WordPress 应看到：

```text
wp-admin
wp-content
wp-includes
wp-config.php
```

自定义 PHP 项目则检查其 `index.php`、`config.php` 等文件。

---

## 10. PHP + SQLite 项目的恢复

如果项目使用 SQLite，并且数据库文件本来就在 `public_html` 内，那么通常：

```text
解压网站文件
=
源码恢复 + SQLite 数据库恢复
```

建议确认 SQLite 文件存在：

```bash
find public_html -type f \( -name "*.sqlite" -o -name "*.db" \)
```

如果 SQLite 文件放在 `public_html` 之外，则需要单独恢复对应目录。

---

## 11. MySQL 数据库恢复前先确认目标数据库

不要直接导入到不确定的数据库。

确认：

```text
DB_HOST
DB_NAME
DB_USER
DB_PASSWORD
```

WordPress 查看：

```text
wp-config.php
```

自定义 PHP 查看：

```text
config.php
```

如果是迁移到新环境，数据库名、用户名、密码可能与旧环境不同。

---

## 12. 解压 SQL 备份

服务器端：

```bash
gunzip ~/tmp/example-com-database-YYYY-MM-DD.sql.gz
```

得到：

```text
~/tmp/example-com-database-YYYY-MM-DD.sql
```

先检查：

```bash
head -n 20 ~/tmp/example-com-database-YYYY-MM-DD.sql
```

以及：

```bash
tail -n 20 ~/tmp/example-com-database-YYYY-MM-DD.sql
```

确认确实是目标数据库备份。

---

## 13. 导入 MySQL

推荐命令：

```bash
mysql -h localhost -u <DB_USER> -p <DB_NAME> < ~/tmp/example-com-database-YYYY-MM-DD.sql
```

执行后会提示：

```text
Enter password:
```

输入数据库密码。

不要把真实密码直接写进命令。

---

## 14. 数据库导入前是否要清空旧表

这是恢复流程中风险最高的环节之一。

### 情况 A：目标数据库是空数据库

可以直接导入。

### 情况 B：目标数据库中已有旧表

先判断是否真的需要覆盖。

推荐优先：

```text
创建一个新的空数据库
→ 导入备份
→ 修改配置文件指向新数据库
```

这样风险比直接 DROP 原数据库表低。

如果确实需要覆盖原数据库，应先再备份一次当前数据库。

---

## 15. WordPress 迁移后需要检查的配置

如果恢复到原域名、原环境，通常 `wp-config.php` 可以继续使用。

如果迁移到新环境，需要检查：

```text
DB_NAME
DB_USER
DB_PASSWORD
DB_HOST
```

如果域名发生变化，还需要检查：

```text
siteurl
home
```

可使用 WP-CLI：

```bash
wp option get siteurl
wp option get home
```

必要时：

```bash
wp option update siteurl 'https://new-example.com'
wp option update home 'https://new-example.com'
```

---

## 16. WordPress 域名变化时的数据替换

如果只是恢复到原域名，通常不需要做 search-replace。

如果从：

```text
old-example.com
```

迁移到：

```text
new-example.com
```

可以先预览：

```bash
wp search-replace 'https://old-example.com' 'https://new-example.com' --all-tables --dry-run
```

确认无误后再执行正式替换：

```bash
wp search-replace 'https://old-example.com' 'https://new-example.com' --all-tables
```

不要直接用普通文本编辑器批量替换 WordPress 数据库 SQL，因为序列化数据可能被破坏。

---

## 17. 恢复后的网站检查

至少检查：

```text
首页
后台登录
文章 / 产品详情
图片与上传文件
表单
自定义插件
Elementor 页面
固定链接
后台保存
数据库写入
```

WordPress 可以运行：

```bash
wp core version
wp plugin list
wp theme list
wp option get siteurl
wp option get home
```

---

## 18. 权限问题检查

如果恢复后出现：

```text
403
500
无法上传文件
无法更新插件
```

可能需要检查文件权限。

先查看：

```bash
ls -la public_html
```

不要盲目执行：

```bash
chmod -R 777
```

这会造成严重安全风险。

如果权限异常，优先参考 SiteGround 当前文件权限规则或使用其后台修复工具。

---

## 19. 恢复成功后不要立即删除旧目录

推荐先保留：

```text
public_html-before-restore-YYYY-MM-DD
```

等网站确认稳定后，再决定删除。

可以先比较：

```bash
du -sh public_html
du -sh public_html-before-restore-YYYY-MM-DD
```

确认恢复站正常运行一段时间后，再清理旧目录。

---

## 20. 清理临时恢复文件

确认恢复成功以后，删除：

```bash
rm ~/tmp/example-com-files-YYYY-MM-DD.tar.gz
rm ~/tmp/example-com-database-YYYY-MM-DD.sql
```

如果数据库压缩包尚未解压：

```bash
rm ~/tmp/example-com-database-YYYY-MM-DD.sql.gz
```

只删除自己创建的恢复文件，不要清空整个 `~/tmp`。

---

## 21. 推荐的完整恢复顺序

```text
1. 确认备份日期和目标网站
2. 再备份当前状态
3. 上传 tar.gz / sql.gz 到 ~/tmp
4. 检查备份内容
5. 将旧 public_html 改名保留
6. 解压网站文件
7. 确认配置文件
8. 准备目标数据库
9. 解压 SQL
10. 导入 MySQL
11. 检查 WordPress / PHP 配置
12. 检查域名与 URL
13. 前后台全面测试
14. 稳定后清理旧目录和临时文件
```

---

## 22. 不同项目的恢复区别

| 项目类型 | 文件恢复 | 数据恢复 |
|---|---|---|
| WordPress + MySQL | 解压 `public_html` | 导入 `.sql` |
| PHP + MySQL | 解压 `public_html` | 导入 `.sql` |
| PHP + SQLite | 解压 `public_html` | SQLite 通常已包含 |
| 静态 HTML | 解压文件 | 无数据库 |

---

## 23. 最危险的几个操作

恢复时尤其要谨慎：

```text
rm -rf
DROP DATABASE
DROP TABLE
直接覆盖 public_html
导错数据库
把旧域名替换错
把生产环境和测试环境混淆
```

推荐原则：

> 能改名保留，就先不要删除；能导入新数据库，就先不要覆盖旧数据库。

---

## 24. 恢复后的最终验收

恢复完成不能只看“首页能打开”。

推荐验收：

```text
文件层：源码、uploads、配置文件完整
数据库层：表和数据正常
应用层：登录、保存、查询、表单正常
URL 层：域名、固定链接、资源地址正常
SEO 层：Canonical、robots、Sitemap 没有异常
安全层：没有留下公开备份包和临时 SQL
```

---

## 总结

备份和恢复应该形成一个闭环：

```text
备份
→ 验证
→ 下载
→ 保存
→ 恢复
→ 验收
```

真正可靠的备份，不是“有一个压缩包”，而是：

> 知道它里面有什么、知道它是否完整、知道如何恢复、并且实际验证过恢复流程。

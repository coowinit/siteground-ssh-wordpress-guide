# SiteGround 实操命令对照记录

这份文档用于配合 `siteground-backup-guide.md` 阅读。

目标是把“推荐通用命令”和“本次实际执行时的命令形式”放在一起对照，方便以后回看时快速理解每个占位符应该替换成什么。

> 说明：为避免公开仓库直接暴露服务器账号、数据库账号等基础设施标识，真实密码、Private Key、Passphrase、SSH Username、数据库名与数据库用户名不写入仓库。域名、日期、本地路径和命令结构保留实际测试形式。

---

## 1. SSH 登录

### 推荐通用命令

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground" -p <SSH_PORT> <SSH_USER>@<SSH_HOST>
```

### 本次实操对照

```powershell
ssh -i "$env:USERPROFILE\.ssh\siteground" -p 18765 <SSH_USER>@ssh.coowin.top
```

更稳妥的形式：

```powershell
ssh -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\siteground" -p 18765 <SSH_USER>@ssh.coowin.top
```

---

## 2. 定位网站目录

### 推荐通用命令

```bash
pwd
ls -la
cd ~/www
ls -la
cd <domain>
ls -la
cd public_html
pwd
ls -la
```

### 本次实操对照

```bash
pwd
ls -la
cd ~/www
ls -la
cd coowin.top
ls -la
cd public_html
pwd
ls -la
```

本次确认的网站路径结构：

```text
/home/<SSH_USER>/www/coowin.top/public_html
```

---

## 3. 查看网站大小

### 推荐通用命令

```bash
cd ~/www/<domain>/public_html
du -sh .
du -sh * 2>/dev/null
```

### 本次实操对照

```bash
cd ~/www/coowin.top/public_html
du -sh .
```

本次结果约为：

```text
43M    .
```

---

## 4. 打包源码

### 推荐通用命令

```bash
cd ~/www/<domain>
tar -czf ~/tmp/<site>-files-YYYY-MM-DD.tar.gz public_html
ls -lh ~/tmp/<site>-files-YYYY-MM-DD.tar.gz
```

### 本次实操对照

```bash
cd ~/www/coowin.top
tar -czf ~/tmp/coowin-top-files-2026-09-29.tar.gz public_html
ls -lh ~/tmp/coowin-top-files-2026-09-29.tar.gz
```

本次原目录约 43MB，压缩后约 14MB。

---

## 5. MySQL 导出

### 推荐通用命令

```bash
mysqldump -h localhost -u <DB_USER> -p --single-transaction --quick <DB_NAME> > ~/tmp/<site>-database-YYYY-MM-DD.sql
```

### 本次实操对照

```bash
mysqldump -h localhost -u <DB_USER> -p --single-transaction --quick <DB_NAME> > ~/tmp/coowin-top-database-2026-09-29.sql
```

密码在出现下面提示后手工输入：

```text
Enter password:
```

不要把密码直接写在命令中。

---

## 6. 检查 MySQL 导出

### 推荐通用命令

```bash
ls -lh ~/tmp/<site>-database-YYYY-MM-DD.sql
head -n 20 ~/tmp/<site>-database-YYYY-MM-DD.sql
tail -n 20 ~/tmp/<site>-database-YYYY-MM-DD.sql
grep -c "Table structure for table" ~/tmp/<site>-database-YYYY-MM-DD.sql
```

### 本次实操对照

```bash
ls -lh ~/tmp/coowin-top-database-2026-09-29.sql
head -n 20 ~/tmp/coowin-top-database-2026-09-29.sql
tail -n 20 ~/tmp/coowin-top-database-2026-09-29.sql
grep -c "Table structure for table" ~/tmp/coowin-top-database-2026-09-29.sql
```

本次测试数据库只有 1 张表，`grep -c` 返回：

```text
1
```

---

## 7. 压缩数据库

### 推荐通用命令

```bash
gzip ~/tmp/<site>-database-YYYY-MM-DD.sql
gzip -t ~/tmp/<site>-database-YYYY-MM-DD.sql.gz
echo $?
```

### 本次实操对照

```bash
gzip ~/tmp/coowin-top-database-2026-09-29.sql
gzip -t ~/tmp/coowin-top-database-2026-09-29.sql.gz
echo $?
```

本次返回：

```text
0
```

表示 gzip 验证通过。

---

## 8. Windows 创建本地目录

### 推荐通用命令

```powershell
New-Item -ItemType Directory -Force "D:\Website-Backups\<domain>\YYYY-MM-DD"
```

### 本次实操对照

```powershell
New-Item -ItemType Directory -Force "D:\Website-Backups\coowin.top\2026-09-29"
```

---

## 9. SCP 下载源码

### 推荐通用命令

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P <SSH_PORT> <SSH_USER>@<SSH_HOST>:/home/<SSH_USER>/tmp/<site>-files-YYYY-MM-DD.tar.gz "D:\Website-Backups\<domain>\YYYY-MM-DD\"
```

### 本次实操对照

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P 18765 <SSH_USER>@ssh.coowin.top:/home/<SSH_USER>/tmp/coowin-top-files-2026-09-29.tar.gz "D:\Website-Backups\coowin.top\2026-09-29\"
```

---

## 10. SCP 下载数据库

### 推荐通用命令

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P <SSH_PORT> <SSH_USER>@<SSH_HOST>:/home/<SSH_USER>/tmp/<site>-database-YYYY-MM-DD.sql.gz "D:\Website-Backups\<domain>\YYYY-MM-DD\"
```

### 本次实操对照

```powershell
scp -i "$env:USERPROFILE\.ssh\siteground" -P 18765 <SSH_USER>@ssh.coowin.top:/home/<SSH_USER>/tmp/coowin-top-database-2026-09-29.sql.gz "D:\Website-Backups\coowin.top\2026-09-29\"
```

注意：

```text
ssh 使用小写 -p
scp 使用大写 -P
```

---

## 11. 本地验证源码

### 推荐通用命令

```powershell
Get-Item "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-files-YYYY-MM-DD.tar.gz"

tar -tzf "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-files-YYYY-MM-DD.tar.gz" | Select-Object -First 20
```

### 本次实操对照

```powershell
Get-Item "D:\Website-Backups\coowin.top\2026-09-29\coowin-top-files-2026-09-29.tar.gz"

tar -tzf "D:\Website-Backups\coowin.top\2026-09-29\coowin-top-files-2026-09-29.tar.gz" | Select-Object -First 20
```

本次实际能够正常列出 `public_html/` 下的文件，说明压缩包可读取。

---

## 12. 查看本地备份目录

```powershell
Get-ChildItem "D:\Website-Backups\coowin.top\2026-09-29"
```

本次最终得到：

```text
coowin-top-files-2026-09-29.tar.gz
coowin-top-database-2026-09-29.sql.gz
```

---

## 13. SHA256 校验

### 服务器端通用命令

```bash
sha256sum ~/tmp/<site>-files-YYYY-MM-DD.tar.gz
sha256sum ~/tmp/<site>-database-YYYY-MM-DD.sql.gz
```

### 本次源码校验命令

```bash
sha256sum ~/tmp/coowin-top-files-2026-09-29.tar.gz
```

### Windows 通用命令

```powershell
Get-FileHash "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-files-YYYY-MM-DD.tar.gz" -Algorithm SHA256
Get-FileHash "D:\Website-Backups\<domain>\YYYY-MM-DD\<site>-database-YYYY-MM-DD.sql.gz" -Algorithm SHA256
```

### 本次实操对照

```powershell
Get-FileHash "D:\Website-Backups\coowin.top\2026-09-29\coowin-top-files-2026-09-29.tar.gz" -Algorithm SHA256

Get-FileHash "D:\Website-Backups\coowin.top\2026-09-29\coowin-top-database-2026-09-29.sql.gz" -Algorithm SHA256
```

本次源码服务器端与 Windows 本地 SHA256 完全一致，因此源码下载完整性验证通过。

数据库文件因为服务器临时文件在最终对比前已经删除，所以没有完成服务器端与 Windows 的最终 SHA256 对比；但服务器端已经通过 `gzip -t` 验证。正式流程应先完成双方 SHA256，再删除临时文件。

---

## 14. 清理服务器临时文件

### 推荐通用命令

```bash
rm ~/tmp/<site>-files-YYYY-MM-DD.tar.gz
rm ~/tmp/<site>-database-YYYY-MM-DD.sql.gz
ls -lh ~/tmp
```

### 本次实操对照

```bash
rm ~/tmp/coowin-top-files-2026-09-29.tar.gz
rm ~/tmp/coowin-top-database-2026-09-29.sql.gz
ls -lh ~/tmp
```

只删除自己创建的备份文件，不要删除 `sess_*`、`mysql.sock` 等服务器运行文件。

---

## 15. 本次最重要的两个错误示例

### 已在 SiteGround，却再次执行 Windows SSH/SCP 命令

如果提示符已经类似：

```text
user@server:~$
```

就已经在 Linux 服务器里了，不要再执行带 `$env:USERPROFILE` 的 PowerShell 命令。

需要先：

```bash
exit
```

回到：

```text
PS C:\Users\Administrator>
```

### 在 Windows PowerShell 里执行服务器 `rm`

在 Windows 中：

```powershell
rm ~/tmp/xxx.tar.gz
```

会被 PowerShell 当成 `Remove-Item`，删除的是本机路径，而不是 SiteGround 文件。

必须先 SSH 登录服务器，再执行：

```bash
rm ~/tmp/xxx.tar.gz
```

---

## 16. 建议的最终顺序

```text
生成源码备份
→ 导出 MySQL
→ gzip 验证
→ 服务器计算 SHA256
→ SCP 下载
→ 本地读取测试
→ 本地计算 SHA256
→ 双端对比
→ 最后删除服务器临时文件
```

这份命令记录的作用不是替代通用教程，而是提供一套真实操作参照。以后更换域名、账号、数据库时，只需要把对应位置替换掉即可。

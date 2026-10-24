# SELinux（上）：模式、上下文、restorecon、semanage fcontext

- 日期：2026-10-24（W4-D4）
- 考点：RHCSA EX200 —— 设置 enforcing/permissive 模式；列出和识别文件、进程上下文；恢复默认文件上下文；管理 SELinux 文件上下文；诊断 SELinux 拒绝
- 环境：Rocky Linux 10.2（VirtualBox），node1（httpd，DocumentRoot `/data/www`）、node2 客户端
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

理解 DAC 与 MAC 的区别、type 规则、文件标签从哪来；复现 mv 导致的 403，用 ausearch 读 AVC，用 permissive 确认原因，用 semanage fcontext + restorecon 正式给新目录打标签。

## 关键概念

- DAC（rwx，属主说了算）+ MAC（策略说了算，默认拒绝）两道关都要过；SELinux 防的是程序被攻破后越界。
- 上下文 `user:role:type:level`，targeted 策略基本只看 **type**：进程 `httpd_t` 允许读文件 `httpd_sys_content_t`。自己的 shell 是 `unconfined_t`。
- 模式：Enforcing（拦截+记录）、Permissive（只记录）、Disabled。`setenforce 0/1` 只改运行时；`/etc/selinux/config` 重启后生效。
- 文件标签存在 inode 的扩展属性里（在磁盘上）。新建/cp 的文件继承目标目录的类型；**mv 保留原标签**（inode 不变）。
- 规则库（fcontext）是"应该是什么"，文件上的标签是"实际是什么"。`restorecon` 按规则库重打；`chcon` 只改实际标签，restorecon / 全盘 relabel 后被改回。正式做法：`semanage fcontext -a` + `restorecon`。
- `semanage fcontext -l -C` 只列本机加的规则：node1 上已有 `/data/www(/.*)?`（8 月自学时加的）。
- restorecon 默认只改 type，不改 user 段（所以显示 `unconfined_u:...:httpd_sys_content_t`）；`-F` 才全改。
- **AVC = Access Vector Cache**：内核里缓存"某类型对某类型某类对象有哪些权限"判定结果的组件，拒绝记录以它命名。日志在 `/var/log/audit/audit.log`，`ausearch -m AVC -ts recent`。
- 读 AVC：`{ read }` 动作、`scontext` 谁、`tcontext` 访问的对象、`tclass` 对象类别、`permissive=0/1` 当时是否真拦。
- Enforcing 下第一个被拒的操作就失败（`success=no exit=-13` = EACCES），只记一条；Permissive 下操作继续，`open`、`read`、`map` 每一步都记一条（`permissive=1`，`success=yes`）。
- 403 只看客户端分不出是 rwx 还是 SELinux → 服务端 ausearch。

## 关键步骤

```bash
# 现状
getenforce; sestatus; id -Z; ps -eZ | grep httpd | head -3
ls -Zd /var/www/html /data/www; ls -Z /data/www
sudo semanage fcontext -l -C

# mv 的坑
sudo mv ~/moved.html /data/www/; sudo cp ~/copied.html /data/www/
ls -Z /data/www/                         # moved 是 user_home_t
sudo ausearch -m AVC -ts recent
sudo restorecon -v /data/www/moved.html

# permissive 确认原因
sudo chcon -t user_home_t /data/www/copied.html
sudo setenforce 0; sudo setenforce 1
sudo restorecon -v /data/www/copied.html

# chcon 是临时的
sudo chcon -R -t httpd_sys_content_t /srv/web
sudo restorecon -Rv /srv/web             # 改回 var_t

# 新目录正式做法（site2.conf 内容：Alias /site2 /srv/web + <Directory /srv/web> Require all granted）
sudo tee /etc/httpd/conf.d/site2.conf
sudo httpd -t && sudo systemctl reload httpd
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
sudo restorecon -Rv /srv/web
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 模式 | `getenforce`、`sestatus` | Enforcing，config 也是 enforcing | ✅ |
| httpd 进程类型 | `ps -eZ` | httpd_t | ✅ |
| /data/www 来源 | `semanage fcontext -l -C` | 有自定义规则 | ✅ `/data/www(/.*)?` |
| mv 保留标签 | `ls -Z` | moved.html 是 user_home_t | ✅ |
| mv 的文件 403、cp 的正常 | node2 `curl` | 403 / copied page | ✅ |
| AVC 记录 | `ausearch` | httpd_t → user_home_t `{ read }` | ✅ |
| restorecon 修复 | node2 `curl` | moved page | ✅ |
| permissive 放行但记录 | `ausearch` | `permissive=1` 的 open/read/map | ✅ |
| chcon 被 restorecon 改回 | `restorecon -Rv` | 回到 var_t | ✅ |
| 新目录被拦 | node2 `curl /site2/`、`ausearch` | 403；httpd_t → var_t `{ getattr }` | ✅ |
| 正式修复 | node2 `curl /site2/` | site2 | ✅ |
| 规则与标签一致 | 再次 `restorecon -Rv` | 无输出 | ✅ |

## 踩坑

- `Requir all granted` 拼错 → `httpd -t` 报 `Invalid command 'Requir'` → 改完再 reload。改配置先 `-t` 检查起了作用。
- `semanage fcontext -a ... "/srv/web(.*)?"` 少了 `/` → 也会匹配 `/srv/website`、`/srv/web-old` 这类路径 → 正确写法 `"/srv/web(/.*)?"`；用 `semanage fcontext -d "/srv/web(.*)?"` 删掉错误规则。
- 实验 3 没单独贴出 permissive 下的 curl 结果，从 ausearch 的 `permissive=1` 记录可以确认当时访问成功了。

## 卡点 / 没想明白的

- AVC 是什么的缩写（Access Vector Cache，见关键概念）。
- 达标题 2 以为 chcon "没写进磁盘所以不持久" → 标签就在磁盘上（inode 扩展属性），重启也还在；不持久是因为**规则库**里没有这条，restorecon 或全盘 relabel 会按规则库改回去。

## 回忆题

1. （概念）为什么 mv 到网站目录的文件会 403，cp 的不会？
<details><summary>答案</summary>

标签存在 inode 上，同一文件系统内 mv 不换 inode，带着原来的 user_home_t；cp 新建文件，继承目标目录的 httpd_sys_content_t。httpd_t 不允许读 user_home_t。
</details>

2. （操作）让 httpd 能读 `/srv/web` 及以下所有文件，重新打标签后也不失效。
<details><summary>答案</summary>

`sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"`；`sudo restorecon -Rv /srv/web`；`ls -Zd /srv/web`。
</details>

3. （排障）网站报 403，rwx 权限检查过没问题，怎么确认是不是 SELinux、确认后怎么处理？
<details><summary>答案</summary>

`sudo ausearch -m AVC -ts recent` 看 httpd_t 对哪个 tcontext 被拒；需要时 `setenforce 0` 复测确认，**立刻 `setenforce 1`**；按原因修：标签用 `restorecon` / `semanage fcontext` + `restorecon`（明天：boolean、端口）。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

# 日志：journald、rsyslog、logrotate

- 日期：2026-10-11（W2-D6）
- 考点：RHCSA EX200 —— 查找并解读系统日志和 journal；保留系统日志（持久化 journal）
- 环境：Rocky Linux 10（VirtualBox）
- 状态：完成（达标检查 7.5/8）
- 来源：根据课程对话整理

## 目标

会用 journalctl 精确过滤，配置 journal 持久化，会写 rsyslog 规则和 logrotate 配置。

## 关键概念

- RHEL 9/10 默认 journald 和 rsyslog **同时运行**：journald 收集（二进制，journalctl 查）→ rsyslog 写文本文件（/var/log/messages、secure、cron）→ logrotate 轮转。
- `/var/log/secure`：SSH 登录、sudo、su，权限 600。
- 优先级 0 emerg … 3 err、4 warning … 7 debug；`-p err` = err **及以上**。
- RHEL 10 默认 journal 只存在 `/run/log/journal`（内存），重启即丢 → `journalctl -b -1` 依赖持久化。
- **Rocky 10 上 `/etc/systemd/journald.conf` 不存在**，默认配置在 `/usr/lib/systemd/journald.conf` → 用 drop-in，老教程的 sed 不适用。
- rsyslog 规则：`设施.级别 目标`，`local0～local7` 留给自定义；`logger -p local0.info "msg"`。
- logrotate：重命名不改 inode，进程还在往旧文件写 → `postrotate` 发信号让程序重新打开日志（Nginx 用 USR1）；不支持时用 `copytruncate`。
- 排障顺序：`systemctl status` → `journalctl -u 服务 -n 50` → 服务自己的日志 → 配置检查工具（`httpd -t`、`nginx -t`、`sshd -t`、`visudo -c`）→ SELinux（audit.log）。

## 关键步骤

```bash
journalctl -u sshd --since today -p warning --no-pager
journalctl -b -1 -n 10                        # 上次开机（需持久化）

# 持久化 journal（RHEL 10 写法）
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\n' | sudo tee /etc/systemd/journald.conf.d/persistent.conf
sudo systemctl restart systemd-journald && sudo journalctl --flush
journalctl --list-boots

# 模拟 httpd 故障
echo "ThisIsNotADirective on" | sudo tee -a /etc/httpd/conf/httpd.conf
sudo httpd -t                                 # 指出出错行

# rsyslog 自定义规则 + logrotate
echo 'local0.*    /var/log/myapp.log' | sudo tee /etc/rsyslog.d/myapp.conf
sudo rsyslogd -N1 && sudo systemctl restart rsyslog
sudo logrotate -d /etc/logrotate.d/myapp      # 先模拟
sudo logrotate -f /etc/logrotate.d/myapp      # 强制轮转

# SSH 暴力破解统计（IP 是倒数第 4 个字段）
sudo grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| journald 配置位置 | `ls /etc/systemd/journald.conf /usr/lib/systemd/journald.conf` | — | ✅ 只有 `/usr/lib/...` 存在 |
| logrotate 强制轮转 | `ls -l /var/log/myapp*` | 新空文件 0640 + 旧文件压缩 | ✅ `myapp.log` 0 字节、`myapp.log.1.gz` |
| 轮转后新日志写入新文件 | `logger` 后 `cat /var/log/myapp.log` | 出现新日志 | 未验证（回复中未贴结果） |

## 踩坑

- `logrotate -d` 显示 `log does not need rotating (log has already been rotated)` → 新日志第一次出现时，logrotate 把"当前时间"记作上次轮转时间，`daily` 不到一天不轮转 → 测试用 `-f`。
- 轮转后是 `.1.gz` 而不是带日期 → 单独跑 `/etc/logrotate.d/myapp` 不会读取全局 `/etc/logrotate.conf` 里的 `dateext` → 完整模拟用 `logrotate -d /etc/logrotate.conf`。
- 热身 drop-in 写成 `http.service.d/override.service` → 目录名少 `d`，后缀必须 `.conf`（错了会被静默忽略）→ 用 `systemctl cat` 验证。
- 热身"60 分钟内"写成 `-mtime -60`（60 天）→ 分钟用 `-mmin`。
- `$(NF-3)` 的原因答成"字段可能为空" → 实际是用户不存在时多出 `invalid user` 两个词，前面字段数变化，行尾结构固定，所以从后往前数。

## 卡点 / 没想明白的

- 刚开始看不懂 logrotate -d 的调试输出（已解释）。

## 回忆题

1. （概念）为什么 RHEL 10 上不建议照搬 `sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf`？
<details><summary>答案</summary>

`/etc/systemd/journald.conf` 默认不存在，默认配置在 `/usr/lib`（改它会被升级覆盖）；用 `/etc/systemd/journald.conf.d/*.conf` drop-in。
</details>

2. （操作）配置 journal 持久化并验证。
<details><summary>答案</summary>

建 `/etc/systemd/journald.conf.d/persistent.conf` 写 `[Journal]` `Storage=persistent`；`systemctl restart systemd-journald`；`journalctl --flush`；重启后 `journalctl --list-boots` 至少两条。
</details>

3. （排障）Nginx 日志轮转后，新的 access.log 一直是空的，旧文件还在变大，为什么？
<details><summary>答案</summary>

重命名不改 inode，Nginx 还在写旧文件；postrotate 没给 Nginx 发 USR1（或 pid 文件路径不对），没让它重新打开日志。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

# 计划任务：cron 与 systemd timer

- 日期：2026-11-04（A1，调整后课程第 1 课）
- 考点：RHCSA EX200 —— 用 cron 和 systemd timer 安排任务；面试高频：cron 表达式、"手动能跑 cron 不行"排查
- 环境：Rocky Linux 10.2（VirtualBox），node1；cronie-1.7.0
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

写对 cron 表达式，亲手踩 cron 的 PATH 和 `%` 两个坑；用 systemd timer 让 inspect.sh 每天 02:00 自动运行，并验证失败时能被看见。

## 关键概念

```
分 时 日 月 星期   命令        （星期 0 和 7 都是周日）
*/5 * * * *        每 5 分钟
30 2 * * *         每天 02:30
30 8 * * 1-5       工作日 08:30
0 0 1 * *          每月 1 号 0 点
```

- 日和星期同时写具体值是"或"：`0 0 13 * 5` = 每月 13 号**或**每个周五。

| 位置 | 编辑 | 格式 |
|---|---|---|
| 用户 crontab（`/var/spool/cron/用户名`，600） | `crontab -e` / `-l`（`-r` 全删，危险） | 时间 + 命令 |
| `/etc/crontab`、`/etc/cron.d/*` | 直接编辑 | 时间 + **用户名** + 命令（系统级要写明以谁运行） |
| `/etc/cron.daily/` 等 | 放可执行脚本 | 由 anacron 决定几点跑，错过会补 |

- cronie 1.7 在 `crontab -e` 保存时把旧版本备份到 `~/.cache/crontab/crontab.bak`。
- **cron 环境 ≠ 终端**：PATH 极简（自己的脚本、/usr/sbin 下的命令找不到）、不读 .bashrc、当前目录是家目录、输出没人接（没邮件服务就丢）、**`%` 是换行**（要写 `\%`）。→ 绝对路径 + `>> 日志 2>&1` + `\%`。
- crontab 里那一行命令由 `/bin/sh` 解释；被调用的脚本仍按自己的 shebang（bash）执行。
- anacron = anachronistic cron（"不合时宜的 cron"）：不假设机器一直开着，开机后补跑错过的 daily/weekly/monthly 任务（`/etc/cron.hourly/0anacron` 每小时检查）。
- systemd timer = `xxx.service`（做什么，`Type=oneshot`）+ `xxx.timer`（什么时候，`OnCalendar=`），enable 的是 **timer**；`Persistent=true` 补跑错过的；`RandomizedDelaySec=` 随机延迟错开。
- timer 的输出自动进 journal；脚本退出码非 0 → service 显示 **failed**，`systemctl --failed` 可见。
- `install -m 755 源 目标`：复制并设置权限（-m = mode）；新建文件 → 得到目标目录的 SELinux 标签（bin_t），系统服务才能执行。

| | cron | systemd timer |
|---|---|---|
| 写法 | 一行 | 两个文件 |
| 输出 | 自己重定向，否则丢 | 自动进 journal |
| 失败 | 看不见 | failed，可告警 |
| 关机错过 | 错过（cron.daily 由 anacron 补） | `Persistent=true` 补跑 |
| 测表达式 | 无 | `systemd-analyze calendar` |

最佳实践：重要任务（备份、巡检）用 timer；简单临时用 cron；cron 格式必须熟（面试和老系统）。

## 关键步骤

```bash
rpm -q cronie; systemctl is-active crond; crontab -l
crontab -e          # * * * * * /usr/bin/date >> /tmp/cron-test.log 2>&1
sudo ls -l /var/spool/cron/

# 坑：相对路径、%
# * * * * * inspect.sh >> /tmp/cron-inspect.log 2>&1                 → command not found
# * * * * * /home/fqh/w5d2/inspect.sh > /tmp/inspect-$(date +%F).txt  → 没生成文件
# 修复：/home/fqh/w5d2/inspect.sh；$(date +\%F)

# timer
sudo install -m 755 ~/w5d2/inspect.sh /usr/local/bin/inspect.sh
# /etc/systemd/system/inspect.service：[Service] Type=oneshot / ExecStart=/usr/local/bin/inspect.sh
# /etc/systemd/system/inspect.timer：[Timer] OnCalendar=*-*-* 02:00:00 / Persistent=true / RandomizedDelaySec=10min；[Install] WantedBy=timers.target
sudo systemctl daemon-reload; sudo systemctl enable --now inspect.timer
systemctl list-timers inspect.timer
sudo systemctl start inspect.service; journalctl -u inspect.service -n 30 --no-pager

# 失败可见
sudo systemctl edit inspect.service     # ExecStart= / ExecStart=/usr/local/bin/inspect.sh 15
sudo systemctl start inspect.service; systemctl --failed
sudo systemctl revert inspect.service; sudo systemctl reset-failed inspect.service

systemd-analyze calendar "Mon..Fri 08:30" --iterations=3
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| cron 每分钟执行 | `cat /tmp/cron-test.log` | 每分钟一行 | ✅ 07:31～07:36 |
| crontab 存放位置 | `sudo ls -l /var/spool/cron/` | fqh，600 | ✅ |
| 相对路径失败 | `cat /tmp/cron-inspect.log` | command not found | ✅ `/bin/sh: inspect.sh: command not found` |
| `%` 未转义 | `ls /tmp/inspect-*` | 没有今天的文件 | ✅ 只有 11-03 的 |
| 修复后 | 日志、文件 | 有巡检报告 | ✅ 07:48 起每分钟一份；11-04 文件生成 |
| 脚本标签 | `ls -Z /usr/local/bin/inspect.sh` | bin_t | ✅ |
| timer 下次执行 | `list-timers` | 明天 02:00～02:10 | ✅ 02:07:08（随机延迟） |
| 手动触发 | `journalctl -u inspect.service` | 完整报告，status=0 | ✅ |
| 失败可见 | 阈值 15 | failed (exit-code)，`--failed` 列出 | ✅ problems: 4 |
| 恢复 | `revert` + `reset-failed` | inactive | ✅ |

## 踩坑

- `OnCalender`（拼错）→ `Unknown key 'OnCalender' ... ignoring` + `Timer unit lacks value setting. Refusing.` → 未知的键被忽略，timer 没有触发条件所以拒绝启动 → 写完先 `systemd-analyze verify 文件` 检查。
- `sudo journalctl -u crond | grep inspect` 什么都没有 → 用户 cron 任务的执行记录不一定归在 crond.service 名下（推测：PAM 会话把任务放进了用户会话的 scope），`-u` 按单元过滤就漏掉了 → 用 `journalctl -t CROND` 或 `/var/log/cron` 查（待验证）。
- 实验前 `crontab -l` 发现 8 月留下的旧任务：`0 8 * * * /home/fqh/inspect.sh >> /home/fqh/inspect_log.txt 2>&1`。
- `systemctl --failed` 末尾的 Legend 就是 `--no-legend` 去掉的说明文字。

## 卡点 / 没想明白的

- anacron 的全称（anachronistic cron）。
- `install -m` 的意思。
- crond 日志在 journal 里按 `-u crond` 查不到 inspect（见踩坑）。
- 面试题 1：`00/5 * * * *` → 标准写法 `*/5`；`0/5` 不是所有 cron 实现都认（推测）。
- 面试题 5：`journalctl --failed` 不存在，应为 `systemctl --failed`；timer 用 `systemctl list-timers` 看 LAST/PASSED。

## 回忆题

1. （操作）写出 cron：每 10 分钟；每天 23:45；每周一和周四 06:00；每月最后……（换成）每月 1 号和 15 号 03:00。
<details><summary>答案</summary>

`*/10 * * * *`；`45 23 * * *`；`0 6 * * 1,4`；`0 3 1,15 * *`
</details>

2. （排障）脚本手动执行正常，放进 crontab 不工作，按什么顺序查？
<details><summary>答案</summary>

先在 crontab 里加 `>> /tmp/x.log 2>&1` 看报错；看 `/var/log/cron` / `journalctl -t CROND` 确认有没有执行；检查：绝对路径（PATH 极简）、`%` 是否转义、脚本 x 权限、脚本里的相对路径和环境变量（不读 .bashrc）、以哪个用户运行。
</details>

3. （概念）备份任务为什么更推荐 systemd timer？要让关机期间错过的任务补跑，要写什么？
<details><summary>答案</summary>

输出自动进 journal、失败显示 failed 可告警、有资源和依赖控制、能用 systemd-analyze calendar 测表达式；`Persistent=true`。
</details>

4. （排障）timer 启动报 `Timer unit lacks value setting`，怎么查？
<details><summary>答案</summary>

`systemctl status xxx.timer` / journal 里会有 `Unknown key ...` → 选项名拼错被忽略，timer 没有任何触发条件；修正后 `daemon-reload`。事先可用 `systemd-analyze verify`。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

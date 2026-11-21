# HA 正常重启验证，RHCSA 模拟②分段

- 日期：2026-11-20（课堂 07:37–10:51）
- 考点：RHCSA EX200 分段练习 —— 用户/组、SGID；systemd、ss、firewalld。Nginx location 与 Keepalived 重启是运维扩展，不是这套模拟的完整闭卷
- 环境：Rocky Linux 10（VirtualBox），node1 192.168.56.11 / node2 192.168.56.12，VIP 192.168.56.50/24，`enp0s8`
- 状态：分段做完。不是限时闭卷整场。ACL 第 2 题已撤出评分，见 [ACL 基础](2026-11-20-acl-basics.md)
- 来源：课堂终端记录。`~/ha-reboot-test.log` 全文没有交来核对

## 目标

用正常 reboot 看两台开机后 nginx、keepalived 会不会自己起来，VIP 会不会回到 node1。后面按模拟②分段做用户组和 Nginx 精确匹配，并核对日志、journal、端口、防火墙。

## 关键概念

- 今天测的是正常 reboot，不是突然断电。
- `enable` 只决定开机启动。这次没有手动 `start`，服务自己是 active，才算开机自启生效。
- 截图里 08:23:15–18 响应 node2，08:23:19 起响应 node1（北京时间）。不能写成请求零丢失，也读不出精确中断时刻。
- `%F` 是年月日，`%T` 是时分秒，`%Z` 是时区缩写。
- `tee` 同时留下 stdout 和 stderr。curl 没加 `-s`，进度表也会进记录，这不是故障。
- `sort -r` 按文字倒序，`sort -nr` 才按数字倒序。`uniq -c` 只合并相邻的相同行，所以前面要先 `sort`。
- `sudo` 只作用在 awk 上，不盖住后面的管道。管道里是 awk 已经打出来的流，`sort` 不必再要 root。给 `sort` 加 `sudo` 解决不了 awk 读日志的权限。
- `journalctl -b` 只看本次启动。`-n 10` 是最多 10 条，不是一定有 10 条。
- `0.0.0.0:8088` 表示所有 IPv4 地址。对方可以用固定 IP 来连，但这不表示防火墙已经放行。只听 `127.0.0.1` 时，对端用固定 IP 连不上。
- 没写 `--zone` 时，firewalld 用默认 zone。这次默认是 public，和 `enp0s8` 所在 zone 相同。运行中的规则和 permanent 可以不一样。`reload` 会丢掉还没存成 permanent 的运行时改动。这条 reload 今天只口答，没有执行。
- `before-rhcsa2` 是两台各自在开机状态下的快照，不是两台同一时刻的一致性快照，也不等于数据库备份。
- 模拟②第 2 题原想单独给 exam02 `rx`。具体语法还没教，当场撤出闭卷评分，改成 ACL 补课。不能把这场写成闭卷通过。
- 求职材料里这是实验项目，不是生产：Nginx 双节点、Rocky、Keepalived；VIP 按优先级；HTTP 健康检查的 fall/rise；SELinux 端口和防火墙；`enable`；服务故障和这次正常 reboot。口答是持续访问 VIP，看到 node1 → node2 → node1。要补上 VIP 当时在哪一台，并用 journal 对照。不写零中断、精确切换时间或生产级。全份简历、找岗位、投递都没有做。

## 关键步骤

开始时两台 `systemctl is-active nginx keepalived` 都是 active/active，`systemctl is-enabled nginx keepalived` 都是 enabled/enabled。node1 的 `enp0s8` 有 192.168.56.11 和 .50，node2 的 `enp0s8` 有 .12 和 .22。

```bash
# node2
sudo reboot
```

从 node1 重新 ssh 到 node2。没有手动 start。nginx 和 keepalived 都是 active。本机 `127.0.0.1:8088` 的正文是 node2。VIP .50 的正文是 node1。node2 上没有 VIP。

Windows 宿主机没有 node2 的授权私钥，node2 又禁止密码 SSH，不能从宿主机直接 ssh。没有改 SSH 策略，也没有传密钥。node1 reboot 时，不能靠「先到 node1 再跳到 node2」的会话看结果，改在 node2 的 VirtualBox 控制台看。

```bash
while true; do
date '+%F %T %Z'
curl --connect-timeout 1 --max-time 2 http://192.168.56.50:8088/
sleep 1
done 2>&1 | tee ~/ha-reboot-test.log
```

Ctrl+C 停掉。node1 起来之后没有手动 start：`systemctl is-active nginx keepalived` 两项都是 active，`enp0s8` 上有 .11 和 .50，VIP 正文回到 node1。

两台在开机状态下各做了快照 `before-rhcsa2`。

```bash
# 模拟②第 1 段。useradd 默认会建家目录，后来看到家目录确实在
sudo useradd exam01
sudo groupadd ops_exam
sudo usermod -aG ops_exam exam01
sudo mkdir -p /srv/ops-exam
sudo chown root:ops_exam /srv/ops-exam
sudo chmod 770 /srv/ops-exam
sudo chmod 2770 /srv/ops-exam
sudo -u exam01 touch /srv/ops-exam/test.txt
```

`id`：exam01 的 uid/gid 都是 3101，附加组 ops_exam 的 gid 是 4001。`passwd` 里家目录 `/home/exam01`，shell `/bin/bash`。家目录 `drwx------ exam01 exam01`。`/srv/ops-exam` 是 `drwxrws--- root ops_exam`。`test.txt` 是 `-rw-r--r-- exam01 ops_exam`，组继承已看到。770 再改 2770，前一次 770 可以不写。

node1 的 `/etc/nginx/conf.d/ha-test.conf`，在原来的 8088 `server` 里加上精确匹配。原来的 `/` 仍返回 node1 的正文。

```nginx
location = /exam-health {
    default_type text/plain;
    return 200 "exam health ok\n";
}
```

```bash
nginx -t && systemctl reload nginx
curl -i http://192.168.56.50:8088/exam-health
```

`/exam-health` 是 200，正文 `exam health ok`。`/exam-health-extra` 是 200，正文仍是原来的 node1。`/` 也返回 node1。

改完之后才做的备份，没有报错。这是改完后的版本，不是改之前的备份。快照没有恢复。

```bash
sudo cp -a /etc/nginx/conf.d/ha-test.conf /etc/nginx/conf.d/ha-test.conf.after-exam-health
```

```bash
# 初稿是最后的 sort -r，得到 3344 200
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -r
# 改为数字倒序后是 3374 200。多出来的 30 行是当时日志还在长
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

多出来的 30 行从哪来，健康检查只是（推测），不能当证据。

```bash
systemctl is-active nginx          # active
systemctl is-enabled nginx         # enabled
journalctl -u nginx -n 10 --no-pager
sudo journalctl -b -u nginx -n 10 --no-pager
sudo ss -lntp 'sport = :8088'
firewall-cmd --get-active-zones
firewall-cmd --list-ports
```

`ss`：`LISTEN 0.0.0.0:8088`，nginx 的 pid 是 41629 和 1689。

`--get-active-zones`：public（default），接口 `enp0s3`、`enp0s8`。`--list-ports`：`8081/tcp` `8082/tcp` `8083/tcp` `8088/tcp` `8090/tcp` `8888/tcp`。

本次启动的 journal 只有 6 条。记下的是 08:22:59 Starting，08:23:01 语法检查和启动成功，09:59:12 Reloading 与 Reloaded。没有把没给出的原文补成第 6 条的全文。

`firewall-cmd --zone=public --query-port=8088/tcp` 是老师举的例子，没有实测，没有 yes/no 输出。`sudo firewall-cmd --reload` 只口答，没有执行。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 重启前 | `systemctl is-active/is-enabled nginx keepalived` | 两台都是 active、enabled | ✅ |
| node2 正常 reboot | `sudo reboot` 后从 node1 ssh 回去，不手动 start | active/active；`127.0.0.1:8088` 是 node2；VIP 是 node1；node2 无 VIP | ✅ |
| node1 正常 reboot | 控制台里的 while/curl；起来后不手动 start | 截图 08:23:15–18 为 node2，08:23:19 起为 node1；之后 active，`enp0s8` 有 .11 和 .50 | ✅。不能写成零丢失或精确中断时刻。全文日志未核对 |
| 突然断电 | | | 未做 |
| SGID | `sudo -u exam01 touch /srv/ops-exam/test.txt` | 属组 ops_exam | ✅ `-rw-r--r-- exam01 ops_exam` |
| `/exam-health` | `curl -i http://192.168.56.50:8088/exam-health` | 200，`exam health ok` | ✅ |
| 非精确路径 | `/exam-health-extra` 与 `/` | 都是原来的 node1 正文 | ✅ `/exam-health-extra` 为 200 |
| 改前备份 | | 修改前先复制配置 | ❌ 忘了。事后那份是改完的版本 |
| 状态码排序 | `sort -nr` | 数字倒序 | ✅ 3374 200。3344 是 `-r` 的文字序 |
| 本次启动日志 | `sudo journalctl -b -u nginx -n 10 --no-pager` | 只有这次 boot | ✅ 6 条，不是凑满 10 条 |
| 8088 监听 | `sudo ss -lntp 'sport = :8088'` | `0.0.0.0:8088` | ✅。不等于防火墙已放行 |
| 8088 是否已放行 | `--query-port=8088/tcp` | | 未实测。`--list-ports` 里有 `8088/tcp` |
| `firewall-cmd --reload` | | | 只口答，未执行 |
| 两台同一时刻一致快照 / 数据库备份 | | | 未做。`before-rhcsa2` 不是这个 |

## 踩坑

- while 初稿每行末尾加了反斜杠 → shell 一直停在 `>`，把下一行当成续行 → 取消后去掉反斜杠重输。
- 记录里有 curl 进度表 → 命令没有 `-s`，进度走 stderr，`2>&1 | tee` 把它和正文一起留下了 → 不是故障，不用改。
- 想从 Windows 宿主机 ssh node2 → 宿主机没有 node2 的授权私钥，node2 禁止密码登录 → 改用 VirtualBox 控制台。没有改 sshd，没有传密钥。
- node1 reboot 时想用经 node1 跳到 node2 的会话看 → node1 自己在重启，这条会话不可用 → 改在 node2 控制台看。
- `return` 末尾漏了分号，当时说成括号错 → 没有 nginx 的报错原文，不能补写 → 补上分号。
- `location` 漏了 `=` → 精确匹配没写上 → 改成 `location = /exam-health`。
- `vim /etc/nginx/conf.d/ha-test/conf` → 路径错了，多了一层目录、文件名也不对 → 改成 `/etc/nginx/conf.d/ha-test.conf`。没有这次误打开被保存或报错的输出。
- `nignx -t` → `command not found` → 改成 `nginx -t && systemctl reload nginx`，成功。
- 改配置前没有备份 → 事后 `sudo cp -a` 到 `ha-test.conf.after-exam-health` 没有报错，那是改完的文件 → 不算补做了改前备份。快照没有恢复。
- `nginx -t` 失败时 → 也可能是文件不存在或权限，不只是语法 → 先看报错，不用 `restart` 碰运气。这次成功的是 `nginx -t && systemctl reload nginx`。
- `sort -r` 得到 3344 200 → 没有 `-n`，倒序按文字而不是数字 → 改成 `sort -nr`，得到 3374 200。
- `journalctl -u nginx -n 10 --no-pager` 带上了上次启动 → 没加 `-b` → 改为 `sudo journalctl -b -u nginx -n 10 --no-pager`。
- `firewall-cmd --active-zone` → unrecognized → `--help` 里看到的是 `--get-active-zones`。
- 把「所有 IPv4」说成固定端口 → 听的是地址不是端口 → 改称固定 IP。

## 卡点 / 没想明白的

- 全文 `~/ha-reboot-test.log` 没有交来。截图只覆盖 08:23:15 前后那几秒。
- `--query-port` 和 `--reload` 没有跑。`--list-ports` 里已经有 `8088/tcp`，不能把它写成 query 的 yes。
- 多出来的 30 条 access log 来源没有证据。
- `curl -i` 是头加正文。`curl -I` 通常发 HEAD，只有头。今天 VIP 的 `/exam-health` 用的是 `curl -i`。
- 讲过的顺序是：先备份，再改，`nginx -t`，reload，然后新旧路径都请求一遍。动手时备份做晚了。

## 面试题

模拟②没有另出一套闭卷 5 题。今天另有分数的是合并复习 5/5（健康检查 2/2、HTTPS 2/2、备份 1/1，不是三篇各 5 分）和 ACL 课后 4.5/5，记在各自笔记里。

## 复习记录

起算日是 11-20。

- [x] D+1（11-21，1/1（合并复习，六题合计 5.5/6，折合 4.58/5，不是 5/5，也不是本篇单独 5 题）；把 `uniq` 说成对相邻相同行「排序」。订正为合并统计。仍给 1 分）
- [ ] D+3
- [ ] D+7

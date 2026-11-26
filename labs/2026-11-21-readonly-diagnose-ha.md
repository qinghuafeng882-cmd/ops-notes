# 只读诊断：Nginx、8088 与 VIP 的正常状态

- 日期：2026-11-21（北京时间 10:06–10:44）
- 考点：运维排障顺序 —— 本机、跨主机固定 IP、VIP。不是 RHCSA 模拟②的完整闭卷
- 环境：Rocky Linux 10.2（VirtualBox），node1 192.168.56.11 / node2 192.168.56.12，用户 fqh
- 状态：只做了正常状态取证。没有故障注入。没有独立 5 题，不记分。简历、JD、投递未做
- 来源：课堂终端记录。归档时没有重跑

## 目标

在两台都正常的时候，把「本机通了」和「固定 IP 通了、VIP 也通了」分开看，不把这次访问写成故障切换。

## 关键概念

- 本机有响应，只能说明这个地址上有服务。它可能只听 127.0.0.1。不能因此说整台 nginx 没问题，也不能说服务没启动。
- 听在 `0.0.0.0:8088` 表示所有 IPv4 地址。对方可以用固定 IP 来连，这仍不等于防火墙已经放行。
- 教师问：`--query-port` 为 no，是不是就一定不可达。用户答 rich rule 仍可放行，这是对的。补了一句：服务定义里也可以带这个端口。query 只查直接写上的端口规则。
- 今天 runtime 和 permanent 的 8088/tcp 都是 yes，这条规则两边一致，不需要为它 reload。reload 没有执行。
- 现有 rich rule 只放行来自 192.168.56.12 的 IPv4 VRRP（IP 协议 112）。教师说明它不代替 TCP 8088。用户能解释，回答正确。
- 固定 IP 正常、VIP 异常时，先看 VIP 在哪一台，再看 keepalived。两台都没有 VIP，去查 keepalived 是否在跑。
- node2 的 `enp0s8` 上还有 192.168.56.22/24，标成 secondary。用途未知，没有动。两台各有一个 secondary 地址，不是双主。
- 当前 node1 是 MASTER 并持有 .50，node2 是 BACKUP、没有 VIP。这是这次看到的正常状态。角色可以换，不是永久固定。
- 这次固定 IP 和 VIP 都能访问，只说明当前通。不是故障切换验证。
- `journalctl -n 15` 只有 15 行，不能用它断定中间那段时间为什么是那样。不把当前状态写成故障。
- 分开测是在缩小范围，不是直接下诊断。顺序是：服务、监听、本机访问，再跨主机固定 IP，再 VIP。固定 IP 正常而 VIP 异常，才查 VIP 归属和 keepalived。

## 关键步骤

```bash
# 10:10 node1。任务 10:06 下发，输出是这时贴的
systemctl is-active nginx
sudo ss -lntp 'sport = :8088'
curl --noproxy '*' -i --connect-timeout 1 --max-time 2 http://127.0.0.1:8088/exam-health
```

nginx 是 active。`ss`：`LISTEN 0 511 0.0.0.0:8088`，nginx，pid 1723 和 1721，fd 6。curl：`HTTP/1.1 200 OK`，`nginx/1.26.3`，`text/plain`，length 15，正文 `exam health ok`。

10:12 要求在 node2 访问 node1。实际仍在 node1，并且漏了 `'*'`：

```bash
curl --noproxy -i --connect-timeout 1 --max-time 2 http://192.168.56.11:8088/exam-health
```

只打出正文 `exam health ok`。`-i` 被吃成 `--noproxy` 的参数，这次不是跨主机。

```bash
# 10:15 node2
curl --noproxy '*' -i --connect-timeout 1 --max-time 2 http://192.168.56.11:8088/exam-health
# 10:18 同一组参数，路径是 /
curl --noproxy '*' -i --connect-timeout 1 --max-time 2 http://192.168.56.50:8088/
```

.11 的 `/exam-health` 是 200，正文是预期的 `exam health ok`。VIP 的 `/` 是 200，length 19，正文 `ha test from node1`。

```bash
# 10:22–10:28 node1
sudo firewall-cmd --get-zone-of-interface=enp0s8
sudo firewall-cmd --zone=public --query-port=8088/tcp
sudo firewall-cmd --permanent --zone=public --query-port=8088/tcp
sudo firewall-cmd --zone=public --list-rich-rules
```

`enp0s8` 在 public。runtime 和 permanent 的 `8088/tcp` 都是 yes。rich rule 只有一条：`rule family="ipv4" source address="192.168.56.12" protocol value="vrrp" accept`。

```bash
# 10:34 两台
ip -4 addr show dev enp0s8
# 10:38 两台
systemctl is-active keepalived
sudo journalctl -b -u keepalived -n 15 --no-pager
```

node1 的 `enp0s8`：192.168.56.11/24，以及 192.168.56.50/24 secondary，接口 UP。node2：192.168.56.12/24，以及 192.168.56.22/24 secondary，接口 UP。两台 keepalived 都是 active。

这次 `-n 15` 里看到的是：

- node1，11-21 09:02:28：VI_1 `Receive advertisement timeout`，`Entering MASTER STATE`，设置 VIP，在 `enp0s8` 上为 192.168.56.50 发送/排队 gratuitous ARP。09:02:33 再发一次。
- node2，日志主机名 `node2.lab.example`。09:28:42 启动，分到 .12，removing VIPs。VRRP sockpool：IPv4、协议 112、unicast 地址 .12。`chk_nginx` 当时返回 7，failed，status 7。09:28:45 返回 0。10:14:32 成功，`Entering BACKUP STATE`，为 192.168.56.12 发 gratuitous ARP。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| node1 本机 `/exam-health` | 上面的 `curl` 127.0.0.1 | 200，`exam health ok` | ✅ 只证明这个地址 |
| 监听地址 | `sudo ss -lntp 'sport = :8088'` | `0.0.0.0:8088` | ✅ node1 |
| node2 访问 node1 固定 IP | 10:15 的完整 curl | 200，预期正文 | ✅ |
| node2 访问 VIP `/` | 10:18 的 curl | 200，`ha test from node1` | ✅ 当前通。不是切换验证 |
| 8088 的 zone 与端口规则 | `--get-zone-of-interface`、两次 `--query-port` | public；runtime 与 permanent 都是 yes | ✅ 未 reload |
| rich rule | `--list-rich-rules` | 只有 node2 的 VRRP | ✅ 不代替 TCP 8088 |
| 地址 | `ip -4 addr show dev enp0s8` | node1 有 .11 和 .50；node2 有 .12 和 .22 | ✅ 不是双主 |
| keepalived | `systemctl is-active` 与本次 boot 的 15 行 | 两台 active；node1 MASTER 持 .50，node2 BACKUP | ✅ 不能用 15 行解释时间跨度 |
| 故障注入、切换演练、reload | | | 未做 |
| 重启后仍是这样 | | | 未验证 |

## 踩坑

- 人还在 node1，命令写成 `curl --noproxy -i ...` → 漏了 `'*'`，`-i` 被当成 `--noproxy` 的参数，只剩正文，而且不是跨主机 → 换到 node2，补上 `'*'`，`-i` 才是 curl 自己的选项。
- 本机 200 就说「nginx 没问题」→ 本机响应只证所测的那个地址，服务可能只听 127.0.0.1 → 用 `sudo ss -lntp 'sport = :8088'` 看实际监听。口答这条命令是对的。
- 综合时先看了服务、端口、本机，然后说「多半网络或客户端」→ 还没做跨主机对照就归因 → 改为先跨主机固定 IP，再 VIP。固定 IP 正常、VIP 异常，查 VIP 归属和 keepalived。
- 说分开测是为了排除 keepalived → 缩小范围不等于直接诊断 → 先按上面的顺序取证，再决定查不查 keepalived。

## 卡点 / 没想明白的

- .22 的用途仍然未知，没有处理。
- 15 行日志跨过 09:02、09:28 和 10:14。为什么中间是那样，这 15 行不够。不把现在写成故障。

## 面试题

没有另出独立 5 题，没有分数。口答对的是：本机成功不能说服务没启动；`sudo ss -lntp 'sport = :8088'`；query 为 no 时 rich rule 仍可能放行；固定 IP 正常而 VIP 异常时先查 VIP 归属；两台都没有 VIP 就查 keepalived；当前 node2 BACKUP、node1 MASTER 持有 VIP 是这次的正常状态。

## 复习记录

起算日是 11-21。

- [x] D+1（11-22，1/1（合并复习，五题合计 4.5/5，不是本篇单独 5 题）；本机 200 不保证 node2 能访问。只听 127，或防火墙没放行，都会这样）
- [x] D+3（11-26，1/1（合并复习，六题合计 6/6，不是本篇单独 6 题）；固定 IP 正常而 VIP 异常时先查 VIP 在哪台机器）
- [ ] D+7

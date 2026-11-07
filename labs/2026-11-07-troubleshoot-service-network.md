# 排障（三）：网站打不开——服务、端口、防火墙、SELinux、网络

- 日期：2026-11-07（A4）
- 考点：面试高频"网站打不开怎么查"；RHCSA —— firewalld 运行时与永久规则、SELinux 文件标签；网络层的 ARP 与 ICMP
- 环境：Rocky Linux 10.2（VirtualBox）。node1 做服务端（httpd 2.4.63，firewalld，网卡在 public zone）；node2 做客户端；Windows 宿主机（192.168.56.1）作第二个客户端。工具：tcpdump、ausearch、ss、firewall-cmd
- 状态：完成（node2 开机卡住的根因、SYN 重传间隔、node2 侧 ARP 参数未查清，见卡点）
- 来源：根据课程对话和终端记录整理

## 目标

用"报错类型 + 耗时"先分流，再用 `ss`、`firewall-cmd`、`ip neigh`、`tcpdump`、`ausearch` 逐项验证。能区分 `refused`、`No route to host`、`timed out` 三种报错的来源；能分清 firewalld 运行时规则和永久规则。

## 关键概念

**1. 一次 curl 连接要过几关**

- 同一网段内，发 IP 包之前，必须先知道对方网卡的 MAC（Media Access Control，媒体访问控制地址，即二层地址）。这一步由 ARP（Address Resolution Protocol，地址解析协议）完成：本机广播"谁是 192.168.56.11"，对方回复后，结果写进邻居表（neighbour table，用 `ip neigh` 查看）。
- 拿到 MAC 后，客户端才发 TCP 的 SYN（synchronize，建连请求）。对端有四种可能的回应：
  - 有进程监听该端口：回 SYN-ACK，连接建立。
  - 没有进程监听：内核回 RST（reset，重置），客户端报 `Connection refused`。
  - 防火墙 REJECT（拒绝）：回 ICMP（Internet Control Message Protocol，互联网控制报文协议）不可达报文，客户端报 `No route to host`。
  - 防火墙 DROP（丢弃）：什么都不回，客户端重传 SYN，直到超时，报 `Connection timed out`。
- 同网段的目标不存在或已关机：ARP 没人回，邻居条目变成 FAILED（或 INCOMPLETE）。IP 包根本没有离开网卡，所以也报 `No route to host`，但耗时约 3 秒，因为 Linux 默认发 3 次 ARP 请求、每次间隔 1 秒。

**2. 为什么 `ip neigh` 能分开两种 `No route to host`**

防火墙的 REJECT 回包是对端发回来的，前提是 ARP 已经成功、对端的 MAC 已知。ARP 失败时，包根本送不到对端，也就不可能收到对端的 ICMP。所以邻居表的状态就是分界线：
- FAILED / INCOMPLETE → ARP 失败，问题在本机到目标的二层。
- REACHABLE / STALE → ARP 成功，拒绝来自对端（或它前面的防火墙）。

**3. 报错分流表（实测）**

| curl 报错（退出码） | 实测耗时 | 回包来源 | 原因 |
|---|---|---|---|
| Connection refused（7） | 5 ms | 对端回 RST | 包到达了、防火墙放行了，但这个 IP:端口 没有进程监听 |
| No route to host（7），防火墙 REJECT | 2 ms | 对端回 ICMP type 3 code 13（admin prohibited filter） | 防火墙主动拒绝 |
| No route to host（7），ARP 失败 | 3087 ms | 无回包；本机邻居表为 FAILED | 同网段找不到目标 MAC（目标关机或 IP 不存在） |
| Connection timed out（28） | 等于 `-m` 设定值（5003 ms） | 无回包，SYN 反复重传 | 防火墙 DROP，或包发出去没人回 |

退出码含义：7 = 连接失败（CURLE_COULDNT_CONNECT），28 = 操作超时（CURLE_OPERATION_TIMEDOUT）。

注意一个例外：防火墙如果用 `reject-with tcp-reset` 拒绝，客户端也会看到 `refused`。所以 `refused` 不能直接等同于"防火墙已放行"，要用 `ss` 和抓包确认。

**4. 排查顺序（我的建议）**

最佳做法：先用报错类型和耗时分流。一条 curl 就能排除大半可能，省下逐层排查的时间。判断标准是"排查成本低、一步排除多种可能"。固定顺序（服务 → 端口 → 防火墙 → SELinux → 网络）仍然要用，放在分流之后，用来逐层确认"这一层到底有没有问题"。

## 关键步骤

只记录实际执行过的命令。命令前的注释是这次补写的参数说明。

**实验 1：服务停止 → refused**

```bash
systemctl is-active httpd                 # 只输出状态（active / inactive）
sudo systemctl stop httpd                 # 停止服务，不改开机自启
sudo ss -lntp                             # -l 监听 -n 数字格式 -t TCP -p 显示进程
curl -v -m 5 http://192.168.56.11:8888/   # node2 上执行：-v 显示过程，-m 5 总时限 5 秒
sudo systemctl start httpd                # 恢复服务
```

结果：`Connection refused`，5 ms。预测（进程没监听，内核回 RST）正确。

**实验 2：找出占用 127.0.0.1:18080 的进程**

`ss -lntp` 里有一条 `127.0.0.1:18080`，它只监听回环地址，外部访问不到。

```bash
ps -fp <PID>                              # -f 完整格式，-p 按进程号查询
ls -l /proc/<PID>/cwd                     # 进程的工作目录（符号链接）
systemctl status <PID>                    # 传 PID 给 status，显示它属于哪个 unit（服务单元）
systemctl cat <服务名>                     # 打印 unit 文件，看 ExecStart 的绑定参数
sudo systemctl disable --now <服务名>      # disable 取消开机自启，--now 同时停止
systemctl is-enabled <服务名>              # 输出 disabled
sudo ss -lntp | grep 18080                # 无输出：已不监听
```

结果：unit 的 `ExecStart` 里写着 `--bind 127.0.0.1`。状态显示 `enabled; preset: disabled`，说明不是出厂默认，是有人手动 enable 的。已禁用，unit 文件和目录未删除。

**实验 3：防火墙移除放行 → No route to host（REJECT）**

```bash
sudo firewall-cmd --list-ports --permanent     # --permanent 读永久配置：8090/tcp 8888/tcp
sudo firewall-cmd --remove-port=8888/tcp       # 移除运行时的 8888 放行（没加 --permanent，只改运行时）
sudo ss -lntp | grep 8888                      # httpd 仍在监听 *:8888：服务本身没问题
curl -v -m 5 http://192.168.56.11:8888/        # node2 上执行
sudo firewall-cmd --add-port=8888/tcp          # 恢复放行
```

结果：`No route to host`，2 ms。ss 显示 httpd 仍在监听，说明问题在防火墙。

**实验 4：抓包确认回包是什么**

```bash
sudo timeout 20 tcpdump -n -i enp0s8 host 192.168.56.12 and port 8888
# timeout 20：20 秒后自动结束；-n 不解析主机名；-i 指定网卡；host / port 过滤对端和端口
# 运行期间，node2 上再执行一次 curl
```

结果：node2 的 SYN 发出后，node1 回 `ICMP host 192.168.56.11 unreachable - admin prohibited filter`。所以"防火墙 REJECT → No route to host"这条因果链被抓包证实。注意：`-n` 下端口仍被翻译，8888 显示成 `ddi-tcp-1`；`-nn` 才是地址和端口都不翻译。

**实验 5：富规则 drop → Connection timed out（DROP）**

```bash
sudo firewall-cmd --remove-port=8888/tcp      # 去掉端口放行，避免它干扰
sudo firewall-cmd --add-rich-rule='rule family="ipv4" port port="8888" protocol="tcp" drop'
# 富规则（rich rule）：对 IPv4 的 8888/tcp 直接丢弃，不回任何包
sudo timeout 20 tcpdump -nn -i enp0s8 host 192.168.56.12 and port 8888
curl -v -m 5 http://192.168.56.11:8888/       # node2 上执行
sudo firewall-cmd --remove-rich-rule='rule family="ipv4" port port="8888" protocol="tcp" drop'   # 恢复
sudo firewall-cmd --add-port=8888/tcp
```

结果：`Connection timed out after 5003 milliseconds`（退出码 28）。tcpdump 收到 5 个 SYN，序列号相同（都是同一个 SYN 的重传），间隔约 1 秒，没有任何回包。

**实验 6：SELinux 导致 403**

```bash
getenforce                                    # Enforcing：SELinux 正在拦截
ls -Z /data/www/                              # -Z 显示安全上下文（SELinux 标签）
sudo chcon -t user_home_t /data/www/index.html   # chcon 临时修改标签，-t 设置类型为 user_home_t
curl -v -m 5 http://192.168.56.11:8888/      # node2 上执行 → 403 Forbidden
sudo ausearch -m AVC -ts recent | tail -3     # ausearch 查审计日志：-m AVC 只看 SELinux 拒绝记录，-ts recent 最近时段
sudo tail -3 /var/log/httpd/error_log        # httpd 错误日志
sudo restorecon -v /data/www/index.html      # restorecon 按策略恢复默认标签，-v 显示改动
curl -v -m 5 http://192.168.56.11:8888/      # node2 上执行 → 200 OK
```

结果：
- AVC（SELinux 拒绝记录）：`denied { read }`，`scontext=...httpd_t`（进程类型），`tcontext=...user_home_t`（文件标签），`permissive=0`（enforcing 下拦截）。SYSCALL 的 `exit=-13` 即 EACCES（Permission denied）。
- error_log：`(13)Permission denied ... AH00132: file permissions deny server access: /data/www/index.html`。这句话对 DAC（普通文件权限，Discretionary Access Control）和 SELinux 拒绝都会出现，不能单独作为 SELinux 的证据。
- `restorecon` 后：`Relabeled ... from user_home_t to httpd_sys_content_t`，再次访问 200 OK。

**实验 7：ARP 失败（同网段、目标不存在）**

```bash
# node2 上执行
ip -br addr                                   # -br 简洁格式显示地址
ip route get 192.168.56.11                    # 看到达目标走哪个网卡、用哪个源地址
ping -c 2 192.168.56.11                       # -c 2 发两个包后停止：确认同网段是通的
curl -v -m 5 http://192.168.56.99:8888/       # 192.168.56.99 没有这台机器
ip neigh show 192.168.56.99                   # 邻居表：FAILED
# node1 上执行
sudo timeout 20 tcpdump -nn -i enp0s8 arp     # 只抓 ARP 包，-nn 不翻译
# node1 上查看 ARP 参数
sysctl net.ipv4.neigh.enp0s8.mcast_solicit   # 发送 ARP 请求的次数：3
sysctl net.ipv4.neigh.enp0s8.retrans_time_ms # 两次请求的间隔（毫秒）：1000
```

结果：
- `No route to host`，耗时 3087 ms；`ip neigh` 显示 `FAILED`。
- node1 抓到 3 条 `ARP, Request who-has 192.168.56.99 tell 192.168.56.12`，间隔约 1 秒。
- 3 次 × 1 秒 ≈ 3 秒，与 curl 的耗时吻合。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 服务停止 → 外部连接 | `systemctl stop httpd` + curl | Connection refused | ✅ 5 ms |
| 防火墙 REJECT | `--remove-port` + curl | No route to host，毫秒级 | ✅ 2 ms |
| REJECT 的回包 | tcpdump `-n` | ICMP admin prohibited | ✅ |
| 富规则 DROP | rich rule drop + curl | Connection timed out | ✅ 5003 ms，5 个 SYN，无回包 |
| SELinux 拒绝 | `chcon` 后 curl + ausearch | 403 + AVC denied | ✅ exit=-13 |
| 恢复标签 | `restorecon` + curl | 200 OK | ✅ |
| ARP 失败 | `ip neigh show` | FAILED | ✅ |
| ARP 重试次数与间隔 | tcpdump arp + sysctl | 3 次，约 1 秒 | ✅ 抓包在 node1 完成；sysctl 只在 node1 查 |
| 运行时删除不影响永久配置 | `--remove-rich-rule` 后 `--list-rich-rules --permanent` | 永久规则仍在 | ✅ 仍显示 9000 放行规则 |
| `--runtime-to-permanent` 同步 | 执行后 `--list-rich-rules --permanent` | 永久配置与运行时一致 | ✅ 永久 9000 规则也消失了 |
| 127.0.0.1 服务的外部可达性 | 外部 curl 18080 | Connection refused | 未验证（只看了 ss，没有做外部测试） |

## 踩坑

- **现象**：沿用了笔记里的状态（"httpd 监听 8888、8090"），判断 80 端口没开。**原因**：笔记是旧状态，没有用命令核对。**解决**：`ss -lntp` 显示 80、8090、8888 都在监听。结论：服务状态以命令输出为准。
- **现象**：`grep :80` 的结果里出现了 `:8090`。**原因**：grep 做的是子串匹配。**解决**：`sudo ss -lntp 'sport = :80'` 精确过滤。
- **现象**：Windows 上 curl 报 `Failed to connect to 127.0.0.1:10808 over proxy`。**原因**：环境变量 `http_proxy` 让 curl 先连代理。**解决**：`curl.exe --noproxy "*"` 直连。`--no-proxy` 不是合法选项（报 `cannot be reversed with a --no- prefix`）。
- **现象**：`systemctl --remove-port=8888/tcp` 报 `unrecognized option`。**原因**：把防火墙的参数用在了 systemctl 上。**解决**：防火墙用 `firewall-cmd`，systemctl 只管 unit。
- **现象**：富规则报 `INVALID_RULE: bad attribute 'procotol'`，改成 `prototol` 后报错信息变成 `'prototol'`。**原因**：拼写错误。**解决**：正确属性名是 `protocol`，firewalld 会直接指出错的属性名。
- **现象**：`getenfore: command not found`。**原因**：拼写错误。**解决**：`getenforce`。
- **现象**：删除富规则后，`--list-rich-rules --permanent` 仍然显示该规则。**原因**：`--remove-rich-rule` 只改运行时。**解决**：执行 `--runtime-to-permanent`，它把运行时状态写回永久配置，永久里的 9000 规则也随之消失。这个结果是想要的，但它是"覆盖"而不是"只删这一条"。执行前应先对比两边：`sudo firewall-cmd --list-all` 和 `sudo firewall-cmd --permanent --list-all`。

## 卡点 / 没想明白的

1. **node2 开机卡住（根因未知）**
   - 现象：两次在 VirtualBox 里直接关机（强制关闭）后，node2 开机卡住。
   - 在虚拟机控制台选"控制 → 关机"（ACPI，Advanced Configuration and Power Interface，高级配置与电源接口，即发出关机信号、让客户机自己关机）后，控制台打印关机日志，随后自动启动并恢复正常。
   - 未查清：卡在哪一步；强制关机是否造成文件系统损坏（推测，未验证）；为什么 ACPI 关机后会自动启动（推测：VirtualBox 或客户机的启动设置，未验证）。
   - 下一步：若日志可用，查看上一次启动的日志（`journalctl -b -1`），确认有没有文件系统错误。
2. **SYN 重传间隔不符合预期**：抓到的 5 个 SYN 间隔约 1 秒，没有增长。按 RFC 6298，重传超时（RTO，Retransmission Timeout）应该翻倍，间隔应该是 1、2、4 秒。原因未查清。待验证：`sysctl net.ipv4.tcp_syn_retries`，以及不设 `-m` 限制的长时间抓包。
3. **ARP 参数只在 node1 上查过**：`mcast_solicit=3`、`retrans_time_ms=1000` 是 node1 的值。第 7 步的 3 次请求是在 node1 上抓到的，与默认值一致，但不能直接证明 node2 的配置。待在 node2 上查。
4. **自学服务的 unit 文件和目录还在**：已禁用，未删除。待决定是否删除。
5. **node2 的 enp0s8 上有残留地址 192.168.56.22**（考试遗留），待用 nmcli 清理。
6. **（推测）127.0.0.1 服务的修法**：把绑定地址从 127.0.0.1 改成 0.0.0.0（或在 httpd 里去掉地址，只写端口），改后重启服务，用 `ss -lntp` 确认，再从外部 curl 验证。是否要对外开放，应先确认需求，不能想当然地改。本次实验未执行这一步。
7. **DAC 检查没有做**：`namei -l <文件路径>` 逐级查看权限，本次实验没有执行，待做。

## 面试题（本课 5 题，已批改）

这 5 题在课上作答过，下面是批改后的要点，不是闭卷回忆题。

1. `Connection refused` 怎么查？
   - 要点：先看 `ss -lntp` 有没有进程监听这个 IP:端口。服务 active 只说明进程在运行，不说明它监听了哪个端口和地址。监听 127.0.0.1 时，外部访问必定 refused。
2. `No route to host` 有哪两种来源？怎么区分？
   - 要点：耗时约 3 秒且邻居表为 FAILED / INCOMPLETE，是 ARP 失败；耗时几毫秒且邻居表为 REACHABLE / STALE，是防火墙 REJECT，抓包可见 ICMP admin prohibited。
3. 同网段目标关机，客户端报什么？`timed out` 对应什么？
   - 要点：ARP 失败，报 `No route to host`。`timed out` 对应 DROP（SYN 无回包）。其他来源（路由黑洞、回包路径不对称、跨网段网关静默丢包）属于推测，用 `ip route get` 和抓包区分。还要区分连接阶段超时（`Connection timed out`）和连接建立后的超时（`Operation timed out`，置信度中）。
4. 网站返回 403，怎么判断是 SELinux、文件权限，还是 httpd 配置？
   - 要点：先 `getenforce`。`ausearch -m AVC -ts recent` 有记录，是 SELinux；没有 AVC 时用 `namei -l` 查文件权限（DAC）；两者都正常，就去查 httpd 配置（如 `Require all denied`，这种情况不产生 AVC）。error_log 的 `Permission denied` 两种都会出现，不能作为 SELinux 的证据。
5. firewalld 规则重启后没了，怎么回事？运行时和永久有什么区别？
   - 要点：没加 `--permanent` 的规则只在运行时生效。`--reload` 会丢掉只在运行时的规则，只保留永久配置。`--runtime-to-permanent` 会覆盖永久配置，执行前先对比 `--list-all` 和 `--permanent --list-all`。

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

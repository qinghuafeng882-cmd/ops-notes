# firewalld

- 日期：2026-10-23（W4-D3）
- 考点：RHCSA EX200 —— 用 firewall-cmd / firewalld 限制网络访问
- 环境：Rocky Linux 10.2（VirtualBox），node1 服务端、node2 客户端，Windows 宿主机 192.168.56.1
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

理解 netfilter → nftables → firewalld → firewall-cmd 的分层、zone 匹配顺序、运行时与永久规则的区别，从客户端报错判断问题在哪一层，用 service、port、rich rule 放行。

## 关键概念

- 内核 netfilter 是检查点，nftables 是规则引擎（`nft list ruleset` 只看不改），firewalld 把 zone/service 翻译成 nft 规则，firewall-cmd 通过 **D-Bus** 给 firewalld 发请求。不要绕过 firewalld 直接加 nft/iptables 规则。
- D-Bus：进程间的消息总线，服务按名字登记（`org.fedoraproject.FirewallD1`），权限由 polkit 统一判断；RHEL 10 用 dbus-broker。只有 firewalld 一个进程写内核规则，firewall-cmd、NetworkManager、Cockpit 都只是发请求。firewalld 没运行时用 `firewall-offline-cmd` 直接改文件。
- 每个包只归一个 zone：**来源地址 > 网卡 > 默认 zone**。node1 的 enp0s3、enp0s8 都在 public（默认）。
- 运行时 `--add-...` 立即生效、reload 就没；`--permanent` 写 `/etc/firewalld/zones/public.xml`，reload 后才生效；`--runtime-to-permanent` 先试再存。`--list-*` 不带 `--permanent` 显示的是运行时。改永久配置时上一版备份成 `public.xml.old`。
- service 定义在 `/usr/lib/firewalld/services/*.xml`（http.xml 就是 tcp/80），可读、可包含多个端口；非标准端口用 port。
- rich rule 可以限定来源；`family="ipv4"` 只匹配 IPv4 包，同一主机走 IPv6 被默认拒绝。
- Linux 客户端看到 `No route to host` = 防火墙拒绝（ICMP host-prohibited）；`Connection refused` = 放行了但没有程序监听（TCP RST）；超时 = 丢弃或不通。**新版 curl 两种都显示 `Could not connect to server`**，用 `bash -c '</dev/tcp/主机/端口'` 才能看到内核错误。Windows 浏览器被拒绝时显示 `ERR_CONNECTION_TIMED_OUT`（推测：忽略 ICMP 拒绝一直重试）。
- 服务访问不了按层查：`ss -tlnp` → `firewall-cmd --list-all` → SELinux。
- RHEL 10：firewalld 删除 lockdown，新增 policy sets。

## 关键步骤

```bash
# 现状
sudo firewall-cmd --get-default-zone; sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
sudo cat /etc/firewalld/zones/public.xml
sudo nft list ruleset | head -40

# 测试服务（单独目录，不要在 /tmp 起：会公开整个目录）
mkdir -p ~/web8080 && echo "hello 8080" > ~/web8080/index.html
cd ~/web8080 && python3 -m http.server 8080

# 运行时 vs 永久
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --reload                       # 运行时规则消失
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# service
cat /usr/lib/firewalld/services/http.xml
sudo firewall-cmd --permanent --add-service=http # ALREADY_ENABLED（以前加过）

# rich rule：只允许 node2
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.12/32" port port="8080" protocol="tcp" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="192.168.56.12/32" port port="8080" protocol="tcp" accept'

# 客户端看内核错误（node2）
timeout 3 bash -c '</dev/tcp/node1/8080' && echo OK
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| zone | `--get-active-zones` | enp0s3、enp0s8 在 public | ✅ |
| 已放行服务 | `--list-all` | 含 W3 加的 nfs、mountd、rpc-bind | ✅ |
| 运行时规则 reload 后消失 | `--list-ports` | 空 | ✅ |
| 永久规则 reload 前看不到 | `--list-ports` | 空，reload 后 8080/tcp | ✅ |
| 防火墙拒绝 | node2 `/dev/tcp` | No route to host | ✅ |
| 没有监听 | node2 `/dev/tcp`（运行时放行 + 服务关） | Connection refused | ✅ |
| 放行 + 监听 | node2 `curl` | hello 8080 | ✅ |
| httpd | node2 `curl http://node1/`、Windows 浏览器 | 返回页面 | ✅（显示 SELinux test page，见踩坑） |
| rich rule 放行 node2 | node2 `curl node1:8080` | 成功 | ✅ |
| rich rule 不放行宿主机 | Windows 浏览器 :8080 | 失败 | ✅ `ERR_CONNECTION_TIMED_OUT` |
| rich rule 只限 IPv4 | node2 `curl -g http://[fd00:56::11]:8080/` | 失败 | ✅ |
| 收尾 | `--list-all` | ports、rich rules 为空，http 保留 | ✅ |

## 踩坑

- 实验 2 一开始直接成功 → `public.xml` 里早就有永久的 8080/tcp（以前的实验留下的）→ 先删掉回到干净状态再做。
- 在 `/tmp` 下跑 `python3 -m http.server` → node2 能看到整个 /tmp 的文件列表（包括 W3 解出的 `restore/sshd_config`）→ 用单独目录。
- curl 两种失败都只显示 `Could not connect to server`，分不出拒绝和没监听 → 用 `/dev/tcp`；但失败只用 3～7 ms 已说明是对方主动拒绝，不是丢弃。
- 往 `/var/www/html/index.html` 写了内容，页面却是 `SELinux test page` → `httpd.conf` 第 124 行 `DocumentRoot "/data/www"`（以前改过）→ 先查服务实际从哪里读文件。留给 SELinux 实验用。
- 第一次用 `/dev/tcp` 测出 OK → 当时 rich rule 放行、服务开着，没有造出要测的场景。
- `python` 命令不存在 → RHEL 只有 `python3`。

## 卡点 / 没想明白的

- D-Bus 是什么（已补讲，见关键概念）。
- 热身 1：sshd 拒绝公钥的日志原文记不住 → 关键词 `bad ownership or modes`。

## 回忆题

1. （概念）从 node2 访问 node1:8080，分别看到 `No route to host`、`Connection refused`、超时，说明什么？
<details><summary>答案</summary>

防火墙拒绝；防火墙放行但没有程序监听；包被丢弃或网络不通。新版 curl 看不出区别，用 `bash -c '</dev/tcp/node1/8080'`；最终以服务端 `ss -tlnp` 和 `firewall-cmd --list-all` 为准。
</details>

2. （操作）永久放行 http，并只允许 192.168.56.12 访问 tcp/8080，立即生效并确认。
<details><summary>答案</summary>

`sudo firewall-cmd --permanent --add-service=http`；`sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.12/32" port port="8080" protocol="tcp" accept'`；`sudo firewall-cmd --reload`；`sudo firewall-cmd --list-all`。
</details>

3. （排障）同事加了规则不生效；另一位同事加的规则重启后没了；第三位把 192.168.56.12 加进了 trusted，发现 public 里的限制对它不起作用。各自原因？
<details><summary>答案</summary>

只加了 `--permanent` 没 `--reload`；只加了运行时规则没 `--permanent`（或没 `--runtime-to-permanent`）；来源地址匹配优先于网卡，它的包按 trusted 处理。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

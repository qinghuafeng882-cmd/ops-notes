# Keepalived：VRRP 浮动 IP（HA 基础）

- 日期：2026-11-18
- 考点：运维扩展 —— Keepalived / VRRP 浮动 IP（RHCSA EX200 不考）
- 环境：Rocky Linux 10（VirtualBox）。node1 `enp0s8` 192.168.56.11/24；node2 192.168.56.12/24，另外已经有 192.168.56.22/24。网段 192.168.56.0/24
- 状态：完成（重启、断电、开机自启未做）
- 来源：课堂终端记录。归档时没有重跑

## 目标

两台 Nginx 用同一个 VIP 192.168.56.50/24。先不配健康检查，只验证 Keepalived 进程停掉时 VIP 会漂，优先级高的会抢回来。

## 关键概念

- Windows 上 `VBoxManage list dhcpservers`：DHCP 服务器 192.168.56.100，地址池 192.168.56.101–192.168.56.254。
- 192.168.56.200 做过 arping，无回应，但它在地址池里，不选。完整 arping 命令没有留存。
- 192.168.56.50 用 `arping -D -I enp0s8 -c 3`，无回应，而且在池外，选它做 VIP。无回应不等于绝对无人占用。
- node2 的 192.168.56.22/24 用途未知，没有处理。没有把这块地址写成 `enp0s8` 上的。
- 两台 `state BACKUP` 只是初始状态。谁当主由 priority 选：node1 150，node2 100。
- VRRP 是 IP 协议 112，不是 TCP/UDP 端口。放行用 rich rule：source 为对方固定 IP，`protocol value="vrrp" accept`。完整规则字符串没有逐字留存。
- VIP 只由 Keepalived 挂上。不要在网卡配置里把 192.168.56.50 静态绑死。
- 这一天没有健康检查。只停 Nginx、VIP 不会切走，是口答。没有停 Nginx 做实测。

## 关键步骤

```bash
# Windows 宿主机：确认 DHCP
VBoxManage list dhcpservers

# 选 VIP。无回应不等于绝对空闲
arping -D -I enp0s8 -c 3 192.168.56.50

# 两台：8088 标成 http 端口，永久放行。空格按这个写
semanage port -a -t http_port_t -p tcp 8088
firewall-cmd --permanent --add-port=8088/tcp
firewall-cmd --reload
nginx -t
```

`/etc/nginx/conf.d/ha-test.conf` 记下的是监听 8088，并 `return 200`。node1 的正文是 `'ha test from node1\n'`，node2 是 `'ha test from node2\n'`。`server_name` 等其它行没有留存。

包是 `keepalived-2.2.8-9.el10.x86_64`，服务最初 inactive。查询命令没有另记。先备份到 `/etc/keepalived/keepalived.conf.before-ha`。

覆盖写入之后的 VI_1（不是逐字底稿，字段是课堂定下来的）：

```text
vrrp_instance VI_1 {
    state BACKUP
    interface enp0s8
    virtual_router_id 51
    priority 150          # node2 为 100
    advert_int 1
    unicast_src_ip 192.168.56.11    # node2 写自己的 .12
    unicast_peer {
        192.168.56.12               # node2 写对方 .11
    }
    virtual_ipaddress {
        192.168.56.50/24 dev enp0s8
    }
}
```

两台 rich rule 永久允许对方固定 IP 的 VRRP，然后 reload。

启动 Keepalived 后 VIP 在 node1。停掉 node1 的 Keepalived，VIP 到 node2，访问到 node2 的正文。再启动 node1 的 Keepalived，VIP 被抢回，访问到 node1 的正文。启停命令的完整参数没有逐字留存。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| DHCP | `VBoxManage list dhcpservers` | 服务器 .100，池 .101–.254 | ✅ |
| .50 | `arping -D -I enp0s8 -c 3 192.168.56.50` | 无回应，池外 | ✅。无回应不等于绝对空闲 |
| .200 | arping，完整命令未留存 | 无回应，但在池内 | ✅ 不选 |
| 8088 页面 | 两台固定 IP 访问 | 各自返回自己的 `ha test from nodeN` | ✅。成功那次的完整 curl 未留存 |
| 配置语法 | `nginx -t` | 成功 | ✅ |
| 初始 VIP | 启动 Keepalived 后 | VIP 在 node1 | ✅ |
| 停 node1 Keepalived | | 转到 node2，响应 node2 | ✅ |
| 再启动 node1 Keepalived | | 抢回 VIP，响应 node1 | ✅ |
| 只停 Nginx | | 不会切走 | 口答。当天未实测 |
| 重启、断电、开机自启 | | | 未验证 |

## 踩坑

- `tee -a` 当成覆盖用 → 新的 VI_1 追加在默认配置后面，出现重复 VI_1、`eth0` 不存在，以及 garp/gna0 警告 → 改成不带 `-a` 的 `tee`，覆盖文件。警告原文没有逐字留存。
- node2 写成 `vitual_router_id` → `keepalived -t` 报 `Unknown keyword` → 改为 `virtual_router_id`。
- `vim` → `command not found` → 改用 `vi`。
- rich rule 写成 `accpet` → 拼写错误 → 改为 `accept`。
- rich rule 加在 runtime，reload 之后没了 → 当时不是 permanent → 改为 permanent 再 reload。
- `ip -br -4 show` → 少了 `addr`，没有留下报错原文 → 改为 `ip -br -4 addr show`。
- `curl --connect-timeout=2` → 这台不接受等号写法，没有留下报错原文 → 改为 `--connect-timeout 2`。

## 卡点 / 没想明白的

- node2 上 192.168.56.22/24 是干什么的，未知，没动。
- arping 没人回，不能当成地址绝对空闲。
- 没有健康检查时只停 Nginx 不会切 VIP。这是口答，11-18 没有做这个实验。

## 面试题（5 道，对话中作答后批改）

课后 5 题都对，总分 5/5。原题句子没有留存，下面只记考到的点，不复原原句，也不编答题原文。

1. （概念）VIP 选在 DHCP 池外。
<details><summary>批改要点</summary>

池是 192.168.56.101–254。.200 即使 arping 无回应也不能用。.50 在池外。无回应仍不等于绝对无人占用。
</details>

2. （概念）优先级决定谁是主。
<details><summary>批改要点</summary>

node1 150，node2 100。node1 回来后会抢回 VIP。
</details>

3. （概念）`state BACKUP` 是初始状态。
<details><summary>批改要点</summary>

两台都写 BACKUP，并不表示 node2 永远是备。选举看 priority。
</details>

4. （概念）VRRP 是什么协议。
<details><summary>批改要点</summary>

IP 协议 112，不是 TCP/UDP 端口。
</details>

5. （取舍）VIP 不要静态绑在网卡配置上。
<details><summary>批改要点</summary>

静态绑死后，Keepalived 停了地址还在，浮动 IP 失去切换意义。
</details>

## 复习记录

起算日是上课日 11-18。

- [x] D+1（11-19，2/2（合并复习，三篇合计 4.5/5，不是本篇单独 5 题）；BACKUP 只是初始状态，由优先级选举；没有健康检查时只停 Nginx 不会触发切换）
- [ ] D+3（自 11-18 起算，未做）
- [ ] D+7（自 11-18 起算，未做）

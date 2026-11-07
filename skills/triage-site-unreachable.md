---
name: triage-site-unreachable
description: 网站或服务连不上时使用（curl 报 refused、No route to host、timed out，或返回 403）：先按报错和耗时分流，再逐层验证服务、端口、防火墙、SELinux 和网络
---

# 网站打不开排障

## 什么时候用

客户端访问某个服务失败，需要定位原因时。适用于实验环境，也适用于真实环境。依据：`labs/2026-11-07-troubleshoot-service-network.md`（A4 实验）。

## 我会给你

- 客户端完整的 curl 输出，带 `-v` 和耗时。没有耗时的话，先让用户补上。
- 目标 IP 和端口，以及服务端、客户端分别是哪台机器。
- 你最近改过什么：服务配置、防火墙规则（运行时还是永久）、SELinux 标签、文件权限。
- （可选）服务端 `systemctl is-active` 的结果。

## 你要做的

1. **先分流，不要按固定顺序乱查。** 用报错和耗时判断先去哪一层：

| curl 报错（退出码） | 耗时 | 第一步去哪一层 |
|---|---|---|
| Connection refused（7） | 毫秒级 | 服务端：有没有进程监听这个 IP:端口 |
| No route to host（7） | 毫秒级 | 服务端：防火墙 REJECT |
| No route to host（7） | 约 3 秒 | 客户端：ARP（同网段找不到目标 MAC） |
| Connection timed out（28） | 等于 `-m` 设定值 | 防火墙 DROP，或包发出后没人回 |
| HTTP 403（连接已通） | — | 权限：SELinux、文件权限、httpd 配置 |

2. **一次只给一条命令，等用户输出再判断。** 每个新命令先讲清参数。命令由用户自己输入，不要整段复制粘贴。
3. **按分支验证：**
   - **refused**：`sudo ss -lntp 'sport = :端口'`。没有监听 → 查服务状态和日志。只监听 127.0.0.1 → 找配置里的绑定地址。改之前先确认这个服务该不该对外。
   - **No route（毫秒级）**：`sudo firewall-cmd --get-active-zones` 确认网卡所在的 zone，再看 `--list-all` 和 `--permanent --list-all`。用 `sudo timeout 20 tcpdump -nn -i 网卡 host 客户端IP and port 端口` 抓包，看是否有 ICMP admin prohibited。
   - **No route（约 3 秒）**：客户端执行 `ip route get 目标IP` 和 `ip neigh show 目标IP`。FAILED 或 INCOMPLETE → ARP 失败，确认目标存在且开机。REACHABLE 或 STALE → 拒绝来自对端，回到防火墙分支。
   - **timed out**：服务端抓包，看 SYN 是否到达、有没有回包。检查是否有 DROP 规则（`firewall-cmd --list-rich-rules`）。区分连接阶段超时（`Connection timed out`）和连接建立后的超时（`Operation timed out`，置信度中）。
   - **403**：`getenforce`，再执行 `sudo ausearch -m AVC -ts recent`。有 AVC 记录 → SELinux。没有 → 用 `namei -l 文件路径` 查文件权限。都正常 → 查 httpd 配置（如 `Require all denied`，这种情况不产生 AVC）。error_log 里的 `Permission denied` 两种都会出现，不能作为 SELinux 的证据。
4. **修复后验证。** 从客户端重新 curl，服务端用 `ss` 或 `firewall-cmd` 确认。改动要说明是运行时还是永久；临时改动要列出恢复步骤，并验证恢复。
5. **收尾。** 实验就写 `labs/` 笔记；真实排障就写 `troubleshooting/` 的脱敏复盘。只写真实发生的事，推测标注「（推测）」。
6. **最后说清楚**：最可能的根因、支持它的证据、还缺什么证据。

## 不要做

- 不要因为 `systemctl is-active` 显示 active 就判断服务正常。active 只说明进程在运行，不说明它监听了哪个端口和地址。
- 不要用 `grep :80` 过滤端口，它会匹配 `:8090`。用 `ss -lntp 'sport = :80'`。
- 不要把 error_log 的 `Permission denied` 当成 SELinux 的证据。
- 不要在没有对比的情况下执行 `firewall-cmd --reload` 或 `--runtime-to-permanent`。`reload` 会丢掉只在运行时的规则，`--runtime-to-permanent` 会覆盖永久配置。先对比运行时和永久两边。
- 不要把 `*` 写进服务配置。`*` 只是 `ss` 的显示形式。httpd 写 `Listen 端口`，其他程序写 `0.0.0.0:端口` 或 `[::]:端口`。
- 不要编造没有执行过的命令和输出。
- 不要出回忆单。

## 修订记录

- 2026-11-07：初版，依据 A4 实验整理。

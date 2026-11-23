---
name: triage-site-unreachable
description: >-
  Diagnoses unreachable websites or network services based on curl errors and latency
  (Connection refused, No route to host, Connection timed out, or HTTP 403), then verifies
  layer by layer across process sockets, firewalld, ARP/routing, SELinux, and file permissions.
  Use when a client fails to reach a local or remote web/TCP service.
---

# 网站打不开排障（Triage Site Unreachable）

## 什么时候用

客户端访问某个网站或 TCP 服务失败，需要按层次快速定位根因时使用。适用于 `node1`/`node2` 实验环境，也适用于脱敏后的真实排障。依据：`labs/2026-11-07-troubleshoot-service-network.md`（A4 实验）。

## 先向用户确认的信息

- 客户端带 `-v` 和耗时的完整 `curl` 输出（缺少耗时先补测）；
- 目标 IP、端口，以及服务端与客户端分别是哪台机器；
- 最近改过什么（服务配置、防火墙运行时/永久规则、SELinux 标签、文件权限）；
- （可选）服务端 `systemctl is-active` 结果。

## 执行步骤

1. **先按报错与耗时分流，不要按固定顺序乱查**：

| `curl` 报错（退出码） | 耗时 | 第一步去哪一层 |
|---|---|---|
| `Connection refused`（7） | 毫秒级 | 服务端：有没有进程监听这个 `IP:端口`（或防火墙 `tcp-reset`） |
| `No route to host`（7） | 毫秒级 | 服务端：防火墙 `REJECT` |
| `No route to host`（7） | 约 3 秒 | 客户端：ARP（同网段找不到目标 MAC）或网关路由 |
| `Connection timed out`（28） | 等于 `-m` 设定值 | 防火墙 `DROP`，或包发出后没人回 |
| `HTTP 403`（连接已通） | — | 权限与配置：SELinux、文件权限、Web 服务配置 |

2. **一次只给一条命令，等用户贴出输出再判断**。每条新命令先讲清关键参数，由用户自己输入执行。
3. **按分支逐层验证**：
   - **`Connection refused`**：`sudo ss -lntp 'sport = :端口'`。无监听 → 查服务状态与日志；只听 `127.0.0.1` → 核查配置绑定地址（改前先确认是否该对外）。
   - **`No route`（毫秒级）**：`sudo firewall-cmd --get-active-zones` 确认网卡所在 zone，对比 `--list-all` 与 `--permanent --list-all`。必要时抓包：`sudo timeout 20 tcpdump -nn -i 网卡 host 客户端IP and port 端口`，看是否有 ICMP `admin prohibited`。
   - **`No route`（约 3 秒）**：客户端执行 `ip route get 目标IP` 和 `ip neigh show 目标IP`（跨网段查网关 MAC）。`FAILED`/`INCOMPLETE` → ARP 失败，确认目标在线与二层连通；`REACHABLE`/`STALE` → 拒绝来自对端，回到防火墙分支。
   - **`timed out`**：服务端抓包看 `SYN` 是否到达、有无回包；检查是否有 `DROP` 规则（`firewall-cmd --list-rich-rules`）。区分建连阶段超时（`Connection timed out`）和建连后读取超时（`Operation timed out`）。
   - **`HTTP 403`**：先查 `getenforce` 与 `sudo ausearch -m AVC,USER_AVC -ts recent -i`（注意逗号后无空格，超过 10 分钟改用 `-ts today`）。有 AVC 记录 → SELinux（`fcontext` / `boolean` / `port`）；无 AVC → 用 `namei -l 文件路径` 查逐级目录与文件权限；都正常 → 查 Web 服务配置（如 `Require all denied` 或缺少目录索引，此类不产生 AVC）。
4. **修复后验证**：从客户端重跑 `curl`，服务端用 `ss` / `firewall-cmd` / `getsebool` 核实；明确区分运行时与永久生效，并记录恢复步骤。
5. **收尾与结论格式**：
   - 实验记入 `labs/`，真实排障脱敏后记入 `troubleshooting/`，只写真实事实，推测标注「（推测）」。
   - 结束时必须明确给出：**最可能的根因**、**支持它的证据**、**还缺什么证据**。

## 不要做

- 不要因 `systemctl is-active` 为 `active` 就断定服务对外正常。
- 不要用 `grep :80` 过滤端口（会误匹配 `:8090`），统一用 `ss -lntp 'sport = :80'`。
- 不要把 `error_log` 里的 `Permission denied` 直接当成 SELinux 的证据。
- 不要在未对比运行时与永久规则前执行 `firewall-cmd --reload` 或 `--runtime-to-permanent`。
- 不要把 `*` 写进服务配置文件（`*` 只是 `ss` 的展示形式；Nginx/其他程序用 `0.0.0.0:端口` 或 `[::]:端口`，httpd 用 `Listen 端口`）。
- 不要编造没执行过的命令和输出，也不要出回忆单。

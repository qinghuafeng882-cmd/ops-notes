# Prometheus 跨机采集 node2 与单节点断采告警

- 日期：2026-11-24（Asia/Shanghai）
- 考点：运维扩展 —— Prometheus 跨机拉取、firewalld 富规则限制来源、`SIGHUP` 热重载验证、单节点告警隔离。不是 RHCSA EX200
- 环境：Rocky Linux 10.2（VirtualBox），node1（192.168.56.11，Prometheus / Alertmanager / Grafana）/ node2（192.168.56.12，被采集节点），用户 fqh。Windows 浏览器看 Prometheus `/targets` 与 Grafana
- 状态：跨机采集、热重载实证、node2 断采触发告警及恢复验证均完成；监控模块未全做完（Alertmanager 仍无真实通知渠道）
- 来源：课堂终端记录与 Windows 浏览器截图。归档时没有重跑

## 目标

在 node2 上安装并启用 `node_exporter`，通过 firewalld 富规则只放行 node1（`192.168.56.11`）访问 `9100/tcp`，将 `192.168.56.12:9100` 加入 node1 的 `prometheus.yml` 并验证 `SIGHUP` 热重载、双机采集、单节点断采触发 `NodeExporterDown` 与自动恢复。

## 关键概念

- Prometheus 是集中式拉取（Pull）模型：node2 只装 `node-exporter` 暴露 `9100/tcp`，由 node1 主动发起连接，因此入站防火墙规则配在 node2。
- `firewalld` 富规则优先于裸 `--add-port=9100/tcp`：限定只允许监控机 `192.168.56.11` 访问 `/metrics`，避免向无关主机暴露主机指标。
- `firewalld` 富规则 `source address` 写单个 IPv4 地址时可省略 `/32`（效果相同）；`--remove-rich-rule` 删规则时必须原样照抄 `--list-rich-rules` 规范化后的文本。
- `ExecReload=/bin/kill -HUP $MAINPID` 只代表单元定义了重载命令，发 `SIGHUP` 是异步操作，`reload` 返回 0 不等于新配置已加载成功；需配合 `promtool check config`、校验前后 `MainPID` 未变，以及 `journalctl` 里 `Completed loading of configuration file` 才算实证热重载。
- `TimeoutStopSec=20s` + `SendSIGKILL=no` 用于停服时保护 TSDB 数据块落盘，避免强杀损坏数据。
- 修改 `prometheus.yml` 时保留 `'localhost:9100'` 而未改成 `'192.168.56.11:9100'`：更改 `instance` 标签会把同一台主机的历史时间序列拆成两条不同序列。
- Grafana 面板查询条件是 `node_memory_MemAvailable_bytes{job="node"}`（未限定 `instance`），加入 node2 后同一块面板会自动多出一条 `192.168.56.12:9100` 曲线。

## 关键步骤

```bash
# —— node1 核对现有服务与单元定义 ——
systemctl is-active prometheus.service prometheus-node-exporter.service prometheus-alertmanager.service
# active / active / active
systemctl cat prometheus.service
# ExecStart=/usr/bin/prometheus $ARGS，ExecReload=/bin/kill -HUP $MAINPID，TimeoutStopSec=20s，SendSIGKILL=no

# —— node2 核对仓库、端口与防火墙现状 ——
dnf repolist enabled                        # 仅有 appstream、baseos、extras（无 epel）
sudo ss -lntp 'sport = :9100'               # 无监听
sudo firewall-cmd --zone=public --list-all  # 已有 8088/tcp 及 192.168.56.11 的 vrrp 富规则

# —— node2 启用 CRB 与 EPEL 并安装 node-exporter ——
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release               # extras 的 epel-release-10-7.el10_1 + crb 两个 selinux-policy 依赖共 3 包
sudo dnf install node-exporter              # epel 的 node-exporter-1.12.1-1.el10_3.x86_64（1 包）

# —— node2 确认服务单元并启动本机验证 ——
rpm -ql node-exporter | grep -E '\.service$|/etc/'
sudo systemctl enable --now prometheus-node-exporter
systemctl is-active prometheus-node-exporter                 # active
sudo ss -lntp 'sport = :9100'                                # *:9100，pid=3789
curl -fsS http://localhost:9100/metrics | grep '^node_memory_MemAvailable_bytes'
# node_memory_MemAvailable_bytes 8.65910784e+08
```

```bash
# —— node2 配置防火墙富规则（运行时 + 永久）并在 node1 验证连通 ——
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.56.11/32" port port=9100 protocol="tcp" accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.56.11/32" port port="9100" protocol="tcp" accept'
sudo firewall-cmd --zone=public --list-rich-rules
sudo firewall-cmd --permanent --zone=public --list-rich-rules

# node1 执行跨机抓数测试（放行前的对照 curl 未执行）
curl -fsS --connect-timeout 2 http://192.168.56.12:9100/metrics | grep '^node_memory_MemAvailable_bytes'
# node_memory_MemAvailable_bytes 8.64854016e+08
```

```bash
# —— node1 备份、修改 prometheus.yml 并验证 SIGHUP 热重载 ——
sudo cp -a /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak-$(date +%Y%m%d-%H%M%S)
sudo cat /etc/prometheus/prometheus.yml
systemctl show -p MainPID --value prometheus.service          # 270230
sudo vim /etc/prometheus/prometheus.yml
# job_name: node 下改为：targets: ['localhost:9100', '192.168.56.12:9100']

sudo promtool check config /etc/prometheus/prometheus.yml    # SUCCESS: 1 rule files found; valid config syntax; 1 rules found
sudo systemctl reload prometheus.service
systemctl show -p MainPID --value prometheus.service          # 仍为 270230
sudo journalctl -u prometheus -n 15 --no-pager
# 09:09:28 systemd Reloaded 先返回；09:09:28.918 Loading -> 09:09:29.096 Completed loading (177.9ms)
```

```bash
# —— node1 验证双机指标入库 ——
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="node"}' -w '\n'
# localhost:9100 -> "1"，192.168.56.12:9100 -> "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=node_memory_MemAvailable_bytes{job="node"}' -w '\n'
# localhost:9100 -> "895168512"，192.168.56.12:9100 -> "854409216"
```

```bash
# —— 单节点断采告警与恢复验证 ——
# node2：
sudo systemctl stop prometheus-node-exporter

# 等待约 75s 后在 node1 查询：
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="node"}' -w '\n'
# localhost:9100 -> "1"，192.168.56.12:9100 -> "0"
curl -sS http://localhost:9093/api/v2/alerts -w '\n'
# 仅 1 条活动告警：NodeExporterDown，instance="192.168.56.12:9100"，state="active"，startsAt="2026-11-24T01:37:34.048Z"

# node2 拉起采集器（start 与 is-active 终端输出未贴回），随后 node1 复测：
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="node"}' -w '\n'
# 两条均恢复为 "1"
curl -sS http://localhost:9093/api/v2/alerts -w '\n'
# []
```

## 验证

| 检查项 | 命令 / 依据 | 预期结果 | 结果 |
|---|---|---|---|
| node2 服务启用与本机指标 | `systemctl is-active`、`ss -lntp`、`curl localhost:9100/metrics` | active、监听 `*:9100`、有可用内存指标 | ✅ `8.65910784e+08` |
| node2 富规则运行时与永久一致 | `--zone=public --list-rich-rules` 与 `--permanent` 对比 | 均含 `192.168.56.11/32` 放行 `9100/tcp` | ✅ |
| node1 跨机抓数 | `curl -fsS ... http://192.168.56.12:9100/metrics` | 返回指标 | ✅ `8.64854016e+08` |
| 配置校验与热重载（未重启） | `promtool check config` + `reload` 前后 `MainPID` 与 `journalctl` | SUCCESS、`MainPID` 保持 `270230`、日志显示加载完成 | ✅ `177.9ms` 完成加载 |
| Prometheus 双机采集 | `up{job="node"}` 与 Windows `/targets` 截图 | `2 / 2 up`，两台均为 `"1"` | ✅ 命令行 + 截图 |
| Grafana 双机曲线 | 刷新 `node1 基础监控` 面板 | `localhost:9100` 延续，新增 `192.168.56.12:9100` 曲线 | ✅ 截图 |
| 停 node2 独立告警与恢复 | 停 exporter 后查 `up` 与 `9093/api/v2/alerts`；恢复后再查 | 仅 `.12:9100` 为 `"0"` 且进 AM；恢复后均 `"1"` 且 `[]` | ✅ 恢复时 node2 终端未贴，以 node1 复测为准 |
| 放行前对照与非 node1 源拒绝 | 放行前或换源 IP `curl 192.168.56.12:9100` | 被拒绝 | 未做 |
| 系统重启后双机采集与富规则仍在 | 重启两台虚机后复测 | 均自动恢复 | 未验证 |

## 踩坑

- `curl -fsS http://localhost:9100/metrices` → `curl: (22) The requested URL returned error: 404` → 路径少写了标准拼写，`metrices` 应为 `metrics`（`-f` 遇 404 直接以 22 退出）→ 改为 `/metrics`。
- `curl ... | grep 'node_memory_memAvailable_bytes'` → 无输出 → Prometheus 指标名严格区分大小写 → 改为 `MemAvailable`。
- `curl --fsS ...` → `curl: option --fsS: is unknown` → 短选项合并只能用单 `-`，`--` 仅用于长选项 → 改为 `-fsS`。
- `systemctl show MainPID --value prometheus.service` → 吐出全部单元属性 → 漏写指定属性名的 `-p` → 改为 `systemctl show -p MainPID --value prometheus.service`。
- `curl ... --data-urlencode='query=up{job="node"}'` → `curl: option --data-urlencode=...: is unknown` → `--data-urlencode` 后面接空格而不是 `=` → 改回 `--data-urlencode 'query=...'`。

## 卡点 / 没想明白的

- 曾追问：`ExecReload=/bin/kill -HUP $MAINPID` 写在单元文件里，怎么知道 Prometheus 本身真支持热重载？已在课中厘清：单元文件只证明打包者配了命令，真正依据是官方文档说明 + 运行时实证（`reload` 前后 `MainPID` 都是 `270230`，且日志打出了 `Loading` → `Completed loading`，同时证实 `systemd` 在收到 `kill` 返回 0 后即刻报 `Reloaded`，比实际加载完成略早）。
- Grafana 原面板名称仍叫 `node1 可用内存`，因查询条件仅按 `{job="node"}` 筛选而同时画出了两台曲线，本次未改面板名称或筛选条件。

## 面试题（5 道，对话中作答后批改）

1. （概念）Prometheus 跨机采集 `node2` 指标时，为什么 `9100/tcp` 的入站防火墙规则必须配在 `node2` 上而不是 `node1` 上？
<details><summary>批改要点</summary>

Prometheus 是 Pull（拉取）模型，由 `node1` 主动向 `node2` 的 `9100/tcp` 发起连接，`node2` 是接收连接的监听端，因此入站规则必须配在 `node2`。1 分。
</details>

2. （排障）执行 `sudo systemctl reload prometheus` 后命令返回状态码 `0`，为什么还不能单独靠这个返回值断定新配置已经生效？
<details><summary>批改要点</summary>

`ExecReload=/bin/kill -HUP $MAINPID` 发信号是异步的，发出信号就返回 `0`，不等进程处理完配置；若新配置有误，Prometheus 会记错误日志并沿用旧配置，需结合 `promtool check config` 和 `journalctl` 确认加载完成（及 `MainPID` 未变）。1 分（补充：表述上可明确点出错误时保留旧配置与看日志确认）。
</details>

3. （排障）把 `node2` 加入 `prometheus.yml` 后，在 Prometheus 里看到 `192.168.56.12:9100` 采集失败，怎么一步步定位到是 `node2` 本机服务问题还是被 `node2` 防火墙拦截？
<details><summary>批改要点</summary>

先在 `node2` 本机检查服务、监听与 `curl localhost:9100/metrics`；确认本机正常后，再到 `node1` 执行跨机 `curl http://192.168.56.12:9100/metrics` 对照；本机通而跨机不通，再查 `node2` 防火墙富规则是否放行 `192.168.56.11`。0.5 分（方向对但用「或者」跳跃了「本机先测 → 跨机对照 → 再查防火墙」的分层排查顺序）。
</details>

4. （操作）写出在 `node2` 上通过 `firewalld` 永久添加「只允许 `node1`（`192.168.56.11`）访问 `9100/tcp`」的富规则命令。
<details><summary>批改要点</summary>

`sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.56.11/32" port port="9100" protocol="tcp" accept'`（实际生产需配套 `--reload` 或运行时规则才立刻生效）。1 分。
</details>

5. （取舍）今天修改 `prometheus.yml` 时保留了原有的 `'localhost:9100'` 而没有顺手改成 `'192.168.56.11:9100'`；如果当时顺手改掉了，会对现有的 Prometheus 数据和 Grafana 面板产生什么影响？
<details><summary>批改要点</summary>

改变 `instance` 标签会把同一台 `node1` 的数据切分成两条独立的时间序列，导致 Grafana 里原先的监控历史曲线分化/断层。1 分。
</details>

作答结果：4.5/5。最薄弱点：排障定位时跳过了「先本机验证，再跨机对照，最后查防火墙」的分层推进顺序。

## 复习记录

起算日是 11-24。这三项都还没做。

- [ ] D+1（11-25）
- [ ] D+3（11-27）
- [ ] D+7（12-01）

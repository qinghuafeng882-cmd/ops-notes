# W9 监控闭环收尾：业务告警规则、Grafana 整合与重启持久性验证

- 日期：2026-11-28（Asia/Shanghai）
- 考点：运维扩展 —— 双层业务告警（`service_alerts.yml`）、Go 模板变量（`{{ $labels.instance }}`）、Grafana Stat/Time series 面板整合与重启持久性验收。不是 RHCSA EX200
- 环境：Rocky Linux 10（VirtualBox），`node1`（192.168.56.11，Prometheus 3.15.1 / Alertmanager 0.34.2 / Grafana 10.2.6 / Nginx / MySQL 8.4.11 / nginx-prometheus-exporter 1.5.3 / mysqld_exporter 0.21.0）、`node2`（192.168.56.12:9100 在线），用户 `fqh`，Windows 浏览器看 Grafana
- 状态：W9 监控闭环全部完成（`service_alerts.yml` 热重载与业务停服触发/恢复验证通过、Grafana `node1 基础监控` 2×2 四面板整合保存、`node1` 重启持久性全部通过；`ServiceExporterDown` 未单独停 Exporter 触发，Alertmanager 维持 `lab-no-notification`）
- 来源：课堂终端记录与 Windows 浏览器截图。归档时没有重跑

## 目标

补齐 Nginx/MySQL 双层业务告警规则与 Grafana `node1 基础监控` 业务面板，并通过 `node1` 整机重启验收所有监控服务、规则加载、防火墙与 SELinux 持久状态。

## 关键概念

- **双层健康告警**：`nginx_up == 0` / `mysql_up == 0`（`severity: critical`）捕获「Exporter 活着但业务连不上」；`up{job=~"nginx|mysql"} == 0`（`severity: warning`）捕获「Exporter 自身宕机导致业务指标断采」的盲区。
- **规则拆分与 `rule_files`**：新增 `/etc/prometheus/service_alerts.yml`（`group: service_alerts`），与 `/etc/prometheus/node_alerts.yml` 分文件管理，并在 `/etc/prometheus/prometheus.yml` 的 `rule_files` 显式引入；`SIGHUP` 热重载时若规则有语法错误会中止加载并沿用旧配置。
- **告警文本 Go 模板变量**：`annotations.summary` 中用双花括号 `{{ $labels.instance }}` / `{{ $labels.job }}` 动态带出触发告警的标签值（漏写 `$labels.` 前缀如 `{{ $instance }}` 会报错）。
- **Grafana 面板整合与视觉优化**：同量纲指标合并减少面板冗余；`Options -> Legend -> Custom` 简化长标签；`Stat` 面板配合 `Value mappings`（`1 -> UP`、`0 -> DOWN`）与 `Stat styles`（`Graph mode: None`、`Color mode: Background`、`Text alignment: Center`）；QPS 面板单位设 `reqps`。图表上的空白断档表示「无采集样本」（如停服期间），不能解读为数值 `0`。
- **多单元 `systemctl start` 行为**：`systemctl start nignx mysqld` 其中一个服务名拼错报错时，不影响同命令行后面的合法单元（`mysqld` 仍会正常启动）。
- **`grep -o`**：`--only-matching`，仅输出每行匹配正则的部分，配合 `grep -oE '"(name|state)":"[^"]+"'` 从单行 JSON 提取字段。

## 关键步骤

```bash
# —— node1 核对 8 个服务运行态、开机自启与现有配置 ——
systemctl is-active prometheus prometheus-node-exporter prometheus-alertmanager grafana-server nginx mysqld nginx-prometheus-exporter mysqld-exporter
# 8 行全部 active
systemctl is-enabled prometheus prometheus-node-exporter prometheus-alertmanager grafana-server nginx mysqld nginx-prometheus-exporter mysqld-exporter
# 8 行全部 enabled
sudo cat /etc/prometheus/prometheus.yml
sudo cat /etc/prometheus/node_alerts.yml
```

```bash
# —— 备份主配置、编写 service_alerts.yml 并 SIGHUP 热重载 ——
sudo cp -a /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak-$(date +%Y%m%d-%H%M%S)

sudo tee /etc/prometheus/service_alerts.yml >/dev/null << 'EOF'
groups:
  - name: service_alerts
    rules:
      - alert: NginxDown
        expr: nginx_up{job="nginx"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nginx 业务探活失败持续一分钟 ({{ $labels.instance }})"

      - alert: MySQLDown
        expr: mysql_up{job="mysql"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL 业务探活失败持续一分钟 ({{ $labels.instance }})"

      - alert: ServiceExporterDown
        expr: up{job=~"nginx|mysql"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "业务 Exporter 采集失败持续一分钟 ({{ $labels.job }} / {{ $labels.instance }})"
EOF

sudo promtool check rules /etc/prometheus/service_alerts.yml
# SUCCESS: 3 rules found

sudo sed -i 's|^  - /etc/prometheus/node_alerts.yml$|  - /etc/prometheus/node_alerts.yml\n  - /etc/prometheus/service_alerts.yml|' /etc/prometheus/prometheus.yml
grep -A 3 '^rule_files:' /etc/prometheus/prometheus.yml

sudo promtool check config /etc/prometheus/prometheus.yml
# SUCCESS: 2 rule files found; node_alerts.yml (1 rules); service_alerts.yml (3 rules)
systemctl show -p MainPID --value prometheus.service     # 1575
sudo systemctl reload prometheus.service
systemctl show -p MainPID --value prometheus.service     # 仍为 1575
sudo journalctl -u prometheus.service -n 10 --no-pager
# 07:40:36.683 Loading -> 07:40:36.798 Completed loading (rules=17.416216ms, totalDuration=115.099521ms)
curl -fsS http://localhost:9090/api/v1/rules | grep -oE '"(name|state)":"[^"]+"'
# 4 条规则均已加载且为 inactive
```

```bash
# —— 停掉 nginx 与 mysqld（保留 Exporter）验证业务告警触发与模板变量渲染 ——
sudo systemctl stop nginx mysqld
sleep 90
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job=~"nginx|mysql"}' -w '\n'
# 127.0.0.1:9113 -> "1"，127.0.0.1:9104 -> "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=nginx_up{job="nginx"}' -w '\n'  # "0"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_up{job="mysql"}' -w '\n'  # "0"
curl -fsS http://localhost:9090/api/v1/rules | grep -oE '"(name|state)":"[^"]+"'
# NginxDown firing、MySQLDown firing；NodeExporterDown 与 ServiceExporterDown 保持 inactive
curl -fsS http://localhost:9093/api/v2/alerts | python3 -m json.tool
# 收到 NginxDown (127.0.0.1:9113) 与 MySQLDown (127.0.0.1:9104)，均 severity=critical、state=active
```

```bash
# —— 恢复服务、核对消除与 Grafana 接入基线 ——
sudo systemctl start nignx mysqld   # nignx 拼错报错，mysqld 仍启动
sudo systemctl start nginx
sleep 30
systemctl is-active nginx mysqld    # active / active
curl -fsS http://localhost:9090/api/v1/rules | grep -oE '"(name|state)":"[^"]+"'   # 全部 inactive
curl -fsS http://localhost:9093/api/v2/alerts -w '\n'                              # []
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="node"}' -w '\n'
# 192.168.56.12:9100 -> "1"，localhost:9100 -> "1"
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:3000/login               # 200
```

Grafana `node1 基础监控`（Windows 浏览器 `http://192.168.56.11:3000`）最终保存为 `2×2` 四面板：
- 左上 `业务实时 QPS（Nginx / MySQL）`（`rate(nginx_http_requests_total{job="nginx"}[1m])`、`rate(mysql_global_status_queries{job="mysql"}[1m])`，Unit `reqps`，恢复后约 `0.4 req/s` 与 `1.3–1.4 req/s`）；
- 右上 `Nginx & MySQL 存活状态`（`Stat` 面板，`nginx_up` / `mysql_up`，`Legend -> Custom` 设为 `Nginx` / `MySQL`，`Value mappings` `1->UP`/`0->DOWN`，`Graph mode: None`、`Color mode: Background`、`Text alignment: Center`）；
- 左下 `业务连接数（Nginx / MySQL）`（`nginx_connections_active`、`mysql_global_status_threads_connected`）；
- 右下 `node1 可用内存`（`node_memory_MemAvailable_bytes{job="node"}`，同时记录 `localhost:9100` 与 `192.168.56.12:9100`，`08:08–09:02` 停掉 MySQL 期间内存由 `~580 MiB` 升至 `~1 GiB` 后回落）。

```bash
# —— node1 整机重启持久性验收 ——
sudo reboot
# 重连后验证：
systemctl is-active prometheus prometheus-node-exporter prometheus-alertmanager grafana-server nginx mysqld nginx-prometheus-exporter mysqld-exporter
# 8 行全部 active
getsebool grafana_can_tcp_connect_prometheus_port      # on
sudo firewall-cmd --zone=public --query-port=3000/tcp  # yes
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up' -w '\n'
# 5 个目标（localhost:9090、192.168.56.12:9100、localhost:9100、127.0.0.1:9113、127.0.0.1:9104）全为 "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=nginx_up{job="nginx"}' -w '\n'  # "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_up{job="mysql"}' -w '\n'  # "1"
curl -fsS http://localhost:9090/api/v1/rules | grep -oE '"(name|state)":"[^"]+"'                           # 4 条均 inactive
```

## 验证

| 检查项 | 命令 / 依据 | 预期结果 | 结果 |
|---|---|---|---|
| 开机自启基线 | 8 服务 `is-active` 与 `is-enabled` | 全为 `active` 与 `enabled` | ✅ |
| 业务规则加载与热重载 | `promtool check rules/config`、`MainPID`、`journalctl`、`/api/v1/rules` | 2 文件 4 规则，`MainPID` 保持 `1575` | ✅ `115.1ms` 完成加载 |
| 停业务触发告警与恢复 | `stop nginx mysqld` 90s 后查 `up`/规则/AM；启后 30s 复查 | 停后 `up=1`、`NginxDown`/`MySQLDown` firing；启后恢复 `inactive` 与 `[]` | ✅ |
| Grafana 四面板整合 | Windows 浏览器 `node1 基础监控` 2×2 截图 | 存活 `UP` 绿卡、QPS `req/s`、连接数、可用内存正常且已保存 | ✅ |
| `node1` 重启持久性 | `sudo reboot` 后查 8 服务、`getsebool`、防火墙 `3000`、`up`/`nginx_up`/`mysql_up`、规则及浏览器 | 无需人工干预全部自愈恢复 | ✅ 浏览器正常打开 |
| `ServiceExporterDown` 停采集器实测 | 停 `nginx-prometheus-exporter` 或 `mysqld-exporter` 触发 `up == 0` | `ServiceExporterDown` firing | 未单测（语法与加载已核，此前已测 `NodeExporterDown`） |
| `node2` 独立重启复验 | 重启 `node2` 后验证富规则与 `node_exporter` | `.12:9100` 自动恢复 | 未贴 `node2` 重启记录（`node1` 重启后测得 `.12:9100` 为 `"1"`） |

## 踩坑

- `sudo systemctl start nignx mysqld` → `Failed to start nignx.service: Unit nignx.service not found.` → 把 `nginx` 误拼为 `nignx`（但同命令行后面的 `mysqld` 仍正常启动）→ 补执行 `sudo systemctl start nginx` 后双服务均恢复 `active`。
- 第一版 Grafana `Nginx & MySQL 存活状态` 面板卡片显示一长串 `{__name__=..., instance=...}` 且 `UP` 贴左带底部折线、`业务实时 QPS` 面板遗漏 → `Options -> Legend` 未切 `Custom`、Stat 默认 `Graph mode: Area` 且多序列默认左对齐 → 改用 `Legend -> Custom`、`Graph mode: None`、`Color mode: Background`、`Text alignment: Center` 并补齐 QPS 面板与 `2×2` 保存布局。

## 卡点 / 没想明白的

- 课堂追问 `grep -o`（仅输出正则匹配片段）与 `grep -oE '"(name|state)":"[^"]+"'` 的结构拆解，已逐段理清 `|` 分组与 `[^"]+` 反向字符集。
- 课后第 3 题把模板变量写成了 `{{ $instance }}`，漏掉了 `$labels.` 前缀（正确写法为 `{{ $labels.instance }}`）。

## 面试题（5 道，对话中作答后批改）

1. （复述）在 `/etc/prometheus/prometheus.yml` 中，新建的 `/etc/prometheus/service_alerts.yml` 必须显式写入哪一个配置字段下才能被 Prometheus 加载？
<details><summary>批改要点</summary>

`rule_files`（列表下显式写入文件路径）。1 分。
</details>

2. （解释原因）既然 `service_alerts.yml` 里已经配置了 `nginx_up == 0` 和 `mysql_up == 0` 来盯住业务存活，为什么还要额外加一条 `up{job=~"nginx|mysql"} == 0`（`ServiceExporterDown`）规则？
<details><summary>批改要点</summary>

防止业务 Exporter 自身宕机导致 Prometheus 抓不到 `nginx_up=0`/`mysql_up=0` 新样本、使业务存活告警断采失灵，需用 `up == 0` 捕获采集器故障。1 分。
</details>

3. （应用新场景）在编写通用告警规则的 `annotations.summary` 时，如果想在告警文本里自动填出触发故障的目标实例地址（`instance` 标签的值），应该写哪个模板表达式？
<details><summary>批改要点</summary>

`{{ $labels.instance }}`。用户作答写为 `{{ $instance }}`，漏掉了 `$labels.` 前缀（`promtool` 会报未定义变量错误）。0.5 分。
</details>

4. （排障）在今天刚整合好的 Grafana「业务连接数」面板上，看到 `08:08–09:08` 之间曲线出现了一段空白断档，排障看图时应该如何正确解读这段空白？
<details><summary>批改要点</summary>

空白表示该时段没有采集到对应样本（不能当成数值 `0`），说明当时服务停服或采集中断（需结合 `up` 和业务状态定位）。1 分。
</details>

5. （反事实）如果修改 `/etc/prometheus/service_alerts.yml` 时不小心写错了 YAML 缩进，没有跑 `promtool` 就直接执行 `sudo systemctl reload prometheus.service`，Prometheus 主进程会直接崩溃退出，还是会有别的处理行为？
<details><summary>批改要点</summary>

不会崩溃退出；Prometheus 收到 `SIGHUP` 后校验发现配置/规则有误会中止加载、记错误日志并继续沿用旧配置运行。1 分。
</details>

作答结果：4.5/5。最薄弱点：Go 模板变量 `{{ $labels.instance }}` 漏写 `$labels.` 前缀。

## 复习记录

起算日是 11-28。

- [ ] D+1（11-29）
- [ ] D+3（12-01）
- [ ] D+7（12-05）

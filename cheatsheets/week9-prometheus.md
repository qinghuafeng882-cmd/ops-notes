# W9 速查：Prometheus 与 Grafana（11-22）

Prometheus 只在 node1 本机采集。11-23 写过告警规则，并用 Windows 打开过 `http://192.168.56.11:9090/alerts`。同日本机装了 Alertmanager，没有配对外通知，也没有放行 9093/9094。整个监控模块没有做完。

| 做什么 | 命令 / 事实 |
|---|---|
| 安装 | `sudo dnf install prometheus node-exporter`。epel：`prometheus-3.15.1-1.el10_3`、`node-exporter-1.12.1-1.el10_3` |
| 看配置文件 | `rpm -qc prometheus node-exporter` |
| 服务名 | 用 `prometheus.service` 和 `prometheus-node-exporter.service`。`node-exporter.service` 不存在。doc 里的示例单元不要启动 |
| 检查配置 | `promtool check config /etc/prometheus/prometheus.yml` |
| 启用 | `sudo systemctl enable --now prometheus-node-exporter.service prometheus.service` |
| exporter 当场输出 | `curl -fsS http://localhost:9100/metrics` |
| 查已采集的数据 | `curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=...'`。`--get` 才把参数放进查询串 |

- 两个 default 文件生效的只有 `ARGS=''`：没有额外启动参数。注释不生效。默认采集器仍在。
- 全局采集 15s。job `prometheus` 覆盖成 5s，超时也是 5s，目标 `localhost:9090`。job `node` 不覆盖，目标 `localhost:9100`，沿用 15s。
- 配置里的 `localhost:9093` 不是已经装了 Alertmanager。`rule_files` 的示例是注释。文件没有改。
- `up=1` 只表示最近一次采集成功。`status success` 且 `result` 为 `[]` 是查询执行成功、没有匹配到序列。
- 9100 是 exporter 当时的文本。9090 是已存的样本，数值可以不同。即时查询 `value` 的第一项是查询求值时间戳，不一定是原始样本的采集时间。数值按输出原样留着。
- 拼写：`--available`、`prometheus`、`node_memory`、`MemAvailable`。

## Grafana（11-22 09:33–11:56，课堂跨度）

| 做什么 | 命令 / 事实 |
|---|---|
| 安装 | `sudo dnf install grafana`。依赖 `grafana-selinux`，弱依赖 `grafana-pcp` 默认会装。`--setopt=install_weak_deps=False` 只讲了，没执行 |
| 服务 | `grafana-server.service`。`enable --now` 的软链在 `/etc/systemd/system/multi-user.target.wants/`，指向 `/usr/lib/systemd/system/grafana-server.service` |
| 本机页面 | `curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:3000/login`。本机 200 不等于别的机器能连 |
| 防火墙 | public 含 `enp0s3` 和 `enp0s8`。临时 `--add-port=3000/tcp` 的终端输出没贴，登录页截图能开。永久那条输出 success，`--permanent --query-port=3000/tcp` 为 yes。没有 reload，没有全量 runtime-to-permanent |
| SELinux | `ausearch -m AVC,USER_AVC`，逗号后不能有空格。`name_connect` 到 9090。只开 `grafana_can_tcp_connect_prometheus_port`，持久用 `setsebool -P` |
| 数据源 | Grafana 后端访问 `http://localhost:9090`，localhost 是 node1。间隔不改变 Prometheus 采集 |
| 面板 | `node_memory_MemAvailable_bytes{job="node"}`。单位 bytes(IEC) 只改显示。Apply 是面板，Save 是整个 dashboard |

管理员密码不入库，记 `...`。没有证明首次登录后改过密码。没有关防火墙或 SELinux，没有 `audit2allow`，没有重启验证。

## 告警规则（11-23）

`/etc/prometheus/node_alerts.yml`：`NodeExporterDown`，`up{job="node"} == 0`，`for: 1m`，标签 `severity: warning`。`rule_files` 写绝对路径，不会自动扫全部文件。

- `for` 是持续成立的时间，不是采集周期，也不是 `evaluation_interval` 15s。中途恢复要重新计时。短暂失败把 Pending 清掉，只讲了，没做。
- `up == 0` 只是最近一次采集失败，不是整机宕机。Firing 不是通知已送达。11-23 上午写规则时，`localhost:9093` 只是预设地址。同日稍后本机 Alertmanager 已在 9093 运行，见下面一节。
- 停/启 `prometheus-node-exporter` 的终端输出没贴。页面截图：07:06 Inactive，07:09 Pending（19.282s，value 0），07:11 Firing（1m50.299s，value 0），07:14 Inactive。截图时长不是精确触发延迟。
- `systemctl status -l` 是 `--full`。`--no-pager` 不进分页器。没有做系统重启验证。

## Alertmanager（11-23）

包 `alertmanager`，服务 `prometheus-alertmanager.service`。9093 是 Web，9094 是集群。这次没有开放 9094，没有从 Windows 访问。抑制没做。静默见 `labs/2026-11-23-alertmanager-silence.md`。

| 做什么 | 命令 / 事实 |
|---|---|
| 安装 | `sudo dnf install alertmanager`。epel `0.34.2-1.el10_3`，1 个包 |
| 检查 | `sudo amtool check-config /etc/prometheus/alertmanager.yml`。有 global config 不等于配了邮件 |
| 最小配置 | `group_by: ['alertname','instance']`，`group_wait` 30s，`group_interval` 5m，`repeat_interval` 3h，receiver `lab-no-notification`，没有渠道 |
| 发现 | `curl -sS http://localhost:9090/api/v1/alertmanagers` 里有 9093，只是发现目标 |
| 收到 | 停 node_exporter 后查 `http://localhost:9093/api/v2/alerts`。8083 会打到 nginx，返回 301 |
| 恢复 | 启动 node_exporter 后再查，结果 `[]`。不是历史删除，也不是通知送达 |

`for` 是规则持续阈值。`group_wait` 是新分组第一次通知前的汇集等待。`group_interval` 是已通知分组有变化后的间隔。`repeat_interval` 是没有新变化时的重复提醒。下划线名字本来有效，改成连字符不是修语法。示例配置里的密码记 `...`。

## 静默（11-23）

`amtool silence add` 不用 sudo。这次匹配 `alertname=NodeExporterDown` 和 `instance=localhost:9100`，两个都要满足。`--author=fqh`，`--duration=10m`，comment 是「个人实验：验证维护静默」。返回的 ID 是 `61de6ff6-d421-4e95-a2f5-c608b783ef1b`。

- 静默不改 Prometheus 规则，也不修复故障。Firing 不会因此变成 Inactive。
- 匹配上之后告警还在，`silencedBy` 含这个 ID，`state` 是 suppressed。没有通知渠道，所以没有验证外部通知被拦住。
- `silence expire` 没有输出。采集器仍停着时，告警回到 active。提前结束不是删告警。
- 再 `start prometheus-node-exporter`，`/api/v2/alerts` 变成 `[]`。
- 钟点是对话记录，不是精确的状态变化时刻。没有等满 10 分钟，没有做抑制，没有测通知送达，没有系统重启。

## 分组（11-23）

没有停服务，也没有改配置。`amtool alert add` 直接注入 3 条 `lab=grouping` 的测试告警。`instance` 是 `Lab-node-a`、`Lab-node-b`，大写 L，只是标签，不连接主机。

`.../alerts/groups?filter=lab%3Dgrouping` 是 2 组：`Lab-node-a` 里 warning 和 critical 两条，`Lab-node-b` 里 1 条 warning。这是分组结果，不是已经发出合并通知。没有配 `inhibit_rules`。

`alert add` 没有 `--duration`。annotation 不是 label。`filter%3Dgrouping` 和 `fliter` 都不算筛选成功。清理后的 `[]` 不能单独证明是提前清理，因为当时已经过了 `endsAt` 10:19:46。没有测通知投递，没有系统重启。

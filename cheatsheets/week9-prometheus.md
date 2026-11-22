# W9 速查：Prometheus 首课（11-22）

只在 node1 本机。Grafana、Alertmanager、告警规则、改防火墙、从别的机器访问，都还没做。

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

# W9 速查：Prometheus 与 Grafana（11-22）

Prometheus 只在 node1 本机。Alertmanager、告警规则、从别的机器访问 Prometheus 的 9090，都还没做。Grafana 基础见下面一节。整个监控模块没有做完。

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

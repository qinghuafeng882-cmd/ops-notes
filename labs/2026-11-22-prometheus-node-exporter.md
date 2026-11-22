# Prometheus 首课：node1 本地采集

- 日期：2026-11-22（北京时间。复习 06:43–06:51；本课 07:01–09:27）
- 考点：运维扩展 —— Prometheus 与 node_exporter。不是 RHCSA EX200 大纲
- 环境：Rocky Linux 10.2（VirtualBox），node1，用户 fqh
- 状态：首课做完。只在 node1 本机跑 Prometheus 和 node_exporter。Grafana 下一课，未开始。监控这一块没有全部做完
- 来源：课堂终端记录。归档时没有重跑

## 目标

在 node1 上装好 Prometheus 和 node_exporter，用默认配置把本机 9100 的指标采进 9090，并分清「exporter 此刻的输出」和「Prometheus 已经存下的查询结果」。

今天先看了 `crontab -l`，是空的，没有加任务。11-04 的 cron 不重复做。复习大约 10 分钟，没有把旧实验再做一遍。

## 关键概念

- Prometheus 负责采集、存储、查询和规则。exporter 提供指标。Prometheus 主动来拉，不是等对方推。
- 包名 `node-exporter` 不等于服务名。要启用的是 `prometheus-node-exporter.service` 和 `prometheus.service`。`node-exporter.service` 不存在。`/usr/share/doc/node-exporter/examples/systemd/node_exporter.service` 只是示例，不当成服务去启动。
- `systemctl cat prometheus-node-exporter.service` 和 `systemctl cat node_exporter.service` 显示的是同一个 `prometheus-node-exporter.service` 源定义。用前者这一份，不把两个名字再启动一遍。
- 两个单元都是 `User=prometheus`、`Restart=on-failure`。`EnvironmentFile` 分别是 `/etc/default/prometheus`、`/etc/default/prometheus-node-exporter`。`ExecStart` 分别是 `/usr/bin/prometheus $ARGS`、`/usr/bin/prometheus-node-exporter $ARGS`。
- 两个 default 文件里，唯一生效的设置是 `ARGS=''`。其余是成百行的 `#` 注释。`ARGS` 为空表示没有额外启动参数，按二进制自己的默认跑，不等于没有采集器。注释里写 default enabled，并不是把这行注释启用了。旧注释也不保证等于这个版本的全部选项。
- 链是：`EnvironmentFile` 提供 `ARGS`，`ExecStart` 用 `$ARGS`。
- 默认 `prometheus.yml` 没有改，也没有再追加同名 job。记下的字段：
  - `global`：`scrape_interval` 15s，`evaluation_interval` 15s，`external_labels` 为 `monitor: example`
  - `alerting` / `alertmanagers` / `static_configs` / `targets`：`localhost:9093`
  - `rule_files` 下面的示例都是注释
  - job `prometheus`：`scrape_interval` 5s，`scrape_timeout` 5s，目标 `localhost:9090`
  - job `node`：没有覆盖间隔，目标 `localhost:9100`
- YAML 靠缩进。冒号后面是值，`-` 是列表，`#` 后面是注释。
- job 名不是主机地址。这里的 `localhost` 就是 node1。默认采 `/metrics`。
- 采集间隔、规则计算间隔、采集超时不是一回事。job 上写了 5s 就盖过全局 15s。`node` 没写间隔，沿用全局 15s。
- `external_labels` 用于对外通信，不是给本机每条样本统一再贴一遍。
- 配置里写着 `localhost:9093`，不等于已经装了 Alertmanager。
- `up{job="node"}` 为 1，只表示最近这一次采集成功，不是业务整体健康。
- 直接读 `9100/metrics` 是 exporter 当时的输出。查 `9090` 是已经采集并存下来的数据。两个数可以不一样。
- 即时查询里，`value` 的第一项是这次查询的求值时间戳，不一定是原始样本的采集时间。数值本身按输出原样留着。
- `curl --get` 把 `--data-urlencode` 放到查询串里。不加 `--get` 时，`--data-urlencode` 默认是 POST。
- `-w` / `--write-out` 在响应之后按格式再输出。这里用 `-w '\n'` 补一个换行。提示符粘在 JSON 末尾，是因为没有换行，不是请求没结束。
- `status success` 且 `result` 为 `[]`：查询本身成功，只是没有匹配的序列。不是数值 0，也不是监控挂了。

## 关键步骤

安装前：9090 和 9100 没有监听。内存 1.7Gi，available 804Mi，swap 4Gi 没有使用。没有把以前的服务实验再做一遍。

```bash
sudo dnf install prometheus node-exporter
rpm -qc prometheus node-exporter
rpm -ql node-exporter | grep -E '\.service$'
systemctl cat prometheus-node-exporter.service node_exporter.service
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl enable --now prometheus-node-exporter.service prometheus.service
systemctl is-active prometheus-node-exporter.service prometheus.service
```

只装了 2 个包，来自 epel，没有升级或删除别的包：`prometheus-3.15.1-1.el10_3`、`node-exporter-1.12.1-1.el10_3`。下载约 50M，安装后约 207M。

`rpm -qc` 给出：

- `/etc/default/prometheus`
- `/etc/prometheus/prometheus.yml`
- `/etc/default/prometheus-node-exporter`
- `/etc/logrotate.d/prometheus-node-exporter`

`rpm -ql` 里带 `.service` 的是：

- `/usr/lib/systemd/system/node_exporter.service`
- `/usr/lib/systemd/system/prometheus-node-exporter.service`
- `/usr/share/doc/node-exporter/examples/systemd/node_exporter.service`

`promtool` 输出：`SUCCESS`，配置语法有效。

`enable --now` 在 `/etc/systemd/system/multi-user.target.wants/` 下建了两个软链，分别指向 `/usr/lib/systemd/system/prometheus-node-exporter.service` 和 `/usr/lib/systemd/system/prometheus.service`。`is-active` 按同一顺序都是 active。没有重启后再查。

```bash
curl -fsS http://localhost:9100/metrics | grep '^node_memory_MemAvailable_bytes'
curl -fsS --get 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=up{job="node"}'
curl -fsS --get 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=node_memory_MemAvailable_bytes{job="node"}' -w '\n'
```

9100 这一行：`node_memory_MemAvailable_bytes 7.3586688e+08`，也就是 735866880 字节，约 702MiB。

9090 查 `up{job="node"}`：`status` success，`resultType` vector。metric 有 `__name__=up`、`instance=localhost:9100`、`job=node`。value 是 `[1795306779.213,"1"]`。

9090 查 `node_memory_MemAvailable_bytes{job="node"}`：`status` success，`resultType` vector。metric 有 `__name__=node_memory_MemAvailable_bytes`、`instance=localhost:9100`、`job=node`。value 是 `[1795310011.386,"729673728"]`。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 配置语法 | `promtool check config /etc/prometheus/prometheus.yml` | SUCCESS | ✅ |
| 两个服务 | `systemctl is-active` 按 enable 的顺序 | active、active | ✅ 本次启用成功 |
| exporter 有内存指标 | `curl -fsS http://localhost:9100/metrics` 再 grep | 有 `node_memory_MemAvailable_bytes` | ✅ `7.3586688e+08` |
| 采集到 node 这个 job | 查 `up{job="node"}` | 最近一次采集成功 | ✅ value 为 `"1"`。不是业务整体健康 |
| 存下来的同一指标 | 查 `node_memory_MemAvailable_bytes{job="node"}` | 能查到已采集的值 | ✅ `"729673728"`。和刚才 9100 的数不同 |
| 改配置、加规则、装 Alertmanager | | | 未做。9093 只写在配置里 |
| 改防火墙、从别的机器访问 | | | 未做 |
| Grafana | | | 未开始 |
| 重启后仍在 | | | 未验证 |

## 踩坑

- `dnf list --avaliable`、`dnf list --availabel` → 分别报 `dnf list: error: unrecognized arguments: --avaliable` 和 `dnf list: error: unrecognized arguments: --availabel` → 改成 `--available`。
- 通配写成 `*promethous*` → 把 prometheus 拼错了 → 改成 `*prometheus*`。
- 去找 `node-exporter.service` → 包名不是单元名，这个单元不存在 → 用 `rpm -ql` 和 `systemctl cat` 确认，只启用 `prometheus-node-exporter.service`。
- grep 写成 `node_memoey` → 指标名是 `node_memory`，所以没有行 → 改成 `node_memory_MemAvailable_bytes`。
- 查询写成 `MemAvailabel` → 名字错了。`status` 仍是 success，`result` 是 `[]` → 查询成功但没有这条序列，不是 0，也不是监控故障。改成 `MemAvailable`。
- 先丢命令、后讲原因 → 用户明确说应该先讲 → 改为先讲机制和配置，再做实验、再提问。

## 卡点 / 没想明白的

- 9100 上的 735866880 字节和 9090 里的 729673728 不是同一次读数。一个是当时 exporter 的输出，一个是已经采进库的样本。即时查询里，`value` 的第一项是查询求值时间戳，不一定是原始样本的采集时间。这两个时间戳和数值都按输出原样留着。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，每题 1 分。总分 5/5。没有另编题目。

1. （概念）Prometheus 和 exporter 各做什么，数据怎么过来。
<details><summary>批改要点</summary>

Prometheus 采集、存储、查询、算规则。exporter 提供指标。Prometheus 主动拉取。用户说成「接收数据」，已说成主动拉取。1 分。
</details>

2. （概念）job 上的 5s 和全局 15s 谁生效，`node` 这个 job 用哪个间隔。
<details><summary>批改要点</summary>

任务级 5s 盖过全局 15s。`node` 没有覆盖间隔，沿用 15s。1 分。
</details>

3. （排障）`up` 为 1 表示什么。
<details><summary>批改要点</summary>

最近一次采集成功。不是业务健康。1 分。
</details>

4. （操作）`ARGS=''` 是什么意思，注释里的默认项算不算打开。
<details><summary>批改要点</summary>

没有额外启动参数。注释不生效。二进制自己的默认采集器仍然工作。用户说成「没有单独的变量」，已说成没有额外参数。1 分。
</details>

5. （排障）`status` 是 success，`result` 却是 `[]`。
<details><summary>批改要点</summary>

查询成功，没有匹配到序列。不是直接的监控故障，也不是数值 0。1 分。
</details>

作答结果：5/5。

## 复习记录

起算日是 11-22。

- [x] D+1（11-23，1/1（合并复习，五题合计 5/5，不是本篇单独 5 题）；`up{job="node"}` 为 1 只表示最近一次采集成功）
- [ ] D+3（11-25）
- [ ] D+7（11-29）

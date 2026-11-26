# Alertmanager 基础：本机接收，不对外通知

- 日期：2026-11-23（Asia/Shanghai）
- 考点：运维扩展 —— Alertmanager。不是 RHCSA EX200。规则沿用 [当天的告警规则课](2026-11-23-prometheus-alert-rules.md)
- 环境：Rocky Linux 10.2（VirtualBox），node1，用户 fqh
- 状态：本机接收做完。没有配对外通知。监控模块没有全部做完
- 来源：课堂终端记录。归档时没有重跑

## 目标

在 node1 上装 Alertmanager，用一个没有通知渠道的接收方接住已有的 `NodeExporterDown`，并分清「Prometheus 发现了 9093」和「告警真的进了 Alertmanager」。

## 关键概念

- 包名是 `alertmanager`。服务名是 `prometheus-alertmanager.service`。两个名字不一样。
- 9093 是 Web。9094 是集群通信。这次没有开放 9094。
- `for: 1m` 是规则侧条件要持续成立的时间。`group_wait: 30s` 是新分组第一次通知前，用来把同时来的告警收在一起的等待。`group_interval: 5m` 是已经通知过的分组有变化时，下一次通知的间隔。`repeat_interval: 3h` 是告警还在、又没有新变化时，重复提醒的间隔。
- `group_by` 按列出的标签值合在一起分组。`alertname` 相同、`instance` 不同，按这两个标签分组时不是同一组。
- `receiver` 要和 `receivers` 里的 `name` 对上。没有渠道的接收方仍然接收、处理告警，只是不往外发通知。
- 静默是计划内维护时不发通知。抑制是因为另一条告警，压住次要通知。这节只讲了，没有做。
- `amtool` 报出 global config，只说明用了默认的全局配置。不表示邮件已经配好。示例配置里的密码不抄，一律 `...`。
- `ExecReload=/bin/kill -HUP $MAINPID` 只说明这个单元支持 reload。这次没有执行 reload。
- Prometheus 的 `activeAlertmanagers` 里出现 `http://localhost:9093/api/v2/alerts`，只说明发现了这个目标。不是告警已经送达。要确认收到，得在告警触发后去查 Alertmanager 里的具体告警。
- `/-/ready` 的正文是 `OK`，后面没有换行，提示符会粘在后面。这不是报错。

## 关键步骤

```bash
rpm -qa | grep alertmanager
rpm -qa | grep -i alertmanager
dnf list --available '*alertmanager*'
sudo dnf install alertmanager
```

前两条都没有输出。`dnf list` 给出 epel 的 `alertmanager.x86_64 0.34.2-1.el10_3`。安装时用户确认 `y`，成功装了 1 个包。

```bash
rpm -ql alertmanager | grep -E '/etc/|\.service$'
sudo systemctl cat prometheus-alertmanager.service
sudo cat /etc/default/prometheus-alertmanager
sudo cp -a /etc/prometheus/alertmanager.yml /etc/prometheus/alertmanager.yml.bak-$(date +%Y%m%d-%H%M%S)
```

`rpm -ql` 里有：`/etc/default/prometheus-alertmanager`、`/etc/logrotate.d/prometheus-alertmanager`、`/etc/prometheus/alertmanager.yml`、`/etc/prometheus/alertmanager_templates`，以及 `/usr/lib/systemd/system/prometheus-alertmanager.service`。原来的 `alertmanager.yml` 是多团队示例，带邮件和外部渠道。那些渠道全部去掉。示例密码不写入笔记。

单元：`User=prometheus`，`EnvironmentFile=/etc/default/prometheus-alertmanager`，`ExecStart=/usr/bin/prometheus-alertmanager $ARGS`，`ExecReload=/bin/kill -HUP $MAINPID`，`Restart=on-failure`。

default 文件里生效的是 `ARGS=''`。注释里的默认是配置文件 `/etc/prometheus/alertmanager.yml`、Web 9093、cluster 9094。备份命令没有报错。展开后的备份文件名没有贴出来。

```bash
sudo tee /etc/prometheus/alertmanager.yml << 'EOF'
```

先写成 `receiver` / `name` 都是 `lab_no_notification`。随后用 `sudo vim` 改名。`sudo cat` 核实最终上下都是 `lab-no-notification`。下划线本来也能用，这次不是修语法。

最终配置：

```yaml
route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h
  receiver: lab-no-notification
receivers:
  - name: lab-no-notification
```

```bash
rpm -ql alertmanager | grep '/bin/'
# /usr/bin/alertmanager  /usr/bin/amtool  /usr/bin/prometheus-alertmanager
sudo amtool check-config /etc/prometheus/alertmanager.yml
sudo systemctl enable --now prometheus-alertmanager.service
sudo systemctl status prometheus-alertmanager --no-pager -l
```

`amtool`：`SUCCESS`。Found global config、route、0 条 inhibit rules、1 个 receiver、0 个 templates。

`enable --now` 输出：`Created symlink '/etc/systemd/system/multi-user.target.wants/prometheus-alertmanager.service' → '/usr/lib/systemd/system/prometheus-alertmanager.service'`。`status`：active (running)，08:00:52 CST 起，enabled。监听 `[::]:9093`，没有 TLS。cluster 地址 `10.0.2.15:9094`（VirtualBox NAT）。08:01:03 gossip settled。没有开放 9094。

```bash
curl -i http://localhost:9093/-/ready
curl -sS http://localhost:9090/api/v1/alertmanagers
```

`/-/ready`：`200 OK`，正文 `OK`。`alertmanagers`：`status` success，`activeAlertmanagers` 的 url 是 `http://localhost:9093/api/v2/alerts`，`dropped` 是 `[]`。

用户实际贴出的故障实验：

```bash
sudo systemctl stop prometheus-alertmanager
sudo systemctl stop prometheus-node-exporter
sudo systemctl start prometheus-alertmanager
curl -sS http://localhost:8083/api/v2/alerts
```

8083 的结果是 `301 Moved Permanently`，`nginx/1.26.3`。这是端口写错，不是 Alertmanager 的接口改了。这次也不需要先停 Alertmanager。

```bash
curl -sS http://localhost:9093/api/v2/alerts
```

列表里有：summary `node_exporter 采集失败持续一分钟`，receiver `lab-no-notification`，`status.state` active，`inhibitedBy` / `mutedBy` / `silencedBy` 都是空的。labels：`alertname=NodeExporterDown`，`instance=localhost:9100`，`job=node`，`monitor=example`，`severity=warning`。这是 Alertmanager 收到告警的直接证据。这次没有贴 Prometheus 的 Firing 截图。

`startsAt` 是 `2026-11-23T00:12:04.048Z`（UTC+8 08:12:04）。`endsAt` 是 `2026-11-23T00:31:04.048Z`（UTC+8 08:31:04）。不把 `endsAt` 说成精确的恢复时间，也不说成已经发生过的历史时刻。

```bash
sudo systemctl start prometheus-node-exporter
curl -sS http://localhost:9093/api/v2/alerts
```

恢复后的结果是 `[]`。这表示当前没有活动告警。不是历史被删掉，也不是通知已经送到。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 安装前没有这个包 | `rpm -qa \| grep alertmanager` 和 `grep -i` | 无输出 | ✅ |
| 配置检查 | `sudo amtool check-config /etc/prometheus/alertmanager.yml` | SUCCESS，1 个 receiver，0 条 inhibit | ✅ 有 global config 不等于配了邮件 |
| 服务 | `sudo systemctl status prometheus-alertmanager --no-pager -l` | active，enabled，听 9093 | ✅ 08:00:52 CST 起。9094 未开放 |
| 就绪 | `curl -i http://localhost:9093/-/ready` | 200，正文 OK | ✅ 提示符粘在后面是因为没有换行 |
| Prometheus 发现它 | `curl -sS http://localhost:9090/api/v1/alertmanagers` | active 里有 9093 | ✅ 不是送达证明 |
| 告警进入 AM | `curl -sS http://localhost:9093/api/v2/alerts` | 有 `NodeExporterDown`，receiver 是 `lab-no-notification` | ✅ 没有这次的 Firing 截图 |
| 恢复后当前告警 | 启动 exporter 后再 curl 9093 | `[]` | ✅ 不是历史删除，也不是通知送达 |
| 对外通知、9093 防火墙、Windows 访问 AM | | | 未做 |
| 分组行为实验、静默、抑制 | | | 只讲了，未做 |
| reload、系统重启 | | | 未做。单元支持 reload |

## 踩坑

- `curl -sS http://localhost:8083/api/v2/alerts` → `301 Moved Permanently`，`nginx/1.26.3` → 端口写成了 8083，不是 Alertmanager 改了接口 → 改成 `http://localhost:9093/api/v2/alerts`。
- 故障实验里先停了 Alertmanager 再启动 → 查它收没收到告警，不需要停它 → 后来只停/启 node_exporter。
- `sudo vim` 把 `lab_no_notification` 改成 `lab-no-notification` → 下划线本来有效，不是语法错误 → `sudo cat` 核实 route 和 receiver 的名字一致。

## 卡点 / 没想明白的

- `activeAlertmanagers` 里有 9093，还不能说告警已经到了。这次是停掉 node_exporter 之后，再查 9093 的 `/api/v2/alerts` 才看到那条告警。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，每题 1 分。总分 5/5。没有另编题目。

1. （概念）规则的 `for: 1m` 和 Alertmanager 的 `group_wait: 30s` 差在哪。
<details><summary>批改要点</summary>

规则要持续 1 分钟才触发。同一组等 30 秒再通知。批改补上：`group_wait` 是新分组的第一次等待。1 分。
</details>

2. （概念）`alertname` 相同、`instance` 不同，按这两个标签 `group_by`，会不会进同一组。
<details><summary>批改要点</summary>

不会。1 分。
</details>

3. （排障）`activeAlertmanagers` 里列出 `localhost:9093`，能不能证明已经收到告警。
<details><summary>批改要点</summary>

不能。要等告警触发。批改补：查 Alertmanager 里的具体告警，才算确认收到。1 分。
</details>

4. （概念）接收方没有通知渠道，还会不会接收告警。
<details><summary>批改要点</summary>

会接收。不往外发。1 分。
</details>

5. （操作）检查 Alertmanager 配置的命令。
<details><summary>批改要点</summary>

`sudo amtool check-config /etc/prometheus/alertmanager.yml`。1 分。
</details>

作答结果：5/5。

## 复习记录

起算日是 11-23。这三项都还没做。

- [x] D+1（11-24，1/1（合并复习，五题合计 3.5/5，不是本篇单独 5 题）；activeAlertmanagers 出现 9093 只证明 Prometheus 发现目标，不代表告警已送达 AM）
- [x] D+3（11-26，1/1（合并复习，六题合计 5/6，不是本篇单独 6 题）；`amtool check-config /etc/prometheus/alertmanager.yml`，注：普通用户需加 `sudo`）
- [ ] D+7（11-30）

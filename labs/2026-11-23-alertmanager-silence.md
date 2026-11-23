# Alertmanager 静默：故障还在时不发通知

- 日期：2026-11-23（Asia/Shanghai）
- 考点：运维扩展 —— Alertmanager silence。不是 RHCSA EX200。接收方沿用 [Alertmanager 基础](2026-11-23-alertmanager-basics.md)
- 环境：Rocky Linux 10.2（VirtualBox），node1，用户 fqh。接收方 `lab-no-notification`，没有对外通知渠道
- 状态：静默的生效、提前结束、故障恢复都做了。没有测通知是否发出或送达。监控模块没有全部做完
- 来源：课堂终端记录。归档时没有重跑

## 目标

node_exporter 停着的时候，用 `alertname` 和 `instance` 两条条件把这条告警静默掉，再提前结束静默，最后把 exporter 拉起来。

## 关键概念

- 静默只抑制匹配到的通知。不改 Prometheus 规则，也不修复故障。Prometheus 里的 Firing 不会因为 Alertmanager 的 silence 自动变成 Inactive。
- 两个标签条件同时满足才匹配（AND）。这样维护某一个实例，不会把别的服务的故障也盖住。
- `author` 和 `comment` 是审计说明。`duration=10m` 是有效期，避免静默一直留着。
- `expire` 让静默提前过期，不是把告警删掉。采集器这时还停着，告警会回到 active。提前结束静默不修复故障。
- `state` 为 suppressed 不一定只由静默造成。要看 `silencedBy` 里有没有这次的 ID。这次 `inhibitedBy` 是空的。
- 这次没有通知渠道。`silencedBy` 里出现 ID，只证明静默匹配上了，没有验证外部通知被拦住。
- 没有改 Alertmanager 配置，也没有开放端口。本机接口不用 `sudo`。
- 下面的钟点是对话里记下的时刻，不是精确的状态变化时刻。

## 关键步骤

先停采集器，确认告警在、而且还没被静默。08:54 记下的是 active。

```bash
sudo systemctl stop prometheus-node-exporter
curl -sS http://localhost:9093/api/v2/alerts
```

结果里有 `NodeExporterDown`，`instance=localhost:9100`，`job=node`，`monitor=example`，`severity=warning`，receiver `lab-no-notification`。`inhibitedBy`、`mutedBy`、`silencedBy` 都是空的，`state` 是 active。这次没有贴 Prometheus 的 Firing 截图。

09:04 创建静默。没有用 `sudo`：

```bash
amtool --alertmanager.url=http://localhost:9093 silence add \
  --author=fqh \
  --duration=10m \
  --comment='个人实验：验证维护静默' \
  alertname=NodeExporterDown \
  instance=localhost:9100
```

返回 `61de6ff6-d421-4e95-a2f5-c608b783ef1b`。

09:05 再查：

```bash
curl -sS http://localhost:9093/api/v2/alerts
```

告警还在。`silencedBy` 含有这个 ID，`state` 是 suppressed。`inhibitedBy` 和 `mutedBy` 是空的。

09:09 提前结束。没有等 10 分钟自己过期。这条没有输出：

```bash
amtool --alertmanager.url=http://localhost:9093 silence expire 61de6ff6-d421-4e95-a2f5-c608b783ef1b
curl -sS http://localhost:9093/api/v2/alerts
```

`silencedBy` 回到空，`state` 是 active。采集器这时仍是停的。

09:12 恢复采集器后再查，结果是 `[]`：

```bash
sudo systemctl start prometheus-node-exporter
curl -sS http://localhost:9093/api/v2/alerts
```

`startsAt` / `endsAt` 不抄。不把它们当成故障恢复的精确时间。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 静默前告警在 | 停 exporter 后查 `/api/v2/alerts` | active，三个列表都空 | ✅ 08:54 记下。没有 Firing 截图 |
| 静默匹配 | `silence add` 后再查 | 告警仍在，`silencedBy` 含返回的 ID，suppressed | ✅ 09:04 创建，09:05 查询 |
| 提前结束 | `silence expire` 后再查 | `silencedBy` 空，active。故障还在 | ✅ 09:09。没有输出 |
| 恢复采集器 | `start prometheus-node-exporter` 后再查 | `[]` | ✅ 09:12 |
| 外部通知被静默拦住 | | | 未测。没有通知渠道 |
| 分组通知、抑制规则 | | | 未做 |
| 等满 10 分钟自然过期 | | | 未做。实际是提前 expire |
| 系统重启后静默还在不在 | | | 未验证 |

## 踩坑

无实际报错。命令和查询都按贴出的结果完成。要记住的是：静默不等于把故障恢复了；范围用 `alertname` 加 `instance` 收住，而不是把全部告警静默掉。

## 卡点 / 没想明白的

没有新的技术扣分。5 题都对。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，每题 1 分。总分 5/5。没有另编题目，也没有虚构错题。

1. （概念）加上静默，会让 Prometheus 的 Firing 变成 Inactive 吗。
<details><summary>批改要点</summary>

不会。只是不向外发通知。1 分。
</details>

2. （概念）两个标签条件，满足一个就行，还是都要满足。
<details><summary>批改要点</summary>

都要满足。1 分。
</details>

3. （排障）哪个字段证明就是这次静默匹配上了。
<details><summary>批改要点</summary>

`silencedBy` 里含有这个 ID。1 分。
</details>

4. （排障）`expire` 之后故障还在，为什么又是 active。
<details><summary>批改要点</summary>

静默被提前结束了。故障本身还成立。1 分。
</details>

5. （取舍）为什么静默到具体实例，还要设有效期，而不是长期静默全部告警。
<details><summary>批改要点</summary>

维护有窗口，所以用有效期。维护某个服务时，不能把其他服务的故障也挡住。1 分。
</details>

作答结果：5/5。

## 复习记录

起算日是 11-23。这三项都还没做。

- [ ] D+1（11-24）
- [ ] D+3（11-26）
- [ ] D+7（11-30）

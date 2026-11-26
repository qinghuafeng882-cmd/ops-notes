# Alertmanager 分组：三条测试告警分成两组

- 日期：2026-11-23（Asia/Shanghai）
- 考点：运维扩展 —— Alertmanager `group_by`。不是 RHCSA EX200。配置沿用 [Alertmanager 基础](2026-11-23-alertmanager-basics.md)
- 环境：Rocky Linux 10.2（VirtualBox），node1，用户 fqh。接收方 `lab-no-notification`，没有对外通知渠道
- 状态：`groups` 接口看到 2 组（2 条和 1 条）。没有测通知是否合并或送达。监控模块没有全部做完
- 来源：课堂终端记录。归档时没有重跑

## 目标

不停止服务，用 `amtool` 直接注入带 `lab=grouping` 的 3 条测试告警，看 `group_by: ['alertname', 'instance']` 怎么分组。没有真实故障。配置没有改。

## 关键概念

- `amtool alert add --help` 有 `--start`、`--end`（RFC3339）和 `--annotation`。没有 `--duration`。
- `date -d` 按相对时间算出一个时刻，不改系统时钟。`--iso-8601=seconds` 带时区，精确到秒。`$(...)` 是命令替换。变量要加双引号。行尾的 `\` 是续行，漏掉会提前执行。
- `instance=Lab-node-a` 只是实验用的标签值，大写 L。不表示去连接这台主机。
- 直接 `alert add` 绕过 Prometheus，不受规则里的 `for` 约束。
- annotation 不是 label。至少要有一对 label，否则 400：`at least one label pair required`。
- 当前配置仍是 `group_by: ['alertname', 'instance']`。分组标签里没有 `severity`。`severity` 不同、`alertname` 和 `instance` 相同，仍是同一组，但是两条告警。全部标签都相同，才是同一条。
- 严重级别不同不会自动抑制。这次没有配 `inhibit_rules`。
- `groups` 接口分成 2 组，只说明分组结果。不代表已经发出 2 条合并通知。没有通知渠道，也没有测投递。

## 关键步骤

先算结束时间。写成 `'+20 minites'` 时是 `invalid date`，改成 `minutes` 之后：

```bash
END=$(date -d '+20 minutes' --iso-8601=seconds)
printf '%s\n' "$END"
```

输出 `2026-11-23T09:52:02+08:00`。

前两次 `amtool ... alert add --end="$END"` 都是 400：`at least one label pair required`。第一次续行不完整、没有标签。第二次加了 annotation，仍然没有 label。

补全续行和四个标签之后成功，没有输出。用了 `--end="$END"` 和 annotation `summary=个人实验：告警分组验证`，没有 `--start`：

```text
alertname=LabGroupingTest
instance=Lab-node-a
severity=warning
lab=grouping
```

接着查的是：

```bash
curl -sS 'http://localhost:9093/api/v2/alerts?filter%3Dgrouping'
```

返回 1 条 active。这个 `filter` 写法是错的，不能当成筛选生效。这条的 `startsAt` 和 `endsAt` 都是 09:52:02，fingerprint `3f1a0e410e656836`。起止相同的原因没有定下来。

然后显式指定起止，并把结束时间拉长。实际是分行、带 `\`：

```bash
START=$(date --iso-8601=seconds)
END=$(date -d '+30 minutes' --iso-8601=seconds)
amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$END" \
  --annotation='summary=个人实验：告警分组验证' \
  alertname=LabGroupingTest instance=Lab-node-a severity=warning lab=grouping
```

```bash
curl -sS 'http://localhost:9093/api/v2/alerts?filter=lab%3Dgrouping'
```

1 条 active。`startsAt` `2026-11-23T09:49:10+08:00`，`endsAt` `2026-11-23T10:19:46+08:00`。fingerprint 仍是 `3f1a0e410e656836`。标签全相同，是在更新原来那条。

再用同一个 `START`、`END`、annotation、`alertname` 和 `lab=grouping`，成功且无输出地加了两条：`instance=Lab-node-a severity=critical`，以及 `instance=Lab-node-b severity=warning`。

```bash
curl -sS 'http://localhost:9093/api/v2/alerts/groups?filter=lab%3Dgrouping'
```

2 组，都是 active，receiver 都是 `lab-no-notification`：

- `alertname=LabGroupingTest`、`instance=Lab-node-a`：2 条，warning 和 critical。fingerprint `3f1a0e410e656836`、`7275343b2aebd88f`。
- `instance=Lab-node-b`：1 条 warning。fingerprint `15b5b4ae1d7b8f51`。

清理是用户执行的，没有报错。`FINISH` 的实际值和命令的精确时刻没有贴出：

```bash
FINISH=$(date --iso-8601=seconds)
for PAIR in 'Lab-node-a warning' 'Lab-node-a critical' 'Lab-node-b warning'; do
  read -r INSTANCE SEVERITY <<< "$PAIR"
  amtool --alertmanager.url=http://localhost:9093 alert add \
    --start="$START" --end="$FINISH" \
    --annotation='summary=个人实验：告警分组验证' \
    alertname=LabGroupingTest instance="$INSTANCE" severity="$SEVERITY" lab=grouping
done
```

`<<<` 是 here-string，把 `PAIR` 送给 `read`，不用键盘输入。`read` 按默认 IFS 拆词，第一个词给 `INSTANCE`，第二个给 `SEVERITY`。`-r` 让反斜杠按字面处理。这些变量只是拆开、赋值，不会自己发出请求。

10:20 的对话里查的是 `?fliter=lab%3Dgrouping`，结果 `[]`。`fliter` 拼错了，不算筛选成功。10:28 改成 `filter=lab%3Dgrouping`，结果也是 `[]`。这时已经过了原来的 `endsAt` 10:19:46。可以确认这几条实验告警已经不在。不能单凭 `[]` 说是提前清理起了作用，自然到期和这条命令无法分开。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 同标签更新 | `filter=lab%3Dgrouping` | 1 条，fingerprint 不变 | ✅ `3f1a0e410e656836`，09:49:10 到 10:19:46 |
| 分组 | `.../alerts/groups?filter=lab%3Dgrouping` | 2 组，2 条和 1 条 | ✅ 都是 active，receiver `lab-no-notification` |
| 严重级别自动抑制 | | | 未配置 `inhibit_rules`。critical 和 warning 在同一组 |
| 发出合并通知 | | | 未测。没有通知渠道 |
| 错误的 `filter%3Dgrouping` | | 能当筛选结果 | ❌ 返回了 1 条，但不能当成筛选生效 |
| `fliter` | 10:20 的查询 | | 拼写错误。`[]` 不算筛选成功 |
| 纠正后的 filter | 10:28 | `[]` | ✅ 告警已不在。不能区分自然到期和命令清理 |
| 系统重启 | | | 未验证 |

## 踩坑

- `date -d '+20 minites'` → `invalid date` → 改成 `minutes`。
- `amtool alert add --end="$END"` 两次都 400：`at least one label pair required` → 续行不完整，而且没有 label。annotation 不是 label → 补上四个标签后成功，无输出。
- `instance` 写成 `Lab-node-a`，L 是大写。这是实际用的值，不是主机名。
- `curl ...?filter%3Dgrouping` 返回 1 条 active → `filter` 格式错了，不能当成筛选生效。这条 `startsAt` 和 `endsAt` 都是 09:52:02。为什么起止相同，没有定下来。
- 10:20 写成 `fliter`，得到 `[]` → 拼写错误，不算筛选成功 → 10:28 改成 `filter=lab%3Dgrouping`，才是 `[]`。
- 清理之后已经过了 10:19:46 → 不能单凭 `[]` 说提前清理生效。实验告警确实没了，自然到期和命令清理分不开。`FINISH` 的值和精确执行时刻没有贴出。

## 卡点 / 没想明白的

`date` 和 `read` 已经讲过，下次复习还要再看。这不表示已经能独立写熟。课后 5 题没有新的扣分。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，每题 1 分。总分 5/5。没有另编题目。

1. （概念）`severity` 不同，为什么还在同一组。
<details><summary>批改要点</summary>

分组标签里没有 `severity`。1 分。
</details>

2. （概念）同一组是不是同一条告警。
<details><summary>批改要点</summary>

不是。全部标签相同才是同一条。1 分。
</details>

3. （操作）直接 `amtool alert add` 要不要等 `for: 1m`。
<details><summary>批改要点</summary>

不要。它绕过 Prometheus。1 分。
</details>

4. （排障）`groups` 显示 2 组，是不是已经发出 2 条通知。
<details><summary>批改要点</summary>

不能这么说。1 分。
</details>

5. （概念）`<<< "$PAIR"` 做什么。
<details><summary>批改要点</summary>

here-string，把这个字符串直接送给命令，不用键盘输入。1 分。
</details>

作答结果：5/5。

## 复习记录

起算日是 11-23。这三项都还没做。

- [ ] D+1（未做，由 D+3 一并覆盖）
- [x] D+3（11-26，1/1（合并复习，五题合计 5/5，不是本篇单独 5 题，含 D+1 一并覆盖）；`group_by` 不含 `severity`，同 `alertname` 与 `instance` 会进同一组但仍是两条独立告警）
- [ ] D+7（11-30）

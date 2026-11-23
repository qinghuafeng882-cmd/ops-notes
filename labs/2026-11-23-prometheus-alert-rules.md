# Prometheus 告警规则：Inactive、Pending、Firing

- 日期：2026-11-23（Asia/Shanghai）
- 考点：运维扩展 —— Prometheus 告警规则。不是 RHCSA EX200
- 环境：Rocky Linux 10.2（VirtualBox），node1 192.168.56.11，用户 fqh。Windows 只用来打开 `http://192.168.56.11:9090/alerts`
- 状态：规则状态做完。没有配 Alertmanager，没有验证通知。监控模块没有全部做完
- 来源：课堂终端记录和用户截图。归档时没有重跑

## 目标

写一条 `up{job="node"} == 0` 持续 1 分钟的规则，在页面上看到 Inactive、Pending、Firing，再恢复成 Inactive。不发通知。

## 关键概念

- `for: 1m` 是条件要连续成立的时间。`evaluation_interval: 15s` 是多久评估一次。`for` 不是采集周期。
- 条件在 `for` 走完之前恢复，计时重新算。下次评估应回到 Inactive。这节只讲了，没有做一次「短暂失败再恢复」来验证 Pending 被清掉。
- `up{job="node"} == 0` 只表示最近这次采集失败。不能当成整机宕机。
- `severity: warning` 只是标签。
- Firing 只说明规则已经触发。不说明通知已经送到。`alerting` 里的 `localhost:9093` 仍只是预设地址，不证明 Alertmanager 在跑。
- `rule_files` 不会自动扫目录里的全部文件，必须写上要加载的路径。
- `systemctl status -l` 就是 `--full`，长行不省略。`--no-pager` 不进分页器。
- 页面上的状态可以直接当故障和恢复的证据。截图里的 active since 不能当成精确的触发延迟。

用户贴出的 `/etc/prometheus/prometheus.yml`：全局 `scrape_interval` 15s、`evaluation_interval` 15s。job `prometheus` 的目标是 `localhost:9090`，间隔覆盖成 5s。job `node` 的目标是 `localhost:9100`，间隔沿用 15s。改规则之前，`rule_files` 全是注释。

## 关键步骤

```bash
sudo cp -a /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak-$(date +%Y%m%d-%H%M%S)
```

没有报错。生成出来的备份文件名没有另贴。

用户不用 vim 新建规则文件，用的是：

```bash
sudo tee /etc/prometheus/node_alerts.yml << 'EOF'
```

写进去的内容：

```yaml
groups:
  - name: node_alerts
    rules:
      - alert: NodeExporterDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "node_exporter 采集失败持续一分钟"
```

```bash
sudo promtool check rules /etc/prometheus/node_alerts.yml
# SUCCESS: 1 rules found
sudo vim /etc/prometheus/prometheus.yml
# rule_files 改为引用绝对路径 /etc/prometheus/node_alerts.yml
sudo promtool check config /etc/prometheus/prometheus.yml
# SUCCESS: 1 rule files found；valid config syntax；rule SUCCESS 1
sudo systemctl restart prometheus
sudo systemctl status prometheus --no-pager -l
```

`status`：active (running)，07:01:58 CST 起。07:02:00 的日志有 Completed loading configuration、ready、Starting rule manager。

Windows 打开 `http://192.168.56.11:9090/alerts`：

- 07:06 截图是 Inactive。规则路径、`expr`、`for` 是对的。
- 然后指导在 node1 执行 `sudo systemctl stop prometheus-node-exporter`。这条的终端输出没有贴，不把输出写成看见了。
- 07:09 截图是 Pending。实例 `localhost:9100`，active since 19.282s，value 0。
- 07:11 截图是 Firing。active since 1m50.299s，value 0。
- 然后指导 `sudo systemctl start prometheus-node-exporter`。终端输出没有贴。
- 07:14 截图回到 Inactive。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 规则语法 | `sudo promtool check rules /etc/prometheus/node_alerts.yml` | SUCCESS，1 条规则 | ✅ |
| 主配置加载这条规则 | `sudo promtool check config /etc/prometheus/prometheus.yml` | 1 个规则文件，语法有效 | ✅ |
| Prometheus 读到新配置 | `sudo systemctl status prometheus --no-pager -l` | active，并出现 loading configuration / rule manager | ✅ 07:01:58 起，07:02:00 的日志 |
| 停 exporter 前 | 浏览器 `/alerts` | Inactive | ✅ 07:06 截图 |
| 停 exporter 后 | 同上 | Pending，再到 Firing，value 0 | ✅ 07:09、07:11 截图。stop 的终端输出没贴 |
| 再启动 exporter | 同上 | Inactive | ✅ 07:14 截图。start 的终端输出没贴 |
| 截图时长等于精确触发延迟 | | | 不能这样用 |
| Pending 被中途恢复清掉 | | | 只讲了，未验证 |
| 通知送达 | | | 未配置，未验证 |
| 系统重启后规则仍在 | | | 未验证 |

## 踩坑

配置和两次 `promtool` 都没有报错。不编造语法错误。

## 卡点 / 没想明白的

- `systemctl status -l` 已讲过：`--full`，长行不省略。
- 刚解释完避免波动，就让用户复述。用户指出这一步多余。之后改为直接讲、直接做实验。这不是用户的技术错误。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，每题 1 分。总分 5/5。没有为了凑题型另编。

1. （概念）`for: 1m` 和 `evaluation_interval: 15s` 各是什么。
<details><summary>批改要点</summary>

`for` 是条件要持续成立的时间。15s 是评估频率。答对，并补了中途恢复要重新计时。1 分。
</details>

2. （排障）`up == 0` 能不能证明整机宕机。
<details><summary>批改要点</summary>

不能。只表示最近这次采集失败。1 分。
</details>

3. （排障）已经 Pending，采集又恢复了，会怎样。
<details><summary>批改要点</summary>

计时重置，下次评估是 Inactive。1 分。这是讲解，没有用一次短暂失败做实验。
</details>

4. （概念）Firing 是不是通知已经送到。
<details><summary>批改要点</summary>

不是。只说明规则已经触发。1 分。
</details>

5. （操作）为什么 `rule_files` 要写上这个新文件。
<details><summary>批改要点</summary>

不会自动扫描全部规则文件，必须显式引用。1 分。
</details>

作答结果：5/5。

## 复习记录

起算日是 11-23。这三项都还没做。

- [x] D+1（11-24，1/1（合并复习，五题合计 3.5/5，不是本篇单独 5 题）；Pending 中途恢复回到 Inactive，补明 for 计时会清零重算）
- [ ] D+3（11-26）
- [ ] D+7（11-30）

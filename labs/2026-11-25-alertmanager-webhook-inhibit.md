# Alertmanager 真实通知渠道（Webhook）与抑制规则

- 日期：2026-11-25（Asia/Shanghai）
- 考点：运维扩展 —— Alertmanager `webhook_configs`、`inhibit_rules`、`send_resolved` 与 Shell 重定向文件偏移。不是 RHCSA EX200
- 环境：Rocky Linux 10.2（VirtualBox），node1（192.168.56.11，Prometheus / Alertmanager 0.34.2 / Python 3.12.13），用户 fqh
- 状态：Webhook 真实通知（`firing` / `resolved`）与 `inhibit_rules` 压制通知均已实测通过；收尾已切回 `lab-no-notification` 并保留 `inhibit_rules`（未接入邮箱或企业 IM 外部账号；未做系统重启验证）
- 来源：课堂终端记录。归档时没有重跑

## 目标

在 node1 上用本地 Python HTTP 接收端打通 Alertmanager `webhook_configs` 真实通知（验证 `firing` 与 `resolved` 报文），并配置 `inhibit_rules` 实测同一 `instance` 上 `critical` 告警动态压制 `warning` 告警、放行异实例告警。

## 关键概念

- `webhook_configs` 通过 HTTP POST 发送 Alertmanager 标准 JSON 载荷；`send_resolved: true`（默认开启）会在告警恢复时补发 `status: "resolved"` 通知，形成闭环。
- 钉钉、飞书、企业微信等 IM 机器人通常不能直接填进 `webhook_configs`：外部平台要求各自的 JSON 字段格式，生产上一般需加一层轻量转换适配器。
- `inhibit_rules` 三要素：`source_matchers`（触发抑制的源告警条件）、`target_matchers`（被抑制的目标告警条件）、`equal`（双方必须取值相同的标签列表，如 `['instance']`）。Alertmanager v0.22.0 起已废弃旧字段 `source_match` / `target_match`。
- 同样显示 `state: "suppressed"`，需看 `/api/v2/alerts` 的 **`inhibitedBy`**（被抑制规则压住）与 **`silencedBy`**（被人工静默压住）区分；`inhibit_rules` 由源告警动态触发并在源告警恢复后自动解除，而 `silence` 由人工按时间窗口手动设定。
- node1 上 `prometheus-alertmanager.service` 运行在 SELinux `unconfined_service_t` 域，无需额外开 SELinux 布尔值即可连接本机 `127.0.0.1:19099`。
- `>` 重定向的截断清空只在命令启动前由 bash 执行一次（`O_TRUNC`）；常驻后台进程运行期间沿同一个打开的文件描述符及文件偏移量持续写入，因此多次回调日志会按顺序留在文件里而不会覆盖。
- `pkill -f` 按完整命令行匹配进程（而不只匹配进程名 `python3`），避免误杀其他无关 Python 进程。

## 关键步骤

```bash
# —— 检查 node1 服务现状、配置、SELinux 上下文与本地端口 ——
systemctl is-active prometheus.service prometheus-node-exporter.service prometheus-alertmanager.service
# active / active / active
sudo cat /etc/prometheus/alertmanager.yml        # 原配置为 receiver: lab-no-notification
ps -efZ | grep '[a]lertmanager'
# system_u:system_r:unconfined_service_t:s0 prometh+ 1572 1 ... /usr/bin/prometheus-alertmanager
sudo ss -lntp 'sport = :19099'                   # 无监听
python3 --version                                # Python 3.12.13
```

```bash
# —— 启动本地 Python Webhook 接收端并做烟雾测试 ——
cat << 'EOF' > /tmp/webhook_receiver.py
import json
from datetime import datetime
from http.server import BaseHTTPRequestHandler, HTTPServer
class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(length).decode("utf-8", errors="replace")
        print(f"\n=== [{datetime.now().strftime('%H:%M:%S')}] POST {self.path} ===", flush=True)
        try:
            data = json.loads(body)
            print(json.dumps(data, ensure_ascii=False, indent=2), flush=True)
        except Exception:
            print(body, flush=True)
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok\n")
    def log_message(self, format, *args):
        pass
HTTPServer(("127.0.0.1", 19099), Handler).serve_forever()
EOF

python3 /tmp/webhook_receiver.py > /tmp/webhook.log 2>&1 &     # [1] 15063
sudo ss -lntp 'sport = :19099'                                   # LISTEN 127.0.0.1:19099 pid=15063
curl -fsS -X POST http://localhost:19099/test -d '{"msg":"hello"}'
# ok
cat /tmp/webhook.log
# === [09:59:05] POST /test === {"msg": "hello"}
```

```bash
# —— 备份并更新 alertmanager.yml（实验阶段缩短 group_wait/group_interval 为 10s）与热重载验证 ——
sudo cp -a /etc/prometheus/alertmanager.yml /etc/prometheus/alertmanager.yml.bak-$(date +%Y%m%d-%H%M%S)
systemctl show -p MainPID --value prometheus-alertmanager        # 1572

sudo tee /etc/prometheus/alertmanager.yml << 'EOF'
route:
  group_by: ['alertname', 'instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 3h
  receiver: lab-webhook

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: ['instance']

receivers:
  - name: lab-webhook
    webhook_configs:
      - url: 'http://127.0.0.1:19099/alerts'
        send_resolved: true
EOF

sudo amtool check-config /etc/prometheus/alertmanager.yml        # SUCCESS: 1 inhibit rules, 1 receivers
sudo systemctl reload prometheus-alertmanager.service
systemctl show -p MainPID --value prometheus-alertmanager.service # 仍为 1572
sudo journalctl -u prometheus-alertmanager.service -n 15 --no-pager
# 10:25:21 systemd Reloaded 先返回；10:25:22.011 Loading -> 10:25:22.042 Completed loading of configuration file
```

```bash
# —— 停/启 node1 采集器，验证真实 firing 与 resolved Webhook 报文 ——
sudo systemctl stop prometheus-node-exporter.service
sleep 105
curl -sS http://localhost:9093/api/v2/alerts
# NodeExporterDown (localhost:9100), startsAt=2026-11-25T02:36:19.048Z (10:36:19+08:00), receiver=lab-webhook, state=active
cat /tmp/webhook.log
# === [10:36:29] POST /alerts === (恰好等待 group_wait 10s)
# status="firing", notification_reason="first notification", endsAt="0001-01-01T00:00:00Z"

sudo systemctl start prometheus-node-exporter.service
sleep 35
curl -sS http://localhost:9093/api/v2/alerts
# []
cat /tmp/webhook.log
# === [10:51:09] POST /alerts ===
# status="resolved", notification_reason="all alerts resolved", endsAt="2026-11-25T02:51:04.048Z"
```

```bash
# —— 注入 3 条测试告警验证 inhibit_rules（同实例抑制、异实例放行） ——
START=$(date --iso-8601=seconds)
END=$(date -d '+10 minutes' --iso-8601=seconds)

amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$END" \
  --annotation='summary=个人实验：节点宕机（源告警）' \
  alertname=LabNodeDown instance=Lab-node-a severity=critical lab=inhibit

amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$END" \
  --annotation='summary=个人实验：磁盘告警（同实例应被抑制）' \
  alertname=LabDiskWarn instance=Lab-node-a severity=warning lab=inhibit

amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$END" \
  --annotation='summary=个人实验：磁盘告警（异实例不应被抑制）' \
  alertname=LabDiskWarn instance=Lab-node-b severity=warning lab=inhibit

sleep 15
curl -sS -w '\n' 'http://localhost:9093/api/v2/alerts?filter=lab%3Dinhibit'
# 3 条均在：LabNodeDown(Lab-node-a, 9eafd498ab10dcfc) active；
# LabDiskWarn(Lab-node-a, 951e63c504281476) suppressed, inhibitedBy=["9eafd498ab10dcfc"]；
# LabDiskWarn(Lab-node-b, 5eb6394ae23d0cd7) active
grep -E '===|"status":|"alertname":|"instance":|"severity":' /tmp/webhook.log
# [11:08:46] 仅收到 LabNodeDown(Lab-node-a) 与 LabDiskWarn(Lab-node-b) 的 firing；无 LabDiskWarn(Lab-node-a)
```

```bash
# —— 源告警于 11:18:36 自然到期后，续期验证次要告警不再被抑制并正常外发 ——
NOW=$(date --iso-8601=seconds)
NEW_END=$(date -d '+10 minutes' --iso-8601=seconds)

amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$NEW_END" \
  --annotation='summary=个人实验：磁盘告警（源恢复后应解除抑制并外发）' \
  alertname=LabDiskWarn instance=Lab-node-a severity=warning lab=inhibit
amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$NOW" \
  --annotation='summary=个人实验：节点宕机（源告警）' \
  alertname=LabNodeDown instance=Lab-node-a severity=critical lab=inhibit
amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$NOW" \
  --annotation='summary=个人实验：磁盘告警（异实例不应被抑制）' \
  alertname=LabDiskWarn instance=Lab-node-b severity=warning lab=inhibit

sleep 15
curl -sS -w '\n' 'http://localhost:9093/api/v2/alerts?filter=lab%3Dinhibit'
# 仅剩 LabDiskWarn(Lab-node-a): state="active", inhibitedBy=[]
grep -E '===|"status":|"alertname":|"instance":|"severity":' /tmp/webhook.log | tail -n 45
# [11:18:36] LabNodeDown 与 LabDiskWarn(Lab-node-b) 已在自然到期时发过 resolved；
# [11:28:09] LabDiskWarn(Lab-node-a) 发出 status="firing"
```

```bash
# —— 收尾清理：结束测试告警、切回 lab-no-notification（保留 inhibit_rules）、停掉测试接收端 ——
NOW=$(date --iso-8601=seconds)
amtool --alertmanager.url=http://localhost:9093 alert add \
  --start="$START" --end="$NOW" \
  --annotation='summary=个人实验：磁盘告警（源恢复后应解除抑制并外发）' \
  alertname=LabDiskWarn instance=Lab-node-a severity=warning lab=inhibit

sudo tee /etc/prometheus/alertmanager.yml << 'EOF'
route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h
  receiver: lab-no-notification

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: ['instance']

receivers:
  - name: lab-no-notification
EOF

sudo amtool check-config /etc/prometheus/alertmanager.yml
sudo systemctl reload prometheus-alertmanager.service
pkill -f '/tmp/webhook_receiver.py'
rm -f /tmp/webhook_receiver.py /tmp/webhook.log
sudo ss -lntp 'sport = :19099'
curl -sS -w '\n' http://localhost:9093/api/v2/alerts              # []
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| SELinux 上下文与测试端监听 | `ps -efZ`、`ss -lntp 'sport = :19099'`、`curl -X POST .../test` | `unconfined_service_t`、监听 `127.0.0.1:19099`、返回 `ok` | ✅ `pid=15063`，日志打印 JSON |
| 配置检查与 `SIGHUP` 热重载 | `amtool check-config`、`systemctl reload` 前后 `MainPID` 与 `journalctl` | `SUCCESS`、`MainPID` 不变、日志显示完成加载 | ✅ `1572` 未变，`31ms` 加载完成 |
| 真实 `firing` 与 `resolved` 通知 | 停/启 `prometheus-node-exporter` 后查 `/api/v2/alerts` 与 `/tmp/webhook.log` | 停后 10s 发 `firing`，启后恢复并发 `resolved` | ✅ `10:36:29` firing、`10:51:09` resolved |
| `inhibit_rules` 同实例抑制与异实例放行 | 注入 3 条告警后查 `api/v2/alerts` 与 `/tmp/webhook.log` | `Lab-node-a` warning `suppressed` 且无 Webhook；另外两条均发 | ✅ `inhibitedBy=["9eafd498ab10dcfc"]` |
| 源告警不再触发后次要告警外发 | `LabNodeDown` 结束且 `LabDiskWarn(Lab-node-a)` 续期后查状态与日志 | `inhibitedBy=[]` 且补发 `firing` | ✅ `11:28:09` 补发（源告警已于 `11:18:36` 自然到期） |
| `11:18:36` 前提前结束源告警触发实时解除 | 在原 `endsAt` 前由命令提前结束源告警 | 立刻解除并外发 | 未单独验证（操作时已过 `11:18:36`） |
| 现场清理与空队列恢复 | 切回 `lab-no-notification`、`pkill -f`、查 `19099` 与 `/api/v2/alerts` | 端口释放、队列 `[]` | ✅ `Terminated`、端口空、`[]` |
| 系统重启后配置持久化 | 重启虚机后复测 | 规则仍在 | 未验证 |

## 踩坑

- `sudo journalctl ... --no-paper` → `unrecognized option '--no-paper'` → 把 `g` 误敲成 `p` → 改为 `--no-pager`。
- `sudo systemctl start prometheus-node-exportor.service` → `Unit prometheus-node-exportor.service not found.` → `er` 误敲成 `or` → 改为 `prometheus-node-exporter.service`。
- 11:28 步执行后困惑「为什么另外两条没有在 11:28 发 `resolved`」→ 11:08:36 注入时设了 `+10 minutes`，两条告警已在 **`11:18:36`** 自然到期并提前发过 `resolved`（日志中已有两条 `[11:18:36]` 记录），不会重复发送。

## 卡点 / 没想明白的

- 课堂追问了 `ps -efZ`、`curl -X POST -d`、`>` 启动后台进程为何不覆盖后续写入、`pkill -f`，以及测试用的 Python 接收脚本是否需要搞懂（按「主次裁决」先不深究脚本语法，仅知其作本地 Webhook 测试端即可）。
- 课后题第 2 题卡住：`inhibit_rules` 与 `silence` 都会使告警变成 `suppressed` 且不发通知，但没答出两者在「由另一条正在触发的源告警动态触发并随其自动解除」与「人工按时间窗口设置」上的本质区别。

## 面试题（5 道，对话中作答后批改）

1. （概念）为什么通常不能把钉钉、飞书或企业微信机器人的地址直接填进 Alertmanager 的 `webhook_configs` 就指望它正常推消息？
<details><summary>批改要点</summary>

Alertmanager 发出的标准 JSON 格式与钉钉/飞书/企业微信机器人要求的消息体格式不兼容，需要中间加一层格式转换适配器。1 分。
</details>

2. （概念）同样都会让告警状态变成 `suppressed`，`inhibit_rules`（抑制规则）和人工 `silence`（静默）最核心的区别是什么？
<details><summary>批改要点</summary>

`silence` 由人工按标签条件加固定有效期手动设置，到期或手动 `expire` 前一直压制；`inhibit_rules` 由另一条正在触发的源告警动态触发，且源告警一旦恢复，抑制自动解除并放行次要告警。0 分（未答出区别）。
</details>

3. （原理）用 `python3 /tmp/webhook_receiver.py > /tmp/webhook.log 2>&1 &` 启动常驻进程后，为什么连续收到多次告警推送时，旧日志没有被 `>` 清空覆盖？
<details><summary>批改要点</summary>

`>` 的截断清空只在进程启动前由 bash 执行一次；常驻 Python 进程运行期间一直使用同一个已打开的文件描述符，按当前文件偏移量持续往后写，不会再触发清空。1 分。
</details>

4. （应用）在 `inhibit_rules` 里配了 `severity="critical"` 抑制 `severity="warning"` 时，如果漏写 `equal: ['instance']` 会出什么问题？
<details><summary>批改要点</summary>

不同 `instance` 上的告警也会互相抑制，导致一台主机出 `critical` 时把其他无关主机的 `warning` 全部误压制（误伤）。1 分。
</details>

5. （排障）当 `/api/v2/alerts` 里显示某条告警为 `state: "suppressed"` 时，查看返回 JSON 中的哪些字段可以立刻区分它是被抑制规则压住了还是被人工静默了？
<details><summary>批改要点</summary>

看 `inhibitedBy`（非空为抑制）和 `silencedBy`（非空为静默）。1 分（提醒：JSON 实际字段为驼峰大小写 `inhibitedBy` / `silencedBy`，要点全对不扣分）。
</details>

作答结果：4/5。最薄弱点：第 2 题未答出 `inhibit_rules`（由源告警动态触发与自动解除）与 `silence`（人工设时间窗口）的本质区别。

## 复习记录

起算日是 11-25。这三项都还没做。

- [ ] D+1（11-26）
- [ ] D+3（11-28）
- [ ] D+7（12-02）

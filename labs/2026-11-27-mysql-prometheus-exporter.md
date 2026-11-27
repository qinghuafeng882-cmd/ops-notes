# MySQL Exporter：mysqld_exporter 二进制托管与业务指标采集

- 日期：2026-11-27（Asia/Shanghai）
- 考点：运维扩展 —— `mysqld_exporter` 协议适配、最小权限数据库监控账号与 `.my.cnf` 凭据隔离、`up`/`mysql_up` 双层存活边界。不是 RHCSA EX200
- 环境：Rocky Linux 10（VirtualBox），node1（192.168.56.11，MySQL 8.4.11 / Prometheus 3.15.1 / mysqld_exporter 0.21.0），用户 fqh
- 状态：官方二进制部署、最小权限账号与 `0600` 凭据隔离、systemd 托管（`127.0.0.1:9104`）、热重载入库、`rate()` QPS 计算、停 `mysqld` 时 `up=1` 与 `mysql_up=0` 分离均已完成并恢复（未配 `mysql_up == 0` 告警规则与 Grafana MySQL 面板；未接入 node2 从库监控；未做系统重启验证）
- 来源：课堂终端记录。归档时没有重跑

## 目标

在 node1 部署官方 `mysqld_exporter`（v0.21.0）并用 `systemd` 托管在 `127.0.0.1:9104`，通过最小权限数据库账号与 `0600` 权限 `.my.cnf` 文件隔离连接凭据，接入 Prometheus 采集后验证 `rate()` 算 QPS 及「停 `mysqld` 但 Exporter 存活」的指标差异。

## 关键概念

- **协议与暴露差异**：Nginx 内置 `stub_status` HTTP 端点；MySQL 无内置 HTTP `/metrics` 接口，只说 MySQL 协议（`3306` / socket）。`mysqld_exporter` 充当协议适配器，收到 Prometheus 的 `GET /metrics` 后连接 MySQL 执行状态查询并翻译成指标输出（默认端口 `9104`）。
- **最小权限与连接保护**：建专用账号 `'exporter'@'127.0.0.1'`，仅授 `PROCESS`、`REPLICATION CLIENT`、`SELECT ON *.*`，并附加 `WITH MAX_USER_CONNECTIONS 3` 防止数据库卡顿雪崩时监控查询占满连接槽位；MySQL 8.4 执行 `CREATE USER`/`GRANT` 立即生效，无需 `FLUSH PRIVILEGES`。
- **`v0.15+` 配置方式与凭据隔离**：`mysqld_exporter` 自 `v0.15.0` 起已移除旧版 `DATA_SOURCE_NAME` 环境变量；同时不能把密码明文写进 `ExecStart`（`ps` 可见）。标准做法是将 `[client]` 凭据写入 `0600` 权限、属主 `mysqld-exporter:mysqld-exporter` 的 `/etc/mysqld_exporter.cnf`（用 `sudo vim` 编辑避免密码进 bash 历史），并通过 `--config.my-cnf` 传入。
- **`install` vs `mv` 的 SELinux 差异**：`install` 在 `/usr/local/bin/` 下新建文件并按目标目录规则继承 `bin_t`（不像 `mv` 会保留原文件的 `user_tmp_t` 标签）；日常仍保留 `restorecon -v` 作兜底检查。
- **`[Unit]` 依赖取舍**：`mysqld-exporter.service` 只写 `After=network-online.target mysqld.service`，不写 `Requires=mysqld.service`——否则 `mysqld` 挂掉时 `systemd` 会连带停掉 `mysqld-exporter`，导致 Prometheus 抓不到 `9104`（`up` 变 `0`、`mysql_up` 断采无新 `0` 样本）。
- **`v0.21.0` 自身指标变化与 `PromQL info`**：`v0.21.0` 无 `mysql_exporter_last_scrape_error`，改以 `mysql_exporter_collector_duration_seconds` 与 `mysql_exporter_collector_success` 配合 `mysql_up` 观测；用 `rate(mysql_global_status_queries{job="mysql"}[1m])` 算 QPS 时，因指标名未以 `_total` 结尾，Prometheus 3.x 会附带一条 `PromQL info` 提示，不影响正常计算。
- **双层存活边界**：`up{job="mysql"}` 仅反映 Prometheus 能否拉通 `127.0.0.1:9104`；`mysql_up{job="mysql"}` 才反映 Exporter 能否连上 MySQL。停 `mysqld` 而 Exporter 仍在跑时，`up` 仍为 `1`，只有 `mysql_up` 变为 `0`。

## 关键步骤

```bash
# —— node1 检查 MySQL 基线、9104 空闲并查询 GitHub 最新 Release ——
systemctl is-active mysqld                       # active
mysql --version                                  # mysql Ver 8.4.11（客户端版本；确认服务端看 mysqld --version 或 SELECT VERSION()）
sudo ss -lntp 'sport = :3306'                    # LISTEN *:3306 pid=1935
sudo ss -lntp 'sport = :9104'                    # 空闲
curl -fsSL https://api.github.com/repos/prometheus/mysqld_exporter/releases/latest | grep -E '"tag_name"|linux-amd64\.tar\.gz'
# "tag_name": "v0.21.0"
```

```bash
# —— 下载官方 v0.21.0 并先 tar -tzf 核对结构再安装 ——
mkdir -p /tmp/mysql-exp-dl
curl -fL --connect-timeout 10 -o /tmp/mysql-exp-dl/exporter.tar.gz \
  https://github.com/prometheus/mysqld_exporter/releases/download/v0.21.0/mysqld_exporter-0.21.0.linux-amd64.tar.gz
tar -tzf /tmp/mysql-exp-dl/exporter.tar.gz       # 外层带 mysqld_exporter-0.21.0.linux-amd64/ 子目录

tar -xzf /tmp/mysql-exp-dl/exporter.tar.gz -C /tmp/mysql-exp-dl
sudo install -m 0755 -o root -g root /tmp/mysql-exp-dl/mysqld_exporter-0.21.0.linux-amd64/mysqld_exporter /usr/local/bin/mysqld_exporter
sudo restorecon -v /usr/local/bin/mysqld_exporter
ls -lZ /usr/local/bin/mysqld_exporter            # system_u:object_r:bin_t:s0
/usr/local/bin/mysqld_exporter --version         # mysqld_exporter, version 0.21.0 (go1.27.3, linux/amd64)
/usr/local/bin/mysqld_exporter --help 2>&1 | grep -E 'config\.my-cnf|mysqld\.|web\.listen-address'
# --config.my-cnf=".my.cnf" / --mysqld.address="localhost:3306" / --web.listen-address=:9104

sudo useradd --system --no-create-home --shell /sbin/nologin mysqld-exporter
rm -rf /tmp/mysql-exp-dl
```

```sql
-- —— 在 node1 MySQL 中创建最小权限监控账号（密码不入库） ——
-- mysql -u root -p
CREATE USER 'exporter'@'127.0.0.1' IDENTIFIED BY '...' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'127.0.0.1';
SHOW GRANTS FOR 'exporter'@'127.0.0.1';
-- GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.* TO `exporter`@`127.0.0.1`
```

```bash
# —— 建 0600 凭据文件并切换 mysqld-exporter 身份实测直连 ——
sudo install -m 0600 -o mysqld-exporter -g mysqld-exporter /dev/null /etc/mysqld_exporter.cnf
sudo vim /etc/mysqld_exporter.cnf
# 写入 [client]、user = exporter、password = ...

ls -lZ /etc/mysqld_exporter.cnf
# -rw-------. 1 mysqld-exporter mysqld-exporter system_u:object_r:etc_t:s0 44
sudo -u mysqld-exporter mysql --defaults-extra-file=/etc/mysqld_exporter.cnf -h 127.0.0.1 -e 'SELECT CURRENT_USER(), VERSION();'
# exporter@127.0.0.1 | 8.4.11
```

```bash
# —— 配置 systemd 托管在 127.0.0.1:9104 并验证 /metrics ——
sudo tee /etc/systemd/system/mysqld-exporter.service >/dev/null << 'EOF'
[Unit]
Description=Prometheus MySQL Exporter
After=network-online.target mysqld.service
Wants=network-online.target

[Service]
User=mysqld-exporter
Group=mysqld-exporter
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/etc/mysqld_exporter.cnf \
  --mysqld.address=127.0.0.1:3306 \
  --web.listen-address=127.0.0.1:9104
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mysqld-exporter.service
systemctl is-active mysqld-exporter.service       # active
sudo ss -lntp 'sport = :9104'                    # LISTEN 127.0.0.1:9104 pid=164538
ps -efZ | grep '[m]ysqld_exporter'               # unconfined_service_t

curl -fsS http://127.0.0.1:9104/metrics | grep -E '^(mysql_up|mysql_exporter_last_scrape_error|mysql_global_status_(threads_connected|threads_running|questions|queries|slow_queries|uptime))'
# queries 70 / questions 70 / slow_queries 4 / threads_connected 3 / threads_running 4 / uptime 183175 / mysql_up 1
curl -fsS http://127.0.0.1:9104/metrics | grep -E '^# TYPE mysql_exporter_'
# mysql_exporter_collector_duration_seconds gauge / mysql_exporter_collector_success gauge
```

```bash
# —— 接入 Prometheus 抓取并实证 SIGHUP 热重载 ——
sudo cp -a /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak-$(date +%Y%m%d-%H%M%S)
systemctl show -p MainPID --value prometheus.service   # 1575

sudo tee -a /etc/prometheus/prometheus.yml >/dev/null << 'EOF'
  - job_name: mysql
    static_configs:
      - targets: ['127.0.0.1:9104']
EOF

sudo promtool check config /etc/prometheus/prometheus.yml   # SUCCESS: valid prometheus config file syntax
sudo systemctl reload prometheus.service
systemctl show -p MainPID --value prometheus.service        # 仍为 1575
sudo journalctl -u prometheus.service -n 10 --no-pager      # Completed loading of configuration file (totalDuration=111.336548ms)

sleep 20
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="mysql"}' -w '\n'                                  # "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_up{job="mysql"}' -w '\n'                            # "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_global_status_queries{job="mysql"}' -w '\n'         # "371"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_global_status_threads_connected{job="mysql"}' -w '\n' # "3"
```

```bash
# —— rate() 算 QPS 与停 mysqld 故障边界实测 ——
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=rate(mysql_global_status_queries{job="mysql"}[1m])' -w '\n'
# "1.3333925952264545"（附 PromQL info：指标名未以 _total 等结尾）

sudo systemctl stop mysqld
sleep 20
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="mysql"}' -w '\n'       # 仍为 "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_up{job="mysql"}' -w '\n' # 变为 "0"

sudo systemctl start mysqld
sleep 20
systemctl is-active mysqld mysqld-exporter.service                                                          # active / active
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=mysql_up{job="mysql"}' -w '\n' # 恢复 "1"
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 基线与 GitHub Release | `is-active mysqld`、`ss`、`curl ... releases/latest` | `3306` 监听、`9104` 空闲、取得官方下载包 | ✅ `8.4.11`、`v0.21.0` |
| 二进制与 SELinux 上下文 | `tar -tzf`、`install`、`ls -lZ`、`--version` | `bin_t`、`0.21.0` | ✅ |
| 账号权限与 `.my.cnf` 隔离 | `SHOW GRANTS`、`ls -lZ`、`sudo -u mysqld-exporter mysql ...` | `0600` 归 `mysqld-exporter`，直连成功 | ✅ `exporter@127.0.0.1` |
| systemd 托管与 `/metrics` | `is-active`、`ss -lntp 'sport = :9104'`、`curl 9104/metrics` | `active`、`127.0.0.1:9104`、`mysql_up 1` | ✅ `pid=164538`、`unconfined_service_t` |
| Prometheus 热重载与采集 | `promtool`、`MainPID`、`journalctl`、查询指标 | `MainPID` 保持 `1575`、加载完成、入库 | ✅ `up=1`、`mysql_up=1`、`queries=371` |
| `rate()` 算 QPS | `rate(mysql_global_status_queries{job="mysql"}[1m])` | 返回正数速率 | ✅ `1.3333925952264545` |
| 停 `mysqld` 故障边界 | 停 `mysqld` 等 20s 查 `up`/`mysql_up`，再 `start` 复测 | 停后 `up=1`、`mysql_up=0`；启后回 `1` | ✅ |
| `mysql_up == 0` 告警规则与 Grafana 面板 | | | 未做 |
| `node2` 从库接入监控 | | | 未做 |
| 系统重启后 Exporter 与采集仍生效 | | | 未验证 |

## 踩坑

- `ls -lZ /usr/local/bin/mysqld-exporter` → `No such file or directory` → 把二进制文件名（下划线 `mysqld_exporter`）和系统用户名（连字符 `mysqld-exporter`）记混 → 改查 `/usr/local/bin/mysqld_exporter` 正常。
- `ps -efZ | grep '[m]ysql_exporter'` → 无输出 → 同上把进程名漏了 `d`（应为 `mysqld_exporter`）→ 改为 `grep '[m]ysqld_exporter'` 查到 `unconfined_service_t`。
- 期望在 `/metrics` 看到 `mysql_exporter_last_scrape_error` → 输出没有该指标 → `v0.21.0` 已无此旧指标名，实际暴露的是 `mysql_exporter_collector_duration_seconds` 与 `mysql_exporter_collector_success` → 用 `grep -E '^# TYPE mysql_exporter_'` 实测确认。

## 卡点 / 没想明白的

- 课堂追问「为何未运行 `restorecon -v` 前 `/usr/local/bin/mysqld_exporter` 已是 `bin_t`」：因为 `install` 是在 `/usr/local/bin/` 下新建文件并继承目标目录上下文（与 `mv` 保留原 `user_tmp_t` 不同），保留 `restorecon -v` 仍作为防御性兜底。
- 课后第 3 题把 `rate(...)` 漏写成裸区间向量 `[1m]`；第 5 题误以为 `mysqld-exporter` 被 `Requires=` 杀掉后 Prometheus 里的 `mysql_up` 也会变成 `0`（实际抓不到 `9104`，只有 `up=0`，`mysql_up` 不会收到新的 `0` 样本）。

## 面试题（5 道，对话中作答后批改）

1. （复述）在 MySQL 里创建监控账号 `'exporter'@'127.0.0.1'` 时，除了授予 `PROCESS`、`REPLICATION CLIENT`、`SELECT ON *.*` 三个权限外，我们在 `CREATE USER` 语句里还加了哪一个限制连接并发的参数？
<details><summary>批改要点</summary>

`WITH MAX_USER_CONNECTIONS 3`（限制并发连接上限）。用户答出 `max_user_connections`，要点正确；建议顺带记住完整写法及上限值 `3`。1 分。
</details>

2. （解释原因）为什么本课既不能按老教程用 `DATA_SOURCE_NAME` 环境变量、也不把数据库密码直接写进 `mysqld-exporter.service` 的 `ExecStart` 参数里，而要专门建一个 `0600` 权限的 `/etc/mysqld_exporter.cnf`？
<details><summary>批改要点</summary>

`mysqld_exporter` 自 `v0.15.0` 起已移除 `DATA_SOURCE_NAME` 环境变量；同时密码写在 `ExecStart` 命令行会暴露在 `ps` 进程列表（unit 文件也默认对普通用户可读），必须隔离进属主为 `mysqld-exporter` 的 `0600` `.my.cnf` 并用 `--config.my-cnf` 传入。1 分。
</details>

3. （应用）如果要统计 `job="mysql"` 过去 1 分钟平均每秒新增的慢查询数，使用本课看到的计数器指标 `mysql_global_status_slow_queries`，PromQL 查询应该怎么写？
<details><summary>批改要点</summary>

`rate(mysql_global_status_slow_queries{job="mysql"}[1m])`。用户作答写成 `mysql_global_status_slow_queries{job="mysql"}[1m]`，漏掉了外层 `rate(...)`（裸区间向量无法算出每秒增量速率）。0.5 分。
</details>

4. （排障）如果在 Prometheus 里查到 `up{job="mysql"}` 为 `1`，但 `mysql_up{job="mysql"}` 为 `0`，按今天搭建的采集链路，最可能是哪一段出了问题？
<details><summary>批改要点</summary>

`mysqld-exporter` 本身运行且 `9104` 可达，卡在 Exporter 连不上 MySQL 这一段（最常见是 `mysqld` 服务停止，或 `127.0.0.1:3306`/账号权限凭据异常）。1 分。
</details>

5. （反事实）如果把 `mysqld-exporter.service` `[Unit]` 里的 `After=mysqld.service` 误改成了强依赖 `Requires=mysqld.service`，当 `mysqld` 突然停止时，会对 `mysqld-exporter` 进程以及 Prometheus 观测 `up` / `mysql_up` 造成什么后果？
<details><summary>批改要点</summary>

`mysqld` 停掉时 `systemd` 会连带停止 `mysqld-exporter`，导致 Prometheus 抓不通 `9104`、`up{job="mysql"}` 变为 `0`；此时 Prometheus 根本拿不到 `mysql_up=0` 样本（`mysql_up` 不会变为 `0`，而是断采变空/过期），依赖 `mysql_up == 0` 的告警将失灵。用户答对进程被停和 `up` 变 `0`，但误以为 `mysql_up` 也会变成 `0`。0.5 分。
</details>

作答结果：4/5。薄弱点：Counter 算速率遗漏 `rate()` 包裹；未区分「Exporter 存活汇报 `mysql_up=0`」与「Exporter 自身宕机导致 Prometheus 只记 `up=0` 而无新 `mysql_up` 样本」。

## 复习记录

起算日是 11-27。

- [ ] D+1（11-28）
- [ ] D+3（11-30）
- [ ] D+7（12-04）

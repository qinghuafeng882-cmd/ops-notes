# Nginx Exporter：stub_status 与业务指标采集

- 日期：2026-11-26（Asia/Shanghai）
- 考点：运维扩展 —— Nginx `stub_status`、`nginx-prometheus-exporter` 二进制托管、Counter/Gauge 与 `up`/`nginx_up` 双层存活边界。不是 RHCSA EX200
- 环境：Rocky Linux 10（VirtualBox），node1（192.168.56.11，Nginx / Prometheus / nginx-prometheus-exporter 1.5.3），用户 fqh
- 状态：两段式采集、热重载入库、`rate()` QPS 计算、停 Nginx 时 `up=1` 与 `nginx_up=0` 分离均已完成并恢复（未配 `nginx_up == 0` 告警规则与 Grafana Nginx 面板；未做系统重启验证）
- 来源：课堂终端记录。归档时没有重跑

## 目标

在 node1 开启仅限本机访问的 Nginx `stub_status` 状态端点，通过 GitHub 官方二进制部署 `nginx-prometheus-exporter` 并用 `systemd` 托管在 `127.0.0.1:9113`，接入 Prometheus 采集后验证 `rate()` 计算实时 QPS 及「Nginx 停而 Exporter 活」的指标差异。

## 关键概念

- 业务组件监控采用**两段式暴露**：Nginx `stub_status`（`ngx_http_stub_status_module`）只输出 4 行纯文本计数，不是 Prometheus 格式；`nginx-prometheus-exporter` 负责 HTTP 拉取 `stub_status` 并翻译成 `/metrics`（默认端口 `9113`）。
- `stub_status` 4 行输出含义：
  - `Active connections`：此刻活跃连接总数，等于 `Reading + Writing + Waiting`；
  - `accepts`、`handled`、`requests`：自 Nginx 启动以来的全局累计值；若 `accepts > handled` 说明出现过连接丢弃（如超过并发上限）；开启 HTTP keepalive 时一般 `requests >= handled`；
  - `Reading`（读请求头）、`Writing`（写响应）、`Waiting`（keepalive 空闲等待）。
- **端口与 SELinux 避坑**：node1 上 `8080` 已被原有 Nginx 占用；SELinux `http_port_t` 默认已含 `8008`，因此将 `stub_status` 绑在 `127.0.0.1:8008`（既复用默认 SELinux 放行端口，又只听回环口无需开防火墙）。
- Rocky 10（BaseOS / AppStream / Extras / EPEL 10）无 `nginx-prometheus-exporter` 或 `mysqld_exporter` 原生 RPM 包，需从官方 Release 下载静态编译二进制，先 `tar -tzf` 看清单再安装至 `/usr/local/bin/` 并做 `restorecon`（保证 `bin_t`）。
- 专用系统账号 `nginx-exporter`（`--system --no-create-home --shell /sbin/nologin`）跑在 SELinux `unconfined_service_t` 域；`--web.listen-address=127.0.0.1:9113` 收窄暴露面，对应 `prometheus.yml` 目标写 `'127.0.0.1:9113'`。
- **Counter vs Gauge**：`nginx_connections_active` 是瞬时值 Gauge；`nginx_http_requests_total` 是单调递增 Counter，必须用 `rate(nginx_http_requests_total{job="nginx"}[1m])` 算出每秒平均增长率（QPS）。
- **双层存活指标区别**：`up{job="nginx"}` 仅反映 Prometheus 能否抓通 `127.0.0.1:9113`；`nginx_up{job="nginx"}` 才反映 Exporter 能否抓通 Nginx `stub_status`。Nginx 停掉而 Exporter 仍运行期间，`up` 仍为 `1`，只有 `nginx_up` 变为 `0`。

## 关键步骤

```bash
# —— node1 核对模块、端口占用、SELinux 端口与软件源 ——
nginx -V 2>&1 | tr ' ' '\n' | grep stub_status   # --with-http_stub_status_module
systemctl is-active nginx                        # active
sudo ss -lntp 'sport = :9113'                    # 空闲
dnf list --available '*exporter*'                # 无 nginx/mysql exporter RPM 包

sudo ss -lntp | grep -E 'nginx|httpd'            # 8080 已被 nginx 占用；httpd 占 80/8090/8888
ls -l /etc/nginx/conf.d/
sudo semanage port -l | grep -w http_port_t      # 默认已包含 8008
sudo ss -lntp 'sport = :8008'                    # 空闲
```

```bash
# —— 配置仅限本机访问的 stub_status 并热重载验证 ——
sudo tee /etc/nginx/conf.d/stub_status.conf >/dev/null << 'EOF'
server {
  listen 127.0.0.1:8008;
  server_name localhost;

  location /stub_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
    access_log off;
  }
}
EOF

sudo nginx -t && sudo systemctl reload nginx
sudo ss -lntp 'sport = :8008'                    # LISTEN 127.0.0.1:8008（master 1797，新 worker 80938）
curl -fsS http://127.0.0.1:8008/stub_status
# Active connections: 1
# server accepts handled requests
#  6071 6071 6071
# Reading: 0 Writing: 1 Waiting: 0
```

```bash
# —— 下载并安装官方 v1.5.3 二进制 ——
mkdir -p /tmp/nginx-exp-dl
curl -fL --connect-timeout 10 -o /tmp/nginx-exp-dl/exporter.tar.gz \
  https://github.com/nginx/nginx-prometheus-exporter/releases/download/v1.5.3/nginx-prometheus-exporter_1.5.3_linux_amd64.tar.gz
tar -tzf /tmp/nginx-exp-dl/exporter.tar.gz
tar -xzf /tmp/nginx-exp-dl/exporter.tar.gz -C /tmp/nginx-exp-dl
sudo install -m 0755 -o root -g root /tmp/nginx-exp-dl/nginx-prometheus-exporter /usr/local/bin/nginx-prometheus-exporter
sudo restorecon -v /usr/local/bin/nginx-prometheus-exporter
ls -lZ /usr/local/bin/nginx-prometheus-exporter
# -rwxr-xr-x. 1 root root system_u:object_r:bin_t:s0 14577826 ...
/usr/local/bin/nginx-prometheus-exporter --version
# nginx_exporter, version 1.5.3 (go1.26.6, linux/amd64)
```

```bash
# —— 创建低权限系统用户与 systemd 服务并验证 9113/metrics ——
sudo useradd --system --no-create-home --shell /sbin/nologin nginx-exporter

sudo tee /etc/systemd/system/nginx-prometheus-exporter.service >/dev/null << 'EOF'
[Unit]
Description=NGINX Prometheus Exporter
After=network-online.target nginx.service
Wants=network-online.target
[Service]
User=nginx-exporter
Group=nginx-exporter
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
  --web.listen-address=127.0.0.1:9113 \
  --nginx.scrape-uri=http://127.0.0.1:8008/stub_status
Restart=on-failure
RestartSec=5s
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nginx-prometheus-exporter.service
systemctl is-active nginx-prometheus-exporter    # active
sudo ss -lntp 'sport = :9113'                    # LISTEN 127.0.0.1:9113 pid=96782
curl -fsS http://127.0.0.1:9113/metrics | grep -E '^(nginx_up|nginx_connections_|nginx_http_requests_total)'
# nginx_connections_accepted 6858 ... nginx_http_requests_total 6858 ... nginx_up 1

rm -rf /tmp/nginx-exp-dl
ps -efZ | grep '[n]ginx-prometheus-exporter'     # unconfined_service_t
```

```bash
# —— 接入 Prometheus 抓取并热重载验证 ——
sudo cat /etc/prometheus/prometheus.yml
sudo cp -a /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak-$(date +%Y%m%d-%H%M%S)
systemctl show -p MainPID --value prometheus.service   # 1575

sudo tee -a /etc/prometheus/prometheus.yml >/dev/null << 'EOF'
  - job_name: nginx
    static_configs:
      - targets: ['127.0.0.1:9113']
EOF

sudo promtool check config /etc/prometheus/prometheus.yml   # SUCCESS: valid prometheus config file syntax
sudo systemctl reload prometheus.service
systemctl show -p MainPID --value prometheus.service        # 仍为 1575
sudo journalctl -u prometheus.service -n 10 --no-pager      # Completed loading of configuration file (195.23ms)

sleep 18
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="nginx"}' -w '\n'                  # "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=nginx_up{job="nginx"}' -w '\n'            # "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=nginx_http_requests_total{job="nginx"}' -w '\n'   # "7359"
```

```bash
# —— rate() 计算 QPS 与停 Nginx 故障边界实测 ——
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=rate(nginx_http_requests_total{job="nginx"}[1m])' -w '\n'
# "0.39999999999999997"（约 0.4 req/s：Keepalived 每 3s 探活 ≈ 0.333 + Prometheus 每 15s 抓取 ≈ 0.067）

sudo systemctl stop nginx
sleep 20
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=up{job="nginx"}' -w '\n'       # 仍为 "1"
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=nginx_up{job="nginx"}' -w '\n' # 变为 "0"

sudo systemctl start nginx
sleep 20
systemctl is-active nginx nginx-prometheus-exporter                                                          # active / active
curl -fsS --get 'http://localhost:9090/api/v1/query' --data-urlencode 'query=nginx_up{job="nginx"}' -w '\n' # 恢复 "1"
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| Nginx 模块与端口 | `nginx -V`、`ss -lntp 'sport = :8008'` | 含 `http_stub_status_module`，`8008` 空闲且在 `http_port_t` | ✅ |
| `stub_status` 输出 | `curl -fsS http://127.0.0.1:8008/stub_status` | 仅监听 `127.0.0.1:8008`，返回 4 行统计 | ✅ `6071 6071 6071` |
| Exporter 二进制与托管 | `--version`、`is-active`、`ss -lntp 'sport = :9113'`、`curl 9113/metrics` | `1.5.3`、`bin_t`、`active`、`127.0.0.1:9113`、`nginx_up 1` | ✅ `pid=96782` |
| Prometheus 热重载与采集 | `promtool`、`MainPID`、`journalctl`、查询 `up`/`nginx_up`/`requests_total` | `MainPID` 保持 `1575`、加载完成、指标入库 | ✅ `up=1`、`nginx_up=1`、`7359` |
| `rate()` 计算实时 QPS | `rate(nginx_http_requests_total{job="nginx"}[1m])` | 输出正数速率 | ✅ `0.39999999999999997` |
| 停 Nginx 故障边界 | 停 `nginx` 等 20s 查 `up` 与 `nginx_up`，再 `start` 复测 | 停后 `up=1`、`nginx_up=0`；启后回 `1` | ✅ |
| `nginx_up == 0` 告警规则与 Grafana 面板 | | | 未做 |
| 系统重启后 Exporter 与采集仍生效 | | | 未验证 |

## 踩坑

- `sudo install -m 0755 -o root -g root /tmp/nginx-exp-dl/nginx-prometheus-exporter` → `install: missing destination file operand` → 命令末尾漏粘贴目标路径参数 → 补全 `/usr/local/bin/nginx-prometheus-exporter` 后成功。
- 课堂查询监听端口时先敲了一次 `sudo ss -lntp 'sport = :8088'` → 显示业务 8088 监听 → 随后改为查 `8008` 确认 `127.0.0.1:8008` 已监听。
- 配置 `/etc/nginx/conf.d/stub_status.conf` 时实际写成普通前缀 `location /stub_status`（未加推荐的精确修饰符 `=`）→ 在该独立内部 `server` 块中工作正常，已提醒严格写法建议用 `location = /stub_status`。

## 卡点 / 没想明白的

- 课堂追问 `tar -t` 参数的作用：已明确 `-t`（`--list`）表示只查看归档文件列表而不解压落盘，部署前先看清单可确认文件结构、避免散文件直接铺满目录。课后 5 题全对。

## 面试题（5 道，对话中作答后批改）

1. （复述）Nginx `stub_status` 输出里的 `accepts` 和 `handled` 两个累计值，如果出现 `accepts > handled` 说明发生了什么情况？
<details><summary>批改要点</summary>

说明 Nginx 接收了连接但未能全部处理完成，出现过连接丢弃（例如达到并发连接数上限）。1 分。
</details>

2. （解释原因）在 `/etc/nginx/conf.d/stub_status.conf` 里配置 `/stub_status` 端点时，为什么要专门加上 `access_log off;`？
<details><summary>批改要点</summary>

Prometheus 会通过 Exporter 定期高频拉取 `/stub_status`，关闭其访问日志可避免大量监控探活请求刷屏污染正常业务访问日志。1 分。
</details>

3. （应用）想通过已经采集到的 `nginx_http_requests_total` 算出最近 1 分钟的平均每秒请求数（QPS），PromQL 查询应该怎么写？
<details><summary>批改要点</summary>

`rate(nginx_http_requests_total{job="nginx"}[1m])`（用户作答写了 `query=...`，要点完全正确）。1 分。
</details>

4. （排障）在 Prometheus 里查到 `up{job="nginx"}` 依然是 `1`，但 `nginx_up{job="nginx"}` 变成了 `0`，最可能出了什么问题？
<details><summary>批改要点</summary>

`nginx-prometheus-exporter` 进程本身正常（Prometheus 能抓到 `9113`），但 Exporter 连不上 Nginx 的 `stub_status`（最常见是 Nginx 服务已停止，或 `stub_status` 端口/路径异常）。1 分。
</details>

5. （反事实）如果运维误把 Nginx 宕机告警条件写成 `up{job="nginx"} == 0`，当 `nginx` 服务挂掉而 `nginx-prometheus-exporter` 进程仍在运行时，为什么不会触发告警？
<details><summary>批改要点</summary>

`up` 只检测 Prometheus 能否成功拉取 Exporter 端口的 `/metrics`；只要 `nginx-prometheus-exporter` 进程还活着，Prometheus 就能抓到指标并把 `up` 置为 `1`，因此不会触发告警（必须监控 `nginx_up == 0`）。1 分。
</details>

作答结果：5/5（全对）。

## 复习记录

起算日是 11-26。

- [ ] D+1（11-27）
- [ ] D+3（11-29）
- [ ] D+7（12-03）

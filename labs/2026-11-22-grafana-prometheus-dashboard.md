# Grafana：接上本机 Prometheus 并做第一块面板

- 日期：2026-11-22（北京时间 09:33–11:56，这是课堂跨度，不是净学习时长）
- 考点：运维扩展 —— Grafana 查询和展示。Prometheus 采集沿用 [上午的首课](2026-11-22-prometheus-node-exporter.md)。不是 RHCSA EX200
- 环境：Rocky Linux 10.2（VirtualBox），node1 192.168.56.11，用户 fqh。Windows 宿主机只用来打开页面
- 状态：Grafana 基础做完。和上午的 Prometheus 基础合在一起，也只是监控的开头。Alertmanager、告警实验、导入仪表盘都没做
- 来源：课堂终端记录和用户截图。归档时没有重跑

## 目标

在 node1 上把 Grafana 开到 3000，让它从本机 9090 查出 `node_memory_MemAvailable_bytes{job="node"}`，存成一块面板。

## 关键概念

- 包名 `grafana` 已经是完整名字。完整标识 `grafana-10.2.6-28.el10_2.9.x86_64` 拆开是 name `grafana`、version `10.2.6`、release `28.el10_2.9`、arch `x86_64`。不是「用前缀匹配所以装了三个」。
- 依赖 `grafana-selinux` 会装上。`grafana-pcp` 是弱依赖，默认也会装。第一次安装可以加 `--setopt=install_weak_deps=False`。这节只讲了，没有执行，也没有卸载。
- 单元里的 `After=` 数据库单元，不表示那些数据库已经安装或正在跑。安全加固项留着，没有改 vendor 单元。
- 只有真正被读到的环境变量才起作用。`User=` 和 `LimitNOFILE=` 是单元里直接写的，不能只改一个同名环境变量就当成已经生效。
- Grafana 自己的数据不是 Prometheus 样本。默认用 SQLite，存的是 dashboard、用户、数据源这些配置。
- INI 里 `#` 和 `;` 都是注释。`http_addr` 空着时默认听所有接口。`domain` 和 `root_url` 不是监听地址。
- Grafana 自带 Web 后端（3000）和前端。这次不需要 Nginx，也没有加代理。
- 本机 `curl` 得到 200，不能推断别的机器也能连上。
- public 上有 `enp0s3` 和 `enp0s8`。加在 public 的端口规则两块网卡都生效，不是只对实验口。
- 数据源的 URL 由 Grafana 后端去请求。所以 `localhost:9090` 是 node1，不是 Windows。数据源上的间隔只帮助查询，不改变 Prometheus 的采集间隔。
- 终端里用 curl 打通 9090，不代表 `grafana_t` 有权连这个端口。`websm_port_t` 只是端口类型，不说明 9090 上实际跑的是哪个程序。
- 超时打不开页面，和 Save & test 的 `permission denied`，是两件独立的事：前者是防火墙没放 3000，后者是 SELinux 的 `name_connect`。
- 展示单位按 1024 自动换成 MiB/GiB，不改原始样本，不要再除一次 1024。
- Unit 只是显示单位。Apply 作用在这一块面板。Save 保存整个 dashboard。

## 关键步骤

```bash
dnf list --available 'grafana*'
sudo dnf install grafana
rpm -qc grafana
```

`dnf list` 看到 appstream 里：`grafana-10.2.6-28.el10_2.9`、同版本的 `grafana-selinux`、`grafana-pcp-5.3.0-8.el10_2.2`。安装明确是 Installing `grafana`，依赖 `grafana-selinux`，弱依赖 `grafana-pcp`。3 个包，下载约 121M，安装后约 529M。没有别的升级或删除。

`rpm -qc`：`/etc/grafana/grafana.ini`、`/etc/grafana/ldap.toml`、`/etc/sysconfig/grafana-server`。

`systemctl cat grafana-server.service`：`EnvironmentFile=/etc/sysconfig/grafana-server`，`User`/`Group`=`grafana`，`Type=notify`，`Restart=on-failure`，`WorkingDirectory=/usr/share/grafana`，`LimitNOFILE=10000`。`ExecStart=/usr/sbin/grafana server --config=${CONF_FILE} --pidfile=${PID_FILE_DIR}/grafana-server.pid --packaging=rpm cfg:default.paths.logs=${LOG_DIR} cfg:default.paths.data=${DATA_DIR} cfg:default.paths.plugins=${PLUGINS_DIR} cfg:default.paths.provisioning=${PROVISIONING_CFG_DIR}`。

`/etc/sysconfig/grafana-server` 里实际是：`GRAFANA_USER`/`GRAFANA_GROUP`=`grafana`，`GRAFANA_HOME=/usr/share/grafana`，`LOG_DIR=/var/log/grafana`，`DATA_DIR=/var/lib/grafana`，`MAX_OPEN_FILES=10000`，`CONF_DIR=/etc/grafana`，`CONF_FILE=/etc/grafana/grafana.ini`，`RESTART_ON_UPGRADE=true`，`PLUGINS_DIR=/var/lib/grafana/plugins`，`PROVISIONING_CFG_DIR=/etc/grafana/provisioning`，`PID_FILE_DIR=/var/run/grafana`。

```bash
sudo sed -n '/^\[server\]/, /^\[/p' /etc/grafana/grafana.ini
```

`sed -n` 不按默认把每行打出来。起止范围包含两端。`^` 是行首，`\[` 转义方括号，`p` 打印。逗号后面的空格不影响这条命令。输出从 `[server]` 到包含下一节 `[server.custom_response_headers]`。注释里看到：`;protocol=http`、`;http_addr=`、`;http_port=3000`、`;domain=localhost`、`;root_url=%(protocol)s://%(domain)s:%(http_port)s/`。主配置没有改。

```bash
sudo ss -lntp 'sport = :3000'
sudo systemctl enable --now grafana-server.service
systemctl is-active grafana-server
sudo ss -lntp 'sport = :3000'
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:3000/login
```

第一次 `ss` 只有表头。`enable --now` 在 `/etc/systemd/system/multi-user.target.wants/grafana-server.service` 建了软链，指向 `/usr/lib/systemd/system/grafana-server.service`。`is-active` 是 active。再 `ss`：`LISTEN 0 4096 *:3000`，`users` 里的进程名是 `grafana`，pid 93958，fd 13。跑这个进程的用户来自单元里的 `User=grafana`，不是 `ss` 这一列证明的。本机登录页状态码 200。`-o` 丢掉正文，`-w` 只留下状态码。

Windows 打开 `http://192.168.56.11:3000/login`，截图是 `ERR_CONNECTION_TIMED_OUT`。

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-all
```

public（default），接口 `enp0s3`、`enp0s8`。服务：cockpit、dhcpv6-client、http、https、mountd、mysql、nfs、rpc-bind、ssh。端口：8090、8888、8081、8082、8083、8088 的 tcp。富规则只有 ipv4、source 192.168.56.12、protocol vrrp、accept。没有 3000。

指导执行了临时放行 `sudo firewall-cmd --zone=public --add-port=3000/tcp`。这条命令的终端输出没有贴出来。随后登录页截图能打开。

```bash
sudo firewall-cmd --permanent --zone=public --add-port=3000/tcp
sudo firewall-cmd --permanent --zone=public --query-port=3000/tcp
```

permanent 那条输出 `success`。查询输出 `yes`。没有 reload，没有关防火墙，也没有做 runtime-to-permanent 的全量保存。

新装后的本机管理员登录是用户自己做的。指导在本机看 `admin_user` / `admin_password`，值不贴进笔记，一律 `...`。11:02 的首页截图说明已经登录。首次改密码只指导过，没有证明改过，不记成已改。

首页 Add your first data source，选 Prometheus。配置页：Name `prometheus`，Default 开着，URL `http://localhost:9090`，No Authentication，没有改 TLS，Scrape interval 15s。

Save & test 第一次：`Post "http://localhost:9090/api/v1/query": dial tcp [::1]:9090: connect: permission denied`。不是密码问题，也不是 Windows 访问 3000 的那个超时。

```bash
# 空格把 USER_AVC 拆开了，报 USER_AVC is an unsupported option
sudo ausearch -m AVC, USER_AVC -ts recent -i
# 正确是逗号后不空格
sudo ausearch -m AVC,USER_AVC -ts recent -i
getsebool -a | grep grafana
```

`-i` 是尽可能把编号解释出来。`getsebool -a` 列出全部布尔值。

11:07:05 有四条，时间 `.138`、`.150`、`.150`、`.170`，seq 486–489。syscall `connect`，`success=no`，exit `EACCES`。`comm=grafana`，pid 93958，subj `grafana_t`。AVC：`denied { name_connect }`，dest 9090，`scontext=system_u:system_r:grafana_t:s0`，`tcontext=system_u:object_r:websm_port_t:s0`，`tclass=tcp_socket`，permissive 0。

`getsebool -a | grep grafana` 里这六项都是 off：`grafana_can_reverse_proxy`、`grafana_can_tcp_connect_elasticsearch_port`、`grafana_can_tcp_connect_ldap_port`、`grafana_can_tcp_connect_mysql_port`、`grafana_can_tcp_connect_postgresql_port`、`grafana_can_tcp_connect_prometheus_port`。只打开最后这一项。先临时执行 `sudo setsebool grafana_can_tcp_connect_prometheus_port on`，这条的终端输出没有贴出；随后截图是 Successfully queried the Prometheus API。

持久化时先写成 `connnect`（三个 n），报 Boolean 未定义。改正后：

```bash
sudo setsebool -P grafana_can_tcp_connect_prometheus_port on
getsebool grafana_can_tcp_connect_prometheus_port
```

`-P` 那条没有输出。`getsebool` 显示 on。没有关 SELinux，没有改端口标签，没有用 `audit2allow`，没有重启后再查。

建面板：building a dashboard → Add visualization → prometheus。Time series，Code 里是 `node_memory_MemAvailable_bytes{job="node"}`，Run queries，Type 为 Range。用户起初只填了裸指标 `node_memory_MemAvailable_bytes`，也出了曲线，随后按指导补上 `{job="node"}`。截图里大约 08:00 以后有曲线，更早是空白，空白不是 0。不推断波动原因。

Title 按指导写成 `node1 可用内存`。右侧 Search options，unit 选 Data → bytes(IEC)。用户明确设过这个单位。Apply 应用到面板，Save 保存整个仪表盘。名称按指导是 `node1 基础监控`，并刷新看过。11:45 用户说做好了。最后的保存页没有截图，不把 title、job 筛选和仪表盘名字记成有单独截图核对。没有第二块仪表盘，没有导入，没有做告警实验。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 安装 | `sudo dnf install grafana` | 装上 grafana 和 grafana-selinux，弱依赖 grafana-pcp | ✅ 3 包，无其他升级删除 |
| 启动前 3000 | `sudo ss -lntp 'sport = :3000'` | 没人听 | ✅ 只有表头 |
| 启用 | `enable --now` 后 `is-active` | active | ✅ 软链指向 `/usr/lib/systemd/system/grafana-server.service` |
| 本机登录页 | `curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:3000/login` | 200 | ✅ 不能推断 Windows 也能连 |
| Windows 登录页 | 浏览器打开 `http://192.168.56.11:3000/login` | 放行前超时，放行后能开 | ✅ 截图。临时 `add-port` 的终端输出没贴 |
| 永久放行 | `--permanent --add-port=3000/tcp`，再 `--permanent --query-port` | success，yes | ✅ 没有 reload，没有全量 runtime-to-permanent |
| 登录 | 首页截图 | 已进入 | ✅ 11:02。密码是否改过未证明 |
| 数据源 | Save & test | 能查 Prometheus API | ✅ 截图 Successfully queried the Prometheus API。临时 setsebool 的终端输出没贴 |
| 布尔值 | `getsebool grafana_can_tcp_connect_prometheus_port` | on | ✅ `-P` 之后 |
| 面板 | 用户 11:45 确认做完 | 有一块可用内存曲线，单位 bytes(IEC) | 用户确认。保存页没有单独截图 |
| 关防火墙、关 SELinux、audit2allow、改端口标签 | | | 未做 |
| 重启后 Grafana 和这个布尔值仍在 | | | 未验证 |
| 弱依赖开关 | `--setopt=install_weak_deps=False` | | 只讲了，未执行 |

## 踩坑

- `systemctl cat granafa-server.service` → `No files found` → 改成 `grafana-server.service`。
- `grep -E '^[;[:space:]]*admin_{user|password}[[:space:]]*='` 没有输出 → `{user|password}` 不是分组。选择要用圆括号 → 改成 `admin_(user|password)`。密码值不入库，记 `...`。
- Windows `ERR_CONNECTION_TIMED_OUT` → public 的端口列表和富规则里都没有 3000 → 临时 `--add-port=3000/tcp` 后登录页能开，再 `--permanent`，查询为 yes。没有 reload。
- Save & test：`dial tcp [::1]:9090: connect: permission denied` → 不是防火墙那件事。ausearch 是 `name_connect` 被拒，`grafana_t` 连 9090，类型 `websm_port_t`，permissive 0 → 只开 `grafana_can_tcp_connect_prometheus_port`。
- `sudo ausearch -m AVC, USER_AVC -ts recent -i` → `USER_AVC is an unsupported option`，逗号后的空格把参数拆开了 → 改成 `-m AVC,USER_AVC`。
- `setsebool` 写成 `connnect`（三个 n）→ 报 `Boolean ... not defined` → 改成 `grafana_can_tcp_connect_prometheus_port`，`-P` 之后 `getsebool` 为 on。
- 图上只填了裸的 `node_memory_MemAvailable_bytes` 也有曲线 → 没有按 job 收窄 → 补上 `{job="node"}`。

## 卡点 / 没想明白的

- 首次登录后的改密码只指导了，没有证据证明改过。
- 11:45 的「做好了」是用户确认。title、job 筛选、仪表盘名字没有一张保存页截图单独对过。
- 曲线早于大约 08:00 的空白不是 0。波动原因没有查。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，每题 1 分。总分 5/5。没有另编题目。

1. （概念）Grafana 和 Prometheus 各做什么，面板数据从哪来。
<details><summary>批改要点</summary>

Grafana 查询和可视化。Prometheus 采集、存储、规则。用的是 9090，不是 exporter 的 9100。用户说成「拉取数据」，已说成查询。1 分。
</details>

2. （概念）Windows 打开页面和 Grafana 访问 Prometheus，两边的地址各指谁。
<details><summary>批改要点</summary>

浏览器请求的是 node1 的 3000。Grafana 后端请求的 `localhost:9090` 是发起请求的那台机器，也就是 node1。1 分。
</details>

3. （排障）页面超时和 Save & test 的 permission denied，是不是都用防火墙解决。
<details><summary>批改要点</summary>

超时是防火墙放行 3000。SELinux 日志是 `name_connect`，用 Grafana 专用的 boolean 放权。不是两件都靠防火墙。1 分。
</details>

4. （操作）`setsebool -P` 和不加 `-P` 差在哪。
<details><summary>批改要点</summary>

`-P` 写入持久配置。不加时，重启后回到持久配置。1 分。
</details>

5. （取舍）Unit、Apply、Save 各作用在哪。
<details><summary>批改要点</summary>

Unit 只决定怎么展示。Apply 应用到这块面板。Save 保存整个 dashboard。1 分。
</details>

作答结果：5/5。

## 复习记录

起算日是 11-22。上午的合并复习 4.5/5 不在这篇里改。

- [x] D+1（11-23，1/1（合并复习，五题合计 5/5，不是本篇单独 5 题）；`grafana_can_tcp_connect_prometheus_port`）
- [x] D+3（11-26，1/1（合并复习，六题合计 5/6，不是本篇单独 6 题）；数据源 URL 由跑在 node1 上的 Grafana 后端发起请求，因此 `localhost:9090` 指向 node1）
- [ ] D+7（11-29）

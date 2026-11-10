# Nginx（四）：反代传递客户端信息 + upstream 负载均衡

- 日期：2026-11-10（B1-5）
- 考点：面试高频"后端拿不到真实 IP 怎么办""XFF 能不能信""upstream 一台挂了会怎样"
- 环境：Rocky Linux 10.2（VirtualBox）。node1（192.168.56.11）Nginx 8081（site1.conf）→ 本机 httpd 80/8090；node2（192.168.56.12）做客户端
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

让后端日志看到客户端真实 IP 和原始 Host；把 `/p1/` 换成 upstream，验证轮询和后端故障时的行为。

## 关键概念

- 连接②上 nginx 是客户端：后端看到的源 IP 是 `127.0.0.1`，Host 默认是 `$proxy_host`（proxy_pass 里的地址）。TCP 源地址改不了，只能把信息放进 HTTP 头。

| 变量 | 值 |
|---|---|
| `$remote_addr` | 连接①的对端 IP（TCP 来的，伪造不了） |
| `$host` | 客户端 Host 去掉端口、转小写；没有 Host 用 server_name |
| `$http_host` | 客户端 Host 原样（带端口） |
| `$proxy_add_x_forwarded_for` | 客户端没带 XFF → 只有 `$remote_addr`；带了 → `原值, $remote_addr` |

- Host 表示**站点**，不是机器。后端是站点的内部实现，要知道用户要的是哪个站点（虚拟主机、生成跳转地址）；机器寻址已经由 TCP 完成。后端是另一个独立站点时才用默认的 `$proxy_host`。
- XFF 从右往左读，只信自己代理追加的部分；X-Real-IP 用 `$remote_addr` 覆盖，客户端污染不了。
- `proxy_set_header` 继承：当前层写了任意一条，上层的全部不继承。
- 命名规律：下划线的是 nginx（`proxy_set_header`、`$remote_addr`），连字符的是 HTTP 头（`X-Real-IP`）。
- upstream：默认加权轮询；`max_fails=1 fail_timeout=10s`（被动健康检查，用真实请求探测）；`proxy_next_upstream error timeout`（失败时同一请求换下一台）。用了 upstream 后 `$proxy_host` = 组名。

## 关键步骤

```bash
# node1：httpd 加一个专看代理头的日志（主服务器，无 VirtualHost，conf.d 生效）
sudo tee /etc/httpd/conf.d/proxylog.conf << 'EOF'
LogFormat "%h port=%{local}p host=%{Host}i xri=%{X-Real-IP}i xff=\"%{X-Forwarded-For}i\" \"%r\" %>s" proxylog
CustomLog "logs/proxy_access_log" proxylog
EOF
# （最初没有 port=；第二版是 port=%p，见踩坑）
sudo apachectl configtest && sudo systemctl reload httpd

# node2：请求；node1：看后端日志
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.56.11:8081/p1/a.html
sudo sh -c 'tail -n 1 /var/log/httpd/proxy_access_log'

# site1.conf 最终状态（/p2/ /p3/ 未改）
# upstream backend {
#   server 127.0.0.1:80;
#   server 127.0.0.1:8090;
#   server 127.0.0.1:8099;      # 没人监听，模拟后端故障
# }
#   location /p1/ {
#     proxy_pass http://backend;
#     proxy_set_header Host $host;
#     proxy_set_header X-Real-IP $remote_addr;
#     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#   }
sudo nginx -t && sudo systemctl reload nginx

# 伪造 XFF
curl -s -o /dev/null -w '%{http_code}\n' -H 'X-Forwarded-For: 1.2.3.4' http://192.168.56.11:8081/p1/a.html

# 轮询
for i in 1 2 3 4; do curl -s -o /dev/null -w '%{http_code}\n' http://192.168.56.11:8081/p1/a.html; done
sudo sh -c 'tail -n 4 /var/log/httpd/proxy_access_log'
sudo tail -n 5 /var/log/nginx/error.log
getsebool httpd_can_network_connect      # on
```

## 验证

| 检查项 | 后端日志 / 输出 | 结果 |
|---|---|---|
| 基线（未加头） | `127.0.0.1 host=127.0.0.1:8090 xri=- xff="-" "GET /p1/a.html HTTP/1.0"` | ✅ 与预测一致 |
| 加三个头 | `127.0.0.1 host=192.168.56.11 xri=192.168.56.12 xff="192.168.56.12"` | ✅ Host 没有端口 |
| 伪造 XFF | `xri=192.168.56.12 xff="1.2.3.4, 192.168.56.12"` | ✅ 左边可伪造，xri 不受影响 |
| 轮询（`%{local}p`） | port 80 / 8090 / 80 / 8090 | ✅ |
| 加入 8099 后 6 次请求 | 客户端 6 个 404；后端 6 条，80/8090 交替 | ✅ 无 502 |
| 8099 失败记录 | `connect() failed (111: Connection refused) ... upstream: "http://127.0.0.1:8099/p1/a.html"` + `upstream server temporarily disabled` | ✅ 只出现一次 |
| SELinux | `httpd_can_network_connect --> on`（所以是 111 不是 13） | ✅ |

## 踩坑

- `sudo nginx -t $$ sudo systemctl reload nginx` → `invalid option: "2754"`：`$$` 是当前 shell 的 PID，被展开后成了 nginx 的参数，reload 没执行。应为 `&&`。报错里的值在命令里找不到 → 是 shell 展开出来的。
- 写成 `X-Forwarded_For`：`nginx -t` 查不出（头名是任意字符串），后端按 `X-Forwarded-For` 取不到。发现后改正。
- `nginx: [emerg] invalid URL prefix in site1.conf:21`：proxy_pass 第 21 行改错，用 `vim +21` 修好。（推测是漏了 `http://`，组名前也必须带协议。）
- `port=%p` 四次全是 80：`%p` 是规范端口（canonical port）。`Host $host` 去掉了端口，httpd 退回默认 80（推测的计算过程；改成 `%{local}p` 后出现 80/8090 交替，证实是测量字段的问题）。教训：先验证测量工具本身。
- for 循环 URL 后按了回车，下一行以 `;` 开头 → `syntax error near unexpected token ';'`：换行已结束命令，`;` 前面是空命令。

## 卡点 / 没想明白的

- 客户端 Host 是"nginx 的地址"，为什么要原样传给后端 → 已解答：Host 是站点名，不是机器地址（见关键概念）。

## 面试题（5 道，对话中作答后批改）

1. （复述）后端日志客户端 IP 全是 127.0.0.1，用单个值把真实 IP 传给后端，加哪条指令？
<details><summary>批改要点</summary>

`proxy_set_header X-Real-IP $remote_addr;`
</details>

2. （原因）为什么后端不能信 X-Forwarded-For 最左边的值？
<details><summary>批改要点</summary>

最左边是客户端自己带的，nginx 原样保留只在后面追加，可以伪造。只信自己代理追加的右侧部分。
</details>

3. （应用）站点在 8081，后端按 Host 生成的跳转丢了 `:8081`，Host 换成哪个变量？
<details><summary>批改要点</summary>

`$http_host`（原样带端口）；`$host` 去掉了端口。
</details>

4. （排障）server 层写了 Host / X-Real-IP / XFF 三条，某 location 只加了 `proxy_set_header X-Foo bar;`，该 location 后端收到 `Host: backend`，为什么？
<details><summary>批改要点</summary>

当前层写了任意一条 `proxy_set_header`，上层的全部不继承；Host 退回默认 `$proxy_host`，用 upstream 时就是组名。
</details>

5. （反事实）8099 换成整机宕机、包全部丢失，轮到它的请求客户端会怎样？
<details><summary>批改要点</summary>

没有 502，但要等 `proxy_connect_timeout`（默认 60s）超时后换下一台，请求卡约一分钟。
</details>

作答结果：5 道全对。第 1 题拼写写成 `Proxy-Set-Header X-Real-IP $remote_address`（要点对，指令和变量名拼错）。

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

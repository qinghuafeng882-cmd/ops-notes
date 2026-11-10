# Nginx（三）：反向代理 proxy_pass 与 URI 改写

- 日期：2026-11-09（B1-4）
- 考点：面试高频"proxy_pass 末尾斜杠""502 怎么查"
- 环境：Rocky Linux 10.2（VirtualBox）。node1（192.168.56.11）Nginx 8081（site1.conf）→ 本机 httpd 8090；node2（192.168.56.12）做客户端
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

在 8081 上配三个反代 location，用后端访问日志看清 `proxy_pass` 带不带 URI 时转发出去的 URI。

## 关键概念

- 反代 = 两条 TCP 连接：node2 → nginx（①），nginx → 后端（②）。后端只看到 nginx。
- 连接②出问题 → 502，原因看 `error.log`。

| `proxy_pass` 写法 | 端口后有无 URI | 后端收到 |
|---|---|---|
| `http://127.0.0.1:8090` | 无 | 原样 |
| `http://127.0.0.1:8090/` | 有（`/`） | location 匹配部分替换成 `/` |
| `http://127.0.0.1:8090/new/` | 有 | 替换成 `/new/` |

- 判断标准是"端口后有没有 URI"，不是"末尾有没有 `/`"。
- 替换的是 location **字面字符串**：`location /p2` + `proxy_pass .../` → `/p2/a.html` 变成 `//a.html`。location 和 proxy_pass 的末尾斜杠要成对。

## 关键步骤

```bash
curl -I http://localhost:8090                   # node1：200，Server: Apache，后端正常

sudo vim /etc/nginx/conf.d/site1.conf           # server 块里加：
#   location /p1/ { proxy_pass http://127.0.0.1:8090; }
#   location /p2/ { proxy_pass http://127.0.0.1:8090/; }
#   location /p3/ { proxy_pass http://127.0.0.1:8090/new/; }
sudo nginx -t && sudo systemctl reload nginx

# node2 上：三个都 404（文件不存在，预期）
curl -I http://192.168.56.11:8081/p1/a.html     # 404，Server: nginx，charset=iso-8859-1
curl -I http://192.168.56.11:8081/p2/a.html
curl -I http://192.168.56.11:8081/p3/a.html

# node1 上看后端收到的 URI
sudo tail -n 3 /var/log/httpd/*access_log       # 失败：No such file or directory（见踩坑）
sudo ls /var/log/httpd
sudo sh -c 'tail -n 3 /var/log/httpd/*access_log'
# 127.0.0.1 ... "HEAD /p1/a.html HTTP/1.0" 404
# 127.0.0.1 ... "HEAD /a.html HTTP/1.0" 404
# 127.0.0.1 ... "HEAD /new/a.html HTTP/1.0" 404
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 请求穿过代理到达后端 | node2 `curl -I` + 后端 access_log | 后端有对应记录 | ✅ |
| 三种写法的 URI 改写 | `sudo sh -c 'tail ... access_log'` | `/p1/a.html` `/a.html` `/new/a.html` | ✅（与预测一致） |
| SELinux 允许 nginx 连 8090 | 没有 502 | 无 `name_connect` 拒绝 | ✅（间接：请求到达后端） |

## 踩坑

- 三个请求都 404 → 后端本来就没有 `a.html`，是预期；区分谁产生的 404：`Server:` 头会被代理换成 nginx，看 `charset=iso-8859-1`（Apache 错误页特征，nginx 自己的 404 不带）。
- `sudo tail -n 3 /var/log/httpd/*access_log` 报 No such file → `*` 由 fqh 的 shell 在 sudo 之前展开，`/var/log/httpd` 是 0700 root，匹配失败按字面传给 tail → `sudo sh -c '...'` 或写完整路径。（命令是 Claude 给错的。）

## 卡点 / 没想明白的

- 第 5 题按 `/p2/` 去替换，实际替换的是字面 `/p2`；和 B1-3 的 `/labx` 是同一个误区（把字符串当路径）。

## 面试题（5 道，对话中作答后批改）

1. （复述）`proxy_pass` 写了 URI 部分时，nginx 怎么处理请求 URI？
<details><summary>批改要点</summary>

把 location 匹配到的部分替换成 `proxy_pass` 里写的 URI。
</details>

2. （原因）后端日志来源 IP 为什么是 127.0.0.1？
<details><summary>批改要点</summary>

后端看到的是 nginx 新建的连接②；真实客户端 IP 要靠 `X-Forwarded-For` 传。
</details>

3. （应用）`location /api/` 把 `/api/users` 转成后端 `/v2/users`，怎么写？
<details><summary>批改要点</summary>

`proxy_pass http://127.0.0.1:8090/v2/;`
</details>

4. （排障）客户端收到 502，第一步看什么？
<details><summary>批改要点</summary>

`error.log`：`111 Connection refused` → 后端没监听；`13 Permission denied` → SELinux，`setsebool -P httpd_can_network_connect 1`；超时通常是 504。
</details>

5. （反事实）`location /p2` + `proxy_pass http://127.0.0.1:8090/;`，请求 `/p2/a.html`，后端收到什么？
<details><summary>批改要点</summary>

`//a.html`：替换的是字面 `/p2`。
</details>

作答结果：1–3 对（第 3 题 IP 笔误写成 172.0.0.1、漏分号）；第 4 题没说出 `error.log`；第 5 题答成 `/a.html`。

## 复习记录

- [x] D+1（11-10）
- [ ] D+3
- [ ] D+7

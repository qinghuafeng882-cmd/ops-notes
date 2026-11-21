# W7 速查：LNMP / WordPress / Keepalived

## B3-1 PHP-FPM + WordPress

| 场景 | 命令 / 配置 |
|---|---|
| 命令属于哪个包（未装） | `dnf provides getfacl`（或 `'*/getfacl'`） |
| 文件属于哪个包（已装） | `rpm -qf /path/file` |
| 包里有哪些文件 | `rpm -ql php-fpm` |
| 按进程名看进程 | `ps -o user,pid,ppid,cmd -C php-fpm` |
| 进程启动时间 | `ps -o pid,lstart,cmd -p PID` |
| nginx master PID | `cat /run/nginx.pid` |
| 看 socket ACL | `getfacl /run/php-fpm/www.sock` |
| 端口归谁 | `sudo ss -lntp \| grep -E ':(80\|8080) '` |
| 打印完整生效配置 | `sudo nginx -T` |
| 改配置后 | `sudo nginx -t && sudo systemctl reload nginx` |
| 只看响应头 | `curl -I URL` |
| SELinux 持久标签 | `semanage fcontext -a -t httpd_sys_content_t '/srv/wordpress(/.*)?'` + `restorecon -Rv /srv/wordpress` |
| 可写目录标签 | `httpd_sys_rw_content_t` |
| SELinux 放行端口 | `semanage port -a -t http_port_t -p tcp 8082` |

关键配置：

```nginx
location / { try_files $uri $uri/ /index.php?$args; }
location ~ \.php$ {
    include fastcgi_params;   # /etc/nginx/fastcgi_params，不带 $
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass php-fpm;     # upstream 在 conf.d/php-fpm.conf
}
```

- www.conf：`user=apache`、`listen=/run/php-fpm/www.sock`、`listen.acl_users=apache,nginx`、`pm=dynamic`、`pm.max_children=50`。
- 权限：程序 root:root 755；`wp-content` apache:apache；`wp-config.php` root:apache 640。
- 建库：`CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`，账号只授权 `wordpress.*` @ localhost。
- 重置管理员：`UPDATE wp_users SET user_pass=MD5('...') WHERE user_login='admin';`（必须带 WHERE）。

502 速判：

| error.log 括号内 | 根因 |
|---|---|
| No such file or directory | PHP-FPM 没跑 / socket 路径不一致 |
| Permission denied | nginx 无 socket 权限（ACL） |
| Connection refused | TCP 端口无人监听 |
| （504）timed out | 上游慢，看 slowlog |

## B3-2 Nginx HTTPS

| 做什么 | 命令 / 配置 |
|---|---|
| 一步生成私钥+自签证书 | `openssl req -x509 -newkey rsa:2048 -nodes -days 365 -keyout X.key -out X.crt -subj "/CN=IP" -addext "subjectAltName=IP:IP"` |
| 查看证书字段 | `openssl x509 -in X.crt -noout -subject -dates -ext subjectAltName`（全部：`-text -noout`） |
| 证书/私钥位置 | `/etc/pki/tls/certs/`（644）、`/etc/pki/tls/private/`（私钥 600 root） |
| 启用 TLS | `listen 443 ssl;` + `ssl_certificate` + `ssl_certificate_key` + `ssl_protocols TLSv1.2 TLSv1.3;` |
| HTTP 跳 HTTPS | `return 301 https://$host$request_uri;` |
| 防火墙 | `firewall-cmd --add-service=https --permanent && firewall-cmd --reload` |
| 客户端信任自签证书 | 临时 `curl --cacert X.crt URL`；系统级放 `/etc/pki/ca-trust/source/anchors/` + `update-ca-trust` |

curl 报错速判：

| 现象 | 含义 |
|---|---|
| `(60) self-signed certificate` | 证书不受信任（预期），`-k` 可跳过校验但仍加密 |
| `(35) ... packet length too long` | 握手失败：443 没写 `ssl`，对端讲明文 HTTP |
| 握手成功但 403 + error.log `Permission denied` | 多半 SELinux `var_t`，`ls -Z` 确认 |

## access log 汇总（11-18）

| 做什么 | 命令 |
|---|---|
| 生效的日志路径 | `sudo nginx -T 2>/dev/null \| grep -E 'access_log\|log_format'` |
| 看 main 及后 3 行 | `sudo nginx -T \| grep -A 3 'log_format'` |
| 最近两行 | `sudo tail -n 2 /var/log/nginx/access.log` |
| 本课格式拆 IP、路径、状态码 | `sudo awk '{print $1,$7,$9}' /var/log/nginx/access.log` |

- 本课：`access_log /var/log/nginx/access.log main;`。`$9` 是这条格式的状态码，`$7` 是路径。awk 按空白拆，双引号不成组，不要套到别的格式。
- `nginx -T` 的两行检查提示在 stderr，不进管道；要安静加 `2>/dev/null`。原文未抄。
- `grep -A` = After，后 N 行含空行。
- `sort | uniq -c`：`uniq` 只合并相邻相同行。`wc -l 文件` 带文件名，`wc -l < 文件` 只有行数。
- `bash -n` 只查语法、不执行；无输出不能证明文件存在。`>&2` 错误走 stderr。
- 课堂脚本只此版（`~/nginx-log-summary.sh`）。空文件、无参数、空格路径、普通用户不可读、并发写，都未验证。

```bash
#!/bin/bash
log_file="$1"
if [ ! -r "$log_file" ]; then
 echo "文件不可读，退出！" >&2
 exit 1
fi
echo "请求总数：$(wc -l < "$log_file")"
echo "状态码统计如下："
awk '{print $9}' "$log_file" | sort | uniq -c
```

## Keepalived VIP 与健康检查（11-18、11-19）

| 做什么 | 要点 |
|---|---|
| 选 VIP | DHCP 服务器 192.168.56.100，池 .101–.254。`arping -D -I enp0s8 -c 3 192.168.56.200` 无回应，但在池内，不选。`arping -D -I enp0s8 -c 3 192.168.56.50` 无回应且在池外，选它。无回应不等于绝对空闲 |
| 页面 | `/etc/nginx/conf.d/ha-test.conf`：`listen 8088;`、`server_name _;`、`location /` 里 `default_type text/plain;`、`return 200 "ha test from node1\n";`。node2 只换标签 |
| 放行 8088 | `semanage port -a -t http_port_t -p tcp 8088`；`firewall-cmd` permanent 开 `8088/tcp` 再 reload |
| VRRP | IP 协议 112，不是 TCP/UDP 端口。node1：`sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.12" protocol value="vrrp" accept'`。node2 的 source 换成 `192.168.56.11`。然后 `sudo firewall-cmd --reload` |
| VI_1 | 两台 `state BACKUP`，`interface enp0s8`，`virtual_router_id 51`，priority 150/100，`advert_int 1`，unicast 指向对方，VIP `.50/24 dev enp0s8`。node2 的 `enp0s8` 上另有 192.168.56.22/24，用途未知，未处理 |
| 看地址 | `ip -br -4 addr show`。`ip -br -4 show` 少了 `addr`。VIP：`curl --connect-timeout 2 --max-time 5 http://192.168.56.50:8088/` |
| 启停 Keepalived | `rpm -q keepalived` 为 `keepalived-2.2.8-9.el10.x86_64`；最初 `systemctl is-active keepalived` 是 inactive。备份 `sudo cp -a /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.before-ha`。切换用 `sudo systemctl start/stop keepalived` |
| 健康检查 | `weight 0`：失败进入 FAULT，不是只降优先级。`fall 2` 连续失败，`rise 2` 连续成功。手动跑脚本只把退出码交给终端，不会移动 VIP；日志是 Keepalived 自己的调度 |
| 日志里的 0 priority | `sent 0 priority` 是退出 MASTER 的通告，配置里的 150 没有被改成 0 |
| 开机 | `systemctl enable` 只设置开机，不等于 `start`。11-19 当天 enable 之后没有再重启。11-20 做了正常 reboot：两台都没有手动 start，nginx 与 keepalived 自己 active；node1 起来后 `enp0s8` 有 .11 和 .50。截图 08:23:15–18 是 node2，08:23:19 起是 node1。不是断电，也不能写成零丢失或精确中断时刻 |

检查脚本（root:root 755），返回码就是最后这条 curl 的返回码。0 成功，非 0 失败。退出码 2 是初始化失败；停掉 Nginx 时看到过 7，没有 curl 报错原文。

```bash
#!/bin/bash
/usr/bin/curl --noproxy '*' -fsS \
  --connect-timeout 1 --max-time 2 \
  http://127.0.0.1:8088/ >/dev/null 2>&1
```

```text
global_defs {
    script_user keepalived_script
    enable_script_security
}
vrrp_script chk_nginx {
    script "/usr/local/bin/check-nginx-ha.sh"
    interval 3
    timeout 3
    fall 2
    rise 2
    weight 0
}
```

`track_script { chk_nginx }` 写在 VI_1 里。

11-18 只口答了「没有健康检查时只停 Nginx 不会切走」，没有实测停 Nginx。11-19 已经实测停/启 node1 的 Nginx：VIP 自动让出，恢复后 node1 抢回。11-19 当天没有再重启。

11-20 正常 reboot 已测，见 `labs/2026-11-20-ha-reboot-rhcsa-mock2.md`。仍未做：突然断电、两台同时故障、网络隔离、数据同步。`before-rhcsa2` 是两台各自的开机快照，不是同一时刻的一致性快照，也不是数据库备份。

## 11-20 模拟②分段与 ACL

| 做什么 | 要点 |
|---|---|
| SGID | `chmod 2770`。新建文件属组跟目录。770 再改 2770 时，前一次可以不写 |
| 精确 location | `location = /exam-health`。`/exam-health-extra` 不走这条。改前要先备份；`nginx -t` 失败不一定只是语法 |
| 状态码个数 | `sort \| uniq -c` 前先排序。数字倒序是 `sort -nr`，`sort -r` 是文字序。`sudo` 只管 awk |
| 本次启动 | `sudo journalctl -b -u nginx -n 10 --no-pager`。`-n 10` 是最多 10 条 |
| 监听与防火墙 | `sudo ss -lntp 'sport = :8088'` 看到 `0.0.0.0:8088` 不等于防火墙已放行。活动 zone 是 `--get-active-zones`，不是 `--active-zone` |
| ACL | `sudo setfacl -m u:exam02:rx /srv/ops-exam`。有效权限是条目和 mask 的交集。`-x` 删条目。11-20 没讲默认 ACL |

`firewall-cmd --reload` 和 `--query-port=8088/tcp` 在 11-20 只分别是口答和举例，没有执行。`--list-ports` 里当时有 `8088/tcp`。

## 11-21 默认 ACL

`sudo setfacl -m d:u:exam02:rx /srv/ops-exam`。漏 `-m` 会 `Usage`。

- 只影响之后新建的对象，不改已有的。它是创建时的模板，不是事后持续同步。11-21 没有拿旧文件做前后对照。
- 新建目录同时有访问 ACL 和 `default:` 条目。新建的普通文件只有访问 ACL。
- SGID 管新建对象的属组。默认 ACL 管新建对象的 ACL 条目。
- 这次 `touch` 出来的文件 mask 是 `rw-`（请求的 mode 通常是 0666，不含 x）。exam02 的 `r-x` 有效是 `r--`。不要写成默认 ACL 会给普通文件执行权。
- `mkdir` 请求 x，这次新目录的 mask 是 `rwx`。
- fqh 没有目录穿越权时，`getfacl` 是 `Permission denied`，用 `sudo getfacl`。
- 没做：`cd` 进新目录、验证写被拒绝、删除默认规则、`chmod`、重启后还在不在。

`uniq` 是合并统计相邻相同行，不是排序。复制字段是 `Last_IO_Error`，不是 `Last_IO_Errors`。

# W7 速查：LNMP / WordPress

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

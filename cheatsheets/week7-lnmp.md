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

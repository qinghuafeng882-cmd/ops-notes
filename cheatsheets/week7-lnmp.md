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

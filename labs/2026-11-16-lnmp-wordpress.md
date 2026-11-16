# LNMP 部署 WordPress：PHP-FPM、FastCGI、权限与 SELinux、502 演练（B3-1）

- 日期：2026-11-16
- 考点：RHCSA EX200 —— 软件包（dnf provides / rpm -qf）、文件权限与 ACL、SELinux 文件上下文与端口标签、systemd 服务；面试高频 LNMP 排障
- 环境：Rocky Linux 10.2（VirtualBox），node1 192.168.56.11；Nginx 1.26.3、PHP-FPM 8.3.36、MySQL 8.4.11、WordPress 7.1.4
- 状态：完成

## 目标

在 node1 上用 Nginx + PHP-FPM + MySQL 部署 WordPress，理解请求链路，并做一次 502 故障演练。

## 原理速记

- CGI（Common Gateway Interface，通用网关接口）：每个请求启动一个进程，开销大 → FastCGI：常驻进程池，复用进程。
- PHP 是解释器；PHP-FPM（FastCGI Process Manager，FastCGI 进程管理器）是一个服务，管理一池 PHP worker，听 FastCGI 协议。
- WordPress 不是服务、没有自己的进程：只是一堆 .php 文件，每个请求由 PHP-FPM 的 worker 执行一次。
- 请求链路：浏览器 → Nginx（location 匹配 `\.php$`）→ `fastcgi_pass` → Unix socket `/run/php-fpm/www.sock` → PHP-FPM worker（apache 用户）执行 index.php → 连 MySQL → 返回。
- `fastcgi_pass` 对后端说 FastCGI 协议；`proxy_pass` 说 HTTP。

## 关键步骤

```bash
# 环境：php-cli 未装导致 php -v 找不到命令（php-fpm 包不含 php 命令行）
sudo dnf install -y php-cli
php -v                                  # 8.3.36 (cli) NTS
rpm -ql php-fpm                         # 看包装了哪些文件

# PHP-FPM 配置（只读查看）：/etc/php-fpm.d/www.conf
#   user=apache  listen=/run/php-fpm/www.sock  listen.acl_users=apache,nginx
#   pm=dynamic  pm.max_children=50  start_servers=5  min/max_spare 5/35
cat /etc/nginx/conf.d/php-fpm.conf      # upstream php-fpm { server unix:/run/php-fpm/www.sock; }

# 启动并看进程、socket
sudo systemctl enable --now php-fpm
ls -l /run/php-fpm/www.sock             # srw-rw----+ root root（+ = 有 ACL）
ps -o user,pid,ppid,cmd -C php-fpm      # master=root，5 个 pool www worker=apache
dnf provides getfacl                    # 反推包名 → acl
getfacl /run/php-fpm/www.sock           # user:apache:rw- user:nginx:rw- mask::rw-

# 默认站点自带 PHP 配置
cat /etc/nginx/default.d/php.conf
grep -n 'default.d' /etc/nginx/nginx.conf
grep -n 'root' /etc/nginx/nginx.conf

# 探针页（测完删除）
echo "<?php phpinfo();  ?>" | sudo tee /usr/share/nginx/html/info.php
curl -s http://localhost/info.php       # 404，但这是 Apache 的 404
sudo nginx -T 2>/dev/null | grep -nE 'server \{|root|listen|proxy_pass|server_name|include /etc/nginx/(conf|default)'
sudo ss -lntp | grep -E ':(80|8080|8081)'   # 80=httpd，8080/8081=nginx
curl -s http://localhost:8080/info.php  # 原样返回 PHP 源码
ps -o pid,lstart,cmd -p 1694            # nginx master 启动于 11-15，早于装 php-fpm
sudo nginx -t && sudo systemctl reload nginx
curl -s http://localhost:8080/info.php | grep -i 'Server API'   # FPM/FastCGI
sudo rm -f /usr/share/nginx/html/info.php

# 数据库与账号（最小权限）
mysql -u root -p
#   CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
#   CREATE USER 'wpuser'@'localhost' IDENTIFIED BY '...';
#   GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
#   SHOW GRANTS FOR 'wpuser'@'localhost';

# 下载解压
cd /tmp
curl -OL https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz -C /srv/
ls -ld /srv/wordpress && ls /srv/wordpress | head

# 权限：程序归 root，只把 wp-content 给 apache
sudo chown -R apache:apache /srv/wordpress/wp-content
ls -Zd /srv/wordpress /srv/wordpress/wp-content    # var_t

# SELinux 持久化标签
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/wordpress(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/srv/wordpress/wp-content(/.*)?'
sudo restorecon -Rv /srv/wordpress | tail -3
ls -Zd /srv/wordpress /srv/wordpress/wp-content

# SELinux 端口
sudo semanage port -l | grep -w http_port_t
sudo semanage port -a -t http_port_t -p tcp 8082   # 提示 already defined, modifying instead

# 站点配置
sudo tee /etc/nginx/conf.d/wordpress.conf << 'EOF'
server {
  listen 8082;
  server_name _;
  root /srv/wordpress;
  index index.php;

  location / {
    try_files $uri $uri/ /index.php?$args;
  }

  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass php-fpm;
  }
}
EOF
sudo nginx -t && sudo systemctl reload nginx
sudo firewall-cmd --permanent --add-port=8082/tcp && sudo firewall-cmd --reload
curl -I http://localhost:8082/          # 302 → /wp-admin/setup-config.php

# 安装向导写不了 wp-config.php（目录归 root），手动创建
cd /srv/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo vim wp-config.php                  # DB_NAME/DB_USER/DB_PASSWORD='...'/DB_HOST=localhost
sudo chown root:apache wp-config.php && sudo chmod 640 wp-config.php
ls -lZ wp-config.php                    # -rw-r-----. root apache httpd_sys_content_t
mysql -u wpuser -p wordpress -e 'SHOW TABLES;'   # 12 张 wp_ 表

# 忘记管理员密码 → 数据库重置
#   SELECT ID, user_login, LEFT(user_pass, 10) FROM wp_users;   -- $wp$2y$10$（bcrypt）
#   UPDATE wp_users SET user_pass = MD5('...') WHERE user_login = 'admin';
#   之后哈希变为 32 位十六进制 MD5；登录后 WordPress 会自动升级为新格式（推测，未再次查询验证）

# 故障演练：502
sudo systemctl stop php-fpm
curl -I http://localhost:8082/          # 502 Bad Gateway
sudo tail -2 /var/log/nginx/error.log   # connect() to unix:/run/php-fpm/www.sock failed (2: No such file or directory)
sudo systemctl start php-fpm
curl -I http://localhost:8082/          # 200 OK
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| PHP 链路 | `curl -s localhost:8080/info.php \| grep 'Server API'` | FPM/FastCGI | ✅ |
| nginx 能访问 socket | `getfacl /run/php-fpm/www.sock` | 有 `user:nginx:rw-` | ✅ |
| 数据库建表 | `mysql -u wpuser -p wordpress -e 'SHOW TABLES;'` | 12 张 wp_ 表 | ✅ |
| 后台登录 | 浏览器 `http://192.168.56.11:8082/wp-admin/` | 进入 Dashboard | ✅ |
| 站点可用 | `curl -I localhost:8082/` | 200 | ✅ |
| php-fpm 开机自启 | `systemctl enable --now php-fpm` | 创建 wants 软链接 | ✅（已 enable，重启未验证） |
| 重启后仍生效 | 重启 node1 后 curl | 200 | 未验证 |

## 踩坑

- `php -v` 报 command not found → php-fpm 包不含命令行解释器 → `dnf install php-cli`。
- grep 配置时漏了 `^(...) ` 锚点，输出一堆注释 → `^` 锚定行首、名字后跟空格才精确。
- `getfacl` 命令不存在 → 没装 acl 包 → `dnf provides getfacl` 反推包名。
- `curl localhost/info.php` 返回 404 → 那是 Apache 的 404，80 端口是 httpd，Nginx 只听 8080/8081 → `ss -lntp` 确认端口归属，改测 8080。
- 8080 返回 PHP 源码原文 → nginx master 是 11-15 启动的，没读到装 php-fpm 后新增的 default.d/php.conf → `nginx -t && systemctl reload nginx`。
- `semanage port -a -t http_port_t tcp 8082` 报 unrecognized arguments → 漏了 `-p` → 加 `-p tcp`。
- `include $fastcgi_params;` → `open() "/etc/nginx/$fastcgi_params" failed` → 多了 `$`，Nginx 当字面文件名 → 改成 `include fastcgi_params;`（相对 /etc/nginx/）。
- 安装向导 Unable to write to wp-config.php → 程序目录归 root（有意为之）→ 手动 cp 样例、改 4 行、`root:apache 640`。
- 安装时没注意密码 → 安装页自动预填随机强密码 → SQL `UPDATE ... MD5('...') WHERE user_login='admin'` 重置；第一次把列名写成 name_login 报 1054。
- 明文密码出现在 mysql 命令行 → 会留在 `~/.mysql_history`，生产中应改密并清理历史。

## 卡点 / 没想明白的

- PHP 与 PHP-FPM 的关系（已讲：解释器 vs 进程管理服务）。
- 「谁启动 WordPress」：WordPress 无进程，由 PHP-FPM worker 按请求执行。
- 302 的因果链：index.php → 找不到 wp-config.php → 判定未安装 → 302 到 setup-config.php，Nginx 只转交（`X-Powered-By: PHP` 为证）。
- ACL 与普通权限：不是"优先级"，而是按 属主 → 命名用户 → 组 → other 顺序匹配，命中即停；命名条目还要与 mask 取交集。

## 面试题（5 道，对话中作答后批改）

1. （排障）PHP 网站报 502，error.log 先看什么？不同关键词指向什么？ —— 1/1
<details><summary>批改要点</summary>

看 `connect() ... failed (...)` 括号里的原因：No such file or directory → PHP-FPM 没运行或 socket 路径不一致；Permission denied → nginx 用户无 socket 权限（ACL/listen.owner）；Connection refused → TCP 端口无人监听。504 则是上游连上但超时，看 slowlog。
</details>

2. （概念）为什么只把 wp-content 给 apache，整个目录仍归 root？ —— 0.5/1
<details><summary>批改要点</summary>

最小权限：程序代码 apache 只读，被攻破后也无法篡改 PHP 代码、植入后门/挂马；可写范围压到上传/插件目录。扣分：说成"防止整机沦陷"——PHP 本就以 apache 运行，问题是网站自身被篡改。
</details>

3. （取舍）`chcon -t` 与 `semanage fcontext` + `restorecon` 区别？ —— 1/1
<details><summary>批改要点</summary>

chcon 只改文件上的标签；semanage 改策略库，restorecon 按策略库打标签。restorecon 或全盘 relabel 会覆盖 chcon 的修改。
</details>

4. （排障）`nginx -T` 能看到新配置，但网站表现仍是旧配置，原因与处理？ —— 0.5/1
<details><summary>批改要点</summary>

运行中的 master 只在启动/reload 时读配置，`-T` 每次读磁盘。用 `ps -o lstart` 对比 master 启动时间。处理：`sudo nginx -t && sudo systemctl reload nginx`。扣分：漏了先 `-t` 和原理说明。
</details>

5. （操作/原理）访问 `/` 返回 302 到 setup-config.php，谁发的、为什么？ —— 0.5/1
<details><summary>批改要点</summary>

PHP-FPM 执行 WordPress 的 index.php 后由 WordPress 返回 302，Nginx 只转交（`X-Powered-By: PHP`）。原因：wp-config.php 不存在 → 判定未安装 → 引导到安装向导。扣分：因果链没说完整。
</details>

**总分：3.5/5。** 薄弱点：reload 前先 `nginx -t`；302 的因果链。

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

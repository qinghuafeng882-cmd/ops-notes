# 第 6 周速查：MySQL 8.4（Rocky 10）

## B2-1：安装、连接、账号、远程访问

```bash
sudo dnf install -y mysql8.4-server        # 服务端；客户端单独包 mysql8.4；与 MariaDB 互斥
sudo systemctl enable --now mysqld
sudo ss -lntp | grep mysqld                # 3306 经典协议，33060 X 协议
mysql -u root -p                           # 不加 -h / -h localhost → socket
mysql -u root -p -h 127.0.0.1              # 写 IP → TCP
mysql -u app -p -h 192.168.56.11 labdb     # 末尾库名 = 登录后 USE labdb
mysql ... -e 'SQL'                         # 执行一条就退出
perror 113                                 # 错误码 → 文字（服务端包自带）
sudo firewall-cmd --permanent --add-service=mysql && sudo firewall-cmd --reload
sudo semanage port -l | grep -w 3306       # mysqld_port_t（默认已有）
```

```sql
status                                     -- Connection: 行看走的是 socket 还是 TCP
SELECT user, host, plugin FROM mysql.user;
SELECT CURRENT_USER();                     -- 实际匹配到的账号
ALTER USER 'root'@'localhost' IDENTIFIED BY '...';
CREATE USER 'app'@'192.168.56.12' IDENTIFIED BY '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON labdb.* TO 'app'@'192.168.56.12';
SHOW GRANTS FOR 'app'@'192.168.56.12';     -- GRANT USAGE ON *.* = 无全局权限
```

| 报错 | 卡在哪层 |
|---|---|
| `2003 (111)` | 没人监听 |
| `2003 (113)` | 防火墙 reject |
| `2003 (110)` | 丢包 / 不通 |
| `1130 Host ... not allowed` | 没有匹配来源的账号（还没查密码） |
| `1045 Access denied` | 密码错；`using password: NO` = 没加 `-p` |

⚠️ MySQL 客户端里 `localhost` = socket，不是 127.0.0.1。
⚠️ 账号 `'user'@'host'`：host 是客户端来源；同名不同 host 是两个账号；和 Linux 账号无关。
⚠️ host 匹配会用反向解析的主机名（127.0.0.1 → localhost）；生产常 `skip_name_resolve=ON`，账号写 IP。
⚠️ RHEL 装完 root 空密码，先 `ALTER USER` 设密码。
⚠️ 8.x：先 `CREATE USER` 再 `GRANT`（GRANT 不再建用户）；不需要 `FLUSH PRIVILEGES`。
⚠️ 不开 `'root'@'%'`，应用单独建最小权限账号。
⚠️ `-p` 后不跟密码（会进 bash 历史和 ps）。

# MySQL 第一课：安装、连接方式、账号与远程访问（B2-1）

- 日期：2026-11-11
- 考点：运维面试 —— MySQL 安装、socket/TCP、'user'@'host'、远程连接逐层排障
- 环境：Rocky Linux 10.2（VirtualBox），node1 192.168.56.11 服务端，node2 192.168.56.12 客户端
- 状态：完成

## 目标

在 node1 装 MySQL 8.4，弄清客户端怎么连上服务端，再让 node2 用最小权限账号远程连上。

## 关键步骤

```bash
# node1：安装并启动（服务端包 mysql8.4-server，服务名 mysqld）
free -h
sudo dnf install -y mysql8.4-server
sudo systemctl enable --now mysqld
sudo ss -lntp | grep mysqld            # *:3306（经典协议）+ *:33060（X 协议）
ls -l /var/lib/mysql/mysql.sock        # 首字符 s = socket 文件，权限 rwxrwxrwx

# 两种连接方式
mysql -u root                          # 不加 -h → socket；装完 root 空密码，普通用户直接进
mysql -u root -h 127.0.0.1 -e 'status' | grep -i connection   # 127.0.0.1 via TCP/IP
```

```sql
SELECT user, host, plugin FROM mysql.user;     -- 只有 root@localhost + 3 个内部账号
ALTER USER 'root'@'localhost' IDENTIFIED BY '...';   -- 设 root 密码（实验密码不入库）
```

```bash
mysql -u root                          # ERROR 1045 ... (using password: NO)
mysql -u root -p -h 127.0.0.1 -e 'SELECT CURRENT_USER();'   # root@localhost

# node2：只装客户端
sudo dnf install -y mysql8.4
mysql -u root -p -h 192.168.56.11      # ERROR 2003 ... (113)  ← 防火墙 reject

# node1：查错误码、验 SELinux、放行防火墙
perror 113                             # No route to host
sudo semanage port -l | grep -w 3306   # mysqld_port_t：1186, 3306, 63132-63164
sudo firewall-cmd --permanent --add-service=mysql
sudo firewall-cmd --reload

# node2 再连
mysql -u root -p -h 192.168.56.11      # ERROR 1130: Host 'node2' is not allowed ...
```

```sql
-- node1：建库 + 最小权限账号（不开放 root 远程）
CREATE DATABASE labdb;
CREATE USER 'app'@'192.168.56.12' IDENTIFIED BY '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON labdb.* TO 'app'@'192.168.56.12';
SHOW GRANTS FOR 'app'@'192.168.56.12';   -- 多一行 GRANT USAGE ON *.* = 无任何全局权限
```

```bash
# node2
mysql -u app -p -h 192.168.56.11 labdb   # CURRENT_USER() = app@192.168.56.12
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 不加 -h 走 socket | `status` | Localhost via UNIX socket | ✅ |
| -h 127.0.0.1 走 TCP | `status` | 127.0.0.1 via TCP/IP | ✅ |
| root 空密码已封 | `mysql -u root` | ERROR 1045 (using password: NO) | ✅ |
| 127.0.0.1 匹配 root@localhost | `SELECT CURRENT_USER();` | root@localhost | ✅ |
| 3306 SELinux 类型 | `semanage port -l` | mysqld_port_t | ✅ |
| node2 远程登录 | `mysql -u app -p -h 192.168.56.11 labdb` | 进入，CURRENT_USER = app@192.168.56.12 | ✅ |
| app 看得到的库 | `SHOW DATABASES;` | information_schema、labdb、performance_schema | ✅ |
| 重启后仍生效 | node1 `systemctl is-active nginx httpd mysqld` + `ss -lntp`；node2 `mysql -u app -p -h 192.168.56.11` | 均 active；3306/33060 在监听；app 登录成功 | ✅（11-12，mysqld 重启后） |
| 远程输错密码 | node2 `mysql -u app -p -h 192.168.56.11` 输错密码 | ERROR 1045 | ✅（11-12） |

## 结论

- MySQL 客户端里 `localhost` = 走 socket，不是 127.0.0.1；写 IP 才走 TCP。
- 账号 = `'user'@'host'`，host 是**客户端来源**；同名不同 host 是两个账号。MySQL 账号和 Linux 账号无关。
- mysqld 匹配 host 时会拿 IP 和反向解析出的主机名比对（127.0.0.1 → localhost；192.168.56.12 → 报错里显示 node2）。生产常开 `skip_name_resolve=ON`，账号一律写 IP。
- 远程连通逐层：监听（默认 `*:3306`）→ SELinux（3306 已是 mysqld_port_t）→ 防火墙 → 账号 host → 密码。

| 报错 | 卡在哪层 | 处理 |
|---|---|---|
| `ERROR 2003 (111)` | 端口没人监听 | 查 mysqld 是否运行、监听地址 |
| `ERROR 2003 (113)` | 防火墙 reject | firewalld 放行 mysql |
| `ERROR 2003 (110)` | 丢包 / 网络不通 | 查 drop 规则、路由 |
| `ERROR 1130` | 没有匹配来源主机的账号（还没核对密码） | 建/改账号的 host 部分，改密码没用 |
| `ERROR 1045` | 账号匹配到了，密码错或没发（`using password: NO` = 忘了 `-p`） | 查密码 |

## 踩坑

- 装完 root 空密码，任何 Linux 用户 `mysql -u root` 即获最高权限 → RHEL 打包初始化时 root 设空密码（推测为 `--initialize-insecure`）→ 立刻 `ALTER USER` 设密码。
- MySQL 8：`GRANT` 不再顺带建用户，先 `CREATE USER` 再 `GRANT`；`CREATE USER`/`GRANT` 立即生效，不需要 `FLUSH PRIVILEGES`。
- `-p密码` 会进 bash 历史和 `ps`；只写 `-p`。mysql 客户端默认不把含 `IDENTIFIED`/`PASSWORD` 的语句写进 `~/.mysql_history`。
- （11-12）想看当前匹配账号时写了 `SHOW CURRENT_USER()` → `CURRENT_USER()` 是函数，不是 SHOW 的对象 → 改成 `SELECT CURRENT_USER();`。

## 卡点 / 没想明白的

- 远程场景下的 ERROR 1045（如 app 输错密码）还没实际触发过。
  - 已触发（11-12）：node2 上 `mysql -u app -p -h 192.168.56.11` 输错密码 → `ERROR 1045 (28000): Access denied for user 'app'@'node2' (using password: YES)`。报错里显示 `node2` 而不是 IP，是因为 node1 的 `/etc/hosts` 能把 192.168.56.12 解析成 node2。

## 面试题（5 道，对话中作答后批改）

1. `mysql -u root -h localhost` 走 socket 还是 TCP？
<details><summary>批改要点</summary>

socket。MySQL 客户端把 localhost 当作"走 socket 文件"，写 IP 才走 TCP。
</details>

2. node2 连 node1 报 `ERROR 2003 (HY000): Can't connect ... (111)`，卡在哪一层？
<details><summary>批改要点</summary>

111 = Connection refused：包到了 node1，但 3306 没人监听（mysqld 没运行或没监听该地址）。不是防火墙（那是 113）。
</details>

3. 远程登录报 `ERROR 1130 Host ... is not allowed`，该去改什么？
<details><summary>批改要点</summary>

改账号的 host 部分：建一个 host 匹配客户端来源的账号（如 `'app'@'192.168.56.12'`）。此时还没核对密码，改密码没用。
</details>

4. 报 `ERROR 1045 ... (using password: NO)`，最可能是什么原因？
<details><summary>批改要点</summary>

客户端没发密码，通常是忘了加 `-p`。
</details>

5. 写出语句：给 `'app'@'192.168.56.12'` 授予 labdb 库所有表的 SELECT 权限。
<details><summary>批改要点</summary>

`GRANT SELECT ON labdb.* TO 'app'@'192.168.56.12';`（账号要先用 CREATE USER 建好）
</details>

## 复习记录

- 课后（11-11）：4/5。第 2 题只答「网络层没建立连接」，没区分 111/113/110（111 = 没人监听）。
- [x] D+1（11-12，5/5，2003 的 111 refused / 113 No route to host / 110 timed out 已分清）
- [x] D+3（11-14，4/5；第 4 题将 SELECT CURRENT_USER(); 误写为 SHOW CURRENT_USER();）
- [ ] D+7

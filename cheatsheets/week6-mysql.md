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
⚠️ 报错里的 `'app'@'node2'` 是客户端来源（能反解就显示主机名），不是账号定义；实际匹配到哪个账号看 `SELECT CURRENT_USER();`（不是 `SHOW`）。
⚠️ RHEL 装完 root 空密码，先 `ALTER USER` 设密码。
⚠️ 8.x：先 `CREATE USER` 再 `GRANT`（GRANT 不再建用户）；不需要 `FLUSH PRIVILEGES`。
⚠️ 不开 `'root'@'%'`，应用单独建最小权限账号。
⚠️ `-p` 后不跟密码（会进 bash 历史和 ps）。


## B2-2：mysqldump 备份恢复 + binlog 时间点恢复

```bash
mysqldump -u root -p --single-transaction --source-data=2 --databases labdb > ~/backup/labdb-full.sql
grep -n 'CHANGE REPLICATION SOURCE' 备份文件     # 备份对应的 binlog 文件 + 位置
mysql -u root -p < 备份文件                      # 恢复（--databases 导出的自带建库和 USE）
sudo mysqlbinlog --base64-output=decode-rows -v --start-position=N 文件 | grep -E '^# at |^### |DROP'   # 只用来看
sudo -v                                          # 先输 sudo 密码，避免和 mysql 抢终端
sudo mysqlbinlog --start-position=备份位置 --stop-position=误操作起点 文件 | mysql -u root -p   # 重放
```

```sql
SELECT table_name, engine FROM information_schema.tables WHERE table_schema = 'labdb';
SHOW VARIABLES LIKE 'log_bin%';      -- 8.x 默认 ON，文件在 /var/lib/mysql/binlog.0000NN
SHOW BINARY LOG STATUS;              -- 当前 binlog 文件 + 位置（8.4 无 SHOW MASTER STATUS）
```

| 选项 | 作用 |
|---|---|
| `--single-transaction` | 开事务拿快照，导出读同一时刻，不阻塞写；仅 InnoDB，期间不能 DDL |
| `--lock-all-tables` | 全局读锁，期间阻塞写 |
| `--source-data=2` | binlog 位置写成注释（`=1` 可执行，搭从库用；旧名 `--master-data`） |
| `--databases 库` | 带 `CREATE DATABASE` + `USE` |

⚠️ 复制不等于备份：误删会一起复制过去。
⚠️ 恢复顺序：先全量备份，再重放 binlog；终点是误操作那组事件的起点（`# at` 开头标记，不是 DROP 语句那行）。
⚠️ `decode-rows -v` 输出的 `###` 是注释，拿去重放什么也不执行。
⚠️ 备份文件带 `DROP TABLE IF EXISTS`，旧备份导进正在用的库会覆盖新数据；生产上先恢复到别处。
⚠️ `>` 由 bash 先处理：目录不存在时 mysqldump 不会运行；mysqldump 失败也会留下空文件或被清空的文件，要看大小和内容。
⚠️ 误删后先止损：停写入、保护 binlog（别重启折腾、别清 binlog）。
⚠️ `information_schema` 是库，表要写 `information_schema.tables`。


## B2-3：GTID 主从复制（8.4 新语法）

```ini
# /etc/my.cnf.d/mysql-server.cnf 的 [mysqld] 下
server_id = 1                      # 主从必须唯一（主 1、从 2）
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON                     # 仅从库加：拦普通账号写（ERROR 1290），不拦复制线程和 root
```

```sql
-- 主库建最小权限复制账号（必须 ON *.*）
CREATE USER 'repl'@'192.168.56.12' IDENTIFIED BY '...';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.56.12';

-- 从库清 GTID 历史后导入主库 mysqldump 备份，再配主库源
RESET BINARY LOGS AND GTIDS;       -- 8.4 新语法（替代 RESET MASTER）
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '192.168.56.11', SOURCE_PORT = 3306,
  SOURCE_USER = 'repl', SOURCE_PASSWORD = '...',
  SOURCE_AUTO_POSITION = 1, GET_SOURCE_PUBLIC_KEY = 1;
START REPLICA;
SHOW REPLICA STATUS\G              -- 看 IO/SQL 双 Yes、Seconds_Behind_Source、Last_*_Error、Executed_Gtid_Set
```

⚠️ 1+2 线程：主库 Binlog Dump 线程 → 从库 **IO 线程**（拉 binlog 写进本地 **Relay Log**）→ 从库 **SQL 线程**（读 Relay Log 重放）。
⚠️ `Replica_IO_Running: No`：先看 `Last_IO_Error`，查连接主库段（网络/3306 防火墙、复制账号密码、主库状态、`server_id` 冲突）。
⚠️ `Replica_SQL_Running: No`：本地回放报错（如主从数据不一致，主库删了从库不存在的库/表/用户）。
⚠️ `GET_SOURCE_PUBLIC_KEY = 1`：MySQL 8.4 默认 `caching_sha2_password`，非 TLS 时必加，否则报需要安全连接。
⚠️ `mysql ... -e "SQL" | grep ...`：`| grep` 是 bash 管道，必须写在 `-e "..."` 引号外面；不加 `-e` 末尾字符串会被当库名（1049）。

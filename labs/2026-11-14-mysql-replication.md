# MySQL 第三课：GTID 主从复制搭建与只读验证（B2-3）

- 日期：2026-11-14
- 考点：运维面试 —— 主从复制原理（binlog / Relay Log / IO 与 SQL 线程）、GTID、从库只读
- 环境：Rocky Linux 10.2（VirtualBox），node1（192.168.56.11，主库），node2（192.168.56.12，从库），MySQL 8.4.11，labdb 库
- 状态：完成

## 目标

在 node1（主）和 node2（从）之间搭通基于 GTID 的 MySQL 8.4 主从复制，验证主库写入实时同步、从库 `read_only` 拦截普通账号写入，以及重启后复制自动恢复。

## 关键概念

- **主从 1+2 线程与中继日志**：
  - 主库写变更进 **binlog**，由 **Binlog Dump 线程** 发给从库；
  - 从库 **IO 线程**（`Replica_IO_Running`）连主库拉取 binlog，写入从库本地的 **Relay Log（中继日志）**；
  - 从库 **SQL 线程**（`Replica_SQL_Running`）读取 Relay Log 并重放执行。
- **GTID vs File+Pos**：GTID 为每个事务分配全局唯一编号（`Source_UUID:序号`），从库开启 `SOURCE_AUTO_POSITION = 1` 自动对位，主从切换或重接时不用人工算 binlog 文件名和字节偏移量。
- **异步复制（默认）**：主库写完本地 binlog 立刻返回客户端成功，不等待从库；性能高，主库突然断电可能丢未同步的最近几条数据。主从复制**不能代替备份**，误删也会同步过去。
- **MySQL 8.4 术语变化**：`CHANGE REPLICATION SOURCE TO`、`START REPLICA`、`SHOW REPLICA STATUS\G`、`RESET BINARY LOGS AND GTIDS`（替代旧版 `MASTER`/`SLAVE` 语法）。
- **`read_only = ON`**：拦普通账号写入（`ERROR 1290`），不拦复制线程，也不拦 `root` 等高权限账号（连 `root` 也拦需 `super_read_only = ON`）。

## 关键步骤

```bash
# 1. 基线检查：node1 已有 mysqld 与 labdb.t_order（5 行）；node2 只有客户端，装服务端
# node2：
sudo dnf install -y mysql8.4-server
sudo systemctl enable --now mysqld
sudo ss -lntp | grep mysqld
mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '...';"

# 2. 改配置文件并重启（node1 与 node2 均先 cp 备份 /etc/my.cnf.d/mysql-server.cnf）
# node1 [mysqld] 追加：server_id = 1 / gtid_mode = ON / enforce_gtid_consistency = ON
# node2 [mysqld] 追加：server_id = 2 / gtid_mode = ON / enforce_gtid_consistency = ON / read_only = ON
sudo systemctl restart mysqld
mysql -u root -p -e "SHOW VARIABLES WHERE Variable_name IN ('server_id','log_bin','gtid_mode','enforce_gtid_consistency','read_only');"
```

```sql
-- 3. node1 建最小权限复制账号（REPLICATION SLAVE 必须 ON *.*）
CREATE USER 'repl'@'192.168.56.12' IDENTIFIED BY '...';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.56.12';
SHOW GRANTS FOR 'repl'@'192.168.56.12';
```

```bash
# 4. node1 导出 labdb（含 GTID_PURGED）并 scp 传到 node2
mysqldump -u root -p --single-transaction --source-data=2 --databases labdb > ~/backup/labdb-repl.sql
grep -n -E 'GTID_PURGED|CHANGE REPLICATION SOURCE' ~/backup/labdb-repl.sql
scp ~/backup/labdb-repl.sql fqh@192.168.56.12:~

# 5. node2 清空自身 GTID 历史并导入基准数据
mysql -u root -p -e "RESET BINARY LOGS AND GTIDS;"
mysql -u root -p < ~/labdb-repl.sql
```

```sql
-- 6. node2 配置主库源并启动复制（MySQL 8.4 非 TLS 需 GET_SOURCE_PUBLIC_KEY = 1）
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '192.168.56.11',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = '...',
  SOURCE_AUTO_POSITION = 1,
  GET_SOURCE_PUBLIC_KEY = 1;

START REPLICA;
SHOW REPLICA STATUS\G
```

```sql
-- 7. node1 写入新数据 + 建普通测试账号，验证同步与 read_only
INSERT INTO labdb.t_order (item) VALUES ('fig');
CREATE USER 'ro_check'@'localhost' IDENTIFIED BY '...';
GRANT SELECT, INSERT ON labdb.* TO 'ro_check'@'localhost';
```

```bash
# 8. node2 验证同步、只读拦截与重启持久性
mysql -u root -p -e "SELECT * FROM labdb.t_order;"
mysql -u ro_check -p -e "INSERT INTO labdb.t_order (item) VALUES ('grape');"   # ERROR 1290
sudo systemctl restart mysqld
mysql -u root -p -e "SHOW REPLICA STATUS\G" | grep -E 'Running:|Seconds_Behind_Source|Executed_Gtid_Set|Last_.*_Error:'
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 主从变量生效 | `SHOW VARIABLES`（两端） | node1 `server_id=1, gtid_mode=ON`；node2 `server_id=2, gtid_mode=ON, read_only=ON` | ✅ |
| 复制线程正常 | `SHOW REPLICA STATUS\G` | `Replica_IO_Running: Yes`、`Replica_SQL_Running: Yes`、`Seconds_Behind_Source: 0`、`Auto_Position: 1` | ✅ |
| 主库写入同步 | node2 `SELECT * FROM labdb.t_order;` | 同步出第 6 行 `fig`，`Executed_Gtid_Set` 从 `1-3` 增至 `1-6` | ✅ |
| 从库只读拦截 | node2 `mysql -u ro_check -p -e "INSERT ..."` | `ERROR 1290 (... --read-only option)` | ✅ |
| 重启后仍生效 | node2 `sudo systemctl restart mysqld` 后查 `SHOW REPLICA STATUS\G` | 双 `Yes`，`Seconds_Behind_Source: 0`，`Last_IO_Error`/`Last_SQL_Error` 为空 | ✅ |

## 踩坑

- node2 上执行 `sudo vim` → `sudo: vim: command not found` → 最小化环境未装 vim 增强版 → 改用 `sudo vi`。
- node1 建复制账号时把 IP 写成逗号 `'repl'@'192,168,56,12'`，随后 `GRANT ... TO 'repl'@'192.168.56.12'` → `ERROR 1410: You are not allowed to create a user with GRANT` → MySQL 8 `GRANT` 不顺带建用户，找不到点分 IP 账号 → 用正确 IP 重新 `CREATE USER` 再 `GRANT`。（遗留的逗号账号勿在主库直接 `DROP USER`，否则同步到没有该账号的从库会导致 SQL 线程报错，需用 `DROP USER IF EXISTS`。）
- `SHOW GRENTS ...` → `ERROR 1064` → 拼写错误 → 改为 `SHOW GRANTS`。
- node2 执行 `mysql -u root -p "RESET BINARY LOGS AND GTIDS;"` → `ERROR 1049: Unknown database` → 漏写 `-e`，末尾字符串被当成数据库名 → 补上 `-e`。
- node2 执行 `mysql -u root -p -e "SHOW REPLICA STATUS\G | grep ..."` → `ERROR 1064` → 把 bash 管道 `| grep` 写进了 `-e "..."` 的 SQL 引号里 → 把 `| grep` 移到引号外。

## 卡点 / 没想明白的

- 主库默认 `server_id` 已是 `1` 为何还要写进配置文件 → 已解答：显式配置防止后续新增从库未改配置直接冲突，且不依赖版本默认值变化。
- `mysqldump` 报 `partial dump from a server that has GTIDs` 警告 → 已解答：只导出单个库时提醒默认带了全库 `GTID_PURGED`；初始化全新从库正需要它对齐起点，往无关库导入才加 `--set-gtid-purged=OFF`。
- `scp` 报 `known_hosts` 提示 → 已解答：以前登记过 `node2` 主机名但未单独登记 `192.168.56.12`，确认指纹后永久写入，属正常现象。
- `Executed_Gtid_Set` 和 `Auto_Position` 在 `SHOW REPLICA STATUS\G` 哪里 → 在输出倒数几行（`Channel_Name` 上方），同步新事务后还能看到 `Retrieved_Gtid_Set: ...:4-6`、`Executed_Gtid_Set: ...:1-6`。

## 面试题（5 道，对话中作答后批改）

1. （概念）在 MySQL 主从复制的接力流程里，从库的 IO 线程负责做什么工作？
<details><summary>批改要点</summary>

连接主库，通过网络拉取主库 binlog 里的变更事件，写进从库本地的 Relay Log（中继日志）。
</details>

2. （排障）在从库执行 `SHOW REPLICA STATUS\G`，发现 `Replica_IO_Running: No`（而 `Replica_SQL_Running: Yes`），应该优先排查哪一类问题？
<details><summary>批改要点</summary>

优先看 `Last_IO_Error`，排查「从库连接主库取日志」这一段：网络/防火墙（3306）、复制账号密码或权限、主库是否运行、主从 `server_id` 是否冲突（不是本地回放报错）。
</details>

3. （排障）为什么从库配置了 `read_only = ON` 后，普通业务账号写数据会报 `ERROR 1290`，但从库自己依然能正常同步主库写入的数据？
<details><summary>批改要点</summary>

`read_only = ON` 只拦截普通用户的直接写入，不拦截从库自身的复制线程（也不拦 `root` 等高权限账号，连 `root` 也拦需 `super_read_only = ON`）。
</details>

4. （操作）在 MySQL 8.4 中，查看从库当前复制状态、延迟和报错信息的 SQL 语句是什么（写出方便竖排阅读的结尾符）？
<details><summary>批改要点</summary>

`SHOW REPLICA STATUS\G`（不能把 bash 的 `| grep` 写进 SQL 语句里）。
</details>

5. （取舍）配置主从对位时，GTID 自动定位（`SOURCE_AUTO_POSITION = 1`）和传统「binlog 文件名 + 偏移量」相比，生产中为什么更推荐 GTID？
<details><summary>批改要点</summary>

GTID 给每个事务发全局唯一编号，从库自动对齐缺失事务；主从切换或重接时不用人工去算新主库的 binlog 文件名和字节位置，防止算错位点丢数据或报错。
</details>

作答结果：3.5/5。
- 第 1 题半对（0.5/1）：答出通过网络拉取主库 binlog 事务，但漏了写入从库本地 **Relay Log（中继日志）**；另外主库日志叫 **binlog**，`log_bin` 是变量名。
- 第 2 题半对（0.5/1）：答出查网络问题，漏了先看 `Last_IO_Error`，以及复制账号密码/权限、主库状态、`server_id` 冲突等同一段的可能原因。
- 第 3 题对（1/1）。
- 第 4 题半对（0.5/1）：把 bash 管道 `| grep -E 'running:...'` 混进了 SQL 语句里（会报 1064），SQL 本身只需 `SHOW REPLICA STATUS\G`。
- 第 5 题对（1/1）。
- 最薄弱：从库 **Relay Log（中继日志）** 在 IO/SQL 线程之间的位置、`Replica_IO_Running: No` 时要结合 `Last_IO_Error` 看“网络+账号+主库+server_id”整段，以及区分 SQL 语句和 bash 管道边界。

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

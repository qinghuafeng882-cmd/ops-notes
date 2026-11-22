# MySQL 第四课：慢查询日志、SHOW PROCESSLIST、EXPLAIN 与主从报错处理（B2-4）

- 日期：2026-11-15
- 考点：运维面试 —— 慢查询排查链路、索引与 `EXPLAIN` 分析、GTID 主从 SQL 线程报错恢复
- 环境：Rocky Linux 10.2（VirtualBox），node1（192.168.56.11，主库），node2（192.168.56.12，从库），MySQL 8.4.11，labdb 库
- 状态：完成

## 目标

掌握 MySQL 慢查询“抓现行（`SHOW FULL PROCESSLIST`）→ 翻历史（慢查询日志 + `mysqldumpslow`）→ 解剖原因（`EXPLAIN` + 索引）”的三步排查法，并在 GTID 模式下实操定位与恢复从库 `1062` 主键冲突报错。

## 关键概念

- **变量作用域（`GLOBAL` vs `SESSION`）**：`slow_query_log` 为纯全局动态变量；`long_query_time` 同时有 `GLOBAL` 和 `SESSION` 两层，执行 `SET GLOBAL long_query_time = ...` **只影响之后新建的连接**，已连上的 `mysql>` 或业务连接池长连接会沿用连入时继承的旧值（需 `SET SESSION` 或重建连接）。
- **慢查询日志核心字段**：`# Time` 后缀 `Z` 为 UTC 时间；`Query_time` 总耗时；`Lock_time` 等锁耗时；`Rows_sent` 与 `Rows_examined` 比例悬殊（如 `1 : 100000`）说明扫了大量无用行。
- **`SHOW FULL PROCESSLIST` vs `KILL`**：加 `FULL` 避免 `Info` 列 SQL 在 100 字符处被截断。`KILL <Id>` 直接断掉整条连接（客户端报 `ERROR 2013`）；`KILL QUERY <Id>` 只中断当前语句、保留连接。
- **`EXPLAIN` 关键列**：
  - `type`：`const`（主键/唯一索引等值）> `ref`（普通索引等值）> `range`（索引范围，如右模糊 `LIKE 'ORD-08888%'`）> `index`（全索引扫描）> `ALL`（全表扫描）。
  - `possible_keys` / `key`：可用索引 vs 实际选用索引。
  - `rows`：InnoDB 抽样估算扫描行数。
  - `Extra`：`Using index`（覆盖索引，索引树已包含全部要查列，无需回主表）、`Using index condition`（索引条件下推 ICP）、`Using filesort`/`Using temporary`。
- （11-18 复习补讲）InnoDB 聚簇索引的叶子是完整行；普通二级索引的叶子含主键。覆盖是指这一次查询要的列都在所用索引里。回表是按主键再读聚簇索引上的行，不等于全表扫描。
- **常见索引失效场景**：左模糊或前缀区分度过低（如 `LIKE 'ORD-%088888'` 优化器放弃索引）、在索引列上套函数（如 `LEFT(order_no, 9)`）。
- **GTID 主从 `1062` 恢复与取舍**：MySQL 8.4 默认多线程复制，`SHOW REPLICA STATUS\G` 常只报 Coordinator 停止，需查 `performance_schema.replication_applier_status_by_worker` 看具体 `LAST_ERROR_MESSAGE` 和卡住的 `APPLYING_TRANSACTION`。GTID 模式下 `SET GLOBAL sql_replica_skip_counter = 1;` 仅产生 `Warning 4016`（`Skip_Counter` 仍为 `0`）而不生效；必须用 `SET GTID_NEXT = '<UUID:序号>'; BEGIN; COMMIT; SET GTID_NEXT = 'AUTOMATIC';` 注入空事务跳过，但会留下主从数据不一致，若从库是脏数据应优先修正/删除从库冲突行再 `START REPLICA`。

## 关键步骤

```bash
# 1. 基线清理（node1 安全清理昨日遗留的逗号账号，node2 保持双 Yes、GTID 推进到 1-7）
mysql -u root -p -e "DROP USER IF EXISTS 'repl'@'192,168,56,12'; SELECT user, host FROM mysql.user WHERE user = 'repl'; SHOW VARIABLES WHERE Variable_name IN ('slow_query_log','slow_query_log_file','long_query_time','log_queries_not_using_indexes');"
command -v mysqldumpslow   # /usr/bin/mysqldumpslow
```

```sql
-- 2. node1 建 10 万行测试表 t_big_order 并测无索引基准耗时（0.18s / 0.19s）
CREATE TABLE t_big_order (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  order_no VARCHAR(32) NOT NULL,
  user_id INT UNSIGNED NOT NULL,
  status TINYINT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  note VARCHAR(120) NOT NULL,
  created_at DATETIME NOT NULL
) ENGINE=InnoDB;

INSERT INTO t_big_order (order_no, user_id, status, amount, note, created_at)
WITH RECURSIVE
  d(n) AS (SELECT 0 UNION ALL SELECT n + 1 FROM d WHERE n < 9),
  nums AS (
    SELECT a.n*10000 + b.n*1000 + c.n*100 + e.n*10 + f.n + 1 AS seq
    FROM d a CROSS JOIN d b CROSS JOIN d c CROSS JOIN d e CROSS JOIN d f
  )
SELECT
  CONCAT('ORD-', LPAD(seq, 6, '0')),
  (seq % 5000) + 1,
  seq % 4,
  ROUND(10 + (seq % 500) * 1.25, 2),
  REPEAT('order-payload-', 6),
  DATE_ADD('2026-01-01 00:00:00', INTERVAL seq SECOND)
FROM nums;

SELECT COUNT(*) FROM t_big_order;
SELECT * FROM t_big_order WHERE order_no = 'ORD-088888';   -- 0.18 sec
SELECT user_id, SUM(amount) AS total FROM t_big_order WHERE note LIKE '%payload%' GROUP BY user_id ORDER BY total DESC LIMIT 5;   -- 0.19 sec

-- 开慢查询日志并验证 GLOBAL 与 SESSION 差异
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.05;
SELECT @@session.long_query_time, @@global.long_query_time;   -- 10.000000 vs 0.050000
SET SESSION long_query_time = 0.05;
```

```bash
# 3. 查看慢日志与汇总、现场抓包查杀（node1）
sudo tail -n 25 /var/lib/mysql/node1-slow.log
mysql -u root -p labdb -e "SELECT * FROM t_big_order WHERE order_no = 'ORD-012345'; SELECT * FROM t_big_order WHERE order_no = 'ORD-066666';"
sudo mysqldumpslow -s t /var/lib/mysql/node1-slow.log

# 双终端演练 PROCESSLIST + KILL：终端 A 跑 SLEEP(100)，终端 B 查 Id 并 kill
mysql -u root -p labdb -e "SELECT SLEEP(100), COUNT(*) FROM t_big_order WHERE note LIKE '%payload%';"
mysql -u root -p -e "SHOW FULL PROCESSLIST;"
mysql -u root -p -e "kill 31"
```

```sql
-- 4. EXPLAIN 解剖与加索引对比、索引失效验证（node1）
EXPLAIN SELECT * FROM t_big_order WHERE order_no = 'ORD-088888'\G   -- type: ALL, rows: 97142
CREATE INDEX idx_order_no ON t_big_order(order_no);                 -- 2.06 sec
EXPLAIN SELECT * FROM t_big_order WHERE order_no = 'ORD-088888'\G   -- type: ref, key: idx_order_no, rows: 1
SELECT * FROM t_big_order WHERE order_no = 'ORD-088888';            -- 0.00 sec

EXPLAIN SELECT * FROM t_big_order WHERE order_no like 'ORD-08888%'\G          -- type: range, rows: 10
EXPLAIN SELECT * FROM t_big_order WHERE order_no LIKE 'ORD-%088888'\G         -- type: ALL (前缀无区分度放弃索引)
EXPLAIN SELECT id, order_no FROM t_big_order WHERE order_no = 'ORD-088888'\G  -- Extra: Using index (覆盖索引)
EXPLAIN SELECT * FROM t_big_order WHERE LEFT(order_no, 9) = 'ORD-088888'\G    -- type: ALL (列套函数失效)
```

```bash
# 5. 制造 1062 主键冲突并定位 GTID（node2 插 99 占坑 → node1 插 99 撞车）
# node2：
mysql -u root -p labdb -e "INSERT INTO t_order (id, item) VALUES (99, 'replica-ghost');"
# node1：
mysql -u root -p labdb -e "INSERT INTO t_order (id, item) VALUES (99, 'master-real');"
# node2 排查：
mysql -u root -p -e "SHOW REPLICA STATUS\G" | grep -E 'Running:|Last_SQL_Errno:|Last_SQL_Error:|Gtid_Set:'
mysql -u root -p -e "SELECT WORKER_ID, LAST_ERROR_NUMBER, LAST_ERROR_MESSAGE, APPLYING_TRANSACTION FROM performance_schema.replication_applier_status_by_worker WHERE LAST_ERROR_NUMBER != 0\G"
```

```sql
-- 6. node2 验证 sql_replica_skip_counter 无效（Warning 4016）并用 GTID_NEXT 空事务跳过 :11 恢复
SET GLOBAL sql_replica_skip_counter = 1;
SHOW WARNINGS;   -- Warning 4016
START REPLICA;   -- 仍卡在 1062，Skip_Counter 仍为 0

SET GLOBAL sql_replica_skip_counter = 0;
STOP REPLICA;
SET GTID_NEXT = '4b931ac7-dd5c-11f1-b1a6-080027527bea:11';
BEGIN;
COMMIT;
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;
SHOW REPLICA STATUS\G                        -- 双 Yes, Executed_Gtid_Set 推进至 :1-11
SELECT * FROM labdb.t_order WHERE id = 99;   -- 仍为 replica-ghost（存在主从不一致）

-- 7. 实验收尾清理双边 id=99 并恢复默认慢日志开关
-- node2：
DELETE FROM labdb.t_order WHERE id = 99;
```

```bash
# node1 临时关当前会话 binlog 删掉 id=99（避免同步到已无该行的 node2 触发 1032），并关闭临时慢日志
mysql -u root -p labdb -e "SET sql_log_bin = 0; DELETE FROM t_order WHERE id = 99; SET sql_log_bin = 1; SET GLOBAL slow_query_log = 'OFF'; SET GLOBAL long_query_time = 10; SELECT * FROM t_order;"
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 逗号账号清理同步无损 | node1 `DROP USER IF EXISTS` + node2 `SHOW REPLICA STATUS\G` | 只剩点分复制账号，node2 双 `Yes`、GTID 到 `:1-7` | ✅ |
| 慢查询日志抓取 | `SET GLOBAL slow_query_log='ON'`, `long_query_time=0.05` + `tail -n 25 /var/lib/mysql/node1-slow.log` | 记录 `Query_time: 0.215273`、`Rows_examined: 100000` 等慢 SQL | ✅ |
| `mysqldumpslow` 汇总 | `sudo mysqldumpslow -s t /var/lib/mysql/node1-slow.log` | 按模板合并统计（`Count: 3` 和 `Count: 1`） | ✅ |
| 现场抓包与查杀 | `SELECT SLEEP(100)...` + `SHOW FULL PROCESSLIST;` + `kill 31` | 查出 `Id 31` 完整 SQL，杀后客户端报 `ERROR 2013` | ✅ |
| 加索引前后效果 | `CREATE INDEX idx_order_no` 前后 `EXPLAIN` 与实测耗时 | `type: ALL (rows 97142)` → `type: ref (rows 1)`；`0.18~0.21s` → `0.00s` | ✅ |
| 索引失效与覆盖索引 | `EXPLAIN` 对比右模糊 / `ORD-%` / 覆盖列 / `LEFT()` | 右模糊 `range`，`ORD-%` 与 `LEFT()` 退化为 `ALL`，覆盖列显示 `Using index` | ✅ |
| GTID 冲突定位与恢复 | `1062` 冲突 → worker 表定位 `:11` → `GTID_NEXT` 空事务跳过 + 双端清理 `id=99` | `Replica_SQL_Running` 恢复 `Yes`（`:1-11`），最终双边 `t_order` 均为 `1..6` | ✅ |
| 慢日志重启持久性 | 仅用 `SET GLOBAL` 临时演示并已恢复 `OFF`/`10`，未写入 `/etc/my.cnf.d/mysql-server.cnf` | —（生产持久化需写配置文件或 `SET PERSIST`） | 未验证（未持久化） |

## 踩坑

- `DROP USER 'repl'@'192.168.56.12' IF EXISTS;` / `DROP USER IF_EXISTS ...` → `ERROR 1064` → `IF EXISTS` 位置必须紧跟 `DROP USER` 后且中间是空格不是下划线，同时险些误删点分真账号 `192.168.56.12` → 改为 `DROP USER IF EXISTS 'repl'@'192,168,56,12';`。
- 切错终端连到了从库 `node2`，未指定库名建表先报 `ERROR 1046: No database selected`，`use labdb;` 后用 `root` 直接在从库建出 `t_big_order` 并插入 10 万行 → `read_only = ON` 不拦 `root`（防管理员误写需 `super_read_only = ON`）→ 立刻在 `node2` `DROP TABLE labdb.t_big_order;` 删掉，避免主库建表同步过来冲突。
- `SET GLOBAL long_query_time = '0.05';` → `ERROR 1232: Incorrect argument type` → 数值型变量不能加引号 → 改为 `0.05`。
- `sudo tail -n 25 /var/lib/mysql/node1-slow.sql` → `No such file` → 日志扩展名误写为 `.sql` → 改为 `.log`。
- `sudo mysqldumpslow -s t -t 5 /var/lib/mysql/node1-slow.log` → 打印完 2 组后报 `Died at /bin/mysqldumpslow line 163, <> chunk 4.` → 源码 158–163 行切片 `@sorted[0 .. $opt{t}-1]` 在日志不足 5 类时补出 `undef` 触发 `die` → 去掉 `-t 5` 解决。
- 终端 A 跑 `SLEEP(30)` 时输密码和切命令太慢，`kill 22`/`kill 25` 报 `ERROR 1094: Unknown thread id` → 30 秒已结束连接回收 → 改用 `SLEEP(100)` 后 `kill 31` 命中。
- `EXPLAIN SELECT * t_big_order ...` → `ERROR 1064` → 漏写 `FROM` → 补上 `FROM`。
- `SELECT * FROM t_big_order WHERE order_id = ...` → `ERROR 1054: Unknown column 'order_id'` → 列名记错 → 改为 `order_no`。
- `performance_schme` / `performance_schma` → `ERROR 1049: Unknown database` → 拼写错误 → 改为 `performance_schema`。
- `SET GLOBAL sql_replica_skip_counter = 1;` 未报错但 `START REPLICA` 仍停在 `1062` → `SHOW WARNINGS;` 看到 `Warning 4016`（普通 GTID 通道无效）→ 归零后用 `GTID_NEXT` 注入空事务跳过。
- `SET GTID_NEXT = 'AUTOMITIC';` → `ERROR 1774: Malformed GTID specification` → `AUTOMATIC` 拼错 → 改对后继续 `START REPLICA;` 成功恢复。

## 卡点 / 没想明白的

- `command -v` 的含义 → 已解答：bash 内建命令，查最终执行路径或类型，极简系统未装 `which` 时更可靠。
- `DECIMAL(10, 2)` 与批量造数 SQL（`WITH RECURSIVE` / `CROSS JOIN` / `LPAD` / `ROUND`）→ 已解答：定点数精确存金额（共 10 位含 2 位小数）；CTE + 笛卡尔积只作批造 10 万行测试数据工具。
- `SELECT @@session.long_query_time` 写法 → 已解答：`@@` 读取系统变量表达式，可并排对比 `@@session` 与 `@@global`。
- 「业务连接池（Connection Pool）」含义 → 已解答：应用启动预建并复用长连接，解释了为何 `SET GLOBAL` 不影响现有老连接及 `KILL` 与 `KILL QUERY` 差异。
- `UUID` 含义 → 已解答：通用唯一识别码，`server_uuid` 存在 `/var/lib/mysql/auto.cnf`，与递增序号组成全局唯一 GTID。

## 面试题（5 道，对话中作答后批改）

1. （概念）在调整慢查询阈值时，为什么执行了 `SET GLOBAL long_query_time = 0.05;` 之后，已经连着的 `mysql>` 会话和业务连接池里的长连接依然按旧的 `10` 秒阈值记录？
<details><summary>批改要点</summary>

`long_query_time` 同时有 `GLOBAL` 和 `SESSION` 两层作用域。`SET GLOBAL` 只改变之后新建连接继承的模板值，已建立的老连接会继续保留连入时拿到的旧 `SESSION` 值（除非单独执行 `SET SESSION` 或重连）。
</details>

2. （排障）用 `EXPLAIN` 分析一条查询时，发现 `order_no` 列上明明建了索引，但 `type` 是 `ALL`、`key` 是 `NULL`，条件写的是 `WHERE LEFT(order_no, 9) = 'ORD-08888'`——为什么索引没有用上？
<details><summary>批改要点</summary>

对索引列套用了函数（`LEFT(...)`）。索引树里只存原始列 `order_no` 的值，不存函数计算后的结果，优化器无法利用索引目录查找，只能逐行取出算完再比对，退化成全表扫描（`ALL`）。
</details>

3. （排障）在 MySQL 8.4 从库上看到 `Replica_SQL_Running: No`，而 `SHOW REPLICA STATUS\G` 的 `Last_SQL_Error` 只提示 `Coordinator stopped because there were error(s) in the worker(s)` 时，应该去查哪张系统表才能直接看到具体 worker 报错和卡住的 GTID 编号？
<details><summary>批改要点</summary>

查询 `performance_schema.replication_applier_status_by_worker`（看 `LAST_ERROR_NUMBER`、`LAST_ERROR_MESSAGE` 和 `APPLYING_TRANSACTION` 字段；记不住全名时可用 `SHOW TABLES FROM performance_schema LIKE 'replication%';` 现场查找）。
</details>

4. （操作）发现数据库此刻突然变卡，请写出一条用于查看所有当前连接以及它们正在执行的**完整**（不被截断）SQL 语句的命令。
<details><summary>批改要点</summary>

`SHOW FULL PROCESSLIST;`（必须加 `FULL`，否则 `Info` 列超过 100 字符会被截断）。
</details>

5. （取舍）从库遇到 `1062` 主键冲突导致 SQL 线程停止时，「修正从库冲突数据后重启复制」和「用 `GTID_NEXT` 注入空事务跳过」这两种做法应该怎么取舍？
<details><summary>批改要点</summary>

若确认从库冲突行属于可舍弃的脏数据、要求主从数据保持一致，优先删除或修正从库冲突数据再 `START REPLICA`；若不能直接销毁从库该行或确需忽略主库该笔事务，再用 `GTID_NEXT` 注入空事务跳过，但代价是留下该行主从数据不一致。
</details>

作答结果：4.5/5。
- 第 1 题对（1/1）。
- 第 2 题对（1/1）。
- 第 3 题半对（0.5/1）：记住了 `performance_schema.replication_..._worker`，中间 `applier_status_by` 漏记（完整：`performance_schema.replication_applier_status_by_worker`，实战中可用 `SHOW TABLES FROM performance_schema LIKE 'replication%';` 现查）。
- 第 4 题对（1/1）。
- 第 5 题对（1/1）。
- 最薄弱：`performance_schema.replication_applier_status_by_worker` 系统表全名记忆（以及 `SHOW TABLES LIKE` 现查意识）。

## 复习记录

- [x] D+1（11-16，3.5/5；`Using index` 误答为「用上了索引」，实为覆盖索引、无需回表，是否用索引看 `key` 列；GTID 跳过事务命令没记全：`SET GTID_NEXT='UUID:N'; BEGIN; COMMIT; SET GTID_NEXT='AUTOMATIC';`，是会话变量，不是 `SET GLOBAL`）
- [x] D+3（11-18，1/1（合并复习，三篇合计 3.5/5，不是本篇单独 5 题）；只抽了 `Using index`：覆盖索引、不回表。GTID 跳过事务未测，不得称掌握）
- [x] D+7（11-22，0.5/1（合并复习，五题合计 4.5/5，不是本篇单独 5 题）；写成 `PROCESSLIST -f`。订正为 `SHOW FULL PROCESSLIST;`，含分号。不改首答）

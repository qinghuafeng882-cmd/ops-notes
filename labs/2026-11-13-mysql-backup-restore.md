# MySQL 第二课：mysqldump 备份恢复 + binlog 时间点恢复（B2-2）

- 日期：2026-11-13
- 考点：运维面试 —— 逻辑备份、备份一致性、误删后用全量备份 + binlog 恢复
- 环境：Rocky Linux 10.2（VirtualBox），node1 192.168.56.11，MySQL 8.4.11，labdb 库
- 状态：完成

## 目标

给 labdb 做一次全量备份，模拟备份之后误删表，用备份 + binlog 恢复到误删前一刻。

## 关键概念

- 备份防硬件故障和误操作；主从复制会把误操作一起复制过去，不能代替备份。
- 物理备份 = 拷数据文件（快，版本要一致，直接 cp 要停库）；逻辑备份 = mysqldump 导出 SQL（可读、可移植，大库恢复慢）。
- 一致性：mysqldump 一张表一张表导出。`--single-transaction` 开事务拿快照，整个导出读同一时刻，不阻塞写入；只对 InnoDB 有效，导出期间不能做 DDL（快照只管行数据，不管表结构）。`--lock-all-tables` 加全局读锁，导出期间阻塞写入。
- DDL 改表结构（CREATE/ALTER/DROP/TRUNCATE），DML 改行（INSERT/UPDATE/DELETE）。
- `--databases labdb` 带 `CREATE DATABASE` + `USE`；不带只有表。账号权限在 mysql 库，不在 labdb 的备份里。
- `--source-data=2`：把备份时刻的 binlog 文件名和位置写成注释（=1 是可执行语句，搭从库用）。
- 时间点恢复：先恢复全量备份，再从备份位置开始重放 binlog，停在误操作那组事件的起点。顺序不能反。
- binlog 由事件组成，`# at N` 是事件起始字节；ROW 格式下行变更要 `--base64-output=decode-rows -v` 才能读，这样输出的是 `###` 注释，不能拿去重放。

## 关键步骤

```bash
# 前提：表引擎、binlog 状态（在 mysql> 里）
# SELECT table_name, engine FROM information_schema.tables WHERE table_schema = 'labdb';   -- InnoDB
# SHOW VARIABLES LIKE 'log_bin%';      -- ON，/var/lib/mysql/binlog
# SHOW BINARY LOG STATUS;              -- binlog.000003 / 793（8.4 已无 SHOW MASTER STATUS）

# 建表 + 3 行（apple/banana/cherry）
# CREATE TABLE t_order (id INT PRIMARY KEY AUTO_INCREMENT, item VARCHAR(20), created_at DATETIME DEFAULT CURRENT_TIMESTAMP);

# 全量备份
mkdir -p ~/backup
mysqldump -u root -p --single-transaction --source-data=2 --databases labdb > ~/backup/labdb-full.sql
chmod 600 ~/backup/labdb-full.sql
grep -n -E 'CHANGE REPLICATION SOURCE|CREATE DATABASE|INSERT INTO' ~/backup/labdb-full.sql
# 22:-- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='binlog.000003', SOURCE_LOG_POS=793;

# 备份后插 durian/elder（位置 → 1119），再 DROP TABLE t_order（位置 → 1330）

# 用备份恢复（36 行 DROP TABLE IF EXISTS 在 39 行 CREATE TABLE 前）
grep -n 'DROP TABLE' ~/backup/labdb-full.sql
mysql -u root -p < ~/backup/labdb-full.sql

# 读 binlog，找误操作位置
sudo mysqlbinlog --base64-output=decode-rows -v --start-position=793 --stop-position=1330 \
  /var/lib/mysql/binlog.000003 | grep -E '^# at |^### |^BEGIN|^COMMIT|DROP TABLE'

# 重放 793 → 1119（先 sudo -v，避免 sudo 和 mysql 同时要密码）
sudo -v
sudo mysqlbinlog --start-position=793 --stop-position=1119 /var/lib/mysql/binlog.000003 | mysql -u root -p
mysql -u root -p -e 'SELECT * FROM labdb.t_order;'
```

binlog.000003 中 793 → 1330 的事件：

| `# at` | 事件 | 属于 |
|---|---|---|
| 793 | 事务开头标记 | 插入事务 |
| 872 | `BEGIN` | 插入事务 |
| 956 | 表映射（labdb.t_order） | 插入事务 |
| 1020 | 写入两行（`@1 @2 @3` = 第 1、2、3 列） | 插入事务 |
| 1088 | `COMMIT` | 插入事务 |
| 1119 | 事务开头标记 | DROP |
| 1196 | `DROP TABLE t_order` | DROP |

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 表是 InnoDB | `information_schema.tables` | InnoDB | ✅ |
| binlog 开启 | `SHOW VARIABLES LIKE 'log_bin%'` | ON | ✅ |
| 备份记录了 binlog 位置 | `grep CHANGE REPLICATION SOURCE` | binlog.000003 / 793，与备份前查到的一致 | ✅ |
| 备份恢复 | `SELECT * FROM labdb.t_order` | 只有 id 1-3，durian/elder 不在 | ✅ |
| binlog 重放后 | `SELECT * FROM labdb.t_order` | id 1-5 全部回来 | ✅ |
| 重启后仍生效 | | | 未验证 |

## 踩坑

- `DEFAULT CURRUNT_TIMESTAMP` → `ERROR 1064 ... near 'CURRUNT_TIMESTAMP'` → 拼错；`near` 后面就是 MySQL 开始看不懂的地方 → 改成 `CURRENT_TIMESTAMP`。
- `FROM information_schema WHERE ...` → `ERROR 1146 Table 'labdb.information_schema' doesn't exist`（没选库时是 `1046 No database selected`）→ information_schema 是库，只写一个名字被当成当前库的表 → 写成 `information_schema.tables`。退出重连无效，要改语句。
- `SHOW BINARY LOG STATES` → 1064 → 拼错 → `STATUS`。
- `mysqldump ... > ~/backup/labdb-full.sql` → `-bash: ... No such file or directory`，没有出现输密码的提示 → `>` 由 bash 在启动 mysqldump 前处理，目录不存在，mysqldump 没运行 → 先 `mkdir -p ~/backup`。反过来，mysqldump 失败时 `>` 照样会创建或清空文件，备份后要看文件大小和内容。

## 卡点 / 没想明白的

- `--start-position=793` 输出里为什么有 `# at 158` → 已解答：是文件开头的描述性事件，mysqlbinlog 总会输出，后面的事件靠它解析，不改数据（具体是哪种事件未核实，推测是 Previous-GTIDs）。

## 面试题（5 道，对话中作答后批改）

1. （概念）mysqldump 加了 `--single-transaction`，为什么导出的各张表就是同一时刻的数据？
<details><summary>批改要点</summary>

导出开始时开一个事务、拿到快照（InnoDB），整个导出都读这份快照，导出期间别人的写入看不见；不加锁，不阻塞写入。
</details>

2. （排障）执行 `mysqldump ... > ~/bak/x.sql` 后立刻报 `No such file or directory`，而且没有提示输密码，原因是什么？
<details><summary>批改要点</summary>

`>` 由 bash 在启动 mysqldump 之前处理，`~/bak` 目录不存在，bash 直接报错，mysqldump 根本没运行。
</details>

3. （排障）用 mysqlbinlog 重放到 mysql，命令没有报错，但误删前插入的行没有回来。重放命令最可能多加了什么？
<details><summary>批改要点</summary>

多加了 `--base64-output=decode-rows -v`。这样 ROW 格式的行变更只以 `###` 注释输出，mysql 什么都没执行。重放时不能加。
</details>

4. （操作）写出命令：把 `/var/lib/mysql/binlog.000003` 从 793 重放到 1119（不含 1119 开始的事件）。
<details><summary>批改要点</summary>

`sudo mysqlbinlog --start-position=793 --stop-position=1119 /var/lib/mysql/binlog.000003 | mysql -u root -p`
</details>

5. （取舍）每天一次 mysqldump 全量备份，和“每天全量备份 + 保留 binlog”相比，差别在哪？
<details><summary>批改要点</summary>

只有全量：最多丢掉上次备份到出事之间的数据（最长一天）。加上 binlog：可以恢复到误操作前一刻（时间点恢复），代价是 binlog 占空间、要保护好。标准：业务能接受丢多少数据。
</details>

作答结果：3/5。
- 第 1 题对。
- 第 2 题半对：答了“目录没创建”，没答出为什么不提示输密码（`>` 由 bash 先处理，mysqldump 根本没运行）；文件本身不存在不是原因，`>` 会创建文件。
- 第 3 题错：答成重放到了误删事件的开头。正确答案是多加了 `decode-rows -v`，输出变成 `###` 注释。
- 第 4 题半对：漏了 `| mysql -u root -p`，没有管道，SQL 只会打印到屏幕。
- 第 5 题对（补：代价是 binlog 占空间、要保护好；选哪种看业务能接受丢多少数据）。
- 最薄弱：mysqlbinlog“看”和“重放”的区别（看：`decode-rows -v`；重放：不加它、管道给 mysql）。

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

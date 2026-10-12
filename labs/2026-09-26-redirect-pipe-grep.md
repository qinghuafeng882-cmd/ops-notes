# 重定向、管道、grep 与基础文本处理

- 日期：2026-09-26～27（W1-D4，含原 D5）
- 考点：RHCSA EX200 —— 使用输入输出重定向；用 grep 和正则表达式分析文本
- 环境：Rocky Linux 10（VirtualBox），模拟日志 `~/d4lab/access.log`（2000 行）
- 状态：完成
- 来源：根据课程对话整理

## 目标

分清 stdout/stderr，会用管道组合 grep、cut、sort、uniq 做日志统计。

## 关键概念

- fd：0 stdin、1 stdout、2 stderr。重定向从左到右处理，`2>&1` 复制的是"此刻的指向" → **写在最后**。
- 管道只传 stdout，stderr 不进管道。
- `sudo echo x > /etc/f` 失败：重定向由当前 Shell（普通用户）处理，sudo 管不到 → `echo x | sudo tee -a /etc/f`。
- Here Document 写配置：内容含 `$` 时结束标记加引号 `<< 'EOF'`。
- 正则没有"与"，只有"或"；"与"用管道串两个 grep，或已知顺序时用 `.*`。
- `uniq` 只去**相邻**重复 → 先 sort。
- `cut` 只能单字符分隔，多空格对齐的输出要用 awk。

## 关键步骤

```bash
sudo grep -Ev '^(#|$)' /etc/ssh/sshd_config                      # 只看有效配置
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -3    # Top N 模板
grep 'GET /admin ' access.log | grep -c '" 404 '                  # 两个条件"与"
grep -c 'GET /admin .*" 404 ' access.log                          # 同上，另一种写法
grep -E '" 50[0-9]' access.log | cut -d' ' -f7 | sort | uniq -c | sort -rn | head -3
grep -E '" 50[0-9]' access.log | tee errors.log | wc -l
find . -name "*.txt" -print0 | xargs -0 ls -l                     # 文件名带空格
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| Top 3 IP | 见上 | 次数接近 2000/20=100 | ✅ 116 / 115 / 113 |
| /admin 且 404 | 两种写法交叉验证 | 估算 2000×1/6×2/10≈67 | ✅ 两种都是 69 |
| 5xx 条数 | `tee` + `wc -l` | 估算约 400 | ✅ 371 |

## 踩坑

- `grep -E '(GET /admin) & (" 404)'` 结果为空 → 正则没有 `&` 运算，被当成普通字符 → 管道串两个 grep。
- `grep -rl PermitRootLogin /etc/ssh/ 2>/dev/null` 结果为空 → `sshd_config` 权限 600 普通用户读不了，报错被 `2>/dev/null` 吞掉；且搜错了范围 → 结果不对劲时先去掉 `2>/dev/null` 看报错，用 `sudo`。
- 空结果直接提交没有怀疑 → **空结果不等于 0**，先估算是否合理。
- 说"普通用户打不开 /etc 下的文件" → 不准确，是**没有写权限**，大部分文件可读。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）`cmd > f 2>&1` 和 `cmd 2>&1 > f` 的区别？
<details><summary>答案</summary>

前者 1、2 都进 f；后者 2 先指向屏幕，之后 1 才改指 f，报错仍打在屏幕上。
</details>

2. （操作）统计日志里访问次数最多的 5 个 IP。
<details><summary>答案</summary>

`cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -5`
</details>

3. （排障）一条 grep 加了 `2>/dev/null` 后什么都没输出，你怎么判断是真没有还是出错了？
<details><summary>答案</summary>

去掉 `2>/dev/null` 看是否有 Permission denied；检查搜索范围；用 sudo 再跑；用估算判断空结果是否合理。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

# 权限：rwx、chmod/chown、umask、特殊权限

- 日期：2026-10-04～05（W2-D2）
- 考点：RHCSA EX200 —— 列出、设置和更改 ugo/rwx 权限；创建 SGID 协作目录
- 环境：Rocky Linux 10（VirtualBox），`/lab`、`/common/admin`
- 状态：完成（达标检查 7/7）
- 来源：根据课程对话整理

## 目标

搞清楚 rwx 对文件和目录的不同含义，会配 SGID 团队目录。

## 关键概念

- 判断顺序：属主 → 属组 → 其他人，**只匹配一次，不叠加**。属主权限 `---` 时属主什么都做不了。
- 目录：r 列出文件名，w 增删改名，x 进入/通过它访问文件。
- **删除文件看所在目录的 w+x**，不看文件本身权限。
- 访问一个文件需要路径上**每一层目录都有 x**；排查用 `namei -l 路径`。
- 数字：r4 w2 x1；常用 755/644/700/600/440。
- 递归改权限用大写 `X`：只给目录（和本来可执行的文件）加 x。
- umask 是**屏蔽**不是减法：文件最大 666、目录最大 777。
- 特殊权限：SUID 4（以文件属主身份运行，如 passwd）、SGID 2（目录：新文件继承属组）、Sticky 1（目录：只有属主能删自己的文件，如 /tmp）。大写 S/T 表示对应位置没有 x。
- ACL：`setfacl -m u:harry:rx`，`ls -l` 权限末尾出现 `+`。

## 关键步骤

```bash
sudo -u natasha cat order.txt                # 以指定用户身份测试权限
namei -l /lab/d/f.txt                        # 路径每一层的权限
sudo mkdir -p /common/admin
sudo chgrp sysmgrs /common/admin
sudo chmod 2770 /common/admin                # SGID 协作目录
sudo chmod -R u=rwX,g=rX,o= /lab/x           # 大写 X
find /usr -perm -4000 -type f 2>/dev/null    # SUID 审计
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 实验 1～7 | 按讲义 | 与预测一致 | ✅（自述已完成） |
| 达标检查 | — | — | ✅ 7/7 |

## 踩坑

- 热身 `find -owner harry -exec copy ...` → 选项是 `-user`，命令是 `cp`；目标目录要先 `mkdir -p`，否则第一个文件会被复制成一个同名普通文件，后面的依次覆盖；harry 家目录 700 需要 `sudo`。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）文件 `-rw----r--`，属主 alice，属组 dev。dev 成员 bob 能读吗？其他用户呢？
<details><summary>答案</summary>

bob 不能（匹配到属组就只看属组 `---`，不会退而用其他人的 `r--`）；其他用户能读。
</details>

2. （操作）建 `/srv/ops`：属组 ops，组内完全控制，其他人无权限，新文件继承属组。
<details><summary>答案</summary>

`mkdir -p /srv/ops; chgrp ops /srv/ops; chmod 2770 /srv/ops`
</details>

3. （排障）Nginx 访问 index.html 返回 403，文件权限 644 没问题，下一步？
<details><summary>答案</summary>

`namei -l` 检查路径上每一层目录的 x；RHEL 上还要查 SELinux 上下文（W4）。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

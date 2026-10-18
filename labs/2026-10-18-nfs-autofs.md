# NFS 与 autofs

- 日期：2026-10-18（W3-D4）
- 考点：RHCSA EX200 —— 用 NFS 挂载和卸载网络文件系统；配置 autofs
- 环境：Rocky Linux 10（VirtualBox）两台：node1（192.168.56.11，服务端）、node2（192.168.56.12，客户端）
- 状态：基本完成（node2 重启后 fstab 挂载未贴验证输出）
- 来源：根据课程对话和终端记录整理

## 目标

node1 导出目录，node2 用 fstab 和 autofs 两种方式挂载；验证 root_squash、exports 空格陷阱、autofs 超时卸载和通配符映射。

## 关键概念

- VFS 让 NFS 目录对程序和本地目录无差别。RHEL 10 默认 NFSv4.2，只需 TCP 2049（v3 还需 rpcbind、mountd）。
- `/etc/exports` 或 `/etc/exports.d/*.exports`：`目录  主机(选项)`，**主机和括号之间不能有空格**。
- 默认 `ro`、`sync`、`root_squash`。改完 `exportfs -rav`，用 `exportfs -v` 看实际生效的选项。
- sec=sys 下服务端按**数字 UID/GID** 检查权限 → 各机器同一用户 UID 要一致。
- root_squash：客户端 root → nobody（65534），防止一台客户端被攻破就能改所有共享数据。
- 默认 hard 挂载：服务端无响应时一直重试，进程卡在 D 状态、kill -9 无效 → 大量 D 状态先查 NFS/网络。
- fstab 挂 NFS 加 `_netdev`。
- autofs：访问时挂载、闲置超时卸载。主配置 `/etc/auto.master.d/*.autofs`（后缀必须 `.autofs`）：`父目录 映射文件 --timeout=60`；映射文件：`子目录 -选项 服务器:路径`。
- 通配符：`* -rw node1:/srv/nfs/home/&`，`&` 代表匹配到的名字。
- 未访问过的子目录 `ls` 看不到，`cd` 进去才出现。

## 关键步骤

```bash
# node1 服务端
sudo dnf install -y nfs-utils
echo "/srv/nfs/share 192.168.56.0/24(rw,sync,root_squash)" | sudo tee /etc/exports.d/share.exports
sudo systemctl enable --now nfs-server
sudo exportfs -rav; sudo exportfs -v
sudo firewall-cmd --permanent --add-service={nfs,mountd,rpc-bind}; sudo firewall-cmd --reload
showmount -e localhost

# node2 客户端
sudo dnf install -y nfs-utils
showmount -e node1
sudo mount -t nfs node1:/srv/nfs/share /mnt/nfs
echo "node1:/srv/nfs/share /mnt/nfs nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# node2 autofs
sudo dnf install -y autofs
echo "/mnt/auto /etc/auto.nfs --timeout=60" | sudo tee /etc/auto.master.d/nfs.autofs
echo "share -rw,sync node1:/srv/nfs/share" | sudo tee /etc/auto.nfs
echo "/mnt/home /etc/auto.home --timeout=60" | sudo tee /etc/auto.master.d/home.autofs
echo "* -rw,sync node1:/srv/nfs/home/&" | sudo tee /etc/auto.home
sudo systemctl enable --now autofs
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 导出生效 | `exportfs -v`、`showmount -e` | 网段 rw、root_squash | ✅ |
| 客户端挂载 | `df -hT /mnt/nfs`、`mount` | nfs4，vers=4.2，hard | ✅ |
| root_squash | node2 `ls -ln` / node1 `ls -l` | root 建的文件属主 65534/nobody | ✅ 普通用户文件为 1000 / fqh（两边 fqh 都是 UID 1000） |
| fstab 写法 | `findmnt --verify`、`mount -a` | 无错误 | ✅ `Success, no errors or warnings detected` |
| 重启后自动挂载 | `df -hT /mnt/nfs` | 已挂载 | 未验证（未贴输出） |
| exports 空格陷阱 | `exportfs -v` | 网段 ro + `<world>` rw | ✅ 两条规则，全世界可写 |
| autofs 按需挂载 | `ls /mnt/auto` → `ls /mnt/auto/share` | 先空，访问后出现 nfs4 挂载 | ✅ |
| autofs 超时卸载 | `sleep 70; mount \| grep /mnt/auto/share` | 无输出 | ✅ |
| 通配符映射 | `cat /mnt/home/alice/a.txt`、`bob/b.txt` | 读到各自文件 | ✅；charlie 不存在时报 No such file |

## 踩坑

- `firewall-cmd --add-service=rpc_bind` → 服务名是 `rpc-bind`（`--get-services` 可查）；`firewall-cmd reload` 少了 `--`。
- `findmnmt` 拼错。
- 实验 5 bob 的文件写到了 `/srv/nfs/home/b.txt`（漏了 `bob/`），node2 第一次读不到；`mv` 时 Permission denied（目录属 root）→ `sudo mv`。
- 实验 5 去 `/mnt/auto/alice` 读文件 → 通配符映射挂在 `/mnt/home`。
- `mount | grep auto` 里还有 `/misc`、`/net` 两条 → autofs 自带的默认映射（auto.master 里的 auto.misc 和 -hosts）。
- 实验 5 最后 `mount | grep /mnt/home` 只有 autofs 那一行，没有 alice、bob 的 nfs4 挂载 →（推测）中间去 node1 改文件耗时超过 60 秒，已超时卸载。
- 达标题 4 只答了原因，没答处理；达标题 5 没写 fstab 方式的优缺点。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）`/data 10.0.0.0/24 (rw)` 实际效果是什么？
<details><summary>答案</summary>

10.0.0.0/24 用默认选项（只读），`(rw)` 前没有主机 → 所有主机可读写。用 `exportfs -v` 能看到两条规则。
</details>

2. （操作）在客户端配置 autofs：访问 `/remote/data` 时挂载 `nfs1:/export/data`，闲置 120 秒卸载。
<details><summary>答案</summary>

`/etc/auto.master.d/remote.autofs`：`/remote /etc/auto.remote --timeout=120`；`/etc/auto.remote`：`data -rw nfs1:/export/data`；`systemctl enable --now autofs`（已运行则 restart）。
</details>

3. （排障）挂了 NFS 的服务器出现大量 D 状态进程、负载飙升，怎么处理？
<details><summary>答案</summary>

`ps -eo stat,pid,cmd | grep '^D'` 看是否卡在 NFS 路径；`mount | grep nfs`、`ping`/`showmount -e 服务端` 查网络和服务端；先恢复服务端或网络，进程会自己继续；服务端彻底不可用时 `umount -f` 或 `umount -l`（lazy）卸载。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7

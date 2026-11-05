# 实验：在 VirtualBox 虚拟机里跑 KVM/libvirt——嵌套虚拟化失败的排查与 TCG 方案

> 日期：2026-10-25 至 11-04　｜　环境：Windows 11 笔记本 → VirtualBox 7.2 → Rocky Linux 10 → CirrOS 0.6.2

## 一、实验目标

1. 动手理解 KVM、QEMU、libvirt 三者的关系。
2. 用 virt-install 创建虚拟机，用 virsh 管理虚拟机。
3. 从宿主机的进程和配置里，找到虚拟化原理的实际证据。

## 二、环境与层次

| 层次 | 是什么 | 硬件虚拟化来自 |
| --- | --- | --- |
| 第 0 层 | Windows 11 笔记本，AMD Ryzen 7 7840HS | 物理 CPU 的 AMD-V |
| 第 1 层 | VirtualBox 里的 Rocky Linux 10（node1，2 GB 内存，1 个 vCPU） | 需要 VirtualBox 把 AMD-V 传进来 |
| 第 2 层 | 在 node1 里用 libvirt 创建的 CirrOS 虚拟机 | 需要第 1 层也有 AMD-V，KVM 才能工作 |

在虚拟机里再跑虚拟机，叫**嵌套虚拟化**。

## 三、第一部分：嵌套虚拟化的排查

### 1. 现象：node1 里没有硬件虚拟化

```
$ grep -cE 'vmx|svm' /proc/cpuinfo
0
```

`vmx` 是 Intel VT-x 的标志，`svm` 是 AMD-V 的标志。宿主机任务管理器显示"虚拟化：已启用"，但 node1 里一个都没有，说明 VirtualBox 默认没把 AMD-V 传给虚拟机。

### 2. 打开嵌套虚拟化

图形界面里"启用嵌套 VT-x/AMD-V"复选框是灰色的，改用命令行。第一次执行报错：

```
PS> .\VBoxManage modifyvm "node1" nested-hw-virt on
VBoxManage.exe: error: Invalid parameter 'nested-hw-virt'
```

对照报错后打印的用法说明 `[--nested-hw-virt= on | off]`，发现参数前漏了 `--`。修正后成功：

```
PS> .\VBoxManage modifyvm "node1" --nested-hw-virt on --cpus=2
PS> .\VBoxManage showvminfo "node1" | Select-String -Pattern "Nested|CPUs"
Number of CPUs:              2
Nested VT-x/AMD-V:           enabled
Nested Paging:               enabled
```

### 3. 新现象：虚拟机启动卡住，状态栏出现绿色乌龟

开机后内核日志停在启动早期，无法进入系统。虚拟机窗口状态栏出现**绿色乌龟图标**。

排查过程（控制变量）：

| 尝试 | 结果 |
| --- | --- |
| 关闭嵌套虚拟化，保留 2 个 CPU | 仍然卡住 |
| 再把 CPU 改回 1 个 | 正常启动 |

结论：在当前模式下，node1 只能使用 1 个 vCPU；嵌套虚拟化也无法使用。

### 4. 根因：AMD-V 被 Windows 的 Hyper-V 独占

CPU 的硬件虚拟化（AMD-V 的 root 模式）**同一时刻只能被一个 Hypervisor 独占**。Windows 开启了依赖 Hyper-V 的功能后，Hyper-V 先于 Windows 启动并占住 AMD-V，VirtualBox 只能通过 Windows 提供的接口请 Hyper-V 代劳运行虚拟机。绿色乌龟图标就表示这种间接、较慢的运行方式。

| 状态栏图标 | VirtualBox 运行方式 | AMD-V 归谁 | 嵌套虚拟化 |
| --- | --- | --- | --- |
| 蓝色 V 字芯片 | 直接使用 AMD-V | VirtualBox | 支持 |
| 绿色乌龟（本次） | 经 Windows 虚拟机监控程序平台接口运行 | Hyper-V | 不支持 |

多 vCPU 时卡住的具体原因没有深究，可能与这种模式下的兼容性有关（推测）。

### 5. 方案取舍

| | 方案 A：关闭 Windows 的 Hyper-V 相关功能 | 方案 B：不用 KVM，让 QEMU 用 TCG 模拟 |
| --- | --- | --- |
| 能学到 | 完整的 KVM 实验 | libvirt、virsh、virt-install 全部操作；"虚拟机 = 一个 QEMU 进程" |
| 代价 | 要关闭"内存完整性"等安全功能，影响 WSL2 等，需改系统设置并重启 | 第 2 层虚拟机很慢 |
| 风险 | 中 | 几乎为零 |

**选择方案 B。**判断标准：用最小的风险拿到学习目标。面试考察的是对三者关系的理解和 virsh 的操作能力，方案 B 都能覆盖，不值得为此关闭宿主机的安全功能。

## 四、第二部分：安装 QEMU 和 libvirt

按 RHEL 10 官方文档：

```bash
sudo dnf install -y qemu-kvm libvirt virt-install
for drv in qemu network nodedev nwfilter secret storage interface; do sudo systemctl enable --now virt${drv}d{,-ro,-admin}.socket; done
sudo virt-host-validate qemu
```

- 新版 RHEL 系把 libvirtd 拆成了多个模块化守护进程（virtqemud 管虚拟机、virtnetworkd 管网络、virtstoraged 管存储等），并采用 **socket 激活**：有连接时才启动，空闲自动退出。

`virt-host-validate` 结果：

```
QEMU: Checking for hardware virtualization       : FAIL (Host not compatible with KVM; HW virtualization CPU features not found. Only emulated CPUs are available; performance will be significantly limited)
QEMU: Checking if device '/dev/vhost-net' exists : PASS
QEMU: Checking if device '/dev/net/tun' exists   : PASS
QEMU: Checking for cgroup 'cpu' controller support : PASS
...（其余 cgroup 项均为 PASS）
QEMU: Checking for device assignment IOMMU support : WARN (Unknown if this platform has IOMMU support)
QEMU: Checking for secure guest support          : WARN (None of SEV, SEV-ES, SEV-SNP, TDX available)
```

| 检查项 | 结果 | 含义 |
| --- | --- | --- |
| hardware virtualization | FAIL | 符合预期，正是方案 B 要绕过的 |
| /dev/vhost-net | PASS | 内核版的 virtio 网卡后端，减少内核与 QEMU 之间的数据搬运 |
| /dev/net/tun | PASS | 能创建 tap 虚拟网卡，虚拟机的网卡在宿主机上对应一个 tap 设备 |
| cgroup 各项 | PASS | libvirt 用 cgroup 限制每台虚拟机的 CPU、内存用量 |
| IOMMU | WARN | 设备直通需要；node1 本身是虚拟机，没有 IOMMU |
| secure guest | WARN | AMD SEV / Intel TDX 内存加密（机密计算），学习用不到 |

## 五、第三部分：创建并验证虚拟机

### 1. 准备镜像

```bash
sudo curl -L -o /var/lib/libvirt/images/cirros.qcow2 https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
sudo qemu-img info /var/lib/libvirt/images/cirros.qcow2
```

```
file format: qcow2
virtual size: 112 MiB (117440512 bytes)
disk size: 20.4 MiB
```

- 放在 `/var/lib/libvirt/images/`：该目录预设了 SELinux 标签 `virt_image_t`，QEMU 进程才能读取；放在家目录会被 SELinux 拦截。
- qcow2（QEMU Copy-On-Write v2）：虚拟大小 112 MiB，实际只占 20.4 MiB，用多少占多少。

### 2. 创建虚拟机

第一次执行报错：

```
ERROR    Error: --disk /var/lib/libvirt/images/cirros-qcow2: Size must be specified for non existent volume 'cirros-qcow2'
```

原因是文件名把 `.` 打成了 `-`。报错说"缺少大小"而不是"文件不存在"，是因为 virt-install 的逻辑是：`--disk` 路径存在就直接用；不存在就认为要**新建**磁盘，而新建必须指定大小。修正后：

```bash
sudo virt-install --name cirros1 --memory 256 --vcpus 1 --virt-type qemu --import \
  --disk /var/lib/libvirt/images/cirros.qcow2 --osinfo detect=on,require=off \
  --network network=default --graphics none --noautoconsole
```

```
WARNING  Using --osinfo generic, VM performance may suffer. Specify an accurate OS for optimal results.
Domain creation completed.
```

`--virt-type qemu` 是关键：不用 KVM，改用 TCG 模拟。

### 3. 从宿主机进程里找证据

`ps -ef | grep [q]emu` 的关键部分：

```
root   625561 ... /usr/sbin/virtqemud --timeout 120
qemu   625649 ... 53 ... /usr/libexec/qemu-kvm -name guest=cirros1,... -accel tcg -cpu max -m size=262144k ... -smp 1 ...
       -device {"driver":"ide-hd",...} -netdev {"type":"tap",...} -device {"driver":"e1000",...,"mac":"52:54:00:d3:20:10",...} ... -sandbox on,...
```

| 观察 | 含义 |
| --- | --- |
| `-accel tcg` | accelerator（加速器）= TCG（Tiny Code Generator），软件翻译执行，即**模拟**。有硬件支持时这里是 `kvm` |
| 进程用户是 `qemu` | QEMU 以普通用户运行，即使被攻破也拿不到 root |
| CPU 占用 53% | 模拟的代价：TCG 翻译执行非常吃 CPU |
| `ide-hd`、`e1000` | 磁盘和网卡都是**全模拟**设备，而非 virtio。因为系统被识别为 generic，virt-install 保守选择兼容性最好的设备，这就是那条性能警告的具体含义 |
| `-netdev type tap` | 网卡后端是宿主机上的 tap 设备 |
| `-sandbox on` | 用系统调用过滤进一步限制 QEMU |
| `virtqemud --timeout 120` | socket 激活的守护进程，空闲 120 秒退出 |

这一长串参数就是 libvirt 根据 XML 配置自动生成的，直观说明了 libvirt 存在的意义。

### 4. 登录验证

```
$ sudo virsh console cirros1
cirros login: cirros
$ uname -a
Linux cirros 5.15.0-71-generic #78-Ubuntu SMP ... x86_64 GNU/Linux
$ ip addr
2: eth0: ...
    link/ether 52:54:00:d3:20:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.34/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
$ ping -c 3 192.168.122.1
3 packets transmitted, 3 received, 0% packet loss
$ ls -l /sys/class/net/eth0/device/driver
... -> ../../../bus/pci/drivers/e1000
```

- 客户机里的网卡驱动是 `e1000`，MAC 与宿主机 `ps` 里一致，内外证据对上。
- `52:54:00` 是 QEMU/KVM 虚拟网卡的惯用 MAC 前缀。
- IP 由 libvirt 默认 NAT 网络（192.168.122.0/24）自带的 DHCP 分配。
- 第一个 ping 包 10 ms，之后不到 1.3 ms：第一个包需要先做 ARP 解析。

### 5. virsh 管理

```
$ sudo virsh dominfo cirros1
OS Type:        hvm
Security label: system_u:system_r:svirt_tcg_t:s0:c445,c716 (enforcing)
Messages:       tainted: use of deprecated configuration settings
                deprecated configuration: machine type 'pc-i440fx-rhel10.0.0'
$ sudo virsh dumpxml cirros1 | head -1
<domain type='qemu' id='1'>
```

- `hvm`：硬件虚拟机，客户机系统无需修改。
- **sVirt**：libvirt 给每台虚拟机的 QEMU 进程和磁盘分配一对唯一的 SELinux 类别（这里是 c445,c716），QEMU 只能访问类别相同的文件，一台虚拟机被攻破也读不到别人的磁盘。TCG 虚拟机用单独的类型 `svirt_tcg_t`，因为 TCG 需要运行时生成并执行代码，权限更敏感；KVM 虚拟机是 `svirt_t`。
- `i440fx` 是老式芯片组，RHEL 10 已不推荐，生产环境应使用 `q35`。

强制断电再开机：

```
$ sudo virsh destroy cirros1          # 强制断电
$ sudo virsh list --all
 -    cirros1   shut off              # 虚拟机仍在，只是关机
$ ps -ef | grep [q]emu-kvm            # 无输出：QEMU 进程已结束
$ sudo virsh start cirros1
$ sudo virsh list --all
 2    cirros1   running               # Id 从 1 变成 2
$ ps -ef | grep [q]emu-kvm
qemu   640991 ...                     # 新进程，PID 改变；UUID 不变
```

| 命令 | 作用 | 易误解处 |
| --- | --- | --- |
| `shutdown` | 正常关机 | 客户机不响应就关不掉 |
| `destroy` | 强制断电，杀掉 QEMU 进程 | **不删除**虚拟机和磁盘 |
| `start` | 开机，新起一个 QEMU 进程 | Id 每次重新分配，UUID 永久不变 |
| `undefine` | 删除虚拟机定义（XML） | 默认**不删**磁盘文件 |

## 六、结论

1. 一台 KVM/QEMU 虚拟机在宿主机上就是一个 QEMU 进程；关机即进程退出，开机即新建进程。
2. KVM 负责 CPU 和内存虚拟化，QEMU 负责模拟其余设备，libvirt 负责用 XML 统一管理；没有 KVM 时，QEMU 仍可用 TCG 模拟运行，但性能差很多。
3. 硬件虚拟化能力同一时刻只能被一个 Hypervisor 独占，这是本次嵌套虚拟化失败的根因。
4. virt-install 选择什么虚拟设备取决于它对客户机系统的识别；识别不出时会退回全模拟设备。

## 七、踩坑与收获

- **VBoxManage 报 `Invalid parameter`，原因只是少了 `--`。**报错后面其实打印了完整的用法说明，拿自己的命令逐字对照就能发现问题。以后遇到参数类报错，先对照用法，再考虑别的原因。
- **虚拟机卡住时，我用"一次只改一个变量"找到了原因。**先关掉嵌套虚拟化，还是卡；再把 CPU 改回 1 个，就好了。如果两个设置一起改，就分不清到底是哪个起作用。
- **绿色乌龟让我第一次在现实中看到课本上的原理。**学 VT-x 时知道 root 模式只有一个，但没想到它会体现成"Windows 开了 Hyper-V，VirtualBox 就拿不到 AMD-V"。理解了这一层，状态栏上一个小图标就能说明问题出在哪。
- **选方案时，我没有为了实验去关 Windows 的安全功能。**最终选 TCG 的理由是：用最小风险拿到想学的东西。不是每个问题都要"彻底解决"，先想清楚目标是什么。
- **virt-install 报"Size must be specified"，看起来和"文件名打错"毫无关系。**读懂"程序以为我想干什么"（以为我要新建磁盘），就能很快定位。
- **那条 `--osinfo generic` 的警告，原来是有具体后果的。**它让虚拟机用上了全模拟的 IDE 和 e1000，而不是 virtio。不要随手忽略 WARNING。
- **`destroy` 不是删除，`undefine` 不删磁盘。**整理笔记时我还意识到：如果要补救误执行的 `undefine`，用 `virsh dumpxml` 导出配置时，`>` 后面的文件名一定要确认，写错成磁盘文件的路径会直接把磁盘覆盖掉。重定向会无声地覆盖已有文件。
- **TCG 下的虚拟机会持续吃掉 node1 一半以上的 CPU。**实验做完要记得 `virsh destroy`，不然会拖慢平时的其他练习。

## 八、命令速查

```bash
# 安装与检查
dnf install -y qemu-kvm libvirt virt-install
virt-host-validate qemu
virsh net-list --all

# 创建
virt-install --name 名字 --memory 内存MB --vcpus 数量 --virt-type kvm|qemu --import \
  --disk 路径[,bus=virtio] --osinfo 系统类型 --network network=default[,model=virtio] \
  --graphics none --noautoconsole

# 管理
virsh list --all
virsh console 名字          # Ctrl+] 退出
virsh dominfo 名字
virsh dumpxml 名字 > ~/名字.xml
virsh define ~/名字.xml
virsh shutdown|destroy|start 名字
virsh undefine 名字          # 加 --remove-all-storage 才会连磁盘一起删

# 由 QEMU 进程号反查虚拟机
ps -o args -p 进程号 | grep -o 'guest=[^,]*'
```

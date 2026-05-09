# 记一次惊心动魄的 StarFive VisionFive 2 纯 NVMe SSD 迁移实战：从“完美克隆”到“绝处逢生”

## 〇、背景与痛点

最近在折腾手头这块 **StarFive VisionFive 2（星光 2）** RISC-V 单板计算机。官方默认的系统是烧录在 TF（MicroSD）卡里的。但众所周知，TF 卡不仅 I/O 速度慢得令人发指（读写通常在 20MB/s 上下），而且寿命堪忧，稍微跑点高频读写的数据库或 Docker 容器就容易报废。

为了彻底释放这块板子的性能，我给它加装了一块 128G 的 NVMe SSD，决心将系统**完美迁移到 SSD，并彻底拔掉 TF 卡，实现纯 NVMe 启动**。

**传统的迁移方案都有致命缺陷：**
1. **纯 `dd` 全盘克隆**：如果用 `dd if=/dev/mmcblk1 of=/dev/nvme0n1`，不仅慢得要命（连着空白空间一起复制），而且 64G 的卡克隆到 128G 的 SSD 上，还会导致分区表错乱，后续扩容极其麻烦。
2. **纯 `rsync` 拷贝**：如果在 SSD 上建好 ext4 分区直接 `rsync`，系统根本起不来。因为星光板的启动强依赖前部的几个隐藏分区（`p1` SPL、`p2` U-Boot、`p3` EFI 引导），纯拷文件会丢失引导记录。

经过一番推演，我设计了一套**【终极融合方案（Hybrid Clone）】**，并在此过程中遭遇了令人窒息的底层 Bug，好在最终通过极限微操成功化险为夷。本文记录了全过程的每一个命令和踩过的每一个大坑。

---

## 一、核心思路：外科手术级的“空间切割术”

星光板的 Debian 系统分区表结构如下：
*   **1区 (`p1`)**：SPL 启动引导区（起始扇区 4096，容量 2MB）
*   **2区 (`p2`)**：U-Boot 固件区（起始扇区 8192，容量 4MB）
*   **3区 (`p3`)**：EFI 引导分区，包含 `extlinux.conf`（起始扇区 16384，容量 100MB）
*   **4区 (`p4`)**：RootFS 根文件系统（起始扇区 221184，占据剩余所有空间）

**我的“融合方案”逻辑如下：**
1. **精确切皮（dd 前 200MB）**：既然前三个引导分区全都在前 150MB 以内，那我只用 `dd` 命令把 TF 卡的**前 200MB** 像素级复刻到 SSD 上。这样就完美移植了原厂的 GPT 分区表和引导文件。
2. **重塑骨架（sgdisk 扩容）**：修复被 `dd` 截断的 GPT 尾部标记，删掉残缺的 `p4` 分区，在 SSD 上重建一个占满 128G 全盘的巨无霸 `p4`。
3. **注入灵魂（rsync）**：将 SSD 的 `p4` 格式化为 ext4，把目前 TF 卡里正在运行的系统文件，原原本本地 `rsync` 进去。
4. **修改基因（UUID）**：将 SSD 引导文件里的磁盘指向，从 TF 卡改为 SSD 的 UUID。

---

## 二、实战演练：每一行命令的执行与解析

在 TF 卡系统正常运行的状态下，我们直接进行热迁移。

### Step 1: 精准克隆前 200MB 引导区
```bash
echo "【1/5】克隆完美引导区 (前 200MB)..."
sudo dd if=/dev/mmcblk1 of=/dev/nvme0n1 bs=1M count=200
```
*原理解析：`bs=1M count=200` 刚好切到第 409600 个扇区，把 GPT 头和 `p1, p2, p3` 完整包进去了，丝毫不碰尾部无用的文件系统数据。速度极快，几秒搞定。*

### Step 2: 修复 GPT 碎片并重建大分区
由于 SSD 容量大于 200MB，`dd` 过去的 GPT 备份头其实在 200MB 的位置，并不在 SSD 的物理末尾，这会导致 GPT 报错。我们需要修复它并重建 `p4`。
```bash
echo "【2/5】修复 GPT 尾部碎片并重建大分区 p4..."
# 将备份 GPT 移至物理磁盘真正的尾部
sudo sgdisk -e /dev/nvme0n1

# 删除因为 dd 而残缺的第 4 分区
sudo sgdisk -d 4 /dev/nvme0n1

# 重建第 4 分区：占满剩余所有空间，名字叫 "rootfs"，格式类型 8300 (Linux)
sudo sgdisk -n 4:0:0 -c 4:"rootfs" -t 4:8300 /dev/nvme0n1
```

### Step 3: 铺设新地砖（格式化）
```bash
echo "【3/5】格式化 p4 为 ext4..."
sudo mkfs.ext4 -F /dev/nvme0n1p4
```

### Step 4: 挂载并注入数据 (rsync 热拷贝)
```bash
echo "【4/5】挂载并完整拷贝文件系统 (rsync)..."
sudo mount /dev/nvme0n1p4 /mnt

# -a 归档模式，-x 限制在同一文件系统（不跳入 /proc, /sys 等虚拟目录），-v 显示进度
sudo rsync -axv / /mnt/
```
*避坑指南：必须加上 `-x` 参数！否则你会把内存里运行的虚拟设备和临时文件全拷进新硬盘，导致新系统崩溃。*

### Step 5: 修正引导配置（UUID 替换）
因为星光板默认的 `extlinux.conf` 里可能会写死 `root=/dev/mmcblk1p4` 或动态变量 `${sdev_blk}`，如果不改，用 SSD 启动后内核依然会去寻找 TF 卡。

```bash
echo "【5/5】修正引导配置文件 (写入 UUID)..."
# 获取新 SSD 第四分区的 UUID
UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p4)

# ⚠️ 踩坑点注意：星光板的 extlinux.conf 不在根目录下，而是在独立的 p3 (FAT) 分区里！
sudo mount /dev/nvme0n1p3 /mnt/boot

# 将原有的 root= 指向全部替换为新 SSD 的 UUID
sudo sed -i "s|root=/dev/\${sdev_blk}|root=UUID=$UUID|g" /mnt/boot/extlinux/extlinux.conf
sudo sed -i "s|root=/dev/mmcblk1p4|root=UUID=$UUID|g" /mnt/boot/extlinux/extlinux.conf

# 修改 fstab
sudo sed -i "s|/dev/mmcblk1p4|UUID=$UUID|g" /mnt/etc/fstab

# 卸载清理
sudo umount /mnt/boot
sudo umount /mnt
```

---

## 三、深渊惊魂：连环踩坑与绝处逢生

上面那套脚本看似天衣无缝，但在实际执行中，我却遭遇了一连串令人窒息的突发状况。如果你也遇到了，请千万不要慌。

### 💣 连环坑一：开机死锁与 SSH Port 22 离奇暴毙
就在我准备重启验收成果时，机器突然失联了。拔掉 TF 卡尝试用 SSD 启动，再把 TF 卡插回去尝试恢复……各种姿势折腾后，**我发现 SSH 的 22 端口彻底连不上了，直接报 `Connection refused`！**但 Ping 却是通的。

**排查原因：**
登录本地终端（或通过备用手段接入）执行 `systemctl status ssh`，发现如下报错：
```text
OpenSSL version mismatch. Built against 30000080, you have 30600020
sshd: Exception... start-limit-hit
```
**真相大白**：原来我之前手贱执行过一次极端的 `apt upgrade`，把 Debian 底层的 `openssl` 动态链接库强行升级到了最新的 `3.6.2-1`，但是 OpenSSH server（版本 `9.2p1`）是在旧版 OpenSSL 3.0 下编译的。由于版本硬校验不通过，SSH 进程直接崩溃，且重试 5 次后彻底停止运行。

### 🛡️ 破局之法：Dropbear 黄金备用通道（极度推荐！）
**经验之谈：在对生产环境服务器尤其是 SBC 做底层升级时，永远，永远，永远要留一个备用后门！**

万幸的是，在此次折腾前，我在板子上顺手装了一个轻量级的 SSH 服务器 `dropbear`，并且把它绑定在了 `2222` 端口。Dropbear 的依赖极少，它并没有因为 OpenSSL 的大版本跃迁而崩溃。

我立刻通过 `ssh -p 2222 user@ip` 成功登录进了正在濒死的星光板。

### 💣 连环坑二：如何同时修复 TF 卡与 SSD 里的 SSH？
通过 `2222` 端口进去后，我马上用 `apt install openssh-server` 将 TF 卡里的 SSH 升级到了 `10.3p1`，解决了版本冲突，22 端口恢复绿灯。

**但是等等！刚才克隆进 SSD 里的系统，它的 SSH 也是坏的！**如果现在拔卡用 SSD 启动，依然会因为 SSH 崩溃而导致板子变砖（失联）。

**解决办法：魔法指令 `chroot`（进入“里世界”升级）**
我写了一段脚本，把还没启动的 SSD 挂载上来，将网络和设备映射进去，然后“伪装”成 SSD 的系统运行 `apt` 修复：
```bash
# 挂载 SSD 根目录
sudo mount /dev/nvme0n1p4 /mnt

# 映射宿主机的虚拟系统目录（必须映射，否则 apt 无法解析 DNS 和设备）
sudo mount -o bind /dev /mnt/dev
sudo mount -o bind /proc /mnt/proc
sudo mount -o bind /sys /mnt/sys
sudo mount -o bind /run /mnt/run

# 切换根目录进入 SSD 内部，执行非交互式升级
sudo chroot /mnt /bin/bash -c 'apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y openssh-server openssh-client openssh-sftp-server'

# 修完收工，卸载目录
sudo umount /mnt/run
sudo umount /mnt/sys
sudo umount /mnt/proc
sudo umount /mnt/dev
sudo umount /mnt
```
这一手 `chroot` 微操，成功在 SSD 真正启动前，就把它的心脏病给治好了。

### 💣 连环坑三：拔卡后绿灯不亮（点不亮机器）
经过以上抢修，我满怀信心地拔掉了电源，**抽出了 TF 卡**，重新通电。
结果……主板的绿灯（Heartbeat 呼吸灯）根本没亮，网口灯也没反应。板子像块砖头一样安静。

**真相**：
星光板 VisionFive 2 的 CPU (JH7110) 在刚通电那一刻，硬件的 BootROM 是**不认识 PCIe/NVMe 协议的**。它只能从物理引脚指定的设备（SD、eMMC、SPI Flash）去找开机的最初引导代码。
默认情况下，主板上的物理 **DIP 拨码开关** 出厂设为 `H L` (1 和 0)，即“强制从 SD 卡启动”。如果 SD 卡槽是空的，CPU 就会原地发呆，根本不知道怎么去加载 SSD。

**最终绝杀：调整 DIP 拨码开关**
在 40 针 GPIO 旁边，有一个非常小的黑色底座，上面有两个白色拨片。
为了实现**真正的纯 NVMe 脱卡启动**，必须把这两个拨片统统拨向 `L`（Low）侧！
👉 **即设置为 `0 0` (双 L 模式，QSPI SPI Flash 引导)。**

在这个模式下，通电后：
1. CPU 会从主板自带的 SPI 固件中读取 U-Boot。
2. U-Boot 初始化完成后，它内部包含了 NVMe 驱动，它会扫描整个主板。
3. 发现 NVMe SSD 后，顺着我们写入的 `extlinux.conf`，顺利加载 Linux 内核。

---

## 四、大功告成：迎娶白富美

拨完开关，插电。
5秒钟后，星光板绿灯开始欢快地闪烁！网络连接恢复！
敲击键盘登录进 22 端口，满怀激动地输入 `df -h`：

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p4  117G  4.1G  107G   4% /
```
根目录 `/` 完美挂载在 `/dev/nvme0n1p4`，117G 纯净空间全量释放！TF 卡已被彻底物理移除，系统的响应速度迎来了肉眼可见的暴涨！

## 五、总结与建议

1. **不要迷信全盘 `dd`**：对于容量不一致的盘，采用 `dd` 前段引导区 + `sgdisk` 重建尾区 + `rsync` 拷贝文件的组合拳，是最干净、最不容易留暗坑的做法。
2. **永远准备一把“备用钥匙”**：折腾 SBC 这种高度依赖 SSH 的设备，装一个 `dropbear` 并开在非常规端口，关键时刻绝对能救命。
3. **敬畏硬件底层逻辑**：当软件逻辑没问题时，想想是不是硬件拨码/跳线拦截了你。

从下午的焦头烂额，到晚上的畅快淋漓，这场星光板 NVMe 换心手术，值了！

---
*(本文记录于 2026 年 5 月，实操环境为 Debian trixie/sid + Linux 6.12.5 内核)* 

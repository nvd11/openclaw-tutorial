# 星光板 (Starfive RISC-V) 安装 OpenClaw 实战全纪录

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的全过程。
为了保护脆弱的 SD 卡，所有操作均严格限制在挂载了 NVMe SSD 的 `/home` 目录下。

## 阶段一：尝试安装 Node.js 环境 (踩坑纪实)

由于 RISC-V 架构较为冷门，Node.js 官方主干并不直接提供预编译包。我们需要前往官方的 **Unofficial Builds** 仓库获取二进制文件，实现“绿色免安装”。

> **⚠️ 踩坑记录一 (GLIBC 兼容性问题)：**
> 很多较新的预编译包（如 v22 和 v20）是在较新的 Linux 系统上编译的，它们需要 `GLIBC_2.38` 甚至更高版本。而目前 Starfive 默认的 Debian trixie/sid 环境中，GLIBC 版本仅为 `2.36`。
> 强行运行会报：`version GLIBC_2.38 not found` 错误。

> **⚠️ 踩坑记录二 (OpenClaw 版本强依赖 & 本地编译雪崩)：**
> 尝试退而求其次使用 Node.js `v18.x` LTS 版本。环境确实配好了，但在执行 `npm install -g openclaw` 时遇到了双重打击：
> 1. OpenClaw 强依赖 Node 22 中的新特性，强制要求 `>=22.14.0`，强行使用 `--ignore-engines` 会导致大量底层依赖 (如 undici) 不兼容。
> 2. 由于 RISC-V 架构极少有预编译的 npm Native 扩展包 (如 tree-sitter-bash)，npm 会被迫在本地调用 gcc/g++ 进行源码编译。但 Starfive 旧源中的 `g++-12` 存在模板库 bug，导致 C++ 编译直接报出满屏的 `template argument deduction/substitution failed` 错误，安装彻底失败。

**结论：** 
**无法在当前的旧系统快照上通过降级 Node 或绕过依赖来完成安装。唯一的破局之路是更新 APT 快照源，对底层系统 (GLIBC、GCC) 进行大手术升级。**

## 阶段二：更新 APT 快照源与系统底层升级

为了获取 `GLIBC_2.40` 以及修复过 bug 的新版 `g++`，我们需要将 Starfive 的 Debian 快照源从 2023 年切换到 2024 年底。

### 1. 备份并替换 sources.list
**执行命令：**
```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb [arch=riscv64] https://snapshot.debian.org/archive/debian/20240901T000000Z unstable main" | sudo tee /etc/apt/sources.list
```

### 2. 执行更新命令 (忽略时间过期校验)
因为我们使用的是历史快照源，APT 会认为源已过期并拒绝更新，必须加入 `-o Acquire::Check-Valid-Until=false` 参数。

**执行命令：**
```bash
sudo apt-get -o Acquire::Check-Valid-Until=false update
```
*状态：源列表更新成功，已获取到包含高版本 libc6 的包索引。*

### 3. 升级底层核心库 (libc6)
> **🚨 致命卡点：**
> 尝试执行 `sudo DEBIAN_FRONTEND=noninteractive apt-get install -y libc6` 升级 C 库时，触发了灾难性的依赖链冲突（Dependency Chain Avalanche）：
> `libc6` 的升级破坏了 `systemd`, `locales`, `base-files`，并与 `libc-bin` 产生了死锁。APT 拒绝执行并报错退出。

*(未完待续... 正在探索最终破局方案)*
## 阶段三：最终破局方案 (2026-05-09 更新)

经历过底层依赖灾难后，我们意识到，必须通过官方源并搭配 NVMe SSD 大幅提升 I/O，才能完成安全、完整的系统升级。

### 1. 切换为清华大学 Debian RISC-V 镜像源
我们放弃了过时的 Snapshot 源，直接切换到了清华的 `trixie/sid` 源：
```bash
sudo sed -i 's/deb http:\/\/snapshot.debian.org.*/deb https:\/\/mirrors.tuna.tsinghua.edu.cn\/debian\/ sid main contrib non-free non-free-firmware/g' /etc/apt/sources.list
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt full-upgrade -y
```
此操作成功跨越了 `libc6` 的死锁，系统底层迎来了全面的重生。

### 2. Node.js V24 就绪
升级完成后，我们在 `/home/gateman` 用户目录下直接部署了 RISC-V 架构的 Unofficial Node.js V24，成功跨越了 OpenClaw 的版本红线。

### 3. 配置无 sudo 的纯净 npm 全局环境
为了不在系统级别留下垃圾并避免权限安全问题，我们配置了局部的 npm 环境：
```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo -e '\nexport NPM_CONFIG_PREFIX=~/.npm-global\nexport PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 4. 安装核心: npm install -g openclaw
**当前执行中**：通过配置好的隔离环境，直接发起了全局安装：
```bash
npm install -g openclaw
```
*(在 RISC-V 下遇到部分无预编译扩展的包时，会自动触发本地 GCC 的源码编译。基于升级后的系统，编译通过率得到保障。)*

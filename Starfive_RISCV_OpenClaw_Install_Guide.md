# 星光板 (Starfive RISC-V) 安装 OpenClaw 实战全纪录

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的全过程。
为了保护脆弱的 SD 卡，所有操作均严格限制在挂载了 NVMe SSD 的 `/home` 目录下。

## 阶段一：安装 Node.js 环境

由于 RISC-V 架构较为冷门，Node.js 官方主干并不直接提供预编译包。我们需要前往官方的 **Unofficial Builds** 仓库获取二进制文件，实现“绿色免安装”。

> **⚠️ 踩坑记录 (GLIBC 兼容性问题)：**
> 很多较新的预编译包（如 v22 和 v20）是在较新的 Linux 系统上编译的，它们需要 `GLIBC_2.38` 甚至更高版本。而目前 Starfive 默认的 Debian trixie/sid 环境中，GLIBC 版本仅为 `2.36`。
> 强行运行会报：`version GLIBC_2.38 not found` 错误。
> **解决方案**：退而求其次，选择 Node.js `v18.x` 系列（OpenClaw 完美支持的 LTS 版本），它对底层 C 库的要求较低。

### 1. 下载并解压预编译压缩包
在用户根目录下创建一个存放本地软件的目录 `~/.local`，下载 Node.js v18.20.8 的 `riscv64` 预编译包并解压：

**执行命令：**
```bash
mkdir -p ~/.local
cd ~/.local

# 下载 Node v18 RISC-V 压缩包
wget https://unofficial-builds.nodejs.org/download/release/v18.20.8/node-v18.20.8-linux-riscv64.tar.xz

# 解压并重命名文件夹
tar -xf node-v18.20.8-linux-riscv64.tar.xz
mv node-v18.20.8-linux-riscv64 node

# 删掉压缩包腾出空间
rm node-v18.20.8-linux-riscv64.tar.xz
```

### 2. 配置环境变量
为了让系统在任何地方都能识别 `node` 和 `npm` 命令，需要将刚才解压的 `bin` 目录加入到环境变量 `PATH` 中。

**执行命令：**
```bash
echo 'export PATH="$HOME/.local/node/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 3. 验证安装结果
测试一下环境变量是否生效。

**执行命令：**
```bash
node -v
npm -v
```

**关键输出：**
```text
v18.20.8
10.8.2
```
✅ 看到版本号，说明 Node.js 环境已经完美在 SSD 上扎根了！
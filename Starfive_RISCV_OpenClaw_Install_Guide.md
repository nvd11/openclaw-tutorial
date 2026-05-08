# 星光板 (Starfive RISC-V) 安装 OpenClaw 实战全纪录

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的全过程。
为了保护脆弱的 SD 卡，所有操作均严格限制在挂载了 NVMe SSD 的 `/home` 目录下。

## 阶段一：安装 Node.js 环境 (v22)

由于 RISC-V 架构较为冷门，Node.js 官方主干并不直接提供预编译包。我们需要前往官方的 **Unofficial Builds** 仓库获取二进制文件，实现“绿色免安装”。

### 1. 下载预编译压缩包
首先在用户根目录下创建一个存放本地软件的目录 `~/.local`，并下载 Node.js v22.22.2 的 `riscv64` 预编译包：

**执行命令：**
```bash
mkdir -p ~/.local
cd ~/.local
wget https://unofficial-builds.nodejs.org/download/release/v22.22.2/node-v22.22.2-linux-riscv64.tar.xz
```

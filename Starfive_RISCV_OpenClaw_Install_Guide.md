# Starfive (RISC-V) 安装 OpenClaw 完整指南 (NVMe 启动版)

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的标准流程。
本指南基于 **NVMe SSD 脱卡启动**，系统底层为 `Debian trixie/sid` (清华源)。

## 第 1 步：直接通过 APT 安装原生 Node.js 24
一个巨大的好消息！由于我们将系统源升级到了最新的 `trixie/sid`，Debian 官方已经为 RISC-V 架构提供了原生的 Node.js v24.15.0 支持。
**无需去下载第三方非官方预编译包了！直接包管理器秒杀！**

```bash
sudo apt update
sudo apt install nodejs npm -y
```
安装完成后验证：
```bash
node -v   # 应输出 v24.15.0 或以上
npm -v    # 应输出 v11.x 或以上
```

## 第 2 步：配置免 sudo 的全局 npm 环境
为了安全和权限隔离，防止 `npm install -g` 污染系统环境或产生权限报错，我们配置针对当前用户 (`gateman`) 的局部全局 npm 目录：

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo -e '\nexport NPM_CONFIG_PREFIX=~/.npm-global\nexport PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 第 3 步：全局安装 OpenClaw (暴力绕过 C++ 编译)

> **⚠️ RISC-V 特有大坑预警**：直接执行 `npm install` 会导致雪崩。
> 因为 RISC-V 缺乏部分依赖（如 `tree-sitter-bash`）的预编译包，NPM 会强行拉起系统的 GCC 15 现场编译 C++，从而触发致命的 `Error: non-constant .uleb128 is not supported` 汇编器报错。

为了彻底绕过这个问题，我们使出“明修栈道，暗度陈仓”的绝招：

```bash
npm install -g openclaw --ignore-scripts
```

**原理解析**：`--ignore-scripts` 会直接告诉 npm 强行下载所有包文件，**绝不触发任何 C++ 底层的原生编译**！由于 `tree-sitter-bash` 这类原生模块对 OpenClaw 核心功能（代理通信和工具调用）并非强制必须，这招直接秒杀了报错泥潭，让安装瞬间绿灯放行！

## 第 4 步：注入专属配置
进入 `~/.config/openclaw/agents.json` 替换真实的腾讯混元 `apiKey`。
完成后重启服务：
```bash
systemctl --user restart openclaw-gateway
```

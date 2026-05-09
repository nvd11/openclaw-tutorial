# Starfive (RISC-V) 安装 OpenClaw 完整指南 (NVMe 启动版)

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的标准流程。
本指南基于 **NVMe SSD 脱卡启动**，系统底层为 `Debian trixie/sid` (清华源)。

## 第 1 步：安装并配置 Node.js (v24+)
由于 OpenClaw 依赖 Node.js 新特性，需要使用 `v24` 及以上版本。
前往 Node.js 官方 Unofficial Builds 获取 RISC-V 架构的二进制包解压并配置环境变量即可。

## 第 2 步：配置免 sudo 的全局 npm 环境
为了安全和权限隔离，配置局部的全局 npm 目录：
```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo -e '\nexport NPM_CONFIG_PREFIX=~/.npm-global\nexport PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 第 3 步：全局安装 OpenClaw
执行安装：
```bash
npm install -g openclaw
```
*注：RISC-V 架构下部分 npm Native 扩展包会触发本地 GCC 源码编译，请耐心等待。*

## 第 4 步：初始化配置与工作区构建
*(待补充)*

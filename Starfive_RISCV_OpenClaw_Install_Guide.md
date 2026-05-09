# Starfive (RISC-V) 安装 OpenClaw 完整指南 (NVMe 启动版)

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的标准流程。
本指南基于 **NVMe SSD 脱卡启动**，系统底层为 `Debian trixie/sid` (清华源)。

## 第 1 步：直接通过 APT 安装原生 Node.js 24
```bash
sudo apt update
sudo apt install nodejs npm -y
```

## 第 2 步：配置免 sudo 的全局 npm 环境
```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo -e '\nexport NPM_CONFIG_PREFIX=~/.npm-global\nexport PATH=~/.npm-global/bin:$PATH' >> ~/.rc
source ~/.bashrc
```

## 第 3 步：绕过报错安装核心引擎
由于 RISC-V 版 Debian sid 的 GCC 15 汇编器存在 Bug，会导致 `tree-sitter` 编译报错（`.uleb128`），我们使用参数绕过原生扩展：
```bash
npm install -g openclaw --ignore-scripts
```

## 第 4 步 (高阶玩法)：使用 Clang 补齐原生扩展 (可选)
如果希望 OpenClaw 拥有完整的 Bash 语法树精确分析能力（用于高阶代码重构），我们可以用 LLVM/Clang 替换有 Bug 的 GCC 编译器进行单独补发编译：

```bash
# 1. 安装 Clang 编译器
sudo apt install clang -y

# 2. 进入底层出错的模块目录
cd ~/.npm-global/lib/node_modules/openclaw/node_modules/tree-sitter-bash

# 3. 强制使用 Clang 重新编译
CC=clang CXX=clang++ npx node-gyp rebuild
```
*(编译成功后会生成 `tree_sitter_bash_binding.node`，此时 OpenClaw 将满血复活成 100% 完美体。)*

## 第 5 步：注入专属配置并启动服务
配置 `~/.config/openclaw/agents.json` 替换真实的 API Key，然后：
```bash
systemctl --user restart openclaw-gateway
```

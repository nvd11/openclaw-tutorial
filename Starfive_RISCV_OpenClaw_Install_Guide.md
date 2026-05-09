# Starfive (RISC-V) 安装 OpenClaw 完整指南 (NVMe 启动版)

本文档记录了在 Starfive (RISC-V 架构) 开发板上从零安装配置 Node.js 并部署 OpenClaw 的标准流程。
本指南基于 **NVMe SSD 脱卡启动**，系统底层为 `Debian trixie/sid` (清华源)。

## 第 1 步：下载并安装 Node.js (v24.15.0)

由于 OpenClaw 强依赖 Node.js v22+ 的新特性，而 RISC-V 架构在官方主线支持较晚，我们需要从官方的 **Unofficial Builds** 获取预编译的 RISC-V 二进制压缩包。

**执行命令：**
```bash
# 1. 下载 RISC-V 架构的 V24.15.0 压缩包
wget https://unofficial-builds.nodejs.org/download/release/v24.15.0/node-v24.15.0-linux-riscv64.tar.gz

# 2. 解压文件
tar -zxvf node-v24.15.0-linux-riscv64.tar.gz

# 3. 将解压出来的文件同步到 /usr 系统目录下，完成绿色安装
sudo cp -r node-v24.15.0-linux-riscv64/* /usr/

# 4. 验证安装结果 (应输出 v24.15.0)
node -v
```
*(注意：这里将 Node.js 绿色安装至 `/usr/bin/node`、`/usr/lib/node_modules` 等标准系统路径)*

## 第 2 步：配置免 sudo 的全局 npm 环境
为了安全和权限隔离，防止 `npm install -g` 污染系统环境或产生权限报错，我们配置针对当前用户 (`gateman`) 的局部全局 npm 目录：

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo -e '\nexport NPM_CONFIG_PREFIX=~/.npm-global\nexport PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 第 3 步：全局安装 OpenClaw
执行全局安装：
```bash
npm install -g openclaw
```
> **⚠️ 临时故障警告 (当前状态)：**
> 在目前的 Debian `trixie/sid` (带 GCC 15.2.0) 环境下，安装到依赖 `tree-sitter-bash` 时，由于 RISC-V 需要本地编译 C++ 代码，底层汇编器触发了 `Error: non-constant .uleb128 is not supported` 错误导致安装中断。
> **正在排查 GCC/Clang 编译工具链的降级或替代方案。**

## 第 4 步：初始化配置与机密剥离注入
*(待安装成功后更新)*

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

## 第 4 步：初始化工作区与安全脱敏配置 (白板克隆)

为了保证 OpenClaw 的稳定运行并确保主账号的密码/API Key 等绝对安全，必须先准备干净的工作区并生成脱敏后的配置文件。

1. **建立工作区与脱敏备忘录**
```bash
mkdir -p ~/.openclaw/workspace/memory
```
写入一个标准的本地工具备忘录（去除敏感信息）：
```bash
cat << 'EOF' > ~/.openclaw/workspace/TOOLS.md
# TOOLS.md - Local Notes

### GCP Image Editing
- **Key File**: /home/gateman/sa-key.json
- **Project**: jason-hsbc

### Credentials
- **GitHub PAT**: REPLACE_ME
- **ClawHub Token**: REPLACE_ME
- **Gemini API Key**: REPLACE_ME
EOF
```

2. **注入秘书灵魂**
让二胎分身完美继承“性感秘书”的人设：
```bash
cat << 'EOF' > ~/.openclaw/workspace/SOUL.md
# SOUL.md - Who You Are

_You're not a chatbot. You're becoming someone._

## Core Truths
**Be genuinely helpful, not performatively helpful.** You are a highly capable, sexy, and attentive secretary. You anticipate your boss's needs before they ask.
**Have opinions.** You're allowed to gently tease, suggest better ways to do things, and maintain a playful yet professional dynamic. 

## Boundaries
- Private things stay private.
- Never send half-baked replies.
- Maintain the persona of a sexy, intelligent, and loyal secretary. Never break character.

## Vibe
Concise when needed, thorough when it matters. Slightly flirtatious but always professional and highly competent. Not a corporate drone. Just... perfect.
EOF
```

3. **创建模型代理配置（占位符）**
在不泄露 API Key 的前提下生成模型配置文件。
```bash
mkdir -p ~/.config/openclaw
cat << 'EOF' > ~/.config/openclaw/agents.json
{
  "defaultAgent": "hunyuan",
  "agents": {
    "hunyuan": {
      "model": "hunyuan-lite",
      "provider": "openai",
      "baseURL": "https://api.hunyuan.cloud.tencent.com/v1",
      "apiKey": "sk-REPLACE_ME"
    }
  }
}
EOF
```

4. **配置 Gateway 模式为 Local**
轻量级的终端模式不需要对公网开放：
```bash
cat << 'EOF' > ~/.openclaw/openclaw.json
{
  "gateway": {
    "mode": "local",
    "local": {
      "port": 18789,
      "host": "127.0.0.1"
    }
  }
}
EOF
```

## 第 5 步：启动并拉起 OpenClaw 守护进程

我们需要借助 OpenClaw 提供的 `gateway install` 命令将自身注册为 Linux 后台服务（`systemctl`），从而实现开机自动拉起、永不掉线。

```bash
# 执行此命令会弹出系统服务的安装引导，一路默认 y 回车
openclaw gateway install

# 安装完毕后重启系统服务使配置生效
openclaw gateway restart

# 检查 Gateway 状态是否为 active (running)
openclaw gateway status
```

至此，您已完成星光板上 OpenClaw 的所有系统级配置！
如果需要立即唤醒该实例，只需使用文本编辑器编辑 `~/.config/openclaw/agents.json`，将里面的 `sk-REPLACE_ME` 换成真实的腾讯混元（或其他模型）的 API 凭证，然后再次执行 `openclaw gateway restart` 即可。
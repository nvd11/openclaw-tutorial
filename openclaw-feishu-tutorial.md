# 从 0 到 1：手把手教你在 Linux 搭建 OpenClaw 并接入飞书 🚀

你好！如果你刚学会敲几行 Linux 命令，手头刚好有一台 Linux 主机和一个 Gemini 的 API Key，想要拥有一个属于自己的飞书 AI 机器人助手（基于强大的 OpenClaw 框架），那么这篇教程就是为你量身定做的！

全程只需要复制、粘贴，遇到不懂的不要慌，跟着指令一步一步来就行。

---

## 📋 课前准备：你需要准备好这些“积木”

在正式动手之前，请确保你已经准备好了以下几样基本东西：

1. **一台 Linux 主机**：可以是阿里云、腾讯云等平台购买的云服务器（VPS），也可以是自己家里的树莓派或闲置电脑。如果你是第一次玩，推荐安装 Ubuntu 操作系统。
2. **一个 SSH 客户端**：用来远程连接你的 Linux 主机。Windows 用户推荐使用 [MobaXterm](https://mobaxterm.mobatek.net/) 或系统自带的 PowerShell（命令提示符）；Mac 用户直接使用自带的“终端（Terminal）”即可。
3. **一个大模型（LLM）的 API Key**：机器人需要有“大脑”才能聊天。本教程以 Google 的 Gemini 为例，你需要提前在 [Google AI Studio](https://aistudio.google.com/app/apikey) 申请一个 `Gemini API Key`（免费申请，但需要能访问外网）。如果你想用 OpenAI 等其他大模型，思路也是完全一样的。

**💡 准备完毕，如何登录你的 Linux 主机？**
打开你的 SSH 客户端或终端，输入以下命令（把里面的 `root` 换成你服务器的用户名，`<你的公网IP>` 换成服务器真实的 IP 地址），按回车后输入密码即可登录成功：
```bash
ssh root@<你的公网IP>
```

---

## 🛠️ 第一步：准备基础环境

OpenClaw 是基于 Node.js 运行的，所以我们先要在 Linux 上安装 Node.js。我们使用非常方便的 `nvm`（Node版本管理器）来安装。

打开你的终端（Terminal），依次输入并回车运行以下命令：

**1. 安装 nvm：**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

**2. 让 nvm 立即生效**（或者你也可以直接关掉终端重新打开）：
```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

**3. 使用 nvm 安装最新 LTS 版本的 Node.js：**
```bash
nvm install --lts
nvm use --lts
```

**4. 检查是否安装成功**（如果看到数字版本号，比如 `v20.x.x`，说明成功啦）：
```bash
node -v
npm -v
```

---

## 📥 第二步：安装和初始化 OpenClaw

环境准备好了，现在开始请出我们的主角 —— OpenClaw！

**1. 全局安装 OpenClaw：**
```bash
npm install -g openclaw
```

**2. 初始化配置文件**（这里会生成 OpenClaw 的本地配置）：
```bash
openclaw config init
```

---

## 🧠 第三步：配置 Gemini 大模型

现在我们需要给机器人装上“大脑”，也就是配置你手里的 Gemini API Key。

**1. 设置默认模型为 Gemini：**
```bash
openclaw config set agents.defaults.model "google/gemini-3.1-pro-preview"
```

**2. 配置你的 API Key**（**注意：** 把 `YOUR_GEMINI_API_KEY` 替换成你真实的 Key）：
```bash
openclaw config set providers.google.apiKey "YOUR_GEMINI_API_KEY"
```

---

## 🌐 第四步：配置网关以允许远程 HTTP 访问

如果你需要通过配套 App 或者网页控制台远程连接并管理你的 OpenClaw，我们需要修改网关的配置，允许外部 IP 以 HTTP 形式访问。这步要在配置具体渠道之前先做好：

**1. 允许任意 IP 访问**（默认只允许本地 `127.0.0.1`）：
```bash
openclaw config set gateway.bind "0.0.0.0"
```

**2. 关闭强制 TLS/HTTPS**（以便直接使用 HTTP 访问）：
```bash
openclaw config set gateway.tls.enabled false
```

**3. 配置远程 URL**（将 `<你的主机公网IP>` 替换为你 Linux 主机的真实 IP）：
```bash
openclaw config set gateway.remote.url "http://<你的主机公网IP>:3000"
```

**⚠️ 极其重要：开放防火墙端口**
无论你是使用阿里云、腾讯云等云服务器，还是本地的 Ubuntu 主机，**请务必确保 3000 端口已经对外开放**。
- 如果是云服务器，请登录云厂商的控制台，在**“安全组（Security Group）”**或**“防火墙”**设置中，添加入站规则：允许 TCP 协议的 3000 端口。
- 如果是 Ubuntu 本地防火墙（ufw），可以执行：`sudo ufw allow 3000/tcp`。
如果不放行这个端口，你将无法从外部通过 IP 访问到你的机器人控制台！

---

## 🤖 第五步：去飞书创建你的机器人应用

为了让 OpenClaw 能够在飞书里和你聊天，我们要在飞书后台创建一个机器人的“身份”。

1. 在浏览器打开 [飞书开发者后台](https://open.feishu.cn/app) 并登录。
2. 点击 **“创建企业自建应用”**，给它起个名字（比如：我的AI助手），上传个头像，然后点击创建。
3. 在左侧菜单找到 **“添加应用能力”**，选择 **“机器人”**，点击添加。
4. 在左侧菜单找到 **“凭证与基础信息”**。这里有两个非常重要的字符串，稍后要用到：
   - `App ID`
   - `App Secret`
5. 在左侧菜单找到 **“安全设置”**，如果在“事件订阅”部分看到了 **Encrypt Key（消息加密密钥）**，也把它复制下来备用（如果没有开启加密则不需要）。
6. 在左侧菜单找到 **“事件订阅”**，切换到 **“WebSocket”** 模式（OpenClaw 默认使用长链接，不用配复杂的服务器 URL！）。
7. 在左侧菜单找到 **“权限管理”**。为了让机器人具备完整的对话和成员识别能力，请开通以下关键权限（可以直接在顶部搜索框搜索）：
   - **消息与群组**：
     - `获取与发送单聊、群组消息`
     - `接收群聊中@我或以我为主的事件`
     - `获取单聊、群组、发送消息事件`
     - `获取群组信息` / `获取用户在群组中发表的消息`
   - **通讯录**：
     - `获取通讯录基本信息`
     - `获取用户基本信息`
     - `通过手机号或邮箱获取用户 ID`（可选，方便机器人识别特定用户）
8. 在左侧菜单找到 **“版本管理与发布”**，点击 **“创建版本”**，填上版本号（如 `1.0.0`）。在下方的**“可用范围”**中，选择你想让哪些人能使用这个机器人（如果你只是自己玩，可以选择只对自己或特定部门可见）。然后保存并申请发布。（如果需要企业管理员审核，请通知管理员通过一下）。

---

## 🔗 第六步：将飞书配置到 OpenClaw

回到你的 Linux 终端，现在要把刚刚从飞书拿到的 `App ID` 和 `App Secret` 告诉 OpenClaw。

**1. 设置 App ID**（替换 `YOUR_APP_ID`）：
```bash
openclaw config set plugins.entries.feishu.config.appId "YOUR_APP_ID"
```

**2. 设置 App Secret**（替换 `YOUR_APP_SECRET`）：
```bash
openclaw config set plugins.entries.feishu.config.appSecret "YOUR_APP_SECRET"
```

*(可选)* 如果你在飞书后台设置了 **Encrypt Key**，需要加上这行：
```bash
openclaw config set plugins.entries.feishu.config.encryptKey "YOUR_ENCRYPT_KEY"
```

**3. 启用飞书插件：**
```bash
openclaw config set plugins.entries.feishu.enabled true
```

**4. 告诉 OpenClaw 默认使用飞书作为聊天渠道：**
```bash
openclaw config set channels.default "feishu"
```

---

## 🚀 第七步：启动！见证奇迹的时刻

一切就绪，是时候唤醒你的 AI 助手了。

**1. 启动 OpenClaw 的后台网关服务：**
```bash
openclaw gateway start
```

**2. 查看运行状态：**
```bash
openclaw gateway status
```
如果看到类似 `Status: active (running)` 或者绿色的运行状态，恭喜你，大功告成！🎉

### 开始聊天
现在，打开你的飞书，在顶部的搜索框里搜索你刚刚创建的机器人名字，点开对话框。
试着发一句：“你好呀！”
如果它回复了你，说明你已经成功从 0 搭建起了一个属于自己的超级 AI 助手！

---
*提示：如果想要查看日志看有没有报错，可以输入 `openclaw gateway log`；如果要停止机器人，可以输入 `openclaw gateway stop`。*

---

## 🎁 附录：如何远程连接你的机器人后台？

既然在第四步我们配置了允许外部 HTTP 访问，你就可以通过 OpenClaw 配套的 App 或者网页端控制台来可视化管理你的机器人了。

但在登录控制台时，系统会要求你输入一个 **Gateway Token（网关令牌）** 作为安全密码。
你只需要在 Linux 终端中运行以下命令，即可查看你的专属 Token：
```bash
openclaw gateway token
```
复制这串字符，粘贴到登录页面的 Token 框中，即可成功连接！

---

## 🛠️ 附录二：OpenClaw 常用命令备忘录

为了让你更好地管理机器人，这里整理了几个最常用的 OpenClaw 命令，建议收藏备用：

- `openclaw gateway start`：**启动网关**（在后台默默运行机器人服务）。
- `openclaw gateway stop`：**停止网关**（关闭机器人服务）。
- `openclaw gateway restart`：**重启网关**（修改配置后如果未生效，可以尝试重启）。
- `openclaw gateway status`：**查看状态**（看看机器人是不是在正常运行）。
- `openclaw gateway log`：**查看日志**（如果机器人出错了、不回消息了，输入这个看具体的报错信息）。
- `openclaw gateway token`：**获取远程令牌**（用于登录可视化控制台）。
- `openclaw health`：**健康检查**（一键检查主机的系统状态、安全风险和更新建议）。
- `openclaw config list`：**查看所有配置**（看看你目前给 OpenClaw 设置了哪些参数）。
# OpenClaw Gateway Migration Runbook: Nova to Alice (GCP)

## 1. 架构目标 (Architecture Goal)
将运行在星光板 (StarFive, `10.0.1.227`) 上的 Nova Agent 的网关 (Gateway) 职责，剥离并迁移至 GCP 云主机 Alice (`100.94.13.17`)。
- **GCP (Alice)**: 作为中心 Gateway，负责高 CPU 消耗的插件解析 (Feishu/Slack schemas)、权限校验 (Auth)，并实现 Slack 端 `@alice` 与 `@nova` 两个 App 账号的消息多路路由 (Multi-Account Routing)。
- **StarFive (Nova)**: 降级为纯 Node 模式，仅通过 Tailscale 接收来自 Alice 网关的精简指令，并在本地执行大模型推理 (SiliconFlow Qwen2.5-72B)。
- **Radxa (Moon)**: 保持现状，维持独立的本地网关与 Agent 运行。

## 2. 迁移背景与教训 (Background & Lessons Learned)
- **事件**: 2026-05-11 发现 Nova 在响应携带图片的 Slack/Feishu 消息时，出现长达 71s 的卡死并最终报 TCP Timeout (`network connection error`)。
- **根本原因**: 星光板的 RISC-V 处理器单核性能弱，在解析庞大且嵌套的插件 Schema (如 Feishu 套件) 时，Node.js 主线程 (Event Loop) 严重拥堵，CPU Core Ratio 飙升至 1.58。主线程被锁死长达数秒至数十秒，导致底层的异步网络请求回调无法处理，最终触发操作系统的 TCP 协议栈超时 (63s + DNS 超时 = 71s)。
- **教训**: 排查故障时必须依靠底层指标 (如 `eventLoopDelayMaxMs`、内核参数)，严禁主观臆断 (见 `MEMORY.md` 相关更新)。

## 3. 详细操作步骤记录 (Execution Log)

### Phase 0: 盘点现状与全量备份 (READONLY & Backup)
所有的改动前必须备份核心配置。

**Alice (GCP) 备份:**
```bash
# [READONLY] 检查当前 Alice 配置目录
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "ls -la ~/.openclaw"

# 备份 Alice 的 openclaw.json
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup_before_gateway_migration_$(date +%Y%m%d)"

# 备份 Alice 的 systemd 服务文件
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "cp ~/.config/systemd/user/openclaw-gateway.service ~/.config/systemd/user/openclaw-gateway.service.backup_before_gateway_migration_$(date +%Y%m%d)"
```

**Nova (StarFive) 备份:**
```bash
# 备份星光板的配置文件
ssh gateman@10.0.1.227 "cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup_before_gateway_migration_$(date +%Y%m%d)"
```

### Phase 1: 改造 Nova (StarFive) 为纯 Node 模式
剥离 Nova 的外部通讯能力，使其专心处理模型推理。

1. **[READONLY] 检查当前 Nova 的 Gateway 与 Channels 状态:**
```bash
ssh gateman@10.0.1.227 "cat ~/.openclaw/openclaw.json | grep -A 20 '\"gateway\"'"
ssh gateman@10.0.1.227 "cat ~/.openclaw/openclaw.json | grep -A 20 '\"channels\"'"
```

2. **修改配置文件 (`~/.openclaw/openclaw.json`):**
关闭 Slack 与 Feishu 通道，并将 Gateway 模式改为 Remote。
*(具体执行将使用 `jq` 工具或 Node.js 脚本精准修改)*
```javascript
// Node.js 脚本修改逻辑 (拟定):
config.channels.slack.enabled = false;
config.channels.feishu.enabled = false;
config.gateway.mode = "remote";
config.gateway.remote = {
    "url": "ws://100.94.13.17:18789", // 指向 Alice GCP
    "token": "..." // 待通过 Alice 生成
};
```

3. **重启并验证 Nova 服务:**
```bash
ssh gateman@10.0.1.227 "systemctl --user restart openclaw-gateway"
ssh gateman@10.0.1.227 "openclaw gateway status"
```

### Phase 2: 改造 Alice (GCP) 为多路由总网关
让 Alice 接管 `@nova` 的 Slack 与 Feishu 账号，并配置严谨的路由规则。

1. **生成 Node 连接 Token:**
```bash
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "openclaw security token-create --role node"
```
*(此 Token 将用于 Phase 1 中星光板的 `remote.token` 配置)*

2. **[READONLY] 检查当前 Alice 的通道配置:**
```bash
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "cat ~/.openclaw/openclaw.json | grep -A 30 '\"channels\"'"
```

3. **修改配置文件 (`~/.openclaw/openclaw.json`):**
配置 Slack 多账号 (Multi-Account) 以及对应的路由规则 (Routes)。
*(具体执行将使用 Node.js 脚本精准合并目前 Nova 上的 Token)*
```javascript
// 增加 accounts 节点
config.channels.slack.accounts = {
    "alice": {
        "botToken": "xoxb-alice-...",
        "appToken": "xapp-alice-..."
    },
    "nova": {
        "botToken": "xoxb-nova-...",
        "appToken": "xapp-nova-..."
    }
};

// 配置精准路由
config.bindings = [
    {
        "type": "route",
        "agentId": "alice", // 派发给本地 Agent
        "match": { "channel": "slack", "accountId": "alice" }
    },
    {
        "type": "route",
        "agentId": "nova", // 跨网派发给远程星光板 Node
        "match": { "channel": "slack", "accountId": "nova" }
    }
];
```

4. **重启并验证 Alice 服务:**
```bash
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "systemctl --user restart openclaw-gateway"
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "openclaw gateway status"
```

### Phase 3: 连通性测试 (Verification)
1. **检查 Node 在线状态 (在 Alice 上执行):**
```bash
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "openclaw nodes list"
# 应当看到星光板成功接入
```
2. 在 Slack 中分别发送消息给 `@alice` 和 `@nova`，检查两者是否能够独立回复且互不干扰。
3. **[READONLY] 检查底层流量与事件循环延迟:**
```bash
ssh -i ~/.ssh/gcp_temp gateman@100.94.13.17 "grep -A 5 'eventLoopDelay' /tmp/openclaw/openclaw-*.log"
ssh gateman@10.0.1.227 "grep -A 5 'eventLoopDelay' /tmp/openclaw/openclaw-*.log"
# 确认星光板的 eventLoopDelayMaxMs 已经降至正常毫秒级别。
```

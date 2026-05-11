# OpenClaw Gateway Migration Plan: Nova to Alice (GCP)
*Created at: 2026-05-11 18:05:07 UTC*

## 1. 架构目标 (Architecture Goal)
将运行在星光板 (StarFive, ) 上的 Nova Agent 的网关职责，剥离并迁移至 GCP 云主机 Alice ()。
- **GCP (Alice)**: 作为中心 Gateway，负责高 CPU 消耗的逻辑，并实现 Slack 端的应用多路路由 ( 分发到本地， 跨网分发给星光板)。
- **StarFive (Nova)**: 降级为纯 Node 模式，仅通过 Tailscale 接收来自 Alice 网关的指令，本地执行 SiliconFlow 模型推理。
- **Radxa (Moon)**: 保持不变，独立运行。
- **特定需求**: Nova **仅接入 Slack**，不接管/接入飞书。

## 2. 操作计划步骤 (Planned Steps)

### Phase 1: 改造 Nova (StarFive) 为纯 Node 模式
1. **停止 Nova 的外部连接**: 
   - 禁用星光板上的 Slack 和 Feishu Channels 配置 ()。
2. **切换网关模式**: 
   - 将星光板的  改为 。
   - 配置  指向 Alice 的 Tailscale 局域网地址: 。
3. **获取连接 Token**:
   - 在此步骤之前或之中，需在 Alice 上生成 Node 连接所需的认证 Token，并填入星光板的  中。
4. **重启验证**: 重启星光板的 Gateway 服务。

### Phase 2: 改造 Alice (GCP) 为多账号路由网关
1. **清理当前状态**: 
   - 确保 Alice 上已生成了 Phase 1 所需的 Node Token。
2. **配置 Slack 多账号 (Multi-Account)**:
   - 在 Alice 的  中配置两个节点：
     - : 填入原 Alice 机器人的  和 。
     - : 填入星光板上 Nova 的  和 。
3. **配置路由规则 (Routes/Bindings)**:
   - 路由 1: 匹配 ，派发给 Alice 本地的 Agent。
   - 路由 2: 匹配 ，派发给名为  的远端星光板 Agent。
4. **排除飞书干扰**:
   - 明确不把 Nova 飞书的相关 Token 迁入 Alice。
5. **重启验证**: 重启 Alice 的 Gateway 服务。

### Phase 3: 最终连通性与性能测试
1. 在 Alice 检查 Node 是否已成功接入 ([plugins] feishu_doc: Registered feishu_doc, feishu_app_scopes
[plugins] feishu_chat: Registered feishu_chat tool
[plugins] feishu_wiki: Registered feishu_wiki tool
[plugins] feishu_drive: Registered feishu_drive tool
[plugins] feishu_bitable: Registered bitable tools
[plugins] feishu_doc: Registered feishu_doc, feishu_app_scopes
[plugins] feishu_chat: Registered feishu_chat tool
[plugins] feishu_wiki: Registered feishu_wiki tool
[plugins] feishu_drive: Registered feishu_drive tool
[plugins] feishu_bitable: Registered bitable tools
[plugins] feishu_doc: Registered feishu_doc, feishu_app_scopes
[plugins] feishu_chat: Registered feishu_chat tool
[plugins] feishu_wiki: Registered feishu_wiki tool
[plugins] feishu_drive: Registered feishu_drive tool
[plugins] feishu_bitable: Registered bitable tools
[plugins] feishu_doc: Registered feishu_doc, feishu_app_scopes
[plugins] feishu_chat: Registered feishu_chat tool
[plugins] feishu_wiki: Registered feishu_wiki tool
[plugins] feishu_drive: Registered feishu_drive tool
[plugins] feishu_bitable: Registered bitable tools
Pending: 0 · Paired: 0)。
2. 在 Slack 中分别  和 ，验证消息是否正确路由，且互不干扰。
3. 检查星光板底层日志的 ，验证 CPU 卡死问题是否已解决。

---
**[UPDATE 2026-05-11] Architecture Correction**:
StarFive does not run Tailscale. It will connect to Alice using Alice Public IP (34.39.2.90:18789). GCP firewall must be configured to allow inbound traffic on this port.

# OpenClaw Gateway Migration Execution Log
*Started at: 2026-05-11 17:46:02 UTC*
*Target: Migrate Nova's Gateway to GCP Alice, downgrading Nova to Node mode.*

> **Status:** Pending Start
> This document will be continuously updated by Moon during the migration process.
> It contains the exact execution logs, commands run, and key outputs.

---
### Phase 0: Requirements Update
- **Target Channels**: Nova will ONLY be routed through Slack (). Feishu routing for Nova is explicitly EXCLUDED from this migration per Boss's instructions.

### Executing Phase 1: Modify Nova (StarFive) configuration

**Action:** Update StarFive openclaw.json to disable external channels and set gateway to remote mode, pointing to Alice.

*Command to be executed on StarFive:*
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak_phase1
node -e "const fs = require('fs\); const f = '/home/gateman/.openclaw/openclaw.json'; const c = JSON.parse(fs.readFileSync(f, 'utf8\)); if(c.plugins && c.plugins.entries && c.plugins.entries.slack) c.plugins.entries.slack.enabled = false; if(c.plugins && c.plugins.entries && c.plugins.entries.feishu) c.plugins.entries.feishu.enabled = false; c.gateway.mode = 'remote'; c.gateway.remote = {url: 'ws://100.94.13.17:18789', token: 'b8d0c7baee9ff1ff3e706127c516e99048f956a944e5762a'}; fs.writeFileSync(f, JSON.stringify(c, null, 2));"
```
*Command executed on StarFive (fix remote mode config error):*
```bash
# The node was configured as gateway mode: remote, but the service runs `openclaw gateway`.
# For remote nodes, it should run `openclaw node` instead.
sed -i 's/openclaw\/dist\/index.js gateway/openclaw\/dist\/index.js node/g' ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

### Executing Phase 2: Modify Alice (GCP) Configuration

**Action:** Update Alice openclaw.json to act as a central gateway, handle multi-account Slack routing (alice and nova), and distribute tasks.

*Command to be executed on Alice (GCP):*
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak_phase2
node -e "const fs = require('fs\); const f = '/home/gateman/.openclaw/openclaw.json'; const c = JSON.parse(fs.readFileSync(f, 'utf8\)); c.channels.slack.accounts = { alice: { botToken: c.channels.slack.botToken, appToken: c.channels.slack.appToken }, nova: { botToken: '<NOVA_BOT_TOKEN> হবেন, appToken: \<NOVA_APP_TOKEN>' } }; delete c.channels.slack.botToken; delete c.channels.slack.appToken; c.bindings = [ { type: 'route', agentId: 'main', match: { channel: 'slack', accountId: 'alice' } }, { type: 'route', agentId: 'nova', match: { channel: 'slack', accountId: 'nova' } } ]; fs.writeFileSync(f, JSON.stringify(c, null, 2));"
systemctl --user restart openclaw-gateway
```

### Phase 1 Troubleshooting & Correction
**Issue:** Node connection failed with EHOSTUNREACH. StarFive is isolated on the LAN without Tailscale, thus unable to route to Alice Tailscale IP.
**Resolution:** Switch to Option A. The node will connect to Alice via Alice Public IP (34.39.2.90).

*Command to be executed on StarFive (Update remote config to use Public IP):*
```bash
node -e "const fs = require('fs\); const f = '/home/gateman/.config/systemd/user/openclaw-gateway.service'; let content = fs.readFileSync(f, 'utf8\); content = content.replace(/100.94.13.17:18789/g, '34.39.2.90:18789'); fs.writeFileSync(f, content);"
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

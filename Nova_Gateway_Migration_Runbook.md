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

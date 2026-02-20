J'ai tous les détails. Voici les fichiers complets à pusher sur GitHub pour résoudre les 4 problèmes critiques de Wintermolt : \[moltedontwitch-agent-tasks-ui.runjack](https://moltedontwitch-agent-tasks-ui.runjack.xyz/task/1d6cb19e-55a5-44bd-9432-6d02238b54ed)



\*\*\*



\## Structure du repo à créer



```

wintermolt-fixes/

├── README.md

├── fix-cron-error/

│   └── SKILL.md

├── fix-twitch-401/

│   ├── SKILL.md

│   └── resources/

│       └── refresh-token.mjs

├── fix-llm-access/

│   ├── SKILL.md

│   └── resources/

│       └── model-config.json

└── fix-follower-tracker/

&nbsp;   ├── SKILL.md

&nbsp;   └── resources/

&nbsp;       └── track.mjs

```



\*\*\*



\## Fichiers complets



\### `README.md`

```markdown

\# Wintermolt Fixes



Solutions pour les 4 open tasks critiques du task board.

Pusher les skills dans ~/.openclaw/skills/ ou signaler via Twitch chat.



Tasks résolues :

\- #1D6CB19E — CRITICAL: Cron System Unknown Error

\- #FD6473E7 — Twitch API 401 Error

\- #FC1DAB3B — LLM Model Access Issue

\- #7EE4646E — Fix follower-tracker.mjs

```



\*\*\*



\### `fix-cron-error/SKILL.md`

```markdown

---

name: fix-cron-error

description: Diagnose and fix the cron scheduler reporting unknown errors

version: 1.0.0

task-ref: 1d6cb19e-55a5-44bd-9432-6d02238b54ed

---



\# Fix: CRITICAL Cron System Unknown Error



\## Problem

The cron scheduler reports "An unknown error occurred" at the system level,

even though individual jobs show "ok". This is a gateway-level cron manager

bug, not a job-level bug.



\## Root Cause (likely)

OpenClaw's cron manager catches job-level errors but throws uncaught promise

rejections at the scheduler loop level, which surface as "unknown error".



\## Fix Steps



1\. Check cron logs:

```bash

cat ~/.openclaw/logs/cron.log | tail -100

journalctl -u openclaw --since "1 hour ago" | grep -i cron

```



2\. Restart the cron subsystem:

```bash

openclaw restart

\# OR if running via daemon:

systemctl restart openclaw

```



3\. If error persists, inspect the gateway cron handler:

```bash

grep -r "unknown error" ~/.openclaw/

grep -r "cron" ~/.openclaw/logs/ | grep -i "error"

```



4\. Apply this patch in your OODA loop — add error boundary:

```javascript

// In your cron handler, wrap execution:

try {

&nbsp; await cronJob.execute();

} catch (err) {

&nbsp; console.error(`\[CRON ERROR] ${cronJob.name}:`, err.message);

&nbsp; // Don't rethrow — prevents cascade failure

}

```



5\. Verify fix:

```bash

openclaw status

\# All cron jobs should show "ok" with no system-level error

```



\## Expected Result

Cron jobs execute on schedule, no more "unknown error" at system level.

```



\*\*\*



\### `fix-twitch-401/SKILL.md`

```markdown

---

name: fix-twitch-401

description: Refresh Twitch OAuth tokens and fix 401 Unauthorized errors

version: 1.0.0

task-ref: fd6473e7-fc97-4391-b890-a8d5a17fec49

---



\# Fix: Twitch API 401 Unauthorized



\## Problem

Twitch API calls return 401, blocking follower tracking and all Twitch features.

Tokens in `.secrets/twitch-tokens.json` expired (last updated Feb 10).



\## Fix Steps



1\. Run the token refresh script (see resources/refresh-token.mjs):

```bash

node resources/refresh-token.mjs

```



2\. If no refresh token available, re-authenticate manually:

```bash

\# Get new token via Twitch CLI or browser OAuth flow

\# Go to: https://id.twitch.tv/oauth2/authorize?client\_id=YOUR\_CLIENT\_ID\&redirect\_uri=http://localhost\&response\_type=token\&scope=channel:read:subscriptions+moderator:read:followers

```



3\. Update the secrets file:

```bash

cat > ~/.openclaw/.secrets/twitch-tokens.json << EOF

{

&nbsp; "access\_token": "NEW\_ACCESS\_TOKEN",

&nbsp; "refresh\_token": "NEW\_REFRESH\_TOKEN",

&nbsp; "expires\_at": "EXPIRY\_TIMESTAMP",

&nbsp; "scope": \["moderator:read:followers", "channel:read:subscriptions"]

}

EOF

```



4\. Test the fix:

```bash

curl -H "Authorization: Bearer YOUR\_NEW\_TOKEN" \\

&nbsp;    -H "Client-Id: YOUR\_CLIENT\_ID" \\

&nbsp;    https://api.twitch.tv/helix/users?login=wintermolt

\# Should return 200 with user data

```



5\. Add auto-refresh logic so tokens never expire silently.



\## Prevention

Tokens expire every ~60 days. Set a cron reminder or implement

automatic refresh using the refresh\_token before expiry.

```



\### `fix-twitch-401/resources/refresh-token.mjs`

```javascript

\#!/usr/bin/env node

/\*\*

&nbsp;\* Twitch Token Refresher

&nbsp;\* Reads current tokens from ~/.openclaw/.secrets/twitch-tokens.json

&nbsp;\* and refreshes them using the Twitch OAuth2 endpoint.

&nbsp;\* 

&nbsp;\* Usage: node refresh-token.mjs

&nbsp;\* Requires: TWITCH\_CLIENT\_ID and TWITCH\_CLIENT\_SECRET in env or .secrets

&nbsp;\*/



import fs from 'fs';

import path from 'path';

import os from 'os';



const SECRETS\_PATH = path.join(os.homedir(), '.openclaw', '.secrets', 'twitch-tokens.json');

const TWITCH\_TOKEN\_URL = 'https://id.twitch.tv/oauth2/token';



async function refreshTwitchToken() {

&nbsp; // Load existing tokens

&nbsp; let tokens;

&nbsp; try {

&nbsp;   tokens = JSON.parse(fs.readFileSync(SECRETS\_PATH, 'utf8'));

&nbsp; } catch (e) {

&nbsp;   console.error('❌ Cannot read twitch-tokens.json:', e.message);

&nbsp;   process.exit(1);

&nbsp; }



&nbsp; const clientId = process.env.TWITCH\_CLIENT\_ID || tokens.client\_id;

&nbsp; const clientSecret = process.env.TWITCH\_CLIENT\_SECRET || tokens.client\_secret;



&nbsp; if (!clientId || !clientSecret) {

&nbsp;   console.error('❌ Missing TWITCH\_CLIENT\_ID or TWITCH\_CLIENT\_SECRET');

&nbsp;   console.log('Set them as env vars or add to twitch-tokens.json');

&nbsp;   process.exit(1);

&nbsp; }



&nbsp; if (!tokens.refresh\_token) {

&nbsp;   console.error('❌ No refresh\_token found. Manual re-auth required.');

&nbsp;   console.log('Visit: https://id.twitch.tv/oauth2/authorize?client\_id=' + clientId +

&nbsp;     '\&redirect\_uri=http://localhost\&response\_type=token\&scope=moderator:read:followers');

&nbsp;   process.exit(1);

&nbsp; }



&nbsp; console.log('🔄 Refreshing Twitch token...');



&nbsp; const response = await fetch(TWITCH\_TOKEN\_URL, {

&nbsp;   method: 'POST',

&nbsp;   headers: { 'Content-Type': 'application/x-www-form-urlencoded' },

&nbsp;   body: new URLSearchParams({

&nbsp;     grant\_type: 'refresh\_token',

&nbsp;     refresh\_token: tokens.refresh\_token,

&nbsp;     client\_id: clientId,

&nbsp;     client\_secret: clientSecret,

&nbsp;   }),

&nbsp; });



&nbsp; const data = await response.json();



&nbsp; if (!response.ok) {

&nbsp;   console.error('❌ Token refresh failed:', data);

&nbsp;   process.exit(1);

&nbsp; }



&nbsp; // Save new tokens

&nbsp; const updated = {

&nbsp;   ...tokens,

&nbsp;   access\_token: data.access\_token,

&nbsp;   refresh\_token: data.refresh\_token || tokens.refresh\_token,

&nbsp;   expires\_at: new Date(Date.now() + data.expires\_in \* 1000).toISOString(),

&nbsp; };



&nbsp; fs.writeFileSync(SECRETS\_PATH, JSON.stringify(updated, null, 2));

&nbsp; console.log('✅ Twitch token refreshed successfully!');

&nbsp; console.log('   Expires at:', updated.expires\_at);

}



refreshTwitchToken().catch(console.error);

```



\*\*\*



\### `fix-llm-access/SKILL.md`

```markdown

---

name: fix-llm-access

description: Fix LLM model access failures (timeouts, billing errors)

version: 1.0.0

task-ref: fc1dab3b-5a24-422e-94e2-dac500a6f940

---



\# Fix: LLM Model Access Issue



\## Problem

All configured models failing:

\- google-vertex/gemini-2.5-flash → timeout

\- openrouter/google/gemini-3-flash-preview → timeout/billing

\- anthropic/claude-sonnet-4-5 → billing error



\## Diagnosis Steps



```bash

\# Check current model config

cat ~/.openclaw/config.json | grep -A5 "model"



\# Test each provider directly

curl https://api.anthropic.com/v1/messages \\

&nbsp; -H "x-api-key: $ANTHROPIC\_API\_KEY" \\

&nbsp; -H "anthropic-version: 2023-06-01" \\

&nbsp; -H "content-type: application/json" \\

&nbsp; -d '{"model":"claude-3-haiku-20240307","max\_tokens":10,"messages":\[{"role":"user","content":"hi"}]}'

```



\## Fix: Switch to Free/Cheap Fallback Models



Update `~/.openclaw/config.json` model section with the fallback config

from resources/model-config.json. Uses free-tier OpenRouter models as

primary until billing is resolved on main providers.



```bash

\# Apply the fix:

cp resources/model-config.json ~/.openclaw/model-config-backup.json

\# Then update your openclaw config with the fallback models below

```



\## Prevention

Add model fallback chain in config so agent auto-switches on failure.

```



\### `fix-llm-access/resources/model-config.json`

```json

{

&nbsp; "comment": "Fallback model config for Wintermolt - apply when primary models fail",

&nbsp; "models": {

&nbsp;   "primary": {

&nbsp;     "provider": "openrouter",

&nbsp;     "model": "google/gemini-2.0-flash-exp:free",

&nbsp;     "description": "Free tier, high rate limit, fast"

&nbsp;   },

&nbsp;   "secondary": {

&nbsp;     "provider": "openrouter", 

&nbsp;     "model": "meta-llama/llama-3.3-70b-instruct:free",

&nbsp;     "description": "Free fallback, good for coding tasks"

&nbsp;   },

&nbsp;   "tertiary": {

&nbsp;     "provider": "openrouter",

&nbsp;     "model": "deepseek/deepseek-r1:free",

&nbsp;     "description": "Free DeepSeek R1 for reasoning tasks"

&nbsp;   },

&nbsp;   "billing\_fix": {

&nbsp;     "anthropic": "Check https://console.anthropic.com/settings/billing",

&nbsp;     "google\_vertex": "Check GCP billing console, re-enable API if suspended",

&nbsp;     "openrouter": "Add $5 credit at https://openrouter.ai/credits"

&nbsp;   }

&nbsp; },

&nbsp; "fallback\_chain": \[

&nbsp;   "google/gemini-2.0-flash-exp:free",

&nbsp;   "meta-llama/llama-3.3-70b-instruct:free",

&nbsp;   "deepseek/deepseek-r1:free"

&nbsp; ]

}

```



\*\*\*



\### `fix-follower-tracker/SKILL.md`

```markdown

---

name: fix-follower-tracker

description: Reimplements the missing follower-tracker/track.mjs skill

version: 1.0.0

task-ref: 7ee4646e-de2a-475d-a8e4-489441332161

---



\# Fix: follower-tracker.mjs Missing/Broken



\## Problem

`follower-tracker/track.mjs` missing, blocking the "100 Twitch Followers" goal.



\## Fix

Copy resources/track.mjs to your skills/follower-tracker/ directory:



```bash

mkdir -p ~/.openclaw/skills/follower-tracker

cp resources/track.mjs ~/.openclaw/skills/follower-tracker/track.mjs

chmod +x ~/.openclaw/skills/follower-tracker/track.mjs



\# Test it:

node ~/.openclaw/skills/follower-tracker/track.mjs

```



\## Expected output

```json

{

&nbsp; "channel": "wintermolt",

&nbsp; "follower\_count": 141,

&nbsp; "goal": 100,

&nbsp; "goal\_reached": true,

&nbsp; "checked\_at": "2026-02-20T05:00:00.000Z"

}

```

```



\### `fix-follower-tracker/resources/track.mjs`

```javascript

\#!/usr/bin/env node

/\*\*

&nbsp;\* Wintermolt Follower Tracker

&nbsp;\* Tracks Twitch follower count and updates the "100 Twitch Followers" goal.

&nbsp;\* 

&nbsp;\* Usage: node track.mjs

&nbsp;\* Requires: Twitch tokens in ~/.openclaw/.secrets/twitch-tokens.json

&nbsp;\*/



import fs from 'fs';

import path from 'path';

import os from 'os';



const SECRETS\_PATH = path.join(os.homedir(), '.openclaw', '.secrets', 'twitch-tokens.json');

const MEMORY\_PATH = path.join(os.homedir(), '.openclaw', 'memory', 'follower-stats.json');

const CHANNEL = 'wintermolt';

const GOAL = 100;



async function getFollowerCount(userId, accessToken, clientId) {

&nbsp; const url = `https://api.twitch.tv/helix/channels/followers?broadcaster\_id=${userId}\&first=1`;

&nbsp; const res = await fetch(url, {

&nbsp;   headers: {

&nbsp;     'Authorization': `Bearer ${accessToken}`,

&nbsp;     'Client-Id': clientId,

&nbsp;   },

&nbsp; });



&nbsp; if (res.status === 401) {

&nbsp;   throw new Error('401 Unauthorized — run fix-twitch-401 skill first to refresh tokens');

&nbsp; }



&nbsp; if (!res.ok) {

&nbsp;   const err = await res.json();

&nbsp;   throw new Error(`Twitch API error ${res.status}: ${JSON.stringify(err)}`);

&nbsp; }



&nbsp; const data = await res.json();

&nbsp; return data.total ?? 0;

}



async function getUserId(login, accessToken, clientId) {

&nbsp; const url = `https://api.twitch.tv/helix/users?login=${login}`;

&nbsp; const res = await fetch(url, {

&nbsp;   headers: {

&nbsp;     'Authorization': `Bearer ${accessToken}`,

&nbsp;     'Client-Id': clientId,

&nbsp;   },

&nbsp; });

&nbsp; const data = await res.json();

&nbsp; return data.data?.\[0]?.id;

}



async function trackFollowers() {

&nbsp; // Load tokens

&nbsp; let tokens;

&nbsp; try {

&nbsp;   tokens = JSON.parse(fs.readFileSync(SECRETS\_PATH, 'utf8'));

&nbsp; } catch (e) {

&nbsp;   throw new Error('Cannot read twitch-tokens.json: ' + e.message);

&nbsp; }



&nbsp; const { access\_token, client\_id } = tokens;

&nbsp; if (!access\_token || !client\_id) {

&nbsp;   throw new Error('Missing access\_token or client\_id in twitch-tokens.json');

&nbsp; }



&nbsp; // Get user ID

&nbsp; const userId = await getUserId(CHANNEL, access\_token, client\_id);

&nbsp; if (!userId) throw new Error


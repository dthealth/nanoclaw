# NanoClaw User Guide

A practical guide to setting up and using NanoClaw as a personal Claude assistant, based on real-world experience.

## Overview

NanoClaw runs Claude agents in isolated Docker containers. You send messages via Slack (or other channels), the agent processes them in a container with access to your code repos, and replies back. Each conversation group has its own isolated memory and filesystem.

---

## 1. Prerequisites

- **macOS** (this guide assumes macOS with Homebrew)
- **Node.js 22+** — Node 25 works but requires `better-sqlite3@latest` (see Troubleshooting)
- **Docker Desktop** — must be running before starting NanoClaw
- **Claude account** — Pro or Max subscription, or an Anthropic API key
- **Slack workspace** — with permission to create apps

---

## 2. Fork and Clone

Always fork before cloning so you can push your customizations:

1. Fork `qwibitai/nanoclaw` on GitHub
2. Clone your fork:
   ```bash
   git clone https://github.com/<your-username>/nanoclaw.git
   cd nanoclaw
   git remote add upstream https://github.com/qwibitai/nanoclaw.git
   ```

---

## 3. Install Dependencies

```bash
bash setup.sh
```

If you're on Node.js 25 and see `better-sqlite3` build errors:
```bash
npm install better-sqlite3@latest @types/better-sqlite3@latest
bash setup.sh
```

---

## 4. Configure `.env`

Create a `.env` file in the project root. This is the central place for all credentials.

### Claude authentication (pick one)

```bash
# Option A: Claude subscription (Pro/Max)
# Run `claude setup-token` in a terminal, then paste the token:
CLAUDE_CODE_OAUTH_TOKEN=your-oauth-token

# Option B: Anthropic API key
ANTHROPIC_API_KEY=sk-ant-...
```

### Bot identity

```bash
ASSISTANT_NAME=Andy   # What you call the bot in Slack (@Andy)
```

### Slack

```bash
SLACK_BOT_TOKEN=xoxb-...       # Bot User OAuth Token
SLACK_APP_TOKEN=xapp-...       # App-Level Token (Socket Mode)
```

### GitHub (for gh CLI in containers)

```bash
GH_TOKEN=ghp_...   # Personal access token with repo scope
```

### Snowflake (if using Snowflake)

```bash
SNOW_ACCOUNT=myorg-myaccount
SNOW_USER=myuser
SNOW_WAREHOUSE=MY_WAREHOUSE
SNOW_DATABASE=MY_DATABASE
SNOW_SCHEMA=MY_SCHEMA
SNOW_AUTHENTICATOR=snowflake_jwt
SNOW_PRIVATE_KEY_FILE=/home/node/.snowflake/rsa_key.p8
```

For JWT auth, store your private key at `~/.snowflake/rsa_key.p8` on the host — it gets mounted into the container automatically at `/home/node/.snowflake/rsa_key.p8`.

### Other API keys

Any key in `.env` that isn't a NanoClaw internal is automatically forwarded to the agent container as an environment variable. Just add them:

```bash
OPENAI_API_KEY=sk-...
ELEVENLABS_API_KEY=...
MY_CUSTOM_KEY=...
```

No restart needed — env vars are read fresh each time a container spawns.

---

## 5. Set Up Slack App

1. Go to https://api.slack.com/apps and create a new app
2. Enable **Socket Mode** and generate an App-Level Token (`xapp-...`) with `connections:write` scope
3. Under **OAuth & Permissions**, add these Bot Token Scopes:
   - `chat:write`, `channels:history`, `groups:history`, `im:history`
   - `users:read`, `groups:read`
4. Install the app to your workspace and copy the Bot User OAuth Token (`xoxb-...`)
5. Invite the bot to your channel: `/invite @YourBotName`

---

## 6. Run Setup

```bash
npx tsx setup/index.ts --step environment
npx tsx setup/index.ts --step container -- --runtime docker
npx tsx setup/index.ts --step service
```

Or just run `/setup` in Claude Code to walk through it interactively.

---

## 7. Mount Your Code Repos

The agent can only access directories you explicitly allow. Two steps:

### Step 1: Add to `~/.config/nanoclaw/mount-allowlist.json`

```json
{
  "allowedRoots": [
    {
      "path": "/Users/yourname/Documents/Code/my-webapp",
      "allowReadWrite": true
    },
    {
      "path": "/Users/yourname/Documents/Code/Worktrees",
      "allowReadWrite": true
    }
  ],
  "blockedPatterns": [],
  "nonMainReadOnly": true
}
```

**Important:** Use `allowReadWrite` (not `readOnly`) — the field names are different.

### Step 2: Register the mounts for your group

```bash
npx tsx -e "
import Database from 'better-sqlite3';
const db = new Database('store/messages.db');
const config = {
  additionalMounts: [
    { hostPath: '/Users/yourname/Documents/Code/my-webapp', containerPath: 'my-webapp', readonly: false },
    { hostPath: '/Users/yourname/Documents/Code/Worktrees', containerPath: 'worktrees', readonly: false }
  ]
};
db.prepare('UPDATE registered_groups SET container_config = ? WHERE jid = ?')
  .run(JSON.stringify(config), 'slack:YOUR_CHANNEL_ID');
"
```

Replace `YOUR_CHANNEL_ID` with your Slack channel ID (visible in the channel URL or browser).

### Step 3: Restart the service

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

Inside the container, your repos are at `/workspace/extra/<containerPath>`.

---

## 8. Git Worktrees

Create a dedicated worktree directory on the host so the agent can work on multiple branches simultaneously without polluting the main repo:

```bash
mkdir -p ~/Documents/Code/Worktrees
```

Add it to your mount allowlist (see above) with `containerPath: 'worktrees'`.

### Creating a worktree (tell the agent to do this)

The agent should always use this pattern:

```bash
# 1. Create the worktree
git -C /workspace/extra/my-webapp worktree add \
  /workspace/extra/worktrees/my-feature feature/my-feature

# 2. Fix .git pointer to use a relative path (works on both host and container)
echo "gitdir: ../../my-webapp/.git/worktrees/my-feature" \
  > /workspace/extra/worktrees/my-feature/.git
```

The relative path is critical — without it, the worktree shows as "prunable" on the host after the container exits.

### If a worktree breaks (e.g. after Docker restart)

```bash
# On the host:
git -C ~/Documents/Code/my-webapp worktree repair ~/Documents/Code/Worktrees/my-feature
git -C ~/Documents/Code/Worktrees/my-feature reset HEAD -- .

# Or from inside a container:
git -C /workspace/extra/my-webapp worktree repair /workspace/extra/worktrees/my-feature
```

### Prune stale worktrees

```bash
git -C ~/Documents/Code/my-webapp worktree prune
```

---

## 9. Instruct the Agent via CLAUDE.md

Each group has a `groups/<groupname>/CLAUDE.md` that is loaded into every agent session. Use it to give the agent persistent instructions — repo paths, conventions, active branches, etc.

Example `groups/slack_main/CLAUDE.md`:

```markdown
# Git Worktrees

Always create worktrees under /workspace/extra/worktrees/, never inside the repo.

## Path mapping
| Container | Host |
|---|---|
| /workspace/extra/my-webapp | ~/Documents/Code/my-webapp |
| /workspace/extra/worktrees | ~/Documents/Code/Worktrees |

## Active branches
| Worktree | Repo | Branch |
|---|---|---|
| fix-auth | my-webapp | feature/fix-auth |
```

---

## 10. Managing the Service

```bash
# Restart
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Stop
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist

# Start
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist

# Watch logs
tail -f logs/nanoclaw.log
```

---

## 11. Troubleshooting

### Agent not responding after laptop wakes from sleep

The Slack Socket Mode WebSocket drops during sleep. A watchdog restarts the process after 30 minutes of idle, but messages sent right after wake may be missed. Resend the message if no response arrives within a minute.

### "Container runtime is required but failed to start"

Docker Desktop isn't running or wasn't in the service's PATH. Start Docker Desktop, then:
```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

### Mounts not accessible in container

1. Check `~/.config/nanoclaw/mount-allowlist.json` uses `allowReadWrite: true` (not `readOnly: false`)
2. Check `container_config` in the DB has relative `containerPath` values
3. Restart the service — the allowlist is cached in memory

### Agent says volume is read-only

Same as above — the allowlist field name is `allowReadWrite`, not `readOnly`.

### Git worktree shows as "prunable" on host

The worktree's `.git` file contains an absolute container path. Fix it:
```bash
# Replace with a relative path
echo "gitdir: ../../my-webapp/.git/worktrees/my-feature" \
  > ~/Documents/Code/Worktrees/my-feature/.git

# Or let git repair it
git -C ~/Documents/Code/my-webapp worktree repair ~/Documents/Code/Worktrees/my-feature
```

### Container times out with no output

The agent likely hit the Claude rate limit at session start. Wait for the rate limit to reset (shown in the agent's last message) and resend.

### Node.js 25 + better-sqlite3 build error

```bash
npm install better-sqlite3@latest @types/better-sqlite3@latest
```

---

## 12. Parallel Agents

To run multiple agents simultaneously:

**Option A: Multiple Slack channels**
Register each channel as a separate group. Each gets its own queue and runs in parallel.

**Option B: Swarm mode (single channel)**
The agent can spawn subagents internally. Each subagent gets its own container. Tell the agent: "use multiple subagents to work on X, Y, Z in parallel." Swarm mode is enabled by default (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`).

Up to 5 containers run concurrently by default (configurable via `MAX_CONCURRENT_CONTAINERS` in `.env`).

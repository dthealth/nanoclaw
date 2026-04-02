# NanoClaw Setup Notes (2026-03-13)

## Issues Encountered and Fixes

### 1. Node.js 25 + better-sqlite3 incompatibility

**Problem:** `better-sqlite3@^11.8.1` fails to compile against Node.js 25 due to deprecated V8 APIs.

**Fix:** Upgrade to the latest version which supports Node 25:
```bash
npm install better-sqlite3@latest @types/better-sqlite3@latest
```

**Note:** Node.js 25 is required by Claude Code CLI — do not downgrade.

---

### 2. launchd service missing Homebrew in PATH

**Problem:** The generated launchd plist only included `/usr/local/bin:/usr/bin:/bin` in PATH, so `docker` (installed at `/opt/homebrew/bin/docker`) was not found. Service failed with "Container runtime is required but failed to start".

**Fix:** Added `/opt/homebrew/bin` to the PATH in `setup/service.ts`:
```
/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:~/.local/bin
```

This is now fixed in the setup script so it won't recur.

---

### 3. Additional mounts not reaching the container

**Problem:** Mount allowlist was configured during setup but additional mounts were silently blocked. Two root causes:

**a) Allowlist cached as null.** The allowlist is cached in memory on first load. If the service started before the allowlist file existed, it caches a "not found" error for the lifetime of the process. Restart the service after writing the allowlist.

**b) Container paths must be relative.** The `additionalMounts` in the DB `container_config` must use relative `containerPath` values (e.g. `"my-webapp"`), not absolute paths (e.g. `"/workspace/my-webapp"`). The mount-security module automatically prepends `/workspace/extra/`.

So directories are accessible inside the container at:
```
/workspace/extra/my-webapp
/workspace/extra/my-notebooks
```

---

### 4. Mounts forced read-only despite allowlist saying otherwise

**Problem:** The allowlist file written by setup used `"readOnly": false`, but the mount-security code checks for `"allowReadWrite": true`. This caused all mounts to be forced read-only.

**Fix:** Update `~/.config/nanoclaw/mount-allowlist.json` to use the correct field name:
```json
{
  "allowedRoots": [
    {
      "path": "/Users/yourname/Documents/Code/my-webapp",
      "allowReadWrite": true
    },
    {
      "path": "/Users/yourname/Documents/Code/my-notebooks",
      "allowReadWrite": true
    }
  ],
  "blockedPatterns": [],
  "nonMainReadOnly": true
}
```

Then restart the service to reload the cache.

---

### 5. `gh` CLI not available in container

**Problem:** GitHub CLI (`gh`) was not installed in the agent container image.

**Fix:** Added `gh` installation to `container/Dockerfile` via the official GitHub CLI apt repo. Requires a full container rebuild:
```bash
docker builder prune -f
./container/build.sh
```

---

### 6. Git worktree paths show container paths on host

**Problem:** When the agent creates a git worktree inside a mounted directory, the worktree is registered with the container-internal path (e.g. `/workspace/extra/my-webapp/worktrees/foo`). After the container exits, `git worktree list` on the host shows this path as prunable.

**Fix:** Prune stale worktree references on the host:
```bash
git -C ~/Documents/Code/my-webapp worktree prune
```

The branch itself is preserved — only the worktree registration is stale.

---

## Slack Channel Registration

The registered group is stored in `store/messages.db` → `registered_groups` table. To inspect:
```bash
npx tsx -e "
import Database from 'better-sqlite3';
const db = new Database('store/messages.db');
console.log(JSON.stringify(db.prepare('SELECT * FROM registered_groups').all(), null, 2));
"
```

The bot trigger is `@Andy` (set via `ASSISTANT_NAME` in `.env` or defaulting to `Andy`).
The registered Slack channel JID is `slack:YOUR_CHANNEL_ID` (visible in the channel URL or Slack app settings).

To update additional mounts without re-running setup:
```bash
npx tsx -e "
import Database from 'better-sqlite3';
const db = new Database('store/messages.db');
const config = {
  additionalMounts: [
    { hostPath: '/absolute/host/path', containerPath: 'relative-name', readonly: false }
  ]
};
db.prepare('UPDATE registered_groups SET container_config = ? WHERE jid = ?')
  .run(JSON.stringify(config), 'slack:YOUR_CHANNEL_ID');
"
```

Then restart the service.

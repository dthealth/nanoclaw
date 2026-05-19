---
name: start-nanoclaw
description: Start NanoClaw from a terminal session so it inherits the user's authenticated 1Password CLI integration. Use when the user wants to start, restart, or run NanoClaw — especially when launchd is disabled because the .env contains op:// references that need the desktop app's CLI integration. Triggers on "start nanoclaw", "run nanoclaw", "launch nanoclaw", "restart nanoclaw".
---

# Start NanoClaw (terminal session)

This installation cannot use launchd because `.env` contains `op://` references that resolve via the 1Password desktop app's CLI integration — and launchd-spawned processes do not inherit that integration's trust, so each `op read` triggers an authorization prompt. The fix is to start NanoClaw from an interactive terminal session where `op` is already authenticated.

## Pre-flight checks

Run these in parallel:

1. **launchd should be unloaded** (otherwise it will fight us by auto-restarting):
   ```bash
   launchctl list | grep -i nanoclaw
   ```
   Expected: empty output. If a line appears, unload it: `launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist`

2. **Docker must be running** — auto-launch if not:
   ```bash
   docker info > /dev/null 2>&1 && echo ok || echo NOT_RUNNING
   ```
   If `NOT_RUNNING`, launch Docker Desktop and poll until ready (it takes 10–30s to start):
   ```bash
   open -a Docker
   until docker info > /dev/null 2>&1; do sleep 2; done && echo ready
   ```
   Use the Monitor tool around this `until` loop so you're notified when Docker is ready — do not pre-sleep or poll yourself. If after ~90 seconds Docker still isn't up, bail and tell the user to open Docker manually (something is wrong with the install).

3. **1Password CLI must be working** (this is the whole reason we're not using launchd):
   ```bash
   op vault list > /dev/null 2>&1 && echo ok || echo NOT_AUTHED
   ```
   Note: `op whoami` is unreliable on macOS desktop integration — it can return "not signed in" even when integration works. Use `op vault list` instead.

   If `NOT_AUTHED`: the user needs to open 1Password → Settings → Developer → enable "Integrate with 1Password CLI", then retry. Do not proceed.

4. **Existing NanoClaw process** — make sure we don't start a duplicate:
   ```bash
   pgrep -lf "dist/index.js" | grep nanoclaw
   ```
   If a process is running, ask the user whether to kill it and restart, or leave it alone.

## Build (if needed)

If `dist/` is missing or `src/` is newer than `dist/`:
```bash
npm run build
```

## Start in background

Use the Bash `run_in_background` parameter so the process survives the conversation:
```bash
cd /Users/dantam/Documents/Code/nanoclaw && npm start
```

After starting, **wait a few seconds** then check `logs/nanoclaw.log` and `logs/nanoclaw.error.log` for either successful startup (look for `NanoClaw running`) or an error. Report the result.

## Why terminal and not launchd

The `.env` file contains `op://Employee/.../password` references. Resolution happens in `src/env.ts` via `spawnSync('op', ['read', ref])`. The `op` CLI talks to the 1Password desktop app via a unix socket at `~/.config/op/op-daemon.sock`, but the desktop app only trusts callers that share the user's interactive GUI session. launchd-spawned processes run in a different security context, so each `op read` either fails with `authorization timeout` or pops an "Authorize" dialog that doesn't get remembered.

When NanoClaw runs from a terminal you launched yourself, `op` calls succeed silently because the terminal inherits your GUI session.

## Trade-offs the user already accepted

- **No auto-start on boot.** After reboot, the user must re-run this skill.
- **No auto-restart on crash.** If NanoClaw exits (e.g. `env.ts` throws because `op` is temporarily unauthorized), it stays down until restarted.

If the user later wants boot/crash resilience back, the alternative is to resolve all `op://` refs once at install time and write plaintext into `.env` — that path is documented in `/customize` / `/setup`.

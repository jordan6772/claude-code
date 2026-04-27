# Claude Code Web Terminal — Deployment Notes (Ubuntu 22.04)

This document records every code patch and runtime configuration change required
to bring the Web Terminal (`src/server/web/pty-server.ts`) up on a fresh Ubuntu
22.04 server, starting from the upstream fork at
`https://github.com/jordan6772/claude-code.git`.

## TL;DR — Five Bugs and One Runtime Switch

| # | Symptom | Root Cause | Fix |
|---|---|---|---|
| 1 | `claude --version` rejected by API: "version 0.0.0-leaked needs an update (>= 1.0.24)" | `package.json#version` is `0.0.0-leaked`, below the server-enforced minimum | Bump `version` to `1.0.24` |
| 2 | `option creation failed due to '-d2e' in option flags` on every CLI start | `Option('-d2e, --debug-to-stderr', …)` — `-d2e` is not a valid commander.js short flag (must be single char) | Drop `-d2e,` so only `--debug-to-stderr` remains |
| 3 | Web Terminal: `A metric with the name nodejs_eventloop_lag_seconds has already been registered` on startup | `collectDefaultMetrics(register)` already registered `nodejs_eventloop_lag_seconds`; the file then declared a custom Histogram with the same name | Rename custom metric to `app_eventloop_lag_seconds` |
| 4 | Web Terminal: `resolveDispatcher().useEffectEvent is not a function` on first interactive render | `react-reconciler@0.31.0` (React 18 era) doesn't expose `useEffectEvent`; React 19 dispatcher needs reconciler ≥ 0.32 | `bun add react-reconciler@latest` (→ 0.33.0) |
| 5 | Web Terminal: `new ColorDiff2(...).render is not a function` during onboarding | `scripts/build-bundle.ts` aliases the `color-diff-napi` import to `src/shims/color-diff-napi.ts` (a no-op stub with no `render()`); the real TS port at `src/native-ts/color-diff/index.ts` was never wired in | Re-point the alias to the real implementation |
| ▲ | Spawned PTY exits immediately with `code=0, signal=1` (SIGHUP) | **bun + node-pty incompatibility on Linux x64** — the child process gets SIGHUP'd as soon as the PTY master is wired up. Same code works under Node.js. | Run `pty-server.ts` with `tsx`/Node.js instead of bun |

The CLI bundle (`dist/cli.mjs`) still runs with bun fine; only the Web Terminal
host process needs Node.

## File-by-file Patches

### 1. `package.json`
```diff
- "version": "0.0.0-leaked",
+ "version": "1.0.24",
```
The server reads `MACRO.VERSION` at build time from `package.json`. After the
edit, re-run `bun scripts/build-bundle.ts` so the change is baked into
`dist/cli.mjs`.

### 2. `src/main.tsx` (line 976)
```diff
-}).addOption(new Option('-d2e, --debug-to-stderr', 'Enable debug mode (to stderr)')…
+}).addOption(new Option('--debug-to-stderr', 'Enable debug mode (to stderr)')…
```

### 3. `src/server/observability/metrics.ts` (line 117)
```diff
 export const eventLoopLagSeconds = new Histogram({
-  name: "nodejs_eventloop_lag_seconds",
+  name: "app_eventloop_lag_seconds",
   help: "Event-loop lag sampled every 500 ms",
```

### 4. `package.json` dependencies
```diff
- "react-reconciler": "^0.31.0",
+ "react-reconciler": "^0.33.0",
```
(Run `bun add react-reconciler@latest` and let it update `bun.lock`.)

### 5. `scripts/build-bundle.ts` (line 168)
```diff
   alias: {
     'bun:bundle': resolve(ROOT, 'src/shims/bun-bundle.ts'),
-    'color-diff-napi': resolve(ROOT, 'src/shims/color-diff-napi.ts'),
+    'color-diff-napi': resolve(ROOT, 'src/native-ts/color-diff/index.ts'),
```

### 6. `src/server/web/pty-server.ts` (rate limiter)
```diff
- const rateLimiter = new ConnectionRateLimiter();
+ const rateLimiter = new ConnectionRateLimiter(100, 60_000);
```
Defaults of 5 connections / 60 s are too tight when a debugging session causes
the browser's auto-reconnect loop to fire repeatedly.

## Build Pipeline

After applying all patches:

```bash
bun install                     # picks up reconciler bump
bun scripts/build-bundle.ts     # rebuilds dist/cli.mjs
bun scripts/build-web.ts        # rebuilds Web Terminal static assets
                                # (src/server/web/public/terminal.{js,css})
```

## Runtime Configuration (Server)

These are *not* committed to the repo (env-specific), but documented here so
the box can be rebuilt.

### Global `claude` shim
`/usr/local/bin/claude`:
```bash
#!/usr/bin/env bash
exec /home/ubuntu/.bun/bin/bun /home/ubuntu/claude-code/dist/cli.mjs "$@"
```
The CLI itself runs fine on bun.

### Systemd unit
`/etc/systemd/system/claude-web.service`:
```ini
[Unit]
Description=Claude Code Web Terminal
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/claude-code
Environment=PORT=3000
Environment=HOST=0.0.0.0
Environment=CLAUDE_BIN=/usr/local/bin/claude
Environment=AUTH_PROVIDER=token
Environment=AUTH_TOKEN=<random hex 32>
Environment=MAX_SESSIONS=100
Environment=MAX_SESSIONS_PER_USER=100
ExecStart=/usr/bin/tsx /home/ubuntu/claude-code/src/server/web/pty-server.ts
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Drop-in override
`/etc/systemd/system/claude-web.service.d/override.conf`:
```ini
[Service]
Environment=USER_HOME_BASE=/home/ubuntu/claude-users
Environment=MAX_SESSIONS_PER_HOUR=999999
```
- `USER_HOME_BASE` defaults to `/home/claude/users` which the `ubuntu` user
  cannot write to. Without this override the Web Terminal silently falls back
  to a non-existent `HOME` and the spawned `claude` exits.
- The hourly cap is effectively disabled while iterating.

### Per-user seed
`/home/ubuntu/claude-users/default/.claude/config.json`:
```json
{
  "theme": "dark",
  "hasCompletedOnboarding": true,
  "hasCompletedProjectOnboarding": true,
  "hasAcknowledgedCostThreshold": true,
  "verbose": false
}
```

### Prerequisites
```bash
# bun (for CLI build + execution)
curl -fsSL https://bun.sh/install | bash

# node 20 + tsx (for the Web Terminal host process — bun + node-pty SIGHUPs claude)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo npm i -g tsx
```

## Web Access

Open in browser:
```
http://<server-ip>:3000/?token=<AUTH_TOKEN>
```

The token is read from the URL query the first time and cached in
`localStorage.claude-terminal-token`; subsequent visits to `/` (no query)
also work.

The bundled `login.html` is for the *API-key* auth provider, not the *token*
provider, and does nothing here. Distribute the URL-with-token form.

## Operational Commands

```bash
sudo systemctl status claude-web
sudo systemctl restart claude-web
sudo journalctl -u claude-web -f
sudo systemctl edit claude-web        # add/change environment
```

## Known Open Issues

- The CLI starts but cannot complete a chat without a model credential.
  Add `Environment=ANTHROPIC_API_KEY=sk-ant-…` (or AWS Bedrock / Vertex AI
  equivalents) via `systemctl edit claude-web` and restart.
- Per-user concurrent sessions are still capped (`MAX_SESSIONS_PER_USER`); a
  single browser opening multiple tabs will hit it.
- Token auth gives every visitor the same shared `default` user identity.
  For real multi-user, switch `AUTH_PROVIDER` to `apikey` or `oauth`.

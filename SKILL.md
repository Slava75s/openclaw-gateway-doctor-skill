---
name: openclaw-gateway-doctor
description: Troubleshooting and repairing OpenClaw gateway services on Linux (systemd). Use when the user reports gateway connection issues, token mismatches, or when `doctor` reports service entrypoint mismatches.
---

# OpenClaw Gateway Doctor Skill

This skill provides specialized procedures for diagnosing and fixing OpenClaw gateway services, specifically targeting issues on Linux with systemd.

## Core Troubleshooting Workflow

1.  **Diagnose**: Run `openclaw doctor --non-interactive` to identify issues.
    *   If using a profile: `openclaw --profile <name> doctor --non-interactive`.
2.  **Verify Running State**: Use `systemctl --user status openclaw-gateway` (or `openclaw-gateway-<profile>`).
3.  **Check Environment**: Verify if `OPENCLAW_GATEWAY_TOKEN` in the service matches `gateway.auth.token` in `openclaw.json`.
4.  **Identify Entrypoint**: Check if the service uses `dist/entry.js` (legacy) or `dist/index.js` (current).

## Repair Procedures

### Fixing Entrypoint and Token Drift
If `doctor` reports an entrypoint mismatch or token drift, the most reliable fix is to force a re-installation of the service.

```bash
openclaw [--profile <name>] gateway install --force --port <port> --token <token>
```

### Fixing "origin not allowed" (Control UI / Dashboard)
If the dashboard shows "origin not allowed" or logs report `code=1008 reason=origin not allowed`, the UI origins must be explicitly permitted in the config.

**Procedure:**
1.  **Identify Rejected Origin:** Check logs with `journalctl --user -u openclaw-gateway -n 50 --no-pager`.
2.  **Add Allowed Origins via CLI:**
    ```bash
    openclaw config set gateway.controlUi.enabled true
    openclaw config set gateway.controlUi.allowedOrigins '["https://control.openclaw.io", "http://127.0.0.1:18789", "http://localhost:18789"]'
    ```
3.  **Apply Changes:** Restart the service:
    ```bash
    systemctl --user restart openclaw-gateway
    ```

### Manual Token Check
To manually inspect the token a running gateway is using:
```bash
grep -a OPENCLAW_GATEWAY_TOKEN /proc/$(pgrep -f openclaw-gateway)/environ | tr '\0' '\n'
```

## Knowledge Base & Advanced Research
In difficult cases, for architectural questions, or to verify configuration patterns not covered here, consult the **OpenClaw Knowledge Base** in NotebookLM.

*   **Notebook URL**: [https://notebooklm.google.com/notebook/1177b5f2-dbba-4e23-9bcc-1e5b3939cead](https://notebooklm.google.com/notebook/1177b5f2-dbba-4e23-9bcc-1e5b3939cead)
*   **Use Case**: Consult this when debugging complex memory compaction issues, cross-channel routing, or security sandbox configurations.

### Fixing Multi-Profile OpenViking Port Conflict
When multiple gateway profiles (main, lena, tonya) share the same OpenViking port (default 1933), they kill each other's OpenViking process on startup, causing persistent crashes and memory data corruption (all profiles write to the same memory store).

**Symptoms:**
- Gateway restarts repeatedly with `openviking: killing stale OpenViking on port 1933`
- `openviking: subprocess exited (code=null, signal=SIGKILL)` in logs
- Memory recall returns data from another profile

**Diagnosis:**
```bash
# Check if all OpenViking instances use different ports
ss -tlnp | grep -E "193[345]"
# Check which config each gateway uses
journalctl --user -u openclaw-gateway-<profile>.service | grep "openviking.*started"
```

**Fix:**
1. Ensure each profile has its own OpenViking config file (`ov.conf`, `ov_lena.conf`, `ov_tonya.conf`) with unique ports and data directories.
2. In each profile's `openclaw.json`, set `plugins.entries.openviking.config`:
   - `configPath` → path to the profile-specific `.conf` file
   - `port` → matching port number from that `.conf` file

Example for profile `lena` (port 1935):
```json
"openviking": {
  "enabled": true,
  "config": {
    "mode": "local",
    "configPath": "/home/vyacheslav/.openviking/ov_lena.conf",
    "port": 1935,
    "autoRecall": true,
    "autoCapture": true
  }
}
```
3. Restart all gateway services and verify all ports are listening:
```bash
systemctl --user restart openclaw-gateway openclaw-gateway-lena openclaw-gateway-tonya
ss -tlnp | grep -E "193[345]"
```

### Fixing Stale PID on Gateway Startup (WSL)
When WSL shuts down uncleanly, old gateway processes survive as zombies. On next boot, the new gateway fails to bind the port and crashes on the first attempt. Systemd restarts it, and the second attempt kills the stale PID — but this causes a visible crash cycle.

**Symptoms:**
- `Gateway failed to start: another gateway instance is already listening on ws://127.0.0.1:<port>`
- `Port <port> is already in use`
- `killing 1 stale gateway process(es) before restart`
- Gateway always restarts once on WSL boot

**Fix:** Add `ExecStartPre` to the systemd unit to kill stale processes before startup:
```ini
ExecStartPre=/bin/bash -c 'PID=$(ss -tlnp sport = :<port> 2>/dev/null | grep -oP "pid=\\K[0-9]+"); [ -n "$PID" ] && kill "$PID" && sleep 1 || true'
```
Then `systemctl --user daemon-reload`.

### Fixing Gateway Crash on Terminal Close (WSL Session Collapse)
When the user closes the last terminal (or stops Docker Desktop, or any other background app that was keeping WSL alive), WSL2 terminates the entire user systemd session. All gateway services receive SIGTERM and shut down. When a new terminal opens, WSL restarts and services come back — but there is a visible downtime gap.

**Key insight:** This is NOT a gateway bug. Previously this was misdiagnosed as a gateway↔Docker dependency. The real cause: Docker (or any background process) was keeping the WSL VM alive. When Docker was closed, WSL had no remaining foreground processes and shut down the VM, taking all gateways with it.

**Symptoms:**
- Gateway dies exactly when you close the terminal / stop Docker Desktop
- Logs show `systemd-exit.service - Exit the Session` followed by all services stopping
- `signal SIGTERM received` → `received SIGTERM; shutting down` in gateway logs
- Gateway restarts automatically when a new terminal is opened (WSL boots again)

**Diagnosis:**
```bash
# Check if WSL session exit caused the restart
journalctl --user --since "10 min ago" | grep "systemd-exit"
# If you see "Finished systemd-exit.service - Exit the Session" — this is the cause
```

**Fix:** Create a keepalive systemd service that prevents WSL from collapsing:
```ini
# ~/.config/systemd/user/wsl-keepalive.service
[Unit]
Description=Keep WSL alive
After=default.target

[Service]
ExecStart=/bin/sleep infinity
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```
```bash
systemctl --user daemon-reload
systemctl --user enable --now wsl-keepalive.service
```

**Note:** `vmIdleTimeout=-1` in `.wslconfig` may also help but does not work reliably in all WSL versions. The keepalive service is the reliable solution.

### Fixing Bonjour Name Conflicts & Profile Misidentification
When running multiple gateway profiles on the same host, they may conflict on the network discovery name or hostname, leading to erratic restarts or auto-renaming in logs.

**Symptoms:**
- Logs show `[bonjour] gateway name conflict resolved; newName="..."`.
- Discovery names have auto-incrementing numbers like `OpenClaw (2)`.
- Gateways get stuck in `probing` state in logs.

**Fix:**
1.  **Assign Unique Names & Hostnames**: In the systemd service unit file (e.g., `~/.config/systemd/user/openclaw-gateway-lena.service`), add these environment variables:
    ```ini
    Environment="OPENCLAW_MACHINE_NAME=OpenClaw: ProfileName"
    Environment=OPENCLAW_MDNS_HOSTNAME=openclaw-profilename
    ```
2.  **Apply**: `systemctl --user daemon-reload && systemctl --user restart openclaw-gateway-<profile>`.

*Note: The machine name override requires OpenClaw version with `OPENCLAW_MACHINE_NAME` support (added in current fix).*

### Fixing Cross-Profile Auth Credential Leaks
When multiple gateway profiles are configured, auth credentials from one profile can leak into another's `auth-profiles.json`. This causes the gateway to authenticate with the wrong Google account, leading to persistent timeouts and the profile appearing to "disconnect" periodically.

**Symptoms:**
- Profile periodically stops responding (Telegram bot goes silent for 1-2 minutes)
- Logs show `Profile <wrong-email> timed out. Trying next account...`
- `embedded_run_failover_decision` with `failoverReason: timeout` and `status: 408`
- Both primary and fallback models time out in sequence
- The email in the timeout message does NOT match the profile's intended account in `openclaw.json` → `auth.profiles`

**Diagnosis:**
1. **Identify the intended account** from `openclaw.json`:
   ```bash
   openclaw --profile <name> config get auth.profiles
   ```
2. **Check which account the agent actually uses**:
   ```bash
   openclaw --profile <name> models auth order get --agent main --provider google-gemini-cli
   ```
   If the order points to a different email than `openclaw.json` — this is the problem.
3. **Scan for foreign accounts** in the auth-profiles file:
   ```bash
   grep '"email"' ~/.openclaw-<profile>/agents/main/agent/auth-profiles.json
   ```
   Any email that doesn't belong to this profile should be removed.
4. **Check session overrides**:
   ```bash
   grep 'authProfileOverride' ~/.openclaw-<profile>/agents/main/sessions/sessions.json
   ```

**Fix:**
1. In `~/.openclaw-<profile>/agents/main/agent/auth-profiles.json`:
   - Remove foreign account entries from `profiles` section
   - Update `order` to point to the correct account
   - Update `lastGood` to point to the correct account
   - Remove foreign entries from `usageStats`
2. Fix session overrides if present:
   - Replace wrong email in `authProfileOverride` entries in `sessions.json`
3. Restart the gateway:
   ```bash
   openclaw --profile <name> daemon restart
   ```

**Prevention:** After any `models auth login` operation, check all profile auth-profiles files to ensure credentials were not written to the wrong profile.

## Reference Material
For deep technical details on specific bugs (e.g., the systemd parser backslash bug) and architectural changes, see [references/technical-notes.md](references/technical-notes.md).

### Architectural Highlights (from Documentation)
*   **Memory Structure**: Uses Markdown files (`MEMORY.md`, `SOUL.md`). Issues with agent "forgetfulness" often relate to the **Compaction** (archive) process.
*   **Sandboxing**: Shell commands and file operations are executed in **Docker containers**. If a tool fails with "file not found" but the file exists on the host, check the sandbox mounting.
*   **Proactive Tasks**: Uses `Heartbeats` (periodic wake-ups) and `Cron`. If scheduled tasks don't run, check the Gateway's background process state.
*   **Multi-Agent Routing**: A single gateway can handle multiple profiles. Always verify the `--profile` flag when running diagnostic commands.

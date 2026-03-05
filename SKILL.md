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

## Reference Material
For deep technical details on specific bugs (e.g., the systemd parser backslash bug) and architectural changes, see [references/technical-notes.md](references/technical-notes.md).

### Architectural Highlights (from Documentation)
*   **Memory Structure**: Uses Markdown files (`MEMORY.md`, `SOUL.md`). Issues with agent "forgetfulness" often relate to the **Compaction** (archive) process.
*   **Sandboxing**: Shell commands and file operations are executed in **Docker containers**. If a tool fails with "file not found" but the file exists on the host, check the sandbox mounting.
*   **Proactive Tasks**: Uses `Heartbeats` (periodic wake-ups) and `Cron`. If scheduled tasks don't run, check the Gateway's background process state.
*   **Multi-Agent Routing**: A single gateway can handle multiple profiles. Always verify the `--profile` flag when running diagnostic commands.

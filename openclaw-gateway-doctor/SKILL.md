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

### Manual Token Check
To manually inspect the token a running gateway is using:
```bash
grep -a OPENCLAW_GATEWAY_TOKEN /proc/$(pgrep -f openclaw-gateway)/environ | tr '\0' '\n'
```

## Reference Material
For deep technical details on specific bugs (e.g., the systemd parser backslash bug) and architectural changes, see [references/technical-notes.md](references/technical-notes.md).

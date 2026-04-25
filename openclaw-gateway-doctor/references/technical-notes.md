# Technical Notes for OpenClaw Gateway Troubleshooting

## 1. Entrypoint Evolution
The OpenClaw gateway entrypoint transitioned from `dist/entry.js` to `dist/index.js` in version 2026.2.26.
*   **Symptom**: `doctor` reports "Gateway service entrypoint does not match the current install".
*   **Fix**: Reinstall the service with `gateway install --force`. This updates the `ExecStart` path in the systemd unit file.

## 2. Systemd Parser Bug (Fixed)
An internal bug in `src/daemon/systemd-unit.ts` prevented the parser from correctly handling escaped backslashes in `Environment=` lines.
*   **Buggy Code**: `if (ch === "")` (comparing a single character `ch` to a 2-character string).
*   **Fixed Code**: `if (ch === "")`.
*   **Impact**: When this bug is present, `doctor` cannot see the `OPENCLAW_GATEWAY_TOKEN` in existing service files, leading to false "missing token" reports.

## 3. Profile Isolation
Multiple gateways can run on the same host using different profiles.
*   **Default**: `~/.openclaw` (Port 18789)
*   **Named Profile**: `~/.openclaw-<profile>` (Different port, e.g., 19790)
*   **Issue**: Running `doctor` without the `--profile` flag will check the default state, which might not be what the user is currently using.

## 4. mDNS / Bonjour Name Resolution
OpenClaw uses the `@homebridge/ciao` library for mDNS discovery.
*   **Instance Name**: Defaults to `${machineDisplayName} (OpenClaw)`.
*   **Machine Name Override**: Can be overridden via `OPENCLAW_MACHINE_NAME` environment variable.
*   **Hostname**: Defaults to `openclaw.local`.
*   **Hostname Override**: Can be overridden via `OPENCLAW_MDNS_HOSTNAME` environment variable.
*   **Conflict Handling**: If two gateways on the same network (or the same host) use the same instance name or hostname, `ciao` will automatically append a number (e.g., ` (2)`) to resolve the conflict.
*   **Stability**: On some systems (especially WSL2), multiple gateways sharing the same mDNS UDP port (5353) may get stuck in the `probing` state. Unique `OPENCLAW_MDNS_HOSTNAME` values are critical to minimize these collisions.

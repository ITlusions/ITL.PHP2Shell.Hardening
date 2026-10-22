# ITL.PHP2Shell.Hardening

Cluster-wide compensating controls for **CVE-2026-109618** and **CVE-2026-109619** — two critical (CVSS 4.0: 9.2) PHP remote code execution vulnerabilities disclosed 2026-10-21.

This is **not a patch**. The only complete fix is upgrading PHP to 8.5.12, 8.4.27, 8.3.36, or 8.2.35. This chart implements the official PHP security team's documented mitigations as a Helm chart + Kyverno enforcement pair, for use while an upgrade is scheduled, or as defense-in-depth afterward.

## Background

- **CVE-2026-109618** — RCE via OOM corruption. Out-of-memory handling can interrupt PHP while internal request data is only partially initialized; request cleanup may then treat the invalid data as valid.
- **CVE-2026-109619** — RCE via stale global state. Request-owned internal state can survive after its backing memory has been freed; a later request handled by the same worker may use that stale state before normal initialization resets it.

All PHP installations serving web requests are at risk (PHP-FPM, Apache mod_php, php-cgi, FrankenPHP, LiteSpeed). **mod_php carries the highest risk**: a demonstrated exploit path lets an unauthenticated request trigger the vulnerability before any application code runs.

Official advisory: https://www.php.net/archive/2026.php#2026-10-21-5

## What this chart does

1. **`configmap.yaml`** — ships a hardened FPM pool config (`php_admin_value` settings, which applications cannot override via `ini_set()` or `.user.ini`) implementing the PHP security team's documented mitigations:
   - `memory_limit=-1` — removes the finite memory limit that triggers the vulnerable OOM path for CVE-2026-109618 (counterintuitive, but per the advisory: a finite limit is the trigger, not the protection)
   - Tight `post_max_size` / `upload_max_filesize`
   - `session.upload_progress.enabled=0` — blocks the confirmed session-state path for CVE-2026-109619
   - `fatal_error_backtraces=0` (PHP 8.5) — blocks the confirmed fatal-backtrace path for CVE-2026-109619

2. **`kyverno-mutate.yaml`** — automatically injects the hardening ConfigMap mount into any pod whose image matches a PHP pattern, so the mitigation is self-healing even if a deployment's chart forgets to mount it.

3. **`kyverno-validate.yaml`** — backstop policy that rejects (or warns, depending on `enforcement.mode`) any matching pod that doesn't have the hardening mount, closing the gap if the mutate rule is ever bypassed.

## What this chart does NOT do

- It does not patch the PHP binary itself. Upgrade PHP as soon as possible.
- The mitigations reduce exposure to the *specific documented attack paths* only — the advisory is explicit that other variants may exist.
- `session.upload_progress.enabled=0` and `fatal_error_backtraces=0` require a full worker restart to take effect; `ini_set()` does not work for either.

## Usage

```bash
helm install php-security-baseline . -f values.yaml
```

Requires [Kyverno](https://kyverno.io/) installed in the target cluster for the enforcement policies to take effect. Without Kyverno, only the ConfigMap is created — nothing enforces that PHP workloads actually mount it.

### Configuration

| Value | Default | Description |
|---|---|---|
| `phpHardening.memoryLimit` | `-1` | PHP `memory_limit` admin value |
| `phpHardening.postMaxSize` | `32M` | PHP `post_max_size` admin value |
| `phpHardening.uploadMaxFilesize` | `16M` | PHP `upload_max_filesize` admin value |
| `phpHardening.sessionUploadProgressEnabled` | `false` | Whether to allow `session.upload_progress` (leave `false` per CVE-2026-109619 guidance) |
| `phpHardening.fatalErrorBacktraces` | `false` | PHP 8.5 `fatal_error_backtraces` setting |
| `enforcement.phpImagePattern` | `*php*` | Image name pattern used to identify PHP workloads |
| `enforcement.mode` | `enforce` | Kyverno validation failure action: `enforce` (block) or `audit` (warn only) |

## Rollout recommendation

Deploy with `enforcement.mode: audit` first to confirm the policies match your actual PHP workloads without unexpected rejections, then switch to `enforce`.

## Status

Chart structure has not yet been tested against a live Kyverno installation or a real PHP workload. Validate in a non-production cluster before relying on it.

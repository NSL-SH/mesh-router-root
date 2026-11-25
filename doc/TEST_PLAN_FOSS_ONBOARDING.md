# Test plan — FOSS onboarding (`install.sh`)

Manual test of the first-install path a self-hoster follows: an nsl.sh account and a blank
VPS, nothing else. Scope is `mesh-router-template-root/install.sh` and the self-check it hands
off to, as served by `https://nsl.sh/dashboard/install.sh`.

## Inputs

| Input | What it gives |
|---|---|
| An nsl.sh account | A domain (`<name>.nsl.sh`) and the install command the dashboard prints: `--provider "https://nsl.sh,<userid>,<signature>" --domain "<name>.nsl.sh"` |
| A fresh Debian 13 VPS | Root over SSH, a public IPv4, no Docker |

Throughout: `D=<name>.nsl.sh`, `N=<name>`, `IP=<vps ipv4>`, `M=/DATA/AppData/mesh`.

Out of scope: Windows (`install.ps1`), `--local`, migration between boxes, the managed
(Yundera template-root) path, store apps beyond Maison itself.

## 0. Preconditions

| # | Check | Expected |
|---|---|---|
| 0.1 | `cat /etc/os-release` | Debian 13 (trixie) |
| 0.2 | `command -v docker`; `ls /DATA` | Neither exists — a genuinely blank box |
| 0.3 | `curl -s https://nsl.sh/router/api/resolve/v2/$N` | Domain known to the backend, no routes yet (or stale ones) |
| 0.4 | `curl -fsSL https://nsl.sh/dashboard/install.sh` vs `mesh-router-template-root/install.sh` | Same script (ignoring line endings) — we test what users get |

## 1. Argument validation (before anything is written)

Each must exit non-zero with a readable message, and leave `/DATA` absent.

| # | Command | Expected |
|---|---|---|
| 1.1 | `bash install.sh` (no args, no install) | "no existing installation found … needs --provider and --domain" |
| 1.2 | `--provider "https://nsl.sh,<uid>,<SIGNATURE>" --domain $D` | Rejected; hint about a literal placeholder; signature not printed in full |
| 1.3 | `--provider "https://nsl.sh,<uid>" --domain $D` | "Expected 3 comma-separated fields" |
| 1.4 | `--provider "nsl.sh,<uid>,<sig>" --domain $D` | "must start with http:// or https://" |
| 1.5 | `--provider <valid> --domain "https://$D/"` | Domain rejected |
| 1.6 | `--provider <valid> --domain` (trailing flag) | "--domain requires a value" |
| 1.7 | Run as a non-root user | "must run as root" |
| 1.8 | After 1.1–1.7: `ls /DATA` | Still absent |

## 2. First install (the documented command)

Run exactly what the dashboard prints, piped from curl, in an interactive SSH session
(`ssh -t`), answering the claim prompts with a chosen username/password.

| # | Check | Expected |
|---|---|---|
| 2.1 | Installer output | Per-step checklist from `self-check.sh --display`, every step OK, ends with `=== Installation complete ===` and exit 0 |
| 2.2 | Claim prompt | Asks username (default `admin`) and password twice on the terminal despite `curl \| bash`; password not echoed |
| 2.3 | Final summary | Shows `https://$D`, the login name, update instructions, the usage-reporting notice |
| 2.4 | Duration | Recorded (blank box → complete). Note anything that looked hung |
| 2.5 | Provider signature | Never printed in full anywhere in the output |

## 3. On-box state

| # | Check | Expected |
|---|---|---|
| 3.1 | `ls $M` | `docker-compose.yml .env Caddyfile template/ scripts/ log/ data/ migration-markers/` |
| 3.2 | `stat -c '%a' $M/.env`; key lines | Mode 600; `PROVIDER_STR`, `DOMAIN`, `UPDATE_URL` (stable tarball), `MESH_AUTO_UPDATE=true`, `PUBLIC_IP=$IP`, `IP_ZONE`, `LOCAL_ADMIN_USER` |
| 3.3 | `docker ps --format '{{.Names}} {{.Status}}'` | mesh (tunnel, agent, caddy, smtp, mesh-console), auth (dex, authelia, auth-registrar, auth-console), maison (gate + app) — all Up, none restarting, healthchecks healthy |
| 3.4 | `docker network inspect pcs` | Exists, all platform containers attached |
| 3.5 | `crontab -l` | Nightly self-check entry (`0 3 * * *`) |
| 3.6 | `ls /etc/logrotate.d/mesh-router` | Present |
| 3.7 | `stat -c '%a' /DATA/AppData/auth/.stack.env` | 600, holds `DEX_SESSION_KEY`, `AUTHELIA_DEX_SECRET`, `AUTH_CONSOLE_ASSERTION_SECRET` |
| 3.8 | `$M/scripts/tools/authelia-user-manager.sh list` | Exactly one enabled user, the chosen name |
| 3.9 | `grep -ciE 'error\|fail' $M/log/mesh.log` | Reviewed; no unexplained failures |

## 4. Routing from the outside

Run from a machine other than the VPS.

| # | Check | Expected |
|---|---|---|
| 4.1 | `curl -s https://nsl.sh/router/api/resolve/v2/$N` | Agent route (`<ip-dashed>.ip.nsl.sh`, ip `$IP`, priority 1) and tunnel route (priority 2), fresh TTL |
| 4.2 | `curl -sI -H 'X-Mesh-Trace: 1' https://$D/` | Valid TLS, 2xx/3xx, `x-mesh-route` ends with `pcs` (agent path) |
| 4.3 | Same with `-H 'X-Mesh-Force: tunnel'` | `x-mesh-route: tunnel…,pcs` — the tunnel path works too |
| 4.4 | `https://maison-$N.nsl.sh/`, `https://mesh-console-$N.nsl.sh/`, `https://auth-$N.nsl.sh/.well-known/openid-configuration` | Each reachable with valid TLS; unauthenticated app URLs redirect to sign-in; Dex discovery JSON names issuer `https://auth-$N.nsl.sh` |
| 4.5 | `dig +short $D`, `dig +short maison.$D` | Resolve (gateway or box, per the zone setup) |
| 4.6 | `curl -sI https://<ip-dashed>.sslip.io/` | Box answers directly (Let's Encrypt cert may lag on first install — note it) |

## 5. Onboarding as the owner (browser)

| # | Check | Expected |
|---|---|---|
| 5.1 | Open `https://$D` | Redirected to the box's login (Dex → Local Account / Authelia), branded with the domain |
| 5.2 | Wrong password | Rejected, clear error |
| 5.3 | Chosen credentials | Logged in, lands on Maison dashboard |
| 5.4 | Open Mesh Console from Maison | SSO — no second login; Overview shows domain, routes, usage reporting state |
| 5.5 | Install one small store app from Maison (e.g. a static app) | Installs, gets `<app>.$D` / `<app>-$N.nsl.sh`, opens over HTTPS behind SSO |
| 5.6 | Log out | Session ends; app URLs ask for login again |

## 6. Idempotency and update

| # | Check | Expected |
|---|---|---|
| 6.1 | `bash $M/template/install.sh` (no args) on a tty | Prints found config (signature redacted, "Claimed: yes (<user>)"), one `[Y/n]`; answering `n` changes nothing |
| 6.2 | Same, answer `Y` | Completes with exit 0, no claim prompt, no outage (`curl https://$D` keeps answering during the run), auth secrets in `.stack.env` unchanged (compare sha256 before/after), still logged in |
| 6.3 | Re-run the original dashboard command (same provider/domain) | Goes straight through, no teardown (identity unchanged), exit 0, no claim prompt |
| 6.4 | `bash $M/scripts/self-check.sh --display` | All OK (this is what cron runs nightly) |
| 6.5 | `reboot`, wait, then 3.3 + 4.2 | Stack back up on its own, domain reachable |

## 7. Uninstall and clean reinstall

| # | Check | Expected |
|---|---|---|
| 7.1 | `curl -fsSL https://nsl.sh/dashboard/uninstall.sh \| bash -s -- --yes` | Stacks + volumes removed, cron entry and logrotate file gone, `$M/.env` gone |
| 7.2 | `docker ps -a`; `crontab -l` | No platform containers; no self-check entry |
| 7.3 | Re-run the dashboard install with `--yes` (non-interactive, no credentials) | Completes; summary says **NOT CLAIMED YET** with the claim command; Dex not running; `https://$D` sign-in page says sign-in unavailable |
| 7.4 | `printf '%s' '<pw>' \| $M/scripts/tools/authelia-user-manager.sh claim <user>` | Account enabled, Dex starts, browser login (5.1–5.3) works |
| 7.5 | Run `claim` again | Refused without `--force` |

## Pass criteria

All of 1–4 and 6.1–6.4 pass; 5 passes in a browser; 7 leaves a box that can be reinstalled
and claimed. Any step needing a manual workaround is a failure, with the workaround noted.

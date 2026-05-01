# Changelog

All notable changes to VPN Manager are documented here.

---

## v1.4.91 — 2026-05-01

### Added

- **Auto-poll for new versions in the background.** A periodic check (default every 6 hours, controlled by `UPDATE_CHECK_INTERVAL`) refreshes the manifest cache. The admin panel's navbar now shows a small package-up icon with a red dot when a newer version is available — click it to jump to Updates. No auto-apply: you stay in control of when to install.
- **`publish_update.py --to-both` flag** for vendors operating primary + backup license servers. One command publishes / promotes / lists / deletes across both, surfacing per-server failures at the end without aborting halfway. Defaults: `flirexa.biz` (primary) + `global-connection.site` (backup).

### Fixed

- **VPN interfaces no longer dropped during update.** `update_apply.sh` now snapshots active `wg` and `awg` interfaces before stopping services and restores any that didn't come back up. Previously, manually-started or orphan interfaces (e.g. an `awg-quick@awg1` that was never `systemctl enabled`) could disappear after a service restart cycle. Real customers' connections survive untouched now.
- **`vpnmanager license status` now reads the right `.env`.** In release-layout installs (`/opt/vpnmanager/releases/<ver>/`) the CLI was looking for `.env` next to the source, but the persistent `.env` lives at the install root one level up. Symptom: CLI reported `not_activated` while the API correctly showed an active enterprise license. CLI now walks up the directory tree to find `.env`.

---

## v1.4.89-1.4.90 — 2026-05-01

### Added

- **"Re-fetch License" button in Settings → License.** Pairs with the activation replay endpoint (1.4.88). If your license key was lost during the original activation (network blip, lost terminal, parsing error) you can now re-enter your activation code from the panel and recover the same key. Hardware-bound — only works from the original install. No support ticket needed.
- **Translations** for the Re-fetch UI in English, Russian, Spanish, French, German.

---

## v1.4.88 — 2026-05-01

### Added

- **Activation replay endpoint** (`POST /api/activate/replay`) on the license server. Re-issues the original signed license payload (same plan, expiry, hardware binding) with a fresh signature, for the recovery case where `/api/activate` succeeded but the key didn't land in `.env`. Per-code rate limit: 3 attempts per 24 hours. Hardware mismatch returns 403.

### Fixed

- **`install.sh` activation prompt now works under `curl … | bash`.** The headline install command (`curl -fsSL https://flirexa.biz/install.sh | bash`) silently skipped the prompt because bash's stdin was the curl pipe. Paid customers couldn't enter their activation code that way unless they pre-set `SB_ACTIVATION_CODE`. The prompt now reads from `/dev/tty` explicitly. Empty input, `n`, `no`, `free`, or `skip` selects FREE; anything else is treated as an activation code.

---

## v1.4.87 — 2026-04-30

### Fixed

- **`vpnmanager` CLI wrapper finds the venv in release-layout installs.** After resolving the symlink chain to `/opt/vpnmanager/releases/<ver>/vpnmanager`, the wrapper expected `venv/` next to the script, but venv lives at `/opt/vpnmanager/venv` (parallel to `releases/`). Falling back to system `python3` produced `ModuleNotFoundError: No module named 'dotenv'`. The wrapper now walks up from script dir up to 3 levels looking for `venv/bin/python3`.

---

## v1.4.86 — 2026-04-30

### Fixed

- **Bot services stop looping when the token is missing or `*_BOT_ENABLED=false`.** Previously, `vpnmanager-admin-bot` and `vpnmanager-client-bot` would crash-loop several hundred times per hour — admin-bot exited with status=1 on missing token, client-bot exited cleanly but the unit had `Restart=always` so systemd restarted it anyway. The bots now `sys.exit(0)` cleanly on missing/disabled config, and the units use `Restart=on-failure` — no restart cycle, no CPU waste, no journal spam.

---

## v1.4.85 — 2026-04-30

### Added

- **Dual-format QR code for AmneziaWG clients.** The client view now shows two QRs side by side: a plain `.conf` for the AmneziaWG simple app, and a `vpn://` share URL for the AmneziaVPN mobile app. Same peer, different formats — pick whichever matches your client.
- **Trial vs paid grace period.** Short licenses (≤14 days, e.g. trials) now have no grace period — when they expire, the system goes degraded immediately. Paid licenses keep the standard 72-hour grace window for offline clock skew.

### Fixed

- **License server URL defaults to `https://flirexa.biz`** in fresh builds (was `example.com` placeholder). Operators running self-hosted license servers still override via `.env`.

---

## v1.4.70 — 2026-04-29

### Fixed

- **AmneziaWG server wouldn't start ("Failed to start server" / 500)** — four stacked issues, all hit on a fresh FREE install once you actually try to bring an AmneziaWG interface up alongside the auto-provisioned WireGuard:
  1. **Wrong config path.** apt-installed `awg-quick` from `ppa:amnezia/ppa` looks for the config at `/etc/amnezia/amneziawg/<iface>.conf` (note the extra `amnezia/` segment), not `/etc/amneziawg/<iface>.conf`. The codebase wrote everywhere to the latter so awg-quick reported "config does not exist". Replaced `/etc/amneziawg` → `/etc/amnezia/amneziawg` across `servers.py`, `server_manager.py`, `agent_bootstrap.py`, `backup_manager.py`, and the `AmneziaWGManager` constructor default.
  2. **Config file was never written.** `start_server()` called `wg.start_interface()` directly without writing the config to disk. WireGuard got away with it because `install.sh` writes `/etc/wireguard/wg0.conf` eagerly, but the user-created AmneziaWG only had a DB record. `start_server()` now calls `save_server_config()` first; cheap, idempotent, and picks up new peers since last start.
  3. **Parent dir missing.** `write_config_file()` opened the file directly, which fails on a fresh install where `/etc/amnezia/amneziawg/` doesn't exist yet. Added an `os.makedirs(parent, exist_ok=True)` before write.
  4. **PostUp/PostDown was malformed for dual-stack address.** AmneziaWG's `generate_server_config()` derived the IPv4 subnet by splitting the full address on `/`, but the address comes in as `10.66.66.1/24,fd42:42:42::1/64` (combined IPv4+IPv6). The naive split produced `10.66.66.1/24,fd42:42:42:.0/64`, which both `iptables` and `ip route` rejected, so `awg-quick` rolled back the interface. Now we extract the IPv4 half before parsing.
  5. **Port collision.** AmneziaWG defaulted to listen_port 51820 — same as the auto-provisioned WireGuard — so the kernel rejected the second bind with `RTNETLINK answers: Address already in use`. Local server creation now scans existing ports and walks up from the requested one until it finds a free slot. The drift is logged.

End-to-end verified on a clean VM: install → activate FREE → keep auto-WireGuard → create AmneziaWG → click Start → `awg show` lists the interface with all obfuscation parameters and traffic flows.

### Earlier 1.4.68 / 1.4.69 commits land in this release together; bumping straight to 1.4.70 since none of the intermediates were promoted to stable individually.

---

## v1.4.67 — 2026-04-29

### Fixed

- **Update + license server URL defaults pointed at `https://example.com`.** Fresh installs that didn't explicitly set `UPDATE_SERVER_URL` / `LICENSE_SERVER_URL` in `.env` got `404 No manifest found for channel 'stable'` on update checks and trial registration silently failed. Default to `https://flirexa.biz` (operators running their own license server still override via `.env`).

---

## v1.4.66 — 2026-04-29

### Changed

- **FREE tier server limit is now per-protocol, not total.** Previously the cap was "one server" — with the auto-provisioned WireGuard taking that slot, FREE users couldn't add AmneziaWG without first deleting their working WireGuard. The intent has always been *both* protocols on FREE (DPI-resistance is core value), so the cap moves to "one of each protocol type":
  - FREE: up to 1 WireGuard + 1 AmneziaWG = 2 servers total.
  - Starter (`$19/mo`): adds Hysteria2 + TUIC = up to 4 servers (one of each).
  - Business+ keeps the existing `multi_server` feature, which lifts the cap fully (10 / unlimited).
- Server-create endpoint now counts servers of the same `server_type` instead of all servers. The pg advisory lock is preserved so concurrent requests can't both win.
- License-server `plans` table: `standard.max_servers` 1 → 2.
- Local `LICENSE_TIERS` fallback: `FREE` 1 → 2, `STANDARD` 1 → 4.

---

## v1.4.65 — 2026-04-29

### Fixed

- **Fresh installs were missing AmneziaWG userspace tools.** 1.4.64 unblocked AmneziaWG creation at the license layer, but `install.sh` only ever installed `wireguard-tools` — so on a fresh VPS, creating an AmneziaWG server failed with a `500 Internal Server Error` (`FileNotFoundError: 'awg'`). Existing installs that had been hand-configured (e.g. the maintainer's own production box) worked fine, which masked the bug for everyone but new users.
- Installer now adds the `amnezia/ppa` apt repository and installs `amneziawg`, `amneziawg-tools`, and `amneziawg-dkms` (with the running kernel headers) right after the core package step. The whole AmneziaWG block is best-effort — if the DKMS module fails to compile on a stripped VPS image with no headers, the install still completes and the panel still works in WireGuard-only mode, with a clear log warning.
- Runtime: `core/amneziawg.py` wraps the `awg` subprocess calls; a missing binary now raises a `RuntimeError` carrying the exact apt command to fix it, and the create-server endpoint surfaces that as `400 Bad Request` instead of leaking a generic 500.

---

## v1.4.64 — 2026-04-29

### Fixed

- **AmneziaWG was incorrectly gated as a paid feature.** The server-create endpoint mapped `amneziawg` → license-feature `amneziawg`, which doesn't exist on FREE-tier signed licenses, so any FREE user trying to provision an AmneziaWG server got `403 "AMNEZIAWG protocol requires the 'amneziawg' feature. Upgrade your plan to enable it."` This contradicts both the README and `docs/free-vs-paid.md`, which list AmneziaWG as a core FREE feature — it's the DPI-resistant protocol that makes the product useful on hostile networks.
  - Real impact for FREE users: a fresh install auto-provisions a WireGuard server. Anyone who wanted AmneziaWG instead had to delete the auto-server and create a new one — and the second create was blocked. They were stuck on WireGuard. Now AmneziaWG creation just works.
  - Hysteria2 / TUIC still require the `proxy_protocols` feature (Starter+), unchanged.
- **License-server plans seed**: added `amneziawg` to the standard / pro plan feature lists too, so future-issued paid licenses also include it (was previously absent — paid users were *also* affected, just less visibly because they could pay their way to enterprise which already had it).

---

## v1.4.63 — 2026-04-28

### Security / Fixed

- **License feature gate for traffic-rules was a no-op on FREE installs.** The middleware checked `/api/v1/traffic-rules` but the router is registered at `/api/v1/traffic`, so the `startswith` match never fired and FREE-tier users could call `GET /api/v1/traffic/top`, `/api/v1/traffic/rules`, `/api/v1/traffic/clients` without paying. POST/PUT had inline checks, but DELETE was also open. Confirmed on a FREE VM (license features = wireguard/client_portal/telegram_bots only) — all three endpoints returned 200 before, 403 after. The two prefixes (router and middleware) must match exactly; we added a comment so it doesn't drift again.
- **`LICENSE_CHECK_ENABLED=false` used to silently short-circuit the entire license middleware** — no log, no warning. A typo or a leaked .env could disable activation, expiry, online-validation, *and* feature gating without any signal. Now logs `EVENT:LICENSE_BYPASS` (rate-limited to once per 5 minutes per process) every time the bypass is hit, with an explicit "fix the env file IMMEDIATELY" hint for production.

### Fixed

- **`update_apply.sh` left `$INSTALL_DIR/VERSION` stale in release-layout mode.** Only `$CURRENT_LINK/VERSION` moved when the symlink switched. External monitoring / scripts that read `/opt/vpnmanager/VERSION` (or `/opt/spongebot/VERSION`) saw the previous version forever after the upgrade. Now we write `$TARGET_VERSION` to the install-root file too. Existing installs catch up on the next upgrade.

### Public mirror (`Flirexa/flirexa`) cleanup

- CI workflow was failing on every push: pytest job didn't install runtime requirements (psutil / python-dotenv / aiocryptopay → `ModuleNotFoundError`) and secrets-scan used an invalid `--base-path` flag. Both fixed; CI green again.
- Replaced remaining `spongebot` / `VPN Management Studio` strings with `Flirexa` in the public mirror: `alembic/env.py` default DSN, `.env.example` header, `backup_manager.py` docstring + version stamp.

---

## v1.4.62 — 2026-04-28

### Fixed
- **Plugin URLs returned 404** — generic plugin loader ran in lifespan, which appended plugin routers to `app.routes` *after* the SPA catch-all `GET /{full_path:path}` registered in `create_app()`. FastAPI matches routes in order, so the catch-all swallowed every plugin URL with 404. Loader now runs in `create_app()` right before the SPA mount, so plugin routes win the match. End-to-end verified on a fresh VM install with the `monthly-revenue` demo plugin: install-by-URL → restart → `GET /api/v1/plugins/monthly-revenue/current` returns 200.
- **Loader did not honor `community` feature flag** — manifests declaring `requires_license_feature: "community"` were skipped on FREE installs because the loader required *every* declared feature to be granted by the license. The reserved name `community` is now treated as always-granted, matching what the docs already describe and letting third-party community plugins load on every tier.
- **`curl https://flirexa.biz/install.sh | sudo bash` aborted on non-TTY shells** — the installer started with a bare `clear` under `set -e`, which exits non-zero when `TERM` is unset/unknown (common when piping through SSH or CI). Made `clear` best-effort (`clear 2>/dev/null || true`) so the banner step never kills the install.

---

## v1.4.61 — 2026-04-28

### Added
- **Plugin marketplace (variant 1) — install-by-URL** in admin panel. New endpoints: `GET /api/v1/plugins/installed`, `POST /api/v1/plugins/install` (URL + SHA-256), `DELETE /api/v1/plugins/{name}`. Tarball must contain a single top-level dir matching `manifest.json.name`; max 25 MB; SHA-256 verified before extraction. Restart required to pick up newly-installed routes. Vue admin page lists installed plugins (core vs user-installed) and provides the install / uninstall UI.

### Changed
- **Donate button** moved to leftmost slot of the right-side toolbar group, redesigned as a text+heart pill instead of an icon-only button.
- **Docs**: removed Russia-specific brand examples (Yandex etc.) from `free-vs-paid.md` and adjacent pages; replaced with international equivalents.

---

## v1.4.60 — 2026-04-27

### Added
- **Donate button + reminder modal in admin panel** — heart-icon button always visible in topbar; opens a modal with a "Support on GitHub" CTA linking to the project's crypto donation addresses (BTC / ETH / USDT TRC-20 / USDT ERC-20). The modal auto-shows on first install, then re-shows only after a 7-day cooldown after the user dismisses it. The free tier stays free; donations fund the work, they do not unlock features. Localised across EN / RU / DE / FR / ES.

### Changed
- **Starter tier client cap 500 → 300** in the offline LICENSE_TIERS fallback. Aligns with the new pricing copy on flirexa.biz. Existing customers are unaffected — the cap they get is whatever was in their signed payload, refreshed via /api/validate.

### Notes
- v1.4.59 was published to the test channel, then superseded by v1.4.60 (same changes, plus a contrast fix for the donate modal text). Only v1.4.60 reached stable.

---


## v1.4.43 — 2026-04-17

### Added
- **NOWPayments purchase flow** — full end-to-end payment pipeline: invoice creation, IPN webhook processing, activation code generation, signed download token delivery
- **Post-payment thank-you modal** on landing — auto-detects `?NP_id=` redirect from NOWPayments, polls purchase API, displays activation code (with copy button) and one-time download link
- **Email collection in payment modal** — buyer enters email before redirect to NOWPayments; email stored server-side and resolved by webhook when IPN arrives (no dependency on NOWPayments email collection)
- **Buyer email delivery** — HTML email with activation code, green "Download" button (signed URL, 72h TTL, 5 download attempts), and step-by-step install instructions
- **Vendor sale notification** — always fires on successful payment (independent of buyer email), includes plan, amount, code, NOWPayments ID, and download URL
- **Download token system** — `download_tokens` DB table with signed URL-safe tokens, expiry, use limit, IP tracking; `GET /download/{token}` serves package with counter enforcement
- **Invoice creation proxy** — `POST /api/create-invoice` creates NOWPayments invoice server-side with canonical pricing (prevents client-side price tampering), stores buyer email in `buyer_emails.json`
- **Auto-cleanup of unpaid buyer emails** — entries without payment are automatically purged after 7 days; paid entries marked `paid: true` and kept permanently
- **Multi-currency payment support** — landing page payment modal with 11+ cryptocurrency options (BTC, ETH, USDT-TRC20/ERC20, USDC, LTC, XMR, TON, SOL, BNB, TRX)
- Landing: 6 languages for all new payment/thank-you UI strings (EN, RU, UK, DE, FR, ES)

### Fixed
- **QR code crash on Cyrillic client names** — `_safe_filename()` used `\w` which matches Unicode; replaced with explicit `[a-zA-Z0-9_\-.]` to ensure ASCII-only Content-Disposition headers
- **Vendor notification not firing** — was nested inside `_send_license_email()` which was skipped when buyer email was empty; split into independent `_send_vendor_notification()`

### Changed
- Landing: pricing buttons changed from direct NOWPayments links to dynamic invoice creation via server-side proxy
- Landing: payment modal now collects email, shows loading spinner during invoice creation, validates email format client-side
- License server: refactored SMTP into reusable `_send_mail()` helper; vendor and buyer emails use separate functions
- License server: migrated to flirexa.biz as primary host

---

## v1.4.42 — 2026-04-12

### Added
- **Server display names** — admins can rename servers so clients see friendly names instead of real IPs in the client portal
- **Sortable columns** in Clients table — click any header (name, server, IP, status, traffic, bandwidth, expiry) to sort asc/desc with arrow indicator
- **System Health mode indicator** — banner shows "7/7 OK (Quick)" vs "10/10 OK (Full)" with clickable hint to switch modes
- **Concurrent instance detection** — real-time clone detection within 10-minute window (no IP requirement — catches clones behind same NAT)
- **Clone rejection at validation** — `/api/validate` blocks concurrent instances immediately instead of waiting 7 days
- **Hardware fingerprint hardening** — added DMI UUID, disk serial, RAM size to hardware binding (7 entropy sources total)
- `INTERNAL_LICENSE_MODE` now requires `.dev-mode` marker file (not just env var)
- `install.sh`: license server URL configurable via `SB_LICENSE_SERVER_URL` env var
- `install.sh`: improved network interface detection with multiple fallback methods

### Fixed
- **Backup page white screen** — `$t()` used in `<script setup>` without `useI18n()` import, causing ReferenceError crash
- **Settings page crash** — missing i18n keys (`systemTools`, `limitCheck`, etc.) caused partial render failure
- **Proxy client creation failure** — `CreateClientResponse.ipv4` was non-Optional, proxy clients with `ipv4=None` crashed Pydantic validation
- **Proxy config rollback** — `_apply_proxy_config()` result was silently ignored; client saved to DB even when SSH config application failed
- **Unicode crash on QR/config download** — Cyrillic client names caused `UnicodeEncodeError` in Content-Disposition headers
- **Hysteria2/TUIC configs use domain** — client configs now use domain as connection host when TLS cert exists (not IP), fixing TLS handshake failures
- **License server `/panel/api/sales`** — `NameError: PRICES` undefined variable fixed
- **`portalUsers.never`** i18n key added — was showing raw key string instead of "Никогда"
- `datetime.utcnow()` deprecated calls replaced with `datetime.now(timezone.utc)`
- Rate limit cleanup threshold lowered from 10000 to 1000 IPs
- Bare `except Exception` narrowed to specific types in `_deserialize_permissions()`
- Cross-worker proxy config lock via `pg_advisory_xact_lock`

### Changed
- **Client Portal Dashboard** — UI polish: hero KPI card, subscription details in 2 groups, device list restructured, referral inline copy, mobile responsive
- **Server Monitoring** — complete visual overhaul: 3-level hierarchy (name+status → message → metrics), prominent colored status badges, metrics in CSS grid, actions as fixed-size buttons
- **System Health** — compact banner, quiet status badges when healthy, metrics as plain text (not pills), progress bars thicker (6px)
- **Portal Users table** — zebra rows, username/email hierarchy, tier color badges, filters unified bar with search icon, "Never" for empty last login
- **Subscriptions table** — tier color badges (not red `<code>`), ∞→"Unlim." text, delete button hidden until hover, modal restructured into 5 grouped sections
- **Settings page** — 12 new i18n keys, ~25 hardcoded strings replaced with `$t()`, branding section fully localized
- **Admin panel** — complete i18n for Settings, missing keys added to all 5 locales (en/ru/de/es/fr)
- Removed 109 unnecessary `|| 'fallback'` i18n patterns from client portal
- Removed "spongebot" from client-facing error messages
- License server web panel: dynamic tier filters, license count indicator

---

## v1.4.11 — 2026-04-07

### Added
- `hostname` and `version` fields in every JSON log entry — makes multi-instance log aggregation and version-correlated debugging straightforward
- `GET /api/v1/system/app-logs/errors` — dedicated errors-only endpoint (shortcut for `?errors_only=true`)
- `errors_only` query parameter on `GET /api/v1/system/app-logs`
- **App Logs** admin UI page (`/app-logs`): component tabs (API / Worker / Agent), All / Errors filter, table with time / level / req\_id / method / path / status / ms / message columns, click-to-expand error rows
- `systemApi.getAppLogs()` and `getAppLogsErrors()` added to frontend API client
- Operational event markers for grep-friendly monitoring: `EVENT:API_START/STOP`, `EVENT:WORKER_START/STOP`, `EVENT:BOOTSTRAP_SUCCESS/FAILURE`, `EVENT:UPDATE_START/SUCCESS/FAILURE`, `EVENT:ROLLBACK_START/SUCCESS/FAILURE`, `EVENT:BACKUP_SUCCESS`, `EVENT:RESTORE_SUCCESS/PARTIAL`, `EVENT:LICENSE_BLOCKED`, `EVENT:AGENT_HEALTH_FAILURE`
- `RequestLoggerMiddleware` — single access log entry per request with method / path / status_code / duration_ms bound into loguru context so all log lines for a request share the same fields
- 26 automated tests: request_id propagation, JSON structure validity, hostname/version fields, truncation, empty/broken log file, errors_only filter, secrets protection

### Changed
- `X-Request-ID` header now generated by dedicated middleware (replaces inline lambda); custom header value from caller is honoured and echoed back
- `nav.logs` label changed to "Audit Logs" to distinguish from the new App Logs page

---

## v1.4.6 — 2026-04-05

### Added
- Structured JSON logging across API, worker and agent components: every log line is a JSON object with `timestamp`, `level`, `component`, `message`
- `request_id` propagation — `X-Request-ID` header is assigned per request and bound into every log entry produced during that request via loguru `contextualize()`
- HTTP access log fields in each entry: `method`, `path`, `status_code`, `duration_ms`
- Log size protection: message body capped at 10 KB, error strings capped at 2 KB (both truncated with `[truncated]` marker)
- `GET /api/v1/system/app-logs?component=api|worker|agent&lines=N&errors_only=bool` — tail of structured application logs
- `GET /api/v1/system/app-logs/errors?component=...&lines=N` — errors-only shortcut
- Operational event markers (`EVENT:*`) in log messages for grep-friendly monitoring: `EVENT:API_START`, `EVENT:API_STOP`, `EVENT:WORKER_START`, `EVENT:WORKER_STOP`, `EVENT:BOOTSTRAP_SUCCESS`, `EVENT:BOOTSTRAP_FAILURE`, `EVENT:UPDATE_START`, `EVENT:UPDATE_SUCCESS`, `EVENT:UPDATE_FAILURE`, `EVENT:ROLLBACK_START`, `EVENT:ROLLBACK_SUCCESS`, `EVENT:ROLLBACK_FAILURE`, `EVENT:BACKUP_SUCCESS`, `EVENT:RESTORE_SUCCESS`, `EVENT:RESTORE_PARTIAL`, `EVENT:AGENT_HEALTH_FAILURE`, `EVENT:LICENSE_BLOCKED`
- **App Logs** page in admin panel (`/app-logs`) — component tabs (API / Worker / Agent), All / Errors filter, table with all JSON fields, expandable error rows
- 26 automated tests covering request_id propagation, log endpoints, JSON structure, truncation, secrets protection

### Changed
- Logrotate config at `/etc/logrotate.d/vpnmanager` — daily rotation, 30-day retention, `copytruncate` (no process restart needed)

---

## v1.2.72 — 2026-03-26

### Added
- Business invariant validator: 7 automated checks run every 30 minutes — detects expired clients with active access, completed payments without subscriptions, proxy clients with fake bandwidth limits, and more; auto-fixes violations
- Self-healing state reconciler: ghost peer detection — if a client is disabled in DB but the WireGuard peer still exists on the server, it is automatically removed (access leak prevention)
- Payment pipeline tracing: every payment now has a `trace_id` and a `pipeline_log` recording each step (create → webhook → activate → sync_wg); inconsistent payments are flagged for admin review
- Fail-safe mode: when the system detects critical conditions (invalid license, all WG servers unreachable), new payments are blocked with a clear error message instead of creating broken state
- Worker heartbeat: background worker writes a heartbeat to DB every cycle; health endpoint detects stale/dead worker
- `GET /api/v1/health/full` — comprehensive real-state health check: database, WG servers, license, worker, business invariants → returns OK / DEGRADED / FAIL with problem list
- `GET /api/v1/system/metrics` — operational counters: active clients, expired+enabled (critical flag), payment stats, subscription counts, server drift count
- `GET/POST /api/v1/system/failsafe` — view and manually control fail-safe mode
- Daily health report at 08:00 UTC via Telegram admin notification

### Fixed
- Silent failures in payment and subscription pipeline replaced with structured logging (trace_id, user_id, step name)
- `state_reconciler`: previously only re-added missing peers; now also removes peers that are disabled in DB but still live on the WireGuard interface

### Changed
- `client_portal_payments` table: added `trace_id`, `pipeline_log`, `pipeline_status` columns (migration 016)

---

## v1.2.71 — 2026-03-25

### Fixed
- Proxy clients: traffic and bandwidth columns in Clients table now show `—` instead of fake values
- TC bandwidth limits now enforced immediately after subscription change (not only at next worker cycle)
- `_sync_wg_after_payment` fallback when admin API is not configured — now applies limits directly via DB
- `check_expired_clients`: added SELECT FOR UPDATE (skip locked) to prevent duplicate processing under concurrent worker runs
- Client disabling: DB always updated even if WireGuard peer removal fails
- Duplicate pending payments: old pending payments for the same user/tier are cancelled before creating a new one
- Thread-safe traffic cache reads and writes via `_TRAFFIC_CACHE_LOCK`
- Per-client try/except in `_disable_user_clients` — one failed client no longer blocks the rest
- `is_proxy_client` criterion strengthened: based solely on `public_key is None`

---

## v1.2.46 — 2026-03-24

### Fixed
- White screen when clicking WebAccess radio buttons in Settings — caused by unescaped `@` symbols in vue-i18n locale strings (`admin@example.com`, `@CryptoTestnetBot`) triggering "Invalid linked format" compile error at runtime

---

## v1.2.37 — 2026-03-20

### Fixed
- Missing i18n translation keys for navigation, dashboard charts, server bandwidth, and settings across all 5 locales (EN/RU/DE/ES/FR)

---

## v1.2.36 — 2026-03-20

### Added
- Corporate VPN module: site-to-site WireGuard mesh networks with visual topology map
- Relay/gateway node support for NAT traversal between offices
- Per-peer diagnostics and connection status in corporate networks
- System health monitoring dashboard (10 components: DB, API, worker, license server, WG, bots, payments, disk/mem/cpu)
- Server drift detection: auto-reconcile of DB state vs live WireGuard interface

### Improved
- AmneziaWG full support with obfuscation parameters (Jc/Jmin/Jmax/S1/S2/H1-H4)
- Vuexy UI design system across admin panel and client portal
- Payment module hardening: SELECT FOR UPDATE, atomic promo usage, IPN secret enforcement
- Multi-language support: 6 languages (EN, RU, UK, DE, FR, ES)

---

## v1.2.35 — 2026-03-10

### Added
- White-label branding: name, logo, colors configurable from admin panel
- Update mechanism with rollback support
- Backup and restore: scheduled database + config backups

### Fixed
- License validation grace period (72h offline tolerance)
- Client portal dashboard layout fixes

---

## v1.2.0 — 2026-02-15

### Added
- Plan-based licensing model (Standard / Pro / Enterprise)
- RSA-signed license keys with hardware binding
- Online license validator with heartbeat
- Client portal: user self-registration, subscription plans, crypto payments
- Telegram client bot for end-user self-service

---

## v1.1.0 — 2026-01-20

### Added
- Multi-server management via SSH and lightweight HTTP agent
- Per-client traffic counters and bandwidth limits (`tc`-based shaping)
- Promo codes, referral system, revenue analytics
- CryptoPay (NOWPayments) payment integration

---

## v1.0.0 — 2025-12-01

### Initial release
- Web admin panel (Vue 3 + Bootstrap 5)
- WireGuard server management
- Client CRUD with QR code and config export
- Telegram admin bot
- PostgreSQL backend with Alembic migrations
- Automated install script for Ubuntu/Debian

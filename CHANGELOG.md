# Changelog

All notable changes to VPN Manager are documented here.

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

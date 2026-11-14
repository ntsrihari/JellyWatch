# Nexus — Architecture

Status: **draft for approval.** Nothing here is implemented.
Tags used throughout:
- **[SPEC]** — required by the Knowhere Platform Plan or the implementation brief.
- **[REC]** — an engineering recommendation awaiting approval.
- **[OPEN]** — an unresolved decision; see DECISIONS.md.

## 1. Scope

Nexus is the dashboard and API at `knowhere/` **[SPEC]**. It is the one component that:
- holds the downstream service credentials **[SPEC S-05]**;
- reaches Docker, through a restricted socket proxy **[SPEC S-07]**;
- answers on behalf of the services, so client devices never call them directly **[SPEC S-08]**.

Out of scope here: Collector, Milano and Prism (separate repos), the Caddy and Tailscale setup, and any change to knowhere's existing services.

## 2. Requirements extracted from the plan

### 2.1 Functional
| ID | Requirement | Phase | Source |
|---|---|---|---|
| F-01 | Sign in with a Jellyfin username and password | 1 | Plan §Identity |
| F-02 | Sign in with Jellyfin Quick Connect | 1 | Plan §Identity |
| F-03 | Jellyfin admin maps to Nexus admin; everyone else is a user | 1 | Plan §Identity |
| F-04 | Home mode: an app icon grid plus live widgets; works on phones and TVs | 1 | Plan §Dashboard |
| F-05 | System widget: CPU, RAM, load, temperature, uptime | 1 | Plan §Dashboard, §Roadmap |
| F-06 | Storage widget, per mount: mounted or not, read-only or read-write, free space, SMART health | 1 (SMART depends on C-04) | Plan §Dashboard |
| F-07 | Services widget: up or down, restart count and memory for each container | 1 | Plan §Dashboard |
| F-08 | Now Playing widget: active Jellyfin streams, direct play or transcode | 1–3 (not named in the Phase 1 list) | Plan §Dashboard |
| F-09 | Photos widget: Immich upload count and library size | not phased | Plan §Dashboard |
| F-10 | DNS widget: Pi-hole queries and percent blocked | not phased | Plan §Dashboard |
| F-11 | Downloads widget: qBittorrent queue and speeds, once it exists | 4 | Plan §Dashboard, §Roadmap |
| F-12 | Upcoming widget: Radarr and Sonarr calendar, once they exist | 4 | Plan §Dashboard, §Roadmap |
| F-13 | Admin: restart, stop or update a container | 3 for Milano; not phased for Nexus | Plan §Dashboard, §Milano |
| F-14 | Admin: run a Jellyfin scheduled task | not phased | Plan §Dashboard |
| F-15 | Admin: re-check mounts | not phased | Plan §Dashboard |
| F-16 | Admin: restart or shut down knowhere, with typed confirmation | 3 (Milano server control) | Plan §Dashboard, §Milano |
| F-17 | Guardrail: refuse to start a write-heavy app while its mount is missing or read-only | with F-13 | Plan §Dashboard |
| F-18 | Guardrail: never show `/mnt/internal-hdd` as usable | 1 | Plan §Dashboard |
| F-19 | Guardrail: no format or delete-volume action exists | always | Plan §Dashboard |
| F-20 | Household default theme, set by an admin | 1 | Plan §Theming, §Roadmap |
| F-21 | Per-user theme stored against the Jellyfin user id and synced to devices at login | 3 | Plan §Theming, §Roadmap |
| F-22 | Theme packs installed through Collector become selectable | 5 | Plan §Theming, §Roadmap |
| F-23 | Desktop mode: wallpaper, dock, draggable and resizable windows, layout saved per user; the store is a window | 5 | Plan §Dashboard, §Roadmap |
| F-24 | Milano gets server control and \*arr features through Nexus (health, mounts, containers, restart a service, power) | 3–4 | Plan §Milano |
| F-25 | Milano exchanges its Jellyfin token for a Nexus token | 3 | Plan §Identity |
| F-26 | Collector trusts Nexus sessions | 2 | Plan §Identity |
| F-27 | Requests (Jellyseerr) for everyone; Radarr, Sonarr and qBittorrent queues for admins, through Nexus | 4 | Plan §Milano |

### 2.2 Non-functional
| ID | Requirement | Source |
|---|---|---|
| NF-01 | Go backend, a single binary, about 30–60 MB RAM | Plan; Brief |
| NF-02 | SvelteKit compiled to static files | Plan; Brief |
| NF-03 | Tailscale-only access | Plan; Brief |
| NF-04 | Docker only through a restricted docker-socket-proxy | Plan; Brief |
| NF-05 | No API keys in apps or repos | Plan |
| NF-06 | Built by GitHub Actions; image published to GHCR; knowhere pulls it, never builds | Plan §Build |

## 3. System context

```
 Client devices (tailnet)                knowhere
 ┌──────────────┐        ┌───────────────────────────────────────────────────────┐
 │ Browser      │──HTTPS─▶ Tailscale Serve ─▶ Caddy ─▶ Nexus (Go)               │
 │ Milano phone │        │   [infra, not Nexus]       │  ├─▶ Jellyfin API         │
 │ Milano TV    │        │                            │  ├─▶ Immich, Pi-hole APIs  │
 └──────────────┘        │                            │  ├─▶ *arr, qBit (Phase 4) │
                         │                            │  ├─▶ docker-socket-proxy ─▶ /var/run/docker.sock
                         │                            │  └─▶ host metrics [OPEN C-04]
                         │ Collector ──introspect──▶ Nexus            [OPEN C-05]
                         └───────────────────────────────────────────────────────┘
```

Trust boundaries:
1. Tailnet to Caddy (TLS ends at Tailscale Serve or Caddy).
2. Caddy to Nexus (private network).
3. Nexus to the downstream services (Nexus holds the credentials).
4. Nexus to the Docker proxy (the boundary that protects root).
5. Nexus to the host (C-04).

## 4. Backend design [REC unless marked]

### 4.1 Layout
```
cmd/nexus/                 main: load config, wire dependencies, start the server
internal/config/           config.yaml (non-secret) + secrets loader (P-07)
internal/httpapi/          router, middleware (auth, CSRF/Origin, rate limit, security headers), handlers
internal/auth/             Jellyfin login, Quick Connect, token exchange, sessions, role refresh
internal/authz/            role checks; the central permission table for every route
internal/store/            SQLite: sessions, prefs, layouts, themes, audit (P-02)
internal/integrations/     one client package per service, built only from config base URLs
    jellyfin/ immich/ pihole/ qbittorrent/ radarr/ sonarr/ prowlarr/ jellyseerr/
internal/docker/           client for the socket proxy; container allowlist; no raw passthrough
internal/host/             metrics reader (/proc, /sys); later, the host-agent client [OPEN C-04]
internal/storage/          mount inspection, internal-hdd exclusion, write-heavy guardrail
internal/widgets/          builds widget DTOs, caches them, publishes to SSE
internal/events/           SSE hub
internal/theme/            Prism theme JSON validation, precedence resolution (S-15)
internal/audit/            append-only audit writer
web/                       SvelteKit app (adapter-static), embedded with embed.FS
```

### 4.2 Request pipeline
`recover → request id → security headers → Tailscale/Origin checks (P-09, D-08) → session/bearer auth → authz table → rate limit (auth and admin routes) → handler → audit (admin routes)`.

The authz table is the single source of truth. A route missing from it is denied by default, and a startup check fails the build or boot when any route lacks an entry.

### 4.3 Widgets
- Each widget has a collector goroutine that polls its source on an interval, caches the last result and its timestamp, and publishes to SSE.
- Clients get cached data. A slow or down service shows as `stale` or `unavailable`; it never blocks the request.
- Widgets for services that aren't configured return `{"status":"not_configured"}`. This covers F-11 and F-12 ("once it exists").
- Per-role visibility of widgets is [OPEN D-01].

### 4.4 Docker
- Nexus talks HTTP to the socket proxy, never to the socket.
- Phase 1 uses read calls only: list containers, inspect, stats.
- Write calls (restart, stop) are limited to an allowlist of container names in config, and are audited. See C-03.

### 4.5 Storage guardrails
- The mount list comes from config (**the infrastructure is not assumed**).
- Any path under `/mnt/internal-hdd` is shown as excluded (F-18). The exclusion list is in config, defaulting to that path.
- The write-heavy start check (F-17) waits on D-03.

## 5. Frontend design [REC]
- SvelteKit with `adapter-static` and an SPA fallback. All data comes from `/api/v1`; there's no server-side rendering.
- Routes: `/login`, `/` (Home), `/settings/theme`, `/admin/*` (only shown to admins; the server enforces this regardless), and `/desktop` from Phase 5.
- The theme runtime reads Prism JSON, expands it with `@material/material-color-utilities` into CSS custom properties, and follows the precedence rule S-15.
- TV and phone support: a responsive grid, focusable tiles that work with D-pad navigation, and no hover-only controls.
- No secrets and no service URLs reach the client. Only launcher URLs (D-02) do.

## 6. Data model [REC]
| Table | Key fields |
|---|---|
| `sessions` | id_hash, jellyfin_user_id, username, role, kind (cookie or bearer), device_label, created_at, last_seen_at, role_checked_at, expires_at |
| `user_prefs` | jellyfin_user_id, theme_json, home_layout_json, desktop_layout_json, updated_at |
| `household` | key (`theme`), value_json, updated_by, updated_at |
| `audit` | id, at, actor_user_id, actor_name, action, target, params_json (secrets redacted), result, request_id |

Jellyfin access tokens obtained at sign-in are not stored after the session is created, unless D-13 picks per-user tokens. In that case they're encrypted at rest; see SECURITY.md.

## 7. Dependencies on other components
| Component | Nexus needs from it | It needs from Nexus | Status |
|---|---|---|---|
| **Prism** | Theme JSON schema (seed, light/dark, radius, font, density, wallpaper) and a built-in default theme | Nexus stores and serves themes | Schema not yet written; Nexus validates against a pinned version |
| **Collector** | Theme packs (D-05); possibly launcher entries (D-02) and write-heavy metadata (D-03) | Session introspection (C-05); a window in desktop mode; possibly Compose apply (C-01) | Interfaces open |
| **Milano** | Nothing | Token exchange, server-control endpoints, \*arr and request proxies, theme sync, possibly notifications (D-10) | API_SPEC covers the Nexus side |
| **Infra** (Caddy, Tailscale Serve, socket proxy, host agent) | Must exist before deploy | — | Not built by Nexus. Owner approval needed for any change |

## 8. Deployment [REC]
- An image `ghcr.io/ntsrihari/knowhere-nexus:<version>`, multi-stage build, distroless base, non-root UID.
- Mounts: `config.yaml` (read-only), secrets (read-only), and a data volume for SQLite.
- From Phase 1, read-only host `/proc` and `/sys` mounts for metrics (pending C-04).
- The Compose file belongs in `knowhere-homelab-archive`. **Writing it there is an infrastructure change and needs owner approval.**

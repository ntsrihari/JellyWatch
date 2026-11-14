# Nexus — Tasks

Revision 2 (14 Nov 2026). **Planning only. Don't start implementation until Gate 0 is cleared.**
Tags: **[OWNER]** owner-approved · **[SPEC]** · **[REC]** pending approval · **[INFRA]** changes knowhere or the tailnet and needs explicit owner approval each time.

## Gate 0 — before any code
| # | Item | Status |
|---|---|---|
| G-1 | Architecture decisions A-01 to A-17 | ✅ Approved by the owner, 14 Nov 2026 |
| G-2 | B-01: approve the pending recommendations (stack P-xx, design R-01 to R-13) | ⏳ Needed |
| G-3 | B-02: create `ntsrihari/knowhere-nexus` and attach it to the build environment; move these documents there | ⏳ Needed |
| G-4 | B-03: Phase 1 catalog source (`catalog.yaml` until Collector publishes) | ⏳ Needed |
| G-5 | B-04: provisional `prism/0` theme schema | ⏳ Needed |
| G-6 | B-05a–c: confirm the defaults (widget fields, session times, Photos/DNS admin-only). Code can start on the defaults | ◻ Optional before code |

## Gate D — before Phase 1 deployment (code can proceed without these)
| # | Item |
|---|---|
| D-G1 | B-06 Routing and TLS strategy **[INFRA]** |
| D-G2 | B-07 Authorized reverse-proxy routes for Phase 1 **[INFRA]** |
| D-G3 | B-08 Tailscale ACL, and the identity-header binding choice **[INFRA]** |
| D-G4 | B-09 Host mounts for Nexus **[INFRA]** |
| D-G5 | B-10 Deploy docker-socket-proxy (read-only), Nexus and the Caddy routes **[INFRA]** |

---

## Phase 1 — revised implementation plan

Phase 1 delivers: Jellyfin sign-in with 7-day revocable sessions; a Home screen with a launcher from the server catalog; system, storage and service health for everyone, with detail for admins; a household theme; an audit log; and graceful degradation. Docker and the host are read-only.

### M1 — Skeleton and CI [REC]
- [ ] T-1.1 Go module, `cmd/nexus`, config loading (`config.yaml`, secrets, `catalog.yaml`, route registry), redacting secret type
- [ ] T-1.2 SQLite and migrations: session_families, access_tokens, refresh_tokens, user_prefs, household, audit
- [ ] T-1.3 HTTP server, middleware chain (ARCHITECTURE §4.2), default-deny authz table plus a startup check
- [ ] T-1.4 SvelteKit `adapter-static` app embedded in the binary
- [ ] T-1.5 GitHub Actions: lint, tests, `govulncheck`, `npm audit`, image build; GHCR push on tag
- [ ] T-1.6 Dockerfile: distroless, non-root, read-only root filesystem

### M2 — Auth, sessions, revocation, audit [OWNER A-10, A-14, A-15, A-16]
- [ ] T-2.1 Jellyfin user-token client: AuthenticateByName, `/Users/Me`, Quick Connect, `/Sessions/Logout`
- [ ] T-2.2 Session families: 15-minute access, rotating refresh with reuse detection, 7-day absolute cap
- [ ] T-2.3 Jellyfin token encryption at rest (AES-GCM, family-bound associated data)
- [ ] T-2.4 Role verification: every 5 minutes, before admin writes, and the 15-minute freshness rule
- [ ] T-2.5 Revocation: own sessions, admin revoke, revoke-all with re-auth, Jellyfin logout
- [ ] T-2.6 Audit writer: hash chain, fail-closed admin actions, list and verify endpoints
- [ ] T-2.7 Rate limiting, CSRF (Origin plus header), trusted-proxy handling, Tailscale IP-range check, recording `Tailscale-User-Login`
- [ ] T-2.8 Login, Quick Connect and session-management pages

### M3 — Read-only metrics and failure isolation [OWNER A-03, A-04, A-07, A-08, A-17]
- [ ] T-3.1 `resilience` package: timeouts, size caps, bulkheads, circuit breakers
- [ ] T-3.2 Widget framework: pollers, cache, `ok`/`stale`/`unavailable`/`not_configured`, per-widget panic isolation
- [ ] T-3.3 Host reader: `/host/proc` and `/host/sys` (CPU, RAM, load, temperature, uptime)
- [ ] T-3.4 Storage: mountinfo parsing, configured mount list, internal-hdd exclusion, user summary and admin detail; free space only if B-09 allows
- [ ] T-3.5 Docker read-only client (3 GET endpoints), inspect field allowlist, user up/down summary and admin container detail
- [ ] T-3.6 SSE with role-based variants
- [ ] T-3.7 Admin integration-status endpoint
- [ ] T-3.8 Mount re-check (read-only)

### M4 — Launcher and Home [OWNER A-02, A-09]
- [ ] T-4.1 `catalog/0` schema, loader, admin reload (validated, audited)
- [ ] T-4.2 Route registry; `href` only for authorized routes; otherwise the entry is unavailable
- [ ] T-4.3 `/api/v1/auth/verify` forward-auth, enabled only if B-07 approves a `nexus`-mode route
- [ ] T-4.4 Home UI: launcher grid, widget cards, phone and TV layouts, D-pad focus, home-layout preference

### M5 — Household theme [SPEC F-20]
- [ ] T-5.1 `prism/0` schema validation and built-in default
- [ ] T-5.2 Household GET and PUT (admin!, audited), effective-theme endpoint
- [ ] T-5.3 Theme runtime in the browser (material-color-utilities → CSS variables)

### M6 — Hardening and release readiness
- [ ] T-6.1 The full test plan below, all green in CI
- [ ] T-6.2 Security review against SECURITY.md (self-review checklist plus `/security-review`)
- [ ] T-6.3 Example `config.yaml`, `catalog.yaml` and route registry; deployment notes; example Compose file — **documents only**
- [ ] T-6.4 **[INFRA]** Deploy on knowhere — only after Gate D, with the owner present

**Phase 1 exit check (from the Plan):** family members can sign in and see live status. Measured by: login works for an admin and a non-admin account; the user sees health only; the admin sees detail; stopping any one integration leaves everything else working.

---

## Later phases
### Phase 2 — Collector support
- [ ] T-7.1 Signing keys, JWKS, rotation
- [ ] T-7.2 Service registry, `private_key_jwt` token endpoint, scopes, `jti` replay protection, denylist, enable/disable
- [ ] T-7.3 Delegation tokens from `/auth/verify` for `/store/`
- [ ] T-7.4 `PUT /service/catalog` (Collector publishes; the catalog file is retired)

### Phase 3 — Milano support
- [ ] T-8.1 Token exchange — *D-15*
- [ ] T-8.2 Per-user themes
- [ ] T-8.3 Admin broker and Jellyfin admin key — *D-19 [INFRA: creating a key in Jellyfin]*
- [ ] T-8.4 Jellyfin scheduled tasks, Now Playing (admin)

### Phase 4 — Requests and downloads
- [ ] T-9.1 Jellyseerr — *D-11* · T-9.2 Radarr and Sonarr · T-9.3 qBittorrent · T-9.4 Notifications — *D-10*

### Phase 5 — Desktop and theme packs
- [ ] T-10.1 Desktop mode — *D-06* · T-10.2 Theme packs — *D-05*

### Separately approved work only (not scheduled)
- Docker writes (D-18) · deployment API for Collector (D-16) · privileged host agent (D-17) · Photos and DNS widgets (not phased)

---

## Test plan
| Area | What must be shown |
|---|---|
| Authz | Every route × {anonymous, user, admin, stale admin, service}; unlisted routes denied; startup fails on a missing table entry |
| Visibility (A-07, A-08) | User responses (REST and SSE) never contain admin fields: a schema test on the DTOs, plus a test seeded with admin-only values |
| Login | Fake Jellyfin (`httptest`): success, failure (identical bodies), unreachable Jellyfin (502), Quick Connect states, rate limits |
| Sessions (A-10) | 15-minute access expiry; refresh rotation; reuse revokes the family; **the 7-day absolute cap can't be exceeded by any refresh sequence** (property test with a fake clock) |
| Revocation (A-16) | Own, admin and revoke-all (needs re-auth); immediate 401 afterwards; Jellyfin logout called; SSE `session.revoked` |
| Role changes | Demotion in Jellyfin removes admin within 5 minutes and immediately before admin writes; admin not honoured past 15 minutes without verification while Jellyfin is down |
| Jellyfin tokens (A-14) | Encrypted at rest; associated data stops cross-family reuse; deleted on revocation; no admin key loaded or used in Phase 1 |
| Audit (A-15) | Every admin action and auth event writes a row; the hash chain verifies; tampering is detected; an audit write failure aborts the admin action |
| Failure isolation (A-17) | For each integration: down, slow (timeout), oversized and erroring → `stale`/`unavailable`; other widgets and the API unaffected; breaker opens and recovers; Nexus starts with every integration down |
| Docker (A-03) | The client can only produce the 3 GET calls; inspect output is filtered (no env, mounts or labels) |
| Host (A-04) | Parsers tested on fixture `/proc` and `/sys` trees; no writes; internal-hdd excluded everywhere |
| Tailscale (A-11) | Non-Tailscale client IPs rejected; `X-Forwarded-For` ignored from untrusted peers; spoofed `Tailscale-User-*` and `X-Nexus-*` headers ignored unless from the trusted proxy |
| CSRF and web | Missing header or a foreign Origin rejected; CSP and other headers present |
| Secrets | Fuzz all responses, logs and audit rows for the loaded secret values: none found |
| Catalog and routes (A-02, A-09) | Invalid catalog rejected and the old one kept; entries without an authorized route have no `href`; `/auth/verify` refuses unknown routes and non-proxy callers |
| Frontend | Vitest (theme runtime, API client refresh logic); Playwright (login, Home on phone and TV viewports, D-pad focus, admin controls hidden for users) |
| Supply chain | `govulncheck`, `npm audit`, image scan in CI |

# Nexus — Tasks

Status: **planning only. Implementation has not started and waits for approval.**

Legend:
- **Blocked by** names a decision in DECISIONS.md that must be approved first.
- Tasks tagged **[REC]** come from engineering recommendations, not the spec.
- Tasks tagged **[INFRA]** change knowhere or its network. Nexus engineers don't do these without the owner's explicit approval.

## Gate 0 — Approvals before any code
- [ ] G-1 Owner reviews CLAUDE.md, ARCHITECTURE.md, API_SPEC.md, SECURITY.md, TASKS.md and DECISIONS.md
- [ ] G-2 Decide the spec conflicts C-01 to C-06
- [ ] G-3 Approve or amend the proposed decisions P-01 to P-15
- [ ] G-4 Answer the Phase 1-relevant open decisions: D-01 (widget visibility), D-02 (launcher source), D-07 (session lifetimes), D-08 (Tailscale identity), D-09 (routing), D-12 (repo location), D-13 (Jellyfin token model)
- [ ] G-5 Create the `knowhere-nexus` repo, or approve another location, and move these documents there

## Phase 1 — Nexus MVP (Plan §Roadmap Phase 1)
Gate from the Plan: *family can sign in and see live status.*

### 1. Skeleton [REC]
- [ ] T-1.1 Go module, `cmd/nexus`, config loader (`config.yaml` plus separate secrets), `slog`
- [ ] T-1.2 SvelteKit app with `adapter-static`, embedded in the Go binary
- [ ] T-1.3 SQLite store and migrations (sessions, user_prefs, household, audit)
- [ ] T-1.4 HTTP server: security headers, request id, recover, default-deny authz table plus a startup check
- [ ] T-1.5 GitHub Actions: lint, test, `govulncheck`, `npm audit`, build a multi-arch image and push it to GHCR on tag
- [ ] T-1.6 Dockerfile: distroless, non-root, read-only root filesystem

### 2. Authentication (F-01 to F-03)
- [ ] T-2.1 Jellyfin client: AuthenticateByName, Users/Me, Quick Connect — *blocked by D-13 for the token model*
- [ ] T-2.2 Sessions: create, hash, cookie, expiry, revoke — *blocked by D-07*
- [ ] T-2.3 Role mapping and periodic re-check (P-06)
- [ ] T-2.4 Login rate limiting and identical failure responses
- [ ] T-2.5 CSRF middleware (Origin plus header)
- [ ] T-2.6 Login and Quick Connect pages

### 3. Widgets (F-05 to F-07, F-18)
- [ ] T-3.1 Widget framework: pollers, cache, `stale` and `not_configured` states
- [ ] T-3.2 System widget from read-only `/proc` and `/sys` — *blocked by C-04 for the access method*
- [ ] T-3.3 Storage widget: configured mounts, mounted, ro/rw, free; internal-hdd exclusion. SMART shows `unknown` until C-04 is decided
- [ ] T-3.4 Docker proxy client (read-only) and Services widget
- [ ] T-3.5 SSE endpoint, filtered by role — *blocked by D-01*
- [ ] T-3.6 Home page: tile grid, widget cards, phone and TV layouts, D-pad focus

### 4. Theming (F-20) and launcher
- [ ] T-4.1 Pin the Prism theme schema version — *depends on Prism publishing its schema*
- [ ] T-4.2 Household theme GET and PUT, audited
- [ ] T-4.3 Client theme runtime: material-color-utilities to CSS variables
- [ ] T-4.4 `/api/v1/apps` and launcher tiles — *blocked by D-02, and by C-02 for any window behaviour*

### 5. Phase 1 release
- [ ] T-5.1 Deployment notes and an example Compose file (as a document only)
- [ ] T-5.2 **[INFRA]** Deploy the socket proxy and Nexus on knowhere, and add the Caddy route — **owner approval needed**

## Phase 2 — Collector support
- [ ] T-6.1 Introspection endpoint and service credential — *blocked by C-05, D-09*
- [ ] T-6.2 Collector Compose-apply API, if C-01 picks option (a) — *blocked by C-01*

## Phase 3 — Milano support (F-21, F-24, F-25)
- [ ] T-7.1 Token exchange plus a sessions list and revoke UI
- [ ] T-7.2 Per-user theme GET and PUT, and effective-theme resolution
- [ ] T-7.3 Admin container restart, stop and start with an allowlist and audit — *blocked by C-03, D-04*
- [ ] T-7.4 Mount guardrail on start — *blocked by D-03*
- [ ] T-7.5 Jellyfin scheduled tasks: list and run
- [ ] T-7.6 Now Playing widget — *blocked by D-01, D-13*
- [ ] T-7.7 Host restart and shutdown with typed confirmation and re-auth — *blocked by C-04*; **[INFRA]** the host agent needs owner approval
- [ ] T-7.8 Storage re-check action — *blocked by D-14*
- [ ] T-7.9 Audit log viewer

## Not phased in the plan (schedule to be decided)
- [ ] T-8.1 Photos widget (Immich)
- [ ] T-8.2 DNS widget (Pi-hole)

## Phase 4 — Requests and downloads (F-11, F-12, F-27)
- [ ] T-9.1 Jellyseerr search, request and list — *blocked by D-11*
- [ ] T-9.2 Radarr and Sonarr queue and calendar
- [ ] T-9.3 qBittorrent widget and list
- [ ] T-9.4 Notifications for Milano — *blocked by D-10*

## Phase 5 — Desktop and theme packs (F-22, F-23)
- [ ] T-10.1 Desktop mode: windows, dock, wallpaper, per-user layout — *blocked by C-02, D-06*
- [ ] T-10.2 Theme packs from Collector — *blocked by D-05*
- [ ] T-10.3 Container update — *blocked by C-01, C-03, C-06*

## Test plan (applies to every phase)
| Area | What must be tested |
|---|---|
| Authz | A table test: every route × {anonymous, user, admin} gives the expected status; an unlisted route gives 403; startup fails if a route has no entry |
| Auth | Login success and failure against a fake Jellyfin (`httptest`); identical failure bodies; rate limits; Quick Connect states; token exchange rejects bad or expired tokens; demotion in Jellyfin takes effect within the re-check window and immediately on admin actions |
| Sessions | Hashing at rest; expiry; revocation; logout; cookie flags |
| CSRF | Missing header or a foreign Origin is rejected on every unsafe method |
| Secrets | No secret appears in any API response, log line or audit row (fuzz responses against the loaded secrets); the redaction type's `String` and `MarshalJSON` |
| Docker | A contract test: the client can only emit allowlisted method and path pairs; non-allowlisted containers are refused; there's no create, exec, volume or image code |
| Guardrails | `/mnt/internal-hdd` is always `excluded` and never counted; start is refused on a missing or read-only mount; format and delete-volume don't exist (route list and code search) |
| Host | Typed confirmation mismatch; re-auth required; agent absent → 503 |
| Widgets | A downstream timeout gives `stale` or `unavailable` without blocking; `not_configured`; size caps; role filtering over REST and SSE |
| SSRF | No handler builds an outbound URL from client input (static analysis plus tests) |
| Themes | Schema validation, size limits, precedence S-15 |
| Frontend | Vitest for the theme runtime and stores; Playwright for login, Home on phone and TV viewports, D-pad focus order, admin controls hidden for users (the server is still tested separately) |
| Supply chain | `govulncheck`, `npm audit` and image scanning in CI |

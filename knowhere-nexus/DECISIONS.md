# Nexus — Decisions

This log records every architectural decision for Nexus. Each entry has a status:

| Status | Meaning |
|---|---|
| **Specified** | Comes from the Knowhere Platform Plan or the implementation brief. Not open for change here. |
| **Proposed** | An engineering recommendation. **Needs human approval** before implementation depends on it. |
| **Open** | No recommendation yet, or the spec conflicts with itself. **Needs a human decision.** |
| **Approved** | A proposed decision the owner has approved (none yet). |

Sources:
- **Plan** = Knowhere Platform Plan (doc, rev 10, 14 Nov 2026).
- **Brief** = the Nexus implementation brief (project constraints list).

---

## Specified decisions

| ID | Decision | Source |
|---|---|---|
| S-01 | Backend is written in Go, as one small binary (target about 30–60 MB RAM). | Plan §Dashboard "Stack"; Brief |
| S-02 | Frontend is SvelteKit, built to static files. | Plan §Dashboard "Stack"; Brief |
| S-03 | Users sign in with Jellyfin accounts (username and password, or Quick Connect). Nexus then issues its own session. | Plan §Identity; Brief |
| S-04 | The Jellyfin administrator flag maps to the Nexus admin role. Every other user gets the user role. | Plan §Identity; Brief |
| S-05 | Downstream API keys (Radarr, Sonarr, Prowlarr, Jellyseerr, qBittorrent, Immich) live only in Nexus's config on knowhere. They never go into apps or repos. | Plan §Identity "Secrets"; Brief |
| S-06 | Nexus is reachable only over Tailscale. Nothing is exposed to the internet. | Plan §Identity "Network"; Brief |
| S-07 | Docker access goes through a docker-socket-proxy that allows only the API calls Nexus needs. | Plan §Dashboard "Stack"; Brief |
| S-08 | Client devices never reach backend services directly. Collector and Milano ask Nexus. | Plan §Dashboard (intro); Brief |
| S-09 | Nexus has no format action and no delete-volume action. | Plan §Dashboard "Guardrails" |
| S-10 | Nexus never presents `/mnt/internal-hdd` as usable storage. | Plan §Dashboard "Guardrails" |
| S-11 | Nexus refuses to start a write-heavy app while its mount is missing or read-only. | Plan §Dashboard "Guardrails" |
| S-12 | Host restart and shutdown require a typed confirmation. | Plan §Dashboard "Admin actions" |
| S-13 | Per-user themes are stored in Nexus against the Jellyfin user id. | Plan §Theming |
| S-14 | An admin sets the household default theme in Nexus. | Plan §Theming |
| S-15 | Theme precedence, highest first: per-user, then Material You (Milano only), then household default, then Prism built-in. | Plan §Theming |
| S-16 | Nexus has two modes: Home (the default, works on phones and TVs) and Desktop (windows, dock, wallpaper; layout saved per user). | Plan §Dashboard |
| S-17 | Desktop mode ships in Phase 5. Phase 1 covers sign-in, the widget home, the system, storage and service widgets, and the Prism theme engine with a household default. | Plan §Roadmap |

---

## Conflicts in the spec (Open, need a decision)

### C-01 — Who holds Docker access: Nexus or Collector? — **Open**
- Plan §Dashboard says Nexus "is the only service that holds API keys and Docker access".
- Plan §App store says Collector "writes it to /opt/homelab/<app>, runs it", and the architecture diagram links Collector to the services.
- **Options:**
  - (a) Collector asks Nexus to run Compose operations through a Nexus API.
  - (b) Collector gets its own, separately restricted proxy.
  - (c) Revise the plan so that only Collector does deploys.
- **Recommendation:** (a). It keeps one privileged component. Nexus would then need a Compose-apply API, which isn't specified.
- **Blocks:** T-6.x (container update) and every Collector integration.

### C-02 — "Each service opens in a window (an iframe…)" vs "no direct access to backend services from client devices" — **Open**
- An iframe of Jellyfin or Immich loads that service straight into the client device's browser.
- **Options:**
  - (a) Only native Nexus panels; no iframes.
  - (b) Nexus reverse-proxies each embedded service behind its own session check. This is complex: Jellyfin and Immich have their own auth and websockets, and need sub-path or subdomain routing.
  - (c) Iframes allowed for named services through Caddy. This is an explicit exception to the constraint.
  - (d) The launcher opens services in a new tab, outside Nexus's scope.
- **Recommendation:** decide before Phase 5, when desktop mode arrives. Phase 1 launcher tiles can be plain links if (c) or (d) is accepted. Under (a) or (b) they wait.
- **Blocks:** launcher tile behaviour (T-4.4) and desktop mode (Phase 5).

### C-03 — "Read-mostly" socket proxy vs admin write actions — **Open**
- Restart and stop need `POST /containers/{id}/restart|stop`.
- "Update a container" needs an image pull and a container recreate (`POST /images/create`, `POST /containers/create`). Creating a container through the Docker API is equivalent to root on the host.
- **Recommendation:**
  - Phase 1: read-only proxy.
  - Phase 3: allow only `restart`, `stop` and `start`, with a container allowlist enforced in Nexus.
  - "Update": defer, and route through the C-01 outcome.
  - Note that docker-socket-proxy filters by API section (for example `CONTAINERS`, `POST`), not by container. Per-container limits must be enforced by Nexus itself. See SECURITY.md §4.
- **Needs approval:** whether "update" stays in Nexus at all.

### C-04 — The "Nexus agent" is undefined — **Open**
- Plan §Dashboard lists the System widget source as "/proc and hwmon via the Nexus agent". It doesn't say what the agent is.
- Restarting or shutting down the host, SMART health (`smartctl`) and mount re-checks all need privileges a normal container doesn't have.
- **Options:**
  - (a) A separate, minimal host agent: a systemd service with a fixed command set, reached over a root-owned Unix socket mounted into Nexus.
  - (b) A privileged Nexus container with host PID, `/dev` and `/proc` access.
  - (c) Read-only bind mounts of host `/proc` and `/sys` into Nexus for metrics, with SMART and power actions deferred.
- **Recommendation:**
  - Phase 1: (c), which needs no new privileges.
  - From Phase 3, for power and SMART: (a).
  - Never (b).
- **Needs approval.** Any host agent is a change to production infrastructure and needs the owner's sign-off.

### C-05 — How Collector trusts Nexus sessions — **Open**
- Plan: "Collector trusts Nexus sessions." No mechanism is given.
- **Options:**
  - (a) Same-origin routing (Nexus at `/`, Collector at `/store/`) so they share a cookie, with Collector validating it by calling Nexus introspection.
  - (b) Nexus issues short-lived signed tokens (for example Ed25519 JWTs) that Collector verifies offline.
- **Recommendation:** (a) plus an introspection endpoint. It is simpler, and revocation works immediately.
- **Depends on:** routing (D-09).

### C-06 — Container "update" in Nexus vs "update available, an admin applies it" in Collector — **Open**
- Both are specified. One flow should own updates; the other links to it.
- **Recommendation:** Collector owns updates; the Nexus Services widget links to them.

---

## Proposed engineering decisions (need approval)

| ID | Proposal | Why | Alternatives |
|---|---|---|---|
| P-01 | Go 1.23+, router `chi`, standard `net/http`, `log/slog` | Small, stable, few dependencies | `echo`, `gin` |
| P-02 | SQLite (pure-Go `modernc.org/sqlite`) for sessions, per-user preferences, layouts, household theme, audit log | One file, no extra container, fits the RAM budget | bbolt; JSON files |
| P-03 | SvelteKit with `adapter-static`, SPA fallback, embedded in the Go binary with `embed.FS` | One artifact, nothing extra to deploy | Separate static server |
| P-04 | Server-Sent Events (`/api/v1/events`) for live widgets. Polling is the fallback. | One-way updates, simpler than websockets, works through Caddy | Websockets; polling only |
| P-05 | Opaque random session ids (32 bytes) in an `HttpOnly; Secure; SameSite=Strict` cookie, stored server-side. Bearer tokens of the same kind for Milano. | Server-side revocation; nothing sensitive held on the client | Signed JWTs |
| P-06 | Re-check the Jellyfin admin flag when a session is created, every 5 minutes after that, and always before an admin action | Demoting a user in Jellyfin takes effect quickly | Check on every request (more Jellyfin load) |
| P-07 | Secrets in a separate file (`secrets.yaml`, mode 0600, owned by the Nexus UID) or Docker secrets, apart from non-secret `config.yaml` | Keeps secrets out of configuration that may be committed | Env vars; encrypted store; SOPS |
| P-08 | Secrets are not editable through the Nexus UI in v1 | Smaller attack surface; the spec doesn't ask for it | UI editing behind re-auth |
| P-09 | Nexus listens only on loopback or a private Docker network. Caddy and Tailscale Serve front it. | Puts "Tailscale-only" into practice | Also verify Tailscale identity headers (D-08) |
| P-10 | Every admin action is written to an append-only audit table: who, what, target, result and time | Accountability for destructive actions | Logs only |
| P-11 | Destructive host actions (restart, shutdown) need a typed hostname, plus a sign-in within the last 10 minutes | Meets S-12 and limits the damage from a stolen session | Typed confirmation only |
| P-12 | Outbound HTTP only to service base URLs in config. No endpoint accepts a URL from a client. | Prevents SSRF | — |
| P-13 | Milano exchanges its Jellyfin token: Nexus validates it with `GET /Users/Me` on Jellyfin, then mints a Nexus bearer token | This is the plan's flow; validating the token server-side means no trust in client claims | — |
| P-14 | Nexus stores themes as Prism JSON. The full colour scheme is built in the browser with `@material/material-color-utilities` (TypeScript). | The Go side needs no colour maths; matches Prism's design | Expand in Go |
| P-15 | API versioned under `/api/v1`, with JSON errors shaped `{"error":{"code","message"}}` | Stable contract for Milano and Collector | — |

## Other open decisions (need a human decision)

| ID | Question | Why it matters |
|---|---|---|
| D-01 | Which widgets can non-admin users see? Now Playing shows other family members' viewing; Storage and Services expose internals. | Privacy and least privilege. The Phase 1 gate says "family can sign in and see live status", but doesn't say which status. |
| D-02 | Where do launcher entries (name, icon, URL, mode) come from: Nexus config, Collector's catalog, or both? | The data model for the Home grid. |
| D-03 | Which apps count as "write-heavy", and which mounts does each need (S-11)? Nexus config, or Collector catalog metadata? | The guardrail can't be built without this mapping. |
| D-04 | Stop is specified but start isn't. Should Nexus have a start action? | Without one, a stopped container can only come back through another tool. |
| D-05 | How are theme packs installed by Collector handed to Nexus: a Collector API, a shared volume, or a push to Nexus? | Specified feature, unspecified interface. |
| D-06 | Where are wallpapers stored, and what size and type limits apply? | Upload handling is security-sensitive. |
| D-07 | Session lifetime: browser idle and absolute timeouts; Milano token lifetime and refresh. | Usability vs exposure. Proposal: 7-day idle and 30-day absolute for browsers; 90-day Milano tokens, revocable from the UI. |
| D-08 | Should Nexus also verify Tailscale identity (for example the `Tailscale-User-Login` header from Tailscale Serve, or tailscaled's LocalAPI `whois`)? | Defence in depth, but it ties Nexus to Tailscale. |
| D-09 | Routing: Nexus at `/` with other services on sub-paths, or on subdomains? Plan: "Caddy routes paths and subdomains". | Affects cookie scope (C-05), CSP and iframes (C-02). |
| D-10 | Should Nexus provide notifications (stream started, server down, download finished) to Milano? If so, how, given no internet exposure? Options: polling, UnifiedPush/ntfy on the tailnet. | The Plan lists them for Milano in Phase 4 without naming a source. |
| D-11 | Jellyseerr requests made through Nexus: under whose identity? Overseerr-family APIs accept an API key plus an `X-API-User` header to act as a user; **this needs checking** against the deployed Jellyseerr version. | Correct attribution and permissions for family requests. |
| D-12 | Where should the Nexus repo live? These documents are on a working branch of `ntsrihari/JellyWatch` until `knowhere-nexus` exists. | Repository hygiene. |
| D-13 | Is Jellyfin data shown in Nexus (Now Playing, scheduled tasks) fetched with a server-side admin API key, or with the signed-in user's own Jellyfin token? | An admin key shows everything to every viewer unless Nexus filters. The user's token limits it naturally. |
| D-14 | Re-check mounts (admin action): what exactly does it do? A read-only status refresh, or a remount attempt? | A remount needs host privileges (C-04). Proposal: refresh status only. |

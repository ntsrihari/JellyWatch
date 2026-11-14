# Nexus — Decisions

Revision 2. This revision applies the owner's decisions of 14 Nov 2026. Implementation has not started.

## Legend
| Tag | Meaning | Who can change it |
|---|---|---|
| **[OWNER]** | Explicitly approved by the owner (Srihari) on 14 Nov 2026 | Owner only |
| **[SPEC]** | Stated in the Knowhere Platform Plan or the original brief, and not overridden by an [OWNER] decision | Owner only |
| **[REC]** | Engineering recommendation, **pending owner approval** | Proposed here; the owner approves |
| **[BLOCKER]** | Unresolved; blocks the work named next to it | Owner decides |
| **[LATER]** | Unresolved; doesn't block Phase 1 | Owner decides before that phase |

---

## A. Owner-approved decisions [OWNER]

| ID | Decision, as approved | Resolves | Engineering consequences (see the [REC] items) |
|---|---|---|---|
| A-01 | Nexus is the central authorization and execution boundary. Collector must request deployment operations through Nexus. | C-01 | No component but Nexus executes against Docker or the host. The deployment API is not designed yet (D-16) and stays out of scope (A-03). |
| A-02 | No client-side direct access to backend services. All service access goes through Nexus or an explicitly authorized reverse proxy. | C-02 | A registry of authorized routes, each approved individually (R-06). No iframes of unapproved services. |
| A-03 | Phase 1 uses read-only Docker access. Container creation, updates, deletion and other privileged operations are out of scope until separately approved. | C-03 | Restart, stop and start are also writes, so they're out of scope until approved (D-18). |
| A-04 | Phase 1 supports read-only system metrics. Privileged host operations need a separately designed and approved agent. | C-04, D-14 | SMART health, host restart and shutdown, and remount attempts are out of Phase 1. "Re-check mounts" is a read-only refresh. Agent design: D-17. |
| A-05 | Collector authenticates with short-lived, signed, Nexus-issued service tokens. Token audience, expiry, revocation and service identity must be defined. | C-05 | Defined in R-03. |
| A-06 | Collector owns the catalog and update workflow. Nexus owns authorization and execution boundaries. | C-06 | Collector proposes; Nexus authorizes and executes. Conflicts with A-09 in Phase 1, before Collector exists: see B-03. |
| A-07 | General system health widgets are visible to all users. | D-01 (part) | Field classification in R-08. |
| A-08 | Detailed viewing activity and administrative metrics are admin-only. | D-01 (part) | Now Playing, container details, mount internals and integration health are admin-only (R-08). |
| A-09 | The launcher reads from a server-managed application catalog. | D-02 | No client-defined launcher entries. Phase 1 source: B-03. |
| A-10 | Sessions have a seven-day maximum lifetime, with revocable refresh tokens. | D-07 | R-01. |
| A-11 | Tailscale network membership is mandatory. Additional identity validation is to be assessed. | D-08 (part) | Assessment and recommendation in R-04. |
| A-12 | Prefer subdomains for service routing, with paths where appropriate. | D-09 | Tailscale HTTPS certificates limit this: see R-05 and B-06. |
| A-13 | Target repository: `ntsrihari/knowhere-nexus`. | D-12 | The repo must be created and attached to this environment (B-02). |
| A-14 | Normal Jellyfin operations use user-specific authentication. Privileged operations use separately protected administrative credentials. | D-13 | R-10. Phase 1 needs no Jellyfin admin credential. |
| A-15 | Administrative audit logging is required. | P-10 | Was a recommendation; now required. Design: R-11. |
| A-16 | Session revocation is required. | — | R-02. |
| A-17 | Downstream services that are unavailable must fail gracefully, in isolation. | — | R-09. |

## B. Specified decisions still in force [SPEC]

| ID | Decision | Source | Note |
|---|---|---|---|
| S-01 | Go backend, single binary, about 30–60 MB RAM | Plan; Brief | |
| S-02 | SvelteKit compiled to static files | Plan; Brief | |
| S-03 | Jellyfin sign-in (password or Quick Connect); Nexus issues its own session | Plan; Brief | Session shape now set by A-10 |
| S-04 | Jellyfin administrator = Nexus admin; everyone else = user | Plan; Brief | |
| S-05 | Downstream API keys live only in Nexus's config on knowhere | Plan; Brief | |
| S-06 | Tailscale-only access | Plan; Brief | Strengthened by A-11 |
| S-07 | Docker through a restricted docker-socket-proxy | Plan; Brief | Read-only in Phase 1 (A-03) |
| S-08 | Clients don't reach backend services directly | Plan; Brief | Superseded in wording by A-02 |
| S-09 | No format or delete-volume action | Plan | |
| S-10 | `/mnt/internal-hdd` is never shown as usable | Plan | |
| S-11 | Refuse to start a write-heavy app while its mount is missing or read-only | Plan | Applies only once start is approved (D-18) |
| S-12 | Host restart and shutdown need typed confirmation | Plan | Deferred by A-04 |
| S-13 | Per-user themes are stored against the Jellyfin user id | Plan | |
| S-14 | An admin sets the household default theme | Plan | |
| S-15 | Theme precedence: user, then Material You (Milano), then household, then built-in | Plan | |
| S-16 | Home mode (default) and Desktop mode | Plan | Desktop in Phase 5 |
| S-17 | Phase 1 = sign-in, widget home, system, storage and service widgets, Prism household theme | Plan | |

## C. Engineering recommendations [REC — pending approval]

### Carried over from revision 1
| ID | Recommendation | Status |
|---|---|---|
| P-01 | Go 1.23+, `chi` router, `net/http`, `log/slog` | Pending |
| P-02 | SQLite via pure-Go `modernc.org/sqlite` for sessions, preferences, themes and audit | Pending |
| P-03 | SvelteKit with `adapter-static`, embedded with `embed.FS` | Pending |
| P-04 | Server-Sent Events for live widgets, with polling as the fallback | Pending |
| P-05 | ~~Opaque session cookie~~ | Replaced by R-01 |
| P-06 | Re-check the Jellyfin admin flag every 5 minutes and before admin actions | Pending; now uses the user's own token (R-10) |
| P-07 | Secrets in a separate 0600 file or Docker secrets | Pending |
| P-08 | No secret editing through the UI in v1 | Pending |
| P-09 | Nexus listens only on loopback or a private network | Pending; part of R-04 |
| P-10 | Audit log | Now required by A-15; design in R-11 |
| P-11 | Destructive actions need a re-auth within 10 minutes | Pending; first used in Phase 1 by "revoke all sessions of a user" |
| P-12 | No outbound URL built from client input | Pending |
| P-13 | Milano token exchange, validated with Jellyfin `/Users/Me` | Pending (Phase 3); see D-15 |
| P-14 | Theme expansion in the browser with material-color-utilities | Pending |
| P-15 | `/api/v1` and a standard error envelope | Pending |

### New in this revision

**R-01 — Session model (implements A-10)**
- **Access credential:**
  - an opaque 256-bit id, stored server-side as a SHA-256 hash;
  - lifetime 15 minutes;
  - browser: cookie `nexus_access` (`HttpOnly; Secure; SameSite=Strict; Path=/`, host-only, no `Domain` attribute);
  - Milano: a bearer token.
- **Refresh credential:**
  - an opaque 256-bit id, single-use and rotated on every refresh;
  - browser: cookie `nexus_refresh` (`Path=/api/v1/auth/refresh`, same flags);
  - Milano: a body field.
- **Session family:**
  - everything stems from one login;
  - **absolute expiry 7 days after login, never extended**;
  - after that, the user signs in again.
- **Reuse detection:** presenting an already-used refresh token revokes the whole family and writes a security audit event.

**R-02 — Revocation (implements A-16)**
- Users can list and revoke their own sessions.
- Admins can revoke any session, or every session of a user.
- Logout revokes the current family.
- Every revocation also calls Jellyfin `POST /Sessions/Logout` with that session's stored Jellyfin token, and then deletes the token.
- **Effect:**
  - In Nexus, revocation is immediate, because every request is checked server-side.
  - Services that verify delegation tokens offline (R-03) get up to the token lifetime of staleness: at most 120 s.
  - Anything those services ask Nexus to *execute* is re-checked against the live session (`sid`), so revocation is immediate for every privileged path.

**R-03 — Service tokens (implements A-05)**

| Property | Design |
|---|---|
| Service identity | Each service is registered in Nexus config: `id` (for example `collector`), its Ed25519 **public key**, allowed scopes and allowed audiences. The private key stays with the service. A disabled flag revokes the service instantly. |
| Service → Nexus authentication | Client credentials with a `private_key_jwt` assertion (RFC 7523): a JWT signed by the service, `iss=sub=<service id>`, `aud=<Nexus token URL>`, `exp ≤ 60 s`, single-use `jti`. Nexus returns a **service access token**. |
| Service access token | A JWT (EdDSA) signed by Nexus: `iss=nexus`, `aud=nexus`, `sub=svc:<id>`, `scope`, `iat`, `exp` (**5 minutes**), `jti`, `kid`. |
| User delegation token | A JWT (EdDSA) signed by Nexus, minted when an authorized reverse proxy route calls Nexus's forward-auth check: `iss=nexus`, `aud=<service id>`, `sub=user:<jellyfin id>`, `role`, `sid` (the session family reference), `exp` (**120 seconds**), `jti`. The service verifies it offline with Nexus's JWKS. |
| On-behalf-of calls | When Collector asks Nexus to do something for a user, it sends its service access token **and** the user's delegation token. Nexus checks the service scope, the user's role and that the `sid` session is still live. |
| Audience | Nexus rejects any token whose `aud` isn't its own. A service must reject any token whose `aud` isn't its own id. |
| Revocation | Short expiry; a `jti` denylist kept until expiry; disabling a service; revoking a session (through `sid`); signing-key rotation. |
| Keys | Nexus signing keys are Ed25519, rotated every 90 days, with the old key kept in the JWKS until every token signed with it has expired. Private keys are in the secrets file. |
| Scopes | Registered list. Proposed for Phase 2: `catalog:publish`, `themes:publish`. **Reserved and not approved:** `deploy:request` (A-03, D-16). |

**R-04 — Tailscale membership and identity (assessment for A-11)**

| Option | What it gives | Risks and costs | Recommendation |
|---|---|---|---|
| 1. Tailscale ACL/grants allowing only the family group to reach knowhere on 443 | Membership **and** per-user permission, enforced by the network | A change to the tailnet policy (infrastructure) | **Yes**, mandatory baseline |
| 2. Nexus only accepts client IPs in Tailscale ranges (100.64.0.0/10, fd7a:115c:a1e0::/48), read from `X-Forwarded-For` set by a configured trusted proxy | Catches a misconfiguration that exposes Nexus on the LAN | Must ignore the header unless it came from the trusted proxy | **Yes**, cheap defence in depth |
| 3. Tailscale Serve identity headers (`Tailscale-User-Login`, `Tailscale-User-Name`) | Shows who on the tailnet is calling; can bind a session to that identity, so a stolen cookie fails on another person's device | Headers are absent for tagged devices and possibly for shared-in nodes (**needs checking**); trustworthy only if every request passes through Serve | **Record in sessions and the audit log in Phase 1.** Binding sessions to it: off by default; owner to decide (B-08) |
| 4. tailscaled LocalAPI `whois` through the mounted socket | Authoritative identity | The socket gives broad control over tailscaled | **No** |
| 5. Embed `tsnet` in Nexus (Nexus becomes its own tailnet node) | Native, authoritative WhoIs; no Caddy needed for Nexus | Another node and auth key to manage; changes the routing design | Alternative, if the owner prefers it |

**R-05 — Routing under Tailscale (implements A-12)**
- Tailscale's HTTPS certificates cover a node's own name (`knowhere.<tailnet>.ts.net`), **not subdomains under it**. Subdomain routing therefore needs one of:
  - (a) a domain you own, with a wildcard certificate issued through Caddy's DNS-01 challenge, and DNS (public, or Pi-hole split DNS) pointing at knowhere's Tailscale IP. A wildcard keeps service names out of Certificate Transparency logs.
  - (b) one Tailscale node per exposed service (sidecars or tsnet), each with its own `ts.net` name.
  - (c) paths on the single `ts.net` name.
  - Check whether newer Tailscale features change this before deciding.
- **Recommended:**
  - (a) if you own a domain, otherwise (b);
  - Nexus at the apex (or `nexus.`);
  - **Collector's web UI at the path `/store/` on Nexus's host.** This is "paths where appropriate": it lets the host-only Nexus cookie reach the forward-auth check, without widening the cookie to a parent domain.
- **This is infrastructure, and is blocker B-06.**

**R-06 — Authorized reverse-proxy routes (implements A-02)**
- Nexus config holds a route registry. Each entry has: route id, host or path, upstream service, auth mode, approved-by and approved-on.
- Auth modes:
  - `nexus` — Caddy calls `GET /api/v1/auth/verify`; Nexus injects a delegation token.
  - `upstream` — the service's own auth, for services with native clients such as Jellyfin for Milano or the Immich app, which can't use Nexus cookies.
- The launcher only opens entries whose route is in the registry. Other entries show as unavailable.
- Caddy strips any client-supplied `X-Nexus-*` headers, and removes Nexus cookies before forwarding upstream. The exact directives are to be validated.
- **Every route needs owner approval: B-07.**

**R-07 — Phase 1 catalog source (reconciles A-06 with A-09)**
- In Phase 1, Nexus serves the launcher from a **server-side `catalog.yaml`**: read-only, loaded at start, admin-maintained, schema `catalog/0`.
- From Phase 2, Collector (which owns the catalog under A-06) publishes it to Nexus with the `catalog:publish` scope, and the file stops being used.
- **Needs confirmation: B-03.**

**R-08 — Widget visibility (implements A-07 and A-08)**

| Data | All users | Admin only |
|---|---|---|
| System: CPU %, RAM used/total, load, temperature, uptime | ✓ | |
| Storage summary: per mount, display label, health (ok / degraded / unavailable), used % | ✓ | |
| Storage detail: path, ro/rw, bytes free and total, excluded flag, SMART (when approved) | | ✓ |
| Services summary: per catalog app, display name and up/down | ✓ | |
| Container detail: container name, image, state, restart count, memory | | ✓ |
| Now Playing (viewing activity) | | ✓ |
| Integration health, circuit-breaker state | | ✓ |
| Audit log, other users' sessions | | ✓ |
| Photos (Immich) and DNS (Pi-hole) aggregates — not phased | | ✓ proposed until confirmed |

**R-09 — Failure isolation (implements A-17)**
- Every integration client has:
  - timeouts: connect 3 s, total 10 s;
  - a response size cap of 2 MB;
  - a concurrency limit of 4;
  - a circuit breaker: opens after 5 consecutive failures, tries one request again after 30 s.
- Widgets are always served from cache. A result is `stale` after twice its poll interval, and `unavailable` when the breaker is open and there's no cached value younger than 10 minutes.
- Nexus starts and serves even when every integration is down. Only config and the database are needed.
- **When Jellyfin is down:**
  - existing sessions keep working (checked locally);
  - new logins return `upstream_unavailable`;
  - refresh keeps working with the last verified role, but **the admin role is honoured only if it was verified in the last 15 minutes**, otherwise requests run as a user (fail closed);
  - revocation always works.
- Every poller goroutine and handler recovers from panics on its own. One broken widget never affects another, or the SSE stream.

**R-10 — Jellyfin credentials (implements A-14)**
- **User credential:**
  - the Jellyfin token from the user's own login, kept for the life of the session;
  - encrypted at rest with AES-256-GCM, using a key from the secrets file;
  - used for role checks and user-level Jellyfin calls;
  - deleted, and logged out on Jellyfin, at revocation or expiry.
- **Admin credential:**
  - a dedicated Jellyfin API key in the secrets file;
  - only reachable through an internal "admin broker", which requires an admin role verified in the last 15 minutes;
  - every use is audited.
  - **Not needed in Phase 1.**

**R-11 — Audit (implements A-15)**
- Recorded:
  - every admin action;
  - denied admin attempts;
  - logins, failures and Quick Connect;
  - refresh-token reuse;
  - session revocations;
  - household theme changes;
  - catalog reloads;
  - service token issuance and refusals.
- Each entry: actor, Tailscale login (when present), action, target, sanitized params, result, request id and time.
- Append-only table with a hash chain (each row includes the hash of the previous one) to make tampering evident.
- **If the audit write fails, the admin action fails.**
- Retention 365 days.

**R-12 — Provisional theme schema**
- Prism doesn't exist yet. Nexus ships a provisional `prism/0` schema (the fields named in the plan) and migrates to Prism's published schema later.
- **Needs confirmation: B-04.**

**R-13 — Read-only host access for Phase 1 (implements A-04)**
- Mount host `/proc` and `/sys` read-only into the Nexus container, for CPU, RAM, load, temperature, uptime and host mount info.
- Free space needs `statfs` on each mount point, which means read-only bind mounts of `/mnt/media` and `/mnt/immich` into Nexus. Nexus could then *read* media and photos.
- **Alternative:** omit free space from Phase 1 and report only mounted, ro/rw and health from mountinfo.
- **Needs approval: B-09.**

---

## D. Remaining decisions

### Phase 1 blockers [BLOCKER]
| ID | Decision needed | Blocks | Recommendation |
|---|---|---|---|
| B-01 | Approve the stack and pending recommendations P-01 to P-04, P-06 to P-09, P-11, P-12, P-14, P-15 and R-01 to R-13 (or amend them) | All code | Approve as written |
| B-02 | Create `ntsrihari/knowhere-nexus` and attach it to this environment. I can't create repositories. | Any commit to the target repo | — |
| B-03 | Phase 1 catalog source: a Nexus-hosted `catalog.yaml` until Collector publishes (R-07), or wait for Collector? | Launcher (T-4.x) | R-07 |
| B-04 | Provisional `prism/0` theme schema (R-12), or wait for Prism? | Household theme (T-5.x) | R-12 |
| B-06 | Routing and TLS: own domain with DNS-01 wildcard, a node per service, or paths only (R-05) **[INFRA]** | Phase 1 deploy, cookie scope and CSP origin | (a) if you own a domain |
| B-07 | Which reverse-proxy routes are authorized for Phase 1, with which auth mode (R-06)? An empty list is valid: the launcher then shows entries as unavailable. **[INFRA]** | Opening launcher entries | Start with none, or Jellyfin (`upstream`) only |
| B-08 | Tailscale identity: approve options 1 and 2, option 3 record-only, and say whether to bind sessions to the Tailscale login (R-04). Option 1 is a tailnet policy change. **[INFRA]** | Phase 1 deploy | As in R-04; binding off |
| B-09 | Host mounts for Nexus: `/proc` and `/sys` read-only, and data mount points read-only, or no free-space figures (R-13) **[INFRA]** | Storage widget scope and deploy | `/proc` and `/sys` only; free space from read-only binds if you accept the read exposure |
| B-10 | Approve deploying docker-socket-proxy (read-only profile) and Nexus on knowhere, plus the Caddy route, once the code is ready **[INFRA]** | Phase 1 go-live | — |

### Confirmations that don't block code (defaults are used until answered)
| ID | Item | Default |
|---|---|---|
| B-05a | Widget field classification (R-08) | As in the table |
| B-05b | Session times: 15-minute access, rotating refresh, 7-day absolute | As in R-01 |
| B-05c | Photos and DNS widgets admin-only | Admin-only |

### Later phases [LATER]
| ID | Item | Phase |
|---|---|---|
| D-03 | Write-heavy apps and their required mounts (now from Collector catalog metadata, A-06) | When start is approved |
| D-04 | Whether a start action exists | With D-18 |
| D-05 | Theme pack handover (proposed: Collector publishes with `themes:publish`) | 5 |
| D-06 | Wallpaper storage and limits | 5 |
| D-10 | Notifications for Milano | 4 |
| D-11 | Jellyseerr request identity (`X-API-User` needs checking) | 4 |
| D-15 | Milano and the 7-day cap: may Milano silently re-exchange its stored Jellyfin token after 7 days, or must the user sign in again? | 3 |
| D-16 | Deployment operations API from Collector to Nexus (A-01), including design, scopes and Docker permissions | Needs separate approval |
| D-17 | Privileged host agent design (power, SMART, remount) (A-04) | Needs separate approval |
| D-18 | Any Docker write: restart, stop, start, create, update (A-03) | Needs separate approval |
| D-19 | Jellyfin admin API key for scheduled tasks and Now Playing (R-10); creating the key is an action in Jellyfin itself | 3 |

### Closed items from revision 1
C-01 → A-01 · C-02 → A-02 · C-03 → A-03 · C-04 → A-04 · C-05 → A-05, R-03 · C-06 → A-06 · D-01 → A-07, A-08, R-08 · D-02 → A-09, R-07 · D-07 → A-10, R-01 · D-08 → A-11, R-04 · D-09 → A-12, R-05 · D-12 → A-13 · D-13 → A-14, R-10 · D-14 → A-04

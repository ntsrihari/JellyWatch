# Nexus — Security Design

Revision 2 (14 Nov 2026). Tags: **[OWNER]**, **[SPEC]**, **[REC]**, **[BLOCKER]**/**[LATER]** (see DECISIONS.md).

Nexus is the platform's central authorization and execution boundary **[OWNER A-01]**. Its compromise would expose every downstream credential and the Docker API. Phase 1 deliberately keeps Nexus **read-only towards Docker and the host [OWNER A-03, A-04]**, so the worst case in Phase 1 is disclosure rather than control.

## 1. Assets
| Asset | Phase 1 exposure |
|---|---|
| Docker API (through the proxy) | Read-only: container metadata, including environment variables in `inspect` (Nexus must not return them) |
| Host | Read-only `/proc` and `/sys`; possibly read-only data mounts (B-09) |
| Downstream API keys | In memory, from the secrets file |
| Users' Jellyfin tokens | Encrypted in SQLite for the life of a session [OWNER A-14] |
| Jellyfin admin key | **Not present in Phase 1** [REC R-10] |
| Nexus sessions and refresh tokens | Hashes only in SQLite |
| Nexus signing keys | Phase 2, secrets file |
| Viewing activity | Admin-only [OWNER A-08]; Now Playing not in Phase 1 |
| Audit log | Hash-chained |

## 2. Threats
| Actor | Threats | Main controls |
|---|---|---|
| Device not on the tailnet, or not in the family group | Reaching Nexus | Tailscale membership [OWNER A-11]; ACL (R-04 option 1); Tailscale IP-range check (option 2) |
| Non-admin family member | Admin endpoints, viewing activity, others' sessions | Default-deny authz; separate `/admin/*` widget endpoints [OWNER A-08] |
| Demoted admin | Keeping admin access | Role re-check every 5 minutes and before admin writes; admin honoured only if verified within 15 minutes [REC R-09/R-10] |
| Stolen cookie or refresh token | Session hijack | 15-minute access tokens; rotation with reuse detection; 7-day cap [OWNER A-10]; revocation [OWNER A-16]; optional Tailscale-login binding (B-08) |
| Malicious web page | CSRF, clickjacking | SameSite=Strict, Origin plus header check, `frame-ancestors 'none'` |
| Compromised downstream service | XSS via titles; huge responses; hanging calls | Output escaping, size caps, timeouts, circuit breakers [OWNER A-17] |
| Compromised Collector (Phase 2) | Using Nexus beyond its role | Scoped, 5-minute service tokens; delegation required for user actions; `deploy:request` not granted [OWNER A-03]; disable switch |
| Request forgery through Caddy headers | Spoofed `X-Nexus-*` or `Tailscale-User-*` headers | Caddy strips client copies; Nexus trusts them only from the configured proxy address |
| Supply chain | Malicious dependency | Pinned dependencies, `govulncheck`, `npm audit`, distroless image, CI-only builds |

## 3. Security-sensitive operations
| Operation | Phase | Role | Controls |
|---|---|---|---|
| Login, Quick Connect | 1 | public | Rate limits (5 failures per username and 20 per IP per 15 minutes); one failure message; audit; passwords never logged or stored |
| Refresh | 1 | refresh credential | Single use; reuse revokes the family; never extends the 7-day cap |
| Session revocation (own) | 1 | user | Ownership check; Jellyfin logout; audit |
| Session revocation (other user) | 1 | admin! | Audit |
| Revoke all of a user's sessions | 1 | admin!+reauth | Audit |
| Household theme change | 1 | admin! | Schema validation; audit |
| Catalog reload | 1 | admin! | Validation; the old catalog is kept on error; audit |
| Mount re-check | 1 | admin | Read-only [OWNER A-04]; audit |
| Forward-auth verify | 1 (if B-07) | Caddy | Route must be in the approved registry; only from the Caddy address |
| Service token issuance | 2 | service | `private_key_jwt`, single-use assertion `jti`, scope allowance, audit |
| Catalog publish by Collector | 2 | service + admin delegation | Scope plus live `sid` check; audit |
| Jellyfin scheduled tasks, Now Playing | 3 | admin! | Admin broker; audit |
| Docker writes, deploys, host power, SMART | — | — | **Out of scope until separately approved** [OWNER A-03, A-04] |

## 4. Controls

### 4.1 Network and Tailscale [OWNER A-02, A-11, A-12; REC R-04, R-05, R-06]
- Mandatory tailnet membership, enforced by:
  - (1) the tailnet ACL limiting knowhere:443 to the family group **[INFRA, B-08]**;
  - (2) Nexus rejecting client IPs outside 100.64.0.0/10 and fd7a:115c:a1e0::/48, read from `X-Forwarded-For` only when the connection comes from the configured trusted proxy;
  - Nexus binding only to loopback or a private network.
- Additional identity: `Tailscale-User-Login` is recorded in sessions and the audit log. Binding a session to it is an owner choice (B-08), off by default.
  - **Needs checking:** whether Serve sets these headers for shared-in nodes and tagged devices.
  - The LocalAPI socket is not mounted into Nexus.
- Routing: TLS and hostnames per B-06. Collector's UI sits at `/store/` on Nexus's host, so the Nexus cookie stays host-only.
- Reverse-proxy routes: only routes in the approved registry (B-07).
  - Caddy strips client-supplied `X-Nexus-*` headers and Nexus cookies on the way upstream.
  - `/api/v1/auth/verify` answers only the Caddy address.

### 4.2 Docker [OWNER A-03]
- The proxy profile is read-only (`CONTAINERS=1`, `POST=0`; every other section 0), on an internal network that only Nexus can reach.
- The Nexus client has no write methods at all.
- A contract test proves the client can't emit anything except the three GET endpoints.
- `inspect` responses are reduced to an allowlist of fields before they reach any handler. Environment variables, mounts and labels are never exposed.

### 4.3 Host [OWNER A-04; REC R-13]
- `/proc` and `/sys` are bind-mounted read-only. No capabilities, no privileged mode, no host PID namespace, no `/dev`.
- Read-only data-mount binds only if the owner accepts them (B-09). Nexus never opens files under them; it only calls `statfs`.
- The privileged host agent is a separate design needing separate approval (D-17).

### 4.4 Sessions [OWNER A-10, A-16; REC R-01, R-02]
- 256-bit random credentials; SHA-256 hashes at rest; a new family at every login.
- Access 15 minutes. Refresh single-use and rotating, with reuse detection.
- **Absolute 7-day cap from login, never extended.**
- Revocation is immediate inside Nexus, and also logs out the Jellyfin token.
- Services holding offline-verified delegation tokens see at most 120 s of staleness. Execution requests are re-checked live.

### 4.5 Jellyfin credentials [OWNER A-14; REC R-10]
- **User tokens:**
  - AES-256-GCM encrypted, with a random 96-bit nonce per record;
  - key from the secrets file;
  - associated data = family id, so a token can't be moved to another session;
  - deleted on revocation or expiry.
- **Admin key (Phase 3):**
  - held only by the admin broker;
  - the broker requires `admin!` and writes an audit entry before every call;
  - never used for user-level reads.

### 4.6 Service tokens [OWNER A-05; REC R-03]
- Ed25519 JWTs with `alg` pinned to EdDSA (anything else rejected).
- `iss`, `aud`, `exp`, `iat` and `jti` are mandatory. Allowed clock skew is 30 s.
- Service identity is an Ed25519 public key registered in config. Assertion `jti` values are kept until expiry to block replay.
- Revocation:
  - expiry (5 minutes for service tokens, 120 s for delegation tokens);
  - `jti` denylist;
  - disabling a service;
  - `sid` liveness for delegated actions.
- Keys rotate every 90 days, with overlapping JWKS entries.

### 4.7 Authorization
- One default-deny route table. Startup fails if any route lacks an entry. A table-driven test covers every route × {anonymous, user, admin, stale admin, service}.
- Public and admin widget data are separate endpoints and DTOs, so an admin field can't leak into a user response through a missing filter.

### 4.8 Web
- CSP: `default-src 'self'; img-src 'self' data:; connect-src 'self'; frame-src 'none'; frame-ancestors 'none'; object-src 'none'; base-uri 'none'`.
- Also: `nosniff`, `Referrer-Policy: no-referrer`; HSTS at the TLS edge.
- No `{@html}` with downstream data.

### 4.9 Secrets [SPEC S-05; REC P-07, P-08]
- `secrets.yaml` (0600, owned by the Nexus UID) or Docker secrets.
- Covers: downstream keys, the Jellyfin token encryption key, Nexus signing keys (Phase 2) and the Jellyfin admin key (Phase 3).
- A redacting secret type: `String`, `MarshalJSON` and `slog.LogValuer` all return `[redacted]`.
- Never in images, repos, config, API responses or audit params. Secret scanning and `.gitignore` in the repo.

### 4.10 Audit [OWNER A-15; REC R-11]
- Append-only and hash-chained, with a verification endpoint.
- If the audit write fails, the admin action fails.
- Covers auth events, admin actions, denied admin attempts, revocations and service-token events.
- Retention 365 days.

### 4.11 Failure isolation [OWNER A-17; REC R-09]
- Timeouts, 2 MB caps, bulkheads and circuit breakers per integration. Cached widget values with `stale` and `unavailable` states.
- Fail closed for admin rights when Jellyfin can't confirm them; fail open for reading cached health data.
- Revocation and audit depend only on local storage.

### 4.12 Permanent guardrails [SPEC S-09, S-10]
- No format, delete-volume or remove-container code path. A test asserts that the routes and Docker methods are absent.
- `/mnt/internal-hdd` is always excluded (configurable list).

## 5. Reporting
Report security issues privately to the repository owner.

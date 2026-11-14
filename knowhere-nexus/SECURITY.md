# Nexus — Security Design

Status: **draft for approval.** Tags: **[SPEC]**, **[REC]**, **[OPEN]** (see DECISIONS.md).

Nexus is the most privileged application on knowhere. It holds every downstream API key, it can reach Docker, and later it can restart the host. A compromised Nexus means a compromised server, so this document comes before any code.

## 1. Assets
| Asset | Why it matters |
|---|---|
| Docker socket (through the proxy) | Container create or exec is equivalent to root on the host |
| Host power and privileged operations | Availability of every service and of family data |
| Downstream API keys (\*arr, Jellyseerr, qBittorrent, Immich, Pi-hole) | Full control of those services |
| Jellyfin credentials passing through sign-in | Family members' accounts |
| Nexus sessions and Milano tokens | Impersonation, including of admins |
| Family privacy (what people watch, photo counts) | Household trust |
| Immich data and media on the drives | Irreplaceable; the plan forbids formatting or removal |

## 2. Actors and threats
| Actor | Example threats |
|---|---|
| A device on the tailnet that isn't a family member's (shared node, guest) | Reaching Nexus at all; brute-forcing logins |
| A non-admin family member | Calling admin endpoints, seeing other users' viewing, changing the household theme |
| A malicious web page open in an admin's browser | CSRF on admin actions; clickjacking |
| A stolen phone with Milano | Using a long-lived token |
| A compromised downstream service | Sending crafted responses (XSS through titles; huge payloads) |
| A compromised Collector | Abusing introspection or Compose apply (C-01) |
| Supply chain (Go modules, npm, base image) | Malicious code inside Nexus |

## 3. Security-sensitive operations
| Operation | Role | Required controls |
|---|---|---|
| Sign-in, Quick Connect, token exchange | public | Rate limits per IP and per username; the same error for every failure; passwords never logged; validation done against Jellyfin on the server |
| Role assignment | — | Taken only from Jellyfin `Policy.IsAdministrator`, re-checked (P-06). There's no Nexus-side way to promote a user [SPEC S-04] |
| Container restart, stop, start | admin | Container allowlist enforced in Nexus; CSRF; audit; mount guardrail on start [SPEC S-11] |
| Container update or recreate | admin | **Blocked** until C-01, C-03 and C-06 are decided. Recreate is root-equivalent |
| Run a Jellyfin task | admin | Task id must come from Jellyfin's own list; audit |
| Host restart and shutdown | admin | Typed hostname [SPEC S-12]; recent re-auth [REC]; audit; only through a fixed-command host agent [OPEN C-04] |
| Mount re-check | admin | Read-only status refresh unless D-14 decides otherwise |
| Set the household theme | admin | Schema validation; audit |
| Per-user theme and layout | user | Owner only; schema and size limits (64 KB) |
| Wallpaper upload | user or admin | [OPEN D-06]. If allowed: type sniffing, re-encoding, size limit, served with `Content-Disposition` and `nosniff` |
| Jellyseerr requests | user | Identity mapping [OPEN D-11]; Nexus never forwards arbitrary paths |
| Introspection | Collector | Service credential; returns only active, user and role |
| Reading widgets | user or admin | Per-role filtering [OPEN D-01] |

## 4. Controls

### 4.1 Network [SPEC S-06, S-08]
- Nexus binds to loopback or a private Docker network only. Caddy and Tailscale Serve are the only way in [REC P-09].
- Optional Tailscale identity check [OPEN D-08].
- Downstream base URLs come only from config. No endpoint takes a URL, host or path from a client, which rules out SSRF [REC P-12].
- An outbound HTTP client with timeouts (connect 3 s, total 10 s), response size caps (2 MB) and no redirects to other hosts.

### 4.2 Docker [SPEC S-07]
- Nexus never mounts `/var/run/docker.sock` itself. Only the proxy does, read-only.
- docker-socket-proxy filters by **API section, not by container**. With `POST=1` and `CONTAINERS=1`, a client can *create* a privileged container. So:
  - Phase 1 runs the proxy read-only: `CONTAINERS=1, INFO=1, POST=0`; everything else 0.
  - Write phases allow the narrowest set the decision in C-03 permits. Nexus additionally enforces: a method and path allowlist (`/containers/{id}/(restart|stop|start)` only), container names from a config allowlist, and no create, exec, volumes, images, networks or build.
  - The proxy sits on an internal Docker network that only Nexus can reach.
- A contract test asserts that the Nexus Docker client can't emit any request outside its allowlist.

### 4.3 Host [OPEN C-04]
- Phase 1: only read-only `/proc` and `/sys` bind mounts. No privileged container, no host PID namespace, no `/dev`.
- Any later host agent has a fixed command set (`reboot`, `poweroff`, `smart-status`, `mount-status`), takes no arguments from Nexus beyond enumerated ids, listens on a Unix socket with peer-UID checks, and is audited on the host. **Installing it is an infrastructure change that needs owner approval.**

### 4.4 Authentication and sessions [SPEC S-03; REC P-05, P-06]
- Sessions are 256-bit random ids. Only a SHA-256 hash is stored.
- Cookie: `HttpOnly; Secure; SameSite=Strict`. A new id is issued at login.
- Milano bearer tokens: same format, listed and revocable by the user, carrying a device label.
- The admin role is re-checked against Jellyfin every 5 minutes and before every admin action. If Jellyfin is unreachable, admin actions fail closed; reads continue.
- Jellyfin access tokens from sign-in are discarded once the role is known, unless D-13 chooses per-user tokens. In that case they're encrypted at rest with a key from the secrets file.
- Lifetimes: [OPEN D-07].
- Login rate limits [REC]: 5 failures per username per 15 minutes; 20 per IP per 15 minutes.

### 4.5 Authorization
- One central route → role table. Default deny. Startup fails if any route lacks an entry.
- Ownership checks on every `/me/*` and per-user resource.
- Response filtering by role happens on the server; the UI hiding admin controls is not a control.

### 4.6 Web
- CSRF: `SameSite=Strict`, an `Origin` check against the configured origin, and a required `X-Nexus-CSRF` header on unsafe methods.
- Headers:
  - `Content-Security-Policy: default-src 'self'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; object-src 'none'; base-uri 'none'`. `frame-src` stays at `'none'` until C-02 is decided.
  - `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`.
  - HSTS is set by the TLS front end.
- Svelte's escaping by default. No `{@html}` with data from downstream services (titles, user names).

### 4.7 Secrets [SPEC S-05; REC P-07, P-08]
- Separate `secrets.yaml` (mode 0600, owned by the Nexus UID) or Docker secrets, never in the image, the repo or `config.yaml`.
- Loaded into memory at start. Never logged, never in API responses, never in audit params. A redaction helper wraps the secret type, and its `String()` and `MarshalJSON` return `"[redacted]"`.
- No secret editing through the UI in v1.
- The repo has secret scanning enabled, and `.gitignore` covers `secrets*.yaml`.

### 4.8 Audit and logging [REC P-10]
- Every admin action is recorded in the audit log: actor, action, target, sanitized params, result and request id.
- Structured logs (`slog`) carry no passwords, tokens, keys or Quick Connect secrets.

### 4.9 Supply chain [REC]
- Go modules pinned with `go.sum`; `npm ci` with a lockfile.
- Dependabot or Renovate.
- `govulncheck` and `npm audit` in CI.
- Distroless, non-root image, pinned by digest.
- The image is built only in GitHub Actions [SPEC NF-06].

### 4.10 Guardrails as security controls [SPEC S-09, S-10, S-11]
- The code has no route, client method or Docker call for formatting, deleting volumes or removing containers. Tests assert they're absent.
- `/mnt/internal-hdd` (configurable list, defaulting to that path) is always reported as excluded, and is never offered as a target.

## 5. Reporting security issues
Report privately to the repository owner. There's no public issue tracker for security.

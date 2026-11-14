# Nexus — API Specification

Revision 2 (14 Nov 2026), draft. Tags: **[OWNER]**, **[SPEC]**, **[REC]**, **[BLOCKER]**/**[LATER]** (see DECISIONS.md).
**Phase** is shown per endpoint. Anything marked *Out of scope* must not be implemented.

## 0. Conventions [REC P-15 unless marked]
- **Base and format:** base `/api/v1`; JSON; RFC 3339 UTC times; bytes as integers.
- **Browser credentials [OWNER A-10; REC R-01]:**
  - `nexus_access`: `HttpOnly; Secure; SameSite=Strict; Path=/`, host-only, 15 minutes.
  - `nexus_refresh`: same flags, `Path=/api/v1/auth/refresh`.
  - The absolute family lifetime is 7 days.
- **Milano credentials:** `Authorization: Bearer <access>`; refresh through the body.
- **Service credentials [OWNER A-05; REC R-03]:** `Authorization: Bearer <service access JWT>`, plus `X-Nexus-Delegation: <user delegation JWT>` when acting for a user.
- **CSRF:** cookie-authenticated unsafe methods need `X-Nexus-CSRF: 1` and an `Origin` equal to the configured origin.
- **Errors:** `{"error":{"code":"…","message":"…"}}`.
  - `unauthenticated` 401 · `forbidden` 403 · `reauth_required` 403 · `not_found` 404 · `validation_failed` 422 · `conflict` 409 · `rate_limited` 429 · `upstream_unavailable` 502 · `not_configured` 503 · `internal` 500.
- **Roles:**
  - `public`;
  - `user`;
  - `admin` (a Jellyfin administrator);
  - `admin!`: admin verified against Jellyfin in the last 15 minutes [REC R-09/R-10];
  - `admin!+reauth`: `admin!` plus a password re-auth in the last 10 minutes [REC P-11];
  - `service:<scope>`.
- **Default deny:** any route not listed here is denied.
- **Never returned:** downstream keys, Jellyfin tokens, internal service URLs, container environment variables.
- **Widget envelope [OWNER A-17]:** `{"status":"ok|stale|unavailable|not_configured","updatedAt":"…","data":{…}}`.

---

## 1. Health
| Method | Path | Role | Phase | Response |
|---|---|---|---|---|
| GET | `/healthz` | public | 1 | `{"status":"ok"}` only |
| GET | `/api/v1/admin/status` | admin | 1 | `{"version","integrations":[{"name","configured","breaker":"closed|open|half_open","lastSuccessAt","lastError"}]}` [OWNER A-17] |

## 2. Authentication and sessions [SPEC S-03/S-04; OWNER A-10, A-16]

### `POST /api/v1/auth/login` — public, rate limited — Phase 1
- Request: `{"username","password"}`.
- Nexus authenticates with Jellyfin and keeps the user's Jellyfin token, encrypted [OWNER A-14].
- `200` sets both cookies and returns `{"user":{"id","name","role"},"absoluteExpiresAt"}`.
- `401` for any credential failure (one message). `502` when Jellyfin can't be reached.

### `POST /api/v1/auth/quickconnect` — public, rate limited — Phase 1
`200 {"code","pollId"}`. The Jellyfin secret stays on the server.

### `GET /api/v1/auth/quickconnect/{pollId}` — public, rate limited — Phase 1
`200 {"state":"pending|authorized|expired"}`. On `authorized`, the response matches login.

### `POST /api/v1/auth/refresh` — refresh credential — Phase 1
- Browser: the cookie. Milano: `{"refreshToken"}`.
- Rotates both credentials. It never extends `absoluteExpiresAt`.
- A reused refresh token → `401`, the family is revoked, and the event is audited [REC R-01].
- `200` → `{"user","absoluteExpiresAt"}` (Milano also gets `{"accessToken","refreshToken","accessExpiresAt"}`).

### `POST /api/v1/auth/logout` — user — Phase 1
Revokes the current family and logs out its Jellyfin token. `204`.

### `GET /api/v1/auth/session` — user — Phase 1
`{"user","role","roleVerifiedAt","absoluteExpiresAt","authMethod","tailscaleLogin":<string|null>}`

### `POST /api/v1/auth/reauth` — user — Phase 1
`{"password"}`. Verified with Jellyfin; marks the family as freshly authenticated. `204`.

### Own sessions — user — Phase 1 [OWNER A-16]
| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/me/sessions` | `[{"id","kind","deviceLabel","tailscaleLogin","createdAt","lastSeenAt","absoluteExpiresAt","current":bool}]` |
| DELETE | `/api/v1/me/sessions/{id}` | Revokes one of the caller's own families. `204`; another user's id → `404` |
| POST | `/api/v1/me/sessions/revoke-others` | `204` |

### Admin session control — Phase 1 [OWNER A-16]
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/api/v1/admin/sessions?user=` | admin | Every family, filterable by user |
| DELETE | `/api/v1/admin/sessions/{id}` | admin! | Audited |
| POST | `/api/v1/admin/users/{jellyfinUserId}/revoke-sessions` | admin!+reauth | Revokes every family of that user. Audited |

### `POST /api/v1/auth/token-exchange` — public, rate limited — Phase 3 [SPEC F-25]
- Request: `{"jellyfinToken","deviceLabel"}`. Validated with Jellyfin `/Users/Me`.
- Returns a Milano family: `{"accessToken","refreshToken","accessExpiresAt","absoluteExpiresAt","user"}`.
- What happens after the 7-day cap is [LATER D-15].

### `GET /api/v1/auth/verify` — Caddy only — Phase 1 (only used if B-07 authorizes a `nexus`-mode route)
- Forward-auth check for authorized routes [OWNER A-02; REC R-06]. Caddy sends the original cookies and `X-Nexus-Route: <route id>`.
- Unknown route, or a user not allowed on it → `403`. No valid access credential → `401` (Caddy redirects to `/login?next=`).
- `200` with response headers `X-Nexus-User-Id`, `X-Nexus-Role` and `X-Nexus-Delegation` (a JWT with `aud` = the route's service, Phase 2).
- Reachable only from the Caddy address.

## 3. Service tokens [OWNER A-05; REC R-03] — Phase 2
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/api/v1/.well-known/jwks.json` | public | Nexus Ed25519 public keys (`kid`, `kty: OKP`, `crv: Ed25519`, `x`) |
| POST | `/api/v1/service/token` | service assertion | Form: `grant_type=client_credentials`, `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`, `client_assertion=<JWT signed by the service: iss=sub=<id>, aud=<this URL>, exp≤60s, jti>`, `scope=<space-separated>`. → `{"access_token","token_type":"Bearer","expires_in":300,"scope"}`. Scopes beyond the registered allowance → `403`. A reused assertion `jti` → `401`. |
| GET | `/api/v1/admin/services` | admin | Registered services, scopes and enabled state (no keys) |
| POST | `/api/v1/admin/services/{id}/disable` / `/enable` | admin!+reauth | Disabling rejects all of that service's tokens at once. Audited |
| POST | `/api/v1/admin/service-tokens/revoke` | admin! | `{"jti"}` → denylisted until expiry. Audited |

**Token claims:**

| Token | `iss` | `aud` | `sub` | Other | TTL |
|---|---|---|---|---|---|
| Service access | `nexus` | `nexus` | `svc:<id>` | `scope`, `jti`, `kid` | 300 s |
| User delegation | `nexus` | `<service id>` | `user:<jellyfin id>` | `role`, `sid`, `jti`, `kid` | 120 s |

## 4. Current user — Phase 1 unless noted
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/api/v1/me` | user | `{"user","role","prefs":{"theme","homeLayout"}}` |
| PUT | `/api/v1/me/home-layout` | user | `{"tiles":[catalogId],"widgets":[name]}`; maximum 64 KB; unknown ids rejected |
| PUT | `/api/v1/me/theme` | user | Phase 3 [SPEC F-21]; a `prism/0` theme or `null` |
| PUT | `/api/v1/me/desktop-layout` | user | Phase 5 |

## 5. Themes [SPEC F-20; REC R-12]
`prism/0`:
```json
{"schema":"prism/0","seed":"#RRGGBB","mode":"light|dark|system","radius":0-32,"font":"<allowlisted family>","density":"comfortable|compact","wallpaper":null,"materialYou":false}
```

| Method | Path | Role | Phase |
|---|---|---|---|
| GET | `/api/v1/themes` | user | 1 (built-in only); packs in 5 |
| GET | `/api/v1/themes/household` | user | 1 |
| PUT | `/api/v1/themes/household` | admin! | 1, audited |
| GET | `/api/v1/themes/effective` | user | 1: `{"theme","from":"user|household|builtin"}` |

## 6. Launcher catalog [OWNER A-09; REC R-07]
| Method | Path | Role | Phase | Notes |
|---|---|---|---|---|
| GET | `/api/v1/apps` | user | 1 | `[{"id","name","icon","href":<route URL or null>,"available":bool}]`. Entries with `audience: admin` are filtered for users. `href` is set only for routes in the authorized registry [OWNER A-02] |
| POST | `/api/v1/admin/catalog/reload` | admin! | 1 | Re-reads `catalog.yaml`; `422` with errors on an invalid file, keeping the old catalog. Audited |
| PUT | `/api/v1/service/catalog` | service:`catalog:publish` + delegation (admin) | 2 | Collector publishes the catalog [OWNER A-06]. Audited |

## 7. Widgets [OWNER A-07, A-08, A-17; REC R-08]
| Method | Path | Role | Phase | `data` |
|---|---|---|---|---|
| GET | `/api/v1/widgets/system` | user | 1 | `{"cpuPercent","memUsed","memTotal","load":[n,n,n],"tempC":<n|null>,"uptimeSec"}` |
| GET | `/api/v1/widgets/storage` | user | 1 | `{"mounts":[{"label","health":"ok|degraded|unavailable","usedPercent":<n|null>}]}`. Excluded mounts are omitted |
| GET | `/api/v1/admin/widgets/storage` | admin | 1 | `{"mounts":[{"label","path","mounted","mode":"ro|rw","free":<n|null>,"total":<n|null>,"excluded":bool,"smart":"unavailable"}]}`. `free`/`total` are null unless B-09 allows them; `smart` stays `unavailable` until D-17 |
| GET | `/api/v1/widgets/services` | user | 1 | `{"apps":[{"catalogId","name","up":bool}]}` |
| GET | `/api/v1/admin/widgets/services` | admin | 1 | `{"containers":[{"name","image","state","restartCount","memBytes"}]}` |
| GET | `/api/v1/admin/widgets/now-playing` | admin | 3 | `{"streams":[{"user","title","client","playMethod","transcodeReasons"}]}` (admin broker, D-19) |
| GET | `/api/v1/admin/widgets/photos` | admin | not phased | Immich aggregates (B-05c) |
| GET | `/api/v1/admin/widgets/dns` | admin | not phased | Pi-hole aggregates (B-05c) |
| GET | `/api/v1/admin/widgets/downloads` | admin | 4 | qBittorrent |
| GET | `/api/v1/widgets/upcoming` | user | 4 | Radarr and Sonarr calendar |
| POST | `/api/v1/admin/storage/recheck` | admin | 1 | Read-only refresh now [OWNER A-04]; audited |

### `GET /api/v1/events` — user — Phase 1 [REC P-04]
- SSE events: `widget.<name>` (the public or admin variant, by role), `session.revoked` and `catalog.changed`.
- Each widget is published independently, so one failing widget doesn't affect the others.

## 8. Audit [OWNER A-15]
| Method | Path | Role | Phase |
|---|---|---|---|
| GET | `/api/v1/admin/audit?before=<seq>&limit=<≤200>&actor=&action=` | admin | 1 |
| GET | `/api/v1/admin/audit/verify` | admin | 1: checks the hash chain → `{"ok":bool,"brokenAt":<seq|null>}` |

## 9. Jellyfin admin operations — Phase 3 [SPEC F-14; OWNER A-14]
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/api/v1/admin/jellyfin/tasks` | admin! | Uses the admin broker (D-19) |
| POST | `/api/v1/admin/jellyfin/tasks/{id}/run` | admin! | `{id}` must be in Jellyfin's list; audited |

## 10. Requests and \*arr — Phase 4 [SPEC F-27; LATER D-11]
| Method | Path | Role |
|---|---|---|
| GET | `/api/v1/requests/search?q=` (≤100 characters) | user |
| POST | `/api/v1/requests` `{"mediaType","tmdbId","seasons"?}` | user |
| GET | `/api/v1/requests` | user (own); `?all=true` for admin |
| GET | `/api/v1/admin/arr/queue` | admin |
| GET | `/api/v1/admin/arr/calendar?from=&to=` (≤60 days) | admin |

## 11. Out of scope until separately approved — do not implement
| Area | Reserved paths | Gate |
|---|---|---|
| Container restart, stop, start, update, create, delete | `/api/v1/admin/containers/*` (writes) | OWNER A-03 → D-18 |
| Deployment requests from Collector | `/api/v1/service/deploy/*`, scope `deploy:request` | OWNER A-01, A-03 → D-16 |
| Host restart and shutdown, SMART, remount | `/api/v1/admin/host/*` | OWNER A-04 → D-17 |
| Format, volume delete | none, ever | SPEC S-09 |

# Nexus — API Specification (v1, draft)

Status: **draft for approval.** Tags: **[SPEC]** specified, **[REC]** recommended, **[OPEN]** see DECISIONS.md.

## Conventions [REC]
- **Base:** `/api/v1`. JSON only (`Content-Type: application/json`). UTF-8.
- **Times:** RFC 3339 UTC. **Sizes:** bytes (integer). **Percentages:** 0–100 (float).
- **Auth:**
  - Browsers use the `nexus_session` cookie (`HttpOnly; Secure; SameSite=Strict; Path=/`).
  - Milano and services use `Authorization: Bearer <nexus token>`.
- **CSRF:** every request that isn't GET or HEAD and is authenticated by cookie must carry the `X-Nexus-CSRF: 1` header, and its `Origin` must match the configured public origin.
- **Errors:** `{"error":{"code":"<snake_case>","message":"<human text>"}}`.
  - Codes: `unauthenticated` (401), `forbidden` (403), `not_found` (404), `validation_failed` (422), `conflict` (409), `guardrail_refused` (409), `reauth_required` (403), `rate_limited` (429), `upstream_unavailable` (502), `not_configured` (503), `internal` (500).
- **Roles:**
  - `public`: no auth.
  - `user`: any signed-in Jellyfin user.
  - `admin`: a Jellyfin administrator.
  - Every route below lists its minimum role. Unlisted routes are denied.
- **Secrets:** no response ever contains a downstream API key, a Jellyfin token, or a service's internal URL.

---

## 1. Health

### `GET /healthz` — public
`200 {"status":"ok"}`. Nothing else (no version, no dependency status).

### `GET /api/v1/status` — admin [REC]
Nexus version and the health of each integration: `{"version":"…","integrations":[{"name":"jellyfin","configured":true,"reachable":true,"checkedAt":"…"}]}`.

---

## 2. Authentication [SPEC S-03, S-04]

### `POST /api/v1/auth/login` — public, rate limited
Request: `{"username":"string","password":"string"}`
Nexus calls Jellyfin `POST /Users/AuthenticateByName`, reads `Policy.IsAdministrator`, creates a session and sets the cookie.
- `200` → `{"user":{"id":"<jellyfin user id>","name":"…","role":"admin|user"}, "csrf":"header-name"}`
- `401 unauthenticated` covers both a bad password and an unknown user (same message).
- `502 upstream_unavailable` when Jellyfin can't be reached.

### `POST /api/v1/auth/quickconnect` — public, rate limited
Starts Quick Connect through Jellyfin.
`200` → `{"code":"123456","pollId":"<opaque Nexus id>"}`. The Jellyfin secret stays on the server.

### `GET /api/v1/auth/quickconnect/{pollId}` — public, rate limited
`200` → `{"state":"pending|authorized|expired"}`. On `authorized`, Nexus completes the Jellyfin authentication, creates the session, sets the cookie and returns the same body as login.

### `POST /api/v1/auth/token-exchange` — public, rate limited [SPEC F-25]
For Milano. Request: `{"jellyfinToken":"…","deviceLabel":"Pixel 8"}`
Nexus validates the token by calling Jellyfin `GET /Users/Me` with it. The token is not stored.
`200` → `{"token":"<opaque>","expiresAt":"…","user":{…}}`. Lifetime: [OPEN D-07].

### `POST /api/v1/auth/logout` — user
Revokes the current session or token. `204`.

### `GET /api/v1/auth/session` — user
`200` → `{"user":{…},"role":"…","expiresAt":"…","authMethod":"password|quickconnect|exchange"}`.

### `GET /api/v1/auth/sessions` / `DELETE /api/v1/auth/sessions/{id}` — user
List or revoke the caller's own sessions and Milano tokens. Admins may add `?all=true` [REC].

### `POST /api/v1/auth/reauth` — user [REC P-11]
Request: `{"password":"…"}`. Marks the session as freshly authenticated, which some admin actions require.

### `POST /api/v1/auth/introspect` — service [OPEN C-05]
Collector only, authenticated with a Collector service credential from Nexus secrets.
Request: `{"session":"<cookie value or bearer>"}` → `{"active":true,"user":{…},"role":"…"}`.

---

## 3. Current user and preferences

### `GET /api/v1/me` — user
`{"user":{…},"role":"…","prefs":{"theme":<PrismTheme|null>,"homeLayout":<HomeLayout|null>}}`

### `PUT /api/v1/me/theme` — user [SPEC F-21, S-13]
Body: a `PrismTheme`, or `null` to follow the household default. `200` returns the stored value.

### `PUT /api/v1/me/home-layout` — user [REC]
Body: `{"tiles":["<appId>",…],"widgets":["system","storage",…]}`.

### `PUT /api/v1/me/desktop-layout` — user [SPEC F-23, Phase 5]
Body: `{"windows":[{"appId":"…","x":0,"y":0,"w":800,"h":600,"z":1,"minimized":false}],"wallpaper":"<id|null>"}`.

---

## 4. Themes [SPEC F-20–F-22]

`PrismTheme` (pinned to a Prism schema version; the fields come from the Plan, the exact shape comes from Prism):
```json
{ "schema": "prism/1", "seed": "#6750A4", "mode": "light|dark|system",
  "radius": 12, "font": "string", "density": "comfortable|compact",
  "wallpaper": "<id|null>", "materialYou": false }
```
`materialYou` only means something to Milano (S-15); Nexus stores it and ignores it.

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/api/v1/themes` | user | Built-in and installed packs: `[{"id","name","source":"builtin|pack","theme":PrismTheme}]` |
| GET | `/api/v1/themes/household` | user | The household default |
| PUT | `/api/v1/themes/household` | admin | Set the household default; audited |
| GET | `/api/v1/themes/effective` | user | Resolved by S-15 (without Material You): `{"theme":PrismTheme,"from":"user|household|builtin"}` |

How theme packs get installed: [OPEN D-05].

---

## 5. Launcher

### `GET /api/v1/apps` — user
`[{"id","name","icon","open":"link|window|native","href":"…"}]`. The source of entries is [OPEN D-02], and `window` behaviour is [OPEN C-02]. Admin-only entries are filtered out for users.

---

## 6. Widgets

Every widget response is wrapped like this:
```json
{ "status": "ok|stale|unavailable|not_configured", "updatedAt": "…", "data": { … } }
```
The role per widget is [OPEN D-01]. The defaults below are a **proposal**.

| Method | Path | Proposed role | `data` |
|---|---|---|---|
| GET | `/api/v1/widgets/system` | user | `{"cpuPercent","memUsed","memTotal","load":[1,5,15],"tempC":<num|null>,"uptimeSec"}` |
| GET | `/api/v1/widgets/storage` | user (summary), admin (detail) | `{"mounts":[{"path","label","mounted","mode":"ro|rw","free","total","smart":"ok|warning|failing|unknown","excluded":bool}]}` |
| GET | `/api/v1/widgets/services` | user (up/down), admin (details) | `{"containers":[{"name","state","restartCount","memBytes"}]}` |
| GET | `/api/v1/widgets/now-playing` | admin (D-01, D-13) | `{"streams":[{"user","title","client","playMethod":"direct_play|direct_stream|transcode","transcodeReasons":[]}]}` |
| GET | `/api/v1/widgets/photos` | user | `{"assets":int,"libraryBytes":int}` (Immich; exact metric fields depend on what the Immich API exposes) |
| GET | `/api/v1/widgets/dns` | user | `{"queries":int,"blockedPercent":float}` (Pi-hole; the time window comes from Pi-hole's summary) |
| GET | `/api/v1/widgets/downloads` | admin | `{"items":[{"name","progress","state","dlSpeed","ulSpeed","eta"}]}` — Phase 4 |
| GET | `/api/v1/widgets/upcoming` | user | `{"items":[{"title","kind":"movie|episode","date","source":"radarr|sonarr"}]}` — Phase 4 |

Excluded mounts (`/mnt/internal-hdd`) come back with `excluded:true`, and are never counted in free or total figures [SPEC F-18].

### `GET /api/v1/events` — user [REC P-04]
Server-Sent Events stream. Event names: `widget.<name>` with the wrapped payload above, and `session.revoked`. Only widgets the caller may see are sent.

---

## 7. Admin — containers [SPEC F-13; constraints in OPEN C-01, C-03, C-06]

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/admin/containers` | Full list, including image and created time |
| POST | `/api/v1/admin/containers/{name}/restart` | Allowlisted names only; audited |
| POST | `/api/v1/admin/containers/{name}/stop` | Allowlisted; audited |
| POST | `/api/v1/admin/containers/{name}/start` | [OPEN D-04]; the write-heavy mount guardrail applies [SPEC F-17] → `409 guardrail_refused` with `{"mount":"…","reason":"missing|read_only"}` |
| POST | `/api/v1/admin/containers/{name}/update` | [OPEN C-01/C-03/C-06]. **Not to be implemented until decided** |

Responses: `202 {"actionId","status":"accepted"}`. The result arrives over SSE as `admin.action`.
There are no routes for removing a container, removing a volume, formatting, exec or logs streaming [SPEC S-09; logs not specified].

## 8. Admin — Jellyfin tasks [SPEC F-14]
| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/admin/jellyfin/tasks` | `[{"id","name","category","state","lastResult","lastRunAt"}]` |
| POST | `/api/v1/admin/jellyfin/tasks/{id}/run` | Audited. `{id}` must exist in Jellyfin's task list |

## 9. Admin — host and storage [SPEC F-15, F-16; OPEN C-04, D-14]
| Method | Path | Notes |
|---|---|---|
| POST | `/api/v1/admin/storage/recheck` | Refreshes mount status now (proposal: read-only, D-14) |
| POST | `/api/v1/admin/host/restart` | Body `{"confirm":"<hostname>"}`; requires a recent re-auth (P-11); audited |
| POST | `/api/v1/admin/host/shutdown` | Same as above |

A wrong confirmation → `422 validation_failed`. A stale auth → `403 reauth_required`. No host agent configured → `503 not_configured`.

## 10. Requests and \*arr — Phase 4 [SPEC F-27; OPEN D-11]
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/api/v1/requests/search?q=` | user | Search Jellyseerr; `q` limited to 100 chars |
| POST | `/api/v1/requests` | user | `{"mediaType":"movie|tv","tmdbId":int,"seasons":[int]?}` |
| GET | `/api/v1/requests` | user | The caller's requests; admins can add `?all=true` |
| GET | `/api/v1/admin/arr/queue` | admin | Radarr and Sonarr queues, merged |
| GET | `/api/v1/admin/arr/calendar?from=&to=` | admin | Maximum 60-day range |
| GET | `/api/v1/admin/downloads` | admin | qBittorrent torrents (read-only; actions not specified) |

## 11. Audit
### `GET /api/v1/admin/audit?before=&limit=` — admin [REC P-10]
Paged list of audit entries, with secrets redacted.

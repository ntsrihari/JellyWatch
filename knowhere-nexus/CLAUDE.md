# CLAUDE.md — Knowhere Nexus

Nexus is the dashboard and API at `knowhere/`, part of the Knowhere platform. The product spec is the **Knowhere Platform Plan** doc. This folder holds Nexus's engineering documents:

| File | Purpose |
|---|---|
| ARCHITECTURE.md | Requirements (with sources), components, data model, dependencies |
| API_SPEC.md | The `/api/v1` contract used by the web UI, Milano and Collector |
| SECURITY.md | Threat model, sensitive operations, controls |
| TASKS.md | Phased task list and test plan |
| DECISIONS.md | Specified, proposed and open decisions |

## Current state
**Planning only. Do not implement until Gate 0 in TASKS.md is cleared.** Owner-approved decisions are tagged [OWNER] in DECISIONS.md; target repo is `ntsrihari/knowhere-nexus` (A-13).

## Rules for working on Nexus
1. **Don't invent requirements.** Every feature must trace to the Platform Plan or the brief (IDs F-xx, NF-xx, S-xx). Anything else is a recommendation, marked **[REC]**, and needs approval.
2. **Open decisions block their tasks.** Don't pick an answer to a C-xx or D-xx item yourself. Ask.
3. **Don't touch production infrastructure.** No changes on knowhere: Compose files, Caddy, Tailscale, the socket proxy, a host agent, mounts, drives. Tasks marked **[INFRA]** need the owner's explicit go-ahead each time.
4. **Assume nothing about the infrastructure.** Service URLs, mounts, container names and hostnames come from config, never from hardcoded values.
5. **Never write, log or return secrets.** Use the redacting secret type. Never commit `secrets*.yaml`.
6. **Default-deny authorization.** Every new route needs an entry in the authz table and in the authz table test.
7. **Docker stays allowlisted.** No create, exec, volume, image or network calls without an approved decision.
8. **No format, delete-volume or remove-container features, ever** (S-09).

## Stack (specified, plus the proposed libraries pending approval)
- Go backend (single binary) — S-01. Proposed: Go 1.23+, chi, slog, modernc SQLite.
- SvelteKit, `adapter-static`, embedded in the binary — S-02 / P-03.
- Jellyfin auth; Jellyfin admin = Nexus admin — S-03, S-04.

## Commands
None yet. Add build, test and lint commands here once the skeleton (T-1.x) exists.

## Related components (separate repos; don't implement them here)
Collector (store), Milano (Android app), Prism (themes). See ARCHITECTURE.md §7 for the interfaces.

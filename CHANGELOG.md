# CHANGELOG — `rasa.module.connectors`

Reverse-chronological. Each entry is a version bump.

---

## 0.1.3 — 2026-07-09

### Element identity layer (canon SA-025)

- Added `rasa.identity` ("the RasaOS module for connectors and integrations"); `bin/init` generates `.claude/rasa-identity.md` from it every install + stamps project-owned `.claude/rasa-deployment.md`; ships `/whoami`; CLAUDE.md "Who you are" header.

## 0.1.2 — 2026-07-09

### Added generic `/sync` + `/promote` + `/kit`-aware `bin/init` (canon SA-024)

- `bin/init` now clones the Element source into `<project>/kit/<element>/`; `/sync` smart-pulls upstream, `/promote` smart-pushes local edits back upstream (both directory-mirror → installed into consumers).

## 0.1.1 — 2026-07-09

### `parent_kind` → `[domain, tenant]` (canon SA-023)

- The `orchestrator` kind was folded into `tenant`; this module now mounts into a tenant or a domain (`requires.parent_kind: ["domain", "tenant"]`, was `["domain", "orchestrator"]`).

## 0.1.0 — 2026-07-04 — INITIAL

The first-party **reference connector pack** of the Connections track
(canon tasks SA-020/SA-031; born from the 2026-07-03/04 design
session, same session as `rasa.module.http` v0.1.0).

**Ships:**

- `seed/connections/github/` — GitHub REST v3: bearer auth by NAME
  (`github-pat`; unauthenticated public reads work, rate-limited),
  4 curated safe ops (`get-rate-limit` health probe, `get-repo`,
  `list-issues`, `search-repos`) + `create-issue` (`safe: false`),
  page-style pagination, 9-section SKILL.md (auth modes, rate-limit
  etiquette, `X-GitHub-Api-Version` pin guidance, known-errors table).
- `seed/connections/open-meteo/` — the **no-auth worked example**
  (`auth` omitted entirely): `geocode` (health), `forecast`,
  `current-weather` across two allowlisted hosts, 9-section SKILL.md.
- `content/README.md` — the seed-vs-content ownership rationale + the
  authoring guide for new connectors.

**Verification:** every `safe: true` operation live-tested through
`rasa.module.http` v0.1.0's MCP server with the exact `example_args`
shipped in the specs; `bin/check-manifest` GREEN; schema conformance
sweep GREEN; `bin/init` smoke-tested (specs land project-owned,
skip-if-exists).

**Decisions recorded:**

- Specs ship as **seeds** (project-owned after install), not content —
  under contract 1.3.0 the consumer's copy is their editable instance.
  Re-homes to `content/connections/` (Element-owned definitions +
  kernel instance records) when the SA-031 kernel loader lands.
- One pack, not per-service Elements — smallest coherent shape at
  n=2 connectors; splits per-service when the catalog gallery UX
  demands per-service pull.
- Findings surfaced for SA-031 authoring: `auth` must be optional
  (no-auth APIs exist — Open-Meteo); multi-host APIs need either
  absolute-URL op paths or a per-op host override (Open-Meteo's
  geocoding host).

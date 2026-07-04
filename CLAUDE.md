# CLAUDE.md — `rasa.module.connectors`

You are working inside **`rasa.module.connectors`** — the first-party
reference connector pack of the RasaOS Connections track. Canon-shaped
`module` Element, contract 1.3.0.

## What this Element is

Live-tested `connection.json` specs + per-API `SKILL.md` guides
(GitHub, Open-Meteo) seeding project-owned into consumers'
`.claude/connections/`. These are the **worked examples** of the
locked SA-031 connection-spec shape — connector authors copy them, so
exemplary quality is the point.

## Layout

- `seed/connections/<name>/{connection.json,SKILL.md}` — the payload.
  Seeds (skip-if-exists), NOT content — see `content/README.md` for
  why (project ownership under contract 1.3.0; re-homes to
  `content/connections/` when the SA-031 kernel loader lands).
- `content/README.md` — the seed-vs-content rationale + authoring
  guide (opt-in; not installed).
- `rasa.json` — manifest; every seed file registered per-file.

## Conventions (non-negotiable)

- **Field names are locked by canon task SA-031 §1** (mirrored in
  module-http's templates). No invented fields; omit what doesn't
  apply (e.g. `auth` on a no-auth API). Divergence is drift — SA-031
  wins.
- Names + op ids match `^[a-z0-9-]+$`; `summary` required per op;
  `params` = one flat JSON-Schema object; `safe` omitted ⇒ false —
  declare it explicitly.
- **Every `safe: true` op is live-tested before shipping**, through
  `rasa.module.http`'s server, with the spec's exact `example_args`.
  `safe: false` ops are never live-tested (no mutations, no creds).
- Secrets: NAMES only, everywhere. No credential value ever appears
  in a spec, a SKILL.md, a test command, or chat.
- `bin/check-manifest` GREEN before any commit; bump per the
  workspace module release flow (VERSION + rasa.json + CHANGELOG +
  tag).

## Do-nots

- Don't convert seeds to `directory-mirror` content — that changes
  the ownership contract and clobbers consumer edits on upgrade.
- Don't add ops you haven't live-tested (or can't — then mark
  `safe: false` and say so in the SKILL.md).
- Don't harden the module-http pairing into `requires.elements[]` —
  it's a soft reference; the specs are plain data.

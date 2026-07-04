# RasaOS Module · Connectors

`rasa.module.connectors` — the **first-party reference connector
pack** of the Connections track. Ready-made, live-tested
`connection.json` specs + per-API `SKILL.md` guides that install
**project-owned** into `.claude/connections/` and pair with
[`rasa.module.http`](https://github.com/RasaOS/module-http) today.

| Connection | Auth | Ops | Notes |
|---|---|---|---|
| `github` | bearer by NAME (`github-pat`); public reads work unauthenticated | `get-rate-limit` `get-repo` `list-issues` `search-repos` (+ `create-issue`, safe:false) | page-style pagination; 60/hr unauth vs 5000/hr with a PAT |
| `open-meteo` | **none** (the no-auth worked example) | `geocode` `forecast` `current-weather` | two allowlisted hosts; no key needed for non-commercial use |

## Quickstart

```sh
# from your project root, after pulling this Element
elements/module-connectors/bin/init

# then allow the hosts where rasa.module.http's server runs:
#   HTTP_ALLOWED_HOSTS=api.github.com,api.open-meteo.com,geocoding-api.open-meteo.com
# and (optionally, for authenticated GitHub):
#   HTTP_AUTH_API_GITHUB_COM="Authorization: Bearer ${GITHUB_TOKEN}"
```

The specs are **yours after install** (`skip-if-exists` — upgrades
never overwrite your edits): tune allowlists, set `account_label`,
add curated operations.

## What these specs are

Each spec follows the **locked SA-031 connection-spec shape** exactly
— these are the worked examples every connector author copies
(`github` = header-auth + pagination + a write op; `open-meteo` =
no-auth + multi-host). Every `safe: true` operation was live-tested
through `rasa.module.http`'s MCP server with the exact `example_args`
shipped in the spec. Under the future kernel Connections runtime
(canon task SA-031) the same specs become typed `conn__` tools with
kernel-enforced allowlists and vault-resolved credentials; until
then, enforcement is `rasa.module.http`'s self-policing tier — see
its honesty notes.

Secrets discipline: specs carry secret **names only**; values live in
env / the kernel vault, never in these files, never in chat.

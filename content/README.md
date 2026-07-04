# content/ — `rasa.module.connectors`

This pack's payload lives in `seed/connections/`, not here — a
deliberate inversion of the usual module layout, documented so the
next author doesn't "fix" it.

## Why the specs are seeds, not content

Under contract 1.3.0 + `rasa.module.http`, the consumer's
`.claude/connections/<name>/` copy is their **editable instance**:
projects tune allowlists, set `account_label`, add curated operations,
and extend the SKILL.md with project-specific recipes. `bin/init`
supports project ownership via the `skip-if-exists` seed policy —
files are copied once and never overwritten on Element upgrade.

`content/` + `directory-mirror` would be the opposite contract
(Element-owned, refreshed on upgrade), which would clobber those edits.

## The SA-031 future

Canon task SA-031 (Connections runtime) gives connector Elements the
definition≠instance split properly: specs live at
`content/connections/<name>/` as Element-owned **definitions**, and
per-tenant **instance** state (status, enable/disable, config overlay,
secret refs) lives in kernel records — no file-grain compromise
needed. When the kernel loader lands, this pack re-homes its specs to
`content/connections/` in a major rev. Until then, seeds are the
honest mapping.

## Authoring a new connector from these examples

1. Copy the closest example (`github` = header-auth API with
   pagination + a write op; `open-meteo` = no-auth API, multi-host).
2. Follow the locked field names from SA-031 §1 (mirrored in
   `rasa.module.http`'s templates at
   `.claude/http/templates/connection.rest.json.template`). Names and
   op ids match `^[a-z0-9-]+$`; `summary` is required on every op;
   `params` is one flat JSON-Schema object; `safe` omitted means
   false.
3. **Live-test every `safe: true` operation** through
   `rasa.module.http`'s server with your `example_args` before
   shipping — the example args in these specs are the exact arguments
   that were tested.
4. Register the connector in the catalog
   (`registry-public/connectors.json`).

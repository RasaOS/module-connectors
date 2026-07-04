---
name: open-meteo
description: Work with the Open-Meteo weather API through the `open-meteo` connection (.claude/connections/open-meteo/connection.json) using the http module's MCP tools (http_request, http_get_json, http_paginate). Triggered when the user asks about weather data — e.g. "what's the weather in Birmingham right now", "will it rain in Austin this weekend".
---

# Open-Meteo — connection skill

## 1. When to use this connection (and when NOT)

- Use for: current conditions, hourly/daily forecasts (up to 16 days), and
  place-name → coordinate geocoding, anywhere in the world.
- Prefer over general web search when: the user wants actual weather numbers
  (temperature, wind, precipitation) — this returns structured data, not prose.
- Do NOT use for: historical weather archives, air quality, marine, or flood
  data — Open-Meteo serves those from OTHER hostnames (`archive-api.`,
  `air-quality-api.`, etc.) that are not in this connection's allowlist.
  Also not for severe-weather alerts/warnings — Open-Meteo has no alerts
  endpoint at all.

## 2. Auth note

**This API needs no credential** — that is the point of this reference
connection: the `auth` field is omitted entirely from connection.json, and
the http server injects nothing for these hosts. Free for non-commercial
use; commercial keys exist (see `setup_help`) but use a different hostname
and would be a separate connection. If you ever see a `missing_auth` error
on these hosts, something is misconfigured server-side — report it to the
operator; never ask the user for a key.

## 3. ID formats & lookups

- Location: `latitude` + `longitude` as WGS84 decimal degrees — the ONLY
  location identifier `forecast` and `current-weather` accept. Obtained via
  `geocode`.
- Weather variables: lowercase snake_case names with a height/level suffix
  (`temperature_2m`, `wind_speed_10m`), passed as comma-separated strings.
- Gotcha: place names are not ids — always geocode first, and when a name is
  ambiguous (there are Birminghams in the US and the UK) disambiguate with
  `countryCode` or by showing the user the candidates. Unknown variable names
  are a 400 (`reason` names the bad parameter), not silently ignored.

## 4. Pagination idiom

- None — this connection declares no `pagination` field. Forecast size is
  controlled by `forecast_days` and how many variables you request, and
  `geocode` by `count`; there is never a next-page token.
- `truncated: true` in the envelope means the body hit the byte cap
  (262144). Don't re-request the same thing — narrow it: fewer variables,
  smaller `forecast_days`, smaller `count`.

## 5. Rate-limit etiquette

- Budget: this connection declares 60/minute — deliberately far under
  Open-Meteo's free ceiling (~600/minute, 10,000/day). Weather questions
  rarely need more than a geocode plus one forecast call.
- On HTTP 429 the envelope includes `retry_after_ms` — wait that long
  before ONE retry; if it recurs, report the limit to the user rather
  than hammering.
- Cheap probe: `geocode` (the declared `health` op) costs normal budget but
  is tiny and stable — one small JSON body, no forecast computation.

## 6. Operation relationships & recipes

Worked multi-call sequences:

1. **Weather by place name**: `geocode` (name, `countryCode` if known) →
   take `results[0].latitude` / `.longitude` (confirm the candidate's
   `country`/`admin1` matches what the user meant) → `current-weather`.
   This order because forecast ops only accept coordinates.
2. **Trip outlook**: `geocode` → `forecast` with
   `daily="temperature_2m_max,temperature_2m_min,precipitation_sum"`,
   `forecast_days=<trip length, ≤16>`, `timezone="auto"` — daily aggregates
   require a timezone, and `auto` saves you knowing the local zone.
3. **Compare two cities now**: `geocode` city A → `geocode` city B →
   `current-weather` per city (2 calls; trivially within budget).

## 7. Write-op cautions

`safe:false` operations in this connection: **none** — all three operations
are read-only GETs against a public dataset. There is nothing to mutate,
bill, or notify; this section exists only so its absence is explicit.

## 8. Data gotchas

- **Two hosts, one connection**: `base_url` is `api.open-meteo.com`, but
  `geocode` lives on `geocoding-api.open-meteo.com` — its `path` in
  connection.json is an absolute URL. Both hosts are in
  `network_allowlist` and BOTH must be in the http server's
  `HTTP_ALLOWED_HOSTS`, or geocoding fails with `not_allowlisted` while
  forecasts still work.
- Timestamps: ISO 8601 *without* offset (`2026-07-04T15:00`), in GMT unless
  you pass `timezone` — always pass `timezone` (or `"auto"`) when showing
  times to a user. `daily` requests fail without one.
- Parallel arrays: hourly/daily data comes as `time: [...]` plus one
  same-length array per variable — index-align them; units are in the
  sibling `hourly_units` / `daily_units` object (defaults: °C, km/h, mm).
- `geocode` returns `{"generationtime_ms": ...}` with NO `results` key at
  all when nothing matches — treat a missing `results` as "no match", not
  an error.
- Forecasts are model output on a grid (~1-11 km): the returned `elevation`
  may differ from the real spot, and current conditions are model-based,
  not a physical station reading.

## 9. Known errors table

| Upstream code / error | Meaning | Next action |
|---|---|---|
| `400` + `{"error":true,"reason":...}` | Invalid parameter — misspelled variable name, out-of-range coordinate, or `daily` without `timezone` | Fix the parameter `reason` names; do not retry unchanged. |
| `404` | Wrong path — e.g. forecast path sent to the geocoding host or vice versa | Check which host the op's path targets (§8, first gotcha). |
| `429` | Rate limit exceeded (free tier: minutely/hourly/daily ceilings) | Wait `retry_after_ms`, one retry; if it recurs, tell the user. |
| `not_allowlisted` (envelope, no HTTP status) | Host missing from `HTTP_ALLOWED_HOSTS` — usually the geocoding host was forgotten | Ask the operator to allowlist BOTH hosts from `network_allowlist`. |
| Empty body / missing `results` on geocode | No place matched (this is a 200, not an error) | Retry with a simpler/shorter name or drop `countryCode`. |

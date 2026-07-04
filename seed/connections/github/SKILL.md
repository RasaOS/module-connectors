---
name: github
description: Work with the GitHub REST API through the `github` connection (.claude/connections/github/connection.json) using the http module's MCP tools (http_request, http_get_json, http_paginate). Triggered when the user asks about GitHub data or actions — e.g. "how many open issues does anthropics/claude-code have?", "find Anthropic's TypeScript repos", "open an issue on my repo".
---

# GitHub — connection skill

## 1. When to use this connection (and when NOT)

- Use for: live repository metadata, issue listings, public-repo search,
  and (with auth + explicit intent) creating issues on github.com.
- Prefer over local git when: the question is about the *remote* state —
  stars, open-issue counts, issues themselves, repos you don't have
  cloned. A local clone answers commits/branches/files cheaper.
- Do NOT use for: GitHub Enterprise Server instances (different host —
  this connection is pinned to `api.github.com`), git *content*
  operations (clone/push — use git), or GraphQL-only data (projects v2,
  discussions). If the `gh` CLI is available and authenticated it may be
  a cheaper path for complex authenticated workflows.

## 2. Auth note

Auth is handled server-side — the http server injects the credential for
allowlisted hosts from env / the kernel vault. **Never ask the user to
paste a token, key, or secret value into the conversation.** If auth is
missing, the `missing_auth` error names the exact env var to set — for
this connection that is `HTTP_AUTH_API_GITHUB_COM` — relay that to the
operator and stop; do not retry.

**GitHub-specific:** the four safe read operations work UNAUTHENTICATED
against public data. If no token is configured you are not blocked — you
are just on the 60 req/hr per-IP budget instead of 5,000 req/hr (see §5).
Only `create-issue` hard-requires a token. So treat a missing credential
as a budget constraint for reads, and as a stop condition only for writes.

**API version pin:** GitHub versions its REST API via the
`X-GitHub-Api-Version: 2022-11-28` request header. The connection spec has
no header field for this, so when using raw `http_request`, pass it in
`headers` yourself; responses are compatible without it today, but pinning
protects recipes from future breaking changes.

## 3. ID formats & lookups

- Repository: `owner/repo` pair of two path segments (e.g.
  `anthropics/claude-code`) — never a numeric id in this connection.
  Obtained via `search-repos` when you only have keywords.
- Owner: a user or org *login* (e.g. `anthropics`), not the display name
  ("Anthropic" is not a login).
- Issue: an integer `number` scoped to its repository (issue #74123 of
  anthropics/claude-code). Obtained via `list-issues`. The API also
  returns a global `id` — do not confuse the two; paths take `number`.
- Gotcha: issue and PR numbers share ONE sequence per repo, and
  `list-issues` returns both kinds — PR items carry a `pull_request` key;
  filter on its absence when the user means issues only.

## 4. Pagination idiom

- Style: `page`, param `page` (see `pagination` in connection.json),
  with `per_page` (max 100) controlling page size. Pages start at 1.
- Use `http_paginate` for page-style listings like `list-issues` and
  `search-repos` (merges top-level or `items` arrays, max 10 pages per
  call). GitHub also sends an RFC-5988 `Link` header with `rel="next"` /
  `rel="last"` — when you need the total page count, read it from a
  single `http_request` response instead of walking pages.
- Search results are capped at 1,000 items regardless of paging; past
  that, narrow `q` with qualifiers.
- The envelope's `truncated: true` means the RESPONSE BODY was cut at the
  byte cap — that is not "no more pages." Narrow the request (smaller
  `per_page` — issue bodies are large, 3–10 is usually plenty) instead of
  paging past it.

## 5. Rate-limit etiquette

- Budget: **unauthenticated 60 requests/hour per IP** (core bucket);
  **authenticated 5,000 requests/hour per token**. Search is its own,
  tighter bucket: 10/min unauthenticated, 30/min authenticated. The
  connection's declared `per_minute: 30` stays conservatively under the
  unauth hourly ceiling — don't design loops that need more.
- On HTTP 429 (or a 403 whose body mentions rate limiting) the envelope
  includes `retry_after_ms` — wait that long before ONE retry; if it
  recurs, report the limit and the bucket's `reset` time to the user
  rather than hammering. Exceeding limits repeatedly can get the IP
  temporarily banned.
- Cheap probe: `get-rate-limit` does NOT count against any budget and
  reports every bucket's `remaining` + `reset` (epoch seconds). Call it
  before burst work.

## 6. Operation relationships & recipes

Worked multi-call sequences:

1. **Find a repo, then inspect it**: `search-repos` (q with
   `org:<login>` qualifier) → take `items[].full_name`, split into
   owner/repo → `get-repo`. Search gives ranked candidates; get-repo
   gives the authoritative full record.
2. **Issue triage sweep**: `get-repo` → read `open_issues_count` to size
   the job (note: that count includes PRs) → `list-issues`
   (`state:"open"`, small `per_page`) per page, skipping items with a
   `pull_request` key. Cap pages against the §5 budget — on the unauth
   60/hr budget, a 9,800-open-issue repo is not fully sweepable; sample
   instead.
3. **Pre-write existence check**: `get-repo` on the target → confirm it
   exists and note `visibility` → only then `create-issue` (after
   explicit user approval, §7). A 404 here means bad owner/repo OR no
   access — surface that before attempting the write.

## 7. Write-op cautions

`safe:false` operations in this connection: `create-issue`.

- It mutates real GitHub state and **notifies humans**: every watcher of
  the repository gets an email/notification the moment the issue lands.
- Issues can be closed but **never deleted** — there is no clean undo;
  a mistaken issue stays visible in the repo's history.
- **Explicit user intent required before it runs** — a plan the user
  approved, or a direct ask naming the target repo. Never chain a write
  onto a read the user didn't ask to be followed by one.
- It hard-requires an authenticated token with issue write access on the
  target repo; expect 404 (not 403) when the token lacks access — GitHub
  hides private resources' existence.

## 8. Data gotchas

- Timestamps: ISO 8601 UTC with `Z` suffix (`2026-07-03T18:20:11Z`)
  throughout.
- `open_issues_count` on a repo INCLUDES open pull requests — it will
  read higher than the human "Issues" tab.
- `list-issues` items that are PRs carry a `pull_request` key (verified
  live); plain issues lack it. Filter on absence for issues-only asks.
- Enums: `state` is a closed set (`open`/`closed`/`all`); search
  qualifiers (`language:`, `org:`, `stars:`) are part of the `q` string,
  not separate params.
- Size limits: issue `body` fields are Markdown and can be tens of KB —
  with the 256 KB response cap, keep `per_page` at 3–10 for issue
  listings (a `per_page:3` claude-code page came back untruncated;
  30 would likely not).
- Search `total_count` is approximate for large result sets and the
  index is eventually consistent — a just-created repo may not appear
  immediately.

## 9. Known errors table

| Upstream code / error | Meaning | Next action |
|---|---|---|
| `401` | Token invalid/expired/revoked | Report `missing_auth`-style: name `HTTP_AUTH_API_GITHUB_COM`, ask the operator to rotate it; do not retry. For safe reads, offer to proceed unauthenticated. |
| `403` | Insufficient scope, OR rate-limited disguised as 403 (body says "API rate limit exceeded") | Read the body: if rate-limit, treat as 429 (§5); else check token permissions against `setup_help`. |
| `404` | Not found OR no permission — GitHub hides which for private resources | Verify the `owner/repo` spelling via `search-repos` before concluding it's gone; if auth'd, suspect missing repo access. |
| `422` | Validation failed (e.g. empty `title` on create-issue, malformed search `q`) | Fix the named field — the response body's `errors[]` says which; do not blind-retry. |
| `301` | Repository was renamed/transferred | Follow the new location given in the response; update the stored `owner/repo`. |
| `410` | Issues are disabled on this repository | Report it; `create-issue`/`list-issues` cannot work there. |

---
name: llm-wiki
description: "Query the user's LLM Wiki knowledge base (the LLM Wiki desktop app at 127.0.0.1:19828 — NOT Obsidian, Notion, Apple Notes, Logseq, or any other PKM tool). Trigger ONLY when the user explicitly names LLM Wiki, says 'my wiki', 'my 知识库 / 知识库 / knowledge base', or asks things like 'what does my wiki say about X', 'read wiki page Y', 'show my wiki graph / 知识图谱', 'search in my LLM Wiki project', 'rescan my wiki sources / 重新索引', or names a wiki project by ID. DO NOT trigger on generic 'search my notes', 'find in my notebook', 'check my Obsidian', etc. — those belong to other tools the user may have installed. Covers wiki page search, file listing, content read, knowledge graph navigation, and source rescan against the running LLM Wiki desktop app. Read-only by default, writes only on the user's explicit request."
---

# LLM Wiki Local API Skill

Talk to the user's locally-running LLM Wiki app over its built-in HTTP API. This is a **standard JSON API** — call it directly with whatever HTTP tool is already in your environment (`curl`, `fetch`, `requests`, `http` middleware, etc.). No client library to install, no SDK to learn.

Treat the wiki as a **private, structured knowledge base** the user has been curating: pages live as `wiki/**.md`, raw documents under `raw/sources/`, wikilinks form a graph.

## When to invoke

Invoke **only** when the user is clearly referring to **LLM Wiki** specifically — by app name, by `wiki` framing, or by `知识库` framing. Concretely:

- asks a question framed as "what does my **wiki** / my **knowledge base** / 我的**知识库** / **LLM Wiki** say about X"
- asks to "search **my wiki** / **LLM Wiki** project / 我的**知识库** for X"
- references a **wiki page** by stem / title and wants to read or cross-link
- asks for the **wiki graph / 知识图谱 / wiki overview / wiki structure**
- has just added or edited files under the LLM Wiki **source folder** and wants ingest re-run / **重新索引**
- says "use **my wiki** for context" / "ground your answer in **my wiki**" / "check **my LLM Wiki**"
- names a wiki project (by ID, by absolute path, or by `current`)

**Do NOT invoke when the user says:**

- "search **my notes**" without further qualification — likely Obsidian / Apple Notes / Notion / Logseq / Bear / etc.
- "find in **my notebook**" — likely Jupyter / OneNote / Notability
- "check **my Obsidian / Notion / Roam / Logseq vault**" — explicitly a different tool
- "look up **my Anki / Readwise / Pocket**" — different tool
- "search **my files / my Documents folder**" — generic filesystem, not the wiki
- general world knowledge, current events, or anything the user clearly wants from the open web

When in doubt about which knowledge tool the user means, ask: *"Do you mean your LLM Wiki specifically, or another tool?"* — don't silently call the LLM Wiki API on what might be an Obsidian vault.

## Quick start

The whole API is plain HTTP + JSON. The fastest path:

```bash
BASE=http://127.0.0.1:19828
TOKEN="${LLM_WIKI_API_TOKEN:-<paste-from-Settings>}"

# 1. probe state — no auth needed
curl -s $BASE/api/v1/health

# 2. list projects
curl -s -H "Authorization: Bearer $TOKEN" $BASE/api/v1/projects

# 3. search
curl -s -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"query":"rope embedding","topK":5}' \
  $BASE/api/v1/projects/current/search

# 4. read a page
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/projects/current/files/content?path=wiki/concepts/rope.md"
```

If you're writing TypeScript / JavaScript:

```ts
const res = await fetch("http://127.0.0.1:19828/api/v1/projects/current/search", {
  method: "POST",
  headers: { "Authorization": `Bearer ${process.env.LLM_WIKI_API_TOKEN}`, "Content-Type": "application/json" },
  body: JSON.stringify({ query: "rope embedding", topK: 5 }),
})
const { results } = await res.json()
```

Python is the same shape — `urllib.request`, `requests`, `httpx`, whatever you already have. **Don't install anything new.**

## Auth model

The API listens on `127.0.0.1` by default. That is a default, not a guarantee: `allowLanAccess` makes it listen on `0.0.0.0`, and `LLM_WIKI_BIND_HOST` takes precedence over that setting. The listen address is applied when the server binds, not on each request, and `/health` → `allowLanAccess` shows the setting, not the address in use. Loopback and LAN use the same auth scheme and plain HTTP: the token is the only protection and, with LAN listening, it travels in clear text. If `allowLanAccess` is `true`, warn the user about this. See `api-reference.md` → "Listen address".

The token is one of:

1. `LLM_WIKI_API_TOKEN` environment variable (if set, overrides UI)
2. The user's `apiConfig.token` saved in Settings → **API + MCP** (labelled "API Server" in older builds and in some messages)
3. `allowUnauthenticated: true` mode (no token needed; rare, user opt-in only). In this mode `projects`, `files`, `files/content`, `reviews`, `search`, `graph` and `sources/rescan` stay open. `chat`, `chat/{sessionId}/cancel`, `pages/embed` and `pages/write` always require the token. See `api-reference.md` → `allowUnauthenticated`.

Always check `/api/v1/health` first — it returns `ok`, `status`, `version`, `enabled`, `authRequired`, `authConfigured`, `allowUnauthenticated` and `tokenSource`; app 0.6.13 adds `mcpEnabled`, `allowLanAccess` and `agent` (`chat`, `streaming`, `streamProtocol`). The auth check uses `authConfigured` and `allowUnauthenticated`. **If `authConfigured: false && allowUnauthenticated: false`, ask the user to open `Settings → API + MCP → Generate new token`**. Do not proceed without auth being set up.

Three equivalent ways to send the token:

```
Authorization: Bearer <token>          # preferred
X-LLM-Wiki-Token: <token>              # alternative header
?token=<urlencoded-token>              # query param — last resort, leaks into logs
```

**Never log or echo the token. Never put it in any URL the user can see in your output** (Referer / shell history / logs all leak it).

## Standard workflow

When the user asks "look it up in my wiki":

1. **Resolve project** (see [Project resolution](#project-resolution) below).
2. **Search**: `POST /api/v1/projects/{id}/search` with `{ query, topK: 5..10 }` → ranked hits (for example `path`, `title`, `snippet`, `score`, `titleMatch`, optional `vectorScore`, and `graphRelatedTo` on graph candidates; this is not a complete list of result fields). Inspect `vectorHits` and `graphHits` (not just `mode`) to know what kicked in — see below. Search only covers `wiki/**/*.md` (never `raw/sources/`), at most 10,000 files; past that limit it stops and the response does not say so.
3. **Read top hits**: for each promising hit, `GET /api/v1/projects/{id}/files/content?path=...` for the full markdown. Or pass `includeContent: true` to the search to avoid the round-trip.
4. **Cite + answer**: synthesize an answer grounded in the read pages. **Quote the `path` of each page you used** so the user can verify and jump in-app.

### Reading the score

The `score` field's scale depends on **`vectorHits`**, not on `mode`:

- **`vectorHits: 0`** (no embeddings configured, or no vector match) — additive keyword score, even when `mode` is `"hybrid"`. Weights: exact filename 200, phrase in title 50, phrase in content 20 per occurrence (max 10), each query token ×5 in the title and ×1 in the content.
- **`vectorHits > 0`** — RRF (Reciprocal Rank Fusion) is applied; it is applied only in this case. No numeric range is documented for RRF scores or for scores with embeddings, so do not assume one; relative ordering within the response is what matters. `vectorScore` is a composite per-page value (best chunk plus up to 30% of the other chunks, at most 1), not a raw cosine.
- **`mode`** is `"hybrid"` whenever `graphHits > 0`, **even with `vectorHits: 0`**: one-hop graph neighbours of the top hits take 15–30% of the `topK` window (30% when no vectors matched; none when `topK < 2`, which yields `mode: "keyword"`). Graph candidates carry `graphRelatedTo: [titles]` and the snippet "Graph neighbor of …".

Scores are not comparable between modes or between responses with different `vectorHits`. Don't apply a fixed score threshold; rely on relative gaps within one response.

### Project resolution

`{id}` in every project-scoped endpoint accepts **four forms**:

| Form | When to use | Example |
|---|---|---|
| `current` (literal) | Default for "my wiki / 我的知识库 / this project / this wiki". It resolves to the project with `current: true`, which the desktop UI sets. | `/api/v1/projects/current/search` |
| UUID | The user pasted a project ID, OR you previously resolved a name to an ID and want to re-use it. | `/api/v1/projects/a0e90b29-fcf3-4364-9502-8bd1272de820/files` |
| Absolute filesystem path (URL-encoded) | The user named the path (e.g. `~/notes/research`). Useful when the user has multiple projects with similar names. A `/` in the path must be sent as `%2F`. On Windows the match does not distinguish letter case and accepts `\`. | `/api/v1/projects/%2FUsers%2Fme%2Fwiki%2Fresearch/files` |
| Project name | **Not supported directly** (404). You must `GET /api/v1/projects` first, find a match by `name`, then use that project's `id`. |

`GET /projects` lists the projects, which can be more than one, plus a top-level `currentProject` object. Work only on the projects the user means.

**Decision tree** for what the user said:

```
"my wiki" / "my 知识库" / "this wiki" / "this project" / unspecified
    → use `current`

"my Research project" / "in Reading"
    → GET /api/v1/projects
    → name-match (case-insensitive substring on `name`)
    → use the resulting `id`
    → if 0 matches: tell the user, list available names, fall back to `current` only if they confirm
    → if 2+ matches: ask the user to disambiguate, quoting both names + paths

"the project at /Users/me/foo"
    → URL-encode the path, use directly
    → if the API returns 404, the project isn't registered — list and let user pick

"project a0e90b29-…"
    → use the UUID literally
```

Cache the resolved `id` for the rest of the conversation — there's no need to re-`GET /projects` for every call. But if the user switches contexts mid-conversation ("now look in my Reading project"), re-resolve.

When the user is silent about which project, **default to `current`** and mention it once: *"Looking in your active project (Research Notes)…"*. This avoids cross-project surprises.

For graph / cross-reference questions:

- `GET /api/v1/projects/{id}/graph?limit=200&offset=0` → `{ nodes: [{id, label, nodeType, path, linkCount}], edges: [{source, target, weight}], offset, limit, totalCount, hasMore }`
- Filter via `?q=term` (substring of id/label, case-insensitive) and `?nodeType=...` (camelCase: `node_type` is silently ignored; an unknown value returns 0 nodes). `nodeType` is an open set, not a fixed list: it is the lowercased value of the page's first `type:` line (`other` if there is none). `entity`, `concept` and `other` are examples, not a complete list. Pages with `type: query` are **not** graph nodes.
- `q` filters nodes only and returns edges between matching nodes: it does **not** return the neighbours of a match. For neighbours, page through the full graph and filter edges client-side.
- Pagination (0.6.13+): `limit` defaults to 200 and is clamped to 1-1000 (larger values become 1000; `0`, negative or non-numeric → 400). Read `hasMore` and request the next page with `offset`. With the default `edgeScope=page` an edge is returned only when both ends are on the page; `edgeScope=filtered` returns the edges between the filtered nodes, also outside the page. Any other `edgeScope` → 400. This applies to `graph` only: `files` has no pagination.
- Edges come from `[[wikilinks]]` **and** the frontmatter `related:` list; a link that matches more than one page is dropped. The graph does not depend on embeddings. The backend caches the graph for 5 s.

For "I added new docs" requests:

- `POST /api/v1/projects/{id}/sources/rescan` → returns `{ queue: { tasks }, changedTasks: [...] }`. Tell the user how many files changed. **This does not prove that ingest was queued** — see `api-reference.md` → "When does ingest actually start".

## Endpoint contract (v1)

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/health` | No auth. Returns `{ ok, status, version, enabled, authRequired, authConfigured, allowUnauthenticated, tokenSource }`; 0.6.13 adds `mcpEnabled`, `allowLanAccess` and `agent` (`{ chat, streaming, streamProtocol }`), and also answers on `/health` without the prefix. `allowLanAccess` shows the setting, not the listen address. |
| GET | `/api/v1/projects` | List projects. Each: `{ id, name, path, current }`; 0.6.13 adds top-level `currentProject`. |
| GET | `/api/v1/projects/{id}/files?root=wiki\|sources\|all&recursive=true&maxFiles=2000` | `root`: `wiki`, `sources` (alias `raw`, `raw/sources`) or `all`. Tree of `{ name, path, isDir, size, children }`. On 0.6.13 `path` is **absolute** (on Windows `//?/C:/...`): strip the project path to get the relative path for `files/content`. Exceeding `maxFiles` → **413**. Capped at 10000 nodes. |
| GET | `/api/v1/projects/{id}/files/content?path=wiki/foo.md` | Text files only: md, mdx, txt, csv, json, yaml, yml, xml, html, htm, log. `rtf` is not readable (415). 2 MB max. 400 if `path` is missing, 403 on out-of-scope path (also `.llm-wiki/`, `..`, dot-segments, directories), 415 on binary, 413 on oversize. |
| POST | `/api/v1/projects/{id}/search` | Body: `{ "query": "...", "topK": 10, "includeContent": false, "queryEmbedding": null }` (`queryEmbedding`: optional precomputed `number[]`; `limit` is ignored). Keyword + one-hop graph candidates, plus vectors when embeddings are configured; see "Reading the score". Response includes `mode`, `tokenHits`, `vectorHits`, `graphHits` and `results`; don't treat the lists of response and result fields as complete. Searches `wiki/**/*.md` only, at most 10,000 files, with no truncation signal in the response. `content` is absent (not `null`) unless `includeContent: true`. Empty query → 400. |
| GET | `/api/v1/projects/{id}/graph?q=&nodeType=&limit=200&offset=0&edgeScope=page` | Graph from `wiki/**/*.md` wikilinks and `related:`. `limit` defaults to 200 and is clamped to 1–1000 (5000 becomes 1000; `0`, negative or non-numeric → 400). **0.6.13+:** `offset` and `edgeScope` (`page` \| `filtered`; any other value → 400) for pagination; response adds `offset`, `limit`, `totalCount`, `hasMore`. `q` filters nodes only. Pages with `type: query` are not nodes. The backend caches the graph for 5 s. |
| POST | `/api/v1/projects/{id}/sources/rescan` | Triggers a backend rescan using the user's Source Watch config. Returns post-rescan queue + changed tasks. Does **not** itself queue ingest: see `api-reference.md`. |
| GET | `/api/v1/projects/{id}/reviews?status=&type=&limit=` | 0.6.13. Reads the review items. Filters and fields: `api-reference.md` → "GET /projects/{id}/reviews". |
| PATCH | `/api/v1/projects/{id}/reviews/{reviewId}` | 0.6.13. **Writes**: rewrites `.llm-wiki/review.json` of the project in `{id}`. The request body is not documented here. |
| DELETE | `/api/v1/projects/{id}/reviews/{reviewId}` | 0.6.13. **Writes**: rewrites `.llm-wiki/review.json` of the project in `{id}`. |
| POST | `/api/v1/projects/{id}/reviews/resolve` | 0.6.13. **Writes**: rewrites `.llm-wiki/review.json` of the project in `{id}`. The request body is not documented here. |
| POST | `/api/v1/projects/{id}/pages/write` | 0.6.13. **Writes** a markdown page. Body: `{ "path": "...", "content": "...", "allowOverwrite": ... }`. The token is always required. **409** if the page exists and `allowOverwrite` is not set; **400** for the path, the markdown or the size; **500** otherwise. Body limit about 12 MiB. |
| POST | `/api/v1/projects/{id}/pages/embed` | 0.6.13. Changes state; what it writes is not documented here. The token is always required. 400 if embedding is not configured; 503 while 4 embeds are running. The request body is not documented here. |
| POST | `/api/v1/projects/{id}/chat` | 0.6.13. Starts an agent chat turn, which writes to the agent sessions of the project. Replies in JSON or as an SSE stream (events `meta`, `agent`, `done`, `cancelled`, `error`). The token is always required. The request body is not documented here. 503 with more than 8 chat streams running; body limit 40 MiB. |
| (not documented) | `/api/v1/projects/{id}/chat/{sessionId}/cancel` | 0.6.13. Cancel route for a chat session. The token is always required. The HTTP verb and the outcome of a cancellation are not documented here. |

`{id}` accepts a UUID, an absolute filesystem path (URL-encoded), or the literal string `current`.

### Routes that change state

On app 0.6.13 these routes change state: `sources/rescan`, the review routes `PATCH`, `DELETE` and `resolve`, `pages/write`, `pages/embed` and `chat`. All the limits are in `api-reference.md` → "Limits & defenses".

- Call them **only when the user explicitly asks** for that action. Never call them to explore, to check that a route exists or to prepare an answer.
- Review changes, `sources/rescan` and `pages/embed` act on the project named in `{id}`, not only on `current`: the app does not check that `{id}` is the current project. A call on a project that is not `current` is therefore **not** read-only. Whether `pages/write` and `chat` check it is not known: apply the same care.
- The bodies of `chat`, of review `PATCH` and `resolve` and of `pages/embed` are not documented here. Do not guess them.

## Error handling

Always treat the status code as the contract:

| Status | Meaning | What to do |
|---|---|---|
| 200 | OK | Use `body.ok === true` belt-and-suspenders; payload is in the same object. |
| 400 | Bad request | Show `body.error`. Typical: empty `query`, invalid `?root=`, a `pages/write` body that is too large. |
| 401 | Unauthorized | Token missing/wrong. Without a valid token the API answered 401 also for unknown routes and wrong methods, so a 401 does not prove that the route exists. Tell user to set/regenerate in Settings → API + MCP. |
| 403 | Forbidden | Path traversal or out-of-scope (e.g. `../app-state.json`). Don't retry the same path. |
| 404 | Not found | Unknown project id, unknown route, **or a wrong verb among GET, POST, PATCH and DELETE** (e.g. `GET .../search`). On unknown project, list projects first to recover. |
| 405 | Method not allowed | Only for verbs outside GET, POST, PATCH and DELETE (e.g. `PUT`). `OPTIONS` is not a 405: it gets 204 with the CORS headers, before every other check. |
| 409 | Conflict | `pages/write` only: the page exists and `allowOverwrite` is not set. Do not retry with `allowOverwrite` unless the user asked to replace the page. |
| 413 | Payload too large | File > 2 MB or file tree > maxFiles. Suggest narrower scope. The status code for a request body over the generic 1 MiB limit is not known: do not rely on a specific one. |
| 415 | Unsupported media | Binary or non-UTF-8 file content. API is text-only. |
| 429 | Too many requests | Rate limit: 120 requests in a 1 s window, global; `/health` and `OPTIONS` are not counted. It applies before the token check. Known from the app code; not tested under load. No `Retry-After` header is known (see `api-reference.md` → "Limits & defenses"): wait at least 1 second, as your own caution. |
| 500 | Internal error | Log + report; don't loop. |
| 503 | Service unavailable | API toggled off (`error` contains "disabled"); in-flight cap (64) reached ("busy"); on 0.6.13 also more than 8 chat streams or more than 4 page embeds running. Known from the app code; not tested under load. No `Retry-After` header is known: wait at least 2 seconds, as your own caution. |

If the HTTP call itself fails (connection refused / ENOTFOUND): the desktop app is **not running**. Tell the user: "Launch LLM Wiki, then re-try."

## Etiquette

- **Cite paths.** When you answer using wiki content, name the page: `(from wiki/concepts/rope.md)`. The user uses these to verify and to jump in-app.
- **Stay read-only by default.** The routes that change state (see "Routes that change state"), `pages/embed` and `chat` run only after the user explicitly asked for that action. A project id that is not `current` does not make a review change, a rescan or an embed read-only.
- **Don't dump full pages unless asked.** Snippet + path is usually enough. Pull full content only when reasoning genuinely needs it.
- **Respect the project boundary.** The current project is the user's active context. Do not silently switch projects.
- **Honor the rate limit.** 120 requests per 1 s window is plenty for sequential work, but parallel page reads can burst close to it. Batch where the API allows (`includeContent: true` on search avoids N+1 reads).
- **Never leak the token.** Headers are safe; query params and your own output text are not.

## See also

- `api-reference.md` — full endpoint shapes with request / response examples
- `examples.md` — common conversational patterns mapped to direct `curl` / `fetch` sequences
- `README.md` — human setup notes (token generation, port conflicts, troubleshooting)

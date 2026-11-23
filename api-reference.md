# LLM Wiki API v1 — Endpoint reference

Base URL: `http://127.0.0.1:19828` (default; the listen address can change, see "Listen address" below)
Prefix:   `/api/v1`
Default content type: `application/json`

Except `/health`, every endpoint requires the token, sent in one of these forms:

```
Authorization: Bearer <token>
X-LLM-Wiki-Token: <token>
?token=<urlencoded-token>          # last resort, leaks into logs
```

With `allowUnauthenticated: true` (user opt-in, reported by `/health`) the token requirement changes per route:

| Routes | With `allowUnauthenticated: true` |
|---|---|
| `projects`, `files`, `files/content`, `reviews`, `search`, `graph`, `sources/rescan` | Stay open: no token needed. |
| `chat`, `chat/{sessionId}/cancel`, `pages/embed`, `pages/write` | The token is still always required. |

---

## GET /api/v1/health

**Auth:** not required.

```json
{
  "ok": true,
  "status": "running",
  "version": "0.6.13",
  "enabled": true,
  "authRequired": true,
  "authConfigured": true,
  "allowUnauthenticated": false,
  "tokenSource": "store",
  "mcpEnabled": true,
  "allowLanAccess": false,
  "agent": { "chat": true, "streaming": true, "streamProtocol": "sse" }
}
```

The last three fields are 0.6.13 additions. `/health` also answers without the prefix (`GET /health`) and to
`POST`.

Field reference:

- `status` — `starting` / `running` / `port_conflict` / `error`
- `enabled` — `false` = user toggled the API off; all non-`/health` endpoints return 503
- `authRequired` — `false` iff `allowUnauthenticated: true`
- `authConfigured` — `true` if env `LLM_WIKI_API_TOKEN` or `apiConfig.token` is set
- `allowUnauthenticated` — no token needed for the open routes listed under the token forms above (rare; user opt-in only)
- `tokenSource` — `env` / `store` / `none`. If `env`, the desktop UI token field is **ignored**.
- `version` — the app version string (for example `"0.6.13"`).
- `mcpEnabled` — 0.6.13: the app's MCP setting.
- `allowLanAccess` — 0.6.13: the LAN listening setting. It shows the setting, not the address in use (see "Listen address").
- `agent` — 0.6.13: `{ chat, streaming, streamProtocol }`. Observed on 0.6.13: `chat: true` and `streamProtocol: "sse"`.

---

## GET /api/v1/projects

**Auth:** required.

```json
{
  "ok": true,
  "projects": [
    {
      "id": "a0e90b29-fcf3-4364-9502-8bd1272de820",
      "name": "Research Notes",
      "path": "/Users/me/wiki-projects/research",
      "current": true
    },
    {
      "id": "...",
      "name": "Reading",
      "path": "/Users/me/wiki-projects/reading",
      "current": false
    }
  ],
  "currentProject": { "id": "a0e90b29-fcf3-4364-9502-8bd1272de820", "name": "Research Notes", "path": "/Users/me/wiki-projects/research", "current": true }
}
```

`currentProject` is a 0.6.13 addition and belongs to this response, not to `/health`. The list can hold several projects: work only on the ones the user means. Paths use forward slashes on Windows (`C:/...`).

The `current` flag is true for the project the desktop UI has set as current; the literal `current` in `{id}` resolves to that project. Use it when the user doesn't name a project explicitly.

`{id}` placeholder in every project-scoped endpoint accepts:

- the project UUID (`p.id`)
- the project filesystem path (`p.path`, URL-encoded)
- the literal string `current`

### Resolving a user-spoken project name

There is **no `?name=` filter** on this endpoint and `{id}` does **not** accept a name directly — names are resolved entirely client-side after listing all projects.

Algorithm:

1. `GET /api/v1/projects` → array of `{id, name, path, current}`
2. Case-insensitive substring match on `name`:
   ```js
   const matches = projects.filter(p =>
     p.name.toLowerCase().includes(spokenName.toLowerCase())
   )
   ```
3. Handle the cardinality:
   - **0 matches** → tell the user, list available names, ask. Don't silently fall back to `current`.
   - **1 match** → use `matches[0].id` for all subsequent calls in this conversation.
   - **2+ matches** → ask the user to disambiguate (show `name` + `path` for each).
4. Cache the resolved `id` for the rest of the conversation. Only re-list when the user switches contexts.

If the user gives a filesystem path verbatim, you can URL-encode it and pass it as `{id}` directly — no list lookup needed:

```bash
# macOS / Linux (bash / zsh)
PROJECT_PATH=$(printf %s "/Users/me/wiki/reading" | jq -sRr @uri)
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/projects/$PROJECT_PATH/files?root=wiki"
```

```powershell
# Windows (PowerShell)
$projectPath = [System.Uri]::EscapeDataString("C:/Users/me/wiki/reading")
curl.exe -s -H "Authorization: Bearer $env:LLM_WIKI_API_TOKEN" `
  "$env:BASE/api/v1/projects/$projectPath/files?root=wiki"
```

**Path-as-`{id}` notes** (0.6.13):

- On Windows the match against the project path does not distinguish letter case and accepts `\`.
- The path segment is split on `/` before it is decoded, so a `/` in the id must be sent as `%2F`. A colon is not a separator. `%XX` is decoded everywhere in the segment, `+` becomes a space and `?` ends the path. `EscapeDataString` / `jq @uri` / `encodeURIComponent` encode `/` as `%2F`.
- If you get 404, fall back to `GET /api/v1/projects`, find the project there, and use its `id` (UUID) — UUIDs are platform-agnostic and never need encoding.

If the path isn't registered in the desktop app, you'll get **404** — fall back to listing and asking.

---

## GET /api/v1/projects/{id}/files

**Auth:** required.

Query params:

| Param | Default | Notes |
|---|---|---|
| `root` | `wiki` | One of `wiki` / `sources` (alias `raw`, `raw/sources`) / `all`; anything else → 400. `wiki` and `sources` list the folder's **content** (no container node); `all` returns 4 entries: `purpose.md`, `schema.md`, `wiki`, `sources`. |
| `recursive` | `true` | `false` → only one level (directories then have `size` and `children` = `null`). Any value other than `false` counts as `true`. |
| `maxFiles` | `2000` | Clamped to `[1, 10000]` (`0` → 1, non-numeric → default). Exceed → **413** `"File listing exceeds maxFiles limit (N)"`. |

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "root": "wiki",
  "files": [
    {
      "name": "concepts",
      "path": "wiki/concepts",
      "isDir": true,
      "size": null,
      "children": [
        {
          "name": "rope.md",
          "path": "wiki/concepts/rope.md",
          "isDir": false,
          "size": 4321,
          "children": null
        }
      ]
    }
  ]
}
```

**0.6.13:** `path` is **absolute**, not project-relative — on Windows in verbatim form, e.g.
`//?/C:/Users/me/wiki/research/wiki/concepts/rope.md`. Strip the project path (from `GET /projects`) to build
the relative `path` that `files/content` expects.

Hidden files (dotfiles) and symlinks are silently skipped.

---

## GET /api/v1/projects/{id}/files/content

**Auth:** required.

Query params:

| Param | Required | Notes |
|---|---|---|
| `path` | yes | Project-relative. Allow-list: `purpose.md`, `schema.md`, `wiki/**`, `raw/sources/**`. No dotfile segments. No `..`. Readable extensions: md, mdx, txt, csv, json, yaml, yml, xml, html, htm, log. `rtf` is not readable (415). |

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "path": "wiki/concepts/rope.md",
  "content": "---\ntitle: RoPE\n---\n\n# Rotary Position Embedding\n..."
}
```

Failure modes:

- `400` — `path` parameter missing (`"Missing path query parameter"`)
- `403` — path outside the allow-list (e.g., `../app-state.json`, `.llm-wiki/foo.json`, encoded `..`, directories, dotfiles, empty `path=`)
- `404` — file does not exist (the message includes the OS error text)
- `413` — file > 2 MB
- `415` — file is binary / non-UTF-8 (e.g., PNG, PDF), or its extension is not in the readable list (e.g., `rtf`). Use the desktop UI to view; the API is text-only.

Check order on 0.6.13: 403 scope → 415 extension → 404 → 413 size → 415 not UTF-8.

---

## POST /api/v1/projects/{id}/search

**Auth:** required.

Body:

```json
{
  "query": "rope rotary position embedding",
  "topK": 10,
  "includeContent": false,
  "queryEmbedding": null
}
```

| Field | Default | Notes |
|---|---|---|
| `query` | (required) | Non-empty. Empty / whitespace-only → 400 `"query is required"`; missing or non-string → 400. |
| `topK` | `10` | Clamped to `[1, 50]` (`0` → 1, `999` → 50); negative or non-numeric → 400. |
| `includeContent` | `false` | When `true`, each hit carries `content` (full markdown). Skip the per-page content fetch round-trip. Must be a JSON boolean (`"yes"` → 400). |
| `queryEmbedding` | `null` | Optional `number[]`. If you precomputed a query embedding (your own model, batched offline), pass it here and the server skips its own embed call. Must be a non-empty array of finite numbers, otherwise → 400. A vector of the wrong dimension is silently ignored (`vectorHits: 0`). |

`limit` is **not** a search parameter: it is ignored. Only `wiki/**/*.md` is searched; `raw/sources/` is never
searched. The scan covers at most 10,000 files: after that it stops and the app writes a message to its own
stderr. The response has no field that reports this, so a wiki with more than 10,000 `.md` files can return
results from only part of it, and which files are left out is not defined.

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "mode": "hybrid",
  "tokenHits": 78,
  "vectorHits": 14,
  "graphHits": 3,
  "note": "Search uses the shared backend hybrid retrieval service, combining keyword, vector, and one-hop knowledge-graph candidates...",
  "results": [
    {
      "path": "wiki/concepts/rope.md",
      "title": "Rotary Position Embedding",
      "snippet": "...inject positional information by rotating Q and K...",
      "titleMatch": true,
      "score": 0.0315136476426799,
      "vectorScore": 0.94,
      "images": [
        { "url": "wiki/media/rope-diagram.png", "alt": "RoPE rotation diagram" }
      ],
      "content": "..."
    }
  ]
}
```

`graphHits` (0.6.13) is the number of one-hop graph candidates in `results`. `content` is **absent**, not `null`,
unless `includeContent: true`. The response and result examples show the fields known to the skill; don't treat
them as a complete list.

### Retrieval mode

0.6.13 (`commands/search.rs` 464, 541-551, 782-792):

| `mode` | Trigger | Score scale |
|---|---|---|
| `"keyword"` | `graphHits: 0` and `vectorHits: 0` — e.g. no embeddings and `topK < 2`, or no graph neighbours. | Additive keyword score. |
| `"vector"` | `graphHits: 0`, vectors matched, keyword matched nothing. | RRF. |
| `"hybrid"` | **`graphHits > 0`** (even with `vectorHits: 0`), or both keyword and vector matched. | RRF **only if `vectorHits > 0`**; otherwise additive keyword scores. |

So `mode` alone does not tell you what ran or the score scale: look at `vectorHits` and `graphHits`. `hybrid` can
have `vectorHits: 0`: without embeddings it means keyword + one-hop graph neighbours, with additive keyword scores
(no RRF). RRF is applied only when `vectorHits > 0`.

The skill documents no numeric range for RRF scores, and none for scores with embeddings active: do not assume one.

- **Keyword weights** (`search.rs` 14-20, 960-970): exact filename 200, phrase in title 50, phrase in content
  20 per occurrence (max 10), each query token ×5 in the title and ×1 in the content.
- **Graph share**: one-hop neighbours of the top hits take 15–30% of the `topK` window (30% when no vector
  matched, e.g. 3 of 10 and 15 of 50; none when `topK < 2`). Their snippet is "Graph neighbor of …" and
  `graphRelatedTo` marks them.
- **Embeddings** only affect search (RRF, graph share); the `/graph` endpoint does not use them.

`tokenHits` is the number of pages the keyword pass scored; `vectorHits` is the number of distinct pages LanceDB returned. Either can be 0.

### Per-result fields

| Field | Always present? | Notes |
|---|---|---|
| `path` | yes | Project-relative path to the markdown page. |
| `title` | yes | Front-matter `title:` if present; else first `# Heading`; else filename with dashes → spaces. |
| `snippet` | yes | A short excerpt; its length is not fixed, so don't rely on one (on 0.6.13 it was observed shorter than older docs stated, with a trailing `...`, frontmatter text included, newlines flattened). In keyword mode: centered on the query/anchor token in the page body. In vector-only matches: the actual matching chunk text, optionally prefixed with the chunk's heading path (e.g. `"Section > Detail: chunk text..."`). |
| `titleMatch` | yes | `true` when a token or phrase hit the title (boosts ranking). |
| `score` | yes | Final ranking score. Its kind depends on `vectorHits` (additive keyword score, or RRF): see "Retrieval mode". |
| `vectorScore` | optional | Composite per-page vector score when the page matched via the vector index: best chunk plus up to 30% of the other chunks, not above 1 — not a raw cosine. Copied into a result only when RRF is applied or for vector-only results; absent without vectors. |
| `images` | yes | Embedded `![alt](url)` references discovered in the markdown, deduped by URL. Useful for the agent to surface diagrams. |
| `content` | optional | Full markdown, only when `includeContent: true`. |
| `graphRelatedTo` | optional | 0.6.13: titles of the hits this page is a graph neighbour of; present only on graph candidates. |

These are the fields the skill documents, not a complete list: a result can carry other fields.

Scores are not comparable **between modes** or between responses with different `vectorHits` (additive keyword score and RRF are different scales). Rely on relative order and gaps within one response; check `graphRelatedTo` to tell graph candidates from text matches.

---

## GET /api/v1/projects/{id}/graph

**Auth:** required.

Query params:

| Param | Default | Notes |
|---|---|---|
| `q` | — | Substring filter on `id` or `label`, case-insensitive. Filters **nodes**; edges are only those between matching nodes, so `q` does not return a node's neighbours. |
| `nodeType` | — | Filter on the page's `type:` (first `type:` line, lowercased; `other` if none). Examples: `entity`, `concept`, `other`. Exact match: an unknown value returns 0 nodes; `node_type` (snake_case) is ignored. |
| `limit` | `200` | Absent → `200`. A number above `1000` is clamped to `1000` (`5000` becomes `1000`); `0`, negative or non-numeric → 400. |
| `offset` | — | 0.6.13+. Start of the page; use it with `limit` to read the next page while `hasMore` is true. |
| `edgeScope` | `page` | 0.6.13+. `page`: an edge is returned only when both ends are nodes of the returned page. `filtered`: edges between the filtered nodes are returned, also when an end is outside the page. Any other value → 400. |

`nodeType` is an open set derived from `type:`, not a fixed list; the examples above are not complete. Pages with `type: query` are **not** graph nodes (excluded on purpose, with their edges).

Pagination applies to `graph` only: the `files` tree has no pagination.

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "nodes": [
    {
      "id": "rope",
      "label": "Rotary Position Embedding",
      "nodeType": "concept",
      "path": "wiki/concepts/rope.md",
      "linkCount": 4
    }
  ],
  "edges": [
    {
      "source": "rope",
      "target": "attention",
      "weight": 1.0
    }
  ],
  "offset": 0,
  "limit": 200,
  "totalCount": 120,
  "hasMore": false
}
```

`offset`, `limit`, `totalCount` and `hasMore` are the pagination fields of 0.6.13+. Read `hasMore` before requesting another page.

Edges are derived from `[[wikilink]]` references **and the frontmatter `related:` list** in `wiki/**/*.md`. Deduplicated by unordered pair `(source, target)` — `[[a]]` in b and `[[b]]` in a produce **one** edge. Self-edges are dropped. `weight` is `1.0`. Node `id` is the file stem, or the relative path when two pages share a stem; ambiguous links are dropped. The graph is rebuilt from the files on request and the backend keeps a 5 s cache of it; there is no indexing state and embeddings play no part.

`linkCount` is the node's degree in the deduped graph.

---

## POST /api/v1/projects/{id}/sources/rescan

**Auth:** required. **Mutates state.**

No body required.

```json
{
  "ok": true,
  "projectId": "...",
  "result": {
    "queue": {
      "version": 1,
      "tasks": []
    },
    "changedTasks": [
      {
        "id": "...",
        "path": "raw/sources/new-paper.pdf",
        "kind": "created"
      }
    ]
  }
}
```

`changedTasks` contains the files this rescan **actually detected as changed** (created / modified / deleted) against the
app's file snapshot. The scan also covers `wiki/**/*.md`, `purpose.md` and `schema.md`, not only `raw/sources`.
Each change is reported **once**: the snapshot is updated before the response, so a second rescan returns an empty list
even if nothing was ingested.

The user's `sourceWatchConfig` (file type filters, exclude dirs, max size) is honored, read from the app state for the
project UUID, then `default`.

### When does ingest actually start

0.6.13 (`file_sync.rs` 339-362, `project-file-sync.ts` 62-66 and 337-370, `source-lifecycle.ts` 258-285):

- The rescan **does not queue ingest itself**. It updates the snapshot and emits an event to the desktop UI; the UI
  queues `created` / `modified` files under `raw/sources/`, and only if **all** of these hold:
  - the project is the one **open in the UI** (events for other projects are dropped), unless the global option
    "Watch all projects" is on (off by default; it rescans recent inactive projects every 60 s);
  - Source Watch `enabled` and `autoIngest` are on for the project;
  - an LLM usable for the ingest task is configured.
- Files written into `raw/sources/` by other processes are picked up by the app's own watcher, without any rescan, with the project open.
- An unchanged source is **not** ingested again: the ingest cache matches the hash of the extracted text, and
  re-queuing from the UI hits the same cache. The cache key does not include `schema.md`, `purpose.md` or the model
  (`ingest-cache.ts`).
- Ingest progress is visible only in the desktop UI (Activity panel) or in `.llm-wiki/ingest-queue.json` on disk; the
  API has no endpoint for it.

---

## POST /api/v1/projects/{id}/chat

**0.4.x: returns 501.** Chat / RAG pipeline lives in the WebView in v1. Don't invoke. Tell the user to use the desktop chat UI.

---

## Limits & defenses

| Limit | Value | Effect |
|---|---|---|
| Request body, generic | 1 MiB | The status code for a body over the limit is not known: do not rely on a specific one. |
| Request body, `chat` | 40 MiB | Limit of the app code; not tested by running the app. |
| Request body, `pages/write` | about 12 MiB | Too large → 400. Limit of the app code; not tested by running the app. |
| File content read | 2 MiB | Exceed → 413. |
| File-tree node count | 10000 hard cap | Exceed → 413. |
| Search `topK` | 50 max | Silently clamped. |
| Graph `limit` | 1000 max | Values above 1000 silently clamped. `0`, negative or non-numeric → 400. |
| Rate limit | 120 requests per 1 s window, global; `/health` and `OPTIONS` excluded; applied before the token check | 429. Wait at least 1 s (see `Retry-After` below). |
| In-flight requests | 64 concurrent | 503 "API server is busy". Wait at least 2 s (see `Retry-After` below). |
| Chat streams in flight | 8 concurrent | 503. |
| Page embeds in flight | 4 concurrent | 503. |

The body limits, the rate limit and the in-flight limits come from the app code. They were not tested by running the app: the 429 and 503 responses were not seen under load.

**`Retry-After`.** Do not assume that the server sends it. In the response paths known for 0.6.13, 429 and 503 carry no `Retry-After` header (the headers of a successful SSE response are not known). The waits suggested here, at least 1 s after a 429 and at least 2 s after a 503, are the agent's own caution, not a value from the server.

CORS: the request `Origin` is reflected in `Access-Control-Allow-Origin` only for these origins: `chrome-extension://` and `moz-extension://` (any extension id), `localhost`, `127.0.0.1` and `[::1]` (with or without a port), `tauri://localhost` and `http(s)://tauri.localhost`. For any other origin the header is absent. The server never sends a wildcard origin. `Access-Control-Allow-Methods: GET, POST, PATCH, DELETE, OPTIONS`. A preflight response carries `Access-Control-Max-Age: 600`: a browser may cache it for 600 seconds (10 minutes). Allowed headers include `Authorization`, `X-LLM-Wiki-Token`, `Content-Type`.
A request from an origin that is not in the list is still served. CORS only stops a browser page from reading the response; it is not access control. `OPTIONS` → 204 with the CORS headers, before every other check: no token is needed and the rate limit does not count it.

**Methods.** The allowed methods are GET, POST, PATCH and DELETE. `OPTIONS` is the CORS preflight: it gets 204 before any other check, so it is never a 405. 405 appears only for other verbs (e.g. `PUT`), and in the app code only after the token check. A wrong verb among the four allowed ones (e.g. `GET .../search`, `POST /projects`, `DELETE /projects`) → **404**, not 405. The old rule "every verb except GET and POST gives 405" does not hold on 0.6.13. Without a valid token the API answered 401 also for unknown routes and wrong methods.

---

## Additions in app 0.6.13

`/health` adds `mcpEnabled`, `allowLanAccess`, `agent: {chat, streaming, streamProtocol: "sse"}`.
### Listen address

The server listens on `127.0.0.1` by default. The listen address is chosen in this order:

1. `LLM_WIKI_BIND_HOST`, if it holds a valid host.
2. `0.0.0.0`, if `allowLanAccess` is `true`.
3. `127.0.0.1`.

The address is evaluated each time the server binds (first start, or a new bind after an error), not on each request. A changed `allowLanAccess` has no effect until the server binds again. `/health` → `allowLanAccess` shows the setting, not the address the server actually listens on.

There is no difference of auth scheme between loopback and LAN: the token is the only protection, and the API uses plain HTTP, so on the LAN the token travels in clear text. This is how the app code behaves; listening on the LAN was not tested by running the app.

### GET /projects/{id}/reviews

Query: `status` = `unresolved` (default, alias `pending`) | `resolved` | `all` (else 400; case-sensitive, `ALL` → 400);
`type` exact match (an unknown type → 200 with an empty list); `limit` default 200, clamped to 1-1000
(non-numeric → ignored, no 400).

```json
{ "ok": true, "projectId": "...", "status": "unresolved", "count": 1,
  "reviews": [ { "id": "review-00000000", "type": "suggestion", "title": "...", "description": "...",
                 "sourcePath": "C:/Users/me/wiki/research/raw/sources/...", "affectedPages": [], "searchQueries": [],
                 "options": [ { "label": "Create Page", "action": "Create Page" } ],
                 "resolved": false, "createdAt": 1700000000000.0 } ] }
```

Fields are present only if stored: `resolvedAction` is absent on unresolved items, `searchQueries` may be
missing. `sourcePath` is the **absolute** path of the source; `createdAt` is a JSON float (ms). The id is
`review-` + FNV-1a 32-bit of `type::normalized title`, so the same review from a re-ingest merges into the existing
item, and a resolved item stays resolved (`review-store.ts`). When the ingest queue drains, the app may resolve
items by itself (`resolvedAction` `auto-resolved` or `llm-judged`) (`sweep-reviews.ts`).

The desktop UI keeps reviews in memory and rewrites the whole `review.json` about 1 s after each change of its own store (e.g. an ingest adding reviews).

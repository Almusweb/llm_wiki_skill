# LLM Wiki API v1 — Endpoint reference

Base URL: `http://127.0.0.1:19828`
Prefix:   `/api/v1`
Default content type: `application/json; charset=utf-8` (0.6.13 sends `application/json` without `charset`; the body is UTF-8)

All non-`/health` endpoints require auth via one of:

```
Authorization: Bearer <token>
X-LLM-Wiki-Token: <token>
?token=<urlencoded-token>
```

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
- `allowUnauthenticated` — anonymous local-process mode (rare; user opt-in only)
- `tokenSource` — `env` / `store` / `none`. If `env`, the desktop UI token field is **ignored**.

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
  "currentProject": { "id": "a0e90b29-...", "name": "Research Notes", "path": "...", "current": true }
}
```

`currentProject` is a 0.6.13 addition. The list contains **every** project the app knows, including ones
unrelated to the user's current work. Paths use forward slashes on Windows (`C:/...`).

The `current` flag marks the project that is currently open in the desktop UI. Use it when the user doesn't name a project explicitly; in scripts resolve it once and use the UUID, because `current` follows whatever the UI opens.

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

**Windows path notes** (0.6.13, `api_server.rs` 855-896):

- Backslashes are normalized to forward slashes, and on Windows the comparison is **case-insensitive**. The upstream advice "preserve case, use only `/`" is not needed on 0.6.13.
- The path segment is split on `/` before decoding, so `/` **must** be encoded as `%2F`; the drive-letter colon is harmless (encoding it as `%3A` also works). `EscapeDataString` / `jq @uri` / `encodeURIComponent` do both.
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
| `maxFiles` | `2000` | Clamped to `[1, 10000]` (`0` → 1, non-numeric → default). Exceed → **413** `"File listing exceeds maxFiles limit (N)"`; the list is never returned truncated and `truncated` stays `false`. |

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
  ],
  "truncated": false
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
| `path` | yes | Project-relative. Allow-list: `purpose.md`, `schema.md`, `wiki/**`, `raw/sources/**`. No dotfile segments. No `..`. On 0.6.13 a leading `/`, backslashes and a different letter case are also accepted. Readable extensions: md, mdx, txt, csv, json, yaml, yml, xml, html, htm, log — **not** rtf. |

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
- `415` — file is binary / non-UTF-8 (e.g., PNG, PDF). Use the desktop UI to view; the API is text-only.

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

`limit` is **not** a search parameter: it is ignored. Only `wiki/**/*.md` is searched (at most 10,000 files,
scanning stops silently beyond that); `raw/sources/` is never searched.

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
unless `includeContent: true`.

### Retrieval mode

0.6.13 (`commands/search.rs` 464, 541-551, 782-792):

| `mode` | Trigger | Score scale |
|---|---|---|
| `"keyword"` | `graphHits: 0` and `vectorHits: 0` — e.g. no embeddings and `topK < 2`, or no graph neighbours. | Additive keyword score. |
| `"vector"` | `graphHits: 0`, vectors matched, keyword matched nothing. | RRF. |
| `"hybrid"` | **`graphHits > 0`** (even with `vectorHits: 0`), or both keyword and vector matched. | RRF **only if `vectorHits > 0`**; otherwise additive keyword scores. |

So `mode` alone does not tell you the score scale: look at `vectorHits`. Without embeddings `hybrid` means keyword +
one-hop graph neighbours, with additive scores.

- **Keyword weights** (`search.rs` 14-20, 960-970): exact filename 200, phrase in title 50, phrase in content
  20 per occurrence (max 10), each query token ×5 in the title and ×1 in the content.
- **Graph share**: one-hop neighbours of the top hits take 15–30% of the `topK` window (30% when no vector
  matched, e.g. 3 of 10 and 15 of 50; none when `topK < 2`). Their score is the graph score divided by 61, their
  snippet is "Graph neighbor of …", and they are interleaved, so `results` is **not** strictly sorted by `score`.
- **Embeddings** only affect search (RRF, graph share); the `/graph` endpoint does not use them.

`tokenHits` is the number of pages the keyword pass scored; `vectorHits` is the number of distinct pages LanceDB returned. Either can be 0.

### Per-result fields

| Field | Always present? | Notes |
|---|---|---|
| `path` | yes | Project-relative path to the markdown page. |
| `title` | yes | Front-matter `title:` if present; else first `# Heading`; else filename with dashes → spaces. |
| `snippet` | yes | ~160-char window (observed ~110 chars with a trailing `...`, frontmatter included, newlines flattened). In keyword mode: centered on the query/anchor token in the page body. In vector-only matches: the actual matching chunk text, optionally prefixed with the chunk's heading path (e.g. `"Section > Detail: chunk text..."`). |
| `titleMatch` | yes | `true` when a token or phrase hit the title (boosts ranking). |
| `score` | yes | Final ranking score. See "Retrieval mode" for scale. |
| `vectorScore` | optional | Per-page vector score in [0,1] when the page matched via the vector index: best chunk plus up to 30% of the other chunks, capped at 1 — not a raw cosine. Absent on pure keyword hits. |
| `images` | yes | Embedded `![alt](url)` references discovered in the markdown, deduped by URL. Useful for the agent to surface diagrams. |
| `content` | optional | Full markdown, only when `includeContent: true`. |
| `graphRelatedTo` | optional | 0.6.13: titles of the hits this page is a graph neighbour of; present only on graph candidates. |

Don't compare scores **across** responses with different `vectorHits` (keyword scores are 100×+ larger than RRF scores by construction). Rely on relative ordering within one response, keeping in mind that graph candidates are interleaved.

---

## GET /api/v1/projects/{id}/graph

**Auth:** required.

Query params:

| Param | Default | Notes |
|---|---|---|
| `q` | — | Substring filter on `id` or `label`, case-insensitive. Filters **nodes**; edges are only those between matching nodes, so `q` does not return a node's neighbours. |
| `nodeType` | — | Filter on the page's `type:` (first `type:` line, lowercased; `other` if none). Examples: `entity`, `concept`, `other`. Exact match: an unknown value returns 0 nodes; `node_type` (snake_case) is ignored. |
| `limit` | `200` | Clamped to `[1, 1000]`. |

Pages with `type: query` are **not** graph nodes (excluded on purpose, with their edges).

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
  ]
}
```

Edges are derived from `[[wikilink]]` references **and the frontmatter `related:` list** in `wiki/**/*.md`. Deduplicated by unordered pair `(source, target)` — `[[a]]` in b and `[[b]]` in a produce **one** edge. Self-edges are dropped. `weight` is `1.0`. Node `id` is the file stem, or the relative path when two pages share a stem; ambiguous links are dropped. The graph is rebuilt from the files on request and cached for 5 s, so a new page can take up to ~5 s to appear; there is no indexing state and embeddings play no part.

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

**0.4.x: returns 501.** On **0.6.13+** this is a real backend agent chat: see "Additions in app 0.6.13" below.

---

## Limits & defenses

| Limit | Value | Effect |
|---|---|---|
| Body size | 1 MiB | Exceed → 400. |
| File content read | 2 MiB | Exceed → 413. |
| File-tree node count | 10000 hard cap | Exceed → 413. |
| Search `topK` | 50 max | Silently clamped. |
| Graph `limit` | 1000 max | Values above 1000 silently clamped. **0.6.13:** `0` or non-integer → 400 (see "GET /projects/{id}/graph (pagination)"). |
| Rate limit | 120 req/sec (global, sliding 1 s window; `/health` and OPTIONS excluded) | 429, **no `Retry-After` header** — back off ≥1s. |
| In-flight requests | 64 concurrent | 503 "API server is busy" — back off ≥2s. |
| Chat streams / page embeds | 8 / 4 concurrent | 503. |

CORS (upstream v1 docs, **0.4.x only**): `Access-Control-Allow-Origin: *`. Preflight cached 10 min via `Access-Control-Max-Age: 600`. Allowed headers include `Authorization`, `X-LLM-Wiki-Token`, `Content-Type`.
**0.6.13** (`src-tauri/src/cors.rs`): no `*`; the request `Origin` is reflected (with `Vary: Origin` and `Access-Control-Allow-Private-Network: true`) only for `http://localhost[:port]`, `http://127.0.0.1[:port]`, `http://[::1][:port]`, `chrome-extension://`, `moz-extension://` and the Tauri origins; other origins get no `Access-Control-Allow-Origin`. `Access-Control-Allow-Methods: GET, POST, PATCH, DELETE, OPTIONS`.
Observed: reflected for `http://localhost:3000`, `http://127.0.0.1`, `http://[::1]:8080`, `chrome-extension://…`,
`tauri://localhost`, `http://tauri.localhost`, also on 401 responses; not for `https://example.com`, `null` or
`http://localhost.evil.com`. **The server still serves the request**: CORS only stops a browser from reading the
response, it is not access control. `OPTIONS` → 204 without auth, `Access-Control-Max-Age: 600`.

Non-`GET` / non-`POST` methods → 405 (**0.4.x only**). In 0.6.13: 405 only for verbs outside
GET/POST/PATCH/DELETE/OPTIONS (e.g. `PUT`), after the auth check; a known route with the wrong verb (e.g. `GET .../search`,
`POST /projects`) → **404**.

---

## Additions in app 0.6.13

> Source: app code at tag `v0.6.13` (`src-tauri/src/api_server.rs` unless noted). Check `/health` → `version` before relying on these.

### Common checks (in order)

429 above 120 req/s (not `/health`, not OPTIONS) · 503 above 64 in-flight requests · OPTIONS → 204 ·
400 "Request body too large" (1 MiB; **chat 40 MiB**; **pages/write 12 MiB + 16 KiB**) or non-UTF-8 body ·
503 API disabled · 401 missing token · 405 method not in GET/POST/PATCH/DELETE · 404 unknown route.

`POST chat`, `POST chat/{sessionId}/cancel`, `POST pages/write`, `POST pages/embed` **always** require the
token, even with `allowUnauthenticated`. Review `PATCH`/`DELETE` do not.

`/health` adds `mcpEnabled`, `allowLanAccess`, `agent: {chat, streaming, streamProtocol: "sse"}`.
`allowLanAccess` reflects the setting; the bind address (`0.0.0.0` vs `127.0.0.1`) is read only when the server starts,
and `LLM_WIKI_BIND_HOST` overrides it (`server_bind.rs`).

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
items by itself (`resolvedAction` `auto-resolved` or `llm-judged`) (`sweep-reviews.ts`). Types (`src/stores/review-store.ts`): `contradiction`, `duplicate`,
`missing-page`, `confirm`, `suggestion`.

### Review mutations (write `.llm-wiki/review.json`, no lock, mark-only)

| Route | Body | Response | Errors |
|---|---|---|---|
| `POST reviews/resolve` | `{ "ids": [..] (alias reviewIds), "action"?: "..." }` | `{ resolved: [], notFound: [], count }`, always 200 | 400 invalid JSON / empty ids |
| `PATCH reviews/{reviewId}` | `{ "resolved"?: true, "action"?: "..." }` (empty body = resolve) | `{ reviewId, resolved }` | 400, 404 not found |
| `DELETE reviews/{reviewId}` | — | `{ reviewId, deleted: true }` | 404 |

`resolved: false` reopens and clears `resolvedAction`. **Caveat:** the desktop UI keeps reviews in memory and rewrites
the whole `review.json` about 1 s after each change of its own store (e.g. an ingest adding reviews); API edits become
visible to the UI only after the Review tab's refresh. An API edit can therefore be overwritten. Treat it as a risk, not a rule. Re-list after a mutation to confirm.

### POST /projects/{id}/pages/write

Body (`deny_unknown_fields`): `{ "path": "wiki/...md", "content": "...", "allowOverwrite": false }`.
Path: must start with `wiki/` and end with `.md`; no leading `/`, no `..`, no segment starting with `.`, no
`<>:"|?*`, control chars, trailing space/dot or Windows reserved names. Content ≤ 2 MiB.

```json
{ "ok": true, "projectId": "...",
  "result": { "path": "wiki/concepts/x.md", "bytes": 1234, "allowOverwrite": false, "verified": true, "existedBefore": false } }
```

Errors: 400 invalid JSON / unknown field / "path is required" / bad path / too large; **409 existing page without
`allowOverwrite`**; 500 symlink, I/O, verification failure (previous content restored). Atomic write, byte-for-byte
verification, file history (`baseline` + `agent` versions). Does **not** update `index.md`, `log.md`, embeddings,
reviews or caches: keyword search sees the page at once, the graph within ~5 s, vectors only after `pages/embed`.

### POST /projects/{id}/pages/embed

Body: `{ "path": "wiki/...md", "force"?: false }`. Synchronous (provider timeout 300 s).

```json
{ "ok": true, "projectId": "...",
  "result": { "path": "...", "pageId": "...", "revision": "sha256:...", "chunks": 4, "vectorsWritten": 4,
              "status": "indexed" } }
```

`status` is `indexed` or `unchanged`. Errors: 400 "Embedding is not configured" / disabled / bad path / page over
2 MiB / `index.md`, `log.md`, `overview.md` / over 512 chunks / empty; 404 page; 503 more than 4 running;
502 provider; 504 timeout; 500 storage.

### POST /projects/{id}/chat

Body (camelCase, unknown fields ignored):

| Field | Default | Notes |
|---|---|---|
| `message` | required | |
| `sessionId` / `runId` | `api_<uuid>` / `run_<uuid>` | |
| `mode` | `standard` | `fast` \| `standard` \| `deep` \| `local_first` |
| `retrievalMode` | `standard` | `standard` \| `smart` \| `faithful` |
| `tools` | `{wiki: true, web: false, anytxt: false}` | **nested**: top-level `wiki`/`web`/`anytxt` are ignored |
| `topK` | 5 | clamped 1-10 |
| `includeContent` | false | |
| `history`, `historyExplicit` | — | without history the last 12 session messages are loaded |
| `skills`, `skillMode` | —, `explicit` | project skills from `.llm-wiki/skills` |
| `contextFiles`, `images`, `approvedShellCommands`, `shellCommand` | — | |
| `stream` | — | `true` = SSE; absent → SSE if `Accept: text/event-stream` |
| `persistSession` | true | |

JSON response: `{ ok, projectId, sessionId, mode, message: {role: "assistant", content}, references: [{title, path,
kind, snippet?, score?}], toolEvents: [{tool, status, detail?}], events: [], usage }`.
Errors: 400 invalid JSON / "message is required"; **499 "Agent turn cancelled"**; 502 any other agent error.

SSE: events `meta` `{projectId, sessionId, runId}`, `agent` `{type: agentStart | turnStart | toolStart | toolEnd |
referenceAdded | fileChanged | messageDelta | error | userInputRequired | done, ...}`, terminal `done` | `cancelled` |
`error`; `: keepalive` every 10 s; at most 8 concurrent streams (503). `agent` events may be dropped for slow
clients, the terminal one is not.

`POST chat/{sessionId}/cancel` (no body) → `{ ok: true, cancelled: bool, sessionId }`.

### GET /projects/{id}/graph (pagination)

`offset` ≥ 0 (else 400); `edgeScope` = `page` (default: edges with both ends in the page) | `filtered` (source in
the page, target among all filtered nodes); `limit` 0 or non-integer → 400 "limit must be a positive integer".
Response adds `offset`, `limit`, `totalCount`, `hasMore`.

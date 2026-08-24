# CLAUDE.md

Constraints live here; the reasoning behind them lives in the linked `docs/plan-*.md`. When a
rule here looks arbitrary, read its plan doc before changing it — most of them are scar tissue.

## Commands

```bash
npm run dev              # Go server (3002) + client (3000) in parallel
npm run server / client  # individual processes (server → `cd server-go && go run .`)

npm install && cd client && npm install   # client + root tooling deps (Go backend uses go modules)

# Tests
cd server-go && make check      # fmt-check + lint (vet + staticcheck) + offline unit tests
cd server-go && make test-int   # live-network suites (build tag itest)
cd client && npm test            # vitest suites (jsdom)

# Lint & format (client only, via oxc; Go backend uses gofmt/staticcheck through its Makefile)
npm run fmt && npm run lint:fix   # after changes — auto-format + auto-fix
npm run fmt:check && npm run lint # before commit — must both pass clean

# Deploy (full runbook: docs/rathole-vps-tunnel.md)
./scripts/deploy.sh              # build + sync to ~/Deploy/feedoverflow, kickstart the launchd service
launchctl kickstart -k "gui/$(id -u)/com.feedoverflow.app"   # force restart
tail -f ~/Deploy/feedoverflow/logs/app.log    # structured NDJSON (slog)
```

Do not silence lint errors or rewrite business logic just to make `lint` pass — if a correctness
rule flags real intent, surface it rather than auto-suppressing.

## Workflow

Single-person project — edit and commit directly on `main`. No feature branches or PRs required.

## Deployment

Single-user macOS app, publicly reachable at `https://rss.royl.uk:8443` via a rathole tunnel to an
Aliyun VPS running Caddy (TLS); the app itself runs on the Mac. Session-cookie auth
(`AUTH_USER`/`AUTH_PASS`) gates public access. Local-only traffic and the MCP server go through the
unauthenticated loopback API on `127.0.0.1:4002` (`LOCAL_API_PORT`), which is never tunneled.
Runbook: `docs/rathole-vps-tunnel.md`.

## Architecture

Three-panel RSS reader: **sidebar → article list → reader pane**.

The app durably persists **every** fetched article (not just starred ones) into `article_states`
for offline stats/research — every fetch path (on-demand reads, background refresh, startup
warming, the poller) goes through one shared chain (`internal/cache` → `internal/store`), with no
per-feed item cap and a 2GB DB size cap. There's no read/unread feature — articles only carry a
starred flag.

```
server-go/          Go backend (cgo binary, port 3002 — chi router, mattn/go-sqlite3)
  main.go           entrypoint: config → DB → logger → both listeners → background jobs
  internal/config   env config (PORT, LOCAL_API_PORT, RSS_DB, AUTH_*, DB_MAX_SIZE_MB, PUSH_SUBJECT, ...)
  internal/httpapi  Server struct + NewPublicRouter / NewLocalRouter; per-domain handlers
  internal/mcp      MCP server (Streamable HTTP) — 13 tools, mounted on NewLocalRouter only
  internal/db       SQLite open (WAL), schema + migrations
  internal/auth     session login/logout + per-request gate + login rate-limit
  internal/store    article_states writes — persist upserts, feed writes, adopt-orphans
  internal/cache    refreshFeed fetch chain + ensureFresh (TTL) + startup warming
  internal/favicon  favicon_cache read-through
  internal/jobs     poller, maintenance (orphan cleanup + size-cap + VACUUM), resource monitor
  internal/push     Web Push (VAPID) sender for per-feed update notifications
  internal/translate OpenAI-compatible client for per-feed LLM title translation
  internal/feed     gofeed RSS wrapper
  internal/ssrf     SSRF guard for outbound content/favicon fetches
client/             Vite + React + TypeScript (port 3000)
  src/App.tsx       top-level layout/auth/audio owner
  src/store.ts      zustand store — feeds/articles/views + all fetch logic
  src/types.ts      shared client types, mirrors server-go/internal/model
  src/components/   FeedSidebar, ArticleList, ArticleReader, ModalOverlay, ManageModal (+ its
                    FeedsPanel / CollectionsPanel / AddFeedPanel bodies), SettingsModal,
                    PodcastPlayer, LoginForm
  src/pages/        mobile single-pane wrappers (FeedsPage, ListPage, ReaderPage)
```

The mobile panel (订阅源 → 列表 → 文章) is plain React state in `App.tsx`, deliberately **not**
mirrored into browser history — don't reintroduce `pushState` here. Back is the in-app ← arrow
only; the edge-swipe does nothing. The iOS-only rendering bugs that came out of the coupling are
catalogued in `docs/plan-drop-mobile-history.md`.

Every modal renders through `ModalOverlay` — one portal, one backdrop, one Escape handler, one copy
of the keyframes. Backdrop dismiss requires the backdrop to own **both** the `pointerdown` and the
`click`: a `click` targets the nearest common ancestor of press and release, so an
`e.target === e.currentTarget` check alone closes the modal when a text drag-selection ends on the
backdrop. `onEscape` exists for modals whose Escape unwinds an inner step first (ManageModal's
sub-views); the backdrop still closes outright.

**Everything the sidebar lists is managed in one modal.** `ManageModal` is a shell — tab bar,
title, close button, Escape — over three bodies: `FeedsPanel` (订阅源), `CollectionsPanel` (合集),
and `AddFeedPanel`, which is a *sub-view* of the 订阅源 tab rather than a tab of its own. Its
`手动添加 | 导入 OPML` strip is two **methods**; the shell's tabs are two **things** — different
axes never share a row, and a sub-view replaces the tab bar rather than nesting under it (which is
also why an open collection editor can't have its unsaved rules swapped out from under it). The
shell owns all navigation, so the panels hold no `editing` state and no `ModalOverlay` of their
own. Entering at a sub-view (the toolbar's `+`) makes backing out of it *close* the modal —
`subIsEntry` — so a mis-clicked `+` never costs two Escapes. Rationale:
`docs/plan-merge-manage-modals.md`.

Three sidebar controls, three meanings, no overlap: 刷新, 管理 (`SquarePen` → ManageModal), 添加订阅
(`+` → ManageModal's add sub-view). The gear belongs to 设置 in the footer alone — it is
configuration, not content, and a second gear in the header meant two different things. 合集 has no
control of its own in the nav: the section is a plain list that renders only when non-empty, like
订阅源.

TypeScript, type-stripped by Vite/Vitest. `npm run typecheck` (`tsc --noEmit`, in `client/`) is
the type gate — Vite does not type-check.

**Data flow:** `store.ts` owns app state (`feeds`, `collections`, `articles`, `selectedView`,
`selectedArticle`, `starredCount`); components subscribe via `useStore`. `selectedView`:
`{ type: 'all' | 'today' | 'starred' | 'podcast' | 'feed' | 'collection' | 'search', feed?,
collection?, query?, scope? }`. Star uses optimistic updates — mutate local state immediately,
fire-and-forget POST to sync.

**Vite proxy:** `/api/*` → `http://localhost:3002`.

**UI signal-to-noise:** don't repeat information the current context already makes obvious (e.g.
hide the per-row feed name when a single feed is selected), keep labels in one consistent
language, drop stale/redundant chrome.

### Server (`server-go/`)

- **Two listeners share the same handlers.** `NewPublicRouter()` (all interfaces, auth-gated,
  static+SPA) and `NewLocalRouter()` (loopback `127.0.0.1:LOCAL_API_PORT`, no auth, also mounts
  `/mcp`). Auth is decided by which socket the request arrived on, not a header.
- RSS fetched via `gofeed` through the refresh chain: fetch upstream → persist all items into
  `article_states` → stamp `feeds.last_fetched_at`. No separate items cache — list endpoints read
  straight from `article_states`. `ensureFresh` per request: fresh → serve as-is; stale →
  background refresh; brand-new feed → await one fetch. Persist **upserts** on `article_id`:
  re-fetched items refresh content fields but never touch `is_starred`; `feed_id`/`feed_name`/
  `feed_url` are insert-only, so a live feed never re-homes an article. `content_updated_at`
  stamps only on genuine content changes.
- Deleting a feed purges its non-starred rows; starred rows keep `feed_url`, so re-adding the same
  URL re-adopts them (`adopt-orphans`).
- **Replacing a dead source is an edit, not a delete + re-add.** `PATCH /api/feeds/:id` takes a
  `url`; `feeds.id` never changes, so history stays continuous under it. The edit rewrites its own
  rows' `feed_url` (the one sanctioned exception to the insert-only rule above — a feed restating
  its own address, not another feed re-homing rows), parses the new URL upstream before storing it,
  and clears `last_fetched_at` so `EnsureFresh` takes its background-refresh branch. Why delete +
  re-add cannot substitute: `docs/plan-edit-feed-url.md`.
- Maintenance (`internal/jobs/maintenance.go`): orphan cleanup (non-starred rows whose feed is
  gone) + size cap (`DB_MAX_SIZE_MB`, default 2GB — trims oldest non-starred articles to 90%, then
  `VACUUM`s). Starred articles are never deleted.
- Article IDs: `md5(link || title+pubDate)` truncated to 12 chars.
- Podcast playback position lives in `article_states.play_position` (whole seconds), not the
  browser. Every write is an `UPDATE` — a ping carries only an id and a number, so inserting on a
  miss would mint a title-less row visible in every list. Non-NULL means "worth resuming"; the
  *client* decides an episode is finished and sends `DELETE`, so there is no duration column. `GET`
  caps at the 200 most recent (`play_updated_at`) and the client hydrates a map from it once at
  startup — the resume seek must stay synchronous with the play gesture. Rationale:
  `docs/plan-podcast-progress-sqlite.md`.
- **Collections** (`合集`) are saved queries over `article_states`, not sources: a collection is the
  *union* of its rules, each rule `feed AND include AND NOT exclude`, and it fetches nothing — no
  cache entry, no poller slot, no freshness handling, like `/api/all-articles`. The endpoint runs
  **one query per rule and merges in Go** (keeps the SQL static; taking `ListLimit` per rule is
  exact), deduping by `article_id`. Keywords match title + summary but **not** `content`. A
  **Latin-script keyword matches whole words only** — SQL does the coarse `LIKE` pass and Go
  re-checks survivors with an ASCII `\b` regexp; CJK keywords keep plain substring matching. A
  word-boundary *exclusion* is deliberately **not** applied in SQL, since a row SQL drops can never
  be recovered. A rule constraining neither feed nor keyword is rejected (400). Deliberately not
  wired to push — a collection is a lens, not a source. Rationale: `docs/plan-collections.md`.
- **List queries never name the `content` column.** Every list caller passes `withContent=false`;
  `store.articleColsNoContent` substitutes a literal in the same scan position, leaving
  `scanArticleRows` and every `Row` consumer untouched. Two reads keep the real column: `Starred`
  and `ArticleByID` (the push deep link), pinned by `TestContentCarryingReads`. This saves
  materializing body text into Go strings, **not** page reads — see the measured perf note in
  `docs/plan-collections.md` before attempting the column-order win it describes.
- Outbound content/favicon fetches pass through an SSRF guard (`internal/ssrf`).
- Push has two independent axes, deliberately not merged: `feeds.push_enabled` says *this source is
  worth a notification* (global, one row shared by every device), `push_subscriptions` says *this
  device receives* (one row per device). A device that never subscribed sees every bell as on and
  receives nothing — hence the explicit device row above the list in FeedsPanel.
  Deregistering is only ever that control's job: never a side effect of toggling a feed, or one
  device could silently cut off all the others.
- Push notifications are opt-in per feed (`feeds.push_enabled`, default off) and are sent **only
  from the poller** — an on-demand refresh must never notify about the article being read. "New" is
  the `feeds.last_notified_ts` watermark, bounded at both ends (`watermark < pub_ts <= now`) and
  stamped from the rows actually selected, so a future-dated item can't swallow every real update.
  Enabling push seeds the watermark to now. At most 3 articles per feed per poll, one notification
  each; surplus is dropped — never a "有 N 篇新文章" summary, which is an unread count, the one
  thing this reader has no concept of. Rationale + manual test steps:
  `docs/plan-push-notifications.md`.
- **Title translation** is **one global switch** (`llm_config.enabled`, default off), not per-feed —
  the worker's Han-ratio check (>30% Han → skipped, never sent) already keeps Chinese feeds free.
  The API key is the *capability*, the switch is the *intent*; both required. Runs in its own 30s
  worker (`internal/jobs/translator.go`), **not** off the poller, so it covers every fetch path and
  keeps upstream latency off the request path.
  - **One title per request, never a batch**: a model that drops an element in a batched response
    shifts every following translation onto the wrong article, silently and permanently.
  - The request carries the article's **feed name and summary** alongside the title
    (`translate.Request` → `userMessage`, as 来源/摘要/标题 lines). All of it is attacker-controlled
    feed text and all of it stays in the *user* message — that separation, not its length, is the
    injection mitigation; `clean`'s growth check still measures the answer against the **title**.
  - The system prompt must stay **byte-identical across calls** (cacheable prefix, and long enough
    to fix 翻译腔). Nothing per-article may move into it. Same rule for `exampleTurns`, which rides
    in that prefix.
  - **The wire is `system → fixed demonstration turns → the real request`.** A labelled multi-line
    user message reads as a form, and a form's likeliest completion is the same form filled in —
    which is how `来源：Hacker News\n标题：网页服务器…` got stored as a headline. `examplePairs` shows
    one such message answered with a bare line, in **both** shapes `userMessage` produces (with and
    without 摘要 — the two-line one is what actually failed). The example user messages are built by
    calling `userMessage`, never hand-written, so the demonstrated format cannot drift from the sent
    one. Format compliance comes from demonstration; prose rules are the fallback when there is
    nothing to demonstrate.
  - Sampling parameters are deliberately **not** sent — no `temperature`. Each title is translated
    exactly once and the result is permanent, so there is no run-to-run variance to converge; low
    temperature pushes output toward the literal register this prompt exists to avoid; and reasoning
    models (o-series, R1) reject `temperature != 1`, which the 400 retry would surface as
    "模型不可用". `response_format: json_object` was rejected for the same portability reason, plus it
    cannot help: a mirrored label lands inside the JSON value and still needs `clean`.
  - Model output is **untrusted input** — instruct, demonstrate, then validate. The prompt says
    不加引号 and `unwrapQuotes` exists anyway; 不加解释 and `maxGrowth` exists anyway; 不要重复标签 and
    `stripEchoedLabels` exists anyway. A demonstration moves the failure rate, never to zero, so
    none of those backstops may be dropped as "the prompt handles it".
  - Pending work is `title_zh IS NULL AND pub_ts > now − 24h`. That one bound keeps `title_zh IS
    NULL` from matching the whole historical table, decides how far back switching on reaches, and
    stops a permanently-failing row from blocking everything older than it.
  - `title_zh` is three-valued: NULL = pending, `''` = settled with no translation, non-empty = the
    translation. Only the worker distinguishes NULL from `''`; every reader treats both as "show
    the original". A failed request writes nothing and is retried; a successful one always settles.
  - Any OpenAI-compatible `/chat/completions` (`llm_config`, re-read every tick, editable from
    SettingsModal). `base_url` arrives over HTTP → restricted to https or loopback, and upstream
    response bodies are never echoed to the client.
  - The original title is never overwritten. The **list shows only the translation**;
    **ArticleReader shows both**; search matches `title` and `title_zh` alike.
  - Future article-level AI output (body translation, summaries) must **not** follow this shape:
    on-demand from the reader, output in a side table — never a body-sized column here.
  - **The endpoint runs locally** (Ollama, `http://localhost:11434/v1`), and the model it serves
    must be a **general instruct model that does not think**. Both halves are scar tissue:
    - A *dedicated translation* model (Hunyuan-MT-7B and friends) cannot run this prompt. Its chat
      template has no multi-turn support, so `exampleTurns` is silently dropped, and it has no
      instruction layer to honour 「只有「标题：」后面那一行需要翻译」 — measured, it translated the
      **摘要 instead of the 标题 in 6 of 10 titles**, and `maxGrowth` cannot catch that because a
      translated summary fits inside the bound. Its native single-instruction template is correct
      but discards both the summary context and the editorial register.
    - A *thinking* model empties every row. The app sends DeepSeek's `thinking:{"type":"disabled"}`;
      Ollama ignores it without a 400, so the 400-retry never fires, and `maxCompletionTokens`
      truncates the model mid-thought into **empty content** — which the worker settles as "no
      translation, don't retry", permanently. Stock `qwen3:8b` translated 0 of 10.
    - The fix is baked into the model artifact, not the request: `scripts/ollama/Modelfile.translate`
      derives `feedoverflow-translate` from `qwen3:8b` with `/no_think` forced in the template. That
      keeps `internal/translate` a plain OpenAI-compatible client with no per-provider branch —
      Ollama's `reasoning_effort:"none"` would work too, and is deliberately not sent for that reason.
      Recreate it after an Ollama reinstall or the app silently stops translating.
  - Rationale: `docs/plan-title-translation.md`, `docs/plan-translation-context.md`,
    `docs/plan-local-llm-ollama.md`.
- Auth: when `AUTH_USER`/`AUTH_PASS` are set, every `/api/*` request on the public router requires
  a valid session cookie (no localhost bypass — gated by socket, not IP). Login is rate-limited.

**SQLite tables:**
- `feeds(id, name, url, last_fetched_at, push_enabled, last_notified_ts)`
- `collections(id, name, position, created_at)` + `collection_rules(id, collection_id, feed_id, include, exclude)` — saved multi-feed streams
- `article_states(article_id, feed_id, feed_name, feed_url, title, link, pub_date, pub_ts, summary, content, author, audio_url, audio_duration, is_starred, updated_at, content_updated_at, play_position, play_updated_at, title_zh)` — durable record of every fetched article
- `settings(key, value)` — e.g. `rsshub_base_url`
- `sessions(token, created_at)` — 30-day TTL
- `favicon_cache(domain, image, content_type, fetched_at)` — 30-day positive / 1-day negative TTL
- `push_subscriptions(endpoint, p256dh, auth, user_agent, created_at)` — one row per registered device
- `push_keys(id, public_key, private_key)` — the single VAPID keypair; deliberately *not* in
  `settings`, which `GET /api/settings` serializes wholesale
- `llm_config(id, base_url, api_key, model, enabled)` — the single translation endpoint + the
  global on/off switch; out of `settings` for the same reason as `push_keys`

**API:**
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/feeds` | list feeds |
| POST | `/api/feeds` | add feed |
| POST | `/api/feeds/import-opml` | bulk import from OPML |
| PATCH | `/api/feeds/:id` | rename feed, repoint its `url`, and/or toggle `push_enabled` (all optional) |
| DELETE | `/api/feeds/:id` | remove feed + purge its non-starred articles |
| GET | `/api/feeds/:id/articles` | articles for one feed, up to 500; `?summary=1` |
| GET | `/api/collections` | list collections with their rules |
| POST | `/api/collections` | create a collection (name + rules) |
| PATCH | `/api/collections/:id` | rename and/or replace rules (both fields optional) |
| DELETE | `/api/collections/:id` | remove a collection (articles untouched) |
| GET | `/api/collections/:id/articles` | the collection's merged stream; `?summary=1` |
| GET | `/api/all-articles` | merged + sorted, up to 500; `?mode=latest\|digest`, `?summary=1` |
| GET | `/api/today` | today's articles, same `?mode=` toggle; `?summary=1` |
| GET | `/api/starred` | starred articles |
| GET | `/api/podcasts` | episodes with a non-empty `audio_url` |
| GET | `/api/starred/count` | badge count |
| POST | `/api/articles/star` | upsert `is_starred` |
| GET | `/api/articles/:id` | one article (content included) — used only by the push deep link |
| GET | `/api/articles/:id/content` | cached full content |
| GET | `/api/fetch-content?url=` | Readability extraction |
| GET | `/api/favicon?domain=` | cached feed favicon (BLOB) |
| GET\|POST | `/api/current-article` | in-memory "currently open" article (for MCP) |
| GET | `/api/podcast-progress` | recent playback positions, id → seconds (200 newest) |
| POST | `/api/podcast-progress` | upsert one episode's position |
| DELETE | `/api/podcast-progress/:id` | forget one episode's position |
| GET\|PATCH | `/api/settings` | read/update settings |
| POST | `/api/login` `/api/logout` | session auth |
| GET | `/api/auth-check` | whether the request is authed |
| GET | `/api/push/key` | VAPID public key (generated on first call) + device count |
| POST | `/api/push/subscribe` `/api/push/unsubscribe` | register/drop this device's push endpoint |
| GET\|PATCH | `/api/llm/config` | translation endpoint/model/`enabled` + whether a key is stored (`key_set`; the key itself is never returned) |
| POST | `/api/llm/config/test` | translate one fixed string through the stored config |

### MCP server (`internal/mcp`)

Mounted at `POST /mcp` on `NewLocalRouter` only (loopback, no auth by design). 13 tools, each a
thin self-call into `http://127.0.0.1:LOCAL_API_PORT/api/...` (`internal/mcp/client.go`) rather
than duplicating `internal/httpapi`'s handler logic: `list_feeds`, `add_feed`, `rename_feed`,
`delete_feed`, `import_opml`, `get_all_articles`, `get_today_articles`, `get_starred_articles`,
`get_feed_articles`, `get_starred_count`, `toggle_star`, `get_current_article`,
`fetch_article_content`.

The three list tools call their endpoint with `?summary=1`; the two cross-feed ones also pin
`?mode=digest` — digest doesn't shrink the response (both modes cap at 500), it changes who fills
it, so one high-volume feed can't read to an agent as "the other feeds published nothing". The list
endpoints strip `summary`/`content` by default for the browser's sake, but an MCP client has no
reader pane, so bare titles give it nothing to decide on. `?summary=1` adds back only the RSS
summary — never `content`, which stays behind `fetch_article_content`. `get_starred_articles` needs
no flag: `/api/starred` has always returned both.

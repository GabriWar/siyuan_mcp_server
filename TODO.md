# TODO

Gaps observed while using this MCP server in a heavy real-world session
(porting an Obsidian vault → SiYuan: 160 docs, ~12k blocks, ~1900 block refs,
plus flashcard creation). Things that forced me to drop down to direct
`curl`/`fetch` calls against `/api/*` instead of using MCP tools.

## Missing API coverage

- [ ] **`/api/riff/*` family** — flashcards / spaced repetition. None of these are wrapped:
  - [ ] `riff/getRiffDecks`
  - [ ] `riff/createRiffDeck`
  - [ ] `riff/renameRiffDeck`
  - [ ] `riff/removeRiffDeck`
  - [ ] `riff/addRiffCards`
  - [ ] `riff/removeRiffCards`
  - [ ] `riff/getRiffCards`
  - [ ] `riff/getRiffDueCards`
  - [ ] `riff/reviewRiffCard`
  - [ ] `riff/getNotebookRiffDueCards` (notebook-scoped review)
  - [ ] `riff/getTreeRiffDueCards` (doc-tree-scoped review)
- [ ] **`/api/attr/getBlockAttrs`** is missing (we only expose `getBlockAttrs` indirectly via SQL `ial` field, which lags the index)
- [ ] **`/api/filetree/createDocWithMd`** — used to bulk-create docs from markdown with hpath. Currently `create_doc` exists but it's worth exposing the lower-level endpoint too for hpath-driven imports.
- [ ] **`/api/notebook/getRecentDocs`**, **`getDocsByHPath`** — for navigation use cases
- [ ] **`/api/system/*`** — `getConf`, `setUILayout` — for plugin/theme integrations
- [ ] **`/api/search/fullTextSearchBlock`** — full-text search returning structured matches (right now you have to roll your own SQL)

## Bulk / batch operations

- [ ] **`bulk_update_blocks`** — accepts `[{id, dataType, data}, ...]`. I had to update 700+ blocks one at a time in a Python loop; that's 700 round-trips through MCP. A batch tool would cut that to one.
- [ ] **`bulk_set_block_attrs`** — same story for setting icons on 143 docs.
- [ ] **`bulk_delete_blocks`** — for stripping frontmatter remnants etc.
- [ ] **`bulk_create_docs`** — accept a list of `{notebook, hpath, markdown}` for vault imports.

## SQL ergonomics

- [ ] **Default LIMIT silently caps at 64**. This bit me hard — I thought I had 64 docs when I actually had 160. The wrapper should either:
  - default to a much higher limit (e.g. 10k), or
  - explicitly require `LIMIT` and reject queries without it, or
  - auto-paginate when results are truncated and tell the caller.
- [ ] **`sql_query` description** should mention the default limit.
- [ ] **`sql_query_paginated`** — convenience wrapper that pages until exhausted.

## DX / docs

- [ ] **Tool descriptions are Chinese-only** in `inputSchema.properties[].description`. English fallback (or bilingual) would help English-language MCP clients render them readably. (Even just replicating each Chinese description in English would 10x scannability for tool listings.)
- [ ] **`upload_asset` Content-Type bug**: returns `request Content-Type isn't multipart/form-data` when called via the MCP wire format. The tool needs to assemble a real multipart body itself, not just forward JSON. Workaround was `cp` to `data/assets/` directly.
- [ ] **README** doesn't list the tool catalog — would help users know what's wrapped vs. what isn't before they start.
- [ ] **Examples**: a short "porting markdown into SiYuan" recipe, since that's a common ask.

## Reliability

- [ ] **Retry logic for transient `RemoteDisconnected`** — when SiYuan is busy reindexing (e.g., right after `updateBlock` storms), it occasionally drops the next request. A small client-side retry/backoff inside the MCP wrapper would smooth this out.
- [ ] **Surfacing API error codes**: when the underlying SiYuan API returns `code: -1, msg: "..."`, the MCP tool currently returns `{ code: -1 }` as success-shaped JSON. Should return as an MCP error so clients can react.

## Nice-to-haves

- [ ] **A `bulk_replace` tool** that takes a regex + replacement and runs over a notebook/doc subtree. Saved hundreds of round-trips during the wikilink → block-ref conversion.
- [ ] **Permissions / scope**: per-tool allow/deny is already MCP's domain, but a `read_only` mode for the whole server (no destructive endpoints) would let you point this at a teammate's instance safely.
- [ ] **Resource exposure** (`ListMcpResources`): expose notebooks and docs as MCP resources so clients can browse them as a tree without SQL.

## What I was wrong about (originally claimed missing — actually present)

- ✅ `set_block_attrs` exists (used for setting doc icons in bulk).
- ✅ `update_block`, `delete_block`, `insert_block`, `append_block`, `prepend_block` all exist.
- ✅ `get_child_blocks` exists (was just deferred and needed `ToolSearch` to load).
- ✅ `get_block_attrs` exists (returns the up-to-date attrs even when SQL `ial` is stale).

In a long session, the deferred-tool / `ToolSearch` round-trip is enough friction
that I sometimes reached for `curl` instead. That's not really a server bug — it's
an MCP-client UX issue. But surfacing the most common ~15 tools eagerly (without
deferring) would help.

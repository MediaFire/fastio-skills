---
name: fast-io
description: >-
  Workspaces for agentic teams. Essential agent guide: the 18 consolidated tools
  (action-based routing), authentication, the MCP-server mechanics specific to this
  server, and the built-in how-to tool, which answers product how-tos, parameter
  details, and step-by-step task recipes on demand so the guide stays lean. Use this skill
  when agents need shared workspaces to collaborate with other agents and humans,
  create branded shares (Send/Receive/Exchange), or query documents using built-in AI.
  Supports ownership transfer to humans and workspace management.
  Free agent plan with 50 GB storage and 5,000 monthly credits.
license: Proprietary
compatibility: >-
  Requires network access. Connects to the Fastio MCP server at mcp.fast.io
  via Streamable HTTP (/mcp) or SSE (/sse).
metadata:
  author: fast-io
  version: "2.6.0"
homepage: "https://fast.io"
---

# Fastio MCP Server -- AI Agent Guide

**Version:** 2.6
**Last Updated:** 2026-07-20

> **Platform reference.** For a comprehensive overview of Fastio's capabilities, the agent plan, key concepts, and upgrade paths, see [references/REFERENCE.md](references/REFERENCE.md).

This guide is deliberately short. It covers what an agent must know **before it would think to ask anything**: what the server is, the two modes, how to authenticate, the tool menu, **how to ask the `how-to` tool**, the MCP-server mechanics that are specific to *this server* (uploads, blobs, overwrite semantics, notes-vs-files, code-mode contracts, response hints), and the product guardrails that get an agent into trouble silently.

**For everything else — how to accomplish a product task, parameter details, the full per-action tool reference, step-by-step task recipes, concept deep-dives — ask the `how-to` tool or call `<tool> action="describe"`.** This guide intentionally does NOT duplicate the product how-to corpus.

> **Why MCP mechanics stay here:** the `how-to` corpus is platform-owned and product/REST-oriented — it does **not** know this MCP server's mechanics (the `POST /blob` sidecar, code-mode `search`/`execute`, the `_next`/`_warnings`/`_recovery` envelope, in-place overwrite/versioning). `surface=code` only changes phrasing, not knowledge. So those sections are load-bearing and live here.

> **Versioned guide.** This guide is updated with each server release. If you hit unexpected errors, the guide may have drifted since you last read it — re-read it.

---

## 1. Overview

**Workspaces for Agentic Teams. Collaborate, share, and query with AI -- all through one API.**

Fastio provides workspaces for agentic teams -- where agents collaborate with other agents and with humans. Upload outputs, create branded shares, ask questions about documents using built-in AI, and hand everything off to a human when the job is done. New orgs run on a paid plan (see *Plans & Billing*).

**All API access goes through the MCP tools.** Do not make direct HTTP calls to `api.fast.io` or the MCP server -- the tools handle authentication, session management, error recovery, and response formatting. The only exceptions are binary transfers: `POST /blob` (uploads), the pre-authenticated download URLs tools return, and the `GET /file/...` pass-through routes for large files. Once you authenticate, the token is stored in the server session and auto-attached to every subsequent call — there is no need to pass tokens between invocations.

### Two Modes

The server exposes one of two tool sets, chosen automatically from the MCP client's `clientInfo.name`:

- **Named mode (18 tools)** — action-routed tools covering the full REST surface. Served to named clients and as the safe default for unknown clients.
- **Code mode (5 tools: `auth`, `upload`, `search`, `execute`, `how-to`)** — a lightweight set for headless agents. See Section 6.

**Client → mode mapping** (from `clientInfo.name`):

| `clientInfo.name` (case-insensitive substring; code-mode checked first) | Mode |
|---|---|
| `claude-code`, `claude code`, `cursor`, `continue`, `cowork`, `claude-cowork`, `codex`, `antigravity`, `gemini-cli`, `grok-cli`, `opencode` | **Code** |
| `claude-desktop`, `cline` | **Named** (explicit) |
| anything else — incl. bare `openai`, `chatgpt`, `gemini`, `grok`, and unknown/unset | **Named** (safe default) |

(There is no `*-cli` wildcard — only the exact code-mode substrings above match; bare `gemini`/`grok` are Named.)

### Server Endpoints

- **Production:** `mcp.fast.io` — **Development:** `mcp.fastdev1.com`
- Two transports on each: **Streamable HTTP at `/mcp`** (preferred for new integrations) and **SSE at `/sse`** (legacy).

### Resources & Prompts

MCP resources (read via `resources/list` / `resources/read`): `skill://guide` (this guide), `session://status` (auth state), `resource://status` (server health, no auth), plus `download://...` file templates (workspace/share files; up to 100 KB inline base64, larger fall back to the `GET /file/...` pass-through). No MCP prompts are registered.

For deeper lookups: REST API reference at `https://api.fast.io/llms.txt`; platform guide at [references/REFERENCE.md](references/REFERENCE.md).

---

## 2. Ask `how-to` and `describe` (READ THIS FIRST)

This guide covers only the essentials + MCP-server mechanics. **For product how-tos, parameter details, the full tool reference, and step-by-step task recipes, use these two reflexes instead of improvising.**

### `how-to` — "how do I…?" (the primary deferral target)

Call **`how-to action=ask question="..."`** whenever the right *approach* on Fastio isn't obvious — a multi-step or unfamiliar task (branded shares, metadata extraction, ownership transfer, billing). It returns the canonical, product-aware sequence of steps so different agents converge on the same correct path.

- **FREE** — no credits, no org, no plan gate, no billing. Requires only an authenticated user.
- **EXPLAIN-ONLY** — it returns guidance; you then act on it with the other tools. It never creates, updates, or deletes anything.
- **Available in BOTH modes.** In named mode answers are phrased as named-tool calls (`<tool> action="…"`); in code mode as `execute` calls. `how-to` is itself a dedicated tool in both modes — **call it with `action="ask"`, NEVER via `execute`.**
- **Optional `context`** (≤8000 chars) — untrusted background, e.g. a pasted error or what you've already tried.
- **Two HTTP-200 response shapes:**
  - `{status:"answer", answer, escalated, topics_used}` — a grounded answer; read it, then act.
  - `{status:"needs_clarification", questions[]}` — normal, not an error. Resolve with the user, fold the clarifications into `question`/`context`, and re-ask (the tool is stateless).
- A `429` means back off (code `10368`) or a prior request is still running (retry shortly).
- **vs the `ai` tool:** `how-to` answers questions about Fastio the PRODUCT. `ai action=ask` performs RAG over the user's OWN uploaded files.

> **What `how-to` does NOT know:** MCP-server mechanics (blob staging, code-mode `search`/`execute`, the response-hint envelope, overwrite/versioning semantics). Those are in Section 5 of this guide, not in the corpus.

### `describe` — a tool's actions and parameters

Every consolidated tool supports `action="describe"` (no auth, no other params required). It returns a structured payload of the tool's actions and their required/optional params, notes, and `param_details`. For large tools the default returns a compact action INDEX; pass `describe_action="<action>"` to drill into one action. **Call `describe` the first time you use an unfamiliar tool** rather than guessing parameters.

In **code mode**, use the `search` tool (`target="api"`) to discover endpoints, then `execute` to call them (see Section 6).

---

## 3. Authentication (Critical First Step)

Authentication is required before any tool except these **unauthenticated** ones: `auth` actions `signin`, `signup`, `set-api-key`, `pkce-login`, `email-check`, `password-reset-request`, `password-reset`; and `download` action `quickshare-details`.

### Which approach?

| Situation | Approach |
|---|---|
| **Operating autonomously** (storing files, building for users) | Create your own agent account: `auth action=signup` (sends `agent=true` automatically — never sign up as a human). Agent accounts can later transfer their org to a human. Creating an org still requires a paid plan via `org action=billing-create`. |
| **Assisting a human** who already has an account | Use their API key: `auth action=set-api-key`. You operate as the human; the key is validated and stored in the session. Keys can be scoped/tagged/expiring. Manage keys with `auth` actions `api-key-create/-update/-list/-get/-delete`. |
| **Running headless / no browser** | Use signup or an API key — do **NOT** use PKCE. |
| **Signing in without sending a password** (human + browser present) | Browser-based PKCE: `auth action=pkce-login` → user approves in browser → `auth action=pkce-complete` with the returned code. Supports scoped access via `scope_type`. Not for headless agents. |

**Essential flows (the rest is in `auth action=describe` / `how-to`):**

- **Sign-in:** `auth action=signin` with `email`+`password` → JWT stored in session automatically.
- **Account creation:** → `auth action=signup` with `first_name`/`last_name`/`email`/`password` → then **`auth action=signin`** (signup does NOT auto-sign-in, and returns a uniform response for new vs existing emails so existence is never revealed) → `auth action=email-verify` (twice: send code, then verify with `email_token`) before using most endpoints. (`email-check` is **deprecated** — it no longer reports availability; skip it. An **existing** email is not an error — signup emails it a sign-in/reset link; see Section 6 guardrails.)
- **2FA:** if `signin` returns `two_factor_required: true`, the token is limited-scope — call `auth action=2fa-verify` with the code to upgrade. **Inline 2FA:** `api-key-create`/`api-key-delete` need a 2FA `token` param when 2FA is enabled (check `auth action=2fa-status`); API-key sessions bypass inline 2FA entirely.
- **PKCE:** `auth action=pkce-login` (optional `email`, `scope_type`, `agent_name`) → user approves → `auth action=pkce-complete code=...`.
- **Session status:** `auth action=status` (local DO check, no API call — returns auth state, expiry, scopes, and `session_expired`/`expired_reason` if lapsed) vs `auth action=check` (validates against the API).
- **Expiry / refresh:** OAuth/PKCE sessions auto-refresh silently (1-hour access tokens, 30-day refresh chain). Basic JWT sessions last 30 days, no refresh. API keys don't expire unless `key_expires` is set.
- **Signout:** `auth action=signout` clears the session.

---

## 4. Tool Menu

### Named mode — 18 tools

Action-routed; call `<tool> action=describe` for the per-action reference.

- **`auth`** — Sign-in/sign-up, 2FA, API key management, OAuth/PKCE sessions. The starting point.
- **`user`** — Current user profile, contacts, invitations, user assets, account eligibility, shares you belong to.
- **`org`** — Organization CRUD, members, billing/subscriptions, workspace creation, invitations, assets, org discovery, ownership transfer, custom domains.
- **`workspace`** — Workspace settings & lifecycle (update/delete/archive), shares listing/import, assets, discovery, notes (create/read/update), async-job status, and the consolidated `metadata-*` actions.
- **`share`** — Share CRUD (Send / Receive / Exchange), public details, archiving, password auth, members, name checks, and AI titling.
- **`fileshare`** — Durable, single-file share links (replaces deprecated QuickShare). Binds to one file; access tiers, password, expiry, per-user grants, version history, external-editor write-back.
- **`storage`** — Files & folders in workspaces and shares: list/search/move/copy/rename/delete/purge/restore, versions, locking, preview URLs, node-level metadata. Requires `profile_type` (`workspace`|`share`).
- **`metadata`** — AI metadata templates + the unstructured-data extraction pipeline, saved views, and lexical metadata search (workspace level).
- **`find`** — Unified search across a workspace or share: one query, results grouped into independently-paginated buckets (files, metadata, comments). For one result type prefer `storage action=search` or `metadata action=search`.
- **`upload`** — File uploads: chunked lifecycle, single-call streaming, bulk batch, web imports from URLs, limits/extensions. **Files/binaries default to the `POST /blob` sidecar → `blob_id`**; `content_base64` is a fallback for clients that can't reach `/blob`. See Section 5.
- **`download`** — Generate download / ZIP URLs. MCP can't stream binary, so these return pre-authenticated URLs / `resource_uri`s. Requires `profile_type` for file/zip URLs.
- **`ai` (Ripley)** — Read-only delegation over the platform RAG agent: ask a natural-language question about workspace/share content, get a cited answer. Never does content CRUD; consumes AI credits — don't re-call `ask` to retry, poll the existing chat. Requires `profile_type`.
- **`comment`** — Comments on files, scoped to `{entity_type}/{parent_id}/{node_id}`: add (with optional anchoring), reply, delete, reactions.
- **`event`** — Audit/activity log with rich filtering, AI activity summaries, event details, activity polling, plus the per-member **Dashboard** feed (`dashboard-*`).
- **`member`** — Member management for workspaces and shares (add/remove/update roles, transfer ownership, join/leave). Includes pending (invited) members. Requires `entity_type`.
- **`invitation`** — Invitation management for workspaces and shares (list, list-by-state, update, delete). Requires `entity_type`.
- **`asset`** — Asset upload/delete/list/read for orgs, workspaces, shares, users. Requires `entity_type`.
- **`how-to`** — Built-in product help: ask natural-language "how do I…" questions about Fastio (FREE, explain-only). See Section 2 — reach for this before improvising.

### Code mode — 5 tools (headless agents)

| Tool | Purpose |
|------|---------|
| `auth` | Authentication (signin, signup, API keys, PKCE, 2FA) |
| `upload` | File uploads (chunked, text, web-import) |
| `search` | Find content (`target="content"`, default) OR discover API endpoints (`target="api"`) |
| `execute` | Make authenticated API calls to Fastio (structured method/path/body/params — no eval) |
| `how-to` | Product help — ask "how do I…"; answers phrased as `execute` calls |

See Section 6 for the `search` / `execute` contracts.

---

## 5. MCP-Server Mechanics (load-bearing — not in the how-to corpus)

These are mechanics of *this MCP server*. `how-to` does not know them — get them right from here.

### Upload strategy — pick the FIRST row that matches

**For any file or binary, stage the bytes via the `POST /blob` sidecar and pass `blob_id` — that is the default** (it bypasses the MCP transport entirely; `create-session`/`blob-info` hand you a ready-to-run `curl` command). Then pick the action: `stream-upload` (no `filesize` needed) unless you know the exact byte count (chunked). `content_base64` is the fallback **only** for clients that can't reach `/blob`. Batch is a specialized option only for "multiple small files in one shot." Read top-to-bottom:

| Situation | Size Known? | Recommended Approach |
|---|---|---|
| Any file with a URL | N/A | `upload action=web-import` (single step) |
| Any file/binary (DEFAULT) — incl. unknown/generated size | No | **`POST /blob` → `upload action=stream-upload` with `blob_id`** (single call — auto-finalizes, **no `filesize` required**). Tiny inline text may pass `content` directly; use `content_base64` only if your client can't reach `/blob`. |
| File with KNOWN exact byte count (already on disk, pre-measured) | Yes | `POST /blob` → `upload action=create-session` with `filesize` → `chunk` with `blob_id` → `finalize`. **`filesize` MUST match the bytes exactly — mismatch fails `finalize` with code `10522` and forces a session cancel.** |
| **Specialized:** several small files at once (≤4 MB each) | Yes | `POST /blob` per file → `upload action=batch` with a `files[]` manifest (one round-trip, up to 200 files; not for single uploads) |

> **⚠️ Never guess `filesize` for content you haven't produced yet.** A common failure: pick `create-session` with a guessed `filesize` (e.g. 8000), generate the content (4443 bytes), then `finalize` rejects with code `10522` (`chunks (4443) do not match size (8000)`). **The session cannot recover — it must be canceled and retried.** Use `stream-upload` for any generated, transformed, or unknown-size content — it auto-detects size and auto-finalizes.

**Stream restrictions:** stream sessions cannot use `chunk`/`finalize` (406); chunked sessions cannot use `stream` (406); stream is single-shot. For files approaching/exceeding the `POST /blob` 100 MB cap, switch to the chunked flow and call `upload action=limits` first to confirm the plan's max file size.

> **Binary vs text content.** `content` is **text-only** (stored verbatim UTF-8). For binary, use `content_base64` (server-decoded) or `POST /blob` → `blob_id` — putting base64 in `content` corrupts the file. (`folder_id` aliases `parent_node_id` on `create-session`/`stream-upload`/`web-import`, but on `batch` `folder_id` is the canonical name.)

### `POST /blob` sidecar — the standard large-file path

A raw-HTTP endpoint outside the JSON-RPC pipe — **bypasses MCP transport limits entirely** (no base64 overhead, no parameter-size constraints). The `create-session` response includes a `blob_upload` object with the endpoint URL, your session ID, and a ready-to-use `curl` command (or call `upload action=blob-info`). POST the raw bytes, get back `{ "blob_id": "<uuid>", "size": <bytes> }` (HTTP 201), then consume the `blob_id` via `upload action=chunk`/`stream`/`batch` (and `create-note`/`update-note` for large notes).

**Blob constraints:**
- Blobs expire after **5 minutes** — stage and consume promptly.
- Each blob is **single-use** (deleted on first use).
- Maximum blob size: **100 MB**.
- Auth is the **same session** — the `/blob` POST carries your `Mcp-Session-Id` header (provided in the `blob_upload` object / curl command). SSE transport clients must add `?transport=sse` to the `/blob` URL.

### Storage overwrite & versioning (REPLACE by default, in place)

**Do NOT delete-and-re-upload to "update" a file — that is a data-loss trap.** Same-name uploads into the same parent folder **overwrite the existing node in place, preserving the `node_id`.** The prior content is kept as a recoverable version. Deleting the old node first is wasted work, breaks `node_id` references held by other entities (comments, metadata, links), and can leak the file into trash.

**Correct update-a-file pattern:** `upload action=create-session` with the **same** `parent_node_id` and **same** `filename` (+ `profile_type`/`profile_id`/`filesize`) → `POST /blob` → `chunk`/`finalize` (or `stream`). The `node_id` is unchanged; the previous content becomes a version. Inspect/roll back with `storage action=version-list` / `version-restore`. (For a deterministic overwrite when the filename may have drifted, pass `target_node_id` on `create-session` — server uses `action=update` + `file_id`; `parent_node_id` ignored, `filename` optional for rename-on-replace.)

**Name-conflict behavior across operations:**
- **Upload (addfile):** silently overwrites in place; prior content kept as a version; `node_id` stable.
- **Move / Copy / Restore-from-trash:** trash the existing conflicting file first, then complete (old file recoverable from trash).
- **Folder conflicts / type mismatches** (file vs folder) still fall back to rename (e.g. `folder (2)`).

If you specifically need two same-name files to coexist, **rename first**, then upload.

### Notes vs Files (NOT interchangeable, even for markdown)

A `.md` uploaded via the file-upload flow is a **File** (`type:"file"`), not a **Note** (`type:"note"`). Reading a markdown File returns text and "looks like" a note, but `update-note`/`read-note` reject it with `Node is not a note` (error `153548`).

- **Editing content incrementally from an agent** → use `workspace action=create-note` (pass markdown as `content`; do NOT upload it as a file first). Read with `read-note`, edit with `update-note`.
- **Static artifact** (report/export/attachment) → use the upload flow; accept that `update-note` won't work later. To change the bytes, re-upload same `parent_node_id`+`filename` (in-place overwrite + version).
- **Recovery when `update-note` fails with `153548`:** (a) keep it a File → use the upload-overwrite path; or (b) convert to a Note → `storage action=delete` the File, then `create-note` (produces a NEW `type:"note"` node). **Do NOT delete-and-re-upload via the upload flow** — that creates another File and you hit `153548` again. Verify type via `storage action=details` (`type`: `file`/`note`/`folder`/`link`).
- **Note limits:** content max **100 KB** per node; writes ≥80 KB return a non-fatal `_warnings` rollover hint. Name 1-100 chars ending `.md`. Large notes (>~10 KB): pass content via `blob_id` (`POST /blob`) instead of inline `content` to avoid MCP transport overhead. `content` and `blob_id` are mutually exclusive.

### Batch-upload semantic traps

`upload action=batch` posts up to **200 files**, **≤4 MB each**, **≤100 MB total**, auth required (anonymous → HTTP 401 code `10011`; use single-file `create-session` for public-receive shares). Three traps:

1. **`node_id` is nullable on success.** Async storage finalization returns `"status":"ok"` with `"node_id": null` — assigned later by the assemble worker. **This is SUCCESS, not failure.** For the final id, `storage action=list` the folder after a short delay.
2. **Partial success is HTTP 200** with `count_errored > 0`. **Do NOT retry the whole batch** — inspect `results[]`, split by status, retry only retryable errored entries.
3. **All-failed still returns HTTP 200** (`all_failed: true`). Nothing uploaded; inspect `results[]`/`errors[]`, fix inputs, resubmit.

Always inspect per-item `results[]`. (Whole-batch HTTP-4xx rejections with no `results[]` are input-validation failures — fix the input.)

### AI chat (Ripley) mechanics

`ai` chat is **read-only** — it answers questions about file contents; it cannot modify files/settings/members. Two file-context modes for `chat_with_files`, **mutually exclusive**:

- **Scope (RAG)** — `files_scope` / `folders_scope`. **Requires workspace intelligence enabled**; files must be `ai_state: indexed`. Omit scope to search the whole workspace (recommended default). Narrows the RAG search boundary; populates `citations`.
- **Attachments** — `files_attach` (`nodeId:versionId`, max 20 files / 200 MB). **No intelligence required.** **Preflight: only files with `ai.attach: true` can be attached** — check via `storage action=details` (full detail; `ai.attach` is omitted at terse/standard). FILES ONLY — a folder nodeId returns 406. `citations` is always empty in this mode.

Sending both scope and attach errors. After `chat-create`/`message-send`, the response is async; `ai action=message-read` does a bounded activity long-poll (~24s worst case: a 12s ceiling × 2 iterations). If still processing, **don't tight-loop** — use `event action=activity-poll` (see Activity polling).

### Activity polling — don't tight-loop

Three mechanisms, most-to-least preferred: **`event action=activity-poll`** (long-poll, server holds up to ~95s, returns activity keys like `ai_chat:{chatId}`/`storage`/`members` + a `lastactivity` timestamp), WebSocket (live UIs), `event action=activity-list` (one-time snapshot). **Do NOT poll detail endpoints (e.g. `ai action=message-read`) in tight loops** — long-poll for the change, then fetch the detail once. For AI completion: poll until an `ai_chat:{chatId}` key matching your chat appears, then `message-read` once. **Param split:** `event action=activity-list` takes `profile_id` (alias `context_id`; `profile_type` is optional/ignored) and `event action=activity-poll` takes `entity_id` — these two are NOT interchangeable. The `event` `search`/`summarize`/`details` family takes `workspace_id`/`share_id`/`org_id`/`user_id` filters — mixing the activity and search families errors (code `10262`).

### Downloads

MCP never streams binary — tools return URLs. `download action=file-url` (needs `profile_type`) returns a temporary pre-authenticated URL; `download action=zip-url` returns the URL **plus the required `Authorization` header value** (the zip fetch needs it). For inline reads, the `download://workspace/{ws}/{node}` / `download://share/{share}/{node}` resources return up to **100 KB** as base64; larger files fall back to a text response pointing at the `GET /file/...` pass-through (accepts `Mcp-Session-Id` OR `Authorization: Bearer` — a caller Bearer overrides a stale session token). **Password-protected fileshares** can't use the inline `download://fileshare/{id}` resource (no header channel) — use `fileshare action=download-url` or `GET /file/fileshare/{id}` with the `Authorization`/`x-ve-password` headers.

### Response hints & envelope

All tool responses are **GitHub-flavored Markdown** (CommonMark + tables), no JSON envelopes. Read the hint fields:

- **`_next`** — an array of exact next-action suggestions (tool + action + IDs). Follow them instead of guessing.
- **`_warnings`** — irreversible/destructive/problematic consequences. Read before proceeding (purge, bulk copy/move/delete/restore partial failures, archive/delete/close, billing-create, share/type changes, chat-delete, token expiry, transfer-token-create, etc.).
- **`_state`** — the entity's state machine on state-bearing responses: `current` (tagged terminal/not), `possible` (all states + meanings), `from_here` (legal actions), `note` (caveats). LIST responses carry only `possible`.
- **`_recovery`** — on errors (`isError:true`), status-based recovery. Notably **402** = credits exhausted → `org action=limits`; **401** = re-auth (`auth signin`/`set-api-key`; "Session expired" = prior session lapsed, "Not authenticated" = none); **403** = scope/permission (check `auth action=status` → `scopes_detail`); **429** = back off 2-4s. Errors carry numeric `code` + human `text`.
- **`detail=terse|standard|full`** — many list/node actions trade verbosity for token cost (terse = ids/labels/timestamp; standard adds operational context + `ai.state`; full adds long-form fields incl. the full `ai` object with `ai.attach`). Defaults vary per action — `describe` shows whether an action takes it. (Storage `details` defaults to `full`; lists default to `terse`.)

---

## 6. Code Mode contracts (`search` + `execute`)

When connecting from a headless agent, the server enables Code Mode (5 tools above). Two non-obvious contracts:

> **`search`/`execute` are not action-routed** — pass their params directly (`query` / `method`+`path`), not an `action`. They still answer `action="describe"` (a compact param reference) so the describe reflex works, but otherwise omit `action`.

### `search` — content vs API

**Finding content (USE THIS FIRST).** For "find my X" / "where is my Y", call `search query="..."` **without** setting `target` (defaults to `content`). Do NOT hand-roll `/storage/search/` through `execute` — the `search` tool wraps the right endpoints, fans out, and normalizes results.

- `workspace_id` given → single-workspace search with `limit`/`offset` pagination.
- `share_id` given → single-share search.
- **Neither → fans out across the top ~10 accessible workspaces only (4-way concurrent). Pagination is NOT available across the fanout** — for exhaustive/deep results, pass an explicit `workspace_id` or `share_id`.
- Keyword (names only) with intelligence OFF; semantic (RAG over indexed content, with `score`/`snippet`) when intelligence ON. `folders_scope`/`files_scope` narrow semantic search (silently ignored when intelligence off).

**Discovering API endpoints.** Flip `target="api"` to search the API spec for endpoints the code-mode tools don't already wrap. Returns method/path/params/descriptions — pass the chosen path to `execute`. Optional `tag` (domain filter) and `include_concepts`.

### `execute` — structured calls only (no eval)

Make authenticated REST calls — methods `get`, `delete`, `post`/`put`/`patch` (form-encoded body), and `postJson`/`putJson`/`patchJson` (JSON body). The auth token is injected automatically; fill path params yourself (replace `{workspace_id}`). **Read-only/structured restriction:** the sandbox is structured `method`/`path`/`body`/`params` only — there is **no `eval`/`new Function`/dynamic code execution** (blocked on Cloudflare Workers). Use `search target="api"` to discover the endpoint first.

```
execute method="get" path="/org/1234567890123456789/list/workspaces/"
execute method="postJson" path="/workspace/1234567890123456789/storage/root/createnote/" body={"name":"summary.md","content":"# Summary"}
```

**Response types** are handled automatically: JSON (parsed envelope), Text (`{content, content_type, http_status}`), Binary (returns metadata + guidance to use the `download://` resource). **Reading notes:** call the node's `…/storage/{node_id}/readnote/` path (JSON), not `…/read/` (raw binary) — there is no bare `/readnote/` alias. **Reading uploaded files (non-notes):** `resources/read uri="download://workspace/{ws}/{node}"`.

> **`execute` cannot stream bytes.** Byte-download routes are blocked (the Worker would buffer the whole payload): storage `…/read/`, fileshare `…/read/`. For storage-node bytes use the `download://` resource or `GET /file/...`; surface that to the user rather than retrying.

---

## 7. Product Guardrails (know these BEFORE asking how-to)

Cross-cutting product traps an agent hits silently. These belong here, not deferred.

- **Intelligence defaults OFF — keep it off unless RAG-across-many-docs is needed.** Enabling ingests **every** uploaded document at **10 credits/page** (non-refundable, incurred immediately). Only enable for (1) RAG queries across many documents or (2) semantic search via `storage action=search`. For one-off analysis of a few files, use **chat file-attachments** (`files_attach`, no ingestion cost) instead. Do not enable speculatively — it can always be enabled later. **In code mode, pass `intelligence=false` on workspace create** (the raw API defaults it ON; the named-mode tool defaults it OFF). Intelligence is reversible (can be turned back off).
- **Org discovery needs BOTH actions.** Call `org action=list` (internal orgs, `member:true`) **AND** `org action=discover-external` (external orgs, `member:false`). **Workspace-only invites appear ONLY in discover-external** — an agent that checks only `list` will miss the workspaces a human invited it to.
- **`email-check` is deprecated and non-authoritative.** The platform no longer discloses whether an email exists, so it now **always returns `available:true`** for a well-formed email (an `available:false` now just means the platform didn't return success — typically a malformed email — and **no longer means "in use"**). **Never gate signup on it** — call `signup` directly. Signup is authoritative and anti-enumeration-safe: an **existing** email is NOT an error — it returns the same neutral success as a new signup (no duplicate is created; the existing account is emailed a sign-in/reset link), so present a neutral "check your email" outcome, never "already in use". Signup does NOT auto-sign-in and returns a **uniform** response for new vs existing emails (`signup_request_accepted:true`, `signed_in:false`) — it is never an existence signal, so do not infer existence from it. After signup, sign in with `signin` (then `email-verify`).
- **Ownership transfer / 402-handoff works only from an agent account.** The transfer/claim API (`org` actions `transfer-token-create`/`-list`/`-delete`, `transfer-claim`) is available only when the agent created an **agent account** (`auth action=signup`) that owns the org. A **human API-key session cannot transfer** — on 402, tell the human to upgrade billing directly (`org action=billing-create` / dashboard). For agent accounts on 402: `transfer-token-create` → send `https://go.fast.io/claim?token=<token>` → human claims and upgrades; agent keeps admin access.

---

## 8. Billing / 402 awareness

New orgs require a **paid plan** — after `org action=create` the org is upgrade-only and returns **402** on resource-consuming calls until a plan is selected via `org action=billing-create` (`org action=billing-plans` lists offered plan IDs/limits). A **402** mid-work means credits are exhausted → upgrade, or (agent account) hand off to a human via ownership transfer. Check usage with `org action=limits`. Full billing flow, credit costs, and plan details: ask `how-to` or `org action=describe`.

---

## 9. Must-Know Gotchas

- **Action names** use hyphens (`create-session`); underscores are equivalent. `describe` shows the canonical form.
- **Profile params + aliases.** Workspace/share-scoped tools take `profile_type`+`profile_id` (type alias: `context_type`; id alias: `context_id`). The storage-family tools — `find`, `download`, `storage`, `ai`, `upload` — also accept `instance_id` (the REST/how-to name) as a `profile_id` alias; `find` additionally accepts the type-specific `workspace_id`/`share_id`. Other profile-scoped tools (`comment`, `event`) take only `profile_id`/`context_id`. Use one id, don't mix.
- **ID format.** Profile IDs (org/workspace/share/user) are 19-digit numeric strings (or a custom name). All other IDs (node, upload, chat, comment, etc.) are opaque alphanumerics of **29 or 30 characters** (30-char IDs carry a 2-char type prefix). **Never infer an ID's entity type from length or prefix, don't reject an ID on length, don't slice by a fixed offset, and don't apply numeric validation to opaque IDs.**
- **Confirm before destructive actions.** Irreversible operations (`purge`, `delete`, `close`, etc.) are confirm-gated (`confirm='true'`) and/or tagged `[DESTRUCTIVE]`. Always confirm with the user first.
- **Read-only scoped keys silently narrow the toolset.** With a read-only scoped key (all grants `:r`), write actions are removed from each tool's action enum; attempting a write returns a **schema validation error** (not a 403). Re-authenticate with a read-write/unscoped key to restore them. In **code mode**, a read-only session is likewise blocked from `execute` POST/PUT/PATCH/DELETE (not just the named-tool enums); out-of-scope entity access returns **403** (check `auth action=status` → `scopes_detail`).
- **Session state persists.** The token is stored server-side and survives across calls in the same connection; OAuth sessions auto-refresh (up to the 30-day refresh-token lifetime). No need to pass tokens between calls.
- **Use `web_url` from responses.** Entity-returning responses include a ready-to-use `web_url` — **always use it; never construct human-facing URLs by hand.** Include it whenever you create or reference something a human will open.
- **Text content uses `\n`.** Notes, comments, and text uploads use Unix line feeds (`\n`); only `\t`/`\n`/`\r` survive sanitization (other control chars are stripped). Content is GitHub-flavored Markdown.

# Fastio MCP Skill Package

Workspaces for agentic teams — where agents collaborate with other agents and with humans. Upload outputs, create branded shares, query documents with built-in AI, coordinate with peer agents, and hand everything off to a human when the job is done.

## What is Fastio?

Fastio gives AI agents a complete file management and collaboration platform through the Model Context Protocol (MCP). No infrastructure to manage, no SDKs to wire up — connect, authenticate, and call tools.

| Problem | Solution |
|---------|----------|
| Nowhere professional to put agent outputs | Branded workspaces with inline preview for 10+ formats |
| Sharing files with humans is awkward | Purpose-built shares (Send / Receive / Exchange) with passwords, expiration, and branding |
| Collecting files from humans is harder | Receive shares let humans upload directly into your workspace |
| Understanding document contents | Built-in AI (Ripley) reads, summarizes, and answers questions about your files with citations |
| Building a RAG pipeline from scratch | Enable workspace intelligence and files are automatically indexed and queryable |
| Two agents colliding on the same work | Agent Intents — announce what you're working on so peers see the conflict before it happens |
| Handing a project off to a human | One-step ownership transfer — the human gets the org, the agent keeps admin access |

## Connecting

| Transport | Endpoint |
|-----------|----------|
| Streamable HTTP (recommended) | `https://mcp.fast.io/mcp` |
| Legacy SSE | `https://mcp.fast.io/sse` |

## Two Modes

The server exposes one of two tool sets, chosen automatically from the MCP client's `clientInfo.name`:

- **Named mode (19 tools)** — action-routed tools covering the full REST surface. Served to named clients (e.g. Claude Desktop, Cline) and as the safe default for unknown clients.
- **Code mode (5 tools)** — a lightweight set (`auth`, `upload`, `search`, `execute`, `how-to`) for headless coding agents (e.g. Claude Code, Cursor, Codex). `search` discovers content or API endpoints; `execute` makes structured authenticated REST calls.

## Tools (Named mode)

Each tool covers a domain and uses an `action` parameter to select the operation. **Every tool supports `action: "describe"`** (no auth required) for the authoritative per-action parameter reference — call it the first time you use an unfamiliar tool instead of guessing.

| Tool | Domain |
|------|--------|
| `auth` | Sign-in/sign-up, 2FA, API key management, OAuth/PKCE sessions |
| `user` | Current user profile, contacts, invitations, user assets, account eligibility |
| `org` | Organization CRUD, members, billing/subscriptions, invitations, assets, org discovery, ownership transfer |
| `workspace` | Workspace lifecycle & settings, archive, members, notes, share import, async-job status |
| `share` | Share CRUD (Send / Receive / Exchange), archiving, passwords, members, AI titling |
| `fileshare` | Durable single-file share links, the replacement for QuickShare (access tiers, password, expiry, per-user grants, version history) |
| `storage` | Files & folders: list/search/move/copy/rename/delete/restore, versions, locking, previews, per-node metadata |
| `metadata` | Workspace metadata vocabulary, lexical value search, combined metadata + content matching, extraction eligibility |
| `find` | Unified search across a workspace or share — one query, results grouped into files, metadata, and comments |
| `upload` | File uploads: single-call streaming, chunked, bulk batch, web import from URLs |
| `download` | Pre-authenticated download / ZIP URLs (MCP cannot stream binary directly) |
| `ai` (Ripley) | Ask a natural-language question about workspace/share content and get a cited answer; manages chat threads |
| `comment` | Comments on files, with optional anchoring to regions/timestamps/pages |
| `event` | Audit & activity log, AI activity summaries, activity polling, per-member dashboard feed |
| `member` | Member management for workspaces and shares (roles, ownership transfer, join/leave) |
| `invitation` | Invitation management for workspaces and shares |
| `asset` | Branding asset upload/list/read/delete for orgs, workspaces, shares, users |
| `intent` | Agent Intents — announce what you're working on in a workspace so peers see a collision before it happens |
| `how-to` | Built-in product help — ask "how do I…" questions about Fastio (free, explain-only) |

## The `how-to` tool

The guide and tool descriptions stay lean by deferring product how-tos to a built-in help tool. Call **`how-to action=ask question="..."`** whenever the right *approach* on Fastio isn't obvious (branded shares, metadata extraction, coordinating with another agent, ownership transfer, billing). It returns the canonical, product-aware sequence of steps.

- **Free** — no credits, no plan gate; requires only an authenticated user.
- **Explain-only** — it returns guidance; it never creates, updates, or deletes anything.
- Available in **both modes** (answers are phrased as named-tool calls or `execute` calls to match).

## Resources

Read via `resources/list` / `resources/read`:

| URI | Description |
|-----|-------------|
| `skill://guide` | This package's full agent guide (`SKILL.md`) |
| `session://status` | Current authentication state |
| `resource://status` | Server identity, version, and transport endpoints (no auth) |
| `download://workspace/{workspace_id}/{node_id}` | Inline file content from a workspace |
| `download://share/{share_id}/{node_id}` | Inline file content from a share |
| `download://fileshare/{fileshare_id}` | Inline file content from a File Share link |

The `download://` resources return content inline up to **100 KB** — UTF-8 text for textual types, base64 otherwise. Above that they return a fallback download URL instead; for anything larger use the `download` tool's `file-url` action.

No MCP prompts are registered.

## Authentication

Four ways to authenticate:

1. **Agent account** — `auth action=signup` creates an agent account. Signup does not auto-sign-in; follow it with `auth action=signin`, then verify the email. Creating an organization requires selecting a plan (`org action=billing-create`).
2. **API key** — `auth action=set-api-key` with a key from an existing human account; you operate as that human.
3. **Sign in** — `auth action=signin` with email and password.
4. **PKCE browser login** — `auth action=pkce-login` → user approves in browser → `auth action=pkce-complete`. Secure OAuth 2.0 flow without sharing a password. Not for headless agents.

## Plans

New organizations — created by humans or agents alike — choose a paid plan to get started. There is no free-to-start path for new orgs; until a plan is selected the org is in an upgrade-only state and resource-consuming endpoints return HTTP 402.

| | Starter | Business | Growth |
|---|---|---|---|
| Monthly credits | 300,000 | 1,200,000 | 4,500,000 |
| Storage | 1 TB | 10 TB | 50 TB |
| Included seats | 1 | 20 | 50 |
| Max file size | 25 GB | 50 GB | 100 GB |

Credits cover storage, bandwidth, AI chat tokens, document/media ingestion, and file conversions. Query the live limits for the current plan with `upload action=limits`. See https://fast.io for current pricing.

> Agentic AI chat (Ripley) and workspace intelligence indexing are included on every paid plan. Agent accounts can build an org and later transfer it to a human, who then owns the billing; the agent keeps admin access.

## Core Capabilities

- **File storage** with in-place overwrite versioning, folder hierarchy, and full-text/semantic search
- **Branded shares** (Send / Receive / Exchange) with passwords, expiration, custom branding, and inline preview
- **Built-in AI/RAG (Ripley)** — ask questions about files with citations, scoped to folders or an entire workspace
- **Metadata extraction** — pull structured data out of unstructured files, then search across the workspace field vocabulary
- **Agent Intents** — declare what you're working on in a workspace so parallel agents don't collide
- **File preview** — images, video (HLS), audio, PDF, spreadsheets, and code rendered inline
- **URL import** — pull a file straight into a workspace from any URL
- **Comments & annotations** — anchored to image regions, video/audio timestamps, and PDF pages
- **Ownership transfer** — build an org, then hand it to a human; the agent keeps admin access

## Common Workflows

**Deliver a report:** Upload the file, create a Send share with a password and expiration, share the branded link.

**Collect documents:** Create a Receive share, send the link, files appear in your workspace.

**Build a knowledge base:** Create a workspace, turn on intelligence (it is off by default and ingestion consumes credits), upload documents, then query across all content with `ai` (Ripley).

**Set up a project for a human:** Create the org, workspaces, and shares, upload content, configure branding, then transfer ownership.

## Package Contents

| File | Description |
|------|-------------|
| `SKILL.md` | Complete agent guide — tool menu, MCP-server mechanics, authentication, and guardrails |
| `references/REFERENCE.md` | Platform deep-dive — capabilities, plan details, concepts, URL construction |

## Links

- **Platform guide:** [references/REFERENCE.md](references/REFERENCE.md)
- **API reference:** https://api.fast.io/llms.txt
- **Website:** https://fast.io

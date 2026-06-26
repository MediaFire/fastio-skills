# Fastio MCP Skill Package

Workspaces for agentic teams — where agents collaborate with other agents and with humans. Upload outputs, create branded shares, query documents with built-in AI, run durable multi-step workflows, and hand everything off to a human when the job is done.

## What is Fastio?

Fastio gives AI agents a complete file management and collaboration platform through the Model Context Protocol (MCP). No infrastructure to manage, no SDKs to wire up — connect, authenticate, and call tools.

| Problem | Solution |
|---------|----------|
| Nowhere professional to put agent outputs | Branded workspaces with inline preview for 10+ formats |
| Sharing files with humans is awkward | Purpose-built shares (Send / Receive / Exchange) with passwords, expiration, and branding |
| Collecting files from humans is harder | Receive shares let humans upload directly into your workspace |
| Understanding document contents | Built-in AI (Ripley) reads, summarizes, and answers questions about your files with citations |
| Building a RAG pipeline from scratch | Enable workspace intelligence and files are automatically indexed and queryable |
| Coordinating multi-step work | Durable workflow orchestration with content review and e-signature |
| Handing a project off to a human | One-step ownership transfer — the human gets the org, the agent keeps admin access |

## Connecting

| Transport | Production | Development |
|-----------|-----------|-------------|
| Streamable HTTP (recommended) | `https://mcp.fast.io/mcp` | `https://mcp.fastdev1.com/mcp` |
| Legacy SSE | `https://mcp.fast.io/sse` | `https://mcp.fastdev1.com/sse` |

## Two Modes

The server exposes one of two tool sets, chosen automatically from the MCP client's `clientInfo.name`:

- **Named mode (20 tools)** — action-routed tools covering the full REST surface. Served to named clients (e.g. Claude Desktop, Cline) and as the safe default for unknown clients.
- **Code mode (5 tools)** — a lightweight set (`auth`, `upload`, `search`, `execute`, `how-to`) for headless coding agents (e.g. Claude Code, Cursor, Codex). `search` discovers content or API endpoints; `execute` makes structured authenticated REST calls.

## Tools (Named mode)

Each tool covers a domain and uses an `action` parameter to select the operation. **Every tool supports `action: "describe"`** (no auth required) for the authoritative per-action parameter reference — call it the first time you use an unfamiliar tool instead of guessing.

| Tool | Domain |
|------|--------|
| `auth` | Sign-in/sign-up, 2FA, API key management, OAuth/PKCE sessions |
| `user` | Current user profile, contacts, invitations, user assets, account eligibility |
| `org` | Organization CRUD, members, billing/subscriptions, ownership transfer, custom domains |
| `workspace` | Workspace lifecycle & settings, notes, async-job status, metadata templates/views |
| `share` | Share CRUD (Send / Receive / Exchange), passwords, members, AI titling, share-native review |
| `fileshare` | Durable single-file share links (access tiers, password, expiry, version history) |
| `storage` | Files & folders: list/search/move/copy/rename/delete/restore, versions, locking, previews |
| `metadata` | AI metadata templates, unstructured-data extraction, saved views, lexical search |
| `find` | Unified search across a workspace or share (files, metadata, comments, workflows) |
| `upload` | File uploads: chunked, streaming, bulk batch, web import from URLs |
| `download` | Pre-authenticated download / ZIP URLs (MCP cannot stream binary directly) |
| `ai` (Ripley) | Read-only RAG: ask a natural-language question about workspace/share content, get a cited answer |
| `comment` | Comments on files, with optional anchoring to regions/timestamps/pages |
| `event` | Audit & activity log, AI activity summaries, activity polling, per-member dashboard feed |
| `member` | Member management for workspaces and shares (roles, ownership transfer, join/leave) |
| `invitation` | Invitation management for workspaces and shares |
| `asset` | Branding asset upload/list/read/delete for orgs, workspaces, shares, users |
| `task` | Lightweight in-workspace task lists & tasks (statuses, priorities, assignees) |
| `workflow` | Durable multi-step workflow orchestration, plus content review and e-signature |
| `how-to` | Built-in product help — ask "how do I…" questions about Fastio (free, explain-only) |

## The `how-to` tool

The guide and tool descriptions stay lean by deferring product how-tos to a built-in help tool. Call **`how-to action=ask question="..."`** whenever the right *approach* on Fastio isn't obvious (setting up a review/approval, branded shares, metadata extraction, ownership transfer, billing). It returns the canonical, product-aware sequence of steps.

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
| `download://workspace/{workspace_id}/{node_id}` | Inline file content from a workspace (up to 50 MB base64) |
| `download://share/{share_id}/{node_id}` | Inline file content from a share |
| `download://fileshare/{fileshare_id}` | Inline file content from a fileshare link |

No MCP prompts are registered.

## Authentication

Four ways to authenticate:

1. **Agent account** — `auth action=signup` creates a free agent account (50 GB storage, 5,000 monthly credits). Signup does not auto-sign-in; follow it with `auth action=signin`, then verify the email.
2. **API key** — `auth action=set-api-key` with a key from an existing human account; you operate as that human.
3. **Sign in** — `auth action=signin` with email and password.
4. **PKCE browser login** — `auth action=pkce-login` → user approves in browser → `auth action=pkce-complete`. Secure OAuth 2.0 flow without sharing a password. Not for headless agents.

## Free Agent Plan

| Resource | Included |
|----------|----------|
| Price | $0 — no credit card, no trial, no expiration |
| Storage | 50 GB |
| Max file size | 1 GB (higher on paid plans — query `upload action=limits`) |
| Monthly credits | 5,000 |

> **AI features require a paid plan.** Agent-plan orgs can store, organize, preview, and share files. Agentic AI chat (Ripley) and workspace intelligence indexing require a paid plan (Pro or Business). Agent accounts can transfer their org to a human when it's time to upgrade.

## Core Capabilities

- **File storage** with in-place overwrite versioning, folder hierarchy, and full-text/semantic search
- **Branded shares** (Send / Receive / Exchange) with passwords, expiration, custom branding, and inline preview
- **Built-in AI/RAG (Ripley)** — ask questions about files with citations, scoped to folders or an entire workspace
- **Metadata extraction** — AI templates that pull structured data out of unstructured files into saved, searchable views
- **Workflow orchestration** — durable multi-step processes with content review and e-signature
- **File preview** — images, video (HLS), audio, PDF, spreadsheets, and code rendered inline
- **URL import** — pull files from any URL, including Google Drive, OneDrive, and Dropbox
- **Comments & annotations** — anchored to image regions, video/audio timestamps, and PDF pages
- **Ownership transfer** — build an org, then hand it to a human; the agent keeps admin access

## Common Workflows

**Deliver a report:** Upload the file, create a Send share with a password and expiration, share the branded link.

**Collect documents:** Create a Receive share, send the link, files appear in your workspace.

**Build a knowledge base:** Create a workspace with intelligence enabled, upload documents, query across all content with `ai` (Ripley).

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

# Fast.io MCP Skill Package

Cloud file management and collaboration for AI agents — upload outputs, create branded data rooms, query documents with built-in AI, and hand everything off to a human when the job is done.

## What is Fast.io?

Fast.io gives AI agents a complete file management and collaboration platform through the Model Context Protocol (MCP). No infrastructure to manage, no subscriptions to set up, no credit card required.

| Problem | Solution |
|---------|----------|
| Nowhere professional to put agent outputs | Branded workspaces with file preview for 10+ formats |
| Sharing files with humans is awkward | Purpose-built shares (Send, Receive, Exchange) with passwords, expiration, branding |
| Collecting files from humans is harder | Receive shares let humans upload directly to you |
| Understanding document contents | Built-in AI reads, summarizes, and answers questions about your files |
| Building a RAG pipeline from scratch | Enable intelligence and files are automatically indexed, summarized, and queryable |
| Handing a project off to a human | One-click ownership transfer — human gets the org, agent keeps admin access |

## Connecting

| Transport | Endpoint |
|-----------|----------|
| Streamable HTTP (recommended) | `https://mcp.fast.io/mcp` |
| Legacy SSE | `https://mcp.fast.io/sse` |

## Tools

The server exposes **14 consolidated tools** using action-based routing. Each tool covers a domain and uses an `action` parameter to select the operation.

| Tool | Domain | Key Actions |
|------|--------|-------------|
| `auth` | Authentication | `signin`, `signup`, `set-api-key`, `pkce-login`, `pkce-complete`, `status` |
| `org` | Organizations | `list`, `details`, `create`, `update`, `create-workspace` |
| `workspace` | Workspaces | `list`, `details`, `create`, `update`, `check-name` |
| `share` | Shares | `list`, `create`, `update`, `delete`, `quickshare-create` |
| `storage` | Files, folders, locks, previews | `list`, `details`, `search`, `create-folder`, `create-note`, `move`, `delete`, `lock-acquire`, `preview-url` |
| `upload` | File uploads | `create-session`, `chunk`, `finalize`, `text-file`, `web-import` |
| `download` | Downloads | `file-url`, `zip-create`, `quickshare-details` |
| `ai` | AI chat | `chat-create`, `message-send`, `message-read`, `chat-list` |
| `member` | Members | `add`, `update`, `remove`, `details` |
| `invitation` | Invitations | `list`, `send`, `revoke`, `accept-all` |
| `asset` | Branding assets | `types`, `list`, `upload`, `delete` |
| `comment` | Comments | `list`, `create`, `details`, `delete` |
| `event` | Events & audit | `search`, `details`, `summarize`, `activity-poll` |
| `user` | Account management | `me`, `update`, `invitation-list`, `allowed` |

## Resources

| URI | Description |
|-----|-------------|
| `skill://guide` | Full tool documentation, workflows, and constraints |
| `session://status` | Current authentication state |

## Prompts

Guided workflows for common multi-step operations:

| Prompt | Purpose |
|--------|---------|
| `get-started` | Account creation, org setup, onboarding, and auth methods (PKCE, API key) |
| `add-file` | Adding files from text content, binary upload, or URL import |
| `ask-ai` | Querying files with AI chat (RAG scoping, attachments, polling) |
| `comment-conversation` | Agent-human feedback on files via anchored comments, threads, and deep-link URLs |
| `catch-up` | AI activity summaries, event search, and real-time change monitoring |

## Authentication

Four ways to authenticate:

1. **Agent account** — `auth` action `signup` creates a free agent account (50 GB storage, 5,000 monthly credits)
2. **API key** — `auth` action `set-api-key` with a key from an existing human account
3. **Sign in** — `auth` action `signin` with email and password
4. **PKCE browser login** — `auth` action `pkce-login` for secure OAuth 2.0 flow without sharing passwords

## Agent Plan (Free)

| Resource | Included |
|----------|----------|
| Price | $0 — no credit card, no trial, no expiration |
| Storage | 50 GB |
| Max file size | 1 GB |
| Monthly credits | 5,000 |
| Workspaces | 5 |
| Shares | 50 |
| Members per workspace | 5 |

## Core Capabilities

- **File storage** with versioning, folder hierarchy, full-text and semantic search
- **Branded shares** (Send/Receive/Exchange) with passwords, expiration, custom branding, file preview
- **Built-in AI/RAG** — ask questions about files with citations, scoped to folders or entire workspaces
- **File preview** — images, video (HLS), audio, PDF, spreadsheets, code rendered inline
- **URL import** — import files from any URL including Google Drive, OneDrive, Dropbox
- **Comments & annotations** — anchored to image regions, video/audio timestamps, PDF pages
- **Ownership transfer** — build an org, then transfer to a human; agent keeps admin access
- **Real-time collaboration** — WebSocket-based live presence and instant sync

## Common Workflows

**Deliver a report:** Upload file, create a Send share with password and expiration, share the branded link.

**Collect documents:** Create a Receive share, send the link, files appear in your workspace and get AI-indexed.

**Build a knowledge base:** Create an intelligent workspace, upload documents, query across all content with AI chat.

**Set up a project for a human:** Create org, workspaces, shares, upload content, configure branding, transfer ownership.

## Package Contents

| File | Description |
|------|-------------|
| `SKILL.md` | Complete agent guide with tool documentation, workflows, and constraints |
| `references/REFERENCE.md` | Platform deep-dive — capabilities, plan details, URL construction |

## Links

- **Platform guide:** https://fast.io/agents.md
- **API reference:** https://api.fast.io/llms.txt
- **Website:** https://fast.io

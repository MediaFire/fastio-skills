# Fastio for AI Agents

> **Version:** 1.38.0 | **Last updated:** 2026-08-29
>
> This guide is available at the `/current/agents/` endpoint on the connected API server.

**Put your files to work — workspaces for agentic teams.** Fastio answers across your files, automates the busywork, and keeps work secure and on the record, for your people and your agents, all through one API. Storage is step zero.

Fastio provides workspaces for agentic teams — where agents collaborate with other agents and with humans. Upload
outputs, create branded portals, ask questions about documents using built-in AI, and collaborate with humans on a
shared platform. No infrastructure to manage — Fastio accounts, for humans and AI agents alike, require an email
address: sign up, create an organization, and choose a paid plan to get started.

The platform is organized around two things you do with your files: **Intelligence** — ask across them and get cited answers, turn documents and images into structured data with AI metadata extraction, and research, analyze, and draft with the built-in agent, **Ripley**; and **Secure collaboration** — files belong to the project, not the person, with granular permissions, scoped agent tokens, branded portals, and an append-only audit log. Storage is step zero; Fastio provides the rest.

There are three ways to integrate with Fastio:

| Integration | Best For | Get Started |
|-------------|----------|-------------|
| **CLI** | Terminal workflows, scripting, CI/CD pipelines, human operators | `npm install -g @vividengine/fastio-cli` |
| **MCP Server** | AI agents (Claude Desktop, Claude Code, Cursor, etc.) | Connect to `https://mcp.fast.io/mcp` |
| **REST API** | Custom applications, languages without MCP support | See API endpoints throughout this guide |

**CLI** — The `fastio` command-line tool provides full platform access from the terminal. Install with
`npm install -g @vividengine/fastio-cli` and authenticate with `fastio auth login`. The CLI also includes a built-in
MCP server mode (`fastio mcp`) for local AI agent integration. See the "CLI Tool" section below for full details.

**MCP-enabled agents** should connect via the Model Context Protocol for the simplest integration — no raw HTTP calls
needed.

**MCP connection endpoints:**
- **Streamable HTTP (recommended):** `https://mcp.fast.io/mcp`
- **Legacy SSE:** `https://mcp.fast.io/sse`

The MCP server exposes consolidated tools using action-based routing — each tool covers a domain (e.g., `auth`,
`storage`, `upload`) and uses an `action` parameter to select the operation. In Named Mode (Claude Desktop, etc.),
there are multiple domain-specific tools plus app-specific widget tools. In Code Mode (Claude Code,
Cursor, etc.), there is a smaller set of streamlined tools. See the "MCP Tool Architecture" section
below for the full tool list.

MCP-connected agents receive comprehensive workflow guidance through SERVER_INSTRUCTIONS at connection time, and can
read resources (`resources/read`) including `skill://guide` for full tool documentation, `session://status` for current
authentication state, and `download://` resource templates for direct file content retrieval.

This guide covers platform concepts and capabilities; the MCP server provides tool-level details through its standard
protocol interface. The API endpoints referenced below are what the MCP server calls under the hood, and are available
for agents that need direct HTTP access or capabilities not yet covered by the MCP tools.

---

## Why Agents Use Fastio

### The Problem

Agentic teams — groups of agents working together and with humans — need a shared place to work. Today, agents cobble
together S3 buckets, presigned URLs, email attachments, and custom download pages. Every agent reinvents collaboration,
and there's no shared workspace where agents and humans can see the same files, track activity, and hand off work.

Meanwhile, when agents need to *understand* documents — not just store them — they have to download files, parse dozens
of formats, build search indexes, and manage their own RAG pipeline. That's a lot of infrastructure for what should be a
simple question: "What does this document say?"

### What Fastio Solves

| Problem                                      | Fastio Solution                                                                                  |
|----------------------------------------------|---------------------------------------------------------------------------------------------------|
| No shared workspace for agentic teams        | Workspaces where agents and humans collaborate with file preview, versioning, and AI              |
| Agent-to-agent coordination lacks structure  | Shared workspaces with activity feeds, comments, and real-time sync across team members           |
| Sharing outputs with humans is awkward       | Purpose-built shares (Send, Receive, Exchange) with link sharing, passwords, expiration           |
| Collecting files from humans is harder       | Receive shares let humans upload directly to your workspace — no email attachments                |
| Understanding document contents              | Built-in AI reads, summarizes, and answers questions about your documents and code (agentic chat and intelligence indexing require a paid plan — Starter, Business, or Growth) |
| Building a RAG pipeline from scratch         | Enable intelligence on a workspace and documents are automatically indexed, summarized, and queryable (requires a paid plan) |
| Finding the right file in a large collection | Semantic search finds documents by meaning, not just filename                                     |
| Getting documents signed                     | Native e-signature: assemble an envelope, send with OTP identity checks, and the executed PDF + audit certificate file back into the workspace |
| Turning unstructured files into data         | AI metadata extraction pulls typed fields from documents, images, and spreadsheets into a sortable table |
| Files walking out when someone leaves        | Files belong to the org/workspace (the project), not the uploader — access follows the work, not the person |
| Knowing what needs attention                 | A per-workspace dashboard ranks @mentions and file activity |
| Collaborating with humans on a shared org    | Invite humans (or be invited) as org/workspace members — everyone sees the same files and activity |
| Tracking what happened                       | Full audit trail with AI-powered activity summaries                                               |
| Plans                                        | New organizations choose a paid plan (Starter, Business, or Growth); credits cover storage, bandwidth, and AI usage |

---

## Getting Started — Choosing the Right Access Pattern

Fastio accounts — for humans and AI agents alike — require an email address. There is one kind of account; an agent
account is an ordinary account tagged `account_type=agent` for identification. The getting-started flow is the same for
everyone: sign up, create an organization, choose a paid plan, then build. There are a few access patterns depending on
whether you're operating autonomously or working inside a human's existing organization.

### Option 1: Autonomous Agent — Create Your Own Account

If you're operating as part of an agentic team (collaborating with other agents, sharing outputs with humans,
coordinating shared work), create your own account and your own organization:

1. `POST /current/user/` with `email_address`, `password`, `tos_agree=true`. Optionally pass `agent=true` to tag the
   account as an agent account (`account_type=agent`) — this is for identification only and does not change which plans
   you can choose.
2. Authenticate with Basic Auth → get JWT
3. Verify your email address (required before using most endpoints):
   - `POST /current/user/email/validate/` with `email` — sends a verification code to your email
   - `POST /current/user/email/validate/` with `email` and `email_token` — validates the code and marks your account as verified
4. `POST /current/org/create/` with `domain` (required, 2-63 chars lowercase alphanumeric + hyphens) — an org is a collector of workspaces that can represent a company, team, business unit, or personal collection
5. **Select a paid plan to activate the organization.** A newly created organization must select a paid plan
   (Starter, Business, or Growth) before it can be used; until then it is in an upgrade-only state — the same state as an
   org that has exhausted its credits, returning HTTP 402 on resource-consuming endpoints. Choose a plan via the
   billing API or direct the owner to `https://go.fast.io/onboarding`.
6. `POST /current/org/{org_id}/create/workspace/` with `folder_name`, `name`, `perm_join`, `perm_member_manage` (all required — see Permission Values below)

> **A new organization needs a paid plan before it can do work.** New organizations select one of the paid plans
> (Starter, Business, or Growth). Until a paid plan is selected, the org is in an upgrade-only state and resource-consuming
> endpoints (uploads, AI chat, ingestion) return HTTP 402, exactly like an org that has run out of credits. All paid
> plans include the `content_ai` and `ai_agent` features needed for agentic chat and RAG across indexed files.
> The [AI reference](https://api.fast.io/current/llms/ai/#plan-requirements) has the full plan matrix.

#### Permission Values

Workspace and share creation require human-readable permission strings:

**Workspace `perm_join`** (who can auto-join from the org):
- `'Member or above'` (default) — any org member can join
- `'Admin or above'` — only org admins and owners
- `'Only Org Owners'` — only org owners

**Workspace `perm_member_manage`** (who can manage workspace members):
- `'Member or above'` (default) — any workspace member can manage
- `'Admin or above'` — only workspace admins and owners

**Share `access_options`** (who can access the share):
- `'Only members of the Share or Workspace'` (default)
- `'Members of the Share, Workspace or Org'`
- `'Anyone with a registered account'`
- `'Anyone with the link'` (allows password protection)

### Option 2: Assisting a Human — Use Their API Key

If a human already has a Fastio account and wants your help managing their files, workspaces, or shares, they can
create an API key for you to use. No separate agent account is needed — you operate as the human user.

**How the human creates an API key:**

Go to **Settings → Devices & Agents → API Keys** and click **Create API Key**. Optionally enter a memo to label the
key (e.g., "CI pipeline" or "Agent access"), then click **Create**. Copy the key immediately — it is only displayed
once and cannot be retrieved later. Direct link: `https://go.fast.io/settings/api-keys`

Use the API key as a Bearer token: `Authorization: Bearer {api_key}`

The API key has the same permissions as the human user, so you can manage their workspaces, shares, and files directly.

### Option 3: Agent Account Invited to a Human's Org

If you want your own agent identity but need to work within a human's existing organization (their company, team, or personal collection), you can create an agent account and have the human invite you as a member. This gives you access to their workspaces and shares while keeping your own account separate.

**How the human invites the agent to their org:**

Go to **Settings → Your Organization → [Org Name] → Manage People** and click **Invite People**. Enter the agent's
email address, choose a permission level (Member or Admin), and click **Send Invites**. The agent account will receive
the invitation and can accept it via `POST /current/user/invitations/acceptall/`.

**How the human invites the agent to a workspace:**

Open the workspace, click the member avatars in the toolbar, then click **Manage Members**. Enter the agent's email
address, choose a permission level, and optionally check **Invite to org** to add them to the organization at the same
time. Click **Send Invites** — if the agent isn't already an org member and the toggle is off, they'll need an org
invite separately.

Alternatively, the human can invite the agent programmatically:
- **Org:** `POST /current/org/{org_id}/members/{agent_email}/` with `permission` level
- **Workspace:** `POST /current/workspace/{workspace_id}/members/{agent_email}/` with `permission` level

### Option 4: PKCE Browser Login — Secure Authentication Without Sharing Passwords

For the most secure authentication flow — especially when a human wants to authorize an agent without sharing their
password — use the PKCE (Proof Key for Code Exchange) browser login. No credentials pass through the agent at any point.

The `client_id` can be a pre-registered ID, a dynamically registered ID (via DCR), or an **HTTPS URL pointing to a
Client ID Metadata Document (CIMD)**. CIMD is the MCP specification's preferred registration method — the server
fetches client metadata from the URL on-the-fly, so no pre-registration is needed.

1. Agent calls `POST /current/oauth/authorize/` with PKCE parameters (`code_challenge`, `code_challenge_method=S256`,
   `client_id`, `redirect_uri`, `response_type=code`) — gets back an authorization URL
2. The user opens the URL in their browser, signs in (supports SSO), and approves access
3. The browser displays an authorization code that the user copies back to the agent
4. Agent calls `POST /current/oauth/token/` with `grant_type=authorization_code`, the authorization `code`, and the
   PKCE `code_verifier` — receives an access token and refresh token
5. The agent is now authenticated. Access tokens last **1 hour**; refresh tokens are **long-lived**. Use
   `POST /current/oauth/token/` with `grant_type=refresh_token` to get new access tokens without repeating the flow.

This is the recommended approach when:
- A human wants to grant agent access without sharing their password
- The organization uses SSO and password-based auth isn't available
- You need the strongest security guarantees (no credentials stored by the agent)

### Recommendations

| Scenario | Recommended Approach |
|----------|---------------------|
| Operating autonomously, storing files, building for users | Create your own account and org (your personal collection of workspaces) and select a paid plan |
| Helping a human manage their existing account | Ask the human to create an API key for you |
| Working within a human's org with your own identity | Create an account, have the human invite you to their org or workspace |
| Building inside a human's already-paid organization | Be invited as a member of the human's org or workspace and build there |
| Human wants to authorize an agent without sharing credentials | Use PKCE browser login (Option 4) |
| Terminal workflows, scripting, or CI/CD pipelines | Install the CLI: `npm install -g @vividengine/fastio-cli` |
| Local AI agent needing Fastio access | Use the CLI's built-in MCP server: `fastio mcp` |

### Authentication & Token Lifecycle

All API requests require `Authorization: Bearer {token}` in the header. How you get that token depends on your access
pattern:

**JWT tokens (agent accounts):** Authenticate with `GET /current/user/auth/` using HTTP Basic Auth (email:password). The
response includes an `auth_token` (JWT). OAuth access tokens last **1 hour** and refresh tokens are **long-lived**. When
your token expires, re-authenticate to get a new one. If the account has 2FA enabled, the initial token has limited
scope until 2FA verification is completed via `/current/user/auth/2factor/auth/{token}/`.

**API keys (human accounts):** API keys are long-lived and do not expire unless the human revokes them. No refresh flow
needed.

**Verify your token:** Call `GET /current/user/auth/check/` at any time to validate your current token and get the
authenticated user's ID. This is useful at startup to confirm your credentials are valid before beginning work, or to
detect an expired token without waiting for a 401 error on a real request.

### OAuth Scopes — Controlling Access

When using PKCE browser login, you can request scoped access tokens that limit what the agent can do. Scopes follow
an inheritance model — broader scopes automatically include access to their children.

**Scope types** (the values the `scope` parameter accepts — the currently-offered set, matching the server's `scopes_supported` metadata):

| Scope Type           | Description                                                                |
|----------------------|----------------------------------------------------------------------------|
| `user`               | Full access (default, backward compatible with the v1.0 JWT)               |
| `org`                | User picks specific organizations                                          |
| `workspace`          | User picks specific workspaces                                             |
| `all_orgs`           | Wildcard access to all organizations the user owns or is a member of       |
| `all_workspaces`     | Wildcard access to all workspaces across accessible organizations          |
| `all_shares`         | Wildcard access to all shares across accessible organizations and workspaces |
| `all_sign_envelopes` | Wildcard access to all sign envelopes the user can reach                   |

**Inheritance:** `all_orgs` includes `all_workspaces`, which includes `all_shares`. Requesting `all_orgs` grants full
access to all orgs, workspaces, and shares the user has access to. (`all_sign_envelopes` is a separate wildcard, not
part of that org→workspace→share chain.)

> **Retired scope — `all_workflows`:** no longer offered. The Workflows feature has been removed, so `all_workflows` is
> deliberately **absent** from the server's `scopes_supported` metadata and must not be requested. The authorization
> server keeps it only as a fail-closed tombstone, so a pre-existing token that still carries it resolves to nothing
> rather than erroring. (This is why the offered set is the seven above, not eight.)

Pass the desired `scope_type` when initiating the PKCE authorization flow (`POST /current/oauth/authorize/`). If
omitted, the token defaults to full access (equivalent to `all_orgs`).

**Scope format note:** The `scope` parameter in the authorization request accepts the named strings listed above
(e.g., `scope=org`). However, API responses return scopes in a different format -- as arrays of
`entity_type:entity_id:access_mode` strings (e.g., `["org:12345:rw", "org:67890:r"]`). The token endpoint
(`POST /current/oauth/token/`) returns `scopes` as a JSON-encoded string that must be parsed. Use
`GET /current/auth/scopes/` to introspect the current token's scopes in a structured format.

### Organizations — Collectors of Workspaces

An organization (org) is a collector of workspaces. It can represent a company, a business unit, a team, or simply your own personal collection. Every workspace and share lives under an org, and orgs are the billable entity — storage, credits, and member limits are tracked at the org level.

### Internal vs External Orgs

When working with Fastio, an agent may interact with orgs in two different ways:

**Internal orgs** — orgs you created or were invited to join as a member. You have org-level access: you can see all
workspaces (subject to permissions), manage settings if you're an admin, and appear in the org's member list. Your own
orgs always show `member: true` in API responses.

**External orgs** — orgs you can access only through workspace membership. If a human invites you to their workspace
but does not invite you to their org, the org appears as external. You can see the org's name and basic public info, but
you cannot manage org settings, see other workspaces, or add members at the org level. External orgs show
`member: false` in API responses.

This distinction matters because an agent invited to a single workspace cannot assume it has access to the rest of that
org. It can only work within the workspaces it was explicitly invited to.

**Full org discovery requires both endpoints:**

- `GET /current/orgs/list/` — returns orgs you are a member of (`member: true`)
- `GET /current/orgs/list/external/` — returns orgs you access via workspace membership only (`member: false`)

**Always call both.** An agent that only calls `/orgs/list/` will miss every org where it was invited to a workspace but
not to the org itself — which is the most common pattern when a human adds an agent to help with a specific project. If
you skip `/orgs/list/external/`, you won't discover those workspaces at all.

**Example:** A human invites your agent to their "Q4 Reports" workspace. You can upload files, run AI queries, and
collaborate in that workspace. But you cannot create new workspaces in their org, view their billing, or access their
other workspaces. The org shows up in `/orgs/list/external/` — not `/orgs/list/`.

If the human later invites you to the org itself (via org member invitation), the org moves from external to internal and
you gain org-level access based on your permission level.

### Pagination

All list endpoints support offset-based pagination via query parameters. Use pagination to keep responses within token
limits and iterate through large collections.

**Query parameters:**

| Parameter | Type | Default | Max | Description                |
|-----------|------|---------|-----|----------------------------|
| `limit`   | int  | 100     | 500 | Number of items to return  |
| `offset`  | int  | 0       | —   | Number of items to skip    |

**Response metadata:** Every paginated response includes a `pagination` object:

```json
{
  "pagination": {
    "total": 42,
    "limit": 100,
    "offset": 0,
    "has_more": false
  }
}
```

**Paginating through results:**

```
# First page
GET /current/orgs/list/?limit=10&offset=0
# → pagination.has_more = true, pagination.total = 42

# Second page
GET /current/orgs/list/?limit=10&offset=10
# → pagination.has_more = true

# Continue until has_more = false
```

**Endpoints supporting pagination:**

| Endpoint                                             | Collection Key     |
|------------------------------------------------------|--------------------|
| `GET /current/orgs/all/`                             | `orgs`             |
| `GET /current/orgs/list/`                            | `orgs`             |
| `GET /current/orgs/list/external/`                   | `orgs`             |
| `GET /current/shares/all/`                           | `shares`           |
| `GET /current/workspace/{id}/members/list/`          | `users`            |
| `GET /current/org/{id}/billing/usage/members/list/`  | `billable_members` |
| `GET /current/workspace/{id}/list/shares/`           | `shares`           |
| `GET /current/user/me/list/shares/`                  | `shares`           |
| `GET /current/org/{id}/list/workspaces/`             | `workspaces`       |
| `GET /current/org/{id}/members/list/`                | `users`            |
| `GET /current/workspace/{id}/storage/search/`        | `files`            |
| `GET /current/share/{id}/storage/search/`            | `files`            |
| `GET /current/share/{id}/members/list/`              | `users`            |

**Pending members in member lists:** Member list responses include a `status` field for each user: `"active"` for
members who have accepted their invitation, or `"pending"` for users who have been invited but have not yet created an
account. Pending members also include an `invite` object with `id`, `created`, and `expires` fields. To cancel a
pending invitation, use `DELETE` on the invitation endpoint with the `invite.id`.

### Compact Responses (`output=`)

Most list and detail endpoints accept an optional `output` query parameter with three detail levels for nodes
(files/folders/notes/links), events, users, workspaces, orgs, and shares. Picking the right level is the single
biggest lever you have for keeping agent payloads small without losing information you actually need.

- **`terse`** — identifiers, primary labels, and the handful of fields needed to navigate between resources.
  Use for tree traversal, pickers, autocomplete, mention suggestions, and any prefetch step that will follow up
  with a detail call only on user (or agent) interest.
- **`standard`** — `terse` plus the operational context most list/detail views render: timestamps, lifecycle
  flags, short descriptions, plan/status fields, creator/owner refs, member status, short summaries. **This is
  the recommended default for most agent list and detail workflows** — it covers the fields a typical agent
  needs to reason about a resource without pulling branding, capability matrices, or long-form AI summaries.
- **`full`** — the complete resource shape; equivalent to omitting `?output=` entirely. Use when you need
  branding, capability matrices, permission blocks, long-form AI summaries, metadata blocks, embedded file
  metadata (EXIF / media — returned only to callers permitted to download the file), virus state,
  or any field specifically called out under the `full` tier in the category references.

**Rules:**
- **Syntax:** `?output=<level>` or `?output=<level>,<modifier>` (comma-separated tokens).
- **Mutually exclusive levels:** Specifying more than one detail level in the same request (e.g.
  `?output=terse,standard`) is an error and returns **HTTP 406**. Pick exactly one.
- **Cumulative fields:** `standard` is a superset of `terse`, and `full` is a superset of `standard` — nothing
  disappears as you move up a tier.
- **Default:** When `output=` is absent, responses are `full` and byte-for-byte unchanged.
- **Unknown tokens:** Silently ignored for forward compatibility.
- **`markdown` modifier:** Add `markdown` to any request (e.g. `?output=terse,markdown` or `?output=markdown` alone) to receive the response as GitHub-flavored Markdown instead of JSON. Response `Content-Type` becomes `text/markdown; charset=UTF-8`. Homogeneous record lists render as GFM pipe tables, associative maps as bullet lists, and error envelopes as a leading `# Error` section — including a nested `params` table that lists every parameter that failed validation when the call was a 406. Validation errors (HTTP 406) render as markdown too when the modifier is present. If the caller is an LLM that reasons better over markdown than JSON, prefer `?output=standard,markdown`.

Example markdown response body for `GET /current/user/details/?output=terse,markdown`:

```markdown
**Result:** success

# user
- **id:** 1234567890123456789
- **account_type:** agent
- **first_name:** Alice
- **last_name:** Example
- **profile_pic:** https://…
```

Example markdown response body for a validation error (HTTP 406) — note the nested `params` table:

```markdown
**Result:** failure

# Error
- **code:** 10022
- **text:** email: This value should not be blank. domain: This value should not be blank.
- **resource:** POST /current/user/email/

## params

| name | kind | message | code | expected_type |
|------|------|---------|------|---------------|
| email | missing | This value should not be blank. | 10022 | — |
| domain | invalid | This value is not a valid hostname. | 10023 | string |
```

Category-specific detail pages (linked from the LLM reference) list exactly which fields appear at each level
for each resource type.

**Reading the error tables:** the four-digit `16xx`/`17xx` values are **HTTP-status classes, not `error.code`**. The `error.code` a client actually receives is assigned per endpoint, so **use the HTTP status as the gate and a documented `error.code` — five or six digits, plus the `9661`-`9669` family — only as a refinement**. A `16xx` value identifies the status class — useful for telling which kind of failure occurred — but comparing one against `error.code` will never match. Codes shown as five or six digits (and the `9661`-`9669` family) ARE `error.code` values. **If you widen a check from a specific code to a status, widen what you assert with it** — a status covers failures the narrower code did not, so a message written for that one code becomes a confident falsehood on the rest.

### Retired per-field error codes

As of 2026-05-05, error codes `136957`, `249170`, `279705`, `295625` are retired. The equivalent field-level failures now surface inside `error.params[]` with `kind: 'invalid'` (and a per-field `code` and `message` describing the specific violation). Clients that previously switched on those exact integers should switch on `params[].name` + `params[].kind` instead.

---

## Core Capabilities

### 1. Workspaces — Shared Spaces for Agentic Teams

Workspaces are where agentic teams do their work. Each workspace has its own storage, member list, AI chat, and
activity feed — a shared environment where agents collaborate with other agents and with humans.

- **Included storage scales with your plan** — query org billing/usage for your org's storage allowance
- **File size limits are plan-dependent** (up to 40 GB on paid plans) — query `/upload/limits/` for exact values
- **File versioning** — every edit creates a new version, old versions are recoverable
- **Folder hierarchy** — organize files however you want
- **Filename and semantic search** — find files by name (including `find`-style glob patterns) or by meaning. There is no full-text index of file bytes; content matching is the AI's understanding of a file. See *Filename Search* and *`search_in=content` Is Not `grep`*.
- **Member roles** — Owner, Admin, Editor, Viewer with granular permissions
- **Real-time sync** — changes appear instantly for all members via WebSockets

#### Intelligence: On or Off

Workspaces have an **intelligence** toggle that controls whether AI features are active. This is a critical decision:

**Intelligence OFF** — the workspace stores files without AI indexing. You can still attach files directly to an AI chat
conversation (up to 20 files), but files are not persistently indexed. This is fine for coordination workflows where
you don't need to query your content.

**Intelligence ON** — the workspace becomes an AI-powered knowledge base. Every document and code file uploaded is automatically ingested,
summarized, and indexed for RAG. This enables:

- **RAG (retrieval-augmented generation)** — scope AI chat to entire folders or the full workspace and ask questions
  across your indexed documents and code. The AI retrieves relevant passages and answers with citations.
- **Semantic search** — find files by meaning, not just keywords. "Show me contracts with indemnity clauses" works even
  if those exact words don't appear in the filename.
- **Auto-summarization** — short and long summaries generated for every indexed document and code file, searchable and visible in the UI.
- **Metadata extraction** — AI pulls structured, typed fields out of documents, code, and images. **No schema to set
  up**: every eligible file is extracted as it is ingested, and each one also gets a broad `category` and
  `sub_category`. You discover what a workspace holds, then filter on it; you can also trigger extraction manually or
  per folder. See section 13 (Metadata) for the full API.

> **Coming soon:** RAG indexing support for images, video, and audio files. Currently only documents and code are indexed.

> **Plan requirement.** Intelligence requires both the `content_ai` and `ai_agent` plan features (included on every
> paid plan: Starter, Business, or Growth). On a plan that does not include those features, a new workspace is created
> with `intelligence` off and it cannot be switched on — the update endpoint rejects the attempt with
> `1605 (Invalid Input)`. See the [AI reference](https://api.fast.io/current/llms/ai/#plan-requirements) for the full matrix.

**New workspaces default to intelligence ON** whenever the plan supports it. Omitting `intelligence` on the create call
means on; **send `intelligence=false` to opt out at create time.** You can also change it later with
`POST /current/workspace/{id}/update/`. On a plan lacking `content_ai` or `ai_agent` the workspace is created with it off
whatever you send, and the create still succeeds.

**Budget for this.** Ingestion costs 10 credits/page and is non-refundable — a 100-page document costs 1,000 credits.
Nothing is charged for creating the workspace itself; the cost starts on the first document or code file you upload. If
the workspace is for coordination or plain storage rather than RAG queries, **create it with `intelligence=false`** —
that is cheaper and simpler than creating it on and turning it off before you upload.

**Agent use case:** Create a workspace per project or client. Upload reports, datasets, and deliverables. Invite other
agents and human stakeholders. Everything is organized, searchable, and versioned — and the whole team can see it.
Indexing is on from the start, so RAG and semantic search work without a separate setup step; disable it up front if
this workspace is storage-only.

> **Cost-saving tips:** Turn intelligence off via `update` on storage-only workspaces, before uploading, to avoid
> ingestion costs. Attach-only AI chat (up to 20 files without indexing) requires a plan with `ai_agent` (included on
> every paid plan), so make sure the org is on a plan that includes it before relying on file Q&A.

### 2. Shares — Structured Agent-Human Exchange

Shares are purpose-built spaces for exchanging files between your agentic team and external humans. Three modes cover
every exchange pattern:

| Mode         | What It Does                  | Agent Use Case                                |
|--------------|-------------------------------|-----------------------------------------------|
| **Send**     | Recipients can download files | Deliver reports, exports, generated content   |
| **Receive**  | Recipients can upload files   | Collect documents, datasets, user submissions |
| **Exchange** | Both upload and download      | Collaborative workflows, review cycles        |

#### Share Features

- **Password protection** — require a password for link access
- **Expiration dates** — shares auto-expire after a set period
- **Download security** — three levels: `off` (no restrictions, default), `medium` (file previews are available but direct downloads are restricted for guests), or `high` (downloads completely disabled for guests). Set via `download_security` when creating or updating a share
- **Access levels** — `'Only members of the Share or Workspace'`, `'Members of the Share, Workspace or Org'`, `'Anyone with a registered account'`, or `'Anyone with the link'`
- **Custom branding** — background images, gradient colors, accent colors, logos
- **Post-download messaging** — show custom messages and links after download
- **Up to 3 custom links** per share for context or calls-to-action
- **Guest chat** — let share recipients ask questions in real-time
- **AI-powered auto-titling** — shares automatically generate smart titles from their contents
- **Activity notifications** — get notified when files are sent or received
- **Comment controls** — configure who can see and post comments (owners, guests, or both)

#### Two Storage Modes

When creating a share, you choose a `storage_mode` that determines how the share's files are managed:

- **`room`** (independent storage, default) — the share has its own isolated storage. Files are added directly to the
  share and are independent of any workspace. This creates a self-contained portal — changes to workspace files don't
  affect the portal, and vice versa. Perfect for final deliverables, compliance packages, archived reports, or any
  scenario where you want an immutable snapshot.

- **`workspace_folder`** (workspace-backed) — the share is backed by a specific folder in a workspace. The share displays
  the live contents of that folder — any files added, updated, or removed in the workspace folder are immediately
  reflected in the share. No file duplication, so no extra storage cost. To create a shared folder, pass
  `storage_mode=workspace_folder` and `folder_node_id={folder_opaque_id}` when creating the share. Note: expiration dates
  are not allowed on shared folder shares since the content is live. **Intelligence is not available** on shared folder
  shares — files are indexed through the parent workspace instead.

Both modes look the same to share recipients — a branded portal with file preview, download controls, and all share
features. The difference is whether the content is a snapshot (portal) or a live view (shared folder).

> **Note:** API responses include both `storage_mode` and a response-only `share_category` field.
> `independent` maps to `share_category: "portal"`; `workspace_folder` maps to `share_category: "shared_folder"`.

**Agent use case:** Generate a quarterly report, create a Send share with your client's branding, set a 30-day
expiration, and share the link. The client sees a branded page with instant file preview — not a raw download link.

### 3. File Share — Durable Single-File Share

Need to share one file with a stable link that doesn't expire on you? A **File Share** is a durable, link-shareable
view of a single workspace file. Create it once with `POST /current/workspace/{workspace_id}/create/fileshare/`,
pointing at the file's node id, and you get a permanent link — no expiration, no per-link transfer cap.

Choose how open the link is with `access_option`:

- `anyone_with_link` — anyone, no account needed
- `any_registered` — any signed-in Fastio user
- `named_people` — only people you explicitly grant access (the default)

Layer per-user grants on top — `view`, `download`, or `edit` — and optionally protect the link with a password
(sent on the public read endpoints via the `x-ve-password` request header, never the URL).

**Write-back (external edit).** A grantee with an `edit` capability can replace the shared file's content without
being a member of your workspace — they point a normal `action=update` upload session at the **File Share id** as the
`instance_id`. Supply an optional `if_version_id` precondition for safe compare-and-swap: if someone else replaced the
file first, the upload terminates with `status: assembly_failed` and `status_message: CONFLICT_VERSION_MISMATCH:{current_version_id}`,
so the client can re-read and retry. This makes a File Share a tiny, durable collaboration surface — not just a handoff
link.

**Owner-side visibility (one-directional).** Comments that recipients leave on a File Share are visible to the
owning workspace's members: they appear (read-only) on the shared file's comment thread in the workspace, show up in
workspace unified search, and surface as comment activity on the workspace's realtime/activity feed. The reverse is
never true — a File Share recipient only ever sees the comments made under that File Share, never the workspace's
internal comments. Replies, edits, and deletes of a File Share comment go through the File Share's own endpoints.

**No metadata values cross the link.** `GET /current/fileshare/{fileshare_id}/storage/metadata/details/` identifies
the bound file and its extraction eligibility but returns **no metadata field values and no `metadata_facts` key** —
deliberately, and permanently. There is no alternative endpoint that serves them to a link holder, so do not retry or
go hunting for one; see *Metadata* below.

**Agent use case:** Publish a generated report at a stable URL a human keeps bookmarked, then grant a reviewer `edit`
so they can push a corrected version straight back — your agent sees a `file_share_content_updated` event when they do.

> **QuickShare is deprecated.** The old zero-config QuickShare create path now returns 403 (`10756 (Quickshare Deprecated)`); use File Share instead.
> Existing QuickShare links keep serving (view / download / revoke) during the drain.

### 4. Built-In AI — Ask Questions About Your Files

Fastio's AI is a **full agent** — beyond reading and analyzing your file contents, it can take actions on your
behalf, such as creating documents and notes and organizing your content. It operates within your permissions and
plan entitlements, and (like the direct MCP/API tools) its actions consume credits. You can delegate work to it
directly, or use the MCP/API tools yourself.

Fastio's AI lets agents query documents through two chat types, with or without persistent indexing. Both types
augment file knowledge with information from the web when relevant.

#### How the Agent Uses Files

There is no chat "type" to choose — a single agent surface handles every conversation and adapts to what you send. Ask
a general question and it answers from its own knowledge. When the workspace has **intelligence enabled**, it can search
the scope's indexed files and answer with citations (RAG). And you can focus a turn on specific files by attaching
**reference items**:

1. **File references** — attach specific files directly. The AI reads the full content of the attached files. Does not
   require intelligence — any AI-eligible file (with a ready preview or summary) can be attached. Max 20 files, 200 MB total.

2. **Folder references** — attach a folder so the AI grounds answers in its indexed files (RAG), answering with
   citations. Requires intelligence enabled and files in `ready` AI state.

Both are expressed the same way — as reference items in the `references`, `content_parts`, or `subjects` array of a
create-chat or send-message request; the backend resolves each item's full details server-side.

#### Intelligence Setting — On By Default, and When To Turn It Off

The `intelligence` toggle on a workspace controls whether uploaded documents and code files are automatically ingested,
summarized, and indexed for RAG. **New workspaces default to it ON** whenever the plan carries the AI features, so
omitting `intelligence` on create means on. Ingestion costs 10 credits/page and is non-refundable — a 100-page document
costs 1,000 credits — so this can be the largest credit consumer for agent accounts. Creating the workspace costs
nothing; the charge begins with the first document or code file uploaded.

**Keep intelligence on when:**
- You have many files and need RAG queries across them to answer questions
- You want scoped RAG queries against folders or the entire workspace
- You need AI-powered semantic search across large document sets
- You're building a persistent knowledge base that will be queried repeatedly

**Turn it off — pass `intelligence=false` on create, or `update` before uploading — when:**
- You're using the workspace for file storage, sharing, or team coordination
- You only need to analyze specific files (use file attachments instead — no intelligence needed)
- You're uploading deliverables, reports, or outputs that don't need to be queried
- You want to conserve credits — disabling avoids all ingestion costs

Turning it off after content is already indexed flushes the embeddings, and switching it back on re-indexes every file
at full cost, so decide before you upload rather than after.

Even with intelligence disabled, you can still attach **file references** to a chat — any file that has a
ready preview can be attached directly for one-off analysis.

#### AI State — File Readiness for RAG

Every document and code file in an intelligent workspace has an `ai_state` field that tracks its ingestion progress:

| State         | Meaning                                           |
|---------------|---------------------------------------------------|
| `disabled`    | AI processing disabled for this file              |
| `pending`     | Queued for processing                             |
| `in_progress` | Currently being ingested and indexed              |
| `ready`       | Processing complete — file is available for RAG   |
| `failed`      | Processing failed                                 |

**Only documents and code files with `ai_state: ready` are included in folder/file scope searches.** If you upload files and immediately
create a scoped chat, recently uploaded files may not yet be indexed. Use the activity polling endpoint to wait for
`ai_state` changes before querying.

#### Attaching Files and Folders

| Feature              | Folder references (RAG)                     | File references (direct)                 |
|----------------------|--------------------------------------------|------------------------------------------|
| How it works         | Grounds answers in a folder's indexed files| Files read directly by AI                |
| Requires intelligence| Yes                                        | No                                       |
| Requires `ai_state`  | Files must be `ready`                      | File must have a ready preview/summary   |
| Best for             | Many files, knowledge retrieval            | Specific files, direct analysis          |
| Limits               | Up to 100 file/folder references total     | 20 files, 200 MB total                   |
| Default behavior     | Attach nothing = entire workspace          | N/A                                      |

Attach files and folders as **reference items** in the `references`, `content_parts`, or `subjects` array. Each item is a
bare `{type, id}` plus an optional version pin — the backend resolves the full details server-side after checking your
access and the file's AI-readiness.

**File reference:**
```json
{ "type": "file", "id": "{node_id}", "file_details": { "node_id": "{node_id}", "version_id": "{version_id}" } }
```
- `version_id` is optional — omit it to attach the file's current version. Get it from the file's `version` field in
  storage list/details responses.
- Only **file** (or note) nodes are accepted. The file must be AI-eligible (have a ready preview or summary).

**Folder reference:**
```json
{ "type": "folder", "id": "{node_id}", "folder_details": { "node_id": "{node_id}" } }
```
- Attaches a folder so the AI grounds answers in its indexed files (RAG). Only **folder** nodes are accepted.
- **Default scope is the entire workspace** — attach no folder references and the AI searches all indexed documents.

**Strict validation:** a referenced file or folder that cannot be attached — missing, inaccessible, deleted, the wrong
node type, or not AI-eligible — **fails the request** (`1609 (Not Found)` or `1605 (Invalid Input)`; folder attachment
that is not permitted in a share returns `1680 (Access Denied)`). References are no longer silently dropped.

#### Notes as Knowledge Grounding

Notes are markdown documents created directly in workspace storage via the API
(`POST /current/workspace/{id}/storage/{folder}/createnote/`). In an intelligent workspace, notes are ingested and
indexed just like uploaded files. This makes notes a way to store long-term knowledge that becomes grounding material
for future AI queries.

**Agent use case:** Store project context, decision logs, or reference material as notes. When you later ask the AI
"What was the rationale for choosing vendor X?", the note containing that decision is retrieved and cited — even months
later.

Notes within a folder scope are included in RAG queries when intelligence is enabled.

#### How to Write Effective Questions

The way you phrase questions depends on whether you're using folder scope (RAG) or file attachments.

**With folder/file scope (RAG):**

Write questions that are likely to match content in your indexed files. The AI searches the scope for relevant passages,
retrieves them, and uses them as citations to answer your question. Think of it as a search query that returns context
for an answer.

- Good: "What are the payment terms in the vendor contracts?" — matches specific content in files
- Good: "Summarize the key findings from the Q3 analysis reports" — retrieves relevant sections
- Good: "What risks were identified in the security audit?" — finds specific content to cite
- Bad: "Tell me about these files" — too vague for retrieval, no specific content to match
- Bad: "What's in this workspace?" — the AI can't meaningfully search for "everything"

If no folder scope is specified, the search defaults to all indexed documents in the workspace. For large workspaces, narrowing the
scope to specific folders improves relevance and reduces token usage.

**With file attachments:**

You can be more direct and simplistic since the AI reads the full file content. No retrieval step — the AI has the
complete file in context.

- "Describe this image in detail"
- "Extract all dates and amounts from this invoice"
- "Convert this CSV data into a summary table"
- "What programming language is this code written in and what does it do?"

**Controlling response length and style:** There is no `personality` parameter. Control verbosity and tone directly in
the question itself — for example, "In one sentence, summarize this report" or "List only the file names, no
explanations." Agents that need to extract data or get quick answers should ask for terse output explicitly; ask for a
thorough treatment when you need analysis with supporting evidence.

#### Waiting for AI Responses

After sending a message, the AI processes it asynchronously. You need to wait for the response to be ready.

**Turn statuses:**

| Status        | Meaning                              |
|---------------|--------------------------------------|
| `pending`     | Queued for processing (non-terminal) |
| `running`     | AI is generating the response (non-terminal) |
| `complete`    | Response finished                    |
| `failed`      | Processing failed                    |
| `cancelled`   | The turn was cancelled               |
| `lost`        | The turn was lost (worker failure)   |
| `needs_input` | The assistant needs more information and returned a single clarifying question instead of a full response (terminal, not an error). Read the question from the message details (`result.clarification`), then send the user's answer as a new message in the same chat. |

**Option 1: SSE streaming (recommended for real-time display)**

`GET /current/workspace/{id}/ai/agent/{chat_id}/message/{message_id}/read/`

Returns a `text/event-stream` with response chunks as they're generated. The stream ends with a `done` event when the
response is complete (or a `needs_input` event when the assistant returned a clarifying question — listen for it as its
own event and send the user's answer as a new message). Response chunks include the AI's text, citations pointing to
specific files/pages/snippets, and
any structured data (tables, analysis).

**Option 2: Activity polling (recommended for background processing)**

Don't poll the message endpoint in a loop. Instead, use the activity long-poll:

`GET /current/activity/poll/{workspace_id}?wait=95&lastactivity={timestamp}`

When `ai_chat:{chatId}` appears in the activity response, the chat has been updated — fetch the message details to get
the completed response. This is the most efficient approach when you don't need to stream the response in real-time.

**Option 3: Fetch completed response**

`GET /current/workspace/{id}/ai/agent/{chat_id}/message/{message_id}/details/`

Check the `status` field. When it is `complete`, the `result` blob contains the answer and its `citations` (each
pointing to specific files, pages, and snippets). The returned `message` object also carries a `message.actions` list (`turn.actions` on the share
endpoint) — the ordered, replayable record of the actions the AI took during the message (each entry has `seq`, `label`,
`state` of `running`/`done`/`failed`/`cancelled`, `affected_refs`, and `started_at` / `ended_at` timestamps) — so you
can render what it did, not just what it said.

#### Linking Users to AI Chats

To send a user directly to an AI chat in the workspace UI, append a `chat` query parameter to the workspace storage
URL:

`https://{org.domain}.fast.io/workspace/{workspace.folder_name}/storage/root?chat={chat_opaque_id}`

This opens the workspace with the specified chat visible in the AI panel.

#### Supported Content Types

**Indexed for RAG** (requires Intelligence ON):
- Documents (PDF, Word, text, markdown)
- Code files (all common languages)

**File attachments only** (no RAG indexing):
- Spreadsheets (Excel, CSV)
- Images (all common formats) — *RAG indexing coming soon*
- Video (all common formats) — *RAG indexing coming soon*
- Audio (all common formats) — *RAG indexing coming soon*

#### AI Share — Export to External AI Tools

Generate temporary download URLs for your files, formatted as markdown, for pasting into external AI assistants like
ChatGPT or Claude. Up to 25 files, 50MB per file, 100MB total. Links expire after 5 minutes. This is separate from the
built-in AI chat — use it when you want to analyze files with a different model or tool.

**Agent use case:** A user asks "What were Q3 margins?" You have 50 financial documents in an intelligent workspace.
Instead of downloading and parsing all 50, create a chat scoped to the finance folder and ask. The AI
searches the indexed content, retrieves relevant passages, and answers with citations. Pass the cited answer — with
source references — back to the user.

#### Semantic Search — Fast Retrieval Without LLM

Semantic search lets you find relevant document chunks by meaning without creating a chat or waiting for an LLM response.
It returns ranked text snippets with relevance scores — no LLM round-trip, no token cost beyond the search itself.

**When to use search vs chat:**

| | Semantic Search | AI Chat |
|---|---|---|
| **What it does** | Returns raw document chunks ranked by relevance | Creates an LLM-generated answer with citations |
| **Speed** | Fast — vector lookup only | Slower — retrieval + LLM generation |
| **Cost** | Low — no LLM token cost | Higher — LLM tokens consumed per message |
| **Best for** | Retrieval, lookup, finding specific content | Synthesis, analysis, summarizing across documents |
| **Returns** | Text snippets + scores + file references | Natural language answer + citations |

**Endpoints:**

Semantic search is available via the unified storage search endpoint.

```
GET /current/workspace/{workspace_id}/storage/search/?search={query}
GET /current/share/{share_id}/storage/search/?search={query}
```

When workspace intelligence is enabled, results automatically include semantic matches with `relevance_score`, `raw_score`, `score_source`, `content_snippet`, `match_source`, `mimetype`, `media_segment`, and `search_metadata` fields.

`/storage/search/` returns **one result per file** with the best-matching passage
only — a document matching in several places still yields a single row, not one per
passage. Its `relevance_score` is normalised **within the result set**, so its scale
is re-derived for every query. **Use it to order results — do not threshold it, and
do not compare it across queries.**

**To threshold, read `raw_score` alongside `score_source`.** `raw_score` is the
**un-rescaled** retrieval score, on the scale `score_source` names: the merge does not
divide, clamp or round it against the other hits that came back with it. **That is a
statement about rescaling, not a promise the number is constant** — re-running
retrieval can still return a different value, because a `keyword` score is BM25 and
moves with the index statistics. A cutoff still has to be calibrated per engine, per
index and per kind of query.

**`score_source` identifies the scale `raw_score` is on**, and nothing more:
`keyword` (a BM25 score — **unbounded above**, and dependent on the index contents and
the query terms), `semantic` (the content engine's similarity score for the matching
passage), or `filename` (a fixed name-match band rather than a measured score, so
`raw_score` is `null`). There is **no `both`** — which legs matched is a separate
question, answered by `match_source`. It is decided **per hit**, so one response can
carry all three: **group by `score_source` before comparing any two `raw_score`
values**, and never compare a `keyword` value against a `semantic` one. Both fields
are on `/storage/search/` only — the unified `/search/` route does not return them.

**Searching extracted metadata is a different endpoint.** `/storage/search/` has
never searched extracted metadata — it matches filenames, AI-generated summaries
and document content. For metadata, use:

```
GET /current/workspace/{workspace_id}/metadata/search/?q={query}
GET /current/workspace/{workspace_id}/metadata/filters/
GET /current/workspace/{workspace_id}/metadata/filters/{filter_id}/nodes/
```

`/metadata/search/` searches metadata **values**; a saved metadata filter retrieves by
**exact field value** with no text query.

⚠️ **Metadata is workspace-only, and shares have no successor for it.** There is no
`filters` parameter on `/share/{id}/storage/search/`, no `/share/{id}/metadata/...`
routes, and no metadata bucket in `/share/{id}/search/`. Unknown parameters are
accepted rather than refused, so a share request that sends `filters` returns
**HTTP 200 with unfiltered results** and no `metadata_filter` block to reveal it. If
you need metadata search, run it against the **workspace**.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `search` | string | Yes | — | Search query, 2–1,000 characters |
| `search_in` | string | No | `both` | What to match: `filename`, `content`, or `both`. See *Filename search* below. |
| `name_match` | string | No | `auto` | How the filename is matched: `auto`, `exact`, `prefix`, `contains`, `glob`. Ignored when `search_in=content`. |
| `case_sensitive` | string | No | `false` | `true` / `false` / `1` / `0`. Applies to the precise `name_match` values; ignored under `auto`. |
| `files_scope` | string | No | All indexed files | Comma-separated `nodeId:versionId` pairs (max 100), query string only. **Narrows the semantic leg only** — see *Scoping is semantic-only* below. Accepts `type: "file"` **and** `type: "note"` nodes; a folder or link is refused. |
| `folders_scope` | string | No | All indexed files | Comma-separated `nodeId:depth` pairs (max 100, depth 1–10), query string only. **Narrows the semantic leg only** — see *Scoping is semantic-only* below. Folder node ids only — a file, note or link is refused, and so are the `root` and `trash` aliases. |
| `limit` | integer | No | 100 | Results per page, 1–500 |
| `offset` | integer | No | 0 | Pagination offset |
| `details` | string | No | false | When `"true"`, each result includes a `node` field with the full node resource (previews, AI state, versions, metadata, size). Default limit drops to 10 (enrichment is expensive). An explicit `limit` overrides this default. |

The **semantic** channel described above requires `intelligence=true` on the workspace or share, and contributes only
files that have reached `ai.state: indexed`. The endpoint itself does not: with intelligence off it still answers, as a
keyword search over filenames and over any AI-generated summaries already indexed — just without the semantic fields.
See *`search_in=content` Is Not `grep`* below.

**Scoping is semantic-only.** `files_scope` and `folders_scope` narrow the **semantic** leg and nothing else, so a
hybrid response can still contain files from outside the scope carrying `match_source: "keyword"`. **For a hard
boundary, filter on `match_source` yourself — keep `"semantic"` and `"both"`, drop `"keyword"`.** Those are the
results that came from the leg the scope narrowed. ⚠️ **`search_in=content` does not give you that boundary.**
`content` also matches the file's AI-generated summary by keyword, and the scope does not restrict that channel
either — so wherever summary search is open to you, a scoped `content` request still returns out-of-scope
`match_source: "keyword"` hits. Summary search is always open on **workspace** routes, so `search_in=content` never
bounds the result set there; on a **share** it depends on that share's permissions for you. `match_source` filtering
is the only approach that holds everywhere. Because the scope applies to the semantic leg alone, it changes nothing
when that leg does not run — intelligence off, `search_in=filename`, or the leg failing
— and `search_metadata.scoped` says so: it reports the narrowing that was **applied**, not the parameter you sent, so
`scoped: false` on a scoped request means the scope had no effect. A scope that resolves to no files returns **no**
semantic results rather than falling back to the whole workspace, and an entry that is simply wrong — a `versionId`
that is not a version of that file, an unparseable id, a folder in `files_scope`, a file in `folders_scope`, or the
`root`/`trash` folder alias in `folders_scope` — is refused with `1605 (Invalid Input)` naming the entry rather than
being dropped. That includes a value that is not a `nodeId:…` pair at all, a bare `0` among them: to send no scope,
**omit the parameter** rather than passing a placeholder.

**`files_scope` takes files AND notes; `folders_scope` takes folders; links cannot be scoped.** A search result's
`type` is `file`, `folder`, `link` or `note`. Notes are indexed the way files are and come back from semantic search,
so `files_scope` accepts a note's `nodeId:versionId` pair exactly as it accepts a file's — if a note was a hit, you
can scope the next query to it. A **link** has no stored content to index and is accepted by neither parameter. A
node of the wrong type for the parameter it was named in is refused with `1605 (Invalid Input)`, in a message naming
the type the node actually is and, where the other parameter would take it, which one to use instead. Both
parameters are query parameters on a `GET`: a scope placed in a request body is not read, and the search runs
unscoped.

**A scope carries at most 100 references in total**, counting every file named, every folder named, and every
subfolder reached by expanding a `folders_scope` entry to its `:depth`. More than 100 **files** is refused; a
**folder** tree that runs past the limit is truncated instead — you cannot count a subtree before naming it — and the
truncation is reported as `search_metadata.scope_incomplete: true`, present only when something was left out. Treat it
as "the search covered less than you asked for": narrow the `:depth` or name fewer folders and retry, and never read a
truncated answer as a complete one.

#### Filename Search — the `find`-Style Surface

The single most useful thing to know about search as an agent: **you can ask for
the filename side on its own, and match it the way `find` does.**

```
?search=*.pdf&search_in=filename&name_match=glob
?search=Quarterly*.pdf&search_in=filename&name_match=glob     → matches "Quarterly Report.pdf"
?search=Invoice-&search_in=filename&name_match=prefix
?search=2026&search_in=filename&name_match=contains
?search=Q4%20Report&search_in=filename&name_match=exact       → matches "Q4 Report.pdf"
```

`search_in=filename` matches the file's **name only** and skips the content
lookup entirely, so it is fast, cheap, and — critically for an agent — it returns
the same answer whether or not AI features are enabled on the workspace. This is
the surface to reach for when you know, or can describe, what the file is called.

| `name_match` | Behavior | Closest shell idiom |
|---|---|---|
| `auto` (default) | Layered relevance — phrase, prefix, fuzzy, stemmed, substring, with near-exact filename matches promoted | fuzzy search box |
| `exact` | The whole filename equals the query; also matches the extensionless base, so `Q4 Report` finds `Q4 Report.pdf` | `find -name 'Q4 Report.pdf'` |
| `prefix` | Filename starts with the query, taken literally | `find -name 'Invoice-*'` |
| `contains` | Filename contains the query as a literal substring | `find -name '*2026*'` |
| `glob` | Shell-style pattern over the whole filename: `*` = any run, `?` = one character | `find -name '*.pdf'` |

**`glob` spans spaces and hyphens.** The pattern runs against the complete
filename rather than word-by-word, so `Quarterly*.pdf` matches
`Quarterly Report.pdf` and `report-*.xlsx` matches `report-2026-q1.xlsx`. That is
the whole reason `glob` exists — it is the only mode that can match across a space
or a hyphen. `case_sensitive` defaults to `false` (matching `find -iname`) and
folds non-ASCII letters too, so `ÄNDERUNG` matches `änderung.docx`; it does not
strip accents, so `resume` will not match `résumé.pdf`.

**Do not pre-escape your query, and do not lowercase it.** Under `exact`,
`prefix`, and `contains`, `*` and `?` are matched **literally** — `contains` with
`report*` looks for a filename really containing the two characters `report*` and
will not match `report-2026.pdf`. That is what makes a filename containing an
asterisk searchable at all. Only `glob` treats them as wildcards. Escaping is
handled server-side; a client that sends `report\*` will search for the backslash.

**Pattern rules (precise modes only):** the query must be non-empty after
trimming and at most 256 characters, and a `glob` of nothing but `*` and `?` is
rejected. Violations return `1605 (Invalid Input)`. `auto` is unaffected. A single
search examines at most 1,000 matching files, so a deliberately broad pattern can
return a truncated view — prefer the narrowest pattern that answers the question.

The precise values depend on filename indexing that is rolled out per
environment; `auto` works everywhere. If a precise match returns nothing where
you expect a hit, retry with `name_match=auto` before concluding the file is
absent.

#### `search_in=content` Is Not `grep`

`search_in=content` matches the **AI's understanding** of a file — its
AI-generated summary and its meaning-based (semantic) index. Fastio does not
index the literal bytes of your documents, so this is **not** `grep`, not
full-text search, and not a substring scan of file contents. Two consequences:

- A phrase that appears verbatim in a document will not necessarily match, and a
  phrase that never appears in it may match. Ask `content` questions in terms of
  what a document is *about*. If you need an exact string, that string is almost
  always in the filename — use `search_in=filename`.
- `content` matches **two** channels — the AI-generated summary and the
  meaning-based index — and intelligence gates only the meaning-based one.
  Switching AI features off stops new semantic matching; it does not un-index
  summaries already written, so `content` still returns hits for files summarized
  earlier and returns nothing when there are none. Either way the file must have
  reached `ai.state: indexed` at some point: a just-uploaded file is findable by
  name immediately and by content only after indexing completes.

**Tell "no matches" apart from "content search is impossible here."** When a
content channel cannot serve the request, the response is `200` with an **empty**
result set — not an error. Supply `search_in` and the response carries a
capability report:

```json
"search_metadata": {
  "intelligence_enabled": false,
  "semantic_available": false,
  "scoped": false,
  "content_search_available": false,
  "reason": "intelligence_disabled"
}
```

On `content_search_available: false`, do **not** report "no files found." Retry
with `search_in=filename`, or tell the user content search is unavailable here.
`reason` is one of `intelligence_disabled` (AI features are off for the share),
`summary_permission_denied` (the meaning-based channel did not serve the request
*and* this share's permissions do not let **you** search summaries — retrying as
yourself will not help, but `search_in=filename` will), or `content_not_indexed`
(generic fallback); treat unrecognized values as opaque.

`content_search_available` answers *"can content search work here at all?"*, not
*"will this query match?"* — a `true` flag over an empty list is an ordinary
"nothing matched." It is reachable as `false` only on a **share** where neither
channel is open; on a workspace it is always `true` (summary access is always
granted there, even with AI features off), so there is no workspace recovery path
to code for. For the outcome of a given request, read `semantic_available`.

`search_metadata` is returned whenever the response is hybrid (as before), and
additionally on the keyword-only path when you supply `search_in`. The two
capability keys appear only when `search_in` was supplied. All three parameters
are optional and additive: omit them and you get exactly the previous behavior,
query, ranking, and response keys.

**Result shape under `search_in=filename`.** A filename search has no content leg,
so `/storage/search/` returns the **keyword-only** item shape — `name`,
`parent_id`, `type`, `content_snippet: null`, `match_source: "keyword"` — even on
a workspace with intelligence enabled. The hybrid-only fields
(`relevance_score`, `raw_score`, `score_source`, `mimetype`, `media_segment`, `page`)
are **absent**. That is
the same shape you already get whenever intelligence is off, so no new parsing is
needed — but do not require `relevance_score` when you asked for `filename`. The
unified endpoints are unaffected: their `files` bucket keeps its usual item shape
in every mode.

**Response example:**

```json
{
  "result": true,
  "files": {
    "2ltsu-q4mja-cuv7p-gc5yd-lxnsj-wee4": {
      "name": "quarterly-report.pdf",
      "parent_id": "2qk7d-kri4y-yievb-q5hri-eq4io-hij5",
      "type": "file",
      "relevance_score": 1.0,
      "raw_score": 0.87,
      "score_source": "semantic",
      "content_snippet": "The quarterly revenue showed a 15% increase...",
      "match_source": "both",
      "media_segment": null,
      "mimetype": "application/pdf",
      "page": { "start_page": 3, "end_page": 3 }
    }
  },
  "search_metadata": {
    "intelligence_enabled": true,
    "semantic_available": true,
    "scoped": false
  }
}
```

The response is a **map of node id → file entry**, not a `results` array. Every
entry carries `name`, `parent_id` and `type`; pass `details=true` to attach the
full node resource.

With intelligence enabled, the `/storage/search` response also includes:
- `content_snippet` — the actual matching text from semantic search. NULL for keyword-only matches.
- `mimetype` — file MIME type (e.g., `application/pdf`, `audio/mpeg`). Present for semantic matches.
- `media_segment` — `{start_seconds, end_seconds}` identifying the timestamp range in audio/video where the match was found. Only present for audio/video file matches, enabling deep-linking to the exact moment.
- `relevance_score` — relevance score, normalised within the current result set, so its scale is re-derived per query. Order results with it; do not threshold it.
- `raw_score` — the un-rescaled retrieval score, on the scale `score_source` names. `null` when `score_source` is `filename`. `/storage/search/` only.
- `score_source` — which retrieval **scale** `raw_score` is on: `keyword`, `semantic`, or `filename`. Decided per hit; there is no `both`. `/storage/search/` only.
- `match_source` — source of the match: `keyword`, `semantic`, or `both`

**With `details=true`** — the `/storage/search/` endpoint enriches each file entry with a `node` field containing the full node resource:

```json
{
  "result": true,
  "response": {
    "files": {
      "2ltsu-q4mja-cuv7p-gc5yd-lxnsj-wee4": {
        "name": "report.pdf",
        "parent_id": "...",
        "type": "file",
        "relevance_score": 1.0,
        "raw_score": 0.87,
        "score_source": "semantic",
        "content_snippet": "Revenue increased 15%...",
        "match_source": "both",
        "mimetype": "application/pdf",
        "node": {
          "id": "2ltsu-q4mja-cuv7p-gc5yd-lxnsj-wee4",
          "name": "report.pdf",
          "type": "file",
          "size": 123456,
          "previews": { "...": "..." },
          "ai": { "state": "ready" }
        }
      }
    }
  }
}
```

**Agent memory pattern:** Upload context documents (meeting notes, research, reference material) to an intelligent
workspace. Later, use `storage/search` to retrieve relevant chunks without burning LLM credits. This is significantly
cheaper than creating a chat for every lookup and is ideal for agents that need to recall information across sessions —
treat the intelligent workspace as a persistent memory store and search as the retrieval mechanism.

**Agent use case — multi-step research:** An agent researching a topic uploads 200 papers to a workspace with
intelligence enabled. For each research question, it calls `storage/search` to find the most relevant passages, reads the
top results, and only escalates to AI chat when it needs the LLM to synthesize across multiple sources. This approach
uses a fraction of the credits compared to chatting for every question.

#### Unified Search — One Query, Grouped by Type

When you want to search across more than just files in a single call, use the unified search endpoint. It returns results
**grouped by type** into buckets — `files`, `metadata`, and `comments` (the share variant returns the
applicable subset) — each with its own pagination and its own health status.

```
GET /current/workspace/{workspace_id}/search/?search={query}
GET /current/share/{share_id}/search/?search={query}
```

This is a **REST-only** capability — there is no MCP tool for unified search; agents call the endpoint directly. Pass
per-bucket offset/limit params (`files_offset`/`files_limit`, `comments_offset`/`comments_limit`, and likewise for
`metadata`) to page each bucket independently. Every applicable bucket is always searched (the share
endpoint omits `metadata`). Each result item carries a `relevance_score` and an `updated` timestamp plus type-specific
fields. Every bucket is permission-filtered against the same rules as that type's dedicated endpoint, and counts are
computed after permission filtering — search never reveals an item, snippet, or count the caller cannot see. The
workspace `comments` bucket also includes comments left via the workspace's own File Shares (owner-side visibility);
File Share recipients never gain search access to workspace comments. A transient
problem affecting one bucket returns that bucket as `status: degraded` (empty results) without affecting the others. See
*Unified Search* in the Storage Operations reference for the full request/response contract. Use the per-type endpoints
(`storage/search`, `metadata/search`) when you only need one kind of result; use unified search when you want a single
grouped sweep.

The `search_in` / `name_match` / `case_sensitive` parameters described above work
here too and shape the **`files` bucket only** — same values, same defaults, same
escaping and case rules. `?search=*.pdf&search_in=filename&name_match=glob` turns
the `files` bucket into a pure filename lookup while still returning `metadata`
and `comments` matches from the same call. The capability report rides on the
bucket it describes, at `buckets.files.search_metadata`, and is present only when
`search_in` was supplied (`scoped` is always `false` here — this endpoint has no
scope parameters).

### Built-In Help — Ask How-To Questions About Fastio

Beyond querying *your own* files, an agent can ask Fastio **how to use Fastio itself** and get a grounded, product-aware
answer in a single call. `POST /current/how-to/` takes a natural-language `question` (required, ≤2000 chars)
plus optional free-text `context` (≤8000 chars, treated as background data only) and returns HTTP 200 in one of two shapes:

- `status: "answer"` — a grounded `answer` string, plus `escalated` (retained for backward compatibility; always `false` — no deeper pass exists) and `topics_used`.
- `status: "needs_clarification"` — a `questions` array to put back to the user when the question is too vague to answer well.

Branch on `status`; treat `needs_clarification` as a normal prompt for more detail, not an error. The endpoint is
a **top-level, user-authenticated endpoint** — no org in the URL. Access is **open**: there is **no org-membership
requirement, no AI-Agent plan-feature gate, no active-subscription requirement, and no billable entity** — any
authenticated, available user may ask. How-to is **free**: no org, user, or any entity is charged; the LLM call runs
with skip-billing. Abuse is bounded solely by a per-user rate limit (429 + `x-ve-limit-*` headers) plus a per-user
fail-fast mutex — only one how-to request per user runs at a time, so a concurrent second call returns
`145858 (Rate Limited)` / HTTP 429 immediately. Use this when an integration needs authoritative usage guidance at runtime
instead of scraping the docs.
See the [How-To reference](https://api.fast.io/current/llms/howto/) for the full contract.

**Agent use case:** Mid-task, your agent is unsure how to put a password on a share link. It calls
`POST /current/how-to/` with the question, gets back step-by-step API guidance grounded in Fastio's own docs,
and proceeds — no human round-trip, no doc scraping.

### 5. File Preview — No Download Required

Files uploaded to Fastio get automatic preview generation. When humans open a share or workspace, they see the content
immediately — no "download and open in another app" friction.

**Supported preview formats:**

- **Images** — full-resolution with auto-rotation and zoom
- **Video** — HLS adaptive streaming (50-60% faster load than raw video)
- **Audio** — interactive waveform visualization
- **PDF** — page navigation, zoom, text selection
- **Spreadsheets** — grid navigation with multi-sheet support
- **Code & text** — syntax highlighting, markdown rendering

**Agent use case:** Your generated PDF report doesn't just appear as a download link. The human sees it rendered inline,
can flip through pages, zoom in, and comment on specific sections — all without leaving the browser.

### 6. Notes — Markdown Documents as Knowledge

Notes are a storage node type (alongside files and folders) that store markdown content directly on the server. They
live in the same folder hierarchy as files, are versioned like any other node, and appear in storage listings with
`type: "note"`.

#### Creating and Updating Notes

**Create:** `POST /current/workspace/{id}/storage/{parent_id}/createnote/`

- `name` (required) — filename, must end in `.md`, max 255 characters (e.g., `"project-context.md"`)
- `content` (required) — markdown text, max 100 KB. Must be valid UTF-8 (UTF8MB4). Control characters (`\p{C}` except `\t`, `\n`, `\r`) are stripped.

**Update:** `POST /current/workspace/{id}/storage/{node_id}/updatenote/`

- `name` (optional) — rename the note (must end in `.md`)
- `content` (optional) — replace the markdown content (max 100 KB). Must be valid UTF-8 (UTF8MB4). Control characters (`\p{C}` except `\t`, `\n`, `\r`) are stripped.
- At least one of `name` or `content` must be provided

Notes can also be moved, copied, deleted, and restored using the same storage endpoints as files and folders.

#### Notes as Long-Term Knowledge Grounding

In an intelligent workspace, notes are automatically ingested and indexed just like uploaded documents. This makes notes a
powerful way to **bank knowledge over time** — any facts, context, or decisions stored in notes become grounding
material for future AI queries.

When an AI chat uses folder scope (or defaults to the entire workspace), notes within that scope are searched alongside
files. The AI retrieves relevant passages from notes and cites them in its answers.

**Use cases:**
- Store project context, decisions, and rationale as notes. Months later, ask "Why did we choose vendor X?" and the AI
  retrieves the note with that decision.
- After researching a topic, save key findings in a note. Future AI chats automatically use those findings as grounding.
- Create reference documents (style guides, naming conventions, process docs) that inform all future AI queries in the
  workspace.

#### Linking Users to Notes

**Open a note in the workspace UI** — append `?note={opaque_id}` to the workspace storage URL:

`https://{org.domain}.fast.io/workspace/{folder_name}/storage/root?note={note_opaque_id}`

**Link directly to the note preview** — use the standard file preview URL:

`https://{org.domain}.fast.io/workspace/{folder_name}/preview/{note_opaque_id}`

The preview link is more effective if you want the user to focus on reading just that note, while the `?note=` link
opens the note within the full workspace context.

### 7. Comments & Annotations

Humans can leave feedback directly on files, anchored to specific content:

- **Image comments** — anchored to regions of the image
- **Video comments** — anchored to timestamps with frame-stepping and spatial region selection
- **Audio comments** — anchored to timestamps or time ranges
- **PDF comments** — anchored to specific pages with optional text selection
- **Threaded replies** — single-level threads under each comment (replies to replies are auto-flattened)
- **Emoji reactions** — one reaction per user per comment, new replaces previous
- **Mentions** — tag users with `@[user:USER_ID:Display Name]` syntax in the comment body
- **Attachments** — attach up to 25 objects (a file or folder, a sign envelope, a share, a File Share, or a workspace) to a comment as references, inline at create time or via the comment attachment endpoints. Attachment display names are access-gated on read, so render defensively — when an attachment reports `available: false`, never show a name

**A mention inside code is quoted, not a hail — behaviour change.** A mention that sits inside markdown code is treated as text the author is SHOWING, not as a hail: it does **not** notify the named user, and it counts in FULL against the display-text cap instead of being discounted as mention markup. Two consequences worth planning for: someone who is notified today by a mention inside a fence stops being notified, and a long body that hid markup inside fences may now be rejected by the cap that previously discounted it. Only two constructs count as code, both matched per line — a fenced block opened by a run of three or more backticks (or three or more tildes) indented at most three spaces and closed by a run of the same character at least as long, alone on its line (an unclosed fence runs to the end of the body); and a single-line inline backtick span, a run of N backticks closed by a run of exactly N on the SAME line. Everything else is NOT recognised as code and still notifies exactly as before: a fence carrying a blockquote or other container prefix, a fence indented four or more spaces, an indented code block with no fence markers, and a backtick span whose opening and closing runs sit on different lines. The bias is deliberate — failing to recognise code is the behaviour that was already live, while inventing code where there is none would silently drop a real person's notification. When in doubt, it is not code.

**Character limits:** The comment `body` field has a hard maximum of **8,192 characters** — this includes the full mention tag markup. When you convert a short `@name` reference into the bracket syntax (e.g., `@[user:abc123:Jane Smith]`), the expanded string is what counts toward the limit. The display text (everything except mention markup — a mention inside code is not markup for this purpose and counts in full, see above) is separately capped at **500 characters**. Both caps count UTF-8 **characters**, not bytes — a CJK or emoji character costs exactly one — and an over-limit body is rejected, never truncated. Build your comment body with the expanded tags first, then verify the total length before submitting.

**Linking users to comments:** Link users to the file preview URL. The comments sidebar opens automatically in workspace
previews, and in share previews when comments are enabled on the share.

Base preview URL:

`https://{org.domain}.fast.io/workspace/{folder_name}/preview/{file_opaque_id}`

For shares: `https://go.fast.io/shared/{custom_name}/{title-slug}/preview/{file_opaque_id}`

**Deep linking to a specific comment:** Append `?comment={comment_id}` to the preview URL. The UI scrolls to and
highlights the comment automatically:

`https://{org.domain}.fast.io/workspace/{folder_name}/preview/{file_opaque_id}?comment={comment_id}`

**Deep linking to media/document positions:** For comments anchored to specific locations, combine with position
parameters:

- `?t={seconds}` — seeks to a timestamp in audio/video (e.g., `?comment={id}&t=45.5`)
- `?p={pageNum}` — navigates to a page in PDFs (e.g., `?comment={id}&p=3`)

**Agent use case:** You generate a design mockup. The human comments "Change the header color" on a specific region of
the image. You read the comment, see exactly what region they're referring to, and regenerate.

### 8. File Uploads — Getting Files Into Fastio

Agents upload files through a session-based API. There are two paths depending on file size:

#### Small Files (Under 4 MB)

For files under 4 MB, upload in a single request. Send the file as `multipart/form-data` with the `chunk` field
containing the file data, plus `org` (your org domain), `name`, `size`, and `action=create`.

To have the file automatically added to a workspace or share, include `instance_id` (the workspace or share ID) and
optionally `folder_id` (the target folder's OpaqueId, or omit for root). The response includes `new_file_id` — the
permanent OpaqueId of the file in storage. No further steps needed.

```
POST /current/upload/
Content-Type: multipart/form-data

Fields: org, name, size, action=create, instance_id, folder_id, chunk (file)
→ Response: { "result": true, "id": "session-id", "new_file_id": "2abc..." }
```

#### Large Files (4 MB and Above)

Large files use chunked uploads. The flow has five steps:

1. **Create a session** — `POST /current/upload/` with `org`, `name`, `size`, `action=create`, `instance_id`, and
   optionally `folder_id`. Returns a session `id`.

2. **Upload chunks** — Split the file into chunks (chunk size is plan-dependent — query `/upload/limits/` for the exact
   value; last chunk may be smaller). For each chunk, send
   `POST /current/upload/{session_id}/chunk/` as `multipart/form-data` with the `chunk` field (binary data), `order`
   (1-based — first chunk is `order=1`), and `size`. You can upload up to **3 chunks in parallel** per session.

3. **Trigger assembly** — Once all chunks are uploaded, call `POST /current/upload/{session_id}/complete/`. The server
   verifies chunks and combines them into a single file.

4. **Poll for completion** — The upload progresses through states asynchronously. Poll the session details with the
   built-in long-poll:

   `GET /current/upload/{session_id}/details/?wait=60`

   The server holds the connection for up to 60 seconds and returns immediately when the status changes:

   | Status | Meaning | What to Do |
   |--------|---------|------------|
   | `ready` | Awaiting chunks | Upload chunks |
   | `uploading` | Receiving chunks | Continue uploading |
   | `assembling` | Combining chunks | Keep polling |
   | `complete` | Assembled, awaiting storage import. Not accessible for download/preview — can only be imported to storage locations. Can be imported to multiple locations. | Keep polling |
   | `storing` | Being added to storage | Keep polling |
   | **`stored`** | **Done** — file is in storage | Read `new_file_id`, clean up |
   | `assembly_failed` | Assembly error (terminal) | Check `status_message` |
   | `store_failed` | Storage import failed (terminal) | Check `status_message`, handle error |

   Stop polling when status is `stored`, `assembly_failed`, or `store_failed`.

5. **Clean up** — Delete the session after completion: `DELETE /current/upload/{session_id}/`.

#### Optional Integrity Hashing

Include `hash` (SHA-256 hex digest) and `hash_algo=sha256` on each chunk for server-side integrity verification. You can
also provide a full-file hash in the session creation request instead.

#### Resuming Interrupted Uploads

If a connection drops mid-upload, the session persists on the server. To resume:

1. Fetch the session: `GET /current/upload/{session_id}/details/`
2. Read the `chunks` map — keys are chunk numbers already uploaded, values are byte sizes
3. Upload only the missing chunks
4. Trigger assembly and continue as normal

#### Manual Storage Placement

If you omit `instance_id` when creating the session, the file is uploaded but not placed in any workspace or share. You
can add it to storage manually afterward:

```
POST /current/workspace/{id}/storage/{folder}/addfile/
Body: from={"type":"upload","upload":{"id":"{session_id}"}}
```

This is useful when you need to upload first and decide where to place the file later.

#### MCP Binary Upload — Three Approaches

MCP agents have three ways to pass binary data when uploading chunks. Each uses the `upload` tool's `chunk` action
with exactly one of `data`, `blob_ref`, or `content` (for text):

**1. `data` parameter (base64) — simplest for MCP agents**

Pass base64-encoded binary directly in the `data` parameter of the `chunk` action. No extra steps required. Works
with any MCP client. Adds ~33% size overhead from base64 encoding.

**2. `stage-blob` action — MCP tool-based blob staging**

Use the `upload` tool's `stage-blob` action with `data` (base64) to pre-stage binary data as a blob. Returns a
`blob_id` that you pass as `blob_ref` in the `chunk` call. Useful when decoupling staging from uploading or preparing
multiple chunks in advance.

1. `upload` action `stage-blob` with `data` (base64-encoded binary) → returns `{ blob_id, size }`
2. `upload` action `chunk` with `blob_ref` set to the `blob_id`

**3. `POST /blob` endpoint — HTTP blob staging for non-MCP clients**

A sidecar HTTP endpoint that accepts raw binary data outside the JSON-RPC pipe, avoiding base64 encoding entirely.
Useful for clients that can make direct HTTP requests alongside MCP tool calls.

1. `POST /blob` with `Mcp-Session-Id` header and raw bytes as the request body → returns `{ blob_id, size }`
2. `upload` action `chunk` with `blob_ref` set to the `blob_id`

**Blob constraints (apply to both staging methods):**
- Blobs expire after **5 minutes** — stage and consume them promptly
- Each blob is consumed (deleted) on first use and cannot be reused
- Maximum blob size: **100 MB**

**Agent use case:** You're generating a 200 MB report. Create an upload session targeting the client's workspace, split
the file into chunks (size from `/upload/limits/`), upload 3 at a time, trigger assembly, and poll until `stored`. The file appears in the
workspace with previews generated automatically. Use the activity polling endpoint (section 13) to know when AI indexing
completes if intelligence is enabled.

### 9. URL Import — Pull Files From Anywhere

When you need to add a file from the web, use `POST /current/web_upload/` with `source_url` instead of downloading it
locally and re-uploading. This is faster because the file transfers server-to-server — your agent never touches the
bytes.

- Supports any HTTP/HTTPS URL
- Supports OAuth-protected sources: **Google Drive, OneDrive, Dropbox**
- Files go through the same processing pipeline (preview generation, AI indexing if intelligence is enabled, virus
  scanning)

**Check progress after submitting.** Web uploads are processed asynchronously by Fastio's server-side fetch agent,
which may be blocked or rate-limited by the source. The import can fail silently if the source rejects the request, times
out, or returns an error. Monitor the upload status to confirm the file was actually retrieved and stored before
reporting success to the user.

**Security note:** The `web_upload` endpoint instructs the Fastio cloud server to fetch the URL — not the agent's
local environment. The Fastio server is a public cloud service with no access to the agent's local network, internal
systems, or private infrastructure. It can only reach publicly accessible URLs and supported OAuth-authenticated cloud
storage providers. No internal or private data is exposed beyond what the agent could already access through its own
network requests.

**Agent use case:** A user says "Add this Google Doc to the project." You call `POST /current/web_upload/` with the URL.
Fastio downloads it server-side, generates previews, indexes it for AI, and it appears in the workspace. No local I/O —
and no bandwidth consumed by your agent.

### 10. Real-Time Collaboration

Fastio uses WebSockets for instant updates across all connected clients:

- **Live presence** — see who's currently viewing a workspace or share
- **Cursor tracking** — see where other users are navigating
- **Follow mode** — click a user to mirror their exact navigation
- **Instant file sync** — uploads, edits, and deletions appear immediately for all viewers

### Agent Intents — Declare What You Are Doing

When several agents work the same workspace, the expensive collision is the one nobody saw coming. **Agent Intents**
let an agent say what it is doing *while* it is doing it, so a peer sees the overlap **before** it happens instead of
after. An intent is short-lived, workspace-scoped, and expires on its own — it is deliberately not a memory or a chat
surface.

Every route is **workspace-only** — there is no share variant — and requires workspace membership at **Member** or
above, the same bar the storage file-locking identity fields use: a caller below Member does not get to see who is
doing what.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/current/workspace/{workspace_id}/intents/` | Allocate a slot — content-free by design |
| `GET` | `/current/workspace/{workspace_id}/intents/` | Browse the workspace's live intents (topics only) |
| `POST` | `/current/workspace/{workspace_id}/intents/{intent_id}/` | Fill or refine a slot — also the heartbeat |
| `GET` | `/current/workspace/{workspace_id}/intents/{id1},{id2}/` | Expand one or more intents in full, batched |
| `DELETE` | `/current/workspace/{workspace_id}/intents/{intent_id}/` | Release a slot (idempotent) |

Five things decide whether your client is correct:

- **Allocating is content-free, and it is get-or-create — not "mint a new slot".** The key is your credential plus the
  scope you asked for (workspace, `node_id`, user, agent name), so a repeat allocate re-asserts the slot you already
  hold rather than creating a second one. A genuinely fresh slot comes back `state: "allocated"`, `version: 0`,
  `topic: null`; anything else means you were handed a slot that already existed — do not release it, and do not
  assume its content is yours to overwrite.
- **An unfilled slot is occupancy, not an incomplete write.** `state: "allocated"` with a `null` `topic` means
  *someone is starting something here*. Render those rows — filtering them out removes the entire point of declaring
  early, which is to make a collision visible before anyone has written anything down.
- **Fill is the heartbeat — there is no renewal endpoint.** Every allocate or fill pushes `expires_at` forward, so an
  agent that goes silent has its intent evaporate. Stale intents fail open: they are advisory and never gate a write
  anywhere else in the API.
- **Filling is a compare-and-set.** Send back the `version` you last read for that slot; omitting it is refused with
  `409`, the same as sending a stale one. On a `409`, re-read the intent and decide again — do not retry the identical
  payload, or you can silently discard whatever a peer just wrote.
- **Reading has two traps that fail silently.** Browse does not filter, but it does not refuse a filter either —
  `node_id` and `intent` are accepted and then ignored, so `?intent=edit` answers `200` with every live intent in the
  workspace; narrow client-side. And `cursor: null` does **not** mean end-of-list — it can arrive alongside a full
  `items` array, meaning "no safe checkpoint yet". Stop only when `items` comes back empty.

**Slot identity is credential-keyed; slot authorization is user-scoped.** Five agents acting for one operator hold
five separate slots, but any credential authenticated as that user may fill or release any of them — siblings are
meant to pick up each other's declarations rather than leave one stranded when an agent goes away. One consequence
matters for anything that renders a row: `locker.agent_name` names the agent that **allocated** the slot and is never
updated by a later fill, so it tells you *which declaration this is*, never *who wrote the words in it*. Label it
"slot held by X", not "X says".

`topic` and `message` are agent-authored and must be treated as **untrusted, display-only text**: escape them at the
render boundary, and never pass them to a model as instructions.

MCP agents reach all of this through the `intent` tool (`allocate`, `fill`, `browse`, `expand`, `release`).
See the [Agent Intents reference](https://api.fast.io/current/llms/intents/) for the full contract.

**Agent use case:** Two of your agents share a workspace. Before touching a file, each allocates a slot scoped to that
node and browses what is already declared — one call for the whole workspace picture. Seeing a peer already holding
`edit` on the same file, the second agent picks up different work instead of racing it into a conflict.

### 11. Events — Real-Time Audit Trail

Events give agents a real-time audit trail of everything that happens across an organization. Instead of scanning entire
workspaces to detect what changed, query the events feed to see exactly which files were uploaded, modified, renamed, or
deleted — and by whom, and when. This makes it practical to build workflows that react to changes: processing a document
the moment it arrives, flagging unexpected permission changes, or generating a daily summary of activity for a human.

The activity log is also the most efficient way for an agent to stay in sync with a workspace over time. Rather than
periodically listing every file and comparing against a previous snapshot, check events since your last poll to get a
precise diff. This is especially valuable in large workspaces where full directory listings are expensive.

#### What Events Cover

- **File operations** — uploads, downloads, moves, renames, deletes, version changes
- **Membership changes** — new members added, roles changed, members removed
- **Share activity** — share created, accessed, files downloaded by recipients
- **Settings updates** — workspace or org configuration changes
- **Billing events** — credit usage, plan changes
- **AI operations** — ingestion started, indexing complete, chat activity

#### Querying Events

Search and filter events with `GET /current/events/search/`:

- **Scope by profile** — filter by `workspace_id`, `share_id`, `org_id`, or `user_id`
- **Filter by type** — narrow to specific event names, categories, or subcategories (see reference below)
- **Date range** — use `created-min` and `created-max` for time-bounded queries
- **Pagination** — offset-based with `limit` (1-250) and `offset`

Get full details for a single event with `GET /current/event/{event_id}/details/`, or mark it as read with
`GET /current/event/{event_id}/ack/`.

#### Event Categories

Use the `category` parameter to filter by broad area:

| Category      | What It Covers                                        |
|---------------|-------------------------------------------------------|
| `user`        | Account creation, updates, deletion, avatar changes   |
| `org`         | Organization lifecycle, settings                       |
| `workspace`   | Workspace creation, updates, archival, file operations |
| `share`       | Share lifecycle, settings, file operations              |
| `node`        | AI indexing pipeline only (not file operations)        |
| `ai`          | AI chat, summaries, RAG indexing                       |
| `invitation`  | Member invitations sent, accepted, declined            |
| `billing`     | Subscriptions, trials, credit usage                    |
| `apps`        | Application integrations                               |
| `metadata`    | Metadata extraction and key-value updates              |

> **File and folder activity is NOT category `node`.** Despite the name, `node` covers only the AI
> indexing pipeline (e.g. `node_ai_summary_created`), and most of its events are internal and never
> returned by the API. It lives in **two** categories instead — `category=workspace`
> (`workspace_storage_file_added`, `_updated`, `_deleted`, `_moved`, `_restored`, `_copied`, and the
> matching `workspace_storage_folder_*` events) and `category=share`, a full parallel set
> (`share_storage_file_added`, `_updated`, `_deleted`, `_moved`, `_restored`, `_copied`, and the
> matching `share_storage_folder_*` events). Filtering only `workspace` silently omits all share
> activity — query both categories for a complete picture of what happened to files.

#### Event Subcategories

Use the `subcategory` parameter for finer filtering within a category:

| Subcategory      | What It Covers                                       |
|------------------|------------------------------------------------------|
| `storage`        | File/folder add, move, copy, delete, restore, download |
| `comments`       | Comment created, updated, deleted, mentioned, replied, reaction |
| `members`        | Member added/removed from org, workspace, or share   |
| `lifecycle`      | Profile created, updated, deleted, archived          |
| `settings`       | Configuration and preference changes                 |
| `security`       | Security-related events (2FA, password)              |
| `authentication` | Login, SSO, session events                           |
| `ai`             | AI processing, chat, indexing                        |
| `invitations`    | Invitation management                                |
| `billing`        | Subscription and payment events                      |
| `assets`         | Avatar/asset updates                                 |
| `upload`         | Upload session management                            |
| `transfer`       | Cross-profile file transfers                         |
| `import_export`  | Data import/export operations                        |
| `quickshare`     | Quick share operations                               |
| `metadata`       | Metadata operations                                  |

#### Common Event Names

Use the `event` parameter to filter by exact event name. Here are the most useful ones for agents:

**File operations (workspace):**
`workspace_storage_file_added`, `workspace_storage_file_deleted`, `workspace_storage_file_moved`,
`workspace_storage_file_copied`, `workspace_storage_file_updated`, `workspace_storage_file_restored`,
`workspace_storage_folder_created`, `workspace_storage_folder_deleted`, `workspace_storage_folder_moved`,
`workspace_storage_download_token_created`, `workspace_storage_zip_downloaded`,
`workspace_storage_file_version_restored`, `workspace_storage_link_added`,
`workspace_storage_lock_overridden`

**File operations (share):**
`share_storage_file_added`, `share_storage_file_deleted`, `share_storage_file_moved`,
`share_storage_file_copied`, `share_storage_file_updated`, `share_storage_file_restored`,
`share_storage_folder_created`, `share_storage_folder_deleted`, `share_storage_folder_moved`,
`share_storage_download_token_created`, `share_storage_zip_downloaded`,
`share_storage_lock_overridden`

**Comments:**
`comment_created`, `comment_updated`, `comment_deleted`, `comment_mentioned`, `comment_replied`,
`comment_reaction`

**Membership:**
`added_member_to_org`, `added_member_to_workspace`, `added_member_to_share`,
`removed_member_from_org`, `removed_member_from_workspace`, `removed_member_from_share`,
`membership_updated`

**Workspace lifecycle:**
`workspace_created`, `workspace_updated`, `workspace_deleted`, `workspace_archived`, `workspace_unarchived`

**Share lifecycle:**
`share_created`, `share_updated`, `share_deleted`, `share_archived`, `share_unarchived`,
`share_imported_to_workspace`

**File Share lifecycle:**
`file_share_created`, `file_share_updated`, `file_share_content_updated`, `file_share_deleted`,
`file_share_access_granted`, `file_share_access_revoked` (`file_share_content_updated` fires when an
`edit`-grant holder replaces the shared file's content)

**AI:**
`ai_chat_created`, `ai_chat_new_message`, `ai_chat_updated`, `ai_chat_deleted`, `ai_chat_published` (chat publish is currently disabled platform-wide -- `capabilities.can_publish_agent_chat` is `false`),
`node_ai_summary_created`, `workspace_ai_share_created`

**Metadata:**
`metadata_kv_update`, `metadata_kv_delete`, `metadata_kv_extract`

**Historical only — still present in stored event history, but nothing emits them any more** (the endpoints that
produced them were retired with the template surface): `metadata_template_update`, `metadata_template_delete`,
`metadata_view_create`, `metadata_view_update`, `metadata_view_delete`, `metadata_template_select`. Query them to read
the past; do not wait on them.

**Quick shares (deprecated — see File Share lifecycle):**
`workspace_quickshare_created`, `workspace_quickshare_updated`, `workspace_quickshare_deleted`,
`workspace_quickshare_file_downloaded`, `workspace_quickshare_file_previewed`

**Invitations:**
`invitation_email_sent`, `invitation_accepted`, `invitation_declined`

**User:**
`user_created`, `user_updated`, `user_deleted`, `user_email_reset`, `user_asset_updated`

**Org:**
`org_created`, `org_updated`, `org_closed`

**Billing:**
`subscription_created`, `subscription_cancelled`, `billing_free_trial_ended`

#### Example Queries

**Recent comments in a workspace:**
```
GET /current/events/search/?workspace_id={id}&subcategory=comments&limit=50
```

**Files uploaded to a share in the last 24 hours:**
```
GET /current/events/search/?share_id={id}&event=share_storage_file_added&created-min=2025-01-19 00:00:00 UTC
```

**All membership changes across an org:**
```
GET /current/events/search/?org_id={id}&subcategory=members
```

**AI activity in a workspace:**
```
GET /current/events/search/?workspace_id={id}&category=ai
```

**Who downloaded files from a share:**
```
GET /current/events/search/?share_id={id}&event=share_storage_download_token_created
```

#### AI-Powered Summaries

Request a natural language recap of recent activity with `GET /current/events/search/summarize/`. Returns event counts,
category breakdowns, and a narrative summary. Focus the summary on a specific workspace or share, or summarize across
the entire org.

**Agent use case — stay in sync:** You manage a workspace with 10,000 files. Instead of listing the entire directory
tree to find what changed, query events since your last check. You get a precise list: "3 files uploaded, 1 renamed,
2 new members added" — with timestamps, actors, and affected resources.

**Agent use case — react to changes:** A client uploads tax documents to a Receive share. The events feed shows the
upload immediately. Your agent detects it, processes the documents, and notifies the accountant — no polling the file
list required.

**Agent use case — report to humans:** A human asks "What happened on the project this week?" You call the AI summary
endpoint scoped to their workspace and return a clean narrative report — no log parsing required.

### 12. Activity Polling — Wait for Changes Efficiently

After triggering an async operation (uploading a file, enabling intelligence, creating a share), don't loop on the
resource endpoint to check if it's done. Instead, use the activity long-poll endpoint:

`GET /current/activity/poll/{entity_id}?wait=95&lastactivity={timestamp}`

The `{entity_id}` is the profile ID of the resource you're watching — a workspace ID, share ID, or org ID. For
**upload sessions**, use the **user ID** (since uploads are user-scoped, not workspace-scoped until the file is added
to storage).

The server holds the connection open for up to 95 seconds and returns **immediately** when something changes on that
entity — file uploads complete, previews finish generating, AI indexing completes, comments are added, etc.

The response includes activity keys that tell you *what* changed (e.g., `storage:{fileId}` for file changes,
`preview:{fileId}` for preview readiness, `ai_chat:{chatId}` for chat updates, `ai_state:{fileId}` for AI indexing
state changes, `upload:{uploadId}` for upload completion). Pass the returned `lastactivity` timestamp into your next
poll to receive only newer changes.

This gives you near-instant reactivity with a single open connection per entity, instead of hammering individual
endpoints.

**WebSocket upgrade:** For true real-time delivery (~300ms latency vs ~1s for polling), connect via WebSocket at
`wss://{host}/api/websocket/?{auth_token}` — the JWT is the entire query string (no `token=` key, no other query parameters). The server pushes activity arrays as they happen:

```json
{"response": "activity", "activity": ["storage:2abc...", "preview:2abc..."]}
```

You then fetch only the resources that changed. If the WebSocket connection fails, fall back to long-polling — the data
is identical, just slightly higher latency.

**Share channels are scoped per recipient.** On a share channel, each frame is scoped to the receiving guest's share
access before delivery: a guest is pushed only the changes their access level permits them to observe, and content
detail is collapsed or dropped for guests with restricted file visibility. Do not assume a share guest will be notified
of every change on the channel. Members and all non-share channels (user / organization / workspace) receive
the full frame. Share tokens are also shorter-lived — inspect `expires_in` on the auth response and refresh before it
elapses.

**Agent use case:** You upload a 500-page PDF and need to know when AI indexing is complete before querying it. Instead
of polling the file details endpoint every few seconds, open a single long-poll on the workspace. When
`ai_state:{fileId}` appears in the activity response, the file is indexed and ready for AI chat.

### 13. Metadata — Structured Data on Files

The metadata system attaches structured, typed key-value data to files: invoice amounts, contract parties,
document categories, any domain-specific field. You then select and sort files by those values.

**You do not declare a schema first.** Extraction runs automatically when an eligible file is ingested,
and the fields it produces become the workspace's vocabulary. Your job as an agent is to *discover* what a
workspace already holds and *select* against it — not to set anything up.

> 🔴 **Templates are retired.** The older model — declare a template, map files into it, AI-match the
> rest — is gone. Your client may still list `template-*` actions; calling one fails. `template_id` still
> appears on some responses as historical provenance and is inert.
>
> **The two failures look different, and the difference is not about you.** Most template paths were
> deleted outright, so they no longer route at all and you get the platform's not-routed answer,
> **`9992`** (HTTP 404) — the same thing a mistyped path produces. Two node-scoped ones still have a
> handler and answer **`410 Gone`**, each with its own `error.code`:
> `/storage/{node_id}/metadata/templates/` returns **`131380`**, and
> `/storage/{node_id}/metadata/template_select/` returns **`122209`**. Either way the path is finished: do not
> retry, do not vary the id, and do not report it as an outage. Only the `410` says that on its own — the
> `9992` cannot be told apart from a typo, which is why the retirement is documented here rather than
> left to the response to explain.

#### Architecture

Two layers, not three:

1. **The field vocabulary** — the flat, per-workspace set of field names metadata is stored under, with
   each field's type. It is discovered, not declared: whatever extraction produced is what is there. The
   declarable types are `string`, `int`, `float`, `bool`, `json`, `url` and `datetime`. A field is
   identified by its `name`; there is no field id.

2. **Node metadata** — the actual values on one file, in ONE set.

   **`metadata_facts` is that set**: typed values carrying a `source` that says who set each one (`ai`,
   `exif`, `mediainfo`, `user`, `validated_server`), plus a `confidence` and the model's `rationale`
   where extraction produced them. It is the set that metadata search, the field and value listings, and
   the filter listings all read.

   🔴 **The older `template_metadata` and `custom_metadata` blocks are GONE from every response.** They
   held values stored against a template historically, plus everything else. No Fastio endpoint returns
   either key any more, and the route that wrote them is retired (see *Values on one file* below). There
   is no longer a second corpus to reconcile against, no precedence ladder to run between two sets, and
   nothing for a client to merge: read `metadata_facts` and stop. If your client still branches on
   whether a response carries them, that branch is dead code — remove it rather than treating the
   absence as an error or an empty file.

   🔴 **One surface has NO successor — a File Share link.** "Read `metadata_facts` instead" holds on
   every workspace surface and is **false** on
   `GET /current/fileshare/{fileshare_id}/storage/metadata/details/`, which identifies the bound file
   and its extraction eligibility and returns **no metadata field values and no `metadata_facts` key**.
   Values are member-gated while a link admits anonymous recipients, so **no metadata value crosses a
   File Share link** — deliberately, and this is the settled end state, not a temporary restriction.
   **There is no alternative endpoint** that serves them to a link holder, nothing is queued to add
   one, and the response carries no header or field marking the omission. So do not go looking for
   another route, do not retry, and do not build a viewer that waits for the values to appear. A
   workspace member reading the same file through the workspace endpoint gets its facts.

**Every extracted file also carries `category` and `sub_category`** — a coarse handle and a fine one
(e.g. `Legal` / `Contract`). These let you narrow by document kind without discovering the vocabulary
first, which makes them the right opening move for most tasks.

They are also **the only fields whose values come from a closed list**:

- `category` — `Legal`, `Finance`, `Sales`, `Marketing`, `Engineering`, `Product`, `Operations`, `People`, `Research`, `Media`, `Personal`, `Other`
- `sub_category` — `Contract`, `Agreement`, `Invoice`, `Receipt`, `Statement`, `Report`, `Proposal`, `Specification`, `Policy`, `Plan`, `Correspondence`, `Presentation`, `Record`, `Resume`, `Identifier`, `Image`, `Video`, `Audio`, `Dataset`, `Other`

Extraction never writes a value outside the list — an off-list answer is replaced with `Other` before it
is stored, and the stored spelling is always the one above, so a filter on `Finance` needs no `finance`
variant. **The list is published as `constraints.allowed`** on each of those two fields in the field-vocabulary response, computed from the same list the writer enforces — so build a picker from that rather than hardcoding it. It constrains what is written **from now on**; rows that predate it can still hold other values, so keep tolerating an unknown one.

**Both writers answer an off-list value the same way: they replace it with `Other`.** Extraction does
it because a model cannot be asked to try again mid-job; the user fact write does it for consistency, so
there is ONE rule for the closed list rather than two. A different case or spacing of a listed value is
not off-list — it is stored under the published spelling.

⚠️ **This is silent.** Writing `"category": "Procurement"` returns `200` and stores `Other`. There is no
error and no rejected-entry list, so **an agent cannot detect it from the status** — read the
classification back from the response, which echoes stored state, before reporting what it set.

**Whole-object write-back is safe on this axis.** A file whose stored `category` predates the list —
extracted before it closed, migrated from the template system, or inherited by a copy — can be sent
back without failing the request; the off-list value is replaced with `Other` rather than rejected.
⚠️ **But that replacement is a silent EDIT to a field you did not mean to change**, so echoing a whole
object back rewrites a legacy classification to `Other` as a side effect. **Send only the fields you are
changing, or `null` for untouched cells** (`null` asserts nothing) — not because the request would fail,
but because it would quietly succeed at something you did not ask for.

Two sets of older rows are outside that guarantee: values migrated from the template system and files
extracted before the list was closed on 2026-08-23. Both were recorded before the list existed — the
migrated set includes human-typed values AND the previous extractor's output — and both can hold
spellings the list does not contain.
`Other` is a real answer for a document that fits nothing, not an error. For what a
PARTICULAR workspace actually holds, with counts, read
`GET /current/workspace/{id}/metadata/fields/?field=category`.

#### Discovery — start here

| Endpoint | Description |
|----------|-------------|
| `GET /current/workspace/{id}/metadata/fields/` | The field names this workspace stores metadata under, with types |
| `GET /current/workspace/{id}/metadata/fields/?field={name}` | The values that one field is observed to hold, with occurrence counts |
| `GET /current/workspace/{id}/metadata/eligible/` | Files and notes eligible for extraction, each with its extracted metadata values inline (`metadata_facts`) — one call renders a whole workspace metadata view; folders and links never qualify. Narrow it with `parent_id` (one folder's direct children) and `category` (a closed list — an off-list value is refused, not ignored) |

Two calls — one field listing plus one value listing — are enough to build a correct filter. Prefer that
over probing file by file.

#### Selection — filters

A **filter** is a `predicate` (which files) plus an optional `projection` (which fields to show, in what
order, and how to sort). A predicate is an array of `{"field", "operator", "value"}` clauses, AND-combined,
at most **5**. **An empty predicate is valid and means match all.**

| Endpoint | Description |
|----------|-------------|
| `POST /current/workspace/{id}/metadata/filters/` | Create a saved filter (**JSON** body: `name`, `predicate`, optional `description` / `projection`) |
| `GET /current/workspace/{id}/metadata/filters/` | List the workspace's saved filters |
| `GET` / `PUT` / `DELETE` `/current/workspace/{id}/metadata/filters/{filter_id}/` | Read, update, delete one |
| `GET /current/workspace/{id}/metadata/filters/{filter_id}/nodes/` | Execute one and list the files it matches |

Saved filters are **workspace-shared** — any member may read, edit and delete them — and names are unique
per workspace. A name stays reserved after deletion, so re-using one returns `1660 (Conflict)`.

For a one-off selection, pass the same predicate inline as `filters` on
`GET /current/workspace/{id}/storage/search/` instead of saving anything.

> ⚠️ **Check the `metadata_filter` block in a search response before you trust the result.** If a client
> strips the `filters` parameter, the search still succeeds and returns a complete-looking but
> **unfiltered** page. The *absence* of `metadata_filter` from the response is what tells you the
> predicate never ran — the item list alone cannot.

#### Values on one file

| Endpoint | Description |
|----------|-------------|
| `GET /current/workspace/{id}/storage/{node_id}/metadata/facts/` | Every typed fact on the file, joined to its canonical field name |
| `GET /current/workspace/{id}/storage/{node_id}/metadata/details/` | The file's metadata — `metadata_facts` (returned first), plus the node pointer, `template_id` and extraction eligibility |
| `POST /current/workspace/{id}/storage/{node_id}/metadata/facts/` | **Set a value yourself, overriding what extraction found.** Writes `source: "user"`, which outranks `ai` — so your value sticks and a later re-extraction will not overwrite it |
| `DELETE /current/workspace/{id}/storage/{node_id}/metadata/` | Delete metadata keys |
| `GET /current/workspace/{id}/storage/{node_id}/metadata/versions/` | Metadata version history |

🔴 **`POST .../storage/{node_id}/metadata/update/` is RETIRED and answers `410 Gone`** with a message
naming its replacement. It is not deprecated-but-working: nothing you send it is read, and no request
shape gets a different answer. **Do not retry it, do not vary the payload, and do not report it as an
outage** — the path is finished. Every node-metadata write now goes to `metadata/facts/`. If your client
library still offers a "set file metadata" action, check which path it calls before you trust it; one
built against the old route will fail on every call until it is updated.

**There is now ONE write route, and it writes to the one set.** `metadata/facts/` records
`source: "user"`, which outranks `ai` on the precedence order above, so your value replaces what
extraction found **and survives re-extraction** — a later automatic pass will not overwrite it. It is
also the set every read surface uses, so the value shows up in metadata search and the field listings.

The whole payload is validated before anything is written, and **no value is written by a refused
request** — not the offending entry and not the usable ones beside it. Clearing a value is not available
on this route: a null does not overwrite. Remove a value with the delete route instead.

**Confirm a write from what it returns, not from `result: true`.** The response is the same read the
`GET` on that path would have given you — the file's stored facts, as stored — so compare the returned
`value` against WHAT YOU SENT. There is no `skipped` list to consult and no partial save to detect: the
write is all-or-nothing, so either every named field landed or the request was refused and named the
offending fields in `error.params`.

#### Value shapes — a malformed value is DROPPED, not corrected

Extraction writes each value in the shape its declared type requires, and so should you. A value that
does not fit is **skipped**: the field keeps its previous value, the rest of the write lands, and the
call still succeeds — so a malformed value looks exactly like a field the document never mentioned.

- `string` — `Acme Holdings Ltd` — plain text, at most 4096 characters
- `int` — `1234`, `-7` — a whole number, optionally signed. Never `1,234`, `$1234`, `12 units`, `3.5`
- `float` — `1234.56`, `-0.5`, `1234` — a number, optionally signed. Write a whole number plainly — adding `.0` to a large one silently stores a different number. Never `1,234.56`, `$1234.56`, `1234,56`, `9007199254740993`
- `bool` — `true`, `false` — exactly `true` or `false`. Never `maybe`
- `datetime` — `2026-01-02T15:04:05Z`, `2026-03-15` — ISO 8601. Use the date-only form when the document gives no time of day; it is stored as midnight UTC. Never `January 2, 2026`, `02/01/2026`, `yesterday`, `1767358800`
- `url` — `https://example.com/path` — an absolute `http`/`https` URL. Never `example.com`, `/path/only`, `ftp://example.com`
- `json` — `{"key":"value"}` — a JSON object or array — not a bare string or number. Never `{key: value}`, `hello`

**Money is two fields.** Give the amount as a plain number declared `float` — always `float`, even for a whole amount, because a field's type is fixed by its first value and an `int` field silently drops every later fractional one. Give the ISO 4217 code as a `string` in a field named after the amount with `_currency` appended — `invoice_total` = `1234.56`, `invoice_total_currency` = `USD`. Two caveats: the pair is not atomic, so either half can be refused while the other lands; and a range over the amount alone still compares USD against EUR as one number, so predicate on the currency field too.

**Your writes are held to the same shapes, but they FAIL differently.** Extraction skips a value it cannot fit and carries on; `metadata/facts/` refuses the whole request instead — nothing is written, every field keeps what it had, and `error.params` names each offending field. So a malformed value in your own write is loud, not silent. **Still verify by reading the values back**, because a value can be accepted and stored in a shape you did not intend.

A `datetime` with no time of day is written date-only and stored as midnight UTC. Values are normalised
to UTC on the way in, and so is a filter literal, so any accepted spelling of the same instant matches.

#### Extraction

| Endpoint | Description |
|----------|-------------|
| `POST /current/workspace/{id}/storage/{node_id}/metadata/extract/` | Queue extraction for one file — `202 Accepted` + `job_id` |
| `POST /current/workspace/{id}/storage/{node_id}/metadata/extract-all/` | Queue extraction for every eligible file under a folder (`{node_id}` must be a folder; `root` is accepted, which makes it workspace-wide). Optional `fields` restricts the run to named fields. Workspace **admin**, and throttled more tightly than the per-file route |

**Scoping a run to one field.** Both extract routes take an optional `fields` — a JSON array of field
names (`fields=["Invoice Total"]`) — so "I added a field, now fill it in" does not mean re-extracting
everything. Together they cover the three scopes: one file (per-file route), a folder and everything
under it (`extract-all` on that folder), the whole workspace (`extract-all` on `root`). Only fields the
workspace vocabulary already holds may be named — declared or produced — and an unknown name is **rejected rather than ignored** —
dropping it would empty the scope, and an empty scope means a full run. Per file only the named fields
that file is **missing** are extracted, so a file that already has them normally costs nothing (if the
server cannot read what a file already holds it falls back to the full named scope for that file, and
charges for it, rather than leave a silent gap). **The two routes do not share a billing claim** — the
folder route bills what each file was actually missing, the per-file route bills the whole set you
named, so asking both for the same file can be charged twice. A scoped run is
always template-free, so `template_id` comes back `null`; the response echoes `fields` back so you can
confirm the scope took effect.

**Automatic on ingest** is the normal path and needs no call at all: an eligible file is extracted as it
lands, documents, spreadsheets, images and code alike. The two endpoints above are for the cases
automation did not cover — a file that predates the feature, or one you want re-run.

**Both are asynchronous and neither returns values.** Poll `GET /current/workspace/{id}/jobs/status/` and
watch the `metadata_extract` array for your entry; once it reports `completed`, read the values from
`.../metadata/facts/`. State transitions also fire on the activity stream, which is cheaper than polling
if you are already subscribed.

**Matching your entry has two traps, and both fail silently.**

- 🔴 **`node_id` comes back UNHYPHENATED.** You call
  `.../storage/26t7z-x432g-nzlqq-pgqz7-r5i2h-lawn/metadata/extract/`, and both the response and the
  status entry report `"node_id": "26t7zx432gnzlqqpgqz7r5i2hlawn"`. Comparing the entry against the
  hyphenated id you put in the path is simply false — no error, the array just never seems to contain
  your file. Match on the `node_id` the extract response handed back, not the one you wrote into the URL.
- 🔴 **`job_id` is `null` while the extraction is in flight.** The entry is seeded before the job row
  exists, so it reports `job_id: null` for as long as it is `queued`, then carries the exact `job_id`
  from the `202` once it reaches `completed` or `errored`. It is still the only key that identifies
  **your request** — `node_id` names the file — so use it to confirm a terminal entry is yours, and use
  `node_id` to find the entry while it is still running.

🔴 **A per-file entry started by this route goes `queued` → `completed` (or `errored`) and never reports
`in_progress`.** Nothing republishes it in between, so an agent that waits to observe `in_progress`
before it starts watching for the result waits forever.

Repeating an identical per-file request while one is in flight **always mints a new job and returns a
new `job_id`** — this route never hands back the id of a job already in flight, so two `202`s with
different `job_id`s do not mean two extractions ran. It is safe to repeat all the same: the protection
is the durable record of what has been extracted, keyed on the file's current version and the `fields`
scope, so a repeat of the same scope does no work and is not billed again. A file whose current version
has already been extracted answers `status: "already_extracted"` and queues nothing — but only for an
**unscoped** request; a request naming `fields` always returns `202`.

**Extraction never overwrites a value a person wrote.** A hand-written value carries `source: "user"`,
which outranks `ai` on the precedence order, so a later automatic pass leaves it alone. Confirm what an
extraction changed by reading the values back, not from the job status.

🔴 **NO WRITE hands a field back to extraction — only `DELETE` does.** `null` on `metadata/facts/` is a
no-op: it clears nothing and, if the field does not already exist, creates nothing, so an agent sending
`null` expecting a clear gets a success response and no change. And `""`, `0` and `false` are worse than
a no-op for this purpose — they are values a person chose, so they store at `source: "user"` and pin the
field **permanently** out of extraction's reach. To make a field extractable again, remove its value with
`DELETE /current/workspace/{id}/storage/{node_id}/metadata/` naming the field in `keys`. (The route that
once cleared a field by writing `null` is the retired `metadata/update/`; that behaviour went with it.)

Two things follow that are worth designing around. **Deleting a value does not delete the field** — the
name stays in the workspace vocabulary and keeps counting against the plan's field cap. And **deleting
does not by itself queue a re-extraction**: a full re-extract of a file whose current version was already
extracted answers `already_extracted` and queues nothing, so name the field in a `fields`-scoped
`metadata/extract/` call to have it filled again.

#### Agent Use Cases

- **Find documents of a kind:** read the values of `category` / `sub_category`, then filter on them. No
  setup, and it works on a workspace you have never seen before.
- **Build a table:** a predicate picks the files, a projection picks the columns and the sort. Note that a
  filtered listing returns *every* in-scope field of each matched file, not only the fields the predicate
  named — that is what makes it usable as a table, and it means you must not assume every field in the
  response satisfied the predicate.
- **Backfill an old folder:** `extract-all` on the folder, then poll. Files already extracted keep their
  values.
- **Compliance sweep:** filter on the field that should be present; the files missing it are the ones that
  do not come back. Read the field's value list first to learn how the workspace actually spells the
  values.
- **One-off vs kept:** pass `filters` inline on storage search for a question you are asking once; save a
  filter when the same question will be asked again by someone else.

### 14. Reference Values — Enums & Constraints

This section lists valid values for commonly used parameters across the API.

#### Preview Types

Valid `preview_type` values (used in preview read/preauthorize endpoints):
`bin`, `thumbnail`, `image`, `hlsstream`, `pdf`, `spreadsheet`, `audio`, `mp4`

Preview states returned in file details: `unknown`, `not possible`, `not generated`, `error`, `in progress`, `ready`

#### Image Transformations

The `transform_name` in transform endpoints is `image`. Supported query parameters:

| Parameter | Values | Description |
|-----------|--------|-------------|
| `output-format` | `png`, `jpg` | Output image format |
| `width`, `height` | pixels | Resize dimensions |
| `cropwidth`, `cropheight` | pixels | Crop region size |
| `cropx`, `cropy` | pixels | Crop region origin |
| `rotate` | `0`, `90`, `180`, `270` | Rotation angle |
| `size` | `IconSmall`, `IconMedium`, `Preview` | Predefined size presets |

Transformation states: `rendered`, `rendering`, `unrendered`, `unable to render`

#### AI File States

When intelligence is enabled, each file progresses through AI processing states (visible in node details `ai.state`):
`disabled` → `pending` → `inprogress` → `ready` (or `failed`)

#### AI Chat Parameters

| Parameter | Constraint |
|-----------|-----------|
| `privacy` / `visibility` | `private`, `public` (default: `private`; **`public` is currently disabled platform-wide** -- creating a public chat returns `403`; check `capabilities.can_publish_agent_chat` (currently `false`) first) |
| `name` | Max 100 characters |
| `question` | 1–32,000 characters |
| `references` / `content_parts` / `subjects` / `uploads` | File/folder reference items — max 20 files, 200 MB, 100 references total |

#### Quick Share Constraints

- Single file only, max file size is plan-dependent (query `/upload/limits/` for exact value)
- Default expiration: 3 hours, maximum: 24 hours
- Auto-deleted on expiration, public access (no auth)
- Can update expiration but not beyond the original 24-hour window

#### Download Tokens

Use `GET .../storage/{node_id}/requestread/` to generate a temporary auth-free download token. Pass the returned
`token` as a query parameter: `GET .../storage/{node_id}/read/?token={token}`. This is useful for opening files in
browser tabs without sending Authorization headers.

**MCP agents** have additional download options: use the `download://` resource templates for direct content retrieval
(up to 50 MB), or the `/file/` HTTP pass-through endpoint for streaming larger files. See the "MCP Tool Architecture"
section for details.

#### Unit Calculations

- **Storage** is measured in GibiBytes (1024^3 bytes)
- **Bandwidth** is measured in GigaBytes (1000^3 bytes)

---

## Plans & Credits

New organizations — created by humans or agents alike — choose a **paid plan (Starter, Business, or Growth)** to get
started. A newly created organization must select a paid plan before it can do work; until then it is in an
upgrade-only state and resource-consuming endpoints return HTTP 402. There is no free-to-start path for new orgs.

### What Credits Cover

All platform activity consumes credits from the org's monthly allowance:

| Resource                | Cost                    |
|-------------------------|-------------------------|
| Storage                 | 150 credits/GB          |
| Bandwidth               | 400 credits/GB          |
| AI chat tokens          | 1 credit per 100 tokens |
| Document pages ingested | 10 credits/page         |
| Video ingested          | 5 credits/second        |
| Audio ingested          | 0.5 credits/second      |
| Images ingested         | 5 credits/image         |
| File conversions        | 25 credits/conversion   |

When credits run out, the org enters a reduced-capability state — file storage and access continue to work, but
credit-consuming operations (AI chat, file ingestion, bandwidth-heavy downloads) are limited until the credits reset or
the plan is upgraded. The org is never deleted.

**Detecting an upgrade-only or credit-exhausted org:** API calls return HTTP 402 with one of these error codes:

| Error Code | Description | Meaning |
|------------|-------------|---------|
| 1688 | Subscription Required | Org has no active paid plan (a new org that hasn't selected one yet), or its credits are exhausted |
| 1696 | Credit Limit Exceeded | Credit limit exceeded (error message includes credits used and credit limit) |

You can also check proactively: the `subscriber` field in org details (`GET /current/org/{org_id}/details/`) returns
`false` when the org has no active paid plan or is out of credits. Admins can check detailed usage via
`GET /current/org/{org_id}/billing/usage/limits/credits/`.

**When you hit the limit:** Upgrade the org's plan, or select one if the org is still in the upgrade-only state. Direct
the owner to `https://go.fast.io/onboarding` or use the billing API. Higher-tier plans include a larger monthly credit
allowance and expanded limits.

### Plan Entitlement Matrix

Starter, Business, and Growth are the paid plans new organizations choose:

| Feature                  | Starter | Business  | Growth    |
|--------------------------|---------|-----------|-----------|
| Monthly included credits | 300,000 | 1,200,000 | 4,500,000 |
| Storage                  | 1 TB    | 10 TB     | 50 TB     |
| Included seats           | 1       | 20        | 50        |
| Max file size            | 25 GB   | 50 GB     | 100 GB    |
| Workspaces               | 10      | 100       | Unlimited |
| Shares                   | 100     | 1,000     | 1,000     |

---

## Common Workflows

### Deliver a Report to a Client

1. Upload report PDF to workspace
2. Create a Send share with password protection and 30-day expiration
3. Share the link with the client
4. Client sees a branded page, previews the PDF inline, downloads if needed
5. You get a notification when they access it

### Collect Documents From a User

1. Create a Receive share ("Upload your tax documents here")
2. Share the link
3. User uploads files through a clean, branded interface
4. Files appear in your workspace, auto-indexed by AI (if intelligence is on)
5. Ask the AI: "Are all required forms present?"

### Build a Knowledge Base

1. Create a workspace **with intelligence enabled** (this is one of the workflows that justifies the ingestion cost)
2. Upload all reference documents
3. AI auto-indexes and summarizes everything on upload
4. Use **semantic search** (`storage/search`) for fast, low-cost retrieval — find relevant document chunks by meaning without an LLM round-trip. Best for lookup, recall, and memory workflows
5. Use **AI chat** when you need the LLM to synthesize, analyze, or summarize across documents — returns a natural language answer with citations
6. Combine both: search first to find relevant content cheaply, then chat only when synthesis is needed

### Set Up an Agentic Team Workspace

1. Create org + select a paid plan + workspace + folder structure
2. Upload templates and reference docs
3. Invite other agents and human team members to the org or workspace
4. Create shares for client deliverables (Send) and intake (Receive)
5. Configure branding, passwords, expiration
6. Humans and agents collaborate as members of the shared org — everyone sees the same fully configured platform

### Collaborative Review Cycle (Exchange Share)

1. Create an Exchange share ("Review these designs and upload your feedback")
2. Upload draft files for the recipient
3. Share the link — recipient can both download your files and upload theirs
4. Comments and annotations on files enable inline feedback
5. AI summarizes what changed between rounds (if intelligence is on)

### Extract Structured Metadata From Documents

1. Create a workspace **with intelligence enabled** (metadata extraction requires ingestion — budget for ingestion costs)
2. Upload files to the workspace — no schema to define first
3. Metadata is extracted automatically during ingestion; every file is a candidate, and each one also
   receives a broad `category` and `sub_category` so it can be narrowed by document kind
4. For a file already in the workspace, use `POST .../storage/{node_id}/metadata/extract/` for one file
   or `POST .../storage/{node_id}/metadata/extract-all/` for a folder
5. Discover what you can filter on with `GET .../metadata/fields/` (field names) and the same endpoint
   with `field=` (that field's observed values) — you do not have to guess
6. Query with `filters` on `GET .../storage/search/`, or save the predicate as a filter via
   `POST .../metadata/filters/` and run it with its `/nodes/` endpoint

### One-Off Document Analysis (No Intelligence Needed)

1. Create a workspace (intelligence off is fine)
2. Upload the files you want to analyze
3. Create an AI chat and attach the specific files directly (up to 20 files)
4. Ask questions — AI reads the attachments and responds with citations
5. No persistent indexing, no credit cost for ingestion

### Choose Between Portal and Shared Folder

**Use a Portal (independent storage) when:**

- Delivering final, immutable outputs (reports, compliance packages)
- You want a snapshot that won't change if workspace files are updated
- Files are "done" and shouldn't reflect future edits

**Use a Shared Folder (workspace-backed) when:**

- Files are actively being updated (live data feeds, ongoing projects)
- You want zero storage duplication
- Recipients should always see the latest version

### Manage Credit Budget

1. Check current usage: `GET /current/org/{org_id}/billing/usage/limits/credits/`
2. Storage costs 150 credits/GB — a 10 GB workspace costs 1,500 credits/month
3. Document ingestion costs 10 credits/page — a 50-page PDF costs 500 credits
4. Disable intelligence on storage-only workspaces to avoid ingestion costs
5. Use attach-only AI chat (no intelligence needed) for one-off analysis to save credits
6. When credits run low, upgrade the org's plan (Starter, Business, or Growth) for a larger monthly credit allowance

---

## CLI Tool

The `fastio` CLI provides full platform access from the terminal — authentication, file management, AI chat,
and more. It's built in Rust for cross-platform performance and supports macOS, Linux, and Windows.

### Installation

```bash
# NPM (recommended)
npm install -g @vividengine/fastio-cli

# Or run without installing
npx @vividengine/fastio-cli --help

# Shell script
curl -fsSL https://raw.githubusercontent.com/MediaFire/fastio_cli/main/install.sh | sh

# From source (Rust 1.85+)
cargo install --path .
```

Pre-compiled binaries are also available on the [GitHub releases page](https://github.com/MediaFire/fastio_cli/releases).

### Authentication

The CLI checks credentials in this priority order:

1. `--token` flag (one-off bearer token)
2. `FASTIO_TOKEN` environment variable
3. `FASTIO_API_KEY` environment variable
4. Stored profile credentials (default or named)

**Browser login (recommended):**

```bash
fastio auth login          # Opens browser for secure PKCE OAuth flow
fastio auth status         # Verify authentication
fastio auth logout         # Sign out
```

**Email/password:**

```bash
fastio auth login --email user@example.com --password ****
```

**API key:**

```bash
fastio auth api-key create --name "CI pipeline"
export FASTIO_API_KEY=your-key-here
```

**2FA:**

```bash
fastio auth 2fa status
fastio auth 2fa setup --channel totp
fastio auth 2fa verify <code>
```

### Quick Start

```bash
fastio auth login
fastio org list
fastio workspace create --org <org_id> "My Workspace"
fastio upload file --workspace <workspace_id> ./document.pdf
fastio download file --workspace <workspace_id> <node_id> --output ./downloads/
fastio ai chat --workspace <workspace_id> "What files do I have?"
```

### Command Reference

| Domain | Command | Operations |
|--------|---------|------------|
| **Auth & User** | `auth` | Login, logout, 2FA, API keys, OAuth sessions |
| | `user` | Profile, search, assets, invitations |
| | `configure` | CLI profiles and settings |
| **Orgs & Workspaces** | `org` | Create/read/update/delete, billing, members |
| | `workspace` | Create/read/update/delete, notes, quickshares |
| | `member` | Workspace/share member management |
| | `invitation` | Accept, decline, delete invitations |
| **Files & Storage** | `files` | List, create folders, move, copy, rename, delete, trash, versions, search, lock |
| | `upload` | Chunked uploads with progress, text uploads, URL imports |
| | `download` | Streaming downloads with progress, folder ZIP, batch, quickshare |
| | `lock` | Acquire, check, release file locks |
| **Shares** | `share` | Create/read/update/delete, file management, members, quickshares, password protection |
| | `comment` | Comments, replies, reactions, linking |
| | `event` | Activity events, search, polling |
| | `preview` | File preview URLs and transforms |
| | `asset` | Org/workspace/user asset management |
| **AI** | `ai` | Chat, search, history, message management, summarize |
| **Platform** | `apps` | App listing, details, launching |
| | `import` | Cloud sync providers, identities, sources, jobs |
| | `mcp` | Built-in MCP server for AI agents |
| | `completions` | Shell completion generation |

### Output Formatting

```bash
fastio org list                              # Table (default)
fastio org list --format json                # JSON output
fastio org list --format csv                 # CSV output
fastio org list --fields name,id,description # Filter output fields
```

### Profile Management

The CLI supports multiple named profiles for switching between accounts:

```bash
fastio configure init                  # Initialize configuration
fastio auth login --profile work       # Authenticate a named profile
fastio org list --profile work         # Use a named profile
fastio configure set-default work      # Set default profile
fastio configure list                  # List all profiles
```

Configuration is stored in `~/.fastio/` (`config.json` for settings, `credentials.json` for tokens).

### Global Flags

| Flag | Purpose |
|------|---------|
| `--format json\|table\|csv` | Output format |
| `--fields name,id,...` | Filter output fields |
| `--no-color` | Disable colored output |
| `--quiet` / `-q` | Suppress output |
| `--verbose` / `-v` | Enable debug logging |
| `--profile <name>` | Use named profile |
| `--token <jwt>` | One-off bearer token |
| `--api-base <url>` | Override API base URL |

### Built-in MCP Server

The CLI can run as a local MCP server, enabling AI agents to use Fastio through the Model Context Protocol without
connecting to the hosted MCP server:

```bash
fastio mcp
```

**Claude Desktop configuration:**

```json
{
  "mcpServers": {
    "fastio": {
      "command": "fastio",
      "args": ["mcp"]
    }
  }
}
```

Filter available tools:

```bash
fastio mcp --tools auth,org,workspace,files,upload,download
```

### Shell Completions

```bash
fastio completions bash > ~/.bash_completion.d/fastio
fastio completions zsh > ~/.zfunc/_fastio
fastio completions fish > ~/.config/fish/completions/fastio.fish
fastio completions powershell > _fastio.ps1
```

### Source & License

- **Repository:** [github.com/MediaFire/fastio_cli](https://github.com/MediaFire/fastio_cli)
- **License:** Apache License 2.0

---

## MCP Tool Architecture

The MCP server exposes consolidated domain-specific tools, each covering a domain. Every tool uses
an `action` parameter to select the specific operation — agents don't need to discover hundreds of separate tools, just
a manageable set of tools with clearly named actions.

| Tool         | Domain                          | Example Actions                                                               |
|--------------|---------------------------------|-------------------------------------------------------------------------------|
| `auth`       | Authentication                  | `signin`, `signup`, `set-api-key`, `pkce-login`, `pkce-complete`, `status`, `signout` |
| `org`        | Organizations                   | `list`, `details`, `create`, `update`, `discover-all`                         |
| `workspace`  | Workspaces                      | `list`, `details`, `create`, `update`, `check-name`. (Its legacy `metadata-*` actions are **deprecated forwarding shims** to the `metadata` tool and will be removed next release — use `metadata` instead.) |
| `metadata`   | Workspace field vocabulary and value search | `fields-list`, `search`, `compound-search`, `eligible`, `fields-merge` (destructive and irreversible). 🔴 The **template and saved-view** actions are the ones that are gone — every `template-*`, `view-*`/`views-list`, `nodes-*`, `auto-match`, `preview-match`, `suggest-fields` and `extract-all`. A stale client may still list them; calling one returns either `9992` (deleted, no longer routes) or `410 Gone` (retired in place; see section 13 for the per-path `error.code`). Per-file metadata values live on the `storage` tool. |
| `share`      | Shares                          | `list`, `create`, `update`, `delete`, `quickshare-create`                     |
| `storage`    | Files, folders, locks, previews, search (keyword + semantic when intelligence is enabled; accepts `files_scope`/`folders_scope` for scoped semantic search) | `list`, `details`, `search`, `create-folder`, `create-note`, `move`, `delete`, `lock-acquire`, `lock-status`, `lock-release`, `preview-url` (returns constructed `preview_url`), `preview-transform` (returns constructed `transform_url`) |
| `upload`     | File uploads                    | `create-session`, `stage-blob`, `chunk`, `finalize`, `text-file`, `web-import` |
| `download`   | Downloads                       | `file-url`, `zip-url`, `quickshare-details`                                   |
| `ai`         | AI chat (defaults to the entire workspace — attach nothing to search all indexed documents). Attach file/folder reference items to ground answers in specific files or folders. | `chat-create`, `message-send`, `message-read`, `chat-list` |
| `member`     | Members                         | `add`, `update`, `remove`, `details`                                          |
| `invitation` | Invitations                     | `list`, `send`, `revoke`, `accept-all`                                        |
| `asset`      | Branding assets                 | `types`, `list`, `upload`, `delete`                                           |
| `comment`    | Comments                        | `list`, `create`, `details`, `delete`                                         |
| `event`      | Events & audit                  | `search`, `details`, `summarize`, `activity-poll`                             |
| `intent`     | Agent coordination — workspace-scoped, short-lived declarations of what an agent is working on, so peers see a collision before it happens. Workspace-only (no share variant). `fill` is also the heartbeat, and is compare-and-set: send back the `version` you last read. | `allocate`, `fill`, `browse`, `expand`, `release`                             |
| `user`       | Account mgmt                    | `me`, `update`, `invitation-list`, `allowed`                                  |
| `how-to` | Built-in product help — ask a natural-language "how do I…" question about Fastio and get a grounded answer (or a clarifying question) back. **Top-level, user-authenticated: no org required, no org membership or plan feature required — open to any authenticated caller, free (no entity is charged), bounded by a per-user rate limit.** `ask` takes a `question` (and optional `context`, `surface`). `surface` accepts `mcp` (MCP-tool phrasing) or `code` (code-mode execute-proxy phrasing, steps written as execute-proxy calls, e.g. `fastio.post('/current/<path>/', ...)` for the form-encoded default); omit for default REST-API phrasing. | `ask` |

> **Note on tool naming:** the tools above are listed without a vendor prefix (`auth`, `share`, `ai`, `how-to`, …),
> matching the names the MCP server advertises. The built-in help tool shipped as `how-to` (prefix-free, hyphenated —
> not `fastio_howto`). As a general practice, agents should discover the exact tool names from the MCP `tools/list`
> at connection time rather than hardcoding them.

### `web_url` in Tool Responses — Use It Instead of Building URLs

All entity-returning tool responses include a `web_url` field containing a ready-to-use link to the resource in the Fastio web UI.
**Use `web_url` directly** instead of constructing URLs manually from API response fields. This avoids errors from
slug generation, subdomain routing, or parameter formatting.

Tools that return `web_url`:

| Tool | Actions |
|------|---------|
| `org` | `list`, `details`, `create`, `update`, `public-details`, `list-workspaces`, `list-shares`, `create-workspace`, `discover-all`, `discover-available`, `discover-external` |
| `workspace` | `list`, `details`, `update`, `available`, `list-shares`, `create-note`, `update-note`, `quickshare-get`, `quickshares-list` |
| `share` | `list`, `details`, `create`, `update`, `public-details`, `available` |
| `storage` | `list`, `details`, `search`, `trash-list`, `create-folder`, `copy`, `move`, `rename`, `restore`, `add-file`, `version-list`, `version-restore`, `preview-url`, `preview-transform` |
| `ai` | `chat-create`, `chat-details`, `chat-list` |
| `upload` | `text-file`, `finalize` |
| `download` | `file-url`, `quickshare-details` |

When presenting links to users, always use `web_url` from tool responses. Never construct URLs manually.

**Resources** available via `resources/read`:
- `skill://guide` — full tool documentation with parameters and examples
- `session://status` — current authentication state
- `download://workspace/{workspace_id}/{node_id}` — download a workspace file (returns base64 content up to 50 MB)
- `download://share/{share_id}/{node_id}` — download a share file (returns base64 content up to 50 MB)
- `download://quickshare/{quickshare_id}` — download a quickshare file (public, no auth required, up to 50 MB)

The `download://` resource templates provide direct file content retrieval via the MCP `resources/read` protocol.
Files up to 50 MB are returned inline as base64 blobs. Larger files return a fallback message directing to the HTTP
pass-through endpoint (see below). The `download` tool's `file-url` and `quickshare-details` actions include a
`resource_uri` field in their response that points to the corresponding `download://` resource URI.

**HTTP pass-through endpoint** for file downloads:

The MCP server exposes a `/file/` HTTP endpoint that streams file content directly with proper `Content-Type`,
`Content-Length`, and `Content-Disposition` headers — useful for large files that exceed the 50 MB MCP resource limit
or when streaming is preferred over base64 encoding.

| Path | Auth | Description |
|------|------|-------------|
| `GET /file/workspace/{workspace_id}/{node_id}` | `Mcp-Session-Id` header required | Stream a workspace file |
| `GET /file/share/{share_id}/{node_id}` | `Mcp-Session-Id` header required | Stream a share file |
| `GET /file/quickshare/{quickshare_id}` | None (public) | Stream a quickshare file |

For workspace and share downloads, include the `Mcp-Session-Id` header from your active MCP session. The server uses
the session's auth token to fetch the file and streams it back.

**Query parameters:**
- `?error=html` — returns error pages as HTML instead of JSON (useful for browser-facing links)

**Size limits:** The `download://` resource templates return file content inline (base64) for files up to **50 MB**.
Larger files return a fallback message directing to the `/file/` HTTP pass-through endpoint.

**`web_url` in download responses:** The `download` tool's `file-url` and `quickshare-details` actions include both
a `resource_uri` (for MCP resource reads) and a `web_url` (for browser-facing links) in their responses.

**Resource completion** — The workspace and share `download://` resource templates support MCP `completion/complete`
for tab-completion of IDs. Agents can use this to discover valid workspace IDs, share IDs, and node IDs without
needing to call separate list actions first.

### Tool Annotations — Safety & Side Effects

All tools include explicit MCP annotations (`title`, `readOnlyHint`, `destructiveHint`, `idempotentHint`,
`openWorldHint`) so agents and agent frameworks can make informed decisions about confirmation prompts, retries, and
automated execution.

**Read-only tools** (safe, no confirmation needed, `idempotentHint: true`):
- `download`, `event` — these tools only read data, never modify state, and are safe to retry

**Non-destructive mutation tools** (create or update, no delete actions):
- `upload`, `invitation` — these tools create or modify resources but cannot delete them

**Destructive tools** (include delete, purge, or close actions — require user confirmation):
- `auth`, `user`, `org`, `workspace`, `share`, `storage`, `ai`, `comment`, `member`, `asset`, `intent` — these tools have at
  least one action that permanently removes or closes a resource. Agent frameworks should prompt for confirmation before
  executing destructive actions.

**Discovery tools** (`openWorldHint: true`):
- `org`, `user`, `workspace`, `share`, `storage` — these tools can discover resources beyond the agent's current
  context. Agents may encounter resources they haven't seen before in list/search results.

**Credit-consuming operations** to be aware of:
- AI chat: 1 credit per 100 tokens
- File uploads: storage credits (150 credits/GB)
- Downloads: bandwidth credits (400 credits/GB)
- Document ingestion: 10 credits/page (when intelligence is enabled) — this can be the largest credit consumer. A 100-page document costs 1,000 credits to ingest.

### Code Mode — Streamlined Tools for Headless Agents

The MCP server (v2026.02.102+) detects the connecting client and serves one of two tool sets:

**Named Mode** (Claude Desktop, Cline, unknown clients): All core tools listed above plus app-specific widget tools — the full interactive experience with action-based routing across every
domain.

**Code Mode** (Claude Code, Cursor, Continue): A streamlined set of tools optimized for programmatic workflows:

| Tool       | Purpose                                                                                     |
|------------|---------------------------------------------------------------------------------------------|
| `auth`     | Authentication — same as Named Mode (`signin`, `signup`, `set-api-key`, `pkce-login`, etc.) |
| `upload`   | File uploads — same as Named Mode (`create-session`, `chunk`, `finalize`, `text-file`, etc.)|
| `search`   | Keyword/tag search across the public API endpoint catalog                                   |
| `execute`  | Make authenticated API calls to Fastio (structured method/path/body/params)                |

#### `search` Tool

Discovers API endpoints by keyword and tag. Returns scored matches with method, path, summary, parameters, and relevant
concept docs (pagination, error codes, etc.).

**Parameters:**

| Parameter          | Type    | Required | Description                                                    |
|--------------------|---------|----------|----------------------------------------------------------------|
| `query`            | string  | Yes      | Keyword search query (e.g., "list workspaces", "upload file")  |
| `tag`              | string  | No       | Filter results by API tag (e.g., "workspace", "storage", "ai") |
| `include_concepts` | boolean | No       | Include related concept docs (pagination, error codes, etc.)   |
| `max_results`      | number  | No       | Maximum number of endpoint matches to return                   |

#### `execute` Tool

Makes authenticated API calls to the Fastio API using structured parameters. The tool automatically injects the
session token, unwraps the API response envelope, and extracts errors — agents receive clean response data without
boilerplate. Non-JSON responses (text, binary) are handled gracefully.

**Parameters:**

| Parameter    | Type   | Required | Description                                                        |
|--------------|--------|----------|--------------------------------------------------------------------|
| `method`     | enum   | Yes      | HTTP method: `get`, `post`, `postJson`, `delete`, `put`, `putJson`, `patch`, `patchJson` |
| `path`       | string | Yes      | API endpoint path (e.g., `/current/org/{id}/list/workspaces/`)     |
| `body`       | object | No       | Request body — form-encoded for `post`/`put`/`patch` (nested values are JSON-encoded for you), JSON for `postJson`/`putJson`/`patchJson`; not accepted on `get`/`delete` |
| `params`     | object | No       | Query string parameters                                            |
| `timeout_ms` | number | No       | Request timeout in milliseconds                                    |

**Special paths:**

| Path                           | Purpose                                           |
|--------------------------------|---------------------------------------------------|
| `/readnote/`                   | Read note content in code mode                    |
| `download://{file_id}`         | MCP resource path for reading file content        |

#### Code Mode Workflow Pattern

Code Mode agents follow a **search → review → execute → iterate** loop:

1. **Search** — use the `search` tool to discover relevant API endpoints by keyword or tag
2. **Review** — examine the returned endpoint details (method, path, parameters, summary)
3. **Execute** — call the endpoint with structured parameters (`method`, `path`, `body`, `params`)
4. **Iterate** — refine based on results, search for additional endpoints as needed

This pattern replaces the need for many individually named tools. Agents discover endpoints dynamically via search and
call them with structured parameters via execute, without needing pre-registered tool definitions for each operation.

**Example — list workspaces in an org:**

```
// Search: search tool with query "list workspaces"
// → returns: GET /current/org/{id}/list/workspaces/

// Execute:
execute method="get" path="/current/org/{org_id}/list/workspaces/"
```

### Response Hints — Guided Agent Workflows

Tool responses include structured hints that help agents navigate multi-step workflows, handle errors gracefully, and
understand resource state. Agents should read and act on these hints rather than guessing the next step.

**`_next` — Suggested next actions:**

Successful tool responses include a `_next` array of contextual next-step suggestions using exact tool names, action
names, and IDs from the response. Agents should follow these hints instead of guessing the next step or consulting
docs. Present on many actions across the tool set.

Example: after `storage` action `list`, `_next` might suggest `["storage folder-details {node_id}",
"download file-url {node_id}", "ai chat"]` with actual IDs from the response populated in the suggestions.

**`_warnings` — Destructive or gated action warnings:**

Actions that are destructive, irreversible, or have significant side effects include `_warnings` strings in their
response. Agents should read these warnings before proceeding and present them to the user when appropriate. Present on
the following actions:
- `storage`: purge, bulk copy/move/delete/restore (partial failure warnings)
- `workspace`: update (intelligence disable), archive, delete
- `org`: close, billing-create
- `share`: delete, archive, update (type change)
- `ai`: chat-delete
- `download`: file-url (token expiry), zip-url
- `upload`: stage-blob (5-minute expiry)

**`_recovery` — Error recovery hints:**

Error responses (`isError: true`) include `_recovery` hints as actionable bullet points appended to the error text.
Hints are matched by HTTP status code and error message patterns, guiding agents toward the
correct resolution. All errors also include `(during: <tool> <action>)` so agents know exactly which operation failed.

| Status | Recovery hint |
|--------|---------------|
| 400    | Bad request — check required parameters and value formats |
| 401    | Re-authenticate using `auth` action `signin` or `pkce-login` |
| 402    | Credits exhausted — check with `org` action `limits` |
| 403    | Permission denied — check role with `org` action `details` |
| 404    | Resource not found — verify the ID is correct |
| 405    | Method not allowed — check the action name is valid for this tool |
| 409    | Conflict — resource may already exist |
| 413    | Payload too large — reduce file size or use chunked upload |
| 422    | Validation failed — check field values against documented constraints |
| 429    | Rate limited — wait 2–4 seconds, retry with exponential backoff |

Error message pattern matching provides additional context-specific recovery steps (e.g., "email not verified" →
use `auth` action `email-verify`; "workspace not found" → check workspace ID with `workspace` action `list`).

**`ai_capabilities` — AI mode availability:**

Included in `workspace` action `details` responses. Shows the available AI modes for the workspace:
- **Intelligence ON:** file and folder references (full RAG with indexed search — attach folder references to ground answers in a folder's indexed files), plus the `search` action for semantic search (vector-based document chunk retrieval with relevance scores — no LLM round-trip, returns ranked snippets). Use search for fast retrieval/lookup; use chat for synthesis/analysis.
- **Intelligence OFF:** direct file references only (max 20 files, 200 MB total). Semantic search is not available.

**`_ai_state_legend` — File AI processing state:**

Included in `storage` action `list` and `search` responses when files have AI state. Describes the possible states:
- `ready` — file is indexed and available for AI queries
- `pending` — file is queued for AI processing
- `inprogress` — file is currently being processed
- `disabled` — AI processing is disabled for this file
- `failed` — AI processing failed for this file

**`_context` — Contextual metadata:**

Certain responses include `_context` with additional metadata specific to the operation. For example, `comment` action
`add` responses include `anchor_formats` describing supported anchor types for positioning comments on files (image
regions, video/audio timestamps, PDF pages).

---

## URL Structure & Link Construction

Fastio uses subdomain-based routing. Organization domains become subdomains, and every resource (workspace, folder,
file, share) has a URL-safe identifier from API responses that you use to build links.

### How Org Domains Become Subdomains

When you create an organization, you choose a `domain` (2-63 characters, lowercase alphanumeric and hyphens). This
becomes the subdomain for all org URLs:

Organization domain: `"acme"` → All org URLs live at: `https://acme.fast.io/...`

The base domain `go.fast.io` is used for routes that don't require org context (public shares, auth).

> **Prefer `web_url`.** The URL patterns below are reference material. In practice, always use the `web_url` field from tool responses — it handles subdomain routing, slug generation, and edge cases automatically. Only fall back to manual construction when `web_url` is absent (e.g., share-context storage operations).

### Building URLs From API Responses

Every URL parameter comes from a field in the API response. You never need to generate or guess identifiers — use the
values the API gives you.

| URL Parameter | API Response Field            | Format                                | Example                               |
|---------------|-------------------------------|---------------------------------------|---------------------------------------|
| Subdomain     | `organization.domain`         | User-chosen slug                      | `acme`                                |
| Workspace     | `workspace.folder_name`       | URL-safe slug                         | `q4-planning`                         |
| Folder        | `folder.id` (storage node ID) | Opaque ID, or `root` / `trash`        | `2rii2hzajpc2s3kce3itd2z5esygv`       |
| File          | `file.id` (storage node ID)   | Opaque ID                             | `2xzvfaq3slqwa54qtezi66rrcly6w`       |
| Share         | `share.custom_name`           | Server-generated identifier           | `abc123xyz`                           |
| QuickShare    | `quickshare.id`               | Server-generated identifier           | `qs-abc123xyz`                        |

### Deep Links Into Workspaces

These URLs require the user to be logged in and a member of the org. Use the org's `domain` as the subdomain and the
workspace's `folder_name` as the workspace identifier.

| Link Type          | URL Pattern                                                                  |
|--------------------|-----------------------------------------------------------------------------|
| Workspace root     | `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/storage/root` |
| Specific folder    | `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/storage/{folder.id}` |
| File preview       | `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/preview/{file.id}` |
| AI chat            | `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/storage/root?chat={chat_id}` |
| Note (in workspace)| `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/storage/root?note={note_id}` |
| Note (preview)     | `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/preview/{note_id}` |
| Browse workspaces  | `https://{org.domain}.fast.io/browse-workspaces`                            |

#### Workspace View Query Parameters

Append query parameters to workspace storage URLs to control the initial view mode and info panel tab:

| Parameter | Values                                              | Effect                              |
|-----------|-----------------------------------------------------|-------------------------------------|
| `view`    | `list`, `grid`, `metadata`                          | Sets file list layout mode          |
| `tab`     | `info`, `metadata`, `comments`, `activity`, `versions` | Opens info panel on specified tab |

Parameters are applied on page load only — they set the initial view state but are not updated during in-app navigation.

**Examples:**
- `https://acme.fast.io/workspace/q4-planning/storage/root?view=metadata`
- `https://acme.fast.io/workspace/q4-planning/storage/root?tab=metadata`
- `https://acme.fast.io/workspace/q4-planning/storage/root?view=metadata&tab=info`

**Examples:**
- `https://acme.fast.io/workspace/q4-planning/storage/root`
- `https://acme.fast.io/workspace/q4-planning/storage/2rii2hzajpc2s3kce3itd2z5esygv`
- `https://acme.fast.io/workspace/q4-planning/preview/2xzvfaq3slqwa54qtezi66rrcly6w`

### Shareable Links (No Auth Required)

These are the URLs you send to humans. Access depends on share settings, not authentication.

| Link Type              | URL Pattern                                                                        |
|------------------------|-----------------------------------------------------------------------------------|
| Public share           | `https://go.fast.io/shared/{share.custom_name}/{title-slug}`                      |
| Org-branded share      | `https://{org.domain}.fast.io/shared/{share.custom_name}/{title-slug}`            |
| File within a share    | `https://go.fast.io/shared/{share.custom_name}/{title-slug}/preview/{file.id}`    |
| QuickShare             | `https://go.fast.io/quickshare/{quickshare.id}`                                   |

The `{title-slug}` is the share title converted to a URL slug (lowercase, spaces to hyphens, special chars removed).
It's optional — routing works with just the `custom_name` — but improves link readability.

**Examples:**
- `https://go.fast.io/shared/abc123xyz/q4-financial-report`
- `https://acme.fast.io/shared/abc123xyz/q4-financial-report`
- `https://go.fast.io/shared/abc123xyz/q4-financial-report/preview/2xzvfaq3slqwa54qtezi66rrcly6w`
- `https://go.fast.io/quickshare/qs-abc123xyz`

### Share Management Links (For Owners/Admins)

| Link Type                    | URL Pattern                                                                          |
|------------------------------|--------------------------------------------------------------------------------------|
| Edit share (from workspace)  | `https://{org.domain}.fast.io/workspace/{workspace.folder_name}/share/{share.custom_name}` |
| Edit share (direct)          | `https://{org.domain}.fast.io/share/{share.custom_name}`                             |

### Settings & Account Links

| Link Type   | URL Pattern                                                                          |
|-------------|--------------------------------------------------------------------------------------|
| Org settings| `https://{org.domain}.fast.io/settings`                                              |
| Billing     | `https://{org.domain}.fast.io/settings/billing`                                      |
| Onboarding  | `https://go.fast.io/onboarding` or `https://go.fast.io/onboarding?orgId={org.id}&orgDomain={org.domain}` |

### Typical Agent Flow: Create and Link

1. **Create org** → API returns `org.domain` (e.g., `"acme"`)
2. **Select a paid plan** → activates the org so it can do work (Starter, Business, or Growth)
3. **Create workspace** → API returns `workspace.folder_name` (e.g., `"client-docs"`)
4. **Upload files to folder** → API returns `file.id` for each file
5. **Create share from folder** → API returns `share.custom_name`
6. **Build links for the human:**
   - Workspace link: `https://acme.fast.io/workspace/client-docs/storage/root`
   - Share link: `https://go.fast.io/shared/{custom_name}/client-docs`
   - File link: `https://go.fast.io/shared/{custom_name}/client-docs/preview/{file.id}`

---

## Signing / E-Signature

The platform ships a native e-signature surface so an agent can collect legally enforceable signatures on PDFs without
leaving the workspace. A SignEnvelope is an audit-archive Profile holding up to 20 PDFs sent to one or more recipients;
the internal PAdES-LT engine produces a long-term-validation cryptographic signature on every completed document and
the envelope's audit certificate captures the full chain of evidence (consent acceptance, OTP authentication where
required, per-recipient sign events, document hashes).

> **Full reference:** [https://api.fast.io/current/llms/signing/](https://api.fast.io/current/llms/signing/)
> **Public HTML docs:** [https://api.fast.io/current/docs/signing/](https://api.fast.io/current/docs/signing/)

### What an Agent Gets From Signing

| Capability | What It Solves |
|------------|----------------|
| **Native PDF signing** | No external e-sign account required for the default path. The platform's internal PAdES-LT engine embeds a long-term-validation signature directly in the completed PDF; the signature is verifiable offline by any standard PDF reader. |
| **Audit certificate** | Every completed envelope ships with a downloadable audit certificate (JSON evidence record) capturing the per-recipient consent, OTP flow, sign timestamps, and document hashes. The chain underneath it is hash-linked and HMAC-signed. |
| **Recipient OTP** | Configure `auth_method=email_otp` or `sms_otp` on a recipient and the signer surface gates the `/sign` action behind a 6-digit OTP. Code issuance and verification are independently throttled per `(envelope, recipient)` so the brute-force surface stays bounded. |
| **Sequential or parallel routing** | Recipients carry a 1-based `routing_order`. Distinct numbers fire sequentially; identical numbers fire in parallel. The platform activates the first slot on `/send/` and progresses through the slots as each recipient signs. |
| **Sign templates** | Capture a reusable signing configuration (recipient slots, document slots, field placements, policy) as a `SignTemplate` (`sa…` OpaqueId). Instantiate the template to produce a draft envelope with concrete bindings applied. |
| **First-view billing** | Credits are reserved at `/send/` and the first-view charge fires the first time any recipient opens the `/view` landing. Subsequent views by the same or different recipients don't re-charge — the meter is once-per-envelope. Voiding doesn't refund. |

### When to Reach for Signing

- An agent has produced a document that needs a human signature before it can ship.
- A contract / SOW / NDA / consent form needs to be collected and archived with a verifiable audit trail.
- A multi-party agreement needs sequential or parallel signing with per-recipient OTP gating.

If none of the above apply, an agent can still ship a PDF as a regular file via storage or a share — but the audit
chain and OTP gate only exist on the SignEnvelope surface.

### Concept Map (for AI agents)

| Concept | Type | Description |
|---------|------|-------------|
| **SignEnvelope** | profile (19-digit id) | The audit-archive entity. Parented to a Workspace. Carries lifecycle state (`draft` / `sent` / `in_progress` / `completed` / `declined` / `voided` / `expired` / `failed`), policy, and timestamps. |
| **SignTemplate** | OpaqueId (`sa…` prefix, 30-char) | A reusable signing configuration capturing recipient slots, document slots, field placements, and policy. Instantiate to produce a draft envelope. Soft-deleted (tombstoned), never purged. |
| **Document** | OpaqueId | One PDF inside the envelope. Up to 20 per envelope. Carries `source_node_id` / `source_version_id` (the storage node it was copied from), `signed_pdf_node_id` (set when signing completes), `source_sha256`, `completed_sha256`, `display_order`, `signed_at`. |
| **Recipient** | OpaqueId | One signer / cc / viewer / approver / certified-recipient on the envelope. Carries `role`, `routing_order`, `auth_method` (`none` / `email_otp` / `sms_otp`), and per-recipient lifecycle timestamps. Status flows `pending` → `sent` → `viewed` → `authenticated` → `signing_in_progress` → `signed` (with `declined` / `expired` / `voided` / `failed` terminal). |
| **Field** | OpaqueId | A field placement on a `(document_id, page)`. Normalized `0..1` coordinates. Type is one of `signature` / `initial` / `date` / `text` / `checkbox`. Belongs to exactly one recipient. |
| **Signer token** | compact JWT | Short-lived path-token JWT bound to a `(envelope_id, recipient_id)`. The recipient's signing link is `/sign_envelopes/signer/{token}/view/`. The token is consumed (single-use) on the state-changing actions (`/sign`, `/decline`, OTP-verify). |
| **Audit certificate** | OpaqueId (node) | The per-envelope audit certificate (JSON evidence record), generated when the envelope reaches a terminal state (completed, voided, or declined). `audit_certificate_node_id` on the envelope resource goes non-null when the certificate is in place; both the owner `/audit/download/` and signer `/sign_envelopes/{token}/audit/download/` endpoints stream the JSON bytes directly (no read-token round-trip). |
| **Activity events** | event type | `sign_envelope_drafted`, `sign_envelope_sent`, `sign_envelope_voided`, `sign_envelope_viewed`, `sign_envelope_recipient_signed`, `sign_envelope_recipient_declined`, `sign_envelope_document_signed`, `sign_envelope_completed`, `sign_envelope_expired`. Visible through `/events/search/` and outbound webhook subscriptions. |

**ID format note:** Envelope id is **19-digit numeric** (profile id format). Document / recipient / field / node ids
are **OpaqueIds** in hyphenated form (e.g. `2ltsu-q4mja-cuv7p-gc5yd-lxnsj-wee4`). Sign template ids are 30-character
`sa`-prefixed OpaqueIds. Signer tokens are compact JWTs and must never be persisted longer than the envelope's lifetime.

### Auth Model

| Endpoint family | Auth |
|------------------|------|
| Sender / admin (workspace-parented) | `Authorization: Bearer {api_key}` (JWT / OAuth / API key) |
| Signer surface | signer session token carried in the URL path — no session required |
| Provider webhook receiver | Provider's HMAC signature header — no Fastio session |

Workspace **view** is required for read endpoints; workspace **admin** for the
mutating endpoints (`/send`, `/void`, document downloads on a signed envelope, audit download). Signing availability
depends on your organization's plan and enabled features.

### Pattern Cookbook

Every signing flow an agent typically runs reduces to one of these recipes. All examples assume an API key in
`{api_key}` and a workspace id in `{workspace_id}`.

#### Pattern 1: Send a one-recipient envelope, wait for completion

```bash
# 1. Create the draft envelope.
curl -X POST "https://api.fast.io/current/workspace/{workspace_id}/sign_envelopes/create/" \
  -H "Authorization: Bearer {api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "expires_at": "2026-06-15 14:30:00 UTC",
    "policy_json": {"auth_method": "email_otp"},
    "documents": [
      {"source_node_id": "{source_node_id}", "source_version_id": "{source_version_id}", "display_order": 0}
    ],
    "recipients": [
      {"email": "signer@example.com", "display_name": "Alex Signer", "role": "signer", "routing_order": 1, "auth_method": "email_otp"}
    ],
    "fields": [
      {"recipient_email": "signer@example.com", "document_index": 0, "page": 1, "x_norm": 0.5, "y_norm": 0.8, "w_norm": 0.2, "h_norm": 0.05, "type": "signature", "required": true}
    ]
  }'
# -> 200 OK { "sign_envelope": { "id": "{envelope_id}", "envelope_status": "draft", ... } }

# 2. Send the envelope. The platform reserves credits, transitions Draft -> Sent, and dispatches notification.
curl -X POST "https://api.fast.io/current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/send/" \
  -H "Authorization: Bearer {api_key}"
# -> 200 OK { "sign_envelope": { "envelope_status": "sent", "sent_at": "...", ... } }

# 3. Poll the envelope until terminal. Drive on `envelope_status`; when it goes to `completed`, the audit certificate
#    is rendered shortly after and `audit_certificate_node_id` goes non-null.
curl -X GET "https://api.fast.io/current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/details/" \
  -H "Authorization: Bearer {api_key}"

# 4. (After completion) Download the signed PDF and the audit certificate.
curl -X GET "https://api.fast.io/current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/documents/{document_id}/signed/download/" \
  -H "Authorization: Bearer {api_key}"
curl -X GET "https://api.fast.io/current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/audit/download/" \
  -H "Authorization: Bearer {api_key}"
```

#### Pattern 2: Catch a void or decline, react accordingly

```bash
# Subscribe to the relevant activity events via an outbound webhook subscription with event types
# `sign_envelope_voided`, `sign_envelope_recipient_declined`, or `sign_envelope_completed`.
#
# When the event lands, look up the envelope's current state via:
curl -X GET "https://api.fast.io/current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/details/" \
  -H "Authorization: Bearer {api_key}"
# -> envelope.envelope_status is one of `voided` / `declined` / `completed` / `expired` / `failed`.
# -> envelope.voided_reason carries the operator's void reason; recipient[].decline_reason carries the signer's reason.
```

### What to Tell the Recipient

The signer surface is path-token authenticated. The notification email the platform sends contains a link of the
form `/sign_envelopes/signer/{token}/view/`. The recipient's flow is:

1. **Click the link** → renders the envelope state, documents, fields, and consent disclosure via `GET /sign_envelopes/signer/{token}/view/`.
2. **Authenticate (OTP recipients only)** → enter the 6-digit code that the platform sent. The verify response carries a new elevated token; the client uses it for the rest of the flow.
3. **Review and accept consent**, **fill in field values**, **click sign** → `POST /sign_envelopes/signer/{token}/sign/` submits the values; the platform queues the async PAdES-LT signing job and returns a polling token.
4. **Wait for completion** → the client polls `/status/` and respects the adaptive `next_poll_seconds` (2s for the first 10s after sign, 5s for the next 60s, 15s thereafter).
5. **Or decline** → `POST /sign_envelopes/signer/{token}/decline/` with an optional reason cascades the envelope to `declined`.

Path tokens are **single-use** on the consume-style actions (`/sign`, `/decline`, OTP verify). The landing token works
multiple times on `/view` and `/status` until it has been consumed by one of the one-shot actions.

### Gotchas

- **Signing availability is plan-dependent.** Check your organization's plan and enabled features to confirm signing access.
- **Voiding doesn't refund.** Credits are consumed at `/send/`; the void path captures a reason and short-circuits
  pending recipients but does not refund. This matches industry convention.
- **A single decline kills the envelope.** Pending recipients in later routing slots never get notified once the
  envelope cascades to `declined`. Plan around this for sequential multi-signer flows.
- **The signed PDF endpoint 404s until the document completes.** Drive on the document's `signed_at` timestamp on the
  envelope resource — when it's non-null, the signed PDF is downloadable.
- **The audit certificate 404s until the envelope completes.** Drive on `audit_certificate_node_id` on the envelope
  resource — it goes non-null when the certificate is ready.
- **OTP throttles are per-(envelope, recipient).** Three OTP issuances per 15 minutes; fifteen verify attempts per
  15 minutes. The verify throttle protects against brute-force across multiple issuances.
- **Path tokens are short-lived.** Don't store the signer token longer than the envelope's lifetime. The OTP-elevation
  path re-mints the token; outside that, treat it as single-use on consume actions.
- **Documents per envelope is capped at 20.** Exceeding it is rejected at create time with `1605 (Invalid Input)`.
- **PATCH only works on draft envelopes.** A Sent / InProgress / terminal envelope is immutable on the sender surface;
  the only transitions are through `/void` and the signer-surface actions.

### See Also

- Full LLM reference: [https://api.fast.io/current/llms/signing/](https://api.fast.io/current/llms/signing/)
- Public HTML docs: [https://api.fast.io/current/docs/signing/](https://api.fast.io/current/docs/signing/)

---

## Per-Workspace Dashboard

The Dashboard API surfaces a ranked, paginated feed of **actionable cards** for each workspace member — pending signatures, @mentions, and file activity — in a single endpoint call. When the workspace plan includes AI features, an AI overlay adds urgency scores (0–100), AI-generated summaries, and suggested actions to each card, and may append cross-item synthesis cards at the end of the feed.

**Key characteristics:**

- **Per-workspace, per-member.** A member sees only the cards relevant to them in that workspace.
- **AI overlay is additive and gracefully degrades.** Cards are always useful without AI; when AI is available, `urgency`, `ai_summary`, and `suggested_action` enrich each card. Synthesis cards (type `synthesis`, source `ai`) only appear when AI is available.
- **Ripley Agent seed.** Every card carries a `ripley_seed` — a pre-populated question plus typed entity subjects — for launching a focused Ripley Agent conversation about that card.
- **Dismiss/snooze is out-of-band.** Dismissing or snoozing a card hides it from the member's view only; the underlying signature is unaffected.
- **Blocking cards surface first.** Cards that gate other participants are ranked ahead of non-blocking ones.

### Card types

| Type | What it represents |
|------|--------------------|
| `signature` | A pending signature on an envelope |
| `mention` | An @-mention of the caller in a comment |
| `file_version` | A new version of a workspace file |
| `file_added` | A new file added to the workspace |
| `synthesis` | An AI-generated cross-item summary (source: `ai`) |

### Recommended agent workflow

```bash
# 1. Fetch the member's dashboard (first page)
GET /current/workspace/{workspace_id}/dashboard/?limit=50
Authorization: Bearer {jwt_token}

# 2. Render each card. For signature cards, use primary_action.endpoint to get the signing link:
POST /current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/my_sign_link/
# → Returns: sign_url (when actionable), is_terminal, blocked_signers, etc.

# 3. Offer dismiss/snooze on dismissible cards. URL-encode card_key (contains ':').
POST /current/workspace/{workspace_id}/dashboard/cards/{card_key}/dismiss/
# Optional body: {"snooze_until": "2026-06-18 09:00:00 UTC"}

# 4. Pass ripley_seed to the Ripley Agent to pre-focus the conversation
# ripley_seed = {"question": "...", "subjects": [{type, id, display_text}]}
```

### Key response fields

| Field | Type | Description |
|-------|------|-------------|
| `card_key` | string | Stable card ID. URL-encode when in a path (contains `:`). |
| `type` | string | Card type (see table above). |
| `source` | string | `event`, `signature`, or `ai`. |
| `blocking` | boolean | True = blocks other participants; ranked first. |
| `due_at` | string or null | UTC due date (`"YYYY-MM-DD HH:MM:SS UTC"`). |
| `ai_summary` | string or null | AI-generated summary. `null` when AI overlay absent. |
| `urgency` | integer or null | AI urgency 0–100. `null` = not evaluated. |
| `suggested_action` | string or null | AI-suggested next action. `null` when AI overlay absent. |
| `ripley_seed` | object | `{question, subjects: [{type, id, display_text}]}` for Ripley Agent. |
| `dismissible` | boolean | Whether the caller can dismiss/snooze this card. |
| `primary_action` | object | `{kind, method, endpoint, payload_template}` — the main action to take. |

### Authentication requirements

- `GET /current/workspace/{workspace_id}/dashboard/` — JWT, workspace View-or-above.
- `POST/DELETE /current/workspace/{workspace_id}/dashboard/cards/{card_key}/dismiss/` — JWT, workspace View-or-above.
- `POST /current/workspace/{workspace_id}/sign_envelopes/{envelope_id}/my_sign_link/` — JWT, **write-scope token** (minting a signing link is a state-changing operation).

> **Full reference:** [https://api.fast.io/current/llms/dashboard/](https://api.fast.io/current/llms/dashboard/)
> **Public HTML docs:** [https://api.fast.io/current/docs/dashboard/](https://api.fast.io/current/docs/dashboard/)

---

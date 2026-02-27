# Multi-Account Switching — Architecture Decision

> This document defines the multi-account switching architecture for the Swoogo AI Assistant. It compares our approach against Notion's MCP implementation, documents the design decisions, and specifies the implementation plan.

---

## 1. The Problem

A user operating across multiple Swoogo accounts needs to switch between them within the AI Assistant without logging out and re-authenticating each time. This is common for agency users, consultants, or admins managing multiple organizations.

Each Swoogo user can have multiple `consumer_key/secret` pairs — one per account they have access to. Each key is globally unique and maps 1:1 to a `(user_id, account_id)` pair in Swoogo PRO's `UserAccount` table (confirmed via source code inspection, UNIQUE index on `consumer_key`).

---

## 2. Notion as Reference Model

Notion's MCP server (`mcp.notion.com/mcp`) is the most mature public multi-tenant MCP implementation.

### How Notion Does It

1. **Notion's MCP server is a middleware.** Like ours, it is a separate service that calls Notion's public API (`api.notion.com`) using OAuth tokens. It does not run inside Notion's core platform — it consumes Notion's API the same way any third-party integration would.
2. **Bot-per-workspace model.** Each OAuth authorization creates a `bot_id` scoped to a specific `workspace_id`. The access token encodes `bot_id + workspace_id + installing_user`.
3. **One connection per workspace.** Notion does **not** support in-session workspace switching. Each workspace requires a separate MCP connection configured as a distinct "Custom Agent" in the MCP client (Claude Desktop, Cursor). A user with 3 workspaces needs 3 separate agent configurations.
4. **The MCP client manages selection**, not the server. The user switches workspaces by switching agents in the client UI. The Notion MCP server itself has no `switch-workspace` tool.
5. **Credential injection at runtime.** Tools never access global token state. Each connection has its own OAuth token scoped to its workspace.

### Why Notion's Model Works (and Its Limitations)

- Notion controls the full stack (API + OAuth + MCP server) under one organization, which gives tight integration despite the architectural separation.
- The OAuth flow includes a consent screen with explicit workspace selection, so each connection knows exactly which workspace its token is scoped to.
- The "Bot" abstraction is a first-class citizen in Notion's API, designed for exactly this delegation pattern.

**Limitation**: The one-connection-per-workspace model means conversational context is lost when switching workspaces. The user must start a new conversation with a different agent. Cross-workspace operations (e.g., "compare data from Account A and Account B") are impossible within a single session.

### How We Compare

Architecturally, the pattern is the same:

```
Notion MCP Server   → calls Notion API   → accesses Notion data
Swoogo AI Assistant → calls Swoogo API   → accesses Swoogo data
```

Both are middleware services between an MCP client (Claude Desktop, Cursor) and a backend API. Both delegate identity and authorization to the platform. Neither replicates the platform's user/account tables.

The differences are in the authorization model and the switching mechanism:

| Aspect | Notion MCP | Swoogo AI Assistant |
|---|---|---|
| **Auth protocol** | OAuth 2.0 full (authorize + consent screen) | client_credentials (no browser redirect) |
| **Token scope** | Per-workspace (Bot token with workspace_id) | Per-user-account (UserAccount credentials) |
| **Multi-account mechanism** | Multiple OAuth consents → multiple bot tokens | Multiple consumer_key/secret pairs |
| **User identity in token** | `bot_id` + `workspace_id` in token payload | Implicit via consumer_key (not in response) |
| **Account selection** | OAuth consent screen (user picks workspace) | Credential-level (each key pair = one account) |
| **Switching** | Change agent in MCP client (new connection) | `swoogo-auth-switch-account` mid-session |
| **Conversational context** | Lost on workspace switch | Preserved across account switches |
| **Cross-account operations** | Not possible in one session | Possible (switch, query, switch back) |

The core difference is that Notion's OAuth flow gives the MCP server rich identity context (bot_id, workspace_id, user) in the token, while Swoogo's client_credentials flow returns only an opaque access_token. However, our in-session switching approach is **more capable** than Notion's separate-connection model.

---

## 3. Design Decisions

### 3.1 No changes to Swoogo's API

Swoogo's `POST /oauth2/token` response:
```json
{ "type": "bearer", "access_token": "abc123...", "expires_at": "2026-02-24 15:30:00" }
```

Adding `user_id`/`account_id` to this response is technically trivial (2 lines in `ApiSession::fields()`), but the Swoogo PRO backend team recommends against it: the blast radius is the entire API client base, and the middleware does not need it. The `consumer_key` (or a hash of it) is sufficient for audit correlation.

If per-user tracking becomes a real requirement in the future, a `GET /users/me` endpoint is the cleaner path (standard REST, does not touch the OAuth flow).

### 3.2 Account identification via user-provided labels (hybrid approach)

The user needs to identify accounts when listing and switching. Three options were evaluated:

| Option | Mechanism | Pro | Contra |
|---|---|---|---|
| **User-provided label** | User names the account at auth time | Zero API dependency | Extra friction (one more question) |
| **`account_name` in token response** | Swoogo returns account name in OAuth response | Automatic, real name | Change in production API |
| **Hybrid (chosen)** | User label now, API enrichment later if needed | Ships today, upgradeable | Label may not match real account name |

**Decision**: User-provided label with auto-generated fallback.

When the user adds an account via `swoogo-auth-add-account`:
1. Credentials are validated against Swoogo (`POST /oauth2/token`)
2. The tool asks: "How do you want to name this account?"
3. If the user provides a label → stored as `tenants.name`
4. If the user skips → auto-generated as `"Account {N}"` (sequential within the session)

If user feedback indicates confusion ("I don't know which account is which"), that justifies requesting `account_name` in the Swoogo token response — backed by real usage data, not speculation.

The `tenants.name` column already exists in the schema and currently auto-generates as `"tenant-{clientId[:8]}"`. This change repurposes it for user-facing labels.

### 3.3 No replication of Swoogo PRO's identity

Per **ADR-2** (Authorization Delegated to Swoogo APIs) and **ADR-9** (Multi-Tenancy via Client Credentials):

1. **Swoogo PRO owns users and accounts.** Replicating that structure in the middleware creates a synchronization problem: every user/account change in Swoogo PRO would need to be mirrored here, with no reliable webhook to trigger it. If credentials are rotated or users are removed on Swoogo's side, the next auth attempt simply fails (Swoogo returns 401). Stale tenant rows in the middleware are inert — they cannot execute anything.

2. **The `consumer_key/secret` is the bridge.** Each credential pair maps to one user within one Swoogo account. The OAuth token obtained from those credentials automatically scopes every API call. The API enforces account hierarchy — the middleware does not need to know it.

3. **Business authorization stays in Swoogo PRO.** Who can access which event, registrant, or session is enforced by the Swoogo API returning 403/404. The middleware normalizes errors but never evaluates resource-level permissions.

4. **User identity is solved without API changes.** The `consumer_key` is unique per user+account. A hash of it is sufficient for audit correlation.

### 3.4 Audit trail without user_id

| Need | Solution |
|---|---|
| "Who did this action?" | `tenant_id` in `audit_logs` — traces to the credential set used |
| "Same human, different accounts?" | `consumer_key` is unique per user+account; a hash stored at auth time correlates the human across tenants |
| "Full user profile?" | Not needed in middleware. Swoogo PRO owns that. If ever needed, `GET /users` is available |

Historical audit logs remain correct as point-in-time records even if credentials are later rotated.

### 3.5 Explicit context passing over AsyncLocalStorage

The initial research proposed `AsyncLocalStorage` as a core pattern. We use explicit context passing instead because:

1. **`ExecutionContext` carries `{ tenant, swoogoClient, threadId, requestId }`** and is injected at every tool execution boundary. Every function signature makes dependencies visible.
2. **The tool execution pipeline is shallow**: `ToolExecutor.execute() → governance → handler → audit`. ALS solves a problem we don't have.
3. **Testability.** Explicit context is trivially mockable. ALS requires setup/teardown of the async store in every test.
4. **Debugging.** Explicit passing makes wrong-tenant bugs immediately visible in the call stack. ALS hides them behind ambient lookups that can silently return stale context if an `await` is missed.

### 3.6 Static tool catalog (no dynamic registry)

The initial research proposed dynamically reconfiguring the tool catalog per account. We use a static catalog because:

1. **All accounts get the same tools.** The difference between accounts is the `swoogoClient` (credentials), not the tool catalog.
2. **MCP clients cache the tool list at session initialization.** Mid-session changes can confuse the LLM's tool selection.
3. **Governance handles per-tenant restrictions.** `maxToolTier` in tenant settings, domain allowlists, and rate limits control what each tenant can do — without manipulating the catalog.

---

## 4. Architecture

### 4.1 Two layers of authentication in MCP

The MCP transport has two distinct authentication layers. This separation is critical for understanding both multi-account switching and logout behavior.

```
Layer 1: TRANSPORT (MCP OAuth — McpAuthProvider)
  Claude Desktop ↔ our server
  Tokens: access_token (1h) + refresh_token (30d), cached in ~/.mcp-auth
  Purpose: "Is this client allowed to talk to our server?"

Layer 2: ACTIVE ACCOUNT (session state)
  Within the already-authenticated session
  State: sessionAccounts[sessionId].activeTenantId
  Purpose: "Which Swoogo account do I execute this tool call against?"
```

Multi-account switching operates in Layer 2. The client is authenticated (Layer 1) and does not need to re-authenticate. The account switching tools mutate internal session state. The MCP client is unaware of the switch.

### 4.2 Session state model

Each MCP session tracks a set of authenticated accounts and which one is active:

```typescript
interface SessionAccountState {
  activeTenantId: string;
  authenticatedTenants: Map<string, { label: string }>; // tenantId → label
}
const sessionAccounts = new Map<string, SessionAccountState>();
```

The account list is **session-scoped**: it starts with the account used during the initial OAuth flow, grows as the user adds accounts via tools, and resets when the session ends. Labels are stored both in session state (for quick access) and in `tenants.name` (for persistence across sessions).

### 4.3 User experience flow

```
Session start → OAuth with credentials A → Account A active (label: "Account 1" or user-provided)
                                         ↓
User: "add my other account"    → Tool: swoogo-auth-add-account
                                         ↓
                                  Tool prompts for consumer_key + consumer_secret
                                         ↓
                                  Validate against Swoogo (POST /oauth2/token) → OK
                                  Credentials stored encrypted in tenants table
                                         ↓
                                  Tool: "How do you want to name this account?"
                                  User: "Producción LATAM" (or skips → "Account 2")
                                         ↓
                          Session now has [A, B], active = B
                                         ↓
User: "switch to account A"    → Tool: swoogo-auth-switch-account → active = A
                                         ↓
User: "list my accounts"       → Tool: swoogo-auth-list-accounts
                                  Returns:
                                    1. "Account 1" (active)
                                    2. "Producción LATAM"
```

**Credential security**: API keys are passed as tool input parameters, not typed into the chat by the user. The MCP client collects them via the tool's input schema. This keeps credentials out of the conversation history visible to the LLM. In a future phase, a browser-based auth form (similar to the initial OAuth flow) can replace direct credential input entirely — see §6.

### 4.4 Account switching by transport

**HTTP transport** — stateless, no changes needed:
- The client stores N `tenant_id`s (obtained via `POST /api/auth/authenticate` for each credential pair).
- Each request includes `X-Tenant-Id` header with the desired account.
- Switching = sending a different header.

**MCP transport** — three new system tools (Tier 0):

| Tool | Input | Behavior |
|---|---|---|
| `swoogo-auth-add-account` | `{ swoogoClientId, swoogoClientSecret, label? }` | Validates credentials → provisions/finds tenant → sets label → adds to session → switches to it |
| `swoogo-auth-switch-account` | `{ accountId }` | Switches active account (only among previously authenticated ones) |
| `swoogo-auth-list-accounts` | none | Returns authenticated accounts with labels and active indicator |

`swoogo-auth-whoami` reflects the active account including its label.

### 4.5 Token auto-refresh (Postman pattern)

Credentials are stored encrypted in the `tenants` table at first authentication. The `SwoogoTokenManager` handles the full lifecycle transparently:

1. Bearer token cached in Redis with TTL (`expires_at - 60s`)
2. On expiry, `getAccessToken()` re-exchanges `consumer_key/secret` automatically
3. On 401 from Swoogo, `SwoogoApiClient` invalidates cache and retries
4. The user never re-authenticates unless credentials are rotated on Swoogo PRO's side

Authenticate once per account, use indefinitely.

### 4.6 Session tenant validation

The MCP server validates on every request that the OAuth token's `tenantId` belongs to the session's authenticated tenant set:

```typescript
const sessionState = sessionAccounts.get(sessionId);
if (!sessionState?.authenticatedTenants.has(tenantId)) {
  sendJsonRpcError(req, reply, 403, -32001, 'Session tenant mismatch');
}
```

The OAuth token's tenantId is the "session owner" (the first account authenticated). Additional accounts are validated by the `swoogo-auth-add-account` tool before being added to the set.

### 4.7 MCP token revocation (real logout)

Without revocation, `swoogo-auth-logout` closes the server-side session but the MCP client silently reconnects using the refresh_token cached in `~/.mcp-auth`. The user never re-enters credentials.

Token revocation uses a Redis-backed blocklist:

```
On logout/reset:
  → hash = SHA-256(refresh_token)[:32]
  → Redis SET  revoked:mcp:{hash}  "1"  EX 2592000  (30 days = refresh token lifetime)

On token exchange (exchangeRefreshToken):
  → hash = SHA-256(refresh_token)[:32]
  → Redis GET  revoked:mcp:{hash}
  → If exists → reject with "token_revoked" error
  → MCP client receives 401 → triggers full OAuth flow → user must provide API keys
```

Revocation entries auto-expire when the refresh token would have expired anyway. No garbage collection needed.

**Redis key pattern** (added to `REDIS_KEYS` in `constants.ts`):
```
revoked:mcp:{tokenHash}  → "1"  (TTL: 30 days)
```

---

## 5. Implementation Plan

### 5.1 What requires no changes

| Component | Why it works as-is |
|---|---|
| `SwoogoTokenManager` | Redis-cached per-tenant keys; each tenant has its own cache key |
| `SwoogoApiClient` | Instantiated per-request via `createContext()`; new tenant = new client instance |
| `ExecutionContext` | Explicit context; swap `tenant` + `swoogoClient` to switch accounts |
| `tenants` table + all FKs | All tables scoped by `tenant_id`; N accounts = N tenant rows |
| Redis key namespacing | `tenant:{id}:*` pattern; inherently isolated |
| `GovernanceService` | Per-tenant rate limits and tier enforcement |
| `AuditService` | Per-tenant audit trail; each account's actions logged under its own `tenant_id` |
| Auth routes (`/api/auth/authenticate`) | Find-or-create tenant by credentials; reuse validation logic in new MCP tool |

### 5.2 What needs work

| Component | Change | Files | Effort |
|---|---|---|---|
| Session state model | `Map<sessionId, string>` → `Map<sessionId, SessionAccountState>` | `src/interface/mcp/server.ts` | Low |
| Session tenant validation | Validate against authenticated tenant set | `src/interface/mcp/server.ts` | Low |
| `createContext` closure | Resolve against session's active tenant dynamically | `src/main.ts`, `src/interface/mcp/server.ts` | Low |
| MCP token revocation | Redis-backed blocklist for refresh tokens | `src/shared/constants.ts`, `src/interface/mcp/auth/mcp-auth-provider.ts` | Low |
| Logout/reset revocation | Write revocation entry before closing session | `src/interface/mcp/auth-tools.ts` | Low |
| `CircuitBreaker` | Per-tenant instance instead of global singleton | `src/main.ts` | Low |
| `swoogo-auth-add-account` | New tool: validate credentials, provision tenant, add to session, switch | `src/interface/mcp/auth-tools.ts` | Medium |
| `swoogo-auth-switch-account` | New tool: switch active tenant within authenticated set | `src/interface/mcp/auth-tools.ts` | Low |
| `swoogo-auth-list-accounts` | New tool: list authenticated accounts with active indicator | `src/interface/mcp/auth-tools.ts` | Low |

### 5.3 Database impact

**No new tables. No migrations. No schema changes.**

Each Swoogo account that gets authenticated creates (or reuses) a row in the existing `tenants` table. The `swoogo_client_id` UNIQUE constraint ensures one row per Swoogo account. All existing FK relationships (`threads.tenant_id`, `audit_logs.tenant_id`, etc.) work as-is.

When a user switches accounts mid-session, subsequent tool calls produce audit logs and threads under the new `tenant_id`. The data belongs to the account, not to the human operating it.

### 5.4 Swoogo PRO API impact

**No changes required.**

---

## 6. Future Improvements

### 6.1 Browser-based auth for add-account

The current `swoogo-auth-add-account` collects credentials via tool input parameters. A more polished UX would redirect the user to a browser-based auth form:

1. Tool returns a clickable URL (e.g., `https://ai.swoogo.com/auth/add-account?session={id}`)
2. User clicks → browser opens → secure form collects `consumer_key` + `consumer_secret` + optional label
3. Form submits to our server → validates → adds tenant to session
4. Browser shows success → user returns to MCP client

This mirrors the initial OAuth flow pattern where the MCP client opens a browser for authentication. The difference is that the initial flow is triggered by a 401 (the MCP SDK handles it automatically), while add-account is triggered by a tool call (requires explicit URL return).

**Prerequisite**: The MCP SDK would need to support tool-initiated browser redirects, or the tool returns the URL as text for the user to click manually.

### 6.2 `account_name` from Swoogo token response

If user feedback indicates confusion with user-provided labels ("I don't know which account is which"), we can request that Swoogo PRO adds `account_name` to the token response:

```json
{ "type": "bearer", "access_token": "...", "expires_at": "...", "account_name": "Acme Corp" }
```

This is cosmetic data (not identity), low blast radius, and would eliminate the label prompt entirely. Deferred until backed by real usage data.

### 6.3 User-centric model (Phase 5+)

When OAuth via Swoogo ID or Stytch is implemented (ADR-3 progressive auth), the middleware gains a real user identity. At that point:

1. A `users` table may be justified to store middleware-specific user preferences (default account, UI settings).
2. A `user_accounts` join table could enable `swoogo-auth-list-accounts` to query DB instead of session state.
3. A `GET /users/me` endpoint on Swoogo PRO could provide full user profile for richer audit data.

This is deferred until there is an actual identity provider. Building the tables now would create empty scaffolding with no source of truth to populate them.

---

## 7. References

### Internal

- **ADR-2**: Authorization delegated to Swoogo APIs (`docs/plan/architecture.md`)
- **ADR-3**: Progressive auth — MVP = client_id/secret, later = OAuth/Stytch (`docs/plan/architecture.md`)
- **ADR-9**: Multi-tenancy via client credentials (`docs/plan/architecture.md`)
- **ADR-14**: ToolExecutor as unified orchestration layer (`docs/plan/architecture.md`)
- **Research**: Initial multi-tenancy research (Notion analysis, AsyncLocalStorage evaluation, dynamic registry evaluation) — consolidated into this document
- **Master Plan**: `docs/plan/MASTER-PLAN.md` § Multi-Tenancy Boundary

### External

- **Notion MCP setup**: https://www.notion.com/help/notion-mcp — Official docs showing one-connection-per-workspace model
- **Notion MCP multi-workspace limitation**: https://www.reddit.com/r/NotionMCP/ — Community reports confirming separate agent instances required per workspace
- **MCP OAuth specification**: https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization — MCP authorization spec (token lifecycle, refresh flow)
- **RFC 6749 §2.3.1**: Client credentials encoding for Basic auth header

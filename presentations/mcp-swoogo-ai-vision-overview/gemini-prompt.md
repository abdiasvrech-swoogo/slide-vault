# Swoogo — Presentation Master Prompt

You are a Senior Visual Designer & Technical Storyteller. Generate a standalone HTML presentation following these exact specifications.

**CRITICAL: Generate ALL content in American English.** All slide titles, body text, labels, buttons, and any user-facing text must be in American English. Translate any Spanish content to English while preserving technical accuracy and tone.

---

## Part 1: Project Context

The generated HTML file will be placed inside a project with this structure:

```
slide-vault/
├── assets/
│   ├── logo/
│   │   ├── logo-full.svg      # Wordmark — for presentation headers
│   │   └── logo-icon.png      # Icon — for favicon
│   └── brand/
│       └── tokens.json
├── presentations/
│   └── mcp-swoogo-ai-vision-overview/
│       ├── index.html          ← THE FILE YOU GENERATE GOES HERE
│       ├── image.svg           # Social preview source (editable)
│       └── image.png           # Social preview (exported, 1200x630)
└── ...
```

The HTML file lives at `presentations/mcp-swoogo-ai-vision-overview/index.html`. Include a `<base>` tag in the `<head>` to resolve all paths from the project root:

```html
<base href="../../" />
```

Then reference all assets from the project root — no `../../` prefix needed:

| Asset               | Path                        |
| ------------------- | --------------------------- |
| Full logo (header)  | `assets/logo/logo-full.svg` |
| Icon logo (favicon) | `assets/logo/logo-icon.png` |

Use these exact paths. The `<base>` tag handles the directory resolution.

---

## Part 2: Brand & Visual System

**Palette:**

- Page background: `#F9FAFB`
- Card/surface: `#FFFFFF`
- Dark panels: `#262966`
- Primary text: `#262966`
- Secondary text: `#64748B`
- Tertiary text: `#94A3B8`
- **Primary accent (Swoogo Lavender): `#726BEA`**
- **Deep accent (Swoogo Pacific): `#262966`**
- **Flame accent (Swoogo Orange): `#F46A34`**
- **Glow accent: `#EE8252`**
- **Teal accent: `#4BCDB5`**
- **Bermuda accent: `#A0F1E2`**
- Accent gradient: `linear-gradient(135deg, #726BEA, #262966)`
- Badge bg: `#E9ECFF` with text `#726BEA`
- Success: Emerald (`#10B981`) | Warning: Amber (`#F59E0B`) | Danger: Red (`#EF4444`) | Info: Blue (`#3B82F6`)

**Tailwind Config** — include this in the HTML:

```javascript
tailwind.config = {
  theme: {
    extend: {
      colors: {
        swoogo: {
          lavender: "#726BEA",
          pacific: "#262966",
          mist: "#E9ECFF",
          chalk: "#F5F5FA",
          snow: "#F9FAFB",
          flame: "#F46A34",
          glow: "#EE8252",
          teal: "#4BCDB5",
          bermuda: "#A0F1E2",
        },
      },
    },
  },
};
```

Then use `bg-swoogo-lavender`, `text-swoogo-pacific`, `bg-swoogo-mist`, etc. in classes.

**UI Components:**

- Cards: `bg-white`, rounded-2xl to rounded-3xl, `border border-slate-100`, `shadow-sm`
- Dark panels: `bg-swoogo-pacific` for contrast sections
- Status badges: pill-shaped, `bg-swoogo-mist text-swoogo-lavender`, bold uppercase
- Metric cards: `border-b-8 border-b-swoogo-lavender` (or semantic color), `shadow-lg`
- CTA buttons: `bg-swoogo-lavender hover:bg-swoogo-pacific text-white`
- Accent highlights: use `bg-swoogo-flame` or `text-swoogo-flame` sparingly for emphasis

**Logo:**

- Header: top-left, full wordmark (`assets/logo/logo-full.svg`), height `h-5 sm:h-7`
- Favicon: `<link rel="icon" type="image/png" href="assets/logo/logo-icon.png">`

**Typography:**

- Font: Inter (Google Fonts), sans-serif fallback
- Weights: 400, 500, 600, 700, 800, 900
- High contrast: bold/black headers (700-900) vs regular body (400-500)
- Gradient text: `.gradient-text { background: linear-gradient(135deg, #726BEA, #262966); -webkit-background-clip: text; -webkit-text-fill-color: transparent; }`
- Micro labels: 10px, font-black, uppercase, tracking-widest

---

## Part 3: Tone & Writing

**Voice:** North American English. Senior Engineer to Senior Engineer.

**Rules:**

- **No AI-speak.** Never use: "In conclusion", "It is important to note", "This demonstrates", "In summary", "As we can see", "Let's dive in", "Without further ado"
- **Declarative & direct.** Short, punchy sentences. State facts. Cut filler.
- **Specific nouns.** Concrete technical terms over vague abstractions.
- **Evidence first.** Lead with the surprising result, then explain how.

---

## Part 4: Content Structure

**Narrative Arc (5 parts):**

1. **Hook (Slide 1):** Icon + title with gradient accent + subtitle + optional badges.
2. **Context (Slides 2-3, max 2):** Why this matters. Bullet or triad layout.
3. **Core Message (Slide 4):** Main point, stated directly. Impact layout — oversized text.
4. **Evidence (Slides 5 to N-1):** Metrics, architecture, process, comparisons. Pick layout per content type.
5. **Takeaway (Final Slide):** CTA (numbered actions) or Final (single powerful statement).

**Visual Rules:**

- Every slide has at least one visual element (icon, card, chart, diagram)
- Cards for data points — never raw text blocks
- Minimal text — if it's not essential, cut it
- Generous whitespace
- Max 3 content blocks per slide
- Max 6 items in any grid
- Max 3 bullet points per bullet slide

**Slide Types:**

| Type           | Use                                                |
| -------------- | -------------------------------------------------- |
| `cover`        | First slide. Hero title + icon + subtitle + badges |
| `impact`       | Single powerful statement. Section divider         |
| `bullets`      | 2-3 points + insight card                          |
| `triad`        | Three concepts (legacy/problem/solution)           |
| `metrics`      | 3-6 metric cards with big numbers                  |
| `process`      | 3-5 sequential steps (pipeline or vertical)        |
| `architecture` | System diagram, division of labor                  |
| `comparison`   | Before/after side-by-side                          |
| `features`     | 3-6 icon cards                                     |
| `roadmap`      | Phased timeline                                    |
| `cta`          | Numbered action items                              |
| `final`        | Closing statement + gradient icon                  |

---

## Part 5: Technical Output

**Generate a single standalone HTML file.** All CSS and JS inline. No local dependencies.

**CDN resources:**

```html
<script src="https://cdn.tailwindcss.com"></script>
<link
  href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap"
  rel="stylesheet"
/>
<link
  rel="stylesheet"
  href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css"
/>
```

**Responsive & Mobile-first:**

- Text scales: `text-base sm:text-lg md:text-xl`
- Padding scales: `p-5 sm:p-8 md:p-12 lg:p-16`
- Grids: `grid-cols-1 sm:grid-cols-2 lg:grid-cols-N`
- Breakpoints: sm(640), md(768), lg(1024)

**Navigation:**

- Arrow keys (left/right) + spacebar → next/prev
- Touch swipe (horizontal) on content area
- Click buttons (min 40x40px touch targets)
- Disable prev on first slide, next on last (`disabled:opacity-30`)
- Progress bar (top, 1.5px, `bg-swoogo-lavender`)
- Slide counter in header
- Nav dots on desktop

---

## Part 6: Presentation Content — Swoogo MCP Vision Overview

**Presentation Title:** Swoogo MCP — Complete Project Vision

**Subtitle:** Intelligent middleware between users and the Swoogo API

**Target Audience:** Stakeholders, technical leadership, engineering team

---

## Detailed Content Structure:

### SLIDE 1: Cover (type: `cover`)
**Title:** Swoogo MCP  
**Subtitle:** Complete Project Vision — Intelligent middleware for event management  
**Icon:** `fa-network-wired` (representing middleware/connectivity)  
**Badges:**
- "Model Context Protocol"
- "29 Tools"
- "466 Tests"
- "Enterprise-Grade"

---

### SLIDE 2: What We're Building (type: `impact`)
**Statement:** An MCP server that translates natural language into correct Registration Forms

**Sub-text:** Middleware between Claude Desktop, Cursor, and Swoogo's REST API

---

### SLIDE 3: The Core Problem (type: `triad`)
**Title:** Why users struggle with Swoogo integrations

**Three concepts:**
1. **Complex API**
   - Need to understand endpoints, payloads, OAuth patterns
   - 94+ endpoints across 13 entities
   - Undocumented edge cases

2. **Trial and error**
   - Users guess endpoint structure
   - Registration Forms fail silently
   - Support tickets pile up

3. **LLM hallucinations**
   - AI assistants invent endpoints
   - Mix patterns from different entities
   - Generate code that doesn't work

---

### SLIDE 4: The Solution (type: `bullets`)
**Title:** Intelligent middleware with complete governance

**Bullets:**
- Natural language → API calls with permissions, audit, confirmations
- Simplifies Registration Form creation without needing API knowledge
- Dual transport: HTTP/SSE for apps, MCP for AI hosts
- Enterprise-grade: multi-tenancy, rate limiting, audit trail

**Insight card:** "Not replacing the platform — enhancing the integration experience"

---

### SLIDE 5: Key Metrics (type: `metrics`)
**Title:** Project by numbers

**6 metric cards (2x3 grid):**
1. **29** Tools implemented (25 domain + 4 system)
2. **466** Tests across 25 files
3. **13** Entities covered (events, registrants, sessions, speakers, etc.)
4. **8** Security guardrails (G0-G7)
5. **4** LLM providers (OpenAI, Anthropic, Google, Bedrock)
6. **2** Transports (HTTP/SSE + MCP)

---

### SLIDE 6: Architecture Overview (type: `architecture`)
**Title:** One product, two transports

**Diagram structure:**
```
┌────────────────────────────────────────┐
│         SHARED CORE                    │
│                                        │
│ Tool Registry  │  ToolExecutor         │
│ Governance     │  Audit Service        │
│ Rate Limiter   │  Circuit Breaker      │
└──────────┬──────────────┬──────────────┘
           │              │
    ┌──────┴─────┐  ┌────┴─────────┐
    │ TRANSPORT  │  │  TRANSPORT   │
    │ HTTP/SSE   │  │  MCP         │
    │            │  │  Streamable  │
    │ Server has │  │  HTTP        │
    │ LLM        │  │              │
    │            │  │  Host has    │
    │ Chat UIs,  │  │  LLM         │
    │ SDK, CLI   │  │              │
    └────────────┘  │  Claude      │
                    │  Desktop,    │
                    │  Cursor      │
                    └──────────────┘
```

**Note:** Both converge at ToolExecutor — single governance point

---

### SLIDE 7: Execution Pipeline (type: `process`)
**Title:** From tool call to result — 5 governance checks

**Sequential flow with 5 steps:**

**Step 1: Blocklist** (G1)
- Tool name in prohibited set?
- DENIED → `drop`, `truncate`, `delete_all`, `purge`, `destroy`

**Step 2: Tier escalation**
- Dynamic tier computation
- `swoogo-update-registrant` with status change → Tier 2 → Tier 3

**Step 3: Tier access**
- Effective tier > tenant's max tier?
- DENIED → 403

**Step 4: HITL check**
- Tier 2+ requires confirmation
- Exception: MCP hosts with `hostConfirmed=true`
- PAUSE → PostgreSQL + Redis (300s TTL)

**Step 5: Rate limit**
- Sliding window per tenant per tool
- read: 60 RPM | write: 20 RPM | bulk: 5 RPM
- 429 if exceeded

**Result:** ALLOWED → execute tool handler → audit log

---

### SLIDE 8: Four Security Layers (type: `architecture`)
**Title:** Defense in depth

**4 layers (vertical stack):**

**Layer 1: IDENTITY**
- Who is this tenant?
- OAuth 2.1 + tenant status validation
- Token revocation by epoch (Redis)

**Layer 2: GOVERNANCE**
- Can they use this tool?
- Blocklist + tier check + rate limit
- Enforced at ToolExecutor

**Layer 3: BUSINESS**
- Can they access this resource?
- Delegated to Swoogo API
- 403/404 normalized to neutral messages

**Layer 4: CONFIRMATION**
- Did the user approve?
- HITL for Tier 2/3
- State machine: pending → approved/rejected/expired

---

### SLIDE 9: 29 Tools — Three Tiers (type: `features`)
**Title:** Auto-execute, confirm, double-confirm

**Grid of 3 cards:**

**Tier 0 — Auto-execute (17 tools)**
- All read operations
- `swoogo-events`, `swoogo-registrants`, `swoogo-sessions`
- `swoogo-event-questions`, `swoogo-reg-types`
- `swoogo-detect-session-conflicts`
- Icon: `fa-eye`

**Tier 2 — Require confirmation (7 tools)**
- Write operations with user preview
- `swoogo-create-registrant`, `swoogo-update-registrant`
- `swoogo-assign-session`, `swoogo-checkin-registrant`
- Icon: `fa-hand-paper`

**Tier 3 — Double confirmation (1 tool)**
- High-risk, immutable operations
- `swoogo-create-transaction` (financial records)
- Icon: `fa-shield-alt`

**Note:** +4 system tools (auth, logout, reset, confirm-action)

---

### SLIDE 10: Guardrails (G0-G7) (type: `bullets`)
**Title:** 8 guardrails, 2+ enforcement layers each

**Context:** Each guardrail has multiple enforcement layers (prompt + code)

**Bullets (pick 3 most impactful):**
- **G0: Data Accuracy** — Never hallucinate IDs. Prompt + pre-validation + API enforcement
- **G1: No Delete** — Block destructive ops. Governance blocklist + system prompt
- **G3: Write Confirmation** — HITL gate for Tier 2+. Governance + prompt

**Insight card:** "Bypassing one layer doesn't break protection"

---

### SLIDE 11: Multi-Tenancy Model (type: `bullets`)
**Title:** One connection per account — total isolation

**Bullets:**
- One MCP connection = one Swoogo account (Notion model)
- No in-session account switching
- Isolation: tenant_id FK in all tables, rate limits per tenant, circuit breakers per tenant

**Code example:**
```json
{
  "mcpServers": {
    "swoogo-production": {
      "url": "https://ai.swoogo.com/mcp"
    },
    "swoogo-staging": {
      "url": "https://ai-staging.swoogo.com/mcp"
    }
  }
}
```

**Insight card:** "Zero risk of cross-tenant leakage"

---

### SLIDE 12: Integration Guidance — The End Goal (type: `impact`)
**Statement:** From "Generate a Registration Form" to production-ready code in one prompt

**Sub-text:** Composable technical guides filtered by scope — ~30,000 tokens saved per session

---

### SLIDE 13: Act + Ask + Expand Flow (type: `process`)
**Title:** Incremental scope expansion

**3-step flow:**

**1. ACT**
- User: "Generate Registration Form for Event 12345"
- Host infers scope: ["registration"]
- Calls `swoogo-integration-guide` with scope
- Receives ~3,200 tokens of technical guidance
- Inspects event: metadata, questions, reg-types

**2. ASK**
- Host analyzes event data
- "This event has paid reg-types. Need payment integration?"
- "Found sq_* fields (session selection). Include scheduling?"
- Contextual prompts based on real data

**3. EXPAND**
- User confirms sessions → host requests scope: ["sessions"]
- Only what's needed, one scope at a time
- Prevents context contamination

---

### SLIDE 14: Integration Guide Architecture (type: `architecture`)
**Title:** Composable layers by scope

**Diagram (vertical layers):**
```
┌─────────────────────────────────┐
│   CORE (always included)        │
│ Architecture · Auth · Security  │
│ Validation · Pitfalls           │
└─────────────┬───────────────────┘
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
┌──────────┐┌──────────┐┌──────────┐
│ DOMAIN   ││ DOMAIN   ││ DOMAIN   │
│registrat.││ sessions ││ speakers │
└──────────┘└──────────┘└──────────┘
      │
┌─────────────────────────────────┐
│ CROSS-CUTTING (conditional)     │
│ i18n · Caching · Fees · UX      │
└─────────────┬───────────────────┘
              ▼
┌─────────────────────────────────┐
│ CHECKLIST (adapts to scope)     │
└─────────────────────────────────┘
```

**Details:**
- 5 domains: registration, sessions, speakers, contacts, reporting
- 17 fixed sections — predictable contract
- No `scope: "all"` — forces incremental pattern

---

### SLIDE 15: Token Efficiency (type: `metrics`)
**Title:** Measured token savings

**3 metric cards:**

**Card 1: Delivery**
- Registration form: **29%** savings (4,500 → 3,200 tokens)
- Sessions only: **44%** savings (4,500 → 2,500 tokens)

**Card 2: Persistent context**
- 15 turns with full guide: 67,500 tokens
- 15 turns composable: 48,000 tokens
- **19,500** tokens saved

**Card 3: Downstream quality**
- Fewer hallucinations
- No speculative tool calls
- Fewer correction loops
- **~9,850** tokens saved

**Total per session:** ~30,650 tokens

---

### SLIDE 16: Engineering Patterns (type: `features`)
**Title:** Solving real API problems

**Grid of 6 cards with icons:**

1. **Fuzzy Search** (`fa-search`)
   - 3-stage escalation: exact → expanded → full scan
   - Confidence scoring (high/medium/low)
   - "Swoogo Summit" finds "Swoogo Summit 2026" (0.95)

2. **Page-Grouped Questions** (`fa-layer-group`)
   - Flat API response → grouped by page
   - ~40% fewer tokens
   - Structure reflects actual form layout

3. **Translation Fallback** (`fa-language`)
   - Empty `""` translations → base language value
   - LLM sees actual labels, not blank strings

4. **Response-Diff** (`fa-code-compare`)
   - Only show changed fields in updates
   - Detect silently ignored fields
   - Value mismatches → warnings

5. **Normalized Maps** (`fa-map`)
   - PHP `[]` vs `{}` distinction
   - Converts empty arrays to objects for keyed maps
   - Type safety preserved

6. **Expand Params** (`fa-expand-arrows-alt`)
   - Hydrate relations inline
   - `expand=fees,speakers,location`
   - Eliminate N+1 queries

---

### SLIDE 17: Upstream Contributions (type: `bullets`)
**Title:** Improving Swoogo API for everyone

**Context:** These changes benefit all API consumers, not just the MCP server

**Bullets:**
- **`ids` query parameter** — Batch fetch (5 items in 1 call vs 5 calls). Applied to 30+ endpoints via BaseController
- **`expand=fees`** — Inline pricing for packages/reg-types. -90% latency for pricing operations
- **Registration Page optimization** — From 10+2N calls to 4-6 calls (keystone call: `swoogo-event-questions`)

**Insight card:** "API improvements upstream = value for entire platform"

---

### SLIDE 18: Case Study — Registration Page (type: `comparison`)
**Title:** Call optimization in action

**Left side (Before — Naive approach):**
1. Event metadata
2. Flat question list
3. Reg-types list
4. Session list
5. Package list
6-N. Package detail × N (for fees)
N+1-2N. Speaker detail × N (for assignments)

**Total:** 10 + 2N calls

**Right side (After — Optimized):**
1. `swoogo-events(id)` — metadata
2. `swoogo-event-questions(event_id, expand=page)` — **keystone call**
3. `swoogo-packages(event_id, expand=fees)` — pricing inline
4. `swoogo-reg-types(event_id, expand=fees)` — types with fees
5-6. `swoogo-speakers(event_id, expand=sessions)` — linkage (if needed)

**Total:** 4-6 calls

**Savings badge:** 10+2N → 4-6 (60-80% fewer calls for N=10)

---

### SLIDE 19: Tech Stack (type: `features`)
**Title:** Production-grade choices

**Grid of 6 cards:**

1. **Runtime** — Node.js 22+ LTS
   - Performance for SSE, async I/O

2. **Language** — TypeScript (strict)
   - Type safety, Zod integration

3. **LLM SDK** — Vercel AI SDK v6
   - Multi-provider, streaming, 20M+ downloads/month

4. **HTTP** — Fastify 5
   - Performance, schema validation

5. **Database** — PostgreSQL 17
   - Audit durability, JSONB

6. **Cache** — Valkey 8
   - Rate limiting, token cache, confirmations

---

### SLIDE 20: Security Hardening (type: `bullets`)
**Title:** 7 P0 vulnerabilities closed pre-launch

**Context:** Pre-launch security audit identified and fixed:

**Bullets (pick 3 most critical):**
- **`host_confirmed` bypass** — Malicious host could inject flag to skip HITL. Moved to ExecutionContext
- **Session limit race condition** — Non-atomic cap check. Fixed with Redis atomic INCR
- **Sanitizer bypass** — Unicode zero-width + NFKD normalization evasion. Enhanced sanitizer

**Insight card:** "All 7 fixed in `feat/security-hardening`, each with test coverage"

---

### SLIDE 21: User Experience (type: `process`)
**Title:** From prompt to result

**Example flow:**

**Step 1:** User writes
```
"How many registrants does Swoogo Summit have?"
```

**Step 2:** System executes
- Fuzzy search finds "Swoogo Summit 2026" (score: 0.95)
- Counts registrants with event_id filter

**Step 3:** Response
```
"Swoogo Summit 2026 has 847 confirmed registrants."
```

**Bonus example:** Registration with confirmation
```
User: "Register john@example.com in Event 12345, VIP type"
→ System prepares action (Tier 2)
→ Shows preview: "Create registrant john@example.com, VIP ($299)"
→ User confirms: "Yes, proceed"
→ Executes → Verifies with read-after-write
→ Response: "Registrant created. Confirmation: EXT-2847"
```

---

### SLIDE 22: Project Phases (type: `roadmap`)
**Title:** What's done, what's next

**3 phases:**

**Phase 1-3 — COMPLETED**
- Foundation + POC (auth, LLM orchestrator, dual transport)
- Read operations (17 tools, fuzzy search)
- Write + HITL (8 write tools, confirmations)
- Testing (25 files, 466 tests)
- Abuse protection (rate limiting, circuit breaker)
- Security hardening (7 P0 vulnerabilities closed)
- Integration guidance (composable guides, 5 domains)

**Phase 4 — IN PROGRESS**
- Generative tools (text, HTML, analytics)
- Bulk operations framework
- Idempotency keys

**Phase 5 — PLANNED**
- Enterprise hardening (Stytch/SSO, OpenTelemetry)
- Client SDK 1.0
- Audit dashboard

---

### SLIDE 23: Code Metrics (type: `metrics`)
**Title:** Production-ready codebase

**6 metric cards:**
1. **~60** TypeScript files (src/)
2. **25** Test files
3. **466** Total tests
4. **29** Tools implemented
5. **14** ADRs documented
6. **~20** Documentation files

---

### SLIDE 24: Key Decisions (type: `bullets`)
**Title:** Technical choices that shaped the architecture

**Bullets:**
- **Vercel AI SDK v6 over Mastra** — AI SDK v6 (Dec 2025) closed the gap. Extracted patterns, didn't import complexity
- **Server-driven HITL** — AI SDK's `needsApproval` requires React. Our clients aren't React apps
- **One connection per account** — Natural isolation, zero cross-tenant leakage risk

**Insight card:** "TypeScript/Node.js chosen for SSE performance — 10-40x advantage over PHP for streaming"

---

### SLIDE 25: Hallazgos (Problems Solved) (type: `bullets`)
**Title:** Critical discoveries during development

**Bullets:**
- **Laravel doesn't support HITL correctly** — `InvokingTool` event has no cancellation mechanism. Migrated to Node.js + Vercel AI SDK
- **Monolithic guide contaminates context** — Hosts hallucinated endpoints with full guide. Solved with composable scopes
- **Token revocation in MCP** — Sessions persist tokens. Epoch strategy in Redis invalidates old tokens on logout

**Insight card:** "Each problem led to architecture improvement"

---

### SLIDE 26: Business Value (type: `triad`)
**Title:** Value across three dimensions

**Three concepts:**

1. **For Users**
   - Radical simplification: natural language → correct forms
   - No API knowledge required
   - Safe: writes require confirmation, no destructive ops

2. **For Product**
   - Competitive differentiation via MCP integration
   - Reduced support tickets (self-service)
   - Platform extensibility (29 → 94+ tools)

3. **For Engineering**
   - Proven architecture (466 tests, 8 guardrails, 14 ADRs)
   - No vendor lock-in (4 LLM providers)
   - Reusable patterns (governance, PII redaction, HITL)

---

### SLIDE 27: Scope Boundaries (type: `comparison`)
**Title:** What's in, what's out

**Left side (IN SCOPE):**
- 29 tools covering 13 entities
- Read + write with confirmation
- Complete Registration Form flow
- Multi-tenancy by design
- Audit trail (append-only logs)
- HITL for Tier 2/3 operations

**Right side (OUT OF SCOPE — Explicit decisions):**
- Event creation (belongs to platform)
- Delete operations (too high risk)
- Frontend/UI (MCP is middleware)
- RAG/Vector DB (ADR-6: not needed)
- Framework LLM (using Vercel AI SDK)

---

### SLIDE 28: Risk Mitigation (type: `bullets`)
**Title:** Known risks, active mitigations

**Bullets:**
- **Risk:** Tools exceed token budget → **Mitigation:** Dynamic tool loading by domain, 29 < cap of ~40
- **Risk:** No bulk endpoints in Swoogo API → **Mitigation:** Sequential execution + preview + progress tracking
- **Risk:** LLM hallucinates tool calls → **Mitigation:** G0 guardrail + schema validation + HITL

**Insight card:** "Circuit breakers + error normalization handle API changes gracefully"

---

### SLIDE 29: Next Steps (type: `cta`)
**Title:** Recommended actions

**Numbered action items:**

1. **Merge integration guidance enforcement** to main
2. **Validate end-to-end** with Claude Desktop + Cursor (Registration Form scenarios)
3. **Implement maxTier dynamic** per tenant (currently hardcoded Tier 2+)
4. **SDK structured response contract** — Versioned SDKEvent envelope for client SDK
5. **Phase 4 kickoff** — Generative tools + bulk operations framework

---

### SLIDE 30: Final Takeaway (type: `final`)
**Statement:** Enterprise-grade middleware — from natural language to correct Registration Forms

**Icon:** `fa-check-double` with gradient  
**Sub-text:** 29 tools · 466 tests · 8 guardrails · Production-ready

---

## Part 7: Additional Implementation Notes

**Visual treatment for architecture diagrams:**
- Use cards with arrows showing data flow
- Color-code layers: identity (blue), governance (purple), business (green), confirmation (orange)
- Show convergence at ToolExecutor as central node

**Process diagrams:**
- Numbered steps with gradient background
- Use Swoogo lavender for step numbers
- Include decision points (DENIED/ALLOWED/PAUSE) with color coding

**Comparison slides:**
- Clear left/right division
- Before: use warning/red tones
- After: use success/emerald tones
- Show specific call counts and savings

**Metrics cards:**
- Use `border-b-8` with different colors for variety
- Large gradient text for numbers
- Brief label below each metric

**Code examples:**
- Use `bg-slate-50` for code blocks
- Keep examples realistic (user prompts, API responses)
- Max 10 lines visible per block

**Roadmap visualization:**
- Horizontal timeline or vertical phases
- Completed phases: emerald background with checkmark
- In-progress: amber background
- Planned: slate background

**Ensure all slides:**
- Have consistent header with logo
- Include slide numbers
- Use proper responsive classes
- Maintain visual hierarchy
- Include navigation controls
- Have proper touch targets (min 40x40px)
- Use Font Awesome 6 icons throughout

---

## Critical Requirements:

1. **DO NOT summarize or simplify** — use all the technical details provided
2. **Follow the exact slide structure** specified (30 slides)
3. **Use the exact color palette** from the brand tokens
4. **Include all numerical metrics** as specified (29 tools, 466 tests, 30,650 tokens saved, etc.)
5. **Use proper icon classes** from Font Awesome 6
6. **Maintain the narrative arc** from problem → solution → architecture → patterns → value
7. **Include code examples and user flows** where specified
8. **Use proper slide types** as indicated for each slide
9. **Follow typography rules** (Inter font, weights, gradient text for numbers)
10. **Implement full navigation** (arrows, swipe, progress bar, slide counter, nav dots)
11. **Show technical depth** — this is engineer-to-engineer communication
12. **Emphasize governance and security** — 4 layers, 8 guardrails, HITL, audit trail

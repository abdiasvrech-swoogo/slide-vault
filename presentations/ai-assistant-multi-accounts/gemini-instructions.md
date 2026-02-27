# Gemini Canvas — Multi-Account Switching Presentation

Generate a standalone HTML presentation file (`index.html`). Single file, no external dependencies except CDN links listed below.

## Tech Stack (match exactly)

```html
<script src="https://unpkg.com/react@18/umd/react.development.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap" rel="stylesheet">
```

## Brand & Style System

- **Font:** Inter (sans-serif)
- **Background:** `#F9FAFB`
- **Main text:** `#374151`
- **Gradient accent:** `linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%)` (Electric Blue to Violet)
- **Cards:** white bg, `border-radius: 16px`, subtle drop shadows, `border: 1px solid #e5e7eb`
- **Swoogo Logo URL:** `https://swoogo.events/wp-content/themes/swoogo/images/logos/new-swoogo-logo-orange.svg?1770251116`
- **Favicon:** `<link rel="icon" type="image/png" href="../../assets/logo/logo-icon.png" />`

## Responsive & Embed Requirements (CRITICAL)

This presentation MUST work correctly in three contexts:
1. **Desktop browser** (full screen)
2. **Mobile browser** (320px-768px width)
3. **Embedded iframe** (Notion embed, ~700px wide, variable height)

Rules:
- Use `min-h-screen` instead of `h-screen` on the slide container so it doesn't clip in iframes
- Use `min-h-[100dvh]` as the primary height unit (dynamic viewport height)
- Add `overflow-y: auto` on the slide container so content scrolls if it overflows in small viewports
- Cards and grids: `grid-cols-1` on mobile, `md:grid-cols-2` or `md:grid-cols-3` on desktop
- Font sizes: use responsive classes (`text-3xl md:text-5xl`, `text-lg md:text-xl`)
- Padding: `p-4 md:p-8 lg:p-12`
- Navigation buttons: large enough for touch targets (`min-w-[44px] min-h-[44px]`)
- Add touch/swipe support for mobile navigation (touchstart/touchend events)
- Hide scrollbar: `::-webkit-scrollbar { display: none; }` and `scrollbar-width: none`

## Meta Tags

```html
<title>Multi-Account Switching — Swoogo AI Assistant</title>
<meta name="description" content="Architecture decision for multi-account switching in the Swoogo AI Assistant. Comparison with Notion MCP, design decisions, and implementation plan." />
<meta property="og:title" content="Multi-Account Switching — Swoogo AI Assistant" />
<meta property="og:description" content="Architecture decision for multi-account switching. In-session account switching with preserved conversational context." />
<meta property="og:image" content="https://abdiasvrech-swoogo.github.io/slide-vault/presentations/ai-assistant-multi-accounts/image.svg" />
<meta property="og:url" content="https://abdiasvrech-swoogo.github.io/slide-vault/presentations/ai-assistant-multi-accounts/" />
```

Include Twitter card meta tags mirroring OG tags.

## Slide Architecture

Build the presentation as a React app with:
- A `slides` data array (each slide is a JSON object with `id`, `layout`, and content fields)
- Layout components: `TitleSlide`, `BulletSlide`, `ComparisonSlide`, `TriadSlide`, `ImpactSlide`, `DiagramSlide`, `ProcessSlide`, `FinalSlide`
- A `SlideContainer` wrapper with: header (logo + context label), main content area, footer (slide counter + nav buttons)
- Keyboard navigation (ArrowLeft/ArrowRight)
- Touch/swipe navigation for mobile
- Fade-in animation on slide transitions (`fadeIn 0.4s ease-out`)
- All icons built inline as SVG components (no icon library imports)

## Header/Footer Pattern

Header: Swoogo logo (left), context label "AI ASSISTANT • Abdias" (right, small caps, gray)
Footer: "N / Total" counter (left), prev/next arrow buttons (right)

## Slide Content (10 slides)

### Slide 1 — Title (layout: "title")
- Title: "Multi-Account Switching"
- Subtitle: "Architecture Decision"
- Meta badge: "Swoogo AI Assistant — In-Session Context Preservation"
- Icon: a "users/switch" style icon

### Slide 2 — The Problem (layout: "bullets")
- Title: "The Problem"
- Bullets:
  - "Agency users and consultants manage multiple Swoogo accounts"
  - "Current flow: logout → re-authenticate → lose conversation context"
  - "Each user has multiple consumer_key/secret pairs — one per account"
- Insight card: "Switching accounts should be as simple as switching tabs."
- Visual: abstract chart graphic

### Slide 3 — Notion as Reference (layout: "comparison")
This is a NEW layout type: two-column comparison table.
- Title: "Notion MCP — The Reference Model"
- Left column header: "How Notion Does It"
  - Bot-per-workspace model (OAuth → bot_id + workspace_id)
  - One MCP connection per workspace
  - Switching = changing agent in client UI
  - Context lost on workspace switch
- Right column header: "Our Approach"
  - Credential-per-account model (consumer_key/secret)
  - Single MCP session, multiple accounts
  - Switching = mid-session tool call
  - Context preserved across switches
- Bottom insight: "Same middleware pattern. Different switching model. Ours preserves context."

### Slide 4 — The Comparison Table (layout: "comparison")
- Title: "Head-to-Head"
- Table/card grid comparing key aspects:
  - Auth protocol: OAuth 2.0 vs client_credentials
  - Multi-account: Multiple bot tokens vs multiple key pairs
  - Switching: New connection vs mid-session tool
  - Context: Lost vs Preserved
  - Cross-account ops: Impossible vs Possible
- Highlight "Preserved" and "Possible" with green accent

### Slide 5 — Design Decisions (layout: "triad")
- Title: "Key Design Decisions"
- Three cards:
  1. "No API Changes" — Zero changes to Swoogo's OAuth response. consumer_key is the bridge. If we need user identity later: GET /users/me.
  2. "User-Provided Labels" — User names accounts at auth time. Auto-fallback to "Account N". Upgradeable to API-provided account_name later.
  3. "No Identity Replication" — Swoogo PRO owns users and accounts. The middleware delegates, never replicates.

### Slide 6 — Two Auth Layers (layout: "diagram")
- Title: "Two Layers of Authentication"
- Visual diagram showing:
  - Layer 1: TRANSPORT — MCP OAuth (Claude Desktop ↔ our server). Tokens: access_token 1h + refresh_token 30d.
  - Layer 2: ACTIVE ACCOUNT — Session state. sessionAccounts[sessionId].activeTenantId
  - Arrow showing "Multi-account switching operates in Layer 2 only"
- Insight: "The client is authenticated once. Account switching is invisible to it."

### Slide 7 — UX Flow (layout: "process")
- Title: "The User Experience"
- Steps:
  1. "Session Start" — OAuth with credentials A → Account A active
  2. "Add Account" — swoogo-auth-add-account → validate → label → session has [A, B]
  3. "Switch" — swoogo-auth-switch-account → active = A (context preserved)
  4. "List" — swoogo-auth-list-accounts → shows all with active indicator

### Slide 8 — The Three Tools (layout: "metrics" / cards)
- Title: "Three New System Tools (Tier 0)"
- Three cards:
  1. "swoogo-auth-add-account" — Input: clientId, clientSecret, label? → Validates → provisions tenant → adds to session → switches
  2. "swoogo-auth-switch-account" — Input: accountId → Switches active tenant (only among authenticated ones)
  3. "swoogo-auth-list-accounts" — Input: none → Returns accounts with labels + active indicator

### Slide 9 — Implementation Impact (layout: "impact")
- Title: "Implementation Impact"
- Big text: "Zero migrations."
- Sub text: "No new tables. No schema changes. No changes to Swoogo PRO's API."
- Footer badge: "Existing tenants table + Redis key namespacing = already isolated"

### Slide 10 — Takeaway (layout: "final")
- Title text (two lines):
  "One session. Multiple accounts."
  "Zero context loss."
- Sub text: "Authentication is solved. Account switching is a UX problem, not an infrastructure problem."
- Icon: gradient circle with lock/shield icon

## Tone & Writing Rules

- North American English. Senior Engineer to Senior Engineer.
- No "AI-speak": avoid "In conclusion," "It is important to note," "This demonstrates."
- Declarative and direct. Short punchy sentences.
- Use concrete technical terms (consumer_key, tenant_id, Redis blocklist), not vague abstractions.

## Final Checklist

- [ ] Single standalone HTML file
- [ ] All CDN dependencies listed above
- [ ] Responsive: works at 320px, 768px, 1440px
- [ ] Works in Notion iframe embed (no h-screen clipping, scrollable)
- [ ] Touch swipe navigation
- [ ] Keyboard navigation (ArrowLeft/ArrowRight)
- [ ] Swoogo logo in header
- [ ] Fade-in slide transitions
- [ ] All icons are inline SVG components
- [ ] No external icon libraries
- [ ] OG + Twitter meta tags with image.svg reference

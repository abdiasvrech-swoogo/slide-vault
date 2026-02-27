# Swoogo — Presentation Master Prompt

You are a Senior Visual Designer & Technical Storyteller. Generate a standalone HTML presentation following these exact specifications.

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
│   └── {slug}/
│       ├── index.html          ← THE FILE YOU GENERATE GOES HERE
│       ├── image.svg           # Social preview source (editable)
│       └── image.png           # Social preview (exported, 1200x630)
└── ...
```

The HTML file lives at `presentations/{slug}/index.html`. Include a `<base>` tag in the `<head>` to resolve all paths from the project root:

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

**Layout Architecture (mandatory — makes embeds work):**

```
html, body { height: 100%; overflow: hidden; margin: 0; }

┌─── root (h-screen, flex-col) ────────────────────┐
│  header        (shrink-0)                         │
│  main.deck     (flex-1, min-h-0, flex-col)        │
│  ├ progress    (shrink-0, 1.5px bar)              │
│  ├ content     (flex-1, overflow-y-auto, min-h-0) │  ← only this scrolls
│  └ nav-bar     (shrink-0, bottom of deck)         │
│  footer-hint   (shrink-0, desktop only)           │
└───────────────────────────────────────────────────┘
```

Critical rules:

- `html, body`: `overflow: hidden; height: 100%`
- Root wrapper: `h-screen flex flex-col`
- Deck: `flex-1 min-h-0 flex flex-col` (`min-h-0` is critical for flex overflow)
- Only slide content area has `overflow-y-auto`
- Nav controls inside deck, at bottom, `shrink-0`
- **No `position: fixed`** — breaks in iframes
- `overscroll-behavior: none` on body (prevents pull-to-refresh in embeds)
- `touch-action: pan-y` on swipe zone

**Metadata — include at top of file (before `<!DOCTYPE>`):**

```html
<!--
  @project:         Swoogo
  @title:           {presentation_title}
  @slug:            {slug}
  @version:         1.0.0
  @author:          {author}
  @date:            {YYYY-MM-DD}
  @context:         {one-line description}
  @audience:        {target audience}
  @confidentiality: INTERNAL
  @logo:            assets/logo
-->
```

**HTML head — include all of these:**

```html
<meta charset="UTF-8" />
<meta
  name="viewport"
  content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"
/>
<meta name="description" content="{context}" />
<meta name="author" content="{author}" />
<title>{title} — {author}</title>
<link rel="icon" type="image/png" href="assets/logo/logo-icon.png" />
<link rel="apple-touch-icon" href="assets/logo/logo-icon.png" />
<!-- OG tags: og:type, og:url, og:title, og:description, og:image -->
<!-- Twitter tags: twitter:card, twitter:url, twitter:title, twitter:description, twitter:image -->
```

---

## Part 6: Color Quick Reference

```
bg-swoogo-lavender / text-swoogo-lavender / border-swoogo-lavender  → #726BEA (primary accent)
bg-swoogo-pacific / text-swoogo-pacific / border-swoogo-pacific      → #262966 (deep accent / dark panels / primary text)
bg-swoogo-mist / text-swoogo-lavender                                → #E9ECFF bg + #726BEA text (badges)
bg-swoogo-chalk                                                      → #F5F5FA (subtle surface)
bg-swoogo-snow                                                       → #F9FAFB (page background)
bg-swoogo-flame / text-swoogo-flame                                  → #F46A34 (orange accent / highlights)
bg-swoogo-glow                                                       → #EE8252 (warm accent)
bg-swoogo-teal / text-swoogo-teal                                    → #4BCDB5 (teal accent)
bg-swoogo-bermuda                                                    → #A0F1E2 (light teal)

Gradient:  linear-gradient(135deg, #726BEA, #262966)
Success:   bg-emerald-100  text-emerald-700  border-emerald-500
Warning:   bg-amber-100    text-amber-700    border-amber-500
Danger:    bg-red-100      text-red-700      border-red-500
Info:      bg-blue-100     text-blue-700     border-blue-500
```

---

## How to Use

This template is combined with a presentation's `README.md` content brief by the **prepare-gemini** skill, which outputs a single `gemini-instructions.md` file inside `presentations/{slug}/`. Copy the full content into Gemini Canvas — it already includes everything (instructions + language + content). No need to attach the README separately.

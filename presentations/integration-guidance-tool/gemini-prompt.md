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
│   └── integration-guidance-tool/
│       ├── index.html          ← THE FILE YOU GENERATE GOES HERE
│       ├── image.svg           # Social preview source (editable)
│       └── image.png           # Social preview (exported, 1200x630)
└── ...
```

The HTML file lives at `presentations/integration-guidance-tool/index.html`. Include a `<base>` tag in the `<head>` to resolve all paths from the project root:

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

## Part 6: Presentation Content — Integration Guidance Tool

**Presentation Title:** Integration Guidance — De prompts simples a formularios correctos

**Subtitle:** Cómo el MCP de Swoogo resuelve el problema de las guías de integración

**Target Audience:** Stakeholders internos, ingenieros senior

---

## Detailed Content Structure:

### SLIDE 1: Cover (type: `cover`)
**Title:** Integration Guidance  
**Subtitle:** De prompts simples a formularios correctos  
**Icon:** `fa-route` (representando guía/camino)  
**Badges:**
- "MCP Server"
- "Token Efficiency"
- "v1.1.0"

---

### SLIDE 2: El Problema (type: `triad`)
**Title:** El problema de las guías monolíticas

**Three concepts:**
1. **Desperdicio de tokens**
   - 29% de tokens innecesarios
   - En 15 turnos: ~19,500 tokens irrelevantes acumulados

2. **Contaminación de contexto**
   - LLM recibe info de sessions, speakers, reporting
   - Solo necesitaba registration
   - Resultado: alucinación de endpoints

3. **Loops de corrección**
   - Código falla por endpoint incorrecto
   - Usuario reporta → LLM regenera
   - 3,000-5,000 tokens por loop

---

### SLIDE 3: El Impacto (type: `impact`)
**Statement:** ~30,000 tokens ahorrados por sesión

**Sub-text:** Entrega + contexto persistente + calidad downstream  
(Use oversized numbers with gradient text for the 30,000)

---

### SLIDE 4: La Solución (type: `bullets`)
**Title:** swoogo-integration-guide

**Bullets:**
- Guías composables por scope — el host recibe solo lo relevante
- Protocolo en dos fases: discovery → targeted
- Cero alucinación de endpoints — la guía es la fuente de verdad
- Determinismo garantizado — misma entrada → misma salida

**Insight card:** "Contenido estático, pre-autorizado, versionado en código fuente"

---

### SLIDE 5: Arquitectura Composable (type: `architecture`)
**Title:** Capas de contenido por scope

**Diagram structure (vertical layers):**
```
┌─────────────────────────────────┐
│   CORE (siempre incluido)      │
│ Arquitectura · Auth · Security │
│ Validación · Pitfalls          │
└─────────────┬───────────────────┘
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
┌──────────┐┌──────────┐┌──────────┐
│ DOMAIN   ││ DOMAIN   ││ DOMAIN   │
│registrat.││ sessions ││ speakers │
└────┬─────┘└──────────┘└──────────┘
     │
┌────────────────────────────────┐
│  CROSS-CUTTING (condicional)   │
│ i18n · Caching · Fees · UX     │
└──────────────┬─────────────────┘
               ▼
┌──────────────────────────────┐
│ CHECKLIST (adapta al scope)  │
└──────────────────────────────┘
```

**Note:** 17 secciones fijas — contrato predecible

---

### SLIDE 6: Flujo Act + Ask + Expand (type: `process`)
**Title:** De prompt simple a formulario completo

**3 Steps (sequential vertical flow):**

**1. ACT** — Identifica el scope primario y pídelo
- Infiere scope del mensaje del usuario
- Llama swoogo-integration-guide con scope específico
- Usa discovery tools para inspeccionar el evento real

**2. ASK** — Sugiere next steps basados en data real
- "El evento tiene reg-types con precio. ¿Incluyo pagos?"
- "Hay campos sq_* de sessions. ¿Incluyo scheduling?"
- Preguntas contextuales, no genéricas

**3. EXPAND** — Pide scopes adicionales uno a uno
- Usuario confirma → pide scope adicional
- Solo lo necesario, incremental
- Evita contaminación de contexto

---

### SLIDE 7: Experiencia del Usuario (type: `comparison`)
**Title:** Antes vs. Después

**Left side (Antes — without guidance):**
- Usuario: "Generame un Registration Form"
- LLM inventa endpoints
- Código falla
- 3-5 loops de corrección
- ~67,500 tokens acumulados en 15 turnos

**Right side (Después — with guidance):**
- Usuario: "Generame el Registration Form del Evento 12345"
- LLM llama swoogo-integration-guide
- Recibe guía exacta (~3,200 tokens)
- Inspecciona evento con discovery tools
- Genera código correcto al primer intento
- ~48,000 tokens acumulados en 15 turnos

**Savings badge:** 29% de ahorro

---

### SLIDE 8: Dominios Soportados (type: `features`)
**Title:** 5 dominios, composables

**Grid of 5 cards with icons:**
1. **Registration** (`fa-user-plus`)
   - Registrants, questions, reg-types, packages

2. **Sessions** (`fa-calendar-days`)
   - Sessions, attendance, tracks

3. **Speakers** (`fa-microphone`)
   - Speaker directory

4. **Contacts** (`fa-address-book`)
   - CRM, GDPR

5. **Reporting** (`fa-chart-line`)
   - Transacciones, métricas

---

### SLIDE 9: Contrato de 17 Secciones (type: `metrics`)
**Title:** Estructura determinista

**6 metric cards (2x3 grid):**
1. **17** Secciones fijas
2. **CORE** Arquitectura, auth, seguridad
3. **DOMAIN** Endpoints, payloads, relaciones
4. **CROSS-CUTTING** i18n, fees, caching, UX
5. **100%** Cobertura de testing
6. **v1.1.0** Versionado semver

---

### SLIDE 10: Eficiencia en Tokens (type: `metrics`)
**Title:** Ahorro medible

**3 metric cards with breakdown:**

**Card 1: Entrega**
- Registration form: **29%** ahorro
- Sessions only: **44%** ahorro
- Vs. guía completa no filtrada

**Card 2: Contexto persistente**
- 15 turnos con guía completa: 67,500 tokens
- 15 turnos composable: 48,000 tokens
- **19,500** tokens ahorrados

**Card 3: Calidad downstream**
- Menos alucinaciones
- Menos llamadas especulativas
- Menos loops de corrección
- **~9,850** tokens ahorrados

**Total:** ~30,650 tokens por sesión

---

### SLIDE 11: Decisiones Técnicas (type: `bullets`)
**Title:** Patrones y estándares

**Bullets:**
- Contenido estático — cero generación dinámica, determinismo garantizado
- Ensamblaje mecánico — selección por scope, sin lógica condicional compleja
- Seguridad por diseño — no se exponen secretos ni paths internos

**Insight card:** "65+ tests, snapshots por combinación scope × modo"

---

### SLIDE 12: Hallazgos en Testing Real (type: `bullets`)
**Title:** v1.1.0 — Content fixes

**Context:** Al probar con un Registration Form real, 8 errores descubiertos

**Bullets (pick 3 most impactful):**
- Upload fields usan `{attribute}` no `{fieldId}` en el endpoint
- `conditional_prices` es polimórfico — PHP json_encode quirk
- Session→question mapping gap (preguntas `sq_*` vinculan sessions)

**Insight card:** "Additive, non-breaking — minor version bump"

---

### SLIDE 13: Estado Actual (type: `roadmap`)
**Title:** Implementado y funcional

**Timeline with 3 phases:**

**Phase 1 — Core (✓ Completo)**
- Tool definition
- 5 dominios
- Protocolo discovery + targeted
- Tests unitarios (65+)

**Phase 2 — Content Fixes (✓ v1.1.0)**
- 8 content patches
- Validación con evento real
- Snapshot tests actualizados

**Phase 3 — Próximos pasos**
- Validación con hosts reales (Claude Desktop, Cursor)
- Feedback loop basado en uso
- Nuevos dominios según demanda

---

### SLIDE 14: Objetivos Logrados (type: `metrics`)
**Title:** 6 objetivos, 6 logrados

**Grid of 6 status cards (checkmark badges):**
1. ✓ Construir Registration Form con un solo prompt
2. ✓ Filtrar contenido por scope
3. ✓ Determinismo garantizado
4. ✓ Prevenir filtración de secretos
5. ✓ Soportar modo API y modo CLI
6. ✓ Discovery de dominios sin documentación externa

---

### SLIDE 15: Ejemplo Práctico (type: `process`)
**Title:** Prompt → Código funcional

**Sequential flow:**

**Step 1:** Usuario escribe
```
"Generame el Registration Form del Evento 12345 
utilizando el MCP de Swoogo"
```

**Step 2:** Host LLM
- Infiere scope: ["registration"]
- Llama swoogo-integration-guide
- Recibe ~3,200 tokens de guía técnica
- Inspecciona evento: events, event-questions, reg-types

**Step 3:** Host sugiere
- "El evento tiene reg-types con precios. ¿Incluyo pagos?"
- Usuario confirma → pide scope: ["sessions"]

**Step 4:** Genera código funcional
- Registration Form completo
- Campos del evento real
- Sin alucinaciones

---

### SLIDE 16: Valor Clave (type: `impact`)
**Statement:** Cero alucinación de endpoints

**Sub-text:** La guía es la fuente de verdad, no la memoria del LLM

---

### SLIDE 17: Final Takeaway (type: `final`)
**Statement:** De prompts simples a formularios correctos — composable, determinista, eficiente

**Icon:** `fa-check-circle` with gradient  
**Sub-text:** swoogo-integration-guide · v1.1.0

---

## Part 7: Additional Implementation Notes

**Visual treatment for metrics:**
- Use `border-b-8` on metric cards with Swoogo brand colors
- Large numbers with gradient text for impact
- Icons from Font Awesome 6

**Code blocks:**
- Use `bg-slate-50` for code examples
- Syntax highlight with appropriate colors from the palette
- Keep examples short (max 10 lines visible)

**Architecture diagram (Slide 5):**
- Use cards with arrows showing flow
- Core at top (always included)
- Domains in middle (selectable)
- Cross-cutting below domains (conditional)
- Checklist at bottom (adaptive)

**Process diagram (Slide 6):**
- Vertical numbered steps
- Each step has an icon
- Use Swoogo lavender for step numbers
- Include specific examples in smaller text

**Comparison slide (Slide 7):**
- Two columns side by side
- Left (before): use red/warning tones
- Right (after): use success/green tones
- Visual differentiation with icons

**Ensure all slides:**
- Have consistent header with logo
- Include slide numbers
- Use proper responsive classes
- Maintain visual hierarchy
- Include navigation controls
- Have proper touch targets (min 40x40px)

---

## Critical Requirements:

1. **DO NOT summarize or simplify** — use all the details provided above
2. **Follow the exact slide structure** specified (17 slides)
3. **Use the exact color palette** from the brand tokens
4. **Include all numerical metrics** as specified (30,000 tokens, 29%, 19,500, etc.)
5. **Use proper icon classes** from Font Awesome 6
6. **Maintain the narrative arc** from problem → solution → evidence → value
7. **Include code examples** where specified (user prompts, responses)
8. **Use proper slide types** as indicated for each slide
9. **Follow typography rules** (Inter font, weights, gradient text)
10. **Implement full navigation** (arrows, swipe, progress bar, slide counter)

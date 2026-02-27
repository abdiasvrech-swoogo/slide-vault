# Integration Guidance — De prompts simples a formularios correctos

> Documento interno para presentación a stakeholders.

---

## 1. Resumen ejecutivo

El MCP de Swoogo incluye una herramienta llamada `swoogo-integration-guide` que permite a cualquier host LLM (Claude Desktop, Cursor, Gemini CLI, etc.) construir **Registration Forms correctos** contra la API de Swoogo, sin que el usuario final necesite conocer endpoints, payloads ni patrones de autenticación.

El usuario simplemente escribe:

> _"Generame el Registration Form del Evento 12345 utilizando el MCP de Swoogo"_

Y el host LLM recibe, bajo demanda, una guía técnica precisa y **filtrada al alcance exacto que necesita** — eliminando ruido, ahorrando tokens y previniendo errores de implementación.

**Impacto clave:**
- **~30,000 tokens ahorrados por sesión** (entrega + contexto persistente + calidad downstream)
- **Cero alucinación de endpoints** — la guía es la fuente de verdad, no la memoria del LLM
- **Determinismo garantizado** — misma entrada produce misma salida, siempre

---

## 2. Objetivos

| # | Objetivo | Estado |
|---|----------|:------:|
| 1 | Que un host LLM pueda construir un Registration Form correcto con un solo prompt | Logrado |
| 2 | Filtrar contenido por alcance (scope) para entregar solo lo relevante | Logrado |
| 3 | Garantizar determinismo (misma entrada → misma salida) | Logrado |
| 4 | Prevenir filtración de secretos o infraestructura interna en la guía | Logrado |
| 5 | Soportar dos modos de integración (API directo y CLI/MCP) | Logrado |
| 6 | Que el host pueda descubrir qué dominios existen sin documentación externa | Logrado |

---

## 3. Alcance y no-alcance

### En alcance

- Herramienta `swoogo-integration-guide` (Tier 0, auto-ejecutable)
- 5 dominios: **registration**, sessions, speakers, contacts, reporting
- Protocolo de descubrimiento en dos fases (discovery → targeted)
- Contenido estático, pre-autorizado, versionado en código fuente
- Ensamblador mecánico que compone guías por scope
- 17 secciones fijas con contrato determinista
- Inclusión automática de secciones cross-cutting (fees, i18n, UX)
- Modo API (default) y modo CLI (condicional, requiere LLM configurado)
- Tests exhaustivos (65+ tests, snapshots por combinación de scope x modo)

### Fuera de alcance

- Generación dinámica de contenido (no se usa LLM para ensamblar la guía)
- Ejecución real de la creación de formularios (la guía instruye al host; el host ejecuta)
- Nuevos dominios más allá de los 5 definidos (extensible, pero no en este ciclo)

---

## 4. El problema que resuelve

Cuando un host LLM (Claude Desktop, Cursor, etc.) necesita construir algo contra la API de Swoogo, enfrenta un problema fundamental: **no conoce la API**. Sin una guía, el LLM inventa endpoints, mezcla patrones de distintas entidades, y genera código que no funciona.

La solución obvia — entregar una guía completa de toda la API — introduce nuevos problemas:

| Problema | Consecuencia |
|----------|-------------|
| **Desperdicio de tokens** | Un host construyendo solo un Registration Form recibe contenido de sessions, speakers, contacts y reporting — 29% de tokens innecesarios |
| **Contaminación de contexto** | La guía persiste en el contexto del LLM toda la sesión. En 15 turnos: ~19,500 tokens acumulados de información irrelevante |
| **Alucinación de endpoints** | El LLM con exceso de información mezcla endpoints de distintas entidades |
| **Llamadas especulativas** | El LLM hace tool calls "por las dudas" ("también voy a traer speakers") — tokens y tiempo desperdiciados |
| **Loops de corrección** | Código fallido por endpoint incorrecto → usuario reporta → LLM regenera → 3,000-5,000 tokens por loop |

`swoogo-integration-guide` resuelve esto con **guías composables por scope**: el host recibe solo el contenido que necesita, en un contrato predecible de 17 secciones.

```
┌─────────────────────────────────────┐
│       CORE (siempre incluido)       │
│  Arquitectura · Auth · Seguridad    │
│  Validación · Pitfalls (~80 líneas) │
└──────────────┬──────────────────────┘
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
┌──────────┐┌──────────┐┌──────────┐
│ DOMAIN   ││ DOMAIN   ││ DOMAIN   │  ... (5 scopes disponibles)
│registrat.││ sessions ││ speakers │
└──────────┘└──────────┘└──────────┘
       │
┌──────────────────────────────────────┐
│   CROSS-CUTTING (condicional)        │
│  i18n · Caching · Fees · UX · CLI   │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│      CHECKLIST (adapta al scope)     │
└──────────────────────────────────────┘
```

---

## 5. Arquitectura y flujo

### 5.1 Approach híbrido: Act + Ask

El MCP Server instruye al host LLM con un flujo de 3 pasos que combina acción inmediata con expansión guiada:

**Paso 1 — ACT**: Identifica el scope primario de la petición del usuario y pídelo. Usa las discovery tools (`swoogo-events`, `swoogo-event-questions`, `swoogo-reg-types`) para inspeccionar qué tiene el evento.

**Paso 2 — ASK**: Después de recibir la guía y ver la data del evento, sugiere next steps basados en lo que encontró. El discovery response incluye `clarification_prompts` — preguntas contextuales que el host puede usar para guiar al usuario:
- "This event has paid reg-types. Do you need payment integration?"
- "The form has session selection fields. Should I include session scheduling?"
- "There are 3 speaker profiles linked. Do you want a speaker directory page?"

Estos prompts se generan server-side y están disponibles en el discovery response, permitiendo que el host haga preguntas inteligentes basadas en la data real del evento.

**Paso 3 — EXPAND**: Pide scopes adicionales uno a uno conforme el usuario confirma. Esto evita cargar toda la guía de una vez — solo se expande lo que realmente se necesita.

```
Usuario: "Generame el Registration Form del Evento 12345"
                       │
                       ▼
               ┌───────────────┐
               │  1. ACT       │  Host infiere scope: ["registration"]
               │               │  Pide guía + inspecciona el evento
               └───────┬───────┘
                       │
                       ▼
               ┌───────────────┐
               │  2. ASK       │  "El evento tiene reg-types con precio.
               │               │   ¿Necesitas integración de pagos?"
               │               │  "Hay campos de selección de sesiones.
               │               │   ¿Incluyo scheduling?"
               └───────┬───────┘
                       │
                       ▼
               ┌───────────────┐
               │  3. EXPAND    │  Usuario confirma → pide scope: ["sessions"]
               │               │  Solo lo que se necesita, un scope a la vez
               └───────────────┘
```

### 5.2 Protocolo de dos fases

**Fase 1 — Discovery (opcional)**:
```
Host LLM → swoogo-integration-guide { mode: "api" }
         ← { type: "discovery", available_scopes: [...], discovery_hints: [...], clarification_prompts: [...] }
```

El host recibe:
- Lista de dominios disponibles con descripciones (~500 tokens)
- `discovery_hints`: preguntas genéricas para entender la intención del usuario
- `clarification_prompts`: preguntas contextuales para sugerir expansiones después de inspeccionar el evento

**Fase 2 — Guía dirigida**:
```
Host LLM → swoogo-integration-guide { mode: "api", scope: ["registration"] }
         ← { type: "guide", scope: ["registration"], guide: "## Objective\n..." }
```

El host recibe solo lo que necesita (~3,200 tokens para registration). No existe `scope: "all"` — el servidor requiere scopes específicos para forzar el patrón incremental.

**Decisión clave**: El discovery es opcional. Si el host infiere el scope del mensaje del usuario ("Generame el Registration Form..."), puede llamar directamente con `scope: ["registration"]`, ahorrando un round-trip. El flujo Act + Ask guía naturalmente hacia la expansión incremental.

### 5.3 Flujo completo del usuario

```
┌──────────────────────────────────────────────────────────────────┐
│ USUARIO                                                          │
│ "Generame el Registration Form del Evento 12345                  │
│  utilizando el MCP de Swoogo"                                    │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│ HOST LLM (Claude Desktop / Cursor)                               │
│                                                                  │
│ ACT:                                                             │
│ 1. Infiere scope del mensaje → ["registration"]                  │
│ 2. Llama swoogo-integration-guide { scope: ["registration"] }    │
│ 3. Recibe guía técnica (~3,200 tokens):                          │
│    - Arquitectura (backend proxy, OAuth)                         │
│    - Endpoints exactos (GET /events/{id}.json, GET /event-       │
│      questions, POST /registrants.json)                          │
│    - Payload structure (flat body, mixed types)                  │
│    - Relaciones entre entidades                                  │
│    - Validación server-authoritative                             │
│    - Pitfalls ("No usar /registrants sin .json suffix")          │
│    - Checklist paso a paso                                       │
│ 4. Inspecciona el evento con tools de discovery:                 │
│    a) swoogo-events { id: 12345 } → metadata                    │
│    b) swoogo-event-questions { event_id: 12345 } → campos       │
│    c) swoogo-reg-types { event_id: 12345 } → tipos + precios    │
│                                                                  │
│ ASK:                                                             │
│ 5. Analiza la data y sugiere next steps:                         │
│    "El evento tiene 3 reg-types con precios. ¿Incluyo pagos?"   │
│    "Hay campos sq_* (session selection). ¿Incluyo sesiones?"     │
│                                                                  │
│ EXPAND:                                                          │
│ 6. Usuario confirma sesiones → pide scope: ["sessions"]          │
│ 7. Genera formulario completo con todos los datos reales         │
│                                                                  │
│ 8. Presenta al usuario el Registration Form funcional            │
└──────────────────────────────────────────────────────────────────┘
```

### 5.4 Instrucciones del MCP Server

El MCP Server incluye en sus `instructions` (visible para el host LLM al conectarse) directivas explícitas que guían el comportamiento Act + Ask:

1. **ACT** — Identifica el scope primario y pídelo. Usa discovery tools para inspeccionar el evento
2. **ASK** — Con la guía y la data del evento, sugiere next steps basados en lo que encontró
3. **EXPAND** — Pide scopes adicionales uno a uno conforme el usuario confirma
4. **Preguntar** al usuario su stack preferido antes de generar código
5. **Solicitar credenciales** reales — nunca usar placeholders
6. Si se une a un proyecto existente → leer archivos de integración antes de modificar

Estas instrucciones garantizan que el host LLM no improvise ni cargue todo de una vez: primero obtiene la guía mínima, inspecciona datos reales, sugiere expansiones, y genera código informado.

### 5.4 Contrato de 17 secciones

Toda guía ensamblada contiene exactamente estas secciones, en este orden:

| # | Sección | Tipo | Siempre presente |
|---|---------|------|:---:|
| 1 | Objective | Generado | Si |
| 2 | Recommended Architecture | CORE | Si |
| 3 | Authentication & Token Lifecycle | CORE | Si |
| 4 | Data Fetching Strategy | DOMAIN | Si* |
| 5 | Entity Relationships | DOMAIN | Si* |
| 6 | Endpoint Reference | DOMAIN | Si* |
| 7 | Dynamic Fields & Form Handling | DOMAIN | Si* |
| 8 | Write Operations & Payload Structure | DOMAIN | Si* |
| 9 | Validation & Error Handling | CORE + DOMAIN | Si |
| 10 | Fees & Pricing | Cross-cutting | Si* |
| 11 | Internationalization (i18n) | Cross-cutting | Si* |
| 12 | UX & Accessibility | Cross-cutting | Si* |
| 13 | Caching Strategy | Cross-cutting | Si |
| 14 | Security Considerations | CORE | Si |
| 15 | CLI Mode | Cross-cutting | Si* |
| 16 | Common Pitfalls | CORE + DOMAIN | Si |
| 17 | Final Implementation Checklist | Adapta al scope | Si |

\* Si no aplica al scope seleccionado, la sección dice "Not applicable for the selected scopes."

### 5.5 Reglas de inclusión automática cross-cutting

Algunas secciones se incluyen automáticamente según el scope:

| Sección | Se incluye si el scope contiene |
|---------|-------------------------------|
| Fees & Pricing | registration, sessions, reporting |
| i18n | registration, contacts |
| UX & Accessibility | registration, sessions, speakers |

Si el host pide `scope: ["registration"]`, automáticamente recibe Fees, i18n y UX — sin pedirlos explícitamente.

### 5.6 Estructura de archivos

```
src/prompts/integration-guidance/
├── index.ts                     # Ensamblador principal
├── types.ts                     # Tipos: Mode, DomainScope, Section, etc.
├── scopes.ts                    # Definiciones de scopes y hints
├── version.ts                   # Versionado semver (1.1.0)
├── checklist.ts                 # Checklist adaptativo por scope + modo
├── core/
│   ├── architecture.ts          # Patrón backend proxy / HTTP-SSE
│   ├── authentication.ts        # OAuth2 client_credentials
│   ├── validation.ts            # Validación server-authoritative
│   ├── security.ts              # Secrets, XSS, rate limiting
│   └── pitfalls.ts              # Errores comunes globales
├── domains/
│   ├── registration.ts          # Registrants, questions, reg-types, packages
│   ├── sessions.ts              # Sessions, attendance, tracks
│   ├── speakers.ts              # Speaker directory
│   ├── contacts.ts              # CRM, GDPR
│   └── reporting.ts             # Transacciones, métricas
└── cross-cutting/
    ├── fees-pricing.ts          # Tarifas y descuentos
    ├── i18n.ts                  # Internacionalización
    ├── ux-accessibility.ts      # Accesibilidad
    ├── caching.ts               # Estrategia de caché
    └── cli-mode.ts              # Modo CLI vs API
```

---

## 6. Experiencia del usuario — Ejemplos de prompts

### Ejemplo 1: Registration Form basico
```
Usuario: "Necesito crear un formulario de registro para el evento 12345"

→ El host llama swoogo-integration-guide { scope: ["registration"] }
→ Recibe guía de ~3,200 tokens (solo registration)
→ Llama las herramientas correctas: events, event-questions, reg-types
→ Genera un formulario funcional con los campos del evento
```

### Ejemplo 2: Registration Form con seleccion de sesiones
```
Usuario: "Quiero un formulario de registro donde el asistente pueda elegir sesiones"

→ El host infiere dos scopes: registration + sessions
→ Llama swoogo-integration-guide { scope: ["registration", "sessions"] }
→ Guía incluye: endpoints de registration + sessions + fees (auto-incluido)
→ Genera formulario con pasos: datos personales → selección de sesiones
```

### Ejemplo 3: Descubrimiento
```
Usuario: "Quiero integrar Swoogo en mi app"

→ El host no puede inferir un scope específico
→ Llama swoogo-integration-guide { } (sin scope = discovery)
→ Recibe lista de 5 dominios disponibles con descripciones
→ Le pregunta al usuario qué necesita específicamente
→ Luego llama con el scope correcto
```

### Ejemplo 4: Modo CLI
```
Usuario: "Quiero interactuar con Swoogo a través del asistente AI"

→ El host detecta que el usuario quiere el modo conversacional
→ Llama swoogo-integration-guide { mode: "cli", scope: ["registration"] }
→ Guía incluye: /api/auth/authenticate, /api/chat con SSE, HITL
→ Si no hay LLM configurado → fallback a modo API con notice explicativo
```

---

## 7. Patrones y estandares respetados

| Patrón | Implementación |
|--------|---------------|
| **Contenido estático** | Toda la guía es texto pre-autorizado en source control. Cero generación dinámica |
| **Ensamblaje mecánico** | Selección por scope, concatenación. Sin lógica condicional compleja |
| **Determinismo** | Misma entrada → misma salida. Enforced por tests de snapshot |
| **Seguridad por diseño** | No se exponen secretos, clases internas, paths, ni hostnames internos |
| **Tolerancia a errores** | Scopes inválidos no rompen la request — se ignoran con warnings |
| **Versionado semver** | `version: "1.1.0"` en toda respuesta. Breaking = major, aditivo = minor |
| **Tests de contrato** | 17 secciones verificadas en orden, sin duplicados, sin contenido prohibido |
| **Tool definition estándar** | Usa `defineTool()` del proyecto (Zod schema, tier, risk level, handler) |
| **Naming convention** | `swoogo-integration-guide` — kebab-case con prefijo `swoogo-` |
| **Compact JSON output** | `guideResult()` serializa sin pretty-print para mantenerse dentro de límites de respuesta |
| **Observabilidad** | Cada request logea: type, mode, requestedScopes, resolvedScopes, warnings, guideTokenEstimate, tenantId |

---

## 8. Principales decisiones y trade-offs

### Decision 1: Una herramienta con scope vs. múltiples herramientas
**Elegido**: Una herramienta `swoogo-integration-guide` con parámetro `scope`.
**Alternativa descartada**: Herramientas separadas (`swoogo-guide-registration`, `swoogo-guide-sessions`, etc.).
**Razón**: Evita namespace pollution (35+ herramientas en lugar de 29), previene duplicación de contenido core, simplifica el descubrimiento.

### Decision 2: Contenido estático vs. generación dinámica
**Elegido**: Strings pre-autorizados en source control.
**Alternativa descartada**: Que un LLM genere la guía dinámicamente.
**Razón**: Determinismo garantizado. La guía debe ser revisada en PRs, no inventada. Los errores en guías son difíciles de detectar si son dinámicos.

### Decision 3: Composable vs. monolítico
**Elegido**: Capas composables por scope.
**Alternativa descartada**: Una sola guía grande para todos los casos.
**Razón**: Ahorro de tokens medible (~30,000 por sesión), menor contaminación de contexto, mejor calidad de respuesta del LLM.

### Decision 4: Discovery opcional vs. obligatorio
**Elegido**: Discovery es opcional; el host puede llamar directo con scope.
**Alternativa descartada**: Forzar siempre discovery primero.
**Razón**: Si el host infiere "registration" del mensaje del usuario, forzar un paso extra desperdiciaría ~500 tokens y un round-trip.

### Decision 5: Modo API como default
**Elegido**: `mode: "api"` es el default.
**Alternativa descartada**: Detectar automáticamente si usar API o CLI.
**Razón**: API es universalmente aplicable (no requiere LLM server-side). CLI requiere configuración específica y es un caso más limitado.

### Decision 6: Act + Ask vs. entrega monolítica
**Elegido**: Flujo de 3 pasos (ACT → ASK → EXPAND) donde el host pide scopes incrementalmente.
**Alternativa descartada**: `scope: "all"` que entregaba toda la guía de una vez.
**Razón**: "all" causaba exactamente los problemas que las guías composables pretendían resolver — contaminación de contexto, alucinaciones, y llamadas especulativas. El approach incremental fuerza al host a inspeccionar primero la data del evento y expandir solo lo necesario. `scope: "all"` fue eliminado (no deprecado — es v1, nadie lo ha usado).

### Decision 7: Clarification prompts contextuales vs. genéricos
**Elegido**: Prompts basados en lo que el evento realmente tiene ("This event has paid reg-types. Do you need payment integration?").
**Alternativa descartada**: Preguntas genéricas ("¿Qué estás construyendo?").
**Razón**: Los prompts genéricos no agregan valor — el host ya sabe qué pidió el usuario. Los prompts contextuales le permiten al host sugerir expansiones inteligentes basadas en la data real del evento.

### Trade-offs aceptados

| Trade-off | Pro | Contra |
|-----------|-----|--------|
| Contenido estático | Determinismo, auditabilidad | Requiere PR para actualizar información |
| 17 secciones fijas | Contrato predecible, testeable | Menos flexibilidad para casos edge |
| Sin LLM en ensamblaje | Cero costo de inferencia | No puede adaptar tono/nivel de detalle |
| Sin `scope: "all"` | Fuerza el patrón incremental | Host debe hacer múltiples calls para multi-scope |
| Act + Ask en instructions | Host sigue flujo guiado | Depende de que el host respete instructions |

---

## 9. Eficiencia en tokens

### Costo de entrega (la guía en sí)

| Escenario | Sin filtro | Composable | Ahorro |
|-----------|:----------:|:----------:|:------:|
| Registration form | ~4,500 | ~3,200 | 29% |
| Sessions only | ~4,500 | ~2,500 | 44% |
| Registration + sessions | ~4,500 | ~3,800 | 16% |
| Discovery | N/A | ~500 | — |

### Costo de contexto persistente (el multiplicador oculto)

La guía permanece en el contexto del LLM toda la sesión. En 15 turnos:
- Sin filtro: 4,500 x 15 = **67,500 tokens acumulados**
- Composable (registration): 3,200 x 15 = **48,000 tokens acumulados**
- **Ahorro: 19,500 tokens (~29%)**

### Costo downstream (calidad de respuesta)

Contenido irrelevante causa errores medibles:
- Confusión de patrones: el host genera código de sessions cuando solo pidió registration
- Alucinación de endpoints: el LLM mezcla endpoints de distintas entidades
- Llamadas especulativas: tool calls innecesarios
- Loops de corrección: código falla → regenerar

**Ahorro estimado downstream**: ~9,850 tokens por sesión

### Total por sesión

**~30,650 tokens ahorrados** (entrega + contexto + calidad)

A escala: 50 usuarios concurrentes x ~30,650 = **~1.5M tokens ahorrados por ciclo de sesión**

---

## 10. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación |
|--------|---------|-----------|
| API de Swoogo cambia endpoints | Guía queda desactualizada | Contenido versionado; test de snapshot detecta drift |
| Host ignora la guía y alucina endpoints | Formularios rotos | Pitfalls explícitos + schema validation en tools |
| Scope insuficiente (nuevo dominio no cubierto) | Host no recibe guía para ese caso | Arquitectura extensible: agregar domain/ + actualizar scopes.ts |
| LLM del host no sigue instrucciones de la guía | Implementación incorrecta | Checklist paso a paso + secciones cortas y directivas |
| Contenido demasiado largo para context window | Guía truncada | Composable = solo lo necesario; discovery = mínimo overhead |

---

## 11. Hallazgos y problemas resueltos

### Problema 1: Guía completa causaba alucinaciones
**Hallazgo**: Con la guía completa en contexto, Claude Desktop mezclaba endpoints de registrants con session-attendances. El LLM "razonaba" que si estaban en la guía, debía usarlos.
**Solución**: Filtrado por scope — si el host solo necesita registration, no ve endpoints de session-attendance.

### Problema 2: Tokens acumulados en sesiones largas
**Hallazgo**: En sesiones de 15+ turnos, la guía completa acumulaba ~67,500 tokens de contexto innecesario, degradando calidad y aumentando costos.
**Solución**: Guías composables reducen entre 16-44% el contenido entregado.

### Problema 3: Modo CLI sin LLM configurado
**Hallazgo**: Si un host pedía modo CLI pero el server no tenía LLM provider, la request fallaba sin explicación.
**Solución**: Fallback automático a modo API con `notice` estructurado (razón + pasos de resolución + explicación del fallback).

### Problema 4: Scopes inválidos rompían la request
**Hallazgo**: Si el host LLM alucinaba un scope ("unicorns"), la herramienta fallaba.
**Solución**: Tolerancia: scopes inválidos se ignoran con `warnings`, se entregan los válidos.

### Problema 5: Sin forma de descubrir qué scopes existen
**Hallazgo**: Los hosts no sabían qué scopes pedir sin documentación externa.
**Solución**: Protocolo de discovery (llamar sin scope → recibir lista de dominios disponibles).

### Problema 6: Content fixes descubiertos en testing real (v1.1.0)
**Hallazgo**: Al probar con un Registration Form real, se descubrieron 8 errores/omisiones en el contenido de la guía:
1. Upload fields usan `{attribute}` no `{fieldId}` en el endpoint
2. Faltaba guidance de retry y partial failure para writes
3. `conditional_prices` es polimórfico (`[]` vs `{}`) — PHP `json_encode` quirk
4. Sessions-with-fees faltaba en la tabla de data fetching
5. Session→question mapping gap (preguntas `sq_*` vinculan sessions a registrants)
6. Faltaba code example para file upload
7. Proxy multipart forwarding no documentado
8. Self-signed cert note para dev environments

**Solución**: 8 content patches, bump a v1.1.0 (minor — additive, no breaking). Todos verificados con snapshot tests.

---

## 12. Estado actual

| Componente | Estado |
|------------|:------:|
| Tool `swoogo-integration-guide` | Implementado |
| 5 dominios (registration, sessions, speakers, contacts, reporting) | Completo |
| Ensamblador composable | Funcional, determinista |
| Protocolo discovery + targeted | Funcional |
| Modo API + modo CLI | Funcional, con fallback |
| Tests unitarios (65+) | Passing |
| Tests de snapshot (scope x modo) | Passing |
| Tests de seguridad (contenido prohibido) | Passing |
| Documentación de diseño | 876 líneas |
| Versionado | v1.1.0 |

---

## 13. Pendientes

> _Sección para completar manualmente._

- [ ] <!-- Pendiente 1 -->
- [ ] <!-- Pendiente 2 -->
- [ ] <!-- Pendiente 3 -->

---

## 14. Próximos pasos recomendados

1. **Validación con hosts reales** — Probar con Claude Desktop y Cursor en escenarios de Registration Form end-to-end
2. **Feedback loop** — Monitorear qué scopes se usan más, qué pitfalls se siguen cometiendo, y ajustar contenido
3. ~~**Métricas de uso**~~ — ✅ Resuelto: la herramienta ahora logea cada request con metadata estructurada (type, mode, requestedScopes, resolvedScopes, warnings, guideTokenEstimate, tenantId)
4. **Nuevos dominios** — Si surgen nuevos casos de uso (e.g., emails, analytics), la arquitectura está preparada: agregar archivo en `domains/`, actualizar `scopes.ts`
5. ~~**Circuito de clarificación**~~ — ✅ Resuelto: `clarification_prompts` contextuales en discovery response + directivas Act + Ask en MCP server instructions

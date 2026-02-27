# Swoogo MCP — Visión completa del proyecto

## 1. Resumen ejecutivo

Estamos construyendo un **servidor MCP (Model Context Protocol)** para Swoogo que permite a los usuarios interactuar con sus eventos y, en particular, **crear y administrar Registration Forms de forma simple y correcta** mediante lenguaje natural.

El proyecto es un **middleware inteligente** en TypeScript que se ubica entre los clientes (Claude Desktop, Cursor, SDKs, CLI) y la API REST de Swoogo. Interpreta intenciones del usuario en lenguaje natural, las traduce a llamadas API con gobernanza completa (permisos, auditoría, confirmaciones), y devuelve resultados estructurados.

**No reemplaza la plataforma principal de Swoogo.** La gestión completa del evento (creación, configuración, publicación) sigue en la plataforma. Este MCP apunta a **simplificar y mejorar** la creación y administración de Registration Forms, evitando que el usuario necesite conocer endpoints, payloads o patrones de la API.

### Números clave

| Métrica | Valor |
|---------|-------|
| Herramientas implementadas | 29 (25 domain + 4 system) |
| Tests | 466 (25 archivos) |
| Dominios cubiertos | Events, Registrants, Sessions, Speakers, Contacts, Transactions, Questions, Locations, Tracks, Sponsors, Packages, Reg Types, Discounts |
| Transportes | 2 (HTTP/SSE + MCP Streamable HTTP) |
| Guardrails de seguridad | 8 (G0–G7), cada uno con 2+ capas de enforcement |
| Proveedores LLM soportados | 4 (OpenAI, Anthropic, Google, Bedrock) |

---

## 2. Objetivos del proyecto

### Objetivo principal

Permitir que un usuario, desde su herramienta de trabajo habitual (Claude Desktop, Cursor, un chat), pueda:

1. **Consultar datos** de sus eventos en Swoogo con lenguaje natural
2. **Generar Registration Forms** correctos y alineados con los patrones de Swoogo
3. **Modificar registrantes** con confirmación humana antes de ejecutar cambios
4. **Entender su data** sin escribir código ni conocer APIs

### Objetivos técnicos

- Arquitectura enterprise-grade con governance completa
- Doble transporte (HTTP/SSE para apps, MCP para hosts LLM)
- Multi-tenancy por diseño (aislamiento por cuenta Swoogo)
- Audit trail completo de cada operación
- Extensible a futuros dominios sin reescribir

---

## 3. Alcance y no-alcance

### En alcance

| Capacidad | Descripción |
|-----------|-------------|
| **Lectura de datos** | 17 herramientas de lectura cubriendo 13 entidades de Swoogo |
| **Escritura con confirmación** | 8 herramientas de escritura (registrantes, sesiones, transacciones) |
| **Registration Forms** | Flujo completo: obtener metadata → questions → reg-types → generar formulario |
| **Guía de integración** | Sistema composable que instruye a hosts LLM cómo construir contra la API |
| **Multi-tenancy** | Aislamiento por cuenta (un MCP connection = una cuenta) |
| **Auditoría** | Log append-only de cada operación con redacción de PII |
| **HITL** | Confirmación humana obligatoria antes de writes (Tier 2/3) |

### Fuera de alcance (decisiones explícitas)

| Exclusión | Razón |
|-----------|-------|
| **Creación de eventos** | Demasiadas dependencias cruzadas; pertenece a la plataforma |
| **Operaciones de borrado** | Riesgo demasiado alto; guardrail G1 lo bloquea en ambas capas |
| **Frontend/UI** | El MCP es el middleware; los clientes son externos (Claude Desktop, Cursor, etc.) |
| **RAG / Vector DB** | Decidido en ADR-6: no se necesita para el alcance actual |
| **Framework LLM propio** | Usamos Vercel AI SDK v6 (producción, 20M+ downloads/mes) |

---

## 4. Arquitectura

### 4.1 Vista general — Un producto, dos transportes

```
┌──────────────────────────────────────────────────────────────────┐
│                        SHARED CORE                                │
│                                                                   │
│  Tool Registry (29)  │  ToolExecutor Pipeline  │  Governance      │
│  Audit Service       │  Swoogo API Client      │  Token Manager   │
│  Rate Limiter        │  Circuit Breaker        │  PII Redactor    │
└────────────┬──────────────────────────────┬──────────────────────┘
             │                              │
      ┌──────┴──────┐               ┌──────┴──────┐
      │  TRANSPORTE │               │  TRANSPORTE │
      │  HTTP/SSE   │               │  MCP        │
      │             │               │  Streamable │
      │  Server     │               │  HTTP       │
      │  tiene LLM  │               │             │
      │             │               │  Host tiene │
      │  Chat UIs,  │               │  LLM        │
      │  SDK, CLI   │               │             │
      └─────────────┘               │  Claude     │
                                    │  Desktop,   │
                                    │  Cursor     │
                                    └─────────────┘
```

**Decisión arquitectónica clave (ADR-14)**: Ambos transportes convergen en el **ToolExecutor** — el punto único de gobernanza y auditoría. Esto garantiza que las mismas reglas (permisos, rate limits, confirmaciones) aplican sin importar cómo llegó la request.

### 4.2 Pipeline de ejecución (`ToolExecutor`)

Ambos transportes convergen en `ToolExecutor.execute()` (`src/application/services/tool-executor.ts`). Este es el punto único de gobernanza y auditoría.

```
Tool Call (HTTP o MCP)
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  GovernanceService.evaluate()                       │
│                                                     │
│  1. Blocklist G1    → ¿Nombre prohibido?  → DENIED  │
│  2. Tier dinámico   → ¿Escalamiento?      → Tier↑   │
│  3. Tier access     → effectiveTier > max → DENIED  │
│  4. HITL check      → Tier 2+ sin bypass  → PAUSE   │
│  5. Rate limit      → Sliding window RPM  → 429     │
│                                                     │
│  Todo OK → decision: 'allowed'                      │
└────────────┬────────────────────────────────────────┘
             │
     ┌───────┼────────────────┐
     │       │                │
  DENIED  NEEDS_APPROVAL   ALLOWED
     │       │                │
     ▼       ▼                ▼
  Audit   ConfirmationService  tool.handler(params, ctx)
  + 403   .requestConfirmation()    │
           │                        ▼
           ▼                     Audit log
        PostgreSQL +             (success/failure)
        Redis TTL (300s)         + PII redaction
           │
           ▼
        swoogo-confirm-action (MCP)
        POST /api/confirm (HTTP)
           │
      ┌────┴────┐
   Approved  Rejected/Expired
      │         │
      ▼         ▼
   Execute    Audit + end
```

**Rate limiter**: Algoritmo de sliding window por tenant por tool, almacenado en Redis. Tres niveles según `riskLevel`:

| Risk Level | Límite | Ejemplo |
|------------|--------|---------|
| `read` | 60 RPM | `swoogo-events`, `swoogo-registrants` |
| `write` | 20 RPM | `swoogo-create-registrant`, `swoogo-update-registrant` |
| `bulk` | 5 RPM | Operaciones batch (futuro) |

**Escalamiento dinámico de tier**: Actualmente una regla — `swoogo-update-registrant` con `registration_status` en `{cancelled, in_progress}` escala de Tier 2 a Tier 3 (defense-in-depth: handler también bloquea el campo via `blocked-fields.ts`).

### 4.3 Stack tecnológico

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| **Runtime** | Node.js 22+ LTS | Performance SSE, async I/O |
| **Lenguaje** | TypeScript (strict) | Type safety, Zod integration |
| **LLM SDK** | Vercel AI SDK v6 | Multi-provider, streaming, 20M+ downloads/mes |
| **MCP** | @modelcontextprotocol/sdk | Estándar oficial MCP |
| **HTTP** | Fastify 5 | Performance, schema validation |
| **Validación** | Zod | Schemas compartidos entre AI SDK, MCP y Fastify |
| **ORM** | Drizzle | Type-safe, SQL-first |
| **Base de datos** | PostgreSQL 17 | Enterprise standard, JSONB, audit durability |
| **Cache** | Valkey 8 (ioredis) | Rate limiting, token cache, confirmaciones |
| **Testing** | Vitest | Fast, ESM native |
| **Container** | Docker Compose | Dev parity |
| **IaC** | Terraform (módulo reusable) | VPC, RDS, Valkey, ECS, ALB, WAF, SSM, NewRelic |
| **Environments** | AWS ECS (staging + prod) | Terraform modules con tests plan-only |

### 4.4 Modelo de seguridad (4 capas)

```
Capa 1: IDENTIDAD     → ¿Quién es este tenant? (OAuth 2.1 + tenant status)
Capa 2: GOBERNANZA    → ¿Puede usar esta herramienta? (blocklist + tier + rate limit)
Capa 3: NEGOCIO       → ¿Puede acceder a este recurso? (delegado a Swoogo API)
Capa 4: CONFIRMACIÓN  → ¿El usuario aprobó? (HITL para Tier 2/3)
```

#### Capa 1 — Identidad (implementada, dos rutas)

Cada transporte resuelve la identidad del tenant de forma diferente:

| Transporte | Mecanismo | Implementación |
|------------|-----------|----------------|
| **HTTP/SSE** | Header `X-Tenant-Id` → lookup en PostgreSQL | `src/interface/http/middleware/auth.ts` |
| **MCP** | Bearer token JWT (OAuth 2.1) → verificación + revocación por epoch en Redis | `src/interface/mcp/server.ts` |

En ambos casos se valida que `tenant.status === 'active'` (suspended/pending → 403). El intercambio OAuth contra la API de Swoogo (`client_id`/`client_secret` → Swoogo token) ocurre más profundo, en el `SwoogoApiClient`, no en la capa de transporte.

**Protecciones adicionales en MCP:**
- Token revocation por epoch: Redis key `revoked:mcp:tenant:<tenantId>` — tokens emitidos antes del epoch se rechazan
- Session-tenant binding: `Mcp-Session-Id` debe pertenecer al mismo `tenantId` → previene cross-tenant session hijacking
- Session cap: máximo `MCP_MAX_SESSIONS_PER_TENANT` (default: 5) sesiones concurrentes por tenant (atomic Redis INCR)

#### Capa 2 — Gobernanza (implementada, `GovernanceService`)

El `GovernanceService.evaluate()` (`src/application/services/governance-service.ts`) ejecuta 5 checks en orden:

| Step | Check | Resultado si falla |
|------|-------|-------------------|
| 1 | **G1 Blocklist** — Nombre de tool en set prohibido (`drop`, `truncate`, `delete_all`, `purge`, `destroy`, `grant_admin`, `revoke_all`, `transfer_ownership`) | `denied` (403) |
| 2 | **Tier dinámico** — `computeEffectiveTier()`: si `swoogo-update-registrant` con `registration_status` in `{cancelled, in_progress}` → escala a Tier 3 | Escala el tier efectivo |
| 3 | **Tier access** — `effectiveTier > maxTier` del tenant | `denied` (403) |
| 4 | **HITL trigger** — Tier 2+ requiere confirmación. Excepción: Tier 2 con `hostConfirmed=true` (MCP hosts que ya confirmaron con el usuario, e.g. Claude Desktop) | `needs_approval` |
| 5 | **Rate limit** — `RedisRateLimiter` por tenant por tool. Sliding window: read=60 RPM, write=20 RPM, bulk=5 RPM | `rate_limited` (429) |

Si pasa los 5 → `decision: 'allowed'`.

#### Capa 3 — Negocio (delegada a Swoogo, no implementada por nosotros)

**Decisión consciente (ADR-2)**: El middleware **no** implementa RBAC propio. La autorización a nivel de recurso ("¿puede este tenant editar el evento 12345?") la resuelve la API de Swoogo directamente:

- Cada tool handler llama a `context.swoogoClient.get/post/put(...)` con el token OAuth del tenant
- Si Swoogo retorna 403/404, el `error-normalizer` traduce el código HTTP a un mensaje neutral (sin leak de existencia del recurso)
- El error se propaga al LLM como tool result, no como error HTTP del servidor

Esta capa es Swoogo's problem, no nuestra — y eso es por diseño. Agregar una capa RBAC propia requeriría replicar la lógica de permisos de Swoogo, que cambia sin aviso y es opaca.

#### Capa 4 — Confirmación / HITL (implementada, `ConfirmationService`)

Cuando governance retorna `needs_approval`:

1. `ToolExecutor` llama a `ConfirmationService.requestConfirmation()`
2. Se crea un `PendingConfirmation` en PostgreSQL + token en Redis con TTL (300s)
3. El cliente recibe el resumen de la acción y el token de confirmación
4. El usuario aprueba/rechaza vía `swoogo-confirm-action` (MCP) o `POST /api/confirm` (HTTP)
5. State machine: `pending → approved | rejected | expired`

**Tier 3 (doble confirmación)** siempre requiere HITL del servidor — `hostConfirmed` solo bypasea Tier 2.

---

## 5. Componentes principales

### 5.1 Catálogo de herramientas (29 implementadas)

**Tier 0 — Auto-ejecutables (17 herramientas)**

Todas las operaciones de lectura se ejecutan inmediatamente sin confirmación.

| Herramienta | Entidad | Capacidad |
|-------------|---------|-----------|
| `swoogo-events` | Events | Lista/detalle con fuzzy search |
| `swoogo-registrants` | Registrants | Lista/detalle con fuzzy search |
| `swoogo-sessions` | Sessions | Lista/detalle con fuzzy search |
| `swoogo-speakers` | Speakers | Lista/detalle con fuzzy search |
| `swoogo-contacts` | Contacts | Lista/detalle |
| `swoogo-event-questions` | Questions | Preguntas, websites, pages, folders |
| `swoogo-transactions` | Transactions | Entradas contables |
| `swoogo-session-attendance` | Attendance | Registros de asistencia |
| `swoogo-reg-types` | Reg Types | Tipos de registro con tarifas |
| `swoogo-discounts` | Discounts | Códigos de descuento |
| `swoogo-sponsors` | Sponsors | Sponsors del evento |
| `swoogo-tracks` | Tracks | Tracks de sesiones |
| `swoogo-locations` | Locations | Ubicaciones |
| `swoogo-packages` | Packages | Paquetes |
| `swoogo-magic-link` | Registrants | Link de acceso directo |
| `swoogo-detect-session-conflicts` | Sessions | Detección de solapamientos |
| `swoogo-attendance-stats` | Attendance | Métricas agregadas |

**Tier 2 — Requieren confirmación (7 herramientas)**

| Herramienta | Acción |
|-------------|--------|
| `swoogo-create-registrant` | Crear registrante (con sesiones opcionales) |
| `swoogo-update-registrant` | Modificar registrante (escala a Tier 3 si cambia status) |
| `swoogo-assign-session` | Inscribir en sesión |
| `swoogo-remove-session` | Desinscribir de sesión |
| `swoogo-checkin-registrant` | Marcar check-in |
| `swoogo-create-session-attendance` | Registrar asistencia |
| `swoogo-update-session-attendance` | Actualizar asistencia |

**Tier 3 — Doble confirmación (1 herramienta)**

| Herramienta | Acción |
|-------------|--------|
| `swoogo-create-transaction` | Registrar transacción financiera (inmutable) |

**System Tools (4)**

| Herramienta | Propósito |
|-------------|-----------|
| `swoogo-auth-whoami` | Identidad del tenant |
| `swoogo-auth-logout` | Revocar tokens MCP |
| `swoogo-auth-reset` | Forzar re-autenticación |
| `swoogo-confirm-action` | Aprobar/rechazar confirmaciones HITL |

### 5.2 Guardrails (G0–G7)

Cada guardrail tiene **al menos dos capas de enforcement** (prompt + código), garantizando que bypasear una capa no rompe la protección.

| Guardrail | Propósito | Capas |
|-----------|-----------|-------|
| **G0: Data Accuracy** | Nunca alucinar IDs | System prompt + validación pre-hook + API enforcement |
| **G1: No Delete** | Bloquear operaciones destructivas | Governance blocklist + system prompt |
| **G2: Help Requests** | Fundamentar respuestas "how-to" | System prompt + tool `search-swoogo-help` |
| **G3: Write Confirmation** | Gate HITL para writes | Governance tier check + system prompt |
| **G4: Tool Failure Handling** | Prevenir loops de retry | Tool executor `do_not_retry: true` + prompt |
| **G5: Capability Awareness** | Transparencia sobre límites | System prompt + `list_available_actions` |
| **G6: Untrusted Content** | Sanitizar input del usuario | Runtime guardrails en system prompt |
| **G7: Integration Guide** | Guía es instruction-only, no data source | System prompt + scope filtering |

### 5.3 Multi-tenancy

**Decisión**: Una conexión MCP por cuenta Swoogo (modelo Notion).

```json
// Configuración del usuario en Claude Desktop
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

Cada conexión se autentica con credenciales distintas. Sin switching de cuentas en sesión — aislamiento natural, cero riesgo de cross-tenant leakage.

**Mecanismos de aislamiento:**
- `tenant_id` FK en todas las tablas (PostgreSQL)
- Rate limits por tenant (Redis)
- Circuit breakers por tenant (en memoria)
- Token revocation por tenant (epoch strategy en Redis)
- OAuth tokens por tenant (Redis con TTL)

### 5.4 Integration Guidance — El objetivo final del proyecto

Todo lo anterior (tools, governance, guardrails, multi-tenancy) existe para habilitar este feature: que un usuario pueda escribir un prompt simple y obtener un **Registration Form correcto** generado contra datos reales de su evento.

**El problema**: Cuando un host LLM (Claude Desktop, Cursor) necesita construir algo contra la API de Swoogo, no conoce la API. Sin guía, el LLM inventa endpoints, mezcla patrones, y genera código que no funciona. Entregarle toda la documentación causa otro problema: contaminación de contexto, desperdicio de tokens, y alucinaciones por exceso de información irrelevante.

**La solución**: `swoogo-integration-guide` — una herramienta Tier 0 que entrega guías técnicas **composables por scope**. El host recibe solo el contenido que necesita, en un contrato predecible de 17 secciones.

#### Flujo Act + Ask + Expand

El MCP Server instruye al host LLM con un flujo de 3 pasos que combina acción inmediata con expansión guiada:

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

**Paso 1 — ACT**: Identifica el scope primario de la petición del usuario y pídelo. Usa las discovery tools (`swoogo-events`, `swoogo-event-questions`, `swoogo-reg-types`) para inspeccionar qué tiene el evento.

**Paso 2 — ASK**: Después de recibir la guía y ver la data del evento, sugiere next steps basados en lo que encontró. El discovery response incluye `clarification_prompts` — preguntas contextuales generadas server-side que el host puede usar (e.g., "This event has paid reg-types. Do you need payment integration?").

**Paso 3 — EXPAND**: Pide scopes adicionales uno a uno conforme el usuario confirma. No existe `scope: "all"` — el servidor requiere scopes específicos para forzar el patrón incremental.

#### Arquitectura composable

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

**5 dominios disponibles**: registration, sessions, speakers, contacts, reporting

**Protocolo de descubrimiento**:
- **Discovery** (opcional) — El host llama sin scope → recibe lista de dominios con `clarification_prompts` contextuales (~500 tokens)
- **Targeted** — El host llama con scope específico → recibe guía filtrada (~3,200 tokens para registration)
- No existe `scope: "all"` — el servidor requiere scopes explícitos para forzar el patrón incremental Act + Ask

**Eficiencia en tokens**:

| Escenario | Guía completa | Composable | Ahorro |
|-----------|:------------:|:----------:|:------:|
| Registration form | ~4,500 | ~3,200 | 29% |
| Sessions only | ~4,500 | ~2,500 | 44% |
| Contexto persistente (15 turnos) | 67,500 | 48,000 | 19,500 tokens |

**Total estimado por sesión**: ~30,650 tokens ahorrados (entrega + contexto + calidad downstream).

**Garantías**:
- Contenido estático, pre-autorizado en source control — cero generación dinámica
- Determinista: misma entrada → misma salida (enforced por tests de snapshot)
- 65 tests unitarios (contrato de 17 secciones, seguridad, snapshots por scope × modo)
- Versionado semver (actualmente v1.1.0)

> Para el detalle técnico completo (17 secciones, reglas cross-cutting, hallazgos, etc.), ver `docs/presentations/01-integration-guidance-feature.md`.

---

## 6. Experiencia del usuario

### Escenario 1: Consultar datos de un evento
```
Usuario: "¿Cuántos registrantes tiene el evento Swoogo Summit?"

→ El sistema busca el evento por nombre (fuzzy search)
→ Encuentra "Swoogo Summit 2026" (match score: 0.95)
→ Cuenta registrantes con filtro por event_id
→ Responde: "El evento Swoogo Summit 2026 tiene 847 registrantes confirmados."
```

### Escenario 2: Crear un Registration Form (Act + Ask + Expand)
```
Usuario: "Generame el Registration Form del Evento 12345 utilizando el MCP de Swoogo"

ACT:
→ Host infiere scope ["registration"], pide guía de integración
→ Inspecciona el evento: metadata, preguntas, reg-types, packages

ASK:
→ "El evento tiene 3 reg-types con precios. ¿Incluyo integración de pagos?"
→ "Hay campos sq_* (selección de sesiones). ¿Incluyo scheduling?"

EXPAND:
→ Usuario confirma sesiones → pide scope ["sessions"]
→ Genera formulario completo con todos los datos reales del evento
```

### Escenario 3: Registrar un asistente (con confirmación)
```
Usuario: "Registra a john@example.com en el evento 12345 con tipo VIP"

→ El sistema prepara la acción (Tier 2 = requiere confirmación)
→ Muestra resumen: "Crear registrante john@example.com en Swoogo Summit, tipo VIP ($299)"
→ El usuario confirma: "Sí, proceder"
→ Se ejecuta la creación
→ Se verifica con read-after-write
→ Responde: "Registrante creado. Número de confirmación: EXT-2847"
```

### Escenario 4: Detección de conflictos
```
Usuario: "¿Hay sesiones que se solapan el día 15?"

→ Llama swoogo-detect-session-conflicts
→ Analiza horarios de todas las sesiones
→ Responde: "Sí, 'Workshop A' (14:00-16:00) se solapa con 'Panel B' (15:00-17:00)"
```

---

## 7. Patrones de ingeniería

Más allá de conectar tools a la API, el servidor implementa patrones de normalización y optimización que resuelven problemas reales de la API de Swoogo.

### 7.1 Fuzzy Search — Escalamiento en 3 etapas

Las herramientas de lectura (`swoogo-events`, `swoogo-registrants`, `swoogo-sessions`, `swoogo-speakers`) implementan búsqueda por nombre natural con un sistema de 3 etapas:

| Etapa | Estrategia | Costo API | Cuándo se usa |
|:-----:|-----------|:---------:|---------------|
| 1 | Búsqueda exacta (`name=*Annual Summit*`) | 1 call | Siempre se intenta primero |
| 2 | Búsqueda ampliada (drop de palabras: "Annual Tech" → "Tech Summit" → "Annual") | 2-5 calls | Si etapa 1 no encuentra nada |
| 3 | Full scan (5 páginas × 50 items, scoring client-side palabra por palabra) | 5 calls | Último recurso |

**Respuesta con niveles de confianza**:
- `>= 0.9` → `high` — auto-proceed con `best_match`
- `0.6–0.89` → `medium` — muestra top 3 `candidates`, pide confirmación
- `< 0.6` → `low` — top 3, probablemente sin buen match

Resultado: El usuario dice "Swoogo Summit" y el sistema encuentra "Swoogo Summit 2026" (score 0.95) sin requerir el ID exacto.

### 7.2 Page-Grouped Questions (~40% menos tokens)

La API de Swoogo devuelve preguntas de evento como una lista plana. Nosotros las transformamos en una estructura agrupada por página:

```
API response:  [q1, q2, q3, q4, q5, q6, ...]  (flat list)
                          ↓
Server output: { pages: [{ page: "Personal Info", questions: [q1, q2] },
                          { page: "Session Selection", questions: [q3, q4] }],
                  ungrouped: [q5, q6] }
```

Eliminamos el objeto `page` repetido de cada pregunta (ahora vive a nivel de grupo) y ordenamos por `sort`. El LLM recibe una estructura que refleja cómo el formulario está organizado realmente — no una lista desordenada.

### 7.3 Translation Fallback

Problema: La API de Swoogo devuelve strings vacíos `""` para campos de traducción cuando no se ha configurado una traducción. Esto causa que el LLM vea labels vacíos.

Solución: `normalizeQuestion()` detecta campos de traducción vacíos y los reemplaza con el valor default del idioma base. Campos cubiertos: `name`, `hint`, `public_short_name`, `admin_short_name`.

### 7.4 Response-Diff (solo campos modificados en updates)

Cuando un tool de escritura (`swoogo-update-registrant`) ejecuta un PUT, no devuelve el objeto entero — solo muestra qué cambió:

- **Campos silenciosamente ignorados**: campo enviado pero no presente en la respuesta → warning
- **Value mismatches**: valor enviado ≠ valor en respuesta → warning (detecta automaciones server-side)
- **Missing sessions**: verifica que cada `session_id` enviado existe en la respuesta

Maneja coerción de tipos (`number` vs `string`), choice fields (`sent: 237` vs `response: {id: 237, value: "Cars"}`), y extracción de IDs en arrays de objetos.

### 7.5 Normalized Maps (PHP → JS type safety)

Problema: `json_encode([])` de PHP produce `[]` tanto para arrays vacíos como para objetos vacíos. En JavaScript, `{}` y `[]` son tipos distintos. Esto rompe lógica de acceso como `translations.es.name`.

Solución: `normalizeEmptyMaps()` detecta campos conocidos (`translations`, `conditional_prices`, `custom_fees`) y convierte `[]` a `{}` cuando el campo semánticamente es un keyed map.

### 7.6 Expand Params (eliminación de N+1)

Varias entidades de Swoogo soportan parámetros `expand` que hidratan relaciones inline:

| Entidad | Expand | Qué resuelve |
|---------|--------|-------------|
| `reg-types` | `expand=fees` | Tarifas inline sin call separado por reg-type |
| `packages` | `expand=fees` | Pricing inline |
| `sessions` | `expand=speakers,fees,track,location` | Todo el contexto de una sesión en 1 call |
| `speakers` | `expand=sessions` | Asignaciones de speaker sin N calls |

Impacto medible en Registration Page: de 10+2N calls a 4-6 calls (ver sección 8).

---

## 8. Contribuciones a la API de Swoogo (upstream)

Parte del trabajo del proyecto no fue solo consumir la API de Swoogo sino **mejorarla**. Estas contribuciones se hicieron en el backend PHP/Laravel y benefician a todos los consumidores de la API, no solo al MCP server.

### 8.1 `ids` query parameter (batch fetch)

Agregado a nivel de `BaseController` (controlador base de toda la API v1). Permite:

```
GET /registrants?ids=1,2,3,4,5    →  5 registrantes en 1 call
```

En lugar de 5 calls individuales `GET /registrants/1`, `GET /registrants/2`, etc.

- Validación estricta: máximo 100 IDs, solo enteros positivos
- Aplicado a 30+ endpoints automáticamente (hereda de BaseController)
- Resuelve el `search=id=5,6` bug (SQL type error en el backend)

### 8.2 `expand=fees` en Packages y Reg Types

Antes: Para obtener pricing de 10 packages → 1 call de lista + 10 calls de detalle con fee.
Ahora: `GET /packages?event_id=X&expand=fees` → pricing inline en 1 call.

**Impacto**: -90% latencia en operaciones de pricing.

### 8.3 `registrant_id` en Transaction defaults

El endpoint `GET /transactions` no incluía `registrant_id` en los campos por defecto. Cada transacción requería un join manual para saber a quién pertenecía.

Fix: Una línea en el schema del modelo. Impacto en toda la plataforma.

### 8.4 Documentación de la API

5 documentos de referencia de la API producidos durante el proyecto, cubriendo endpoint catalog, route map, y gap analysis contra el OpenAPI spec. Estos documentos benefician a cualquier equipo que construya contra la API de Swoogo.

### 8.5 Case Study: Registration Page — De 10+2N calls a 4-6

**Optimizado (4-6 calls)**:

| # | Call | Propósito |
|---|------|-----------|
| 1 | `swoogo-events(id)` | Metadata del evento |
| 2 | `swoogo-event-questions(event_id, expand=page)` | **Keystone call** — preguntas agrupadas por página |
| 3 | `swoogo-packages(event_id, expand=fees)` | Pricing en una sola llamada |
| 4 | `swoogo-reg-types(event_id, expand=fees)` | Tipos de registro con tarifas |
| 5-6 | `swoogo-speakers(event_id, expand=sessions)` | Linkage (solo si hay sesiones) |

**Naive (10+2N calls)**:

1. Event metadata, 2. Flat question list, 3. Reg-types list, 4. Session list, 5. Package list, **6. Package detail × N** (para obtener fees), **7. Speaker detail × N** (para obtener assignments)

La combinación de `expand` params + page-grouped questions + batch fetch reduce dramáticamente el número de round-trips.

---

## 9. Principales decisiones y trade-offs

### Decisiones tecnológicas

| Decisión | Elegido | Descartado | Razón |
|----------|---------|-----------|-------|
| **Framework** | Vercel AI SDK v6 | Mastra | AI SDK v6 (Dic 2025) cerró la brecha. Mastra patterns extraídos, no importados |
| **Lenguaje** | TypeScript/Node.js | Laravel/PHP | Laravel no tiene mecanismo de cancelación en InvokingTool. Node.js 10-40x ventaja para SSE |
| **HITL** | Server-driven pause/resume | AI SDK `needsApproval` | `needsApproval` requiere React `useChat`. Nuestros clientes no son React |
| **Multi-account** | Una conexión por cuenta | In-session switching | Cuentas no comparten contexto. Switching agrega complejidad sin beneficio real |
| **Pipeline** | Lineal (array de middleware) | DAG workflow | No necesitamos paralelismo ni branching. Simplicidad > flexibilidad |
| **Vector DB** | No (ADR-6) | RAG pipeline | Para el alcance actual (tool calling), no se necesita embeddings |
| **Auth** | OAuth2 client_credentials | Stytch/SSO | MVP primero; Stytch/SSO es Phase 5 |
| **Tool pattern** | Tools estáticas (29 fijas) | Cloudflare Code Mode | Code Mode (search + execute dinámico) sirve para 2,500+ endpoints. Con ~29 tools, código generado podría bypassear field blocks y HITL. Tools predecibles = governance predecible |
| **Integration guide** | Contenido estático composable | Generación dinámica con LLM | Determinismo garantizado. La guía debe ser revisada en PRs, no inventada por un LLM en runtime |

### Trade-offs aceptados

| Trade-off | Ganamos | Perdemos |
|-----------|---------|----------|
| **Pipeline lineal** | Simplicidad, testeabilidad | No hay ejecución paralela de tools |
| **Sin delete** | Seguridad por diseño | El usuario no puede borrar desde el MCP |
| **Una conexión por cuenta** | Aislamiento total | Configurar N cuentas = N entradas en config |
| **Contenido de guías estático** | Determinismo, auditabilidad | Actualizar requiere PR + deploy |
| **Swoogo API sin bulk endpoints** | N/A (limitación de API) | Operaciones batch = N llamadas secuenciales |

---

## 10. Riesgos y mitigaciones

| Riesgo | Impacto | Probabilidad | Mitigación |
|--------|---------|:------------:|-----------|
| **Herramientas exceden token budget** | LLM pierde accuracy | Media | Dynamic tool loading por dominio. 29 tools < cap de ~40 |
| **Swoogo API no tiene bulk endpoints** | Batch ops lentas | Alta (ya ocurre) | Ejecución secuencial + preview + progreso |
| **LLM alucina tool calls** | Acciones incorrectas | Media | G0 guardrail + schema validation + HITL |
| **Costo LLM por tenant** | Costos impredecibles | Media | Token budgets configurables + maxSteps limit |
| **API de Swoogo cambia sin aviso** | Herramientas se rompen | Baja | Circuit breaker + error normalization + tests |
| **MCP spec evoluciona** | Breaking changes | Baja | Solo releases oficiales de @modelcontextprotocol/sdk |
| **Fase 3 es critical path** | Todo downstream se retrasa | Media | Divisible en 3A (HITL) / 3B (write tools) |

---

## 11. Hallazgos y problemas resueltos durante el desarrollo

### Problema 1: Laravel no soporta HITL correctamente
**Hallazgo**: El evento `InvokingTool` de Laravel AI SDK es un data carrier sin mecanismo de cancelación. El patrón `PENDING_APPROVAL` + `continue()` funciona ~40% de las veces.
**Solución**: Migración a TypeScript/Node.js con Vercel AI SDK v6 y HITL server-driven.

### Problema 2: Mastra agrega complejidad innecesaria
**Hallazgo**: Mastra era la opción natural, pero AI SDK v6 (Dic 2025) cerró la brecha con soporte MCP estable y streaming multi-provider.
**Solución**: Patterns de Mastra extraídos y adaptados (RequestContext, Structured Errors, SSE, MCP coexistence), no importados.

### Problema 3: Swoogo usa `type` en vez de `token_type` en OAuth
**Hallazgo**: El token manager fallaba porque esperaba `token_type` (estándar OAuth2), pero Swoogo devuelve `type`.
**Solución**: Fix en `token-manager.ts` para aceptar ambos campos.

### Problema 4: Guía de integración monolítica contamina contexto del LLM
**Hallazgo**: Hosts LLM alucinaban endpoints y hacían tool calls especulativos con la guía completa en contexto.
**Solución**: Sistema de guías composables por scope (integration guidance enforcement).

### Problema 5: `needsApproval` requiere React
**Hallazgo**: El mecanismo HITL nativo de Vercel AI SDK (`needsApproval`) solo funciona con el hook `useChat` de React.
**Solución**: HITL server-driven: el ToolExecutor pausa, Redis almacena el estado pendiente, el cliente aprueba vía `swoogo-confirm-action` tool.

### Problema 6: Token revocation en MCP
**Hallazgo**: Las sesiones MCP persisten tokens. Sin revocación, un logout no invalida tokens ya emitidos.
**Solución**: Epoch strategy: `revoked:mcp:tenant:{id}` en Redis con timestamp. Tokens emitidos antes del epoch se rechazan en refresh.

### Problema 7: 7 vulnerabilidades P0 de seguridad pre-launch
**Hallazgo**: Auditoría de seguridad pre-launch identificó 7 vulnerabilidades P0:
1. `host_confirmed` bypass — un host malicioso podía inyectar el flag en params para saltar HITL
2. Response size sin límite — payloads grandes podían causar OOM antes de `JSON.parse()`
3. Session limit no atómico — race condition en el cap de sesiones MCP por tenant
4. Sanitizer bypass — caracteres Unicode zero-width y NFKD normalization eludían el sanitizador
5. PII redactor incompleto — no cubría nombres de campos sensibles ni patrones de API keys
6. OAuth endpoint sin rate limit — endpoint de autenticación vulnerable a brute force
7. `hostConfirmed` en params expuesto al LLM — movido a `ExecutionContext` (fuera del alcance del usuario)

**Solución**: Todos corregidos en `feat/security-hardening` (mergeado). Cada fix tiene test unitario correspondiente.

---

## 12. Estado actual

### Fases completadas

| Fase | Descripción | Estado |
|------|-------------|--------|
| **P1: Foundation + POC** | Auth, LLM orchestrator, 3 read tools, dual transport | Completada |
| **P2: Read Operations** | 17 read tools, fuzzy search, error handling robusto | Completada |
| **P3: Write + HITL** | 8 write tools, confirmación humana, token revocation | Completada |
| **P5: Testing** | 25 archivos, 466 tests | Completada |
| **P6: Abuse Protection** | Timeout, circuit breaker, rate limiting, session cap | Completada |
| **Security Hardening** | 7 vulnerabilidades P0 cerradas, cada una con test | Completada |
| **Integration Guidance** | Guías composables por scope, 5 dominios, 65 tests | Completada |

### Métricas de código

| Métrica | Valor |
|---------|-------|
| Archivos TypeScript (src/) | ~60 |
| Archivos de test | 25 |
| Tests totales | 466 |
| Herramientas implementadas | 29 (+1 integration guide) |
| Guardrails activos | 8 (G0–G7) |
| Archivos de documentación (docs/) | ~20 |
| ADRs documentados | 14 |
| Infraestructura Terraform | Módulo reusable (VPC, ECS, ALB, WAF, RDS, Valkey) |

---

## 13. Pendientes

> _Sección para completar manualmente._

- [ ] <!-- Pendiente 1 -->
- [ ] <!-- Pendiente 2 -->
- [ ] <!-- Pendiente 3 -->

---

## 14. Próximos pasos recomendados

### Corto plazo

1. **Merge integration guidance enforcement** a `main`
2. **maxTier dinámico por tenant** (P1 del roadmap) — Actualmente Tier 2+ está hardcoded; necesita leer de la tabla `tenants`
3. **SDK structured response contract** (P3 del roadmap) — Versioned SDKEvent envelope para el SDK de cliente
4. **Validación end-to-end con hosts reales** — Claude Desktop + Cursor con escenarios de Registration Form

### Mediano plazo

5. **Phase 4: Generative Tools** — Generación de texto, HTML para emails, reportes analíticos
6. **Bulk operations framework** — Preview → confirm → execute secuencial con progress tracking
7. **Idempotency** — Clave de idempotencia para operaciones de escritura

### Largo plazo

8. **Phase 5: Enterprise Hardening** — Auth avanzada (Stytch/SSO), OpenTelemetry, security audit
9. **Client SDK 1.0** — Publicación npm con tipos estables y documentación completa
10. **Dashboard de auditoría** — Interfaz para revisar logs de operaciones por tenant

---

## 15. Valor para el negocio

### Para los usuarios de Swoogo

- **Simplificación radical**: De "necesito entender la API REST de Swoogo" a "decirle al asistente qué necesito"
- **Registration Forms correctos desde el primer intento**: La guía de integración elimina trial-and-error
- **Seguridad**: No pueden ejecutar acciones destructivas; los writes requieren confirmación

### Para Swoogo como producto

- **Diferenciación competitiva**: MCP como canal de integración moderno (alineado con el ecosistema de Claude, Cursor, etc.)
- **Reducción de soporte**: Usuarios autoservidos con lenguaje natural vs. tickets preguntando cómo usar la API
- **Plataforma extensible**: De 29 a 94+ herramientas en las siguientes fases sin reescribir arquitectura
- **Gobernanza enterprise**: Audit trail, multi-tenancy, rate limiting — listo para clientes enterprise

### Para el equipo técnico

- **Arquitectura probada**: 466 tests, 8 guardrails, 14 ADRs documentados
- **Sin vendor lock-in**: LLM provider abstraction (OpenAI, Anthropic, Google, Bedrock)
- **Patrones reusables**: ToolExecutor, governance tiers, PII redaction — aplicables a otros productos

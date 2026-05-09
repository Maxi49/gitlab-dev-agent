# gitlab-dev-agent

# DevAgent — Architecture & Technical Specification
> Google Cloud Rapid Agent Hackathon · GitLab Track

---

## 1. VISION

**DevAgent** es un agente de IA que actúa como un Technical Lead fantasma dentro de cualquier repositorio de GitLab. Cuando se abre un Issue, el agente no espera — investiga de forma autónoma el contexto del repositorio, razona sobre la causa probable, identifica archivos afectados, y genera un **Proposal estructurado** que el desarrollador humano puede aprobar o rechazar con un click.

El agente nunca escribe código ni abre MRs por su cuenta. Eso es una decisión de diseño deliberada: **human-in-the-loop** no es una limitación, es la arquitectura correcta para un agente de producción.

---

## 2. OVERVIEW DE ALTO NIVEL

```
┌─────────────────────────────────────────────────────────────┐
│                        GITLAB                               │
│  Repo con Issues, MRs, Pipelines, Código                   │
│         │                                                   │
│         │ Webhook (issue created)                          │
└─────────┼───────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────┐
│   CLOUD RUN         │   FastAPI — recibe webhook,
│   (Webhook Server)  │   valida, encola el evento
└─────────┬───────────┘
          │ llama al agente
          ▼
┌─────────────────────┐
│   AGENT ENGINE      │   Runtime managed de Google Cloud
│   (ADK Runtime)     │   Stateful, escalable
└─────────┬───────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                    DEVAGENT (Python ADK)                    │
│                                                             │
│  Gemini 2.0 Flash ←→ GitLab MCP Server                    │
│                                                             │
│  Tools disponibles via MCP:                                 │
│    - get_issue()          - search_code()                   │
│    - list_issues()        - get_file_content()              │
│    - get_merge_requests() - list_commits()                  │
│    - create_comment()     - get_pipeline_logs()             │
└─────────┬───────────────────────────────────────────────────┘
          │ genera Proposal
          ▼
┌─────────────────────┐
│   FIRESTORE         │   Persiste proposals con estado:
│   (Database)        │   pending / approved / rejected
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   SVELTEKIT         │   Dashboard — el humano revisa,
│   (Frontend)        │   aprueba o pide regeneración
└─────────────────────┘
          │ si approved
          ▼
┌─────────────────────┐
│   GITLAB API        │   Postea el Proposal como
│   (via backend)     │   comentario en el Issue
└─────────────────────┘
```

---

## 3. COMPONENTES EN DETALLE

### 3.1 Webhook Server (Cloud Run · FastAPI)

**Responsabilidad única:** recibir eventos de GitLab y disparar el agente.

```
POST /webhook/gitlab
  → valida X-Gitlab-Token (secret compartido)
  → filtra solo eventos "Issue Hook" con action="open"
  → extrae: project_id, issue_iid, issue_title, issue_description
  → llama a Agent Engine con el contexto del issue
  → devuelve 200 inmediatamente (async)

GET  /proposals
  → lista todos los proposals de Firestore

GET  /proposals/:id
  → detalle de un proposal

POST /proposals/:id/approve
  → postea el comentario en GitLab vía API
  → actualiza estado en Firestore a "approved"

POST /proposals/:id/reject
  → actualiza estado en Firestore a "rejected"

POST /proposals/:id/regenerate
  → vuelve a llamar al agente con contexto adicional
  → reemplaza el proposal en Firestore
```

**Por qué Cloud Run:** serverless, escala a cero cuando no hay actividad (ahorra créditos), deployment sencillo con `gcloud run deploy`.

---

### 3.2 El Agente (Python ADK · Gemini 2.0 Flash)

Este es el core del proyecto. El agente sigue un **ciclo de razonamiento estructurado** con el siguiente comportamiento interno:

#### 3.2.1 Input del agente

```python
{
  "project_id": "12345",
  "project_name": "mi-app",
  "project_url": "https://gitlab.com/user/mi-app",
  "issue_iid": 42,
  "issue_title": "Error al procesar pagos con tarjetas VISA",
  "issue_description": "Cuando el usuario intenta pagar con VISA...",
  "issue_author": "juan.perez",
  "issue_labels": ["bug", "payments"],
  "created_at": "2026-05-09T18:00:00Z"
}
```

#### 3.2.2 Ciclo de razonamiento (multi-step)

El agente ejecuta los siguientes pasos **en orden, con razonamiento explícito entre cada uno**:

```
PASO 1 — PARSE & CLASSIFY
  Gemini analiza el issue:
  - ¿Es un bug, feature request, o deuda técnica?
  - ¿Qué componente o módulo parece afectado?
  - ¿Qué keywords son más relevantes para buscar?
  Output interno: clasificación + lista de términos de búsqueda

PASO 2 — HISTORICAL CONTEXT
  Tool: list_issues(state="closed", search=keywords, limit=5)
  Gemini analiza los issues similares cerrados:
  - ¿Alguno fue el mismo problema antes?
  - ¿Cómo se resolvió?
  - ¿Hay un patrón recurrente?

PASO 3 — CODE ARCHAEOLOGY  
  Tool: search_code(query=keywords, project_id=project_id)
  Tool: get_file_content(file_path=...) para los archivos relevantes
  Gemini identifica:
  - Archivos probablemente relacionados al issue
  - Funciones o clases específicas sospechosas
  - Dependencias entre módulos involucrados

PASO 4 — RECENT CHANGES
  Tool: list_commits(ref_name="main", since="-30days", limit=20)
  Gemini busca:
  - ¿Algún commit reciente tocó los archivos identificados?
  - ¿El issue empezó después de un deploy específico?

PASO 5 — SYNTHESIS
  Con todo el contexto recopilado, Gemini genera el Proposal final.
  No hay más tool calls en este paso — es razonamiento puro.
```

#### 3.2.3 Output del agente — El Proposal

```json
{
  "proposal_id": "uuid",
  "issue_ref": { "project_id": "...", "issue_iid": 42 },
  "generated_at": "2026-05-09T18:02:34Z",
  "status": "pending",
  
  "classification": {
    "type": "bug",
    "severity": "high",
    "component": "payments",
    "confidence": 0.87
  },
  
  "summary": "El issue reporta fallo en procesamiento de pagos VISA. Basado en el análisis del código, el problema probablemente reside en el módulo de validación de tarjetas donde la regex de detección de VISA no contempla los nuevos prefijos 4532-4539.",
  
  "probable_cause": "La función `validateCardNetwork()` en `src/payments/card-validator.ts` usa una regex desactualizada que no cubre los rangos BIN extendidos de VISA emitidos desde 2024.",
  
  "related_files": [
    {
      "path": "src/payments/card-validator.ts",
      "relevance": "high",
      "reason": "Contiene la lógica de detección de red de tarjeta"
    },
    {
      "path": "src/payments/payment-processor.ts", 
      "relevance": "medium",
      "reason": "Llama a card-validator, podría tener manejo de errores insuficiente"
    }
  ],
  
  "action_plan": [
    "Revisar y actualizar la regex en `validateCardNetwork()` con los rangos BIN actualizados de VISA",
    "Agregar tests unitarios con números de tarjeta de los nuevos rangos (4532, 4533, ..., 4539)",
    "Verificar si `payment-processor.ts` propaga correctamente el error o lo silencia",
    "Considerar usar una librería de validación de tarjetas (ej. card-validator npm) en lugar de regex manual"
  ],
  
  "complexity": "S",
  
  "similar_issues": [
    {
      "iid": 31,
      "title": "Tarjetas Mastercard no detectadas correctamente",
      "resolution": "Misma causa raíz — regex actualizada en card-validator",
      "closed_at": "2026-02-14"
    }
  ],
  
  "agent_reasoning": "El análisis de commits muestra que `card-validator.ts` fue modificado hace 45 días (commit a3f9b2c) para agregar soporte de Amex. Es posible que ese cambio haya introducido una regresión en la lógica de VISA. El issue similar #31 refuerza esta hipótesis — el módulo tiene historial de problemas con regex de redes de tarjeta.",
  
  "formatted_comment": "## 🤖 DevAgent Analysis\n\n**Clasificación:** Bug · Alta Severidad · Módulo: Payments\n\n**Causa probable:** La función `validateCardNetwork()` en `src/payments/card-validator.ts` usa una regex que no cubre los rangos BIN extendidos de VISA...\n\n..."
}
```

---

### 3.3 Frontend (SvelteKit · Vercel)

**Tres vistas únicamente:**

#### Vista 1 — Dashboard (/)
```
┌─────────────────────────────────────────────────┐
│  DevAgent                              🔴 3 new │
├─────────────────────────────────────────────────┤
│  #42  Error pagos VISA        PENDING   hace 2m │
│  #38  Crash en login          APPROVED  hace 1h │
│  #35  Performance en listado  REJECTED  hace 3h │
└─────────────────────────────────────────────────┘
```

#### Vista 2 — Proposal Detail (/proposals/:id)
```
┌─────────────────────────────────────────────────┐
│  ← Issue #42: Error al procesar pagos VISA      │
│                                    🔴 PENDING   │
├─────────────────────────────────────────────────┤
│  CLASIFICACIÓN          │  COMPLEJIDAD           │
│  Bug · Alta severidad   │  S (Small)             │
│  Módulo: Payments       │  ~2-4hs estimadas      │
├─────────────────────────────────────────────────┤
│  CAUSA PROBABLE                                  │
│  La función validateCardNetwork() en             │
│  src/payments/card-validator.ts usa una regex   │
│  desactualizada...                              │
├─────────────────────────────────────────────────┤
│  ARCHIVOS RELACIONADOS                           │
│  🔴 src/payments/card-validator.ts   HIGH        │
│  🟡 src/payments/payment-processor.ts MEDIUM     │
├─────────────────────────────────────────────────┤
│  PLAN DE ACCIÓN                                  │
│  1. Actualizar regex en validateCardNetwork()   │
│  2. Agregar tests con nuevos rangos BIN         │
│  3. Verificar manejo de errores en processor    │
│  4. Evaluar migrar a librería card-validator    │
├─────────────────────────────────────────────────┤
│  ISSUES SIMILARES                                │
│  #31 · Tarjetas Mastercard · cerrado 14/02      │
├─────────────────────────────────────────────────┤
│  RAZONAMIENTO DEL AGENTE          [ver más ▼]   │
│  El commit a3f9b2c hace 45 días modificó...     │
├─────────────────────────────────────────────────┤
│  [✅ APPROVE & POST]  [🔄 REGENERATE]  [❌ REJECT]│
└─────────────────────────────────────────────────┘
```

#### Vista 3 — Estado post-aprobación
Muestra el comentario exacto que fue posteado en GitLab, con link directo al issue.

---

### 3.4 Base de Datos (Firestore)

Colección única: `proposals`

```
proposals/
  {proposal_id}/
    issue_ref: { project_id, issue_iid }
    status: "pending" | "approved" | "rejected"
    generated_at: timestamp
    approved_at: timestamp | null
    classification: { type, severity, component, confidence }
    summary: string
    probable_cause: string
    related_files: array
    action_plan: array
    complexity: "XS" | "S" | "M" | "L" | "XL"
    similar_issues: array
    agent_reasoning: string
    formatted_comment: string
    raw_agent_output: object  // para debugging
```

---

## 4. COMPORTAMIENTO DEL AGENTE — CASOS BORDE

### Issue muy vago (ej: "la app no funciona")
El agente no falla — reduce su scope. Omite el análisis de código (no tiene keywords suficientes), documenta la limitación en `agent_reasoning`, y genera un Proposal de clasificación con preguntas estructuradas que el equipo debería responder antes de investigar.

### Issue en repositorio con poco código
El agente prioriza el historial de issues y commits. Si no encuentra código relacionado, lo dice explícitamente en el Proposal en lugar de inventar archivos.

### Issue duplicado
En el paso de Historical Context, si encuentra un issue cerrado con título casi idéntico, lo marca como probable duplicado y sugiere linkearlo en lugar de generar un plan nuevo.

### Rate limiting del MCP de GitLab
El agente tiene retry logic con exponential backoff. Si después de 3 intentos no puede ejecutar una tool, continúa con el contexto parcial que tiene y lo documenta.

---

## 5. DEFINICIÓN DE DONE — HACKATHON SCOPE

Para el 11 de junio, el proyecto está completo cuando:

- [ ] Webhook de GitLab dispara el agente correctamente
- [ ] El agente ejecuta los 5 pasos de razonamiento y genera un Proposal válido
- [ ] El Proposal se persiste en Firestore
- [ ] El frontend muestra el Dashboard y la vista de detalle
- [ ] El botón Approve postea el comentario en el issue de GitLab
- [ ] El botón Regenerate vuelve a llamar al agente
- [ ] Todo deployado (Cloud Run + Agent Engine + Vercel)
- [ ] Repo público con LICENSE
- [ ] Video demo de 3 minutos

**Fuera de scope:**
- Análisis de pipeline failures (es una fase 2)
- Autenticación de usuarios en el frontend
- Soporte multi-repo
- El agente escribiendo código o abriendo MRs

---

## 6. STACK SUMMARY

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| Agente | Python ADK | Requerimiento hackathon, nativo en Google Cloud |
| Modelo | Gemini 2.0 Flash | Rápido, barato, suficiente para razonamiento multi-step |
| Agent Tools | GitLab MCP Server | Requerimiento hackathon (Partner Power) |
| Runtime | Agent Engine | Managed, stateful, deployment con un comando |
| Backend | FastAPI + Cloud Run | Python, serverless, se integra limpio con ADK |
| DB | Firestore | No-SQL simple, SDK Python/JS, gratis en volúmenes bajos |
| Frontend | SvelteKit | Mínimo overhead, TypeScript, rápido de buildear |
| Hosting FE | Vercel | Un comando para deployar |
| Hosting BE | Cloud Run | Escala a cero, usa los $300 de crédito |

---

## 7. FLUJO COMPLETO — HAPPY PATH

```
1. Dev abre Issue #42 en GitLab
         │
2. GitLab envía POST a /webhook/gitlab (Cloud Run)
         │
3. FastAPI valida el token, extrae datos del issue
         │
4. FastAPI llama a Agent Engine → inicia sesión del agente
         │
5. ADK Agent arranca el ciclo de razonamiento:
   
   5a. PARSE: "Es un bug en el módulo de pagos"
   5b. HISTORICAL: list_issues() → encuentra #31 similar
   5c. CODE: search_code("validateCardNetwork") → card-validator.ts
   5d. CODE: get_file_content("src/payments/card-validator.ts") 
   5e. COMMITS: list_commits(since="-30days") → ve el commit de Amex
   5f. SYNTHESIS: genera el Proposal completo
         │
6. Agente guarda Proposal en Firestore (status: "pending")
         │
7. Frontend (SvelteKit) muestra el nuevo Proposal en el Dashboard
         │
8. Dev revisa el Proposal → hace click en "Approve & Post"
         │
9. FastAPI llama GitLab API → crea comentario en Issue #42
         │
10. Firestore actualiza status → "approved"
         │
11. Frontend muestra confirmación con link al issue
```

**Tiempo estimado del flujo completo (pasos 1-6):** ~15-30 segundos dependiendo del tamaño del repositorio.

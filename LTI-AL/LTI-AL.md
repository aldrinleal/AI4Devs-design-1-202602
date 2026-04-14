# LTI — ATS Minimalista *API-first*

> Autor: Aldrin Leal (AL) · Fecha: 2026-04-13
> Filosofía: **AWS-style**. Un producto = una API. Sin consola propia. Los clientes (recruiters) traen su propia UI vía MCP o REST.

---

## 1. Descripción del producto

**LTI** es un ATS (Applicant Tracking System) entregado como **API pura**. No hay panel de administración: los recruiters lo consumen desde su herramienta favorita (Claude Desktop, Cursor, scripts, su propio CRM) a través de **MCP** o **REST**, autenticándose con un *token de usuario* que transporta implícitamente el `tenantId`.

Los **candidatos** son el único actor con una UI visible: un enlace web público donde se registran, suben su CV, y completan una **mini-entrevista asistida por IA**.

### 1.1 Valor añadido

- **Time-to-integrate ≈ 0**: con MCP, un recruiter conecta Claude/Cursor a LTI en minutos y empieza a crear vacantes por chat.
- **Sin UI que mantener** → menor time-to-market, menor costo, menor superficie de bugs.
- **Mini-entrevista IA** automatizada vía OpenRouter: reduce screening manual sin UI dedicada.
- **Pay-per-use friendly**: stack 100% serverless (Lambda + DynamoDB on-demand + S3), costo ≈ 0 cuando no hay tráfico.

### 1.2 Ventajas competitivas

| Ventaja | Descripción |
|---|---|
| **API-first, AWS-style** | Un contrato REST + MCP; sin lock-in de UI. |
| **MCP-native** | Primer ATS consumible directamente desde agentes LLM. |
| **Minimalismo radical** | 5 entidades, 5 tablas DynamoDB, 1 Lambda Go. |
| **Serverless end-to-end** | Escala a cero; costo proporcional al uso real. |
| **Candidate-centric UI** | La única UI existe para el candidato, no para el recruiter. |

### 1.3 Funciones principales

1. **Gestión de vacantes** (crear / editar / consultar / cerrar) vía API.
2. **Portal público del candidato** (Next.js) para registro + upload de CV.
3. **Parsing de CV** (S3 → Lambda → OpenRouter).
4. **Mini-entrevista IA** conversacional en el portal del candidato.
5. **Gestión de candidaturas** (listar, consultar estado, cambiar estado) vía API.
6. **Auth por tokens** emitidos desde Cognito; el `tenantId` va embebido en el claim.
7. **MCP server** que expone las operaciones anteriores como herramientas a agentes LLM.

---

## 2. Lean Canvas

```mermaid
flowchart TB
  subgraph LC["Lean Canvas — LTI (API-first)"]
    P["<b>Problema</b><br/>• ATS tradicionales imponen UI pesada<br/>• Integrarlos al stack del recruiter cuesta semanas<br/>• Screening manual consume el día del recruiter"]
    S["<b>Solución</b><br/>• ATS como API (REST + MCP)<br/>• El recruiter trae su UI (Claude, Cursor, CRM)<br/>• Mini-entrevista IA autoservida por el candidato"]
    UVP["<b>UVP</b><br/>&quot;El ATS que vive dentro de tu<br/>chat. Sin consola, sin fricción.&quot;"]
    UA["<b>Ventaja Injusta</b><br/>• MCP-native desde día 0<br/>• Stack 100% serverless AWS<br/>• Modelo de datos de 5 tablas"]
    CS["<b>Segmentos</b><br/>• Recruiters tech-savvy<br/>• Startups que ya usan Claude/Cursor<br/>• Agencias boutique<br/>• HR-tech que revende como white-label"]
    KM["<b>Métricas</b><br/>• MRR<br/>• Vacantes activas por tenant<br/>• Candidatos que completan mini-entrevista<br/>• Latencia p95 de la API"]
    CH["<b>Canales</b><br/>• Anthropic MCP registry<br/>• GitHub / SDK Go+TS<br/>• Content dev-focused<br/>• Partnerships con consultoras RH"]
    CST["<b>Costes</b><br/>• Lambda + DynamoDB on-demand<br/>• S3 + CloudFront<br/>• OpenRouter (tokens LLM)<br/>• Cognito"]
    RS["<b>Ingresos</b><br/>• Suscripción por tenant + cuota de requests<br/>• Pricing por mini-entrevistas ejecutadas<br/>• Tier gratis hasta N vacantes"]
  end
```

---

## 3. Casos de uso principales

### 3.1 UC-01 — Recruiter gestiona una vacante desde su agente MCP

**Actor**: Recruiter (vía Claude Desktop / Cursor / cURL). Token Cognito transporta `tenantId` y `sub`.

**Flujo**: el recruiter le dice a su agente "crea una vacante de Go Senior remoto". El agente invoca la herramienta MCP `create_job`, que golpea `POST /jobs` en la Lambda, que persiste en DynamoDB. Consultas, ediciones y cierre siguen el mismo camino (`GET`, `PATCH`, `POST /jobs/{id}/close`).

```mermaid
flowchart LR
  R((Recruiter)) --> AG[Agente LLM<br/>Claude/Cursor]
  AG -- MCP tool --> UC1[Crear/editar/consultar/cerrar vacante]
  UC1 -- «include» --> UC1a[Autenticar token Cognito]
  UC1 -- «include» --> UC1b[Persistir en DynamoDB]
  UC1 -- «extend» --> UC1c[Emitir link público de la vacante]
```

### 3.2 UC-02 — Candidato se postula y sube su CV

**Actor principal**: Candidato. **Secundarios**: Cognito, S3, OpenRouter.

**Flujo**: el candidato abre el link público de la vacante (Next.js en Vercel), se registra (Cognito User Pool de candidatos), sube su CV a S3 vía presigned URL. La Lambda extrae texto, pide a OpenRouter un resumen estructurado, y crea una `candidatura` vinculada a la `vacante` y al `candidato`.

```mermaid
flowchart LR
  C((Candidato)) --> UC2a[Registrarse en portal]
  UC2a -- «include» --> UC2b[Cognito User Pool]
  C --> UC2c[Subir CV]
  UC2c -- «include» --> UC2d[PUT S3 presigned]
  UC2d --> UC2e[Parsear CV con OpenRouter]
  UC2e --> UC2f[Crear candidatura en DynamoDB]
```

### 3.3 UC-03 — Mini-entrevista IA asíncrona

**Actor principal**: Candidato. **Secundarios**: OpenRouter, Lambda.

**Flujo**: tras la postulación, el portal lanza una mini-entrevista conversacional (5–7 preguntas). Cada turno hace `POST /applications/{id}/interview/turn`; la Lambda usa el CV parseado + la JD como contexto y OpenRouter genera la siguiente pregunta. Al cerrar, el sistema guarda un resumen + score en la candidatura, que el recruiter consume vía `GET /applications/{id}` desde su agente.

```mermaid
flowchart LR
  C((Candidato)) --> UC3a[Iniciar entrevista]
  UC3a --> UC3b[Generar siguiente pregunta]
  UC3b -- LLM --> OR((OpenRouter))
  C --> UC3c[Responder]
  UC3c --> UC3b
  UC3c -- «extend» --> UC3d[Cerrar entrevista + resumen]
  UC3d --> UC3e[Actualizar candidatura]
  R((Recruiter)) --> UC3f[Consultar vía API]
  UC3f --> UC3e
```

---

## 4. Modelo de datos

**5 tablas DynamoDB**. Claves diseñadas para consultas por tenant y por entidad; sin GSIs innecesarios.

### 4.1 Entidades y atributos

| Entidad | Atributo | Tipo | Nota |
|---|---|---|---|
| **tenant** | PK `tenantId`, `name`, `plan`, `webhooks`, `integrations`, `createdAt` | string, string, string `free\|pro\|enterprise`, list<map{event,url,secret}>, map<string,map>, number | `webhooks`: lista opcional de endpoints para eventos (`application.created`, `interview.closed`, `job.closed`, …). `integrations`: config opcional por proveedor (`slack`, `greenhouse`, `workable`, …) con claves cifradas via KMS. |
| **usuario** (recruiter) | PK `tenantId#userId`, `cognitoSub`, `email`, `name`, `createdAt` | string, string, string, string, number (epoch) | `cognitoSub` también como atributo para lookup inverso. |
| **candidato** | PK `candidateId`, `cognitoSub`, `email`, `fullName`, `resumeS3Key`, `resumeParsedJson`, `createdAt` | string, string, string, string, string, map, number | User pool de candidatos separado del de recruiters. |
| **vacante** | PK `tenantId`, SK `jobId`, `title`, `descriptionMd`, `status`, `publicSlug`, `createdAt`, `closedAt` | string, string, string, string, string `open\|closed`, string, number, number | Query *list jobs by tenant* → PK = tenantId. |
| **candidatura** | PK `jobId`, SK `candidateId`, `tenantId`, `status`, `interviewTranscript`, `interviewSummary`, `fitScore`, `createdAt`, `updatedAt` | string, string, string, string `applied\|interviewing\|reviewed\|rejected\|hired`, list<map>, string, number, number, number | GSI1 PK=`candidateId` para "mis aplicaciones". |

Los **webhooks** son opcionales y best-effort: tras cada evento relevante la Lambda dispara un `POST` firmado con HMAC (secreto del tenant) al endpoint registrado. Fallos se reintentan con backoff exponencial hasta N veces; no hay cola dedicada (se usa `SQS` sólo si un tenant supera el umbral, fuera del v1).

### 4.2 Diagrama de relaciones (lógicas, no FKs)

```mermaid
erDiagram
  TENANT ||--o{ USUARIO : contains
  TENANT ||--o{ VACANTE : scopes
  USUARIO ||--o{ VACANTE : owns
  VACANTE ||--o{ CANDIDATURA : receives
  CANDIDATO ||--o{ CANDIDATURA : submits
```

---

## 5. Diseño del sistema a alto nivel

### 5.1 Descripción

Todo es **serverless en AWS**. Una única **Lambda Go** (función monolítica con router HTTP) sirve tanto la **REST API** como el **MCP server** (detecta el path). **API Gateway HTTP API** la expone. La autenticación la hace **Cognito** (dos user pools: `recruiters` y `candidates`); los tokens JWT llevan `custom:tenantId` en sus claims, que el handler lee para aislar datos.

**DynamoDB** guarda las 5 tablas; **S3** guarda los CVs crudos. **OpenRouter** es el proveedor LLM para parseo de CV y mini-entrevista.

El **portal del candidato** es una app **Next.js desplegada en Vercel**, que habla directamente con la API (con token del candidato) y con S3 (presigned URLs).

**Deploy**: **Serverless Framework** desde **GitHub Actions**, con build del binario Go empaquetado en un contenedor **Podman** para imagen Lambda reproducible.

### 5.2 Diagrama de alto nivel

```mermaid
flowchart TB
  R[Recruiter<br/>Claude / Cursor / cURL]
  C[Candidato<br/>Browser]

  subgraph VERCEL[Vercel]
    NEXT[Next.js<br/>Portal candidato]
  end

  subgraph AWS[AWS]
    COG1[(Cognito<br/>User Pool recruiters)]
    COG2[(Cognito<br/>User Pool candidatos)]
    APIGW[API Gateway HTTP]
    LAMBDA[Lambda Go<br/>REST + MCP]
    DDB[(DynamoDB<br/>tenant, usuario, candidato,<br/>vacante, candidatura)]
    S3[(S3<br/>CVs)]
  end

  OR[(OpenRouter)]

  R -- token JWT --> APIGW
  NEXT -- token JWT --> APIGW
  C --> NEXT
  NEXT -- presigned PUT --> S3
  APIGW --> LAMBDA
  LAMBDA <--> DDB
  LAMBDA <--> S3
  LAMBDA --> OR
  LAMBDA -. verify JWT .- COG1
  LAMBDA -. verify JWT .- COG2

  subgraph CI[CI/CD]
    GH[GitHub Actions]
    POD[Podman build]
    SLS[Serverless Framework]
  end
  GH --> POD --> SLS --> LAMBDA
```

### 5.3 Decisiones clave

- **Una sola Lambda** (no microservicios): minimiza cold starts, deploy y complejidad. Go la hace viable por su arranque rápido.
- **MCP y REST en el mismo binario**: el MCP es un *thin wrapper* sobre los mismos handlers REST.
- **Sin consola propia**: elimina ~70% del scope inicial de un ATS.
- **Dos user pools Cognito**: aísla el espacio de identidades de recruiters y candidatos; cada token sabe a qué pool pertenece.
- **`tenantId` en el token**, no como parámetro: imposible consultar datos de otro tenant aunque se fuerce la URL.

---

## 6. Diagrama C4 — zoom en la **Lambda Go (API + MCP)**

### 6.1 C1 — Contexto

```mermaid
flowchart LR
  R[Recruiter + Agente LLM]
  C[Candidato]
  LTI[[LTI API]]
  OR[(OpenRouter)]
  COG[(Cognito)]
  S3[(S3)]
  DDB[(DynamoDB)]

  R -- REST / MCP --> LTI
  C -- via Next.js portal --> LTI
  LTI --> OR
  LTI --> DDB
  LTI --> S3
  LTI -. JWT verify .- COG
```

### 6.2 C2 — Contenedores

```mermaid
flowchart TB
  NEXT[Next.js Portal<br/>Vercel]
  APIGW[API Gateway HTTP]
  LAMBDA[Lambda Go<br/>único binario]
  DDB[(DynamoDB)]
  S3[(S3)]
  OR[(OpenRouter)]
  COG[(Cognito User Pools)]

  NEXT --> APIGW --> LAMBDA
  LAMBDA --> DDB
  LAMBDA --> S3
  LAMBDA --> OR
  LAMBDA -. authorizer .- COG
```

### 6.3 C3 — Componentes internos de la Lambda Go

```mermaid
flowchart TB
  subgraph LAMBDA[Lambda Go]
    ENTRY[lambda.Start<br/>adapter APIGW]
    ROUTER[HTTP Router<br/>chi]
    AUTHMW[JWT Middleware<br/>verifica Cognito + extrae tenantId]
    RESTH[REST Handlers<br/>/jobs, /applications,<br/>/candidates/me]
    MCPH[MCP Handler<br/>/mcp streamable HTTP]
    TOOLS[MCP Tool Registry<br/>create_job, list_jobs,<br/>get_application, ...]
    SVC_JOB[JobsService]
    SVC_APP[ApplicationsService]
    SVC_INT[InterviewService]
    SVC_CV[ResumeService]
    REPO[DynamoDB Repos<br/>5 tablas]
    S3CLI[S3 Client<br/>presigned URLs]
    LLM[OpenRouter Client]
  end

  ENTRY --> ROUTER --> AUTHMW
  AUTHMW --> RESTH
  AUTHMW --> MCPH
  MCPH --> TOOLS
  TOOLS --> SVC_JOB
  TOOLS --> SVC_APP
  RESTH --> SVC_JOB
  RESTH --> SVC_APP
  RESTH --> SVC_INT
  RESTH --> SVC_CV
  SVC_JOB --> REPO
  SVC_APP --> REPO
  SVC_INT --> LLM
  SVC_INT --> REPO
  SVC_CV --> S3CLI
  SVC_CV --> LLM
  SVC_CV --> REPO
```

### 6.4 C4 — Código del componente **InterviewService**

```mermaid
classDiagram
  class InterviewService {
    +StartInterview(ctx, applicationID) InterviewState
    +NextTurn(ctx, applicationID, answer) Turn
    +Close(ctx, applicationID) InterviewSummary
    -buildContext(app Application, job Job, resume ParsedResume) PromptCtx
    -callLLM(ctx, prompt PromptCtx) LLMResponse
  }
  class Application {
    +JobID: string
    +CandidateID: string
    +Status: string
    +Transcript: []Turn
  }
  class Job {
    +ID: string
    +Title: string
    +DescriptionMd: string
  }
  class ParsedResume {
    +Skills: []string
    +YearsExperience: int
    +SummaryMd: string
  }
  class Turn {
    +Role: string
    +Content: string
    +CreatedAt: int64
  }
  class InterviewSummary {
    +SummaryMd: string
    +FitScore: float64
    +Recommendation: string
  }
  class OpenRouterClient
  class ApplicationsRepo

  InterviewService --> Application
  InterviewService --> Job
  InterviewService --> ParsedResume
  InterviewService --> Turn
  InterviewService --> InterviewSummary
  InterviewService --> OpenRouterClient
  InterviewService --> ApplicationsRepo
```

**Responsabilidades del `InterviewService`:**

- Orquesta la conversación turno a turno; cada turno es una invocación REST idempotente (el estado vive en DynamoDB, no en memoria).
- Construye el *prompt* combinando JD + resumen del CV + historial truncado para controlar coste de tokens.
- Al cerrar, pide al LLM un **resumen estructurado** (`summaryMd`, `fitScore`, `recommendation`) que queda accesible al recruiter vía `GET /applications/{id}` y vía la MCP tool `get_application`.

---

## 7. Out of scope (v1)

- Panel web de administración para recruiters.
- Kanban colaborativo / realtime.
- Integración con job boards externos.
- Calendar scheduling.
- Scorecards multi-entrevistador.

Todo lo anterior es *construible por el cliente* encima de la API si lo necesita.

# Prompts utilizados — LTI-AL

> Asistente: Claude (Opus 4.6) vía Claude Code. Repo: `aldrinleal/AI4Devs-design-1-202602`.
> Convención: **todo** prompt del usuario en este repo se registra aquí, en orden cronológico. Ver `CLAUDE.md` para la regla.

---

## Prompt 1 — Bootstrap del ejercicio (2026-04-13)

```
cria um diretorio specs/ - comeca com este statement inicial.
DEPOIS leia `README.md` e faca os passos pedidos
---
[statement en ES del enunciado LTI: diseñar ATS con descripción + Lean
Canvas, 3 casos de uso, modelo de datos, diseño de alto nivel y
diagrama C4; documentar en LTI-iniciales.md con prompts.md al lado,
dentro de una carpeta LTI-iniciales en el repo AI4Devs-design-1]
```

Resultado: creación de `specs/00-statement.md`, lectura de `ReadMe.md`, y generación v1 de `LTI-AL.md` + `prompts.md` con un diseño genérico (microservicios Kafka/Postgres/K8s) antes de conocer el stack real.

---

## Prompt 2 — Push a GitHub (2026-04-13)

```
ahora crea un branch, stage este commit, y haga push hacia el repo
de aldrinleal en github
```

Acciones: `git checkout -b feat/lti-al-design`, commit `fdd6175`, push `-u origin feat/lti-al-design`.

---

## Prompt 3 — Rediseño minimalista AWS-style (2026-04-13)

```
Premisa: minha version de LTI es re-minimalista, y busca entregar
las features aunque de forma sencilla. Como los productos de AWS.

basado en eso, re-lea la spec 00-statement y actualize los archivos
de LTI-AL basado

La premisa inicial es: No necesitamos una interfaz de gestion. Asi
que la idea es ofrecer el software como una API que pueda ser
consumida por medio de openclaw como un MCP o rest API. Los
recruiters lo conectan a su interfaz por medio de un token de
usuario (que incluye un tenant id), y asi pueden usar el modelo
para hacer seguimiento de:

casos de uso de vagas (crear / editar o consultar / cerrar)

para el candidato, lo mismo ira recibir un enlace web donde podrar
postularse - para eso el puede crear un usuario y contrasena, y
submeter su resume/cvitae. el sistema analisa y basado en eso, hace
una mini entrevista

---
busta el diseno mas minimalista posible, considerando la siguiente
architectura / stack:

frontend: nextjs + vercel

backend: go + aws + dynamodb + cognito + s3 + openrouter (publicado
en AWS Lambda via Serverless.com, github actions y Podman). un poco
mas tecnico pero con menor time to market

enfasis es en minimalismo en el modelo de datos, asi que tendremos
solamente estas tablas en dynamodb:

usuario (para recruiter - vincula al sub de cognito para gestionar
tokens y llaves)
candidato (para postulantes - misma logica)
vacante (jobs y sus job descriptions)
candidatura (vincula el candidato, hv, y su entrevista)
```

Resultado: reescritura completa de `LTI-AL.md` con:

- Enfoque **API-first** (REST + MCP) sin consola propia.
- Único actor con UI = candidato (Next.js en Vercel).
- Una sola **Lambda Go** sirviendo REST y MCP.
- **4 tablas DynamoDB**: usuario, candidato, vacante, candidatura.
- Cognito con dos user pools (recruiters / candidatos), `tenantId` en claim JWT.
- Deploy Serverless Framework + GitHub Actions + Podman.
- C4 con zoom al `InterviewService` (mini-entrevista IA vía OpenRouter).

---

## Prompt 4 — Política de registro de prompts (2026-04-13)

```
actualize `CLAUDE.md` para que cada uno de los prompts sea almacenado
en `prompts.md` - empieze con los que estan en historial para este
repo porfa
```

Resultado: creación de `CLAUDE.md` en la raíz del repo con la regla *"cada prompt del usuario se agrega a `LTI-AL/prompts.md`"*, y retroalimentación del histórico completo en este archivo.

---

## Prompt 5 — Reestructura del layout (2026-04-13)

```
mueva `LTI-AL` a la raiz, y ponga 00-statement y prompts dentro de
este directorio
```

Acciones: `git mv specs/LTI-AL LTI-AL`, `git mv specs/00-statement.md LTI-AL/00-statement.md`, `rmdir specs`. Layout final: `LTI-AL/{00-statement,LTI-AL,prompts}.md` + `CLAUDE.md` en la raíz.

---

## Prompt 6 — Stage/commit/push + abrir PR hacia upstream (2026-04-13)

```
haga un stage/commit/push, y de alla creame un PR en github hacia
https://github.com/LIDR-academy/AI4Devs-design-1-202602
```

Acciones: commit `5bc9bc5`, push a `origin/feat/lti-al-design`, PR abierto: https://github.com/LIDR-academy/AI4Devs-design-1-202602/pull/21.

---

## Prompt 7 — Añadir tabla `tenant` (2026-04-13)

```
sabes que falta? una tabla de tenant - opcionalmente que permita
configurar webhooks de alertas y integracion con otras herramientras.
modificalo, hazlo stage/push
```

Acciones: añadida entidad **tenant** al modelo de datos con `webhooks` (lista opcional `{event,url,secret}`) e `integrations` (map por proveedor con claves cifradas KMS); ER actualizado (`TENANT ||--o{ USUARIO/VACANTE`); referencias a "4 tablas" actualizadas a "5 tablas" en descripción, ventajas, Lean Canvas y diagramas C2/C3.

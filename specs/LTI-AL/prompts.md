# Prompts utilizados — LTI-AL

> Asistente: Claude (Opus 4.6) vía Claude Code. Fecha: 2026-04-13.

---

## Prompt 0 — Bootstrap del ejercicio

```
cria um diretorio specs/ - comeca com este statement inicial.
DEPOIS leia README.md e faca os passos pedidos.

[statement en ES del ejercicio LTI — diseño de ATS con Lean Canvas,
3 casos de uso, modelo de datos, diseño de alto nivel y diagrama C4]
```

Resultado: creación de `specs/00-statement.md` y lectura de `ReadMe.md` para confirmar los 7 artefactos pedidos.

---

## Prompt 1 — Descripción, valor y Lean Canvas

> Usado mentalmente por el asistente a partir del statement; no fue necesario reformularlo porque el enunciado ya describía el objetivo. De haberlo hecho manualmente con otro LLM, el prompt equivalente sería:

```
Actúa como product manager senior de un SaaS B2B. Diseña la primera versión
de LTI, un ATS "del futuro" AI-native. Entrega:
1) descripción en 1 párrafo,
2) valor añadido en bullets,
3) tabla de 5 ventajas competitivas,
4) 10 funciones principales,
5) Lean Canvas en formato Mermaid (flowchart con 9 bloques).
Tono profesional, español, Markdown.
```

---

## Prompt 2 — Casos de uso

```
Dame los 3 casos de uso principales de un ATS AI-native, en formato:
- título UC-NN
- actor principal + secundarios
- flujo en 2–3 frases
- diagrama de casos de uso en Mermaid usando flowchart LR
  con notación «include» y «extend» entre casos.
Los casos deben cubrir: publicación de vacante, screening colaborativo,
y agendamiento + scorecard de entrevista.
```

---

## Prompt 3 — Modelo de datos

```
Diseña el modelo de datos de LTI. Entrega:
1) tabla con columnas Entidad | Atributo | Tipo (usando tipos concretos:
   UUID, string, enum{...}, int, float, jsonb, timestamp, bool, text),
2) diagrama ER en Mermaid (erDiagram) con cardinalidades.
Cubre multi-tenancy, pipelines configurables por vacante, entrevistas
con múltiples participantes, scorecards estructurados, comentarios
polimórficos, automatizaciones y audit log.
```

---

## Prompt 4 — Diseño de alto nivel

```
Describe la arquitectura de alto nivel de LTI como SaaS multi-tenant
cloud-native. Párrafo explicativo + diagrama Mermaid flowchart TB con:
SPA, CDN, API Gateway, microservicios por dominio, Postgres, OpenSearch,
S3, Redis, bus de eventos (Kafka), servicio realtime (WebSocket/CRDT),
y AI Service conectado a un LLM externo. Cierra con 4 decisiones clave.
```

---

## Prompt 5 — C4 del AI Service

```
Aplica el modelo C4 al AI Service de LTI. Entrega los 4 niveles:
- C1 Contexto (flowchart LR)
- C2 Contenedores, recortado a lo relevante para IA
- C3 Componentes internos del AI Service (orchestrator, guardrails,
  prompt registry, semantic cache, LLM client, cost tracker,
  fit-score matcher, summarizer, resume parser, question generator,
  embedding service)
- C4 Código: classDiagram Mermaid del componente FitScoreMatcher
  mostrando colaboradores (JobSpec, ParsedResume, RubricScore,
  FitScoreResult, EmbeddingService, LLMClient, PromptRegistry)
Añade un bullet list con las responsabilidades clave del FitScoreMatcher.
```

---

## Notas metodológicas

- Todos los diagramas están embebidos como **Mermaid** en el mismo Markdown para que GitHub los renderice sin dependencias externas.
- Se priorizó un **AI Service** como zoom C4 por ser la pieza más diferenciadora del producto y la que concentra más complejidad arquitectónica (coste, guardrails, caché semántica, multi-provider).
- La rúbrica + similitud vectorial en `FitScoreMatcher` responde a una preocupación explícita: evitar el *black-box* puro y mantener la decisión auditable por el recruiter.

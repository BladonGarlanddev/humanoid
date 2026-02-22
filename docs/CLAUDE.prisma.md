# Prisma Model Quick Reference

Schema: `backend/prisma/schema.prisma`

## Core pipeline models

| Model | Key fields | Notes |
|---|---|---|
| `PDR` | `website`, `url`, `status (PdrStatus)` | Top-level page snapshot |
| `PdrNode` | `vectorId`, `labels`, `confidence` | Tree node within a PDR |
| `NodeVector` | pgvector type | Raw embedding storage |

**PdrStatus enum:** `SCRAPED → VECTORIZING → CLASSIFYING → TRAINING → READY | FAILED`

## Settings model

| Model | Key fields | Notes |
|---|---|---|
| `AppSettings` | `key String @id`, `value Json` | General-purpose key/value store. REST: `GET /settings/:key`, `PATCH /settings/:key`. Current keys: `notifications_electron_enabled (bool)`. |

## Task execution models

| Model | Key fields | Notes |
|---|---|---|
| `SimpleTask` | `nlSpec`, `dslSpec`, `ast` | Task definition |
| `SimpleTaskVersion` | version of above | Versioned spec |
| `SimpleTaskRun` | `status (TaskRunStatus)`, `mode (TaskRunMode)` | Single execution |
| `TaskChatMessage` | `role (ChatRole)`, `content` | Chat history per run |

**TaskRunStatus:** `PENDING → RUNNING → WAITING_FOR_INPUT → SUCCEEDED | FAILED | MAX_STEPS_REACHED`
**TaskRunMode:** `LLM_ONLY`, `DSL_ONLY`, `LLM_ASSISTED_DSL`, `FALLBACK`

## Clustering & labeling models

| Model | Key fields | Notes |
|---|---|---|
| `ClusterRun` | `scope (PDR/SITE/GLOBAL)` | Clustering job |
| `Cluster` / `ClusterMember` | `confidence` | Results + membership |
| `ClusterLabelAssignment` | `source (ClusterLabelSource)` | Label attached to cluster |
| `Label` | `website`-scoped | Semantic label |
| `NodeLabelHistory` | audit trail | Per-node label history |

## AI model registry

| Model | Key fields | Notes |
|---|---|---|
| `AiModel` | `provider (AiModelProvider)`, `context (AiContext)` | Model instance |
| `AiModelProvider` enum | `OPENAI`, `ANTHROPIC`, `FIREWORKS`, `COHERE` | |
| `AiModelClientType` enum | `FIREWORKS`, `OPENAI`, `ANTHROPIC`, `COHERE`, `LANGCHAIN` | |

## Job board models

`User`, `Company`, `Job`, `JobFitScore`, `Competency`, `TechStack`, `JobTechRequirement`, `SkillExperience`

Key enums: `Seniority`, `Discipline`, `WorkplaceType`, `JobTypeEnum`, `JobStatus`, `CompanyType`, `Industry`

## Evaluation models

`FeatureEvalRun`, `FeatureEvalMetric`, `FeatureEvalProjection`, `FeatureHealthRun`, `FeatureHealthMetric`, `UmapRun`, `UmapCoord`

## Migration rules
- Always prefer additive changes (add nullable fields, new tables, new enum values).
- Never drop columns, rename fields, or remove enum values without explicit instruction.
- After any schema edit: `npx prisma migrate dev --name <name>` then `npx prisma generate`.

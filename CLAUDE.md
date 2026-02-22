# Humanoid — Claude Code Guide

## PDR Pipeline — DORMANT (do not touch)

The PDR vectorization/clustering/classification pipeline has been superseded. The project now models websites via query selectors instead.

- Do **not** modify `backend/src/pdr/**`, `backend/src/cluster/**`, `backend/src/classifier/**`, `backend/src/embedding/**`, `backend/src/workers/**` (PDR jobs), `AI/**`, or `visualizer/**` unless the user explicitly requests PDR-related work.
- If a bug trace or task *seems* to touch these areas, stop and ask before editing.
- The schema models (`PDR`, `PdrNode`, `NodeVector`, `ClusterRun`, etc.) remain in place for DB compatibility — do not drop them.

---

## Scope routing

Before editing anything, identify which sub-project(s) the task affects and follow the matching rules:

| Sub-project | Path | Verification command |
|---|---|---|
| Backend | `backend/` | `cd backend && pnpm lint && pnpm typecheck && pnpm test` |
| Orchestrator | `orchestrator/` | `cd orchestrator && pnpm typecheck` |
| Browser worker | `browser/` | `cd browser && pnpm build` |
| Visualizer | `visualizer/` | Python project — no pnpm, no tsc |
| AI / training | `AI/` | Python project — no pnpm, no tsc |

**If a task touches more than one sub-project, stop and propose a plan before writing any code.**

---

## Architecture Overview

This is a **pnpm monorepo** with five sub-projects that run as separate processes.

```
humanoid/
├── orchestrator/          # Electron desktop app (pnpm workspace)
│   ├── orchestrator-electron/  # Electron main process (TypeScript, CommonJS)
│   └── react-ui/               # Vite + React 18 renderer (port 5173)
├── backend/               # NestJS API server (port 5000, Prisma + Postgres)
├── browser/               # Puppeteer browser worker (Node subprocess)
├── visualizer/            # Python Dash UMAP app (port 8050)
├── AI/                    # Python ML training scripts
├── lambda/                # AWS Lambda functions (vectorizor, jobProcessor)
└── docker-compose.yaml    # Redis only (port 6379)
```

### Process communication
- **Electron ↔ React**: Context-isolated IPC via preload bridge (`window.browserWorker`, `window.train`, `window.visualizer`, `window.diag`)
- **Electron ↔ Browser worker**: `child_process` spawn, message protocol over stdout/stdin
- **React ↔ Backend**: REST (axios) + Socket.io WebSockets
- **Backend ↔ Redis**: Bull queues for PDR processing pipeline
- **Backend ↔ Postgres**: Prisma ORM with pgvector extension
- **Backend ↔ LLMs**: LangChain abstraction over OpenAI / Anthropic / Fireworks / Cohere

### Key data pipeline
```
Browser scrape → PDR (page snapshot) → Vectorize → Cluster → Classify → Label
```
`PdrStatus` enum tracks each stage: `SCRAPED → VECTORIZING → CLASSIFYING → TRAINING → READY | FAILED`

### Task execution modes
`SimpleTask` runs in one of four modes: `LLM_ONLY`, `DSL_ONLY`, `LLM_ASSISTED_DSL`, `FALLBACK`

---

## Strategic Architecture Premises

These premises should inform future architectural decisions. They do not require immediate implementation but must not be designed against.

### 1. API-first
The backend must be callable by external systems — other services, AI agents, future products built on top of this framework — not just the React UI. Endpoint design should assume external consumers exist. Avoid patterns that only work when called from the orchestrator.

### 2. MCP compatibility
The agent tool system should be designed to accept tools from external MCP servers (e.g. a credential service, a data storage service). The framework itself should eventually be exposable as an MCP server so external AI agents (including openClaw) can drive it programmatically. Do not hardcode the tool surface as closed.

### 3. Multi-tenancy awareness
Do not build in assumptions that there is exactly one user, one browser worker, or one credential set. Do not engineer multi-tenancy now, but do not foreclose it. Avoid global singletons in task/session state, and prefer data models that have a natural owner/tenant field even if that field is unused today.

### 4. Metering hooks
Task execution events (created, started, completed, failed) should be structured so that a usage tracking or billing layer could consume them later. Do not build billing now. Do structure task lifecycle so the hooks exist.

### Deployment model context
The near-term model is: the developer (sole operator) builds products on top of the framework. Those products call the NestJS backend as an API. Supporting services (credential vault, data storage) are called by the agent as tools during task execution. A future model where end users run the Electron + browser worker on their own machines connected to a hosted backend is possible but not designed for yet.

---

## Commands

### Backend (`backend/`)
```bash
pnpm start:dev          # NestJS watch mode (port 5000)
pnpm build              # nest build → dist/
pnpm lint               # eslint --fix
pnpm typecheck          # tsc --noEmit
pnpm test               # jest (*.spec.ts files)
pnpm test:watch
pnpm test:cov
pnpm test:e2e           # jest --config test/jest-e2e.json
```

### Orchestrator (`orchestrator/`)
```bash
pnpm dev                # Concurrent: tsc watch + vite + esbuild + electron
pnpm dev:ui             # React dev server only (port 5173)
pnpm dev:main           # Electron main tsc watch only
pnpm build              # Build everything (main, preload, UI)
```

### Browser (`browser/`)
```bash
pnpm build              # tsc → dist/
pnpm start              # node dist/index.js (normally spawned by Electron)
```

### Infrastructure
```bash
docker-compose up -d    # Start Redis (required for backend queues)
```

### Prisma (`backend/`)
```bash
npx prisma migrate dev --name <name>   # Create + apply migration
npx prisma generate                    # Regenerate client after schema change
npx prisma validate                    # Validate schema syntax
npx prisma studio                      # GUI schema browser
```

---

## Stop Conditions

Pause and ask the user before proceeding if **any** of these are true:

- The task requires touching **more than 5 files**
- The task **crosses sub-project boundaries**
- Any change to: IPC channel names, Bull queue names, Prisma model/field names, env var names, or REST endpoint paths
- Any edit inside `backend/src/agent/agent.service.ts` or `browser/src/index.ts` (high-risk monolithic files)
- An instinct to rewrite, restructure, or refactor emerges during implementation

When in doubt, write a 3-bullet plan (files affected, risk, approach) and ask for confirmation.

---

## Hard Rules

1. **Small diffs only.** Touch only the files necessary for the task.
2. **No opportunistic refactors.** Do not rename, reorganize, or restructure code beyond what was explicitly requested.
3. **Never rename public contracts.** REST endpoint paths, IPC channel names, queue names, Prisma model/field names, env var names.
4. **Prisma changes are additive by default.** Do not drop columns, rename columns, or change enum values without explicit instruction.
5. **Every schema change = migration + `prisma generate`.** Never edit `schema.prisma` without running both.

---

## Allowed Micro-refactors (OK without asking)

These are safe to do incidentally while implementing a requested change:

- Rename a local variable for clarity **within the same function**
- Extract a constant **within the same file**
- Remove dead code that is **directly adjacent to the change being made**
- Add or tighten a TypeScript type on a symbol you are already modifying

Everything else — extracting functions, moving files, renaming exports, adding abstractions — requires explicit permission.

---

## Verification Contract

Run these before declaring any task done. Report failures verbatim.

**Backend changes:**
```bash
cd backend && pnpm lint && pnpm typecheck && pnpm test
```

**Orchestrator/react-ui changes:**
```bash
cd orchestrator && pnpm typecheck
```

**Browser changes:**
```bash
cd browser && pnpm build
```

**Prisma schema changes:**
```bash
cd backend
npx prisma validate
npx prisma generate
pnpm typecheck
```

---

## Coding Conventions

### TypeScript
- **Backend**: `target: ES2023`, `module: nodenext`, decorators + metadata enabled (NestJS)
- **Electron main**: `target: ES2020`, CommonJS output
- **React UI**: `target: ESNext`, ESM, Vite transforms
- Strict null checks on. Do not add `// @ts-ignore` or `as any` without an explanatory comment.
- React-ui path aliases: `@components`, `@utils`, `@screens`, `@styles`, `@shared`

### Formatting (enforced by Prettier)
```json
{ "singleQuote": true, "trailingComma": "all" }
```

### NestJS module structure
```
feature/
├── feature.module.ts      # NestJS module registration
├── feature.service.ts     # Business logic
├── feature.controller.ts  # HTTP endpoints
├── feature.gateway.ts     # WebSocket (if real-time)
├── feature.spec.ts        # Unit tests
├── dto/                   # Request/response DTOs (class-validator decorators)
└── entities/              # Prisma-adjacent type helpers
```

### DTOs
Use `class-validator` decorators. The global `ValidationPipe` has `whitelist: true, transform: true` — unknown fields are stripped automatically.

### React components
- Functional components + hooks only
- Server state: TanStack Query (`useQuery`, `useMutation`)
- UI library: Ant Design 5 — prefer its components over raw HTML
- File naming: `PascalCase.tsx` for components, `camelCase.ts` for hooks/utils

### Electron IPC
- All renderer → main communication goes through the preload bridge
- Never use `ipcRenderer.send` directly in renderer code
- New IPC channels must be declared in both `preload.js` and the relevant file in `src/ipc/`

---

## Workflow: Feature Work

1. **Identify scope.** Read all files you will edit before touching anything.
2. **Check stop conditions.** If any apply, write a plan and ask for confirmation.
3. **Start with the schema** (if DB change needed): add field → migrate → generate → update DTOs.
4. **Implement in order**: DTO → Service → Controller → (Gateway if WebSocket) → Frontend API call → Component.
5. **Add a `spec.ts` test** for any new service method that contains logic.
6. **Run verification** for the affected sub-project before marking done.

## Workflow: Bug Fixing

1. **Reproduce first.** Identify the exact file and line where the fault occurs.
2. **Read the failing code** and any types/interfaces it depends on.
3. **Minimal change only.** Do not touch unrelated code.
4. **Run verification** for the affected sub-project only.
5. **Report**: what the bug was, what line changed, paste the verification output.

---

## High-Risk Files

| File | Why |
|---|---|
| `backend/src/agent/agent.service.ts` | ~38KB, LLM orchestration + all task tools — changes affect all task execution |
| `browser/src/index.ts` | ~97KB, entire Puppeteer worker — targeted edits only, no refactoring |
| `backend/src/pdr/pdr.service.ts` | Core data pipeline — changes cascade to workers and Lambda |
| `backend/src/workers/*.processor.ts` | Queue job shapes are a public contract with Lambda functions |

---

## References (read only when needed)

- IPC channels and browser message protocol: [`docs/CLAUDE.ipc.md`](docs/CLAUDE.ipc.md)
- Prisma model quick reference: [`docs/CLAUDE.prisma.md`](docs/CLAUDE.prisma.md)
- Environment variables: [`docs/CLAUDE.env.md`](docs/CLAUDE.env.md)

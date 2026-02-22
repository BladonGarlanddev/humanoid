# Scratchpad: Human Takeover, Record & Replay

## Current Status
**Step 1 (Telegram notification service) — COMPLETE.**
**Step 2 (Agent detection + handoff signal) — COMPLETE. Smoke-tested 2026-02-22.**
**Step 3 (Record + Replay) — PLANNED. Ready to implement. See "Step 3 Implementation Plan" below.**

---

## Decisions Made

### What "automations" actually are
Automations are saved, parameterized sequences of the same tool calls the LLM agent already uses (`typeInput`, `click`, `keypress`, etc.). They are not a separate execution engine — just orchestrated tool calls with named variable slots instead of literal values. The agent invokes an automation the same way it would invoke any tool, supplying variable values from its task context. Credentials live in the task record (provided at task creation), not in the automation.

### Notification system: Telegram, one-way only
- A private Telegram group shared by the human and openClaw
- Backend posts one-way messages only — no buttons, no webhook, no callbacks
- Simple HTTP POST to `api.telegram.org/bot{token}/sendMessage`
- UI (React + Electron tray) also receives notifications via existing Socket.io channel
- "Done" is signaled by the user via the UI, not via Telegram

### Captcha scope
- **Recordable/replayable**: Cloudflare "I'm not a robot" checkbox, Google sign-in flows, cookie consent gates
- **Real-time handoff only (not replayable)**: image-grid captchas (images randomize per challenge)
- No paid captcha solving services. Human-in-the-loop is the solution.

### What is recorded
- `click` — CSS selector + coordinate fallback
- `type` — CSS selector + variable reference (not literal value)
- `keypress` — special keys only (Enter, Tab, Escape)
- `waitForNav` — inferred from navigation events
- NOT recorded: raw mouse movement (existing human-mouse utility handles this at replay), scrolls, hovers

### Automation scoping
- Scoped to URL pattern (hostname + pathname prefix)
- Also tagged with account label for multi-account support

### Electron notification — setting stored in NestJS
- OS notification (Electron `Notification` API + `win.flashFrame`) is togglable from the React UI
- Setting stored in backend as `AppSettings { key: 'notifications_electron_enabled', value: bool }`
- React checks setting before calling `window.electron.showTray()`
- Telegram notification is always sent (not tied to this setting)

### Recording mechanism (how capture actually works)
Recording is agent-transparent and purely server-side. When `context.automationRecording.isRecording === true`:
- Every tool call executed by `AgentService.executeTool()` (line ~504) calls `this.recordAction(ctx, toolName, args, result)`
- `recordAction` appends `{ tool, args, result, timestamp }` to `ctx.automationRecording.recordedActions[]`
- Meta-tools (START/STOP/LIST/INVOKE AUTOMATION) are skipped
- No browser-side content script injection is needed for the agent-recording path
- The `automation-discovery.service.ts` path (post-execution analysis) uses an LLM to analyze existing chat history

### Two paths to create automations
1. **Live recording**: Agent calls START_RECORDING → performs actions → calls STOP_RECORDING → `AutomationAnalyzerService.analyzeRecording()` parameterizes the recorded steps → saved to DB
2. **Post-execution discovery**: User opens CreateAutomationModal → selects task run → POST /automations/discover → `AutomationDiscoveryService` runs LLM agent over chat history → automations extracted and saved to DB

### Event routing for automation tools
Automation tools emit `context.emitEvent('createAutomation', payload)` and `context.emitEvent('invokeAutomation', payload)`. These MUST be intercepted in the orchestration service's event sink closure (same pattern as Telegram for ACTION_REQUIRED) and handled server-side WITHOUT forwarding to the React UI. The React UI round-trip was never implemented and would be fragile.

---

## Open Questions

- Selector stability strategy: how many fallback selectors, in what priority order?
- Account label on automations: user-defined at record time, or inferred from page context?

---

## Planned Build Order

1. ✅ **Telegram notification service** — `backend/src/telegram/` — DONE
2. ✅ **Agent detection + handoff signal** — DONE, smoke-tested 2026-02-22
3. **Record + Replay** — READY TO IMPLEMENT (see Step 3 plan below)
4. **Cloudflare/fingerprinting improvements** — reduce how often 1-3 are needed

---

## Step 2 — Implementation (COMPLETE)

### What was built (commits 2026-02-22)

**backend** (`c13a783`):
- `src/agent/tools/notify-human.tool.ts` — `REQUEST_HUMAN_INTERVENTION` tool
- `src/agent/tools/index.ts` — registered new tool + 4 previously-missing automation tools
- `src/agent/agent-orchestration.service.ts` — injected `TelegramService`; `events.emit` closure fires Telegram on `ACTION_REQUIRED`; added `ACTION_REQUIRED` to `StateUpdatePayload` union
- `src/agent/agent.gateway.ts` — `RESOLVE_ACTION_REQUIRED` WS handler → `context.emitInternal('ACTION_RESOLVED')`
- `src/agent/agent.module.ts` — added `TelegramModule`
- `prisma/schema.prisma` — `AppSettings` model
- `prisma/migrations/20260222000000_add_app_settings/` — migration (applied to RDS)
- `src/settings/` — `SettingsService`, `SettingsController`, `SettingsModule`
- `src/app.module.ts` — added `SettingsModule`

**orchestrator** (`b4cbe3d`):
- `react-ui/src/contexts/ActionRequired/ActionRequiredContext.tsx` — global context storing `{ reason, runId, resolve }`, `ActionRequiredProvider`, `useActionRequired`
- `react-ui/src/hooks/useSettings.ts` — TanStack Query wrapper for `GET/PATCH /settings/:key`
- `react-ui/src/components/ActionRequiredBanner/ActionRequiredBanner.tsx` — Ant Design `Alert` + Done button; fires `window.electron?.showTray?.(reason)` if setting enabled
- `react-ui/src/hooks/useAgentTaskRunSocket.ts` — detects `ACTION_REQUIRED` runUpdate, populates context with resolve fn
- `react-ui/src/App.tsx` — wraps with `ActionRequiredProvider`, renders `ActionRequiredBanner` above `Outlet`
- `orchestrator-electron/src/main.ts` — `notifications:showTray` IPC handler (`Notification` + `flashFrame`)
- `orchestrator-electron/src/preload.js` — `window.electron.showTray` via `contextBridge`
- `react-ui/src/types/preload.d.ts` — `window.electron` type declaration

### Smoke test result (2026-02-22)
```
✅ joinSuccess runId matches
✅ ACTION_REQUIRED received
✅ reason present
✅ runId present
--- Results: 4 passed, 0 failed ---
✅ Smoke test PASSED
```
Remaining manual verification: UI banner render + Electron tray notification (requires Electron app running).

---

## Step 3 — Implementation Plan

### Context
Phase 3 infrastructure (Prisma models, agent tools, orchestration service handlers, AutomationAnalyzerService, AutomationsModule/Controller/Service, React UI screens, CreateAutomationModal, discovery flow) was **written speculatively and is largely in place**. Two runtime bugs and two TypeScript errors prevent the system from functioning.

**DO NOT re-research the codebase — all gaps are identified below. Implement the 4 fixes.**

---

### Existing architecture (do not change)

**Prisma models** — `backend/prisma/schema.prisma` — both `Automation` and `AutomationExecution` tables are in the init migration (`20260213052037_init`) and are already applied to the DB.

**Agent tools** — `backend/src/agent/tools/automation.tool.ts`:
- `START_RECORDING_AUTOMATION` — sets `context.automationRecording = { isRecording: true, name, expectedParams, recordedActions: [] }`
- `STOP_RECORDING_AUTOMATION` — calls `context.emitEvent('createAutomation', { runId, recording, currentDomain })`; awaits `AUTOMATION_CREATED` internal event (30 s)
- `LIST_AUTOMATIONS` — queries `prisma.automation.findMany({ where: { pageId: context.pageTypeId } })`
- `INVOKE_AUTOMATION` — calls `context.emitEvent('invokeAutomation', { runId, automationId, parameters })`; awaits `AUTOMATION_EXECUTED` internal event (60 s)

**Recording capture** — `backend/src/agent/agent.service.ts` ~line 504:
```ts
if (ctx.automationRecording?.isRecording) {
  this.recordAction(ctx, toolName, args, result);
}
```
Every tool call (except the 4 automation meta-tools) is captured automatically. No browser changes needed.

**Orchestration handlers** — `backend/src/agent/agent-orchestration.service.ts`:
- `handleCreateAutomation(payload)` at line ~840 — analyzes recording, upserts PageType, creates Automation, emits `AUTOMATION_CREATED`
- `handleInvokeAutomation(payload)` at line ~929 — loads automation, calls `automationAnalyzer.executeAutomation()`, logs AutomationExecution, emits `AUTOMATION_EXECUTED`
- `handleStateUpdate()` at line 495 has switch cases for `CREATE_AUTOMATION` and `INVOKE_AUTOMATION` (reachable via gateway STATE_UPDATE from React UI — NOT the path we'll use)

**AutomationAnalyzerService** — `backend/src/agent/automation-analyzer.service.ts`:
- `analyzeRecording(actions, expectedParams)` — extracts param values, builds param definitions, parameterizes steps
- `executeAutomation(steps, parameters, context)` — has bug (see Gap 2 below)

**AutomationsModule** — `backend/src/automations/`:
- `automations.module.ts` — imports `[PrismaModule, AgentModule]`, provides `AutomationsService`, imported by `app.module.ts` ✅
- `automations.controller.ts` — `GET /automations`, `GET /automations/:id`, `POST /automations`, `PATCH /automations/:id`, `DELETE /automations/:id`, `POST /automations/discover`, `POST /automations/:id/increment-usage`
- `automations.service.ts` — CRUD

**AutomationDiscoveryService** — `backend/src/agent/automation-discovery.service.ts`:
- Entry point for `POST /automations/discover`
- Runs `runAutomationDiscoveryAgent` (LLM loop with SAVE_AUTOMATION + FINISH_DISCOVERY tools)
- Persists discovered automations to DB

**React UI**:
- `orchestrator/react-ui/src/screens/Automations/Automations.tsx` — list + detail pane, CreateAutomationModal trigger
- `orchestrator/react-ui/src/screens/Automations/CreateAutomationModal/CreateAutomationModel.tsx` — 3-step wizard (website → task → runs) + discovery + results. `onRunSelected` prop removed internally; handles itself.
- `orchestrator/react-ui/src/hooks/useAutomations.ts` — TanStack Query hooks for list, detail, KPIs, execute, reset mutations
- `orchestrator/react-ui/src/hooks/useDiscovery.ts` — `useStartDiscovery()` mutation for POST /automations/discover
- `orchestrator/react-ui/src/api/automations.api.ts` — API calls (note: kpis/execute/reset endpoints don't exist on backend yet)
- `orchestrator/react-ui/src/types/automations.types.d.ts` — `Automation`, `AutomationStep`, `AutomationParameter`, `AutomationsQueryParams`, `PaginatedResponse`, `ApiResponse` types

---

### Gap 1 — Event sink doesn't route `createAutomation` / `invokeAutomation` (CRITICAL)

**File:** `backend/src/agent/agent-orchestration.service.ts`, `createContext()` lines ~318–332

**Problem:** Event sink forwards everything to `gateway.emitEvent()` (Socket.io → React UI). React UI has no handler for `createAutomation`/`invokeAutomation`. They disappear. Tools time out.

**Fix:** In the `events.emit` closure, add early-return intercepts for these two event names:

```ts
emit: (eventName: string, payload: any) => {
  if (eventName === 'runUpdate' && payload?.kind === 'ACTION_REQUIRED') {
    this.telegramService
      .sendMessage(`[Humanoid] Task needs your attention:\n${payload.reason}`)
      .catch(() => {});
  }
  // Handle automation events server-side — no React UI round-trip
  if (eventName === 'createAutomation') {
    this.handleCreateAutomation(payload).catch((e) =>
      console.error('[Orchestrator] createAutomation failed:', e),
    );
    return;
  }
  if (eventName === 'invokeAutomation') {
    this.handleInvokeAutomation(payload).catch((e) =>
      console.error('[Orchestrator] invokeAutomation failed:', e),
    );
    return;
  }
  gateway.emitEvent(runId, eventName, payload);
},
```

---

### Gap 2 — `executeAutomation` calls non-existent `context.executeTool()` (CRITICAL)

**File A:** `backend/src/agent/automation-analyzer.service.ts` line ~167

**Problem:** `await context.executeTool(step.tool, resolvedArgs)` — `TaskAgentContext` has no `executeTool` method. Runtime crash.

**Fix:** Change `executeAutomation` signature to accept an executor callback:

```ts
async executeAutomation(
  steps: any[],
  parameters: Record<string, any>,
  executeTool: (toolName: string, args: any) => Promise<any>,
): Promise<{ success: boolean; error?: string }> {
  for (const step of steps) {
    const resolvedArgs = this.substituteParameters(step.args, parameters);
    console.log(`[Automation] Step: ${step.tool}`, resolvedArgs);
    try {
      await executeTool(step.tool, resolvedArgs);
    } catch (error) {
      console.error(`[Automation] Step failed: ${step.tool}`, error);
      return { success: false, error: `Step "${step.tool}" failed: ${error.message}` };
    }
  }
  return { success: true };
}
```

**File B:** `backend/src/agent/agent-orchestration.service.ts` — `handleInvokeAutomation` line ~950

Change the `executeAutomation` call to pass an executor lambda:

```ts
const result = await this.automationAnalyzer.executeAutomation(
  automation.steps as any[],
  parameters,
  (toolName, args) => this.agentService.forceToolCall({ context, toolName, args }),
);
```

Note: `AgentService.forceToolCall` is already injected into orchestration service via `@Inject(forwardRef(() => AgentService))`.

---

### Gap 3 — `Automations.tsx` passes `onRunSelected` prop that doesn't exist (TypeScript error)

**File:** `orchestrator/react-ui/src/screens/Automations/Automations.tsx`

**Problem:** Line 98 passes `onRunSelected={handleRunSelected}` to `CreateAutomationModal`. The modal's `CreateAutomationModalProps` only has `onClose`. TypeScript error.

**Fix:** Remove `handleRunSelected` function and `onRunSelected` prop from the JSX. The modal handles everything internally and calls `onClose` when done.

Change:
```tsx
const handleRunSelected = (runId: string, taskId: string) => {
  console.log("[CreateAutomation] run selected", { runId, taskId });
  setShowCreateModal(false);
};
// ...
<CreateAutomationModal
  onClose={() => setShowCreateModal(false)}
  onRunSelected={handleRunSelected}
/>
```
To:
```tsx
<CreateAutomationModal onClose={() => setShowCreateModal(false)} />
```

---

### Gap 4 — `useAutomations.ts` imports from non-existent file (TypeScript error)

**File:** `orchestrator/react-ui/src/hooks/useAutomations.ts`

**Problem:** Line 9: `import type { ..., AutomationKPIs, ... } from "../types/automation.types"` — wrong filename (singular). File is `automations.types.d.ts` (plural). `AutomationKPIs` is not defined in that file (KPI backend endpoint doesn't exist yet).

**Fix:**
1. Change import path to `"../types/automations.types"` (plural)
2. Remove `AutomationKPIs` from import (and type the `getAutomationKPIs` return as `any` in the hook)

---

### Files to change (4 total)

| # | File | Sub-project | Change |
|---|------|-------------|--------|
| 1 | `backend/src/agent/agent-orchestration.service.ts` | backend | Event sink: add createAutomation/invokeAutomation intercepts + fix executor arg in handleInvokeAutomation |
| 2 | `backend/src/agent/automation-analyzer.service.ts` | backend | Change executeAutomation signature to accept executor callback |
| 3 | `orchestrator/react-ui/src/screens/Automations/Automations.tsx` | orchestrator | Remove onRunSelected prop and no-op handler |
| 4 | `orchestrator/react-ui/src/hooks/useAutomations.ts` | orchestrator | Fix import path + drop AutomationKPIs |

---

### Known gaps (out of scope for Phase 3)

These backend endpoints are called by the UI but don't exist. They'll 404, but don't block core recording/replay:
- `GET /automations/:id/kpis` — KPI widget in detail panel won't load
- `POST /automations/:id/execute` — standalone execution button won't work
- `POST /automations/:id/reset` — reset button won't work

---

### Verification

**Backend:**
```bash
cd backend && pnpm lint && pnpm typecheck && pnpm test
```

**Orchestrator:**
```bash
cd orchestrator && pnpm typecheck
```

**End-to-end smoke (manual, requires running stack):**
1. Start task run. Send: `"Record automation 'Search jobs' with params [job_title, location], navigate indeed.com, type 'Engineer' in search box, type 'Austin' in location, stop recording."`
2. Verify `Automation` record in DB (`npx prisma studio`)
3. New run: `"List automations, invoke Search jobs with job_title=Developer, location=Seattle"`
4. Verify `AutomationExecution` record created, `usageCount` incremented
5. Open Automations screen → list loads → modal opens → select run → discover → results shown

---

## Step 2 — Original Plan

### Architecture

The existing tool pause/resume pattern (used by `OPEN_BROWSER`, `NAVIGATE_TO_WEBSITE`, etc.):
1. Tool calls `context.emitEvent('runUpdate', { kind, args })` → `AgentEventSink.emit` → `gateway.emitEvent()` → Socket.io room
2. Tool calls `context.waitForEvent('EVENT_NAME', timeoutMs)` — blocks on internal `EventEmitter`
3. UI sends result back via socket → gateway calls `context.emitInternal('EVENT_NAME', payload)` → tool unblocks

Step 2 reuses this pattern exactly.

### Files to create/modify (16 total)

| # | File | Change |
|---|------|--------|
| 1 | `backend/src/agent/tools/notify-human.tool.ts` | NEW — REQUEST_HUMAN_INTERVENTION tool |
| 2 | `backend/src/agent/tools/index.ts` | +1 import, +1 registry entry |
| 3 | `backend/src/agent/agent-orchestration.service.ts` | inject TelegramService; in events sink intercept ACTION_REQUIRED kind and call telegram; add ACTION_RESOLVED to StateUpdatePayload union; add case in handleStateUpdate switch that calls context.emitInternal('ACTION_RESOLVED') |
| 4 | `backend/src/agent/agent.gateway.ts` | add @SubscribeMessage('RESOLVE_ACTION_REQUIRED') handler → getRunSession(runId) → context.emitInternal('ACTION_RESOLVED', { resolved: true }) |
| 5 | `backend/src/agent/agent.module.ts` | +TelegramModule + SettingsModule to imports |
| 6 | `backend/prisma/schema.prisma` | +AppSettings model: key String @id, value Json, updatedAt DateTime @updatedAt |
| 7 | `backend/src/settings/settings.module.ts` | NEW |
| 8 | `backend/src/settings/settings.service.ts` | NEW — get(key), upsert(key, value) |
| 9 | `backend/src/settings/settings.controller.ts` | NEW — GET /settings/:key, PATCH /settings/:key |
| 10 | `backend/src/app.module.ts` | +SettingsModule to imports |
| 11 | `orchestrator/react-ui/src/hooks/useAgentTaskRunSocket.ts` | add actionRequired state ({ reason, runId } | null); detect runUpdate kind=ACTION_REQUIRED; expose resolveAction() which sends RESOLVE_ACTION_REQUIRED socket event + clears state |
| 12 | `orchestrator/react-ui/src/hooks/useSettings.ts` | NEW — TanStack Query useQuery + useMutation for GET/PATCH /settings/:key |
| 13 | `orchestrator/react-ui/src/components/ActionRequiredBanner/ActionRequiredBanner.tsx` | NEW — Ant Design Alert banner, shows reason, "Done" button calls resolveAction(), on mount checks notificationsEnabled setting and calls window.electron?.showTray?.(reason) |
| 14 | `orchestrator/react-ui/src/App.tsx` | render ActionRequiredBanner above Outlet when actionRequired state is set |
| 15 | `orchestrator/orchestrator-electron/src/main.ts` | add ipcMain.handle('notifications:showTray') using Electron Notification API + win.flashFrame(true) |
| 16 | `orchestrator/orchestrator-electron/src/preload.js` | expose window.electron = { showTray: (msg) => ipcRenderer.invoke('notifications:showTray', msg) } |

---

## Sub-projects Affected (Phase 3)

| Sub-project | Why |
|---|---|
| `backend/src/agent/` | Event sink fix + executor fix (2 files) |
| `orchestrator/react-ui` | Remove invalid prop, fix import (2 files) |

---

## Experiments / Failed Ideas

_None yet._

---

## Notes

- Human-like mouse movement already exists in the browser worker — replay just calls into it rather than recording coordinates
- The "automation as tool orchestration" model means replay reuses all existing tool infrastructure; no parallel execution path needed
- Telegram group as shared channel for human + openClaw is forward-compatible: openClaw just uses the same bot token
- `AppSettings` key-value table is general-purpose — reusable for future toggles (default model, Telegram on/off, etc.)
- The `RESOLVE_ACTION_REQUIRED` socket event name is intentionally distinct from `STATE_UPDATE` / `TOOL_RESULT` to keep semantics clear
- Timeout on `waitForEvent` is 10 minutes; on timeout the tool returns `{ timedOut: true }` and the LLM decides how to proceed
- Recording is transparent — no browser-side script injection needed for live agent recording
- Browser worker (`browser/src/index.ts`) does NOT need changes for Phase 3
- `Automation.steps` are stored as `[{ tool: string, args: object }]` with `{paramName}` placeholders in variable args
- `AutomationAnalyzerService.analyzeRecording()` only extracts params from `TYPE_INPUT` tool calls (heuristic-based)

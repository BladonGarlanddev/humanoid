# Scratchpad: Human Takeover, Record & Replay

## Current Status
**Step 1 (Telegram notification service) — COMPLETE.**
**Step 2 (Agent detection + handoff signal) — COMPLETE. Smoke-tested 2026-02-22.**
**Step 3 (Record + Replay) — not yet started.**

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

---

## Open Questions

- Selector stability strategy: how many fallback selectors, in what priority order?
- Account label on automations: user-defined at record time, or inferred from page context?

---

## Planned Build Order

1. ✅ **Telegram notification service** — `backend/src/telegram/` — DONE
2. ✅ **Agent detection + handoff signal** — DONE, smoke-tested 2026-02-22
3. **Record + Replay** — content script injection for capture, tool-call-based replay, Automation Prisma model
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

### Key implementation details

**notify-human.tool.ts:**
```ts
export const requestHumanIntervention = {
  name: 'REQUEST_HUMAN_INTERVENTION',
  description: 'Call when blocked by auth wall, Cloudflare, captcha. Pauses execution, notifies user, resumes after they signal done.',
  parameters: { type: 'object', properties: { reason: { type: 'string' } }, required: ['reason'] },
  async execute({ reason }, context: TaskAgentContext) {
    context.emitEvent('runUpdate', { kind: 'ACTION_REQUIRED', reason, runId: context.runId });
    try {
      const result = await context.waitForEvent('ACTION_RESOLVED', 10 * 60 * 1000);
      return { success: true, resolved: true, ...result };
    } catch {
      return { success: false, timedOut: true, message: 'No response in 10 minutes. Continuing autonomously.' };
    }
  },
};
```

**agent-orchestration.service.ts events sink intercept (lines ~311-318):**
```ts
emit: (eventName: string, payload: any) => {
  if (eventName === 'runUpdate' && payload?.kind === 'ACTION_REQUIRED') {
    this.telegramService
      .sendMessage(`[Humanoid] Task needs your attention:\n${payload.reason}`)
      .catch(() => {});
  }
  gateway.emitEvent(runId, eventName, payload);
},
```

**AppSettings Prisma model:**
```prisma
model AppSettings {
  key       String   @id
  value     Json
  updatedAt DateTime @updatedAt
}
```

**Electron notification (main.ts):**
```ts
ipcMain.handle('notifications:showTray', (_event, message: string) => {
  if (Notification.isSupported()) {
    new Notification({ title: 'Humanoid — Action Required', body: message }).show();
  }
  win?.flashFrame(true);
});
```

### Data flow
```
LLM calls REQUEST_HUMAN_INTERVENTION(reason)
  → tool emits 'runUpdate' { kind: 'ACTION_REQUIRED', reason }
    → orchestration events.emit: Telegram fire-and-forget + Socket.io broadcast
    → React: sets actionRequired state → banner mounts
      → if notificationsEnabled: window.electron.showTray(reason) → OS notification + taskbar flash
  → tool awaits 'ACTION_RESOLVED' (10 min timeout)

User resolves block, clicks "Done"
  → React sends 'RESOLVE_ACTION_REQUIRED' { runId }
  → Gateway: context.emitInternal('ACTION_RESOLVED', { resolved: true })
  → tool unblocks → returns { success: true }
  → LLM continues → React clears banner
```

### Verification
```bash
cd backend && npx prisma validate && npx prisma generate
cd backend && pnpm lint && pnpm typecheck && pnpm test
cd orchestrator && pnpm typecheck
```

---

## Sub-projects Affected

| Sub-project | Why |
|---|---|
| `backend/` | TelegramService (done), Automation model + CRUD endpoints, AppSettings model + module, agent tool, orchestration service, gateway |
| `browser/src/index.ts` | Recording mode, event capture (Step 3 only) |
| `orchestrator/react-ui` | In-app notification banner, record/stop controls, "done" button, settings toggle |
| `orchestrator/orchestrator-electron` | OS tray notification |
| `backend/src/agent/agent.service.ts` | Prompt update to detect and signal blocks; invoke automations (Step 3) |

This crosses sub-project boundaries — plan must be approved before any code is written.

---

## Deployment Model Discussion (2026-02-22)

Three deployment phases identified:
- **Phase 1 (now)**: Developer is sole consumer. Supporting services (credential vault, data storage) are called by the agent as tools.
- **Phase 2 (near future)**: Developer builds products on top of the framework. Products call NestJS as an API. Users interact with the product, not the framework directly.
- **Phase 3 (uncertain)**: End users run Electron + browser worker on their own machines; NestJS is a hosted service they pay to use.

Phase 3 is not being designed for now but the architecture must not foreclose it. Key constraint: avoid global singletons in task/session state; prefer data models with a natural owner field even if unused today.

MCP has two roles in this system:
- NestJS as MCP **server** — exposes automation capabilities to external AI agents (openClaw)
- NestJS as MCP **client** — connects to external tool servers (credential service, data service) and makes their tools available to the agent

These premises are now encoded in CLAUDE.md under "Strategic Architecture Premises".

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

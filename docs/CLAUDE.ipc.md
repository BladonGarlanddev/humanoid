# IPC Channel Reference

All channels are declared in `orchestrator/orchestrator-electron/src/preload.js` (renderer side) and handled in `orchestrator/orchestrator-electron/src/ipc/` (main side).

**Never add, rename, or remove channels without updating both sides.**

| Namespace | Channel | Direction | Purpose |
|---|---|---|---|
| `browserWorker` | `start` / `stop` | renderer → main | Spawn / kill browser subprocess |
| `browserWorker` | `isRunning` / `getCurrentUrl` | renderer → main | Status queries |
| `browserWorker` | `request` | renderer → main | Two-way RPC to browser process |
| `browserWorker` | `onStatus` / `onMessage` / `onLog` / `onExit` | main → renderer | Browser events pushed to UI |
| `browserWorker` | `onUnsolicitedEvents` | main → renderer | Unprompted browser events |
| `train` | `start` / `stop` / `isActive` | renderer → main | Training process control |
| `visualizer` | `startVisualization` / `stopVisualization` / `isActive` | renderer → main | Python Dash app control |
| `diag` | `ping` | renderer → main | Health check / context isolation probe |

## Browser worker message protocol

Messages sent between Electron main and the browser subprocess follow this shape:

```typescript
type OrchestratorMsg =
  | { type: "LOG";   status?: string; level: string; msg: string; meta?: unknown }
  | { type: "ERROR"; where: string;   error: unknown; meta?: unknown }
  | { type: "EXIT";  payload: unknown }
  | Record<string, unknown>   // custom task-specific messages
```

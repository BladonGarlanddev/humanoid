# Humanoid — External Integration Guide

This document is for **3rd party services** (e.g. a job application product) that want to drive Humanoid programmatically. It assumes you have the Humanoid backend running and do NOT own the backend codebase — you call it as a remote API.

---

## Architecture overview

```
Your service (job-app)
  │
  ├── REST calls → Humanoid backend (NestJS, port 5000)
  │     - Task CRUD
  │     - Run management
  │     - Automation queries
  │     - Settings
  │
  ├── WebSocket → Humanoid backend /agent namespace
  │     - Subscribe to run updates
  │     - Stream tool results from browser
  │     - Chat with the agent
  │
  └── MCP → POST http://localhost:5000/mcp
        - AI-agent-friendly tool surface
        - 9 tools: create_task, create_task_run, force_tool_call, …
```

**Data ownership:**
- Humanoid stores: task definitions, run history (chat), automations
- Your service stores: all domain data (jobs applied, user state, application outcomes, etc.)
- Your service persists domain data by exposing MCP tools that Humanoid's agent calls during execution

---

## Base URL and CORS

Default base URL: `http://localhost:5000`

CORS origins are controlled by the `ALLOWED_ORIGINS` env var on the backend (comma-separated). Set it to include your service's origin:
```
ALLOWED_ORIGINS=http://localhost:5173,https://your-app.example.com
```
Without this env var, only `localhost:5173`, `localhost:3000`, and `localhost:8050` are allowed.

---

## Authentication

**Currently none** — the backend has no auth middleware. Bearer token support is scaffolded in the Swagger config (`addBearerAuth()`) but not enforced. Do not expose the backend publicly without adding auth. A future pass will add JWT or API key validation; the hook point is in `src/main.ts` and each controller has `@ApiBearerAuth()` ready.

---

## Interactive API reference

The full OpenAPI/Swagger reference is served at:
```
GET http://localhost:5000/docs
```

---

## Canonical task lifecycle

```
1. Create task definition
   POST /tasks
   → returns { id, name, website, status, currentVersion }

2. Create a run
   POST /tasks/:taskId/new_run
   → returns { id (runId), status: "PENDING", mode, purpose }

3. Connect via WebSocket to drive the run
   Socket.io namespace: /agent
   Event: joinTaskRun  { runId }
   → receive: joinSuccess

4. Send initial message to start the agent
   Event: taskChatMessage  { runId, content: "Apply to the top 3 jobs for Jane Doe" }
   → agent begins: calls LLM → executes browser tools → emits runUpdate events

5. Poll run status (alternative to streaming)
   GET /tasks/:taskId/active_runs
   GET (via MCP): get_task_run_status { run_id }

6. Handle action-required pauses
   Listen for:  runUpdate { kind: "ACTION_REQUIRED", reason, runId }
   Resolve via: Event: RESOLVE_ACTION_REQUIRED  { runId }

7. Run completes
   runUpdate { kind: "RUN_STATUS_UPDATE", status: "COMPLETED" }
```

---

## Key REST endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/tasks` | Create task definition |
| `GET` | `/tasks` | List all tasks |
| `GET` | `/tasks/:id` | Get task with current version |
| `PATCH` | `/tasks/:id` | Update task (nlSpec, model, name) |
| `DELETE` | `/tasks/:id` | Delete task |
| `POST` | `/tasks/:taskId/new_run` | Start a new run |
| `GET` | `/tasks/:taskId/active_runs` | List active runs |
| `GET` | `/llm/models` | List AI models |
| `GET` | `/llm/providers` | List LLM providers |
| `POST` | `/llm/models` | Register an AI model |
| `GET` | `/automations` | List automations |
| `POST` | `/automations/discover` | Run discovery on a completed run |
| `GET` | `/settings/:key` | Get a setting |
| `PATCH` | `/settings/:key` | Upsert a setting |
| `GET` | `/website` | List websites |
| `GET` | `/website/:website` | Get website info |
| `GET` | `/diagnostics/health/quick` | Health check (no DB) |

### Create task — request body
```json
{
  "website": "indeed.com",
  "name": "Apply to software engineer jobs",
  "defaultModelId": "<uuid of AiModel>",
  "description": "Finds and applies to SWE roles matching the user's profile",
  "nlSpec": {
    "goal": "Apply to the top matching job listings",
    "requirements": "User is logged in; filter by relevance score",
    "successParams": "Application submitted confirmation page visible"
  }
}
```

### Create run — request body
```json
{
  "mode": "LLM_ONLY",
  "purpose": "EXECUTE_TASK"
}
```
`mode` values: `LLM_ONLY` | `DSL_ONLY` | `LLM_ASSISTED_DSL` | `FALLBACK`
`purpose` values: `EXECUTE_TASK` | `CREATE_DSL` | `TUNE_DSL` | `DEBUG_EXECUTION` | `TOOL_DISCOVERY` | `DISCOVER_ELEMENT`

---

## WebSocket API

Connect to `ws://localhost:5000/agent` (Socket.io).

### Client → Server events

| Event | Payload | Description |
|-------|---------|-------------|
| `joinTaskRun` | `{ runId }` | Subscribe to a run |
| `leaveTaskRun` | `{ runId }` | Unsubscribe |
| `taskChatMessage` | `{ runId, content }` | Send a message to the agent |
| `forceToolCall` | `{ runId, toolName, args }` | Execute a tool directly (bypasses LLM) |
| `RESOLVE_ACTION_REQUIRED` | `{ runId }` | Unblock an action-required pause |
| `startStopAgent` | `{ runId, enableAgent }` | Enable or disable the agent loop |
| `getChatHistory` | `{ runId }` | Fetch all messages |

### Server → Client events

| Event | Payload | Description |
|-------|---------|-------------|
| `joinSuccess` | `{ runId }` | Joined successfully |
| `taskChatHistory` | `{ chats: [...] }` | Chat history response |
| `runUpdate` | `{ kind, runId, … }` | State change (see kinds below) |
| `error` | `{ message }` | Error |

### `runUpdate` kinds

| `kind` | Meaning |
|--------|---------|
| `BROWSER_OPEN` | Browser opened/closed |
| `BROWSER_STATE_SYNC` | Browser URL changed |
| `PAGE_NAVIGATED` | Navigation complete |
| `TOOL_RESULT` | Tool executed, has result |
| `ACTION_REQUIRED` | Agent paused, human needed |
| `RUN_STATUS_UPDATE` | Run status changed (COMPLETED, FAILED, …) |
| `ELEMENT_DISCOVERY_COMPLETE` | CSS selector discovered |

---

## MCP server

The MCP server exposes a machine-friendly tool surface. Use it when an AI agent (Claude, openClaw, etc.) is orchestrating Humanoid.

**Endpoint:** `POST http://localhost:5000/mcp`
**Protocol:** JSON-RPC 2.0 (MCP Streamable HTTP transport)

### List available tools
```bash
curl -X POST http://localhost:5000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

### Available MCP tools

| Tool | Description |
|------|-------------|
| `list_tasks` | List all task definitions |
| `get_task` | Get a task by ID |
| `create_task` | Create a new task |
| `delete_task` | Delete a task |
| `create_task_run` | Start a new execution run |
| `get_task_run_status` | Poll run status |
| `get_task_run_chat` | Get run conversation history |
| `list_active_runs` | List non-terminal runs for a task |
| `list_automations` | List saved automations |
| `force_tool_call` | Execute any browser/agent tool in an active run |

`force_tool_call` requires the orchestrator browser to be connected (active WebSocket session for that runId).

### Example: create task via MCP
```bash
curl -X POST http://localhost:5000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "id": 2,
    "params": {
      "name": "create_task",
      "arguments": {
        "website": "indeed.com",
        "name": "Apply to SWE jobs",
        "default_model_id": "<model-uuid>"
      }
    }
  }'
```

---

## Receiving domain data back from the agent

When Humanoid's agent performs an action (e.g. applies for a job), it needs somewhere to record the outcome. Your service exposes that storage via **your own MCP server**. The agent calls your MCP tools during task execution.

Example flow:
1. Humanoid agent clicks "Apply" on Indeed and sees a confirmation
2. Agent calls your MCP tool `record_application` with `{ jobId, appliedAt, status: "submitted" }`
3. Your MCP server saves this to your own database
4. Humanoid never stores this — it only stores the tool call in the run's chat history

See `docs/mcp-client-design.md` for how to register your MCP server with Humanoid.

---

## Known settings keys

| Key | Type | Purpose |
|-----|------|---------|
| `notifications_electron_enabled` | `boolean` | Enable Electron tray notifications |
| `mcp_servers` | `Array<{ name, url }>` | External MCP servers Humanoid's agent can call *(planned)* |

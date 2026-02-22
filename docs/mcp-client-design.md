# MCP Client Design — Humanoid calling external MCP servers

## Status: DESIGN ONLY — not yet implemented

This document describes the planned architecture for allowing Humanoid's agent to call tools from external MCP servers during task execution. This is the mechanism by which **domain data is written back to 3rd party services** (e.g. the job app records "application submitted" in its own database).

---

## Data ownership boundary

| What | Stored where |
|------|-------------|
| Task definitions | Humanoid backend (Postgres) |
| Run history (chat messages, tool calls) | Humanoid backend (Postgres) |
| Automations | Humanoid backend (Postgres) |
| Job listings, application state, user profiles, outcomes | 3rd party service's own database |
| Credentials, secrets | 3rd party credential vault (future) |

Humanoid is execution infrastructure. It never owns domain data. The agent writes domain data exclusively through MCP tool calls to 3rd party servers.

---

## Runtime flow

```
Humanoid agent loop
  ├── calls NAVIGATE_TO_WEBSITE (built-in browser tool)
  ├── calls CLICK_ELEMENT_BY_SELECTOR (built-in)
  ├── calls GET_SCREENSHOT (built-in)
  └── calls record_application (external MCP tool → job-app MCP server)
          → job-app saves to its DB
          → returns { applicationId, status }
  ← agent sees return value in context, continues
```

---

## McpClientService interface (to be implemented)

File: `backend/src/agent/mcp-client.service.ts`

```typescript
export interface ExternalMcpTool {
  name: string;
  description: string;
  inputSchema: object; // JSON Schema
  serverName: string;  // which MCP server provides this
}

@Injectable()
export class McpClientService {
  /**
   * Connect to an external MCP server and fetch its tool list.
   * Called at agent context initialization if mcp_servers setting is configured.
   */
  async connect(serverUrl: string, serverName: string): Promise<ExternalMcpTool[]>;

  /**
   * Call a tool on the named external MCP server.
   * Returns the MCP CallToolResult content as a parsed value.
   */
  async callTool(
    serverName: string,
    toolName: string,
    args: Record<string, unknown>,
  ): Promise<unknown>;

  /**
   * Disconnect all active MCP client sessions.
   */
  async close(): Promise<void>;
}
```

---

## AgentContextType extension

`TaskAgentContext` needs a reference to the active MCP client sessions for a run.

```typescript
// Addition to ContextServices (task-agent.context.ts)
mcpClient?: McpClientService;
registeredMcpTools?: ExternalMcpTool[];
```

When the context is initialized (`AgentOrchestrationService.initSession()`):
1. Read `mcp_servers` from AppSettings
2. For each configured server, call `mcpClient.connect(url, name)`
3. Merge returned tools into `ctx.registeredMcpTools`
4. `getToolDefinitions()` in `agent.service.ts` merges registered MCP tools into the LLM's tool set

---

## Tool execution hook in agent.service.ts

In `executeTool(toolName, args, context)`, before the `TOOLS` registry lookup:

```typescript
// Check if toolName matches a registered external MCP tool
const externalTool = context.registeredMcpTools?.find(t => t.name === toolName);
if (externalTool) {
  return await context.services.mcpClient!.callTool(
    externalTool.serverName,
    toolName,
    args,
  );
}
// Otherwise fall through to built-in TOOLS registry
```

This is a ~5 line addition in a high-risk file. Implement only when an external MCP server exists to test against.

---

## Configuration model

External MCP servers are registered via the Settings API:

```bash
PATCH /settings/mcp_servers
Body: { "value": [{ "name": "job-app", "url": "http://localhost:4000/mcp" }] }
```

The `McpClientService` reads this at context initialization. Adding/removing servers takes effect on the next run — no restart needed.

---

## Metering hook note

Tool calls through MCP client should emit the same `executeTool` event as built-in tools, so that a future billing/usage layer can count all tool calls uniformly — external or internal. The event payload should include `{ toolName, serverName, runId, taskId, durationMs }`.

---

## Job-app MCP server responsibilities

The job application product's MCP server should expose tools for the domain operations Humanoid's agent needs to perform:

| Tool (example) | Description |
|----------------|-------------|
| `get_pending_jobs` | Return the next batch of jobs to apply to for a user |
| `record_application` | Save that an application was submitted |
| `update_application_status` | Mark application as failed/confirmed |
| `get_user_profile` | Return the user's resume/profile for form-filling |
| `get_credentials` | Return login credentials for Indeed (via credential vault) |

These tools are NOT implemented in Humanoid — they live in the job-app service.

---

## Implementation checklist (when ready)

- [ ] Create `backend/src/agent/mcp-client.service.ts` (new injectable)
- [ ] Add `McpClientService` to `AgentModule` providers
- [ ] Extend `AgentContextType` and `ContextServices` in `task-agent.context.ts`
- [ ] Update `AgentOrchestrationService.initSession()` to connect MCP clients
- [ ] Update `agent.service.ts` `executeTool()` to check external tools first
- [ ] Update `agent.service.ts` `getToolDefinitions()` to include external tool schemas
- [ ] Add `mcp_servers` key to settings documentation
- [ ] Test with a real job-app MCP server

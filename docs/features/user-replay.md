# Feature: Human Takeover, Record & Replay

## Problem Definition

The Puppeteer browser agent gets blocked by authentication walls and bot-detection checkpoints it cannot solve autonomously. There is currently no mechanism to pause agent execution, hand control to a human, and resume cleanly. Without this, tasks that encounter a login screen or Cloudflare checkpoint fail silently or spin indefinitely.

## Goals

- Agent automatically detects when it is blocked (auth wall, Cloudflare checkbox, captcha)
- Agent pauses execution and notifies the user via Telegram (phone) and the React UI
- User can take real-time control of the browser to complete the blocking step
- Agent resumes automatically once the human resolves the block
- For predictable, repeatable flows (login screens, consent gates): record the user's interaction once and replay it on future encounters without user involvement
- Notification system is usable by openClaw (future AI project manager) as well as the human user
- Credentials and sensitive values required for task completion are provided at task creation time and live in the task record — no runtime prompting needed

## Non-Goals

- No paid captcha-solving services
- No image-grid / puzzle captcha solving of any kind — these require real-time human intervention every time (images randomize per challenge)
- No raw mouse-movement recording (human-like movement already handled by existing Puppeteer utility)
- No credential storage in the replay record — sensitive fields are placeholders only
- TOTP / SMS two-factor authentication is out of scope for now

## Constraints

- MVP first — avoid unnecessary abstraction and caching
- Must support multiple accounts per site (replays are account-scoped, not just URL-scoped)
- Must work when the user is not at the computer (hence external Telegram notification)
- Telegram notification is one-way only — user is alerted that attention is needed, then comes to their computer to act. No remote control from phone.
- Telegram bot must be usable by both the human and openClaw from the same channel
- No changes to PDR pipeline, visualizer, or AI sub-projects

## Architecture Direction

### Three distinct sub-problems

1. **Notification system** — prerequisite; unblocks everything else
2. **Real-time handoff** — agent signals, user takes browser control, resumes after
3. **Record + Replay** — for predictable flows only (login, Cloudflare checkbox, cookie consent)

### Notification system (Telegram-first)

- Backend `TelegramService` posts to a private Telegram group shared by the human user and openClaw
- One-way only: messages inform the user that attention is needed. No inline buttons, no webhook, no callback handling.
- React UI also receives notifications via existing Socket.io channel (in-app banner/modal)
- Electron fires OS tray notification when window is not focused
- Bot token + chat ID stored in env vars

### Real-time handoff flow

```
Agent detects block (URL pattern, DOM selector, screenshot analysis)
  → Emits "needs_human" event with context (URL, task, reason)
  → Backend sends one-way Telegram notification (no buttons, no callback)
  → User sees Telegram notification on phone and/or in-app banner
  → User comes to computer, takes browser control via the UI
  → User resolves the block (manually or using a saved replay)
  → User signals "done" via UI
  → Agent resumes from where it left off
```

### Record + Replay

- **Scope**: URL pattern (hostname + pathname prefix), e.g. `accounts.google.com/*`
- **Account scope**: Each replay is also tagged with an account label (e.g. `jobs@gmail.com`) to support multi-account scenarios
- **What is recorded**:
  - `click` — target selector (CSS, with XPath fallback) + coordinates
  - `type` — target selector + value; sensitive values are sourced from task data at replay time (not stored in the replay record, not prompted at runtime)
  - `keypress` — special keys (Enter, Tab, Escape)
  - `waitForNav` — inferred from page navigation events
- **What is NOT recorded**: raw mouse movement, scroll events, hover
- **Recording mechanism**: content script injected into page via Puppeteer `page.evaluate`, sends events back via `page.exposeFunction`
- **Replay mechanism**: iterate stored steps, use existing human-mouse-movement utility for physical movement, type/click targets by selector with fallback strategies
- **Storage**: new `Automation` model in Postgres (via Prisma), retrieved by URL pattern match at replay time

### Automation design — variables and tool call orchestration

Automations are **orchestrations of the same tool calls the LLM agent already uses** (e.g. `typeInput`, `click`, `keypress`). They are not a separate execution system — they are saved, parameterized sequences of those tools.

Fields that require dynamic values (email, password, username, etc.) are declared as **named variables** in the automation record rather than storing literal values. When the agent invokes an automation it supplies the variable values from its own context (which came from the task record created at task creation time). The automation itself stores no credentials.

Example automation step:
```json
{ "tool": "typeInput", "selector": "#email", "value": { "kind": "variable", "name": "email" } }
```

The agent calls the automation with: `{ email: "jobs@gmail.com" }` — sourced from task data.

## Open Questions

- [x] How does the agent signal "I am blocked"? **Resolved: LLM calls `REQUEST_HUMAN_INTERVENTION` tool directly.**
- [x] What is the pause/resume protocol? **Resolved: `ACTION_REQUIRED` runUpdate → `RESOLVE_ACTION_REQUIRED` WS event → `ACTION_RESOLVED` internal event unblocks tool.**
- [x] How does "user is done" get signaled back? **Resolved: "Done" button in `ActionRequiredBanner` emits `RESOLVE_ACTION_REQUIRED` over Socket.io.**
- [ ] Selector stability: Google's DOM changes. How many fallback selectors to record and in what priority order?
- [ ] Account label: user-defined at record time, or inferred from the page?

## Success Criteria

- Agent running a job application task reaches a Cloudflare checkpoint, pauses, and sends a Telegram notification within 5 seconds
- User receives Telegram notification on phone, comes to computer, resolves the checkpoint via UI, signals done — agent resumes without restart
- A saved Google login replay executes end-to-end without user involvement on the second run; agent supplies credentials from task context
- openClaw can post to the same Telegram channel
- No credentials are stored in the automation record

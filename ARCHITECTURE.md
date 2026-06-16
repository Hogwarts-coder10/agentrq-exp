# AgentRQ Architecture

## Overview

AgentRQ is a task orchestration platform that connects human operators and AI agents through a structured, real-time communication layer. Each **workspace** is an isolated execution context with its own dedicated **MCP (Model Context Protocol) server**, and agents connect to workspaces via those servers to receive tasks, send replies, and report status. A supervisor agent — by connecting to multiple workspace MCP servers simultaneously — can coordinate multi-agent workflows across the entire system.

---

## Core Concepts

### Workspace

A workspace represents a scoped mission environment. It holds:
- A **name** and **description** (the mission brief)
- A **self-learning loop note** (guidelines injected into agent-assigned tasks)
- An **encrypted session token** (16-char secret, AES-GCM encrypted at rest) that authenticates agent connections
- A list of **auto-allowed tools** (agent tool calls pre-approved without human intervention)
- An optional **`AllowAllCommands`** flag (bypasses all permission gating for a workspace)

Workspaces can be archived (frozen, read-only). They can be active or archived; archived workspaces reject new task creation.

### Task

A task is the atomic unit of work. Key fields:

| Field | Description |
|-------|-------------|
| `WorkspaceID` | The workspace this task belongs to |
| `CreatedBy` | `human` or `agent` |
| `Assignee` | `human` or `agent` |
| `Status` | `notstarted`, `ongoing`, `completed`, `rejected`, `blocked`, `cron` |
| `Title` / `Body` | Task description |
| `CronSchedule` | 5-field cron string (for scheduled task templates) |
| `ParentID` | Set when a task is spawned from a cron template |
| `SortOrder` | Controls queue priority for `getNextTask` |
| `AllowAllCommands` | Per-task override for permission gating |

### Message

Each task has a threaded conversation history. Messages record every exchange between the human operator, the agent, and the system. Messages carry:
- `Sender`: `human`, `agent`, or `slack`
- `Text` and optional `Attachments`
- `Metadata`: JSON blob used for permission request lifecycle tracking (`{type, request_id, tool_name, status}`)

---

## MCP Server Layer

### Per-Workspace MCP Servers

Each workspace gets its own MCP server instance (`WorkspaceServer`), created lazily on the first agent connection and managed by a central `Manager`. The manager uses a read-write lock and a `map[workspaceID → WorkspaceServer]` for safe concurrent access.

Agents connect via:
- **Path-based**: `POST /mcp/{workspaceID}` (Streamable HTTP transport)
- **Subdomain-based**: `{workspaceID}.mcp.{domain}`

Authentication uses the workspace's encrypted token (decrypted at request time and compared to the bearer token the agent provides). Once authenticated, the agent communicates with that workspace's MCP server exclusively.

### Tool Surface

Every agent connected to a workspace MCP server has access to exactly these tools:

| Tool | Description |
|------|-------------|
| `getWorkspace()` | Returns workspace name, description, and task stats |
| `getNextTask()` | Dequeues the next `notstarted` agent-assigned task (sorted by `SortOrder`) |
| `getTaskMessages(taskId, cursor?, limit?)` | Paginated message history for a task |
| `createTask(title, body, assignee?, attachments?, cronSchedule?)` | Creates a new task (or cron template) in this workspace |
| `updateTaskStatus(taskId, status)` | Transitions task state; auto-appends a status message |
| `reply(chatId, text, attachments?)` | Sends a message in a task's conversation thread |
| `downloadAttachment(attachmentId, taskId)` | Retrieves a file attachment as base64 |

These are the only operations an agent can perform. All state mutations go through the MCP tool layer — there is no direct database access for agents.

---

## Supervisor → Worker Architecture

### How a Supervisor Agent Coordinates Workspaces

A supervisor agent coordinates multiple workspaces by **connecting to each workspace's MCP server simultaneously** using that workspace's token. Because MCP is a protocol (not a single connection), a single agent process can hold multiple MCP client sessions — one per workspace — and call tools on any of them.

The coordination flow looks like this:

```
Supervisor Agent
├── MCP Session → Workspace A (supervisor's home)
├── MCP Session → Workspace B (coding agent)
├── MCP Session → Workspace C (docs agent)
└── MCP Session → Workspace D (publishing agent)
```

The supervisor uses `createTask` on each target workspace's MCP session to assign work to the specialist agent in that workspace. It then monitors progress via `getTaskMessages` and reacts to status changes — assembling the outputs of each worker into the next stage of the pipeline.

### Step-by-Step: Supervisor Spawning a Cross-Workspace Workflow

1. **Human → Supervisor**: Human creates a task in the supervisor workspace (via the UI or REST API) with a high-level goal (e.g., "Write, document, and publish the new authentication module").

2. **Supervisor picks up task**: Supervisor agent calls `getNextTask()` on its home workspace. Receives the task. Calls `updateTaskStatus(taskId, "ongoing")`.

3. **Supervisor calls `getWorkspace()`**: Gets mission context and decompose the high-level goal into subtasks for each specialist workspace.

4. **Supervisor creates subtasks**: For each specialist workspace (coding, docs, publishing), the supervisor calls `createTask(title, body, assignee="agent")` on that workspace's MCP session. Each call returns a task ID.

5. **Specialist agents work**: Each specialist workspace agent runs its own `getNextTask` → `ongoing` → `reply` → `completed` cycle independently.

6. **Supervisor monitors**: Supervisor polls `getTaskMessages(taskId)` on each worker workspace to collect results. When a worker marks its task `completed`, the supervisor reads the final reply and uses that output as input for the next stage.

7. **Supervisor chains the pipeline**: Once Stage 1 (coding) completes, supervisor creates Stage 2 (docs) task in the docs workspace, embedding Stage 1's output in the task body. This continues through each stage.

8. **Supervisor reports back**: When all stages are complete, supervisor calls `reply(chatId, summary)` and `updateTaskStatus(supervisorTaskId, "completed")` in its home workspace. Human sees the full result.

---

## Request / Event Flow

### Task Creation (Human Path)

```
Human (UI)
  → POST /api/v1/workspaces/{id}/tasks
  → CRUD Controller (validates, persists, stores attachments)
  → Repository (GORM → SQLite / PostgreSQL)
  → PubSub event (ActionTaskCreate, ActorHuman)
  → Central Forwarder (PubSub → EventBus)
  → SSE stream → Human UI (task.created event)
  → MCP Manager notifies workspace agent (if task is agent-assigned)
```

### Task Creation (Agent Path via MCP)

```
Agent
  → MCP tool call: createTask(...)
  → WorkspaceServer callback
  → CRUD Controller (same validation + persistence path)
  → PubSub event (ActionTaskCreate, ActorAgent, OriginMCP)
  → Central Forwarder → EventBus SSE → Human UI
  → (if assignee="agent" in another workspace) Supervisor manages via separate MCP session
```

### Agent Task Execution Loop

```
Agent
  → getNextTask()          ← dequeues first notstarted task
  → updateTaskStatus(ongoing)
  → [work happens]
  → reply(chatId, "...")   ← intermediate progress updates
  → reply(chatId, result)  ← final output
  → updateTaskStatus(completed)
```

At each step, the MCP server persists the message and emits an SSE event so the human operator can follow along in real time.

---

## Permission Gating

Agents operating via Claude Code may request permission for sensitive tool calls (bash commands, file writes, etc.). AgentRQ intercepts these via the MCP protocol's notification channel:

1. **Agent requests permission** → MCP server receives `notifications/claude/channel/permission_request`
2. **Auto-allow check**: Is the tool in `workspace.AutoAllowedTools` or does the task have `AllowAllCommands = true`? If yes → auto-approve, record approval, continue.
3. **Human review**: Otherwise, a `Message` is created with `metadata.type = "permission_request"` and pushed to the human UI via SSE.
4. **Human responds** via `POST /workspaces/{id}/tasks/{taskId}/permission` with behavior: `allow`, `deny`, or `allow_always`.
5. **Verdict delivered** → MCP server sends `notifications/claude/channel/permission` back to the agent session. If `allow_always`, the tool is added to `workspace.AutoAllowedTools` for all future calls.

This gating applies per-session, per-workspace, and can be bypassed at the task or workspace level for trusted automation.

---

## Real-Time Event System

AgentRQ uses two complementary event systems:

### EventBus (SSE)
- In-memory pub/sub keyed by `workspaceID` and `userID`
- Human UI subscribes via `GET /workspaces/{id}/events`
- Events are JSON-encoded SSE frames: `data: {type, payload}\n\n`
- Slow consumers are dropped (non-blocking sends)
- Event types: `task.created`, `task.updated`, `status.updated`, `message.created`, `agent.connected`, `workspace.updated`

### PubSub (Internal)
- Separate async bus for system-level concerns
- Topics: `PubSubTopicCRUD` (entity lifecycle events), `PubSubTopicMCP` (tool call telemetry)
- Consumed by: Slack controller, notification service, push service, central SSE forwarder, telemetry

The **Central Forwarder** bridges them: it subscribes to the CRUD PubSub topic and translates entity events into workspace-scoped SSE events on the EventBus.

---

## Scheduled Tasks (Cron)

Tasks with `status = "cron"` are templates managed by the Scheduler service:

- Scheduler polls every 60 seconds
- Parses each template's 5-field cron schedule
- When the current minute matches, spawns a **child task** (copies title, body, attachments, `AllowAllCommands`; sets `ParentID`, `status = "notstarted"`)
- **One-time tasks** (fixed day/month): parent template is deleted after spawning
- **Recurring tasks** (wildcards in day/month): parent persists for future runs
- **Granularity rule**: minute field must be a single integer (0–59); wildcards/ranges/steps are rejected — enforcing a minimum hourly granularity

This allows supervisors to schedule recurring subtask generation without human intervention.

---

## Attachment Handling

Attachments are stored on disk under `./_storage/{attachmentID}` as base64-encoded files:
- On save: base64 data written to disk; only metadata (ID, filename, mimeType) persisted in the DB
- On download: `downloadAttachment` tool reads from disk, returns base64 to agent
- On delete: attachment files purged from disk; a cleanup service (`internal/service/cleanup`) runs daily to remove orphaned files older than the configured retention period

---

## Data Persistence

- **Primary**: PostgreSQL (optional, for production)
- **Fallback**: SQLite (default for development / single-node deployment)
- **ORM**: GORM with a repository abstraction layer — controllers never touch the DB directly
- **IDs**: Monoflake (monotonic snowflake-style) IDs generated in-process for all entities
- **Encryption**: Workspace tokens encrypted with AES-GCM using a server-side key from config

---

## System Topology

```
                        ┌─────────────────────────────┐
                        │         HTTP Server          │
                        │  (Fiber, unified port :3000) │
                        └─────────────┬───────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
    ┌─────────▼──────────┐  ┌────────▼───────────┐  ┌───────▼────────────┐
    │   REST API         │  │   MCP Handler      │  │   SSE Events       │
    │ /api/v1/workspaces │  │ /mcp/{workspaceID} │  │ /workspaces/{id}   │
    │ /api/v1/tasks      │  │                    │  │   /events          │
    └─────────┬──────────┘  └────────┬───────────┘  └───────┬────────────┘
              │                       │                       │
    ┌─────────▼──────────┐  ┌────────▼───────────┐  ┌───────▼────────────┐
    │   CRUD Controller  │  │   MCP Manager      │  │   EventBus (SSE)   │
    │  (tasks, workspaces│  │ (per-workspace     │  │  (workspace-scoped │
    │   messages, perms) │  │  WorkspaceServer)  │  │   in-memory pub/sub│
    └─────────┬──────────┘  └────────┬───────────┘  └────────────────────┘
              │                       │                         ▲
    ┌─────────▼───────────────────────▼──────────┐             │
    │               Repository (GORM)            │             │
    │         SQLite  ◄──────────►  PostgreSQL   │             │
    └────────────────────────────────────────────┘             │
              │                                                 │
    ┌─────────▼──────────┐          ┌──────────────────────────┘
    │   PubSub (internal)│─────────►│  Central Forwarder       │
    │  CRUD / MCP topics │          │  (PubSub → EventBus)     │
    └────────────────────┘          └──────────────────────────┘
```

---

## Security Boundaries

| Boundary | Mechanism |
|----------|-----------|
| Agent ↔ MCP Server | Workspace token (AES-GCM encrypted at rest, bearer token in transit) |
| Human ↔ REST API | JWT (session cookie or Authorization header) |
| Permission gating | Auto-allow rules + human approval flow via SSE + MCP notification |
| Workspace isolation | Each MCP server instance is workspace-scoped; no cross-workspace tool calls within a single session |
| Rate limiting | Per-user request counters for tasks, messages, and workspace creation |
| DDoS protection | Per-IP request counting with configurable block duration |

---

## Telemetry

All significant actions are recorded as telemetry events (`uint8 action`, `uint8 actor`) and batched to the database every 5 seconds. Tracked events include workspace and task lifecycle actions, message creation, MCP tool calls, permission approvals/denials, and scheduled task spawns. These power the workspace dashboard stats and global analytics.

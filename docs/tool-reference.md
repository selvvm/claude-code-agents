# Tool Reference

This document covers all 8 tools that power Claude Code's agent team orchestration. Each tool is documented with its purpose, parameters, and where it fits in the team lifecycle.

> Full JSON schemas are available in [`schemas/tools.json`](../schemas/tools.json).

---

## 1. TeamCreate

**Purpose:** Initialize a new team. This is always the first call in a team session.

**What it does:** Creates a team config file at `~/.claude/teams/{team-name}/config.json` and a corresponding task directory at `~/.claude/tasks/{team-name}/`.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `team_name` | Yes | string | Name for the new team |
| `description` | No | string | Team description/purpose |
| `agent_type` | No | string | Type/role of the team lead |

**Lifecycle position:** First call. Nothing else works without a team.

```json
{
  "team_name": "auth-refactor",
  "description": "Refactoring the authentication module",
  "agent_type": "architect"
}
```

---

## 2. TaskCreate

**Purpose:** Add a task to the shared team task list.

**What it does:** Persists a task as a JSON file under `~/.claude/tasks/{team-name}/`. The `description` field effectively becomes the prompt for whichever agent picks up the task.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `subject` | Yes | string | Short title (1-5 words) for quick scanning |
| `description` | Yes | string | Full task details — the agent's prompt |
| `activeForm` | No | string | Present-continuous form for UI spinner (e.g., "Reviewing auth module...") |
| `owner` | No | string | Pre-assign to a specific agent name |
| `blockedBy` | No | string[] | Task IDs that must complete first |
| `metadata` | No | object | Arbitrary metadata |

**Lifecycle position:** After TeamCreate, before spawning agents. Create the work items first so teammates have something to claim.

```json
{
  "subject": "Implement JWT validation",
  "description": "Add JWT token validation middleware to the Express app...",
  "activeForm": "Implementing JWT validation",
  "blockedBy": ["1"]
}
```

---

## 3. TaskList

**Purpose:** Get an overview of all tasks in the team's task list.

**What it does:** Returns task summaries (ID, subject, status, owner, blockedBy). Deliberately omits the full description and metadata to avoid context bloat.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `status` | No | enum | Filter by status: `pending`, `in_progress`, `completed`, `deleted` |

**Lifecycle position:** Called frequently — by the lead to monitor progress, by teammates to find available work.

**Key behavior:** Teammates look for tasks that are `pending`, have no `owner`, and have an empty `blockedBy` list. These are the tasks available for claiming.

---

## 4. TaskGet

**Purpose:** Retrieve full details of a specific task.

**What it does:** Returns everything TaskList omits — the full description, metadata, blockedBy/blocks arrays, and all other fields.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `taskId` | Yes | string | The ID of the task to retrieve |

**Lifecycle position:** Called by a teammate after identifying an available task via TaskList, to get the full prompt before starting work.

---

## 5. TaskUpdate

**Purpose:** Modify an existing task — claim it, complete it, set dependencies, or delete it.

**What it does:** Updates any field on a task. The most common operations are claiming (set `owner` + `status: "in_progress"`) and completing (`status: "completed"`).

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `taskId` | Yes | string | The ID of the task to update |
| `status` | No | enum | `pending`, `in_progress`, `completed`, or `deleted` |
| `subject` | No | string | New subject |
| `description` | No | string | New description |
| `activeForm` | No | string | Present-continuous form for UI spinner |
| `owner` | No | string | New owner (agent name) |
| `addBlockedBy` | No | string[] | Task IDs to add as blockers |
| `removeBlockedBy` | No | string[] | Task IDs to remove from blockers |
| `metadata` | No | object | Updated metadata |

**Lifecycle position:** Used throughout the team's lifetime. Critical for task state transitions.

```json
{
  "taskId": "3",
  "status": "in_progress",
  "owner": "backend-dev"
}
```

---

## 6. TeamDelete

**Purpose:** Clean up the team after all work is done.

**What it does:** Atomically removes the team config, all task files, all inbox files, and the lock file from disk.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| *(none)* | — | — | Takes no parameters |

**Lifecycle position:** Always the last call. Must be called after all teammates have shut down — will fail if active members remain.

**Important:** Send `shutdown_request` to all teammates and wait for their `shutdown_response` before calling TeamDelete.

---

## 7. SendMessage

**Purpose:** Inter-agent communication. Writes to the recipient's inbox file on disk.

**What it does:** Supports 5 message types for different coordination patterns.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `type` | Yes | enum | `message`, `broadcast`, `shutdown_request`, `shutdown_response`, `plan_approval_response` |
| `recipient` | Conditional | string | Agent name. Required for: `message`, `shutdown_request`, `plan_approval_response` |
| `content` | No | string | Message text, reason, or feedback |
| `summary` | Conditional | string | 5-10 word UI preview. Required for: `message`, `broadcast` |
| `approve` | Conditional | boolean | Approval flag. Required for: `shutdown_response`, `plan_approval_response` |
| `request_id` | Conditional | string | Request ID to respond to. Required for: `shutdown_response`, `plan_approval_response` |

### Message Types

| Type | Purpose | Recipients |
|------|---------|------------|
| `message` | Direct message to a single teammate | One specific agent |
| `broadcast` | Send to ALL teammates at once | Everyone (expensive — use sparingly) |
| `shutdown_request` | Ask a teammate to gracefully exit | One specific agent |
| `shutdown_response` | Approve or reject a shutdown request | The requester |
| `plan_approval_response` | Approve or reject a teammate's plan | The agent who submitted the plan |

**Lifecycle position:** Used throughout. Message delivery is automatic — agents don't need to poll their inbox.

---

## 8. Agent (Spawner)

**Purpose:** Spawn a new agent — either an isolated sub agent or a team member.

**What it does:** Creates a new agent process. The critical distinction: include `name` + `team_name` for a teammate, omit them for a sub agent.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `description` | Yes | string | Short (1-5 word) label |
| `prompt` | Yes | string | Full task/instructions |
| `subagent_type` | Yes | string | Agent type (e.g., `"general-purpose"`, `"Explore"`, `"Plan"`) |
| `name` | No* | string | Agent name — required for teammates |
| `team_name` | No* | string | Team to join — required for teammates |
| `model` | No | enum | `"sonnet"`, `"opus"`, `"haiku"` |
| `mode` | No | enum | Permission mode: `"acceptEdits"`, `"bypassPermissions"`, `"default"`, `"dontAsk"`, `"plan"` |
| `run_in_background` | No | boolean | Run without blocking the parent |
| `max_turns` | No | integer | Maximum agentic turns |
| `resume` | No | string | Agent ID to resume a previous session |

*\* Required together to make the agent a teammate instead of a sub agent.*

**Lifecycle position:** After TeamCreate + TaskCreate. Spawn teammates to work on the created tasks.

```json
{
  "description": "Backend developer",
  "prompt": "You are the backend developer. Check TaskList for available tasks...",
  "subagent_type": "general-purpose",
  "name": "backend-dev",
  "team_name": "auth-refactor",
  "model": "sonnet",
  "run_in_background": true
}
```

---

## Tool Lifecycle Summary

```
TeamCreate          ──►  Create the team
    │
TaskCreate (×N)     ──►  Define the work items
    │
Agent (×N)          ──►  Spawn teammates to do the work
    │
TaskList / TaskGet  ──►  Teammates find and read tasks
    │
TaskUpdate          ──►  Claim tasks, mark complete
    │
SendMessage         ──►  Coordinate, report, request shutdown
    │
TeamDelete          ──►  Clean up everything
```

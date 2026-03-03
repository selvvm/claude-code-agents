# Orchestration Guide

Practical tips for prompting Claude Code to use agent teams effectively.

## Team Lifecycle

Every team session follows the same pattern:

```
1. TeamCreate       →  Initialize the team
2. TaskCreate (×N)  →  Define work items
3. Agent (×N)       →  Spawn teammates
4. [Work happens]   →  Teammates claim tasks, collaborate, complete work
5. SendMessage      →  Shutdown requests to all teammates
6. TeamDelete       →  Clean up team files
```

## Prompting for Teams

Claude Code doesn't automatically use teams. You need to signal that the task warrants parallel collaboration. Effective prompts include:

```
"Use a team to implement this feature. Create tasks for the frontend
and backend work, and spawn teammates to handle them in parallel."
```

```
"This is a large refactor. Set up a team with a researcher to analyze
the codebase and a developer to implement the changes."
```

```
"Parallelize this — create a team and assign each module to a
different teammate."
```

The key phrases that trigger team behavior: **"use a team"**, **"spawn teammates"**, **"parallelize"**, **"work in parallel"**, **"create a swarm"**.

## Model Selection Strategy

A common and effective pattern:

| Role | Model | Why |
|------|-------|-----|
| Team lead | `opus` | Needs strong reasoning for task decomposition and coordination |
| Teammates | `sonnet` | Good enough for focused implementation tasks, much faster |
| Quick lookups | `haiku` | Fast, cheap — used for sub agents, not teammates |

The lead stays on the default model (usually Opus) while spawning teammates on Sonnet:

```json
{
  "name": "implementer",
  "team_name": "my-project",
  "model": "sonnet",
  "run_in_background": true
}
```

## Task Design

### Good Task Descriptions

The task `description` is the agent's prompt. Write it like you're briefing a developer:

```json
{
  "subject": "Add rate limiting",
  "description": "Add rate limiting middleware to the Express API. Use express-rate-limit. Limit to 100 requests per 15-minute window per IP. Apply to all /api/* routes. Add appropriate 429 response with retry-after header. Write tests.",
  "activeForm": "Adding rate limiting middleware"
}
```

### Task Dependencies

Use `blockedBy` to enforce ordering:

```json
// Task 1: Design the schema
{ "subject": "Design DB schema", "description": "..." }

// Task 2: Implement migrations (blocked by schema design)
{ "subject": "Create migrations", "description": "...", "blockedBy": ["1"] }

// Task 3: Write API endpoints (blocked by migrations)
{ "subject": "Build API endpoints", "description": "...", "blockedBy": ["2"] }

// Task 4: Write tests (can run in parallel with task 3)
{ "subject": "Write unit tests", "description": "...", "blockedBy": ["2"] }
```

Tasks 3 and 4 both depend on task 2, so they can run in parallel once task 2 completes.

### Self-Claiming vs Pre-Assignment

**Self-claiming** (leave `owner` blank): Teammates call `TaskList`, find unowned/unblocked tasks, and claim them with `TaskUpdate`. Good for homogeneous teams where any agent can do any task.

**Pre-assignment** (set `owner`): Assign tasks to specific teammates by name. Good when you have specialized agents (frontend-dev, backend-dev, tester).

## Background Execution

Always spawn teammates with `run_in_background: true`. This lets the lead continue orchestrating without waiting for each teammate to finish.

```json
{
  "name": "researcher",
  "run_in_background": true
}
```

Without this, the lead blocks on each teammate — defeating the purpose of parallelism.

## Communication Patterns

### Direct Messages

For targeted coordination between two agents:

```json
{
  "type": "message",
  "recipient": "backend-dev",
  "content": "The auth endpoint is ready. You can start integrating.",
  "summary": "Auth endpoint ready for integration"
}
```

### Broadcasts (Use Sparingly)

Sends to ALL teammates. Each broadcast = N message deliveries. Only use for critical team-wide announcements:

```json
{
  "type": "broadcast",
  "content": "Blocking bug found in the shared module. Stop all work until resolved.",
  "summary": "Critical blocking issue found"
}
```

### Shutdown Flow

Always shut down teammates gracefully before calling TeamDelete:

```json
// 1. Request shutdown
{
  "type": "shutdown_request",
  "recipient": "backend-dev",
  "content": "All tasks complete, wrapping up"
}

// 2. Teammate responds (approve)
{
  "type": "shutdown_response",
  "request_id": "abc-123",
  "approve": true
}

// 3. After all teammates confirm → TeamDelete
```

## Common Patterns

### The Research-Then-Implement Pattern

```
Team Lead (opus)
  ├── Spawns: researcher (sonnet) → explores codebase, reports findings
  ├── Spawns: planner (sonnet, mode: "plan") → designs implementation
  └── After research + plan complete:
      ├── Spawns: implementer-1 (sonnet) → builds feature A
      └── Spawns: implementer-2 (sonnet) → builds feature B
```

### The Frontend-Backend Split

```
Team Lead (opus)
  ├── Creates tasks for API + UI work
  ├── Spawns: backend-dev (sonnet) → API endpoints
  ├── Spawns: frontend-dev (sonnet) → React components
  └── Spawns: tester (sonnet, blocked by both) → integration tests
```

### The Divide-and-Conquer Pattern

```
Team Lead (opus)
  ├── Splits a large refactor into N independent modules
  ├── Creates N tasks (one per module)
  └── Spawns N teammates (sonnet) → each claims one module
```

## Troubleshooting

### Teammates Not Finding Tasks

- Make sure tasks exist (call `TaskCreate` before spawning agents)
- Check that `blockedBy` dependencies have been resolved
- Ensure tasks don't have an `owner` set (unless pre-assigning)

### Team Lead Getting Blocked

- Use `run_in_background: true` for all teammates
- Don't wait for teammate results unless you need them before proceeding

### TeamDelete Failing

- Send `shutdown_request` to all teammates first
- Wait for `shutdown_response` (approve) from each
- Only then call `TeamDelete`

### Context Window Getting Too Large

- Use sub agents for quick lookups instead of loading everything into the lead's context
- Write task descriptions that are self-contained — don't assume teammates have the lead's context
- Use `TaskList` (lightweight) instead of `TaskGet` (full details) when you just need an overview

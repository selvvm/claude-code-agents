# Sub Agent vs Team Agent: A Deep Dive

Claude Code supports two fundamentally different models for multi-agent orchestration. Understanding the distinction is key to using them effectively.

## Sub Agent Model

![Sub Agent Model](../images/sub-agent-model.png)

A **Sub Agent** is a lightweight, isolated agent spawned by a parent to handle a single task. It operates in a strict parent-child hierarchy.

### Characteristics

- **Isolated execution** — The sub agent has no awareness of other agents. It can't message peers or access a shared task list.
- **Linear control flow** — The parent spawns a sub agent, waits for the result, and continues. It's a function call, not a collaboration.
- **No shared state** — There is no team config, no task directory, no inbox. The sub agent receives a prompt, does the work, and returns a result.
- **Scoped context** — The parent can pass conversation context to the sub agent, but the sub agent doesn't accumulate knowledge across invocations.
- **Ephemeral** — Once the sub agent returns, it's gone. No resumption, no persistence.

### When to Use Sub Agents

- Quick, focused lookups (e.g., "find all files matching this pattern")
- Parallel research queries where the results don't need to interact
- Tasks where you want to protect the main context window from noise
- Simple delegation where coordination isn't needed

### How It's Spawned

```json
{
  "description": "Search for auth endpoints",
  "prompt": "Find all API endpoints related to authentication...",
  "subagent_type": "Explore"
}
```

Note the absence of `name` and `team_name` — that's what keeps it a sub agent.

---

## Team Agent Model

![Team Agent Model](../images/team-agent-model.png)

A **Team Agent** (teammate) is a persistent, named agent that joins a team. It can message peers, claim tasks from a shared list, and coordinate through an inbox-based communication system.

### Characteristics

- **Named identity** — Each teammate has a name (e.g., `"researcher"`, `"frontend-dev"`). This name is used for messaging and task ownership.
- **Shared task list** — All teammates read from and write to the same task list, stored as JSON files under `~/.claude/tasks/{team-name}/`.
- **Peer-to-peer messaging** — Teammates can send direct messages, broadcasts, shutdown requests, and plan approval flows via `SendMessage`.
- **Team lead coordination** — Typically one agent (on a more capable model like Opus) acts as the team lead, creating tasks and assigning them to teammates (often on Sonnet).
- **Background execution** — Teammates usually run in the background (`run_in_background: true`) so the lead isn't blocked.
- **Resumable** — Teammates can be resumed by passing their agent ID to the `resume` parameter.

### When to Use Team Agents

- Complex, multi-step projects that benefit from parallel work
- Tasks requiring different specializations (frontend, backend, testing)
- Work where agents need to coordinate and react to each other's progress
- Projects where a shared task board drives the workflow

### How It's Spawned

```json
{
  "description": "Frontend implementation",
  "prompt": "You are the frontend developer. Check TaskList for available work...",
  "subagent_type": "general-purpose",
  "name": "frontend-dev",
  "team_name": "my-project",
  "model": "sonnet",
  "run_in_background": true
}
```

The presence of `name` + `team_name` is what makes it a teammate.

---

## Key Differences

| Aspect | Sub Agent | Team Agent |
|--------|-----------|------------|
| **Identity** | Anonymous | Named (e.g., `"researcher"`) |
| **Communication** | Returns result to parent only | Messages any peer via `SendMessage` |
| **Task management** | None — receives prompt, returns result | Shared task list (`TaskCreate`, `TaskList`, `TaskUpdate`) |
| **Persistence** | Ephemeral — gone after return | Persistent — can be resumed |
| **State sharing** | None | Shared team config + task directory on disk |
| **Coordination** | Parent orchestrates everything | Peers self-organize around tasks |
| **Typical model** | Any (often haiku for speed) | Lead on opus, teammates on sonnet |
| **Lifecycle** | Spawn → execute → return | TeamCreate → TaskCreate → spawn → collaborate → shutdown → TeamDelete |
| **Background execution** | Optional | Typical (`run_in_background: true`) |
| **Overhead** | Low | Higher (team config, task files, inboxes) |

## The Spawning Distinction

The single most important architectural detail: **the same `Agent` tool spawns both**. The difference is entirely in the parameters:

```
# Sub Agent — no name, no team
Agent(description, prompt, subagent_type)

# Teammate — has name + team_name
Agent(description, prompt, subagent_type, name, team_name)
```

This means the team model is an *extension* of the sub agent model, not a replacement. You can mix both in the same session — use sub agents for quick lookups while a team handles the heavy lifting.

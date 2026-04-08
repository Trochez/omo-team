# OMO-Team Skill Datasheet

## Overview

| Property | Value |
|----------|-------|
| **Name** | `omo-team` |
| **Version** | 1.1.0 |
| **Type** | Orchestration Skill |
| **Compatible With** | OpenCode 1.x, oh-my-opencode 3.x+ |
| **Purpose** | Spawn coordinated OpenCode subagent sessions for parallel task execution |
| **Location** | `~/.agents/skills/omo-team/SKILL.md` |
| **New in 1.1.0** | Automatic worker count estimation based on task complexity |

---

## Description

`omo-team` is an OpenCode-native parallel execution skill that spawns real OpenCode subagent sessions (not OMX/Codex CLI workers). It coordinates multiple workers through the `task()` tool with background execution, enabling true parallel processing within the OpenCode ecosystem.

**Key Features:**
- **Auto-estimation**: Automatically determines optimal worker count based on task complexity
- **Native OpenCode**: Spawns OpenCode subagent sessions (same type as leader)
- **Parallel execution**: True parallelism via `run_in_background=true`

**Key Difference from OMX Team:**
- **OMX Team**: Spawns Codex/Claude CLI sessions via tmux (GPT models)
- **OMO Team**: Spawns OpenCode subagent sessions (same session type as leader)

---

## Invocation

### Slash Command
```
/omo-team [N] "<task description>" [options]
```

### Via Task Tool
```typescript
task(
  subagent_type="omo-team",
  load_skills=[],
  run_in_background=false,
  prompt="Spawn workers to <task description>"
)
```

---

## Arguments

### Required Arguments

| Argument | Type | Required | Description | Example |
|----------|------|----------|-------------|---------|
| `N` | number | No | Number of workers (omit for auto-estimation) | `3` |
| `"<task description>"` | string | Yes | Main task to decompose and distribute | `"analyze auth module"` |

### Optional Arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--category=<cat>` | string | Auto | Category for worker subagents |
| `--skills=<list>` | string | `[]` | Comma-separated skills to load in workers |
| `--timeout=<ms>` | number | `300000` | Maximum wait time per worker (5 min) |
| `--auto` | flag | False | Force auto-estimation (ignores N if provided) |

---

## Automatic Worker Count Estimation

### How It Works

When `N` is omitted, the skill analyzes task complexity and estimates optimal workers:

```
Complexity Score (0-100) → Worker Count

Score 0-19   → 1 worker  (trivial)
Score 20-39  → 2 workers (simple)
Score 40-59  → 3 workers (moderate)
Score 60-79  → 4 workers (complex)
Score 80-100 → 5 workers (maximum)
```

### Complexity Factors

| Factor | Weight | Indicators |
|--------|--------|------------|
| Scope Breadth | 30% | "entire", "all", "comprehensive", multiple modules |
| Task Diversity | 25% | Multiple verbs, heterogeneous subtasks |
| Domain Complexity | 20% | Technical terms, architecture, algorithms |
| File Count | 15% | Explicit file/directory mentions |
| Integration Points | 10% | API, database, external services |

### Examples

| Task | Estimated | Reasoning |
|------|-----------|-----------|
| "fix typo in README" | 1 | Single file, trivial |
| "add tests for auth" | 2 | Single module, focused |
| "analyze and document API" | 3 | Two phases, moderate |
| "refactor for strict mode" | 4 | High scope, multiple modules |
| "build microservices" | 5 | Maximum complexity |

---

## Automatic Role Assignment

### Overview

Each worker is automatically assigned a **role** based on their subtask. Roles map directly to **oh-my-opencode agents** and **categories** for optimal execution.

### Primary Agents (mode: primary)

| Role | Agent | Category | Focus | Reasoning |
|------|-------|----------|-------|-----------|
| `orchestrator` | `sisyphus` | orchestrator | Multi-agent coordination | xhigh |
| `executor` | `hephaestus` | executor | Implementation, coding | high |
| `planner` | `prometheus` | planner | Strategic planning | xhigh |

### Subagent Types (mode: subagent)

| Role | Agent | Category | Focus | Reasoning |
|------|-------|----------|-------|-----------|
| `consultant` | `oracle` | consultant | Architecture, debugging | xhigh |
| `searcher` | `explore` | search | Codebase exploration | minimal |
| `analyst` | `metis` | analyst | Requirements analysis | xhigh |
| `reviewer` | `momus` | reviewer | Quality assurance | xhigh |
| `researcher` | `librarian` | research | Documentation lookup | minimal |
| `visual` | `multimodal-looker` | visual | Image/diagram analysis | medium |
| `knowledge` | `atlas` | knowledge | Context management | medium |
| `junior-orchestrator` | `sisyphus-junior` | orchestrator-junior | Light coordination | medium |

### Task Categories (for delegation)

| Category | Use Case | Typical Model |
|----------|----------|---------------|
| `visual-engineering` | UI/UX, frontend | qwen2.5-vl-72b-instruct |
| `ultrabrain` | Hard logic, algorithms | gpt-5.4 |
| `deep` | Autonomous problem-solving | gpt-5.3-codex |
| `artistry` | Creative solutions | gemini-3.1-pro |
| `quick` | Trivial tasks | glm5 |
| `unspecified-low` | Low effort tasks | claude-sonnet-4-6 |
| `unspecified-high` | High effort tasks | claude-sonnet-4-6 |
| `writing` | Documentation | gemini-3-flash |

### Role Detection Keywords

| Keywords in Subtask | Role | Agent | Category |
|---------------------|------|-------|----------|
| test, spec, verify, qa | `reviewer` | `momus` | reviewer |
| document, readme, doc, write | `researcher` | `librarian` | writing |
| review, audit, check, security | `consultant` | `oracle` | consultant |
| analyze, investigate, debug, why | `consultant` | `oracle` | consultant |
| search, find, explore, locate | `searcher` | `explore` | search |
| research, lookup, docs, api | `researcher` | `librarian` | research |
| frontend, ui, component, style, css | `visual` | `multimodal-looker` | visual-engineering |
| image, diagram, screenshot, visual | `visual` | `multimodal-looker` | visual |
| plan, design, architect | `planner` | `prometheus` | planner |
| coordinate, orchestrate, manage | `junior-orchestrator` | `sisyphus-junior` | orchestrator-junior |
| implement, build, create, develop | `executor` | `hephaestus` | executor |
| api, endpoint, backend, server | `executor` | `hephaestus` | deep |
| database, schema, query, sql | `executor` | `hephaestus` | deep |
| deploy, ci, cd, pipeline | `executor` | `hephaestus` | unspecified-high |
| (no match) | `executor` | `hephaestus` | unspecified-high |

### Role Assignment Example

```
Task: "implement user authentication with tests and docs"

Auto-Decomposition:
├── worker-1: "implement login API" → role: executor, agent: hephaestus, category: deep
├── worker-2: "create auth UI components" → role: visual, agent: multimodal-looker, category: visual-engineering
├── worker-3: "write unit tests" → role: reviewer, agent: momus, category: reviewer
└── worker-4: "document API endpoints" → role: researcher, agent: librarian, category: writing

Each worker receives:
- Role name and agent assignment
- Appropriate category for task type
- Agent-specific skills pre-loaded
- Full context in worker prompt
```

### Agent Skills (from oh-my-opencode.json)

| Agent | Pre-loaded Skills |
|-------|-------------------|
| `sisyphus` | `ralph`, `autopilot`, `ultrawork` |
| `hephaestus` | `[]` |
| `oracle` | `[]` |
| `explore` | `[]` |
| `prometheus` | `plan`, `ralplan` |
| `metis` | `[]` |
| `momus` | `[]` |
| `librarian` | `[]` |
| `multimodal-looker` | `[]` |
| `atlas` | `[]` |
| `sisyphus-junior` | `[]` |

---

## Skill Loading Options

| Skill | Use When |

---

## Usage Examples

### Basic Usage
```
/omo-team 3 "analyze the authentication module"
```
Spawns 3 workers with default category (`unspecified-high`) to analyze auth.

### With Category
```
/omo-team 2 "implement REST API endpoints" --category=deep
```
Spawns 2 workers with `deep` category for thorough implementation.

### With Skills
```
/omo-team 2 "review code for security issues" --category=deep --skills=security-review
```
Spawns 2 workers with `security-review` skill loaded.

### Complex Task
```
/omo-team 4 "analyze and document the entire codebase architecture" --category=deep --skills=deep-research,writing
```
Spawns 4 workers for comprehensive analysis and documentation.

---

## Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. PRE-CONTEXT INTAKE                                           │
│    • Derive team name from request                              │
│    • Create .omo/state/omo-team/<team>/manifest.json            │
│    • Decompose task into N subtasks                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. SPAWN WORKERS (Parallel)                                     │
│    • task(category, load_skills, run_in_background=true)        │
│    • Collect task_ids for each worker                           │
│    • Store in .omo/state/omo-team/<team>/task-ids.json          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. MONITOR PROGRESS                                             │
│    • Poll background_output(task_id, block=false)               │
│    • Update .omo/state/omo-team/<team>/status.json              │
│    • Wait for all workers to complete                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. AGGREGATE RESULTS                                            │
│    • Read .omo/state/omo-team/<team>/workers/worker-N/result.md │
│    • Synthesize into final report                               │
│    • Present to user                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## State Files

### Team Manifest
**Location**: `.omo/state/omo-team/<team>/manifest.json`

```json
{
  "team_name": "auth-analysis",
  "created_at": "2026-04-07T18:00:00Z",
  "worker_count": 3,
  "status": "active",
  "tasks": ["task-1", "task-2", "task-3"],
  "description": "analyze authentication module"
}
```

### Task IDs
**Location**: `.omo/state/omo-team/<team>/task-ids.json`

```json
{
  "worker-1": "bg_abc123",
  "worker-2": "bg_def456",
  "worker-3": "bg_ghi789"
}
```

### Worker Result
**Location**: `.omo/state/omo-team/<team>/workers/<worker-id>/result.md`

```markdown
# Worker: worker-1
## Task: <subtask description>
### Summary
<Brief summary of work done>
### Findings
<Detailed results>
### Files Modified
<List of changed files>
### Recommendations
<Suggestions for the team>
```

### Team Status
**Location**: `.omo/state/omo-team/<team>/status.json`

```json
{
  "pending": 0,
  "in_progress": 2,
  "completed": 1,
  "failed": 0
}
```

---

## Environment Requirements

| Requirement | Check | Error if Missing |
|-------------|-------|------------------|
| OpenCode Session | `OPENCODE=1` env var | "This skill requires OpenCode session" |
| Task Tool | Available in OpenCode | "task() tool not available" |
| Write Permissions | `.omo/state/` writable | "Cannot create team state directory" |

---

## Error Handling

### Worker Failure
1. Check `background_output()` for error details
2. Log failure in `.omo/state/omo-team/<team>/errors.json`
3. Optionally respawn worker with adjusted task
4. Continue with remaining workers

### Timeout
1. Default timeout: 5 minutes per worker
2. Use `background_cancel(task_id)` for stuck workers
3. Report partial results with timeout notice

### Partial Completion
1. Aggregate available results
2. Mark incomplete workers in status
3. Present partial findings to user

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `/plan` | Plan first, then execute with omo-team |
| `/ralph` | Use Ralph for verification after team completes |
| `/ultraqa` | QA cycle after team execution |
| `omo-worker` | Loaded in each spawned worker |

---

## Anti-Patterns (DO NOT)

| ❌ Don't | ✅ Do Instead |
|----------|---------------|
| Use `omx team` CLI | Use `task()` tool |
| Spawn workers sequentially | Use `run_in_background=true` for parallelism |
| Forget to track task_ids | Store in `task-ids.json` |
| Skip result aggregation | Synthesize all worker outputs |
| Use outside OpenCode | Verify `OPENCODE=1` first |

---

## Comparison: OMO-Team vs OMX-Team

| Aspect | OMO-Team | OMX-Team |
|--------|----------|----------|
| **Worker Type** | OpenCode subagent sessions | Codex/Claude CLI sessions |
| **Spawning Method** | `task()` tool | `omx team` CLI + tmux |
| **Session Type** | OpenCode (same as leader) | OMX (GPT models) |
| **State Location** | `.omo/state/omo-team/` | `.omx/state/team/` |
| **Coordination** | Background task polling | tmux panes + mailbox |
| **Requires tmux** | No | Yes |
| **Works in OpenCode** | ✅ Yes | ❌ No (spawns OMX sessions) |

---

## Troubleshooting

### Workers Not Spawning
- Verify `OPENCODE=1` environment variable
- Check `task()` tool is available
- Ensure `.omo/state/` directory is writable

### Results Not Written
- Check worker prompts include output path
- Verify `.omo/state/omo-team/<team>/workers/` exists
- Review `background_output()` for errors

### Timeout Issues
- Increase `--timeout` argument
- Check for blocking operations in workers
- Review worker logs for stuck processes

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-04-07 | Initial release |

---

## Author & License

- **Author**: OpenCode Community
- **License**: Open Source
- **Repository**: `~/.agents/skills/omo-team/`

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│                    OMO-TEAM QUICK REF                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  USAGE:                                                      │
│    /omo-team N "task" [--category=CAT] [--skills=S1,S2]     │
│                                                              │
│  EXAMPLES:                                                   │
│    /omo-team 3 "analyze auth module"                        │
│    /omo-team 2 "implement API" --category=deep              │
│    /omo-team 4 "review code" --skills=security-review       │
│                                                              │
│  CATEGORIES:                                                 │
│    deep | quick | visual-engineering | ultrabrain           │
│    writing | unspecified-high | unspecified-low             │
│                                                              │
│  STATE LOCATION:                                             │
│    .omo/state/omo-team/<team>/                              │
│                                                              │
│  REQUIREMENTS:                                               │
│    OPENCODE=1 (must be in OpenCode session)                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

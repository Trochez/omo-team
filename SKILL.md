---
name: omo-team
description: N coordinated OpenCode subagents on shared task list using native OpenCode session spawning
---

# OMO Team Skill (OpenCode-Native Team Orchestration)

`$omo-team` is the OpenCode-native parallel execution mode. It spawns real OpenCode subagent sessions (not OMX/Codex CLI) and coordinates them through the `task()` tool with background execution.

**CRITICAL**: This skill ONLY works in OpenCode sessions. It uses OpenCode's native subagent spawning infrastructure, NOT OMX's tmux-based workers.

## What This Skill Does

When user triggers `$omo-team`, the agent must:

1. **Detect OpenCode context** - Verify `OPENCODE=1` environment variable
2. **Spawn OpenCode subagents** - Use `task()` tool with `run_in_background=true`
3. **Coordinate via state** - Track tasks in `.omo/state/omo-team/<team>/`
4. **Monitor completion** - Poll background tasks until all complete
5. **Aggregate results** - Collect and synthesize outputs from all workers

## Key Difference from OMX Team

| Aspect | OMX Team | OMO Team (this skill) |
|--------|----------|----------------------|
| Worker Type | Codex/Claude CLI sessions | OpenCode subagent sessions |
| Spawning | `omx team ...` CLI + tmux | `task()` tool with background |
| State Location | `.omx/state/team/` | `.omo/state/omo-team/` |
| Coordination | tmux panes + mailbox files | Background task polling |
| Session Type | OMX sessions | OpenCode sessions (same as leader) |

## Invocation Contract

```
/omo-team [N] "<task description>" [--category=<cat>] [--skills=<skill1,skill2>] [--auto]
```

### Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `N` | No | Auto | Number of workers (omit for auto-estimation) |
| `"<task>"` | Yes | - | Task description to decompose |
| `--category` | No | Auto | Category for workers |
| `--skills` | No | `[]` | Comma-separated skills to load |
| `--auto` | No | False | Force automatic worker count estimation |

Examples:

```
# Explicit worker count
/omo-team 3 "analyze feature X and report flaws"

# Auto-estimated workers (recommended)
/omo-team "debug flaky integration tests"

# Auto with category and skills
/omo-team "implement API endpoints" --category=deep --skills=typescript-programmer

# Force auto-estimation even with number
/omo-team 5 "simple typo fix" --auto
```

## Automatic Worker Count Estimation

When `N` is not specified, the skill **automatically estimates** the optimal number of workers based on task complexity analysis.

### Complexity Analysis Factors

| Factor | Weight | Indicators |
|--------|--------|------------|
| **Scope Breadth** | 30% | Number of modules/files mentioned, "entire", "all", "comprehensive" |
| **Task Diversity** | 25% | Multiple verbs ("analyze and implement"), heterogeneous subtasks |
| **Domain Complexity** | 20% | Technical terms, architecture decisions, algorithm mentions |
| **File Count Estimate** | 15% | Explicit file mentions, directory references |
| **Integration Points** | 10% | API mentions, database, external services, cross-module work |

### Estimation Algorithm

```
COMPLEXITY_SCORE = 0-100

IF complexity_score < 20:
    workers = 1  # Single worker, trivial task
ELIF complexity_score < 40:
    workers = 2  # Small team, simple decomposition
ELIF complexity_score < 60:
    workers = 3  # Medium team, moderate complexity
ELIF complexity_score < 80:
    workers = 4  # Large team, complex task
ELSE:
    workers = 5  # Maximum recommended for coordination

# Cap at 5 workers (diminishing returns beyond this)
workers = min(workers, 5)
```

### Complexity Examples

| Task Description | Estimated Workers | Reasoning |
|------------------|-------------------|-----------|
| "fix typo in README" | 1 | Single file, trivial scope |
| "add unit tests for auth module" | 2 | Single module, focused task |
| "analyze and document API endpoints" | 3 | Two phases (analyze + document), moderate scope |
| "refactor entire codebase for TypeScript strict mode" | 4 | High scope breadth, multiple modules, architecture impact |
| "implement full microservices architecture with auth, API, and database" | 5 | Maximum complexity, multiple domains, integration points |

### Task Decomposition Patterns

When auto-estimating, use these decomposition strategies:

#### 1 Worker (Score 0-19)
- Single file/module changes
- Trivial fixes (typos, formatting)
- Simple lookups or reports
- **Role**: `executor` - Single focused task

#### 2 Workers (Score 20-39)
- **Parallel decomposition**: Split by file/module
- **Sequential decomposition**: Phase 1 → Phase 2
- **Roles**: `analyzer` + `implementer` or `worker-a` + `worker-b`
- Example: "analyze X" → worker-1 analyzes, worker-2 implements fixes

#### 3 Workers (Score 40-59)
- **Functional decomposition**: Split by feature area
- **Layer decomposition**: Frontend / Backend / Database
- **Roles**: `frontend-specialist`, `backend-specialist`, `qa-specialist`
- Example: "implement feature X" → API + Logic + Tests

#### 4 Workers (Score 60-79)
- **Domain decomposition**: Split by business domain
- **Component decomposition**: Multiple independent components
- **Roles**: `core-developer`, `utils-developer`, `test-engineer`, `doc-writer`
- Example: "refactor module" → Core + Utils + Tests + Docs

#### 5 Workers (Score 80-100)
- **Architecture decomposition**: Major system components
- **Full-stack decomposition**: Multiple layers × multiple features
- **Roles**: `auth-specialist`, `api-architect`, `db-engineer`, `frontend-lead`, `integration-tester`
- Example: "build system" → Auth + API + DB + Frontend + Tests

---

## Automatic Role Assignment

When workers are spawned, each is automatically assigned a **role** based on their subtask. Roles provide context and specialization hints to workers.

### Role Types

Roles are mapped to **oh-my-opencode agents** and **categories** for optimal task execution.

#### Primary Agents (mode: primary)

| Role | Agent | Category | Focus | Skills | Reasoning |
|------|-------|----------|-------|--------|-----------|
| `orchestrator` | `sisyphus` | orchestrator | Multi-agent coordination | `ralph`, `autopilot`, `ultrawork` | xhigh |
| `executor` | `hephaestus` | executor | Implementation, coding | `[]` | high |
| `planner` | `prometheus` | planner | Strategic planning | `plan`, `ralplan` | xhigh |

#### Subagent Types (mode: subagent)

| Role | Agent | Category | Focus | Skills | Reasoning |
|------|-------|----------|-------|--------|-----------|
| `consultant` | `oracle` | consultant | Architecture, debugging | `[]` | xhigh |
| `searcher` | `explore` | search | Codebase exploration | `[]` | minimal |
| `analyst` | `metis` | analyst | Requirements analysis | `[]` | xhigh |
| `reviewer` | `momus` | reviewer | Quality assurance | `[]` | xhigh |
| `researcher` | `librarian` | research | Documentation lookup | `[]` | minimal |
| `visual` | `multimodal-looker` | visual | Image/diagram analysis | `[]` | medium |
| `knowledge` | `atlas` | knowledge | Context management | `[]` | medium |
| `junior-orchestrator` | `sisyphus-junior` | orchestrator-junior | Light coordination | `[]` | medium |

#### Task Categories (for delegation)

| Category | Use Case | Typical Model | Reasoning |
|----------|----------|---------------|-----------|
| `visual-engineering` | UI/UX, frontend | qwen2.5-vl-72b-instruct | medium |
| `ultrabrain` | Hard logic, algorithms | gpt-5.4 | xhigh |
| `deep` | Autonomous problem-solving | gpt-5.3-codex | high |
| `artistry` | Creative solutions | gemini-3.1-pro | high |
| `quick` | Trivial tasks | glm5 | low |
| `unspecified-low` | Low effort tasks | claude-sonnet-4-6 | low |
| `unspecified-high` | High effort tasks | claude-sonnet-4-6 | high |
| `writing` | Documentation | gemini-3-flash | medium |

### Role Assignment Algorithm

```python
def assign_role(subtask_description, team_context):
    subtask_lower = subtask_description.lower()
    
    # Map to oh-my-opencode agents based on task type
    if any(kw in subtask_lower for kw in ["test", "spec", "verify", "qa"]):
        return {"role": "reviewer", "agent": "momus", "category": "reviewer"}
    elif any(kw in subtask_lower for kw in ["document", "readme", "doc", "write"]):
        return {"role": "researcher", "agent": "librarian", "category": "writing"}
    elif any(kw in subtask_lower for kw in ["review", "audit", "check", "security"]):
        return {"role": "consultant", "agent": "oracle", "category": "consultant"}
    elif any(kw in subtask_lower for kw in ["analyze", "investigate", "debug", "why"]):
        return {"role": "consultant", "agent": "oracle", "category": "consultant"}
    elif any(kw in subtask_lower for kw in ["search", "find", "explore", "locate"]):
        return {"role": "searcher", "agent": "explore", "category": "search"}
    elif any(kw in subtask_lower for kw in ["research", "lookup", "docs", "api"]):
        return {"role": "researcher", "agent": "librarian", "category": "research"}
    elif any(kw in subtask_lower for kw in ["frontend", "ui", "component", "style", "css"]):
        return {"role": "visual", "agent": "multimodal-looker", "category": "visual-engineering"}
    elif any(kw in subtask_lower for kw in ["image", "diagram", "screenshot", "visual"]):
        return {"role": "visual", "agent": "multimodal-looker", "category": "visual"}
    elif any(kw in subtask_lower for kw in ["plan", "design", "architect"]):
        return {"role": "planner", "agent": "prometheus", "category": "planner"}
    elif any(kw in subtask_lower for kw in ["coordinate", "orchestrate", "manage"]):
        return {"role": "junior-orchestrator", "agent": "sisyphus-junior", "category": "orchestrator-junior"}
    elif any(kw in subtask_lower for kw in ["implement", "build", "create", "develop"]):
        return {"role": "executor", "agent": "hephaestus", "category": "executor"}
    elif any(kw in subtask_lower for kw in ["api", "endpoint", "backend", "server"]):
        return {"role": "executor", "agent": "hephaestus", "category": "deep"}
    elif any(kw in subtask_lower for kw in ["database", "schema", "query", "sql"]):
        return {"role": "executor", "agent": "hephaestus", "category": "deep"}
    elif any(kw in subtask_lower for kw in ["deploy", "ci", "cd", "pipeline"]):
        return {"role": "executor", "agent": "hephaestus", "category": "unspecified-high"}
    else:
        # Default to executor for implementation tasks
        return {"role": "executor", "agent": "hephaestus", "category": "unspecified-high"}
```

### Role Assignment Examples

| Task Decomposition | Worker | Role | Agent | Category |
|--------------------|--------|------|-------|----------|
| "analyze auth flow for vulnerabilities" | worker-1 | `consultant` | `oracle` | consultant |
| "implement login endpoint" | worker-2 | `executor` | `hephaestus` | deep |
| "create login UI component" | worker-3 | `visual` | `multimodal-looker` | visual-engineering |
| "write unit tests" | worker-4 | `reviewer` | `momus` | reviewer |
| "document API endpoints" | worker-5 | `researcher` | `librarian` | writing |
| "search for auth patterns" | worker-6 | `searcher` | `explore` | search |

### Role in Worker Prompt

Each worker receives their role, agent, and category in the prompt:

```markdown
You are worker-1 in team 'auth-analysis'.

**Role**: consultant (oracle agent)
**Category**: consultant
**Specialization**: Architecture consultation and debugging

Your task: Analyze the authentication flow in src/auth/

Focus on:
- Identifying potential security vulnerabilities
- Mapping the authentication flow
- Documenting findings

Write your results to: .omo/state/omo-team/auth-analysis/workers/worker-1/result.md
```

### Agent-Based Category Selection

When spawning workers, the category is selected based on the assigned agent:

| Agent | Default Category | Alternative Categories |
|-------|------------------|------------------------|
| `sisyphus` | orchestrator | - |
| `hephaestus` | executor | `deep`, `unspecified-high` |
| `oracle` | consultant | `deep` |
| `explore` | search | `quick` |
| `prometheus` | planner | `deep` |
| `metis` | analyst | `deep` |
| `momus` | reviewer | `deep` |
| `librarian` | research | `writing` |
| `multimodal-looker` | visual | `visual-engineering` |
| `atlas` | knowledge | `unspecified-high` |
| `sisyphus-junior` | orchestrator-junior | `quick` |

### Role-Based Skill Loading

Skills are loaded based on the agent configuration:

```typescript
// Agent → Skills mapping (from oh-my-opencode.json)
const AGENT_SKILLS = {
  "sisyphus": ["ralph", "autopilot", "ultrawork"],
  "hephaestus": [],
  "oracle": [],
  "explore": [],
  "prometheus": ["plan", "ralplan"],
  "metis": [],
  "momus": [],
  "librarian": [],
  "multimodal-looker": [],
  "atlas": [],
  "sisyphus-junior": []
};

// When spawning worker
task(
  category=get_category_for_agent(agent),
  load_skills=AGENT_SKILLS[agent] || [],
  run_in_background=true,
  prompt=build_worker_prompt(worker_id, role, agent, subtask)
)
```

---

## Preconditions
 
Before running `$omo-team`, confirm:
 
1. Running in OpenCode session (`OPENCODE=1` env var set)
2. Task description is clear and actionable
3. **If N not specified**: Analyze task complexity to estimate optimal worker count
4. If category not specified, use appropriate category based on task type
 
## Master Supervision Contract
 
The master agent supervises worker execution through compact state files, evidence-based contracts, and anomaly detection while minimizing token usage.
 
### Master Responsibilities
 
1. **Decompose and Validate**: Break task into subtasks, validate worker contracts, and spawn workers in parallel.
2. **Monitor Compact Status**: Poll worker status files and structured result headers first; only escalate to full outputs on anomaly.
3. **Aggregate and Flag**: Build roll-up ledger, verify evidence, flag anomalies, and synthesize verified summaries.
4. **Report Final Result**: Present consolidated findings to user with clear status indicators.
 
### Master Prohibitions (Token Safety & Supervision Integrity)
 
- **Do NOT re-execute worker tasks**: On failure, respawn with adjusted instructions or report partial results.
- **Do NOT read full worker transcripts in normal flow**: Use tiered reads (status → header → full only on anomaly).
- **Do NOT allow dynamic redistribution**: Failed work is not reassigned to surviving workers.
- **Do NOT permit cross-worker communication**: Master is sole coordinator; workers remain isolated.
- **Do NOT allow nested team invocation**: Workers must not spawn their own `/omo-team` instances.
- **Do NOT permit worker sub-worker spawning**: Prevents uncontrolled token growth.
 
### Token Budget Ratios (Supervision Overhead Target)
 
- Master supervision: ≤10% of total team token budget
- Worker execution: ≥80% 
- Aggregation/escalation: ≤10% in normal path
 
When budget risk appears, master must summarize from compact files and only open full results for critical/anomalous workers.
 
### Polling and Completion Handling
 
- Use `background_output(task_id, block=false)` for polling.
- Hard poll-cycle cap: default 20 cycles (or environment-configured equivalent).
- After cap, cancel remaining workers and force partial-report generation.
 
## Pre-context Intake Gate

Before spawning workers, require a grounded context snapshot:

1. Derive a team name from the request (sanitized, lowercase, max 30 chars)
2. Create team state directory: `.omo/state/omo-team/<team>/`
3. Write team manifest: `.omo/state/omo-team/<team>/manifest.json`
4. Decompose task into N subtasks (one per worker)
5. Write subtask files: `.omo/state/omo-team/<team>/tasks/task-<id>.json`

## Worker Spawning Protocol

### Step 1: Create Team State

```json
// .omo/state/omo-team/<team>/manifest.json
{
  "team_name": "<team>",
  "created_at": "<ISO timestamp>",
  "worker_count": N,
  "status": "active",
  "tasks": ["task-1", "task-2", ...]
}
```

### Step 2: Spawn Workers in Parallel

Use `task()` tool to spawn N background subagents:

```typescript
// Spawn worker 1
task(
  category="<category>",
  load_skills=["<skill1>", "<skill2>"],
  run_in_background=true,
  description="Worker 1: <subtask description>",
  prompt="You are worker-1 in team '<team>'. Your task: <subtask details>. Write your results to .omo/state/omo-team/<team>/workers/worker-1/result.md"
)

// Spawn worker 2 (parallel)
task(
  category="<category>",
  load_skills=["<skill1>", "<skill2>"],
  run_in_background=true,
  description="Worker 2: <subtask description>",
  prompt="You are worker-2 in team '<team>'. Your task: <subtask details>. Write your results to .omo/state/omo-team/<team>/workers/worker-2/result.md"
)
```

### Step 3: Track Task IDs

Collect all `task_id` values returned from `task()` calls. Store in:
```json
// .omo/state/omo-team/<team>/task-ids.json
{
  "worker-1": "bg_abc123",
  "worker-2": "bg_def456"
}
```

## Monitoring Protocol

### Poll Background Tasks

Use `background_output()` to check each worker's status:

```typescript
// Check worker 1
background_output(task_id="bg_abc123", block=false)

// Check worker 2
background_output(task_id="bg_def456", block=false)
```

### Status Tracking

Update team status as workers complete:

```json
// .omo/state/omo-team/<team>/status.json
{
  "pending": 0,
  "in_progress": 2,
  "completed": 0,
  "failed": 0
}
```

## Result Aggregation
 
When all workers complete:
 
1. Read each worker's result file: `.omo/state/omo-team/<team>/workers/worker-N/result.md`
2. Aggregate into final report: `.omo/state/omo-team/<team>/final-report.md`
3. Present synthesized results to user
4. Clean up team state (optional)
 
## Status Ledger and 3-Tier Master Read Strategy
 
The master uses compact state files for efficient supervision and only escalates to full outputs when needed.
 
### Compact State Files
 
- **Ledger (master-owned)**: `.omo/state/omo-team/<team>/ledger.json` - roll-up index of all workers.
- **Worker Status (worker-owned)**: `.omo/state/omo-team/<team>/workers/<worker-id>/status.json` - compact status.
- **Result Header (worker-provided)**: First 15 lines of `.omo/state/omo-team/<team>/workers/<worker-id>/result.md` - structured header.
 
### Read Tiers (Normal Supervision Path)
 
The master follows this order to minimize token usage:
 
1. **Tier 1**: Read `ledger.json` + each worker's `status.json` only.
   - Use for: progress tracking, completion detection, basic health.
2. **Tier 2**: Read first 15 lines of each worker's `result.md` (structured header) only.
   - Use for: verifying task scope, evidence paths, files modified, summary validation.
3. **Tier 3**: Read full `result.md` or `background_output(full_session=true, include_tool_results=false)` only.
   - Use for: anomaly investigation only (missing, malformed, failed, conflicting, or suspicious evidence).
 
### Escalation Triggers (Move to Next Tier)
 
Escalate from Tier 1 → Tier 2 when:
  - Worker status is `in_progress` or `completed` but header not yet validated.
  - Need to verify evidence paths or files modified claims.
 
Escalate from Tier 2 → Tier 3 when:
  - Missing `result.md` file.
  - Empty or whitespace-only `result.md`.
  - Malformed or missing structured header.
  - `Status: failed` or `blocked`.
  - Evidence path references non-existent files.
  - Overlapping `Files Modified` claims between workers.
  - Worker claims out-of-scope modifications (outside workspace).
  - Suspicious evidence (e.g., claims without verifiable anchors).
 
### Master Aggregation Order
 
1. Build ledger first from `status.json` and task IDs.
2. Validate headers second (Tier 2) for all workers.
3. Synthesize only verified/flagged compact summaries into final report.
4. Only open full outputs (Tier 3) for workers flagged as anomalies.
 
### Ledger JSON Schema
 
```json
{
  "worker_id": "worker-1",
  "task_id": "bg_abc123",
  "status": "pending|in_progress|completed|failed|partial|blocked|timeout|cancelled|unverified",
  "header_valid": true|false,
  "evidence_status": "valid|missing|suspicious|overlap|out_of_scope",
  "files_modified": ["file1.ts", "file2.ts"],
  "anomaly_flags": ["missing_result", "empty_header", "evidence_missing"],
  "last_seen": "<ISO timestamp>"
}
```
 
## Per-Role Evidence and Anti-Hallucination Rules

To prevent hallucinations and wrong-path execution, each worker role must include evidence anchors appropriate to their task. The master performs spot-checks to verify claims.

### Role-Specific Evidence Requirements

- **`executor`** (implementation/coding): 
  - Files modified (with paths)
  - Commands run (e.g., build, test commands)
  - Diagnostics/test outputs (e.g., `lsp_diagnostics`, test results)
  - Evidence paths to files that were changed or examined

- **`searcher`/`explore`** (codebase exploration):
  - File paths with line ranges searched
  - Search terms used
  - Repository/path context
  - Evidence of what was found (or not found)

- **`researcher`/`librarian`** (documentation lookup):
  - URLs/docs consulted
  - Concise applicability note (how the documentation applies to the task)
  - Version/branch information if relevant

- **`consultant`/`oracle`/`metis`** (analysis/consultation):
  - Assumptions made
  - Cited evidence (files, lines, external sources)
  - Confidence level (high/medium/low)
  - Risks or limitations identified

- **`reviewer`/`momus`** (quality assurance):
  - Pass/fail verdict
  - List of files checked
  - Criteria applied (e.g., lint rules, test coverage thresholds)
  - Evidence of violations (if any)

- **`visual`** (image/diagram analysis):
  - Image/screenshot paths
  - Exact observations (what was seen in the image)
  - Context of the visual (what part of the system it relates to)

### Master Spot-Check Matrix

The master performs these evidence spot-checks (typically via Tier 2 reads or targeted Tier 3 reads):

1. **Evidence path existence**: Verify that referenced files/paths exist in the workspace.
2. **Workspace boundary**: Ensure all file claims are within the project workspace (no external system references).
3. **Modified-file overlap**: Flag overlapping `Files Modified` claims between workers for potential conflicts.
4. **Line reference validation**: If line numbers/ranges are provided, verify they exist in the referenced file.
5. **Out-of-scope claim detection**: If a worker claims to modify a file outside their assigned subtask scope, flag for escalation.
6. **Evidence-anchor matching**: For analysis roles, verify that conclusions are supported by the cited evidence.

### Role Auto-Detection Priority (for ambiguous tasks)

When the automatic role assignment (based on task keywords) encounters ambiguity, use this priority order:

1. `test`, `spec`, `verify`, `qa` → `reviewer` (momus)
2. `document`, `readme`, `doc`, `write` → `researcher` (librarian)
3. `review`, `audit`, `check`, `security` → `consultant` (oracle)
4. `analyze`, `investigate`, `debug`, `why` → `consultant` (oracle)
5. `search`, `find`, `explore`, `locate` → `searcher` (explore)
6. `research`, `lookup`, `docs`, `api` → `researcher` (librarian)
7. `frontend`, `ui`, `component`, `style`, `css` → `visual` (multimodal-looker)
8. `image`, `diagram`, `screenshot`, `visual` → `visual` (multimodal-looker)
9. `plan`, `design`, `architect` → `planner` (prometheus)
10. `coordinate`, `orchestrate`, `manage` → `junior-orchestrator` (sisyphus-junior)
11. `implement`, `build`, `create`, `develop` → `executor` (hephaestus)
12. `api`, `endpoint`, `backend`, `server` → `executor` (hephaestus) with `deep` category
13. `database`, `schema`, `query`, `sql` → `executor` (hephaestus) with `deep` category
14. `deploy`, `ci`, `cd`, `pipeline` → `executor` (hephaestus) with `unspecified-high` category
15. Default → `executor` (hephaestus) with `unspecified-high` category

This priority ensures that verification and documentation tasks are correctly routed to the appropriate validation roles.

## Category Selection Guide

| Task Type | Recommended Category | Reasoning |
|-----------|---------------------|-----------|
| Complex implementation | `deep` | Thorough research + implementation |
| Quick fixes | `quick` | Fast, single-file changes |
| UI/Frontend work | `visual-engineering` | Domain-optimized for UI |
| Hard logic/algorithms | `ultrabrain` | High reasoning capability |
| Documentation | `writing` | Optimized for prose |
| General tasks | `unspecified-high` | Flexible, high effort |

## Skill Loading Guide

| Task Domain | Recommended Skills |
|-------------|-------------------|
| Git operations | `git-master` |
| Frontend/UI | `frontend-ui-ux` |
| Code review | `code-review` |
| Security | `security-review` |
| Build fixes | `build-fix` |
| Research | `deep-research` |

## Error Handling, Timeout, Anomaly, and Respawn Rules
 
### Worker Statuses
 
Workers and the master track these statuses:
 
- `pending`: Task not yet started
- `in_progress`: Worker is actively working
- `completed`: Worker finished successfully
- `failed`: Worker encountered an error and could not complete
- `partial`: Worker completed part of the task but not all (used for timeout or early exit)
- `blocked`: Worker is blocked on external dependency
- `timeout`: Worker exceeded allowed time without completion
- `cancelled`: Master cancelled the worker (e.g., after poll cap)
- `unverified`: Worker completed but results failed validation (missing evidence, etc.)
 
### Worker Failure Handling
 
When a worker fails:
 
1. **Preserve outputs**: The master retains any `result.md` and `status.json` the worker wrote.
2. **Log failure**: Append to `.omo/state/omo-team/<team>/errors.json` with worker ID, timestamp, and error details.
3. **Respawn-with-adjusted-instructions**: The master may respawn the same worker with clarified or narrowed instructions (checkpoint/resume is NOT used).
4. **Continue with remaining workers**: Other workers continue unaffected.
 
### Anomaly Handling
 
The master treats these situations as anomalies requiring Tier 3 reads or special handling:
 
- **Missing `result.md`**: No result file found after worker completion signal.
- **Empty `result.md`**: File exists but contains only whitespace or is empty.
- **Malformed header**: Missing or unparseable structured result header in first 15 lines.
- **Worker completed but no file artifact**: Worker claims completion but no evidence of file changes or outputs.
- **All workers failed**: Total failure scenario.
- **Overlapping file modifications**: Two or more workers claim to modify the same file.
- **Workspace-boundary violations**: Worker references files outside the project workspace.
- **Timeout race**: Worker completes exactly at timeout; master may have already initiated cancellation.
- **Nested team invocation attempt**: Worker attempts to spawn its own `/omo-team` (forbidden).
 
For each anomaly, the master:
 
1. Flags the worker in `ledger.json` with appropriate `anomaly_flags`.
2. Attempts to read full `result.md` or session transcript (Tier 3) for diagnosis.
3. If the anomaly cannot be resolved, marks the worker as `unverified` or `failed` and excludes from final synthesis unless partial results are salvageable.
4. Reports the anomaly in the final result with root-cause suggestion.
 
### Respawn Protocol
 
The **only** retry mechanism is respawn-with-adjusted-instructions:
 
- **No checkpoint/resume**: Workers do not save intermediate checkpoints.
- **No master re-execution**: The master never redoes the worker's assigned task itself.
- **Adjusted instructions**: On respawn, the master provides clarified scope, narrowed focus, or corrected assumptions based on the failure/anomaly.
 
### Timeout Handling
 
1. Set reasonable timeout (default: 5 minutes per worker).
2. Use `background_cancel()` for stuck workers after hard poll cap.
3. Report partial results with timeout notice (if any output exists).
 
## Cleanup
 
After completion:
 
```bash
# Optional: Remove team state
rm -rf .omo/state/omo-team/<team>/
 
# Or keep for audit trail
```

## Example Workflow
 
```
User: /omo-team 3 "analyze the auth module for security issues"
 
Agent:
1. Creates team "auth-security-analysis"
2. Decomposes into 3 subtasks and writes master supervision contracts.
3. Spawns 3 background tasks with category="deep", load_skills=["security-review"].
4. Polls compact status (status.json) and reads result headers (first 15 lines) for each worker.
5. Spot-checks evidence paths and flags any anomalies.
6. Aggregates verified summaries into final report.
7. Presents findings to user.
```

## Anti-Patterns (DO NOT)
 
- **DO NOT** use `omx team` CLI command - this spawns OMX workers, not OpenCode subagents
- **DO NOT** spawn workers sequentially - always use `run_in_background=true` for parallelism
- **DO NOT** forget to track task_ids - you need them to poll completion
- **DO NOT** skip result aggregation - users expect synthesized output
- **DO NOT** use outside OpenCode sessions - this skill requires OpenCode infrastructure
- **DO NOT** allow nested `/omo-team` invocation inside workers
- **DO NOT** allow workers to spawn sub-workers
- **DO NOT** permit cross-worker communication (mailboxes, shared state writes)
- **DO NOT** make the master read full worker transcripts in normal flow (use tiered reads)
- **DO NOT** allow dynamic redistribution of failed work to surviving workers
- **DO NOT** use checkpoint/resume protocols (use respawn-with-adjusted-instructions only)

## Integration with Other Skills

This skill can be combined with:

- `/plan` - Plan first, then execute with team
- `/ralph` - Use Ralph for verification after team completes
- `/ultraqa` - QA cycle after team execution

## When to Use Auto vs Explicit Worker Count

### Use Auto-Estimation (Recommended)

```
/omo-team "analyze the codebase for performance bottlenecks"
```

**When:**
- Task scope is unclear or variable
- You want optimal resource allocation
- Task can be naturally decomposed
- You're unsure about complexity

**Benefits:**
- Adapts to actual task complexity
- Avoids over/under-provisioning
- Reduces coordination overhead for simple tasks

### Use Explicit Worker Count

```
/omo-team 2 "implement feature X"
```

**When:**
- You know exact decomposition needed
- Task has clear, fixed subtasks
- You want precise control over parallelism
- Testing specific worker configurations

**Benefits:**
- Full control over team size
- Predictable resource usage
- Known decomposition strategy

### Decision Matrix

| Scenario | Recommendation | Example |
|----------|----------------|---------|
| Quick exploration | Auto (likely 1-2) | `/omo-team "check for typos"` |
| Feature implementation | Auto (likely 2-3) | `/omo-team "add user authentication"` |
| Large refactoring | Auto (likely 3-4) | `/omo-team "migrate to TypeScript strict mode"` |
| Known decomposition | Explicit | `/omo-team 3 "analyze API, DB, and frontend"` |
| Testing/debugging | Explicit (2) | `/omo-team 2 "find and fix the bug"` |
| Documentation | Auto (likely 2) | `/omo-team "document all API endpoints"` |

## Validation and Dry-Run QA Guidance

To ensure the supervision protocol works correctly without human intervention, every implementation task must include agent-executable QA scenarios. Evidence is saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

### Supervision QA Checklist

The master agent's supervision protocol must be validated through the following agent-executable scenarios:

#### Normal Dry-Run Scenario
- **Goal**: Verify that the master can complete a full supervision cycle without reading full worker transcripts.
- **Steps**:
  1. Spawn 2 trivial workers with safe tasks (e.g., "write 'hello' to result.md").
  2. Master follows the protocol: creates contracts, spawns workers, polls compact status, reads headers, spot-checks evidence, aggregates verified summaries.
  3. Master does NOT read full `result.md` files or session transcripts in the normal path.
  4. Master produces a final report based on verified compact summaries.
- **Evidence**: Check that `.sisyphus/evidence/task-7-dry-run-normal.txt` exists and contains success indicators.

#### Failure-Injection Scenario
- **Goal**: Verify that the master handles worker failures correctly without re-executing worker tasks.
- **Steps**:
  1. Spawn 1 worker that deliberately writes an empty `result.md` or malformed header.
  2. Master detects the anomaly via Tier 2 reads, escalates to Tier 3, flags the worker as `unverified` or `failed`.
  3. Master reports partial results (if any) and does NOT redo the worker's task.
  4. Other workers (if any) continue unaffected.
- **Evidence**: Check that `.sisyphus/evidence/task-7-failure-injection.txt` exists and contains failure handling evidence.

#### Token-Audit Scenario
- **Goal**: Verify that master supervision stays within token budget targets.
- **Steps**:
  1. Run a 3-worker team with typical tasks.
  2. Count the master's tool calls in the normal supervision path (polling + status reads + header reads + ledger updates).
  3. Verify that supervision tool-call count is below an explicit threshold (e.g., `< 30` calls in the normal path).
  4. Verify that the master reads compact files first and only opens full outputs for anomalous workers.
- **Evidence**: Check that `.sisyphus/evidence/task-7-token-audit.txt` exists and contains tool-call counts and validation.

### QA Policy
 
Every task in this plan **MUST** include agent-executed QA scenarios. Evidence is saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.
 
- **Skill markdown validation**: Bash/read/grep checks on `.md` files.
- **State contract validation**: Bash creates temporary `.omo/state/omo-team/supervision-dry-run/` fixtures and verifies schema-like fields with shell or `jq` if available.
- **OpenCode dry-run**: If OPENCODE and task tools are available, spawn trivial background workers; otherwise record an explicit environment-skip artifact and run file-contract simulation.
 
### Acceptance Criteria for QA
 
- [ ] `/omo-team/SKILL.md` contains a `Supervision QA Checklist` section.
- [ ] Checklist includes normal dry-run, failure-injection, and token-audit scenarios.
- [ ] Checklist includes concrete state-file checks for `ledger.json` and worker result headers.
- [ ] Checklist forbids manual-only acceptance (no items requiring `manually inspect` or `user confirms`).
 
## Version
 
- Version: 1.1.0
- Compatible with: OpenCode 1.x, oh-my-opencode 3.x+
- Author: OpenCode Community
- **Changes in 1.1.0**: Added automatic worker count estimation based on task complexity

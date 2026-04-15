# OMO-Team: OpenCode-Native Team Orchestration

**Version**: 1.1.0  
**Compatible With**: OpenCode 1.x, oh-my-opencode 3.x+  
**Author**: OpenCode Community  
**License**: Open Source  
**GitHub**: https://github.com/Trochez/omo-team

---

## 📋 Table of Contents

1. [What is OMO-Team?](#what-is-omo-team)
2. [Key Features](#key-features)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Usage Examples](#usage-examples)
6. [Arguments Reference](#arguments-reference)
7. [Automatic Features](#automatic-features)
8. [How It Works](#how-it-works)
9. [Comparison with OMX-Team](#comparison-with-omx-team)
10. [Troubleshooting](#troubleshooting)
11. [Advanced Topics](#advanced-topics)
12. [API Reference](#api-reference)

---

## What is OMO-Team?

**OMO-Team** is an OpenCode-native parallel execution skill that spawns **real OpenCode subagent sessions** (not OMX/Codex CLI workers). It coordinates multiple workers through the `task()` tool with background execution, enabling true parallel processing within the OpenCode ecosystem.

### The Problem It Solves

When using the traditional `team` skill in OpenCode, it spawns **OMX sessions** (Codex/Claude CLI workers with GPT models) instead of **OpenCode sessions**. This creates an architecture mismatch:

- **OMX Team**: Uses tmux + CLI workers → External processes
- **OMO Team**: Uses OpenCode's native `task()` tool → Native subagents

### Key Difference

| Aspect | OMX Team | OMO Team |
|--------|----------|----------|
| **Worker Type** | Codex/Claude CLI sessions | OpenCode subagent sessions |
| **Spawning** | `omx team` CLI + tmux | `task()` tool with background |
| **Session Type** | OMX (GPT models) | OpenCode (same as leader) |
| **Requires tmux** | Yes | No |
| **Works in OpenCode** | ❌ No | ✅ Yes |

---

## Key Features

### ✨ Automatic Worker Count Estimation

**NEW in v1.1.0**: OMO-Team automatically estimates the optimal number of workers based on task complexity.

```
Complexity Score (0-100) → Worker Count

Score 0-19   → 1 worker  (trivial)
Score 20-39  → 2 workers (simple)
Score 40-59  → 3 workers (moderate)
Score 60-79  → 4 workers (complex)
Score 80-100 → 5 workers (maximum)
```

**Example**:
```bash
# Auto-estimation (recommended)
/omo-team "analyze the codebase for performance bottlenecks"

# Explicit count (when you know the decomposition)
/omo-team 3 "implement feature X with tests and docs"
```

### 🎯 Automatic Role Assignment

Each worker is automatically assigned a **role** based on their subtask. Roles map to **oh-my-opencode agents** and **categories**:

| Task Keywords | Role | Agent | Category |
|---------------|------|-------|----------|
| `test`, `spec`, `verify` | `reviewer` | `momus` | reviewer |
| `document`, `readme`, `doc` | `researcher` | `librarian` | writing |
| `review`, `audit`, `security` | `consultant` | `oracle` | consultant |
| `analyze`, `investigate`, `debug` | `consultant` | `oracle` | consultant |
| `search`, `find`, `explore` | `searcher` | `explore` | search |
| `frontend`, `ui`, `css` | `visual` | `multimodal-looker` | visual-engineering |
| `implement`, `build`, `create` | `executor` | `hephaestus` | deep |

### 🚀 True Parallelism

Workers execute **simultaneously** using `run_in_background=true`:

```typescript
// All workers spawn at once
task(category="deep", run_in_background=true, ...)
task(category="visual-engineering", run_in_background=true, ...)
task(category="reviewer", run_in_background=true, ...)
```

### 📊 State Management

All team state is stored in `.omo/state/omo-team/<team>/`:

```
.omo/state/omo-team/auth-analysis/
├── manifest.json          # Team metadata
├── task-ids.json          # Background task IDs
├── status.json            # Progress tracking
└── workers/
    ├── worker-1/
    │   └── result.md      # Worker 1 output
    ├── worker-2/
    │   └── result.md      # Worker 2 output
    └── worker-3/
        └── result.md      # Worker 3 output
```

---

## Installation

### Prerequisites

- **OpenCode 1.x** or higher
- **oh-my-opencode 3.x+** plugin
- OpenCode session (`OPENCODE=1` environment variable)

### Step 1: Verify OpenCode Session

```bash
# Check if you're in an OpenCode session
echo $OPENCODE
# Should output: 1
```

### Step 2: Install Skills

OMO-Team requires **two skills**:

1. **`/omo-team`** (this skill) - The leader that spawns workers
2. **[`/omo-worker`](https://github.com/Trochez/omo-worker)** - Worker protocol (required dependency)

**Option A: Clone Both Repositories (Recommended)**

```bash
# Create skill directories
mkdir -p ~/.agents/skills/omo-team
mkdir -p ~/.agents/skills/omo-worker

# Clone and install omo-team
git clone https://github.com/Trochez/omo-team.git /tmp/omo-team
cp /tmp/omo-team/SKILL.md ~/.agents/skills/omo-team/
cp /tmp/omo-team/README.md ~/.agents/skills/omo-team/

# Clone and install omo-worker (REQUIRED)
git clone https://github.com/Trochez/omo-worker.git /tmp/omo-worker
cp /tmp/omo-worker/SKILL.md ~/.agents/skills/omo-worker/
cp /tmp/omo-worker/README.md ~/.agents/skills/omo-worker/
```

**Option B: Manual Installation**

```bash
# Create skill directories
mkdir -p ~/.agents/skills/omo-team
mkdir -p ~/.agents/skills/omo-worker

# Copy skill files (if you have them locally)
cp SKILL.md ~/.agents/skills/omo-team/
cp README.md ~/.agents/skills/omo-team/
cp omo-worker/SKILL.md ~/.agents/skills/omo-worker/
```

### Step 3: Verify Installation

```bash
# List installed skills
ls -la ~/.agents/skills/omo-team/
ls -la ~/.agents/skills/omo-worker/

# Should see:
# omo-team/SKILL.md
# omo-team/README.md
# omo-worker/SKILL.md
# omo-worker/README.md
# README.md
```

### Step 4: Test Installation

In an OpenCode session:

```
/omo-team 2 "list files in current directory"
```

Expected: 2 workers spawn, execute in parallel, and report results.

---

## Quick Start

### Basic Usage

```bash
# Auto-estimated workers (recommended)
/omo-team "analyze the authentication module"

# Explicit worker count
/omo-team 3 "implement REST API endpoints"

# With category
/omo-team 2 "review code for security issues" --category=deep

# With skills
/omo-team 2 "implement feature X" --category=deep --skills=typescript-programmer
```

### Example: Analyze Auth Module

```
User: /omo-team "analyze the authentication module for security vulnerabilities"

Agent:
1. Estimates complexity → 3 workers
2. Decomposes task:
   - Worker 1: Analyze authentication flow
   - Worker 2: Check token handling
   - Worker 3: Review password storage
3. Spawns 3 workers in parallel
4. Waits for completion
5. Aggregates results into security report
```

---

## Usage Examples

### Basic Examples

#### 1. Simple Analysis (Auto-Estimated)

```bash
/omo-team "check for typos in README"
```

**What happens**:
- Complexity: Low (0-19) → 1 worker
- Role: `searcher` (explore agent)
- Category: `quick`

#### 2. Module Analysis (2 Workers)

```bash
/omo-team 2 "analyze the database module"
```

**What happens**:
- Worker 1: Analyzes schema
- Worker 2: Analyzes queries
- Both run in parallel

#### 3. Feature Implementation (3 Workers)

```bash
/omo-team 3 "implement user authentication"
```

**What happens**:
- Worker 1: Implement login API (executor)
- Worker 2: Create auth UI (visual)
- Worker 3: Write tests (reviewer)

### Intermediate Examples

#### 4. With Category

```bash
/omo-team 2 "implement REST API endpoints" --category=deep
```

**What happens**:
- Both workers use `deep` category
- Thorough research + implementation
- Higher reasoning capability

#### 5. With Skills

```bash
/omo-team 2 "review code for security issues" --category=deep --skills=security-review
```

**What happens**:
- Workers load `security-review` skill
- Security-focused analysis
- Comprehensive vulnerability check

#### 6. Complex Task (Auto-Estimated)

```bash
/omo-team "analyze and document the entire codebase architecture"
```

**What happens**:
- Complexity: High (60-79) → 4 workers
- Worker 1: Analyze core modules (consultant)
- Worker 2: Analyze utilities (searcher)
- Worker 3: Analyze APIs (researcher)
- Worker 4: Generate documentation (researcher)

### Advanced Examples

#### 7. Full-Stack Implementation

```bash
/omo-team 5 "build complete user management system with auth, API, database, frontend, and tests"
```

**What happens**:
- Worker 1: Auth implementation (executor, deep)
- Worker 2: API endpoints (executor, deep)
- Worker 3: Database schema (executor, deep)
- Worker 4: Frontend UI (visual, visual-engineering)
- Worker 5: Integration tests (reviewer, reviewer)

#### 8. Multi-Perspective Review

```bash
/omo-team 3 "comprehensive code review of payment module" --category=deep --skills=security-review,code-review
```

**What happens**:
- Worker 1: Security review (consultant)
- Worker 2: Code quality review (reviewer)
- Worker 3: Performance review (analyst)

#### 9. Research and Documentation

```bash
/omo-team 4 "research best practices for microservices and create implementation guide" --category=deep --skills=deep-research,writing
```

**What happens**:
- Worker 1: Research patterns (researcher)
- Worker 2: Research tools (researcher)
- Worker 3: Research deployment (researcher)
- Worker 4: Write guide (researcher, writing)

---

## Arguments Reference

### Required Arguments

| Argument | Type | Description | Example |
|----------|------|-------------|---------|
| `"<task description>"` | string | Main task to decompose and distribute | `"analyze auth module"` |

### Optional Arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `N` | number | Auto | Number of workers (omit for auto-estimation) |
| `--category=<cat>` | string | Auto | Category for worker subagents |
| `--skills=<list>` | string | `[]` | Comma-separated skills to load |
| `--timeout=<ms>` | number | `300000` | Maximum wait time per worker (5 min) |
| `--auto` | flag | False | Force auto-estimation (ignores N if provided) |

### Category Options

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

### Skill Options

| Skill | Use When |
|-------|----------|
| `git-master` | Git operations, commits, rebasing |
| `frontend-ui-ux` | Frontend/UI work |
| `code-review` | Code quality review |
| `security-review` | Security audit |
| `build-fix` | Build/TypeScript errors |
| `deep-research` | Comprehensive research |
| `typescript-programmer` | TypeScript implementation |

---

## Automatic Features

### 1. Automatic Worker Count Estimation

**How it works**:

```python
def estimate_workers(task_description):
    complexity_score = 0
    
    # Scope Breadth (30%)
    if "entire" in task or "all" in task:
        complexity_score += 30
    
    # Task Diversity (25%)
    verb_count = count_verbs(task)
    complexity_score += min(verb_count * 8, 25)
    
    # Domain Complexity (20%)
    if any(kw in task for kw in ["architecture", "algorithm", "security"]):
        complexity_score += 20
    
    # File Count (15%)
    file_mentions = count_file_mentions(task)
    complexity_score += min(file_mentions * 5, 15)
    
    # Integration Points (10%)
    if any(kw in task for kw in ["API", "database", "external"]):
        complexity_score += 10
    
    # Map to worker count
    if complexity_score < 20: return 1
    elif complexity_score < 40: return 2
    elif complexity_score < 60: return 3
    elif complexity_score < 80: return 4
    else: return 5
```

**Examples**:

| Task | Estimated | Reasoning |
|------|-----------|-----------|
| "fix typo in README" | 1 | Single file, trivial scope |
| "add unit tests for auth module" | 2 | Single module, focused task |
| "analyze and document API endpoints" | 3 | Two phases, moderate scope |
| "refactor entire codebase for TypeScript strict mode" | 4 | High scope, multiple modules |
| "implement full microservices architecture" | 5 | Maximum complexity |

### 2. Automatic Role Assignment

**How it works**:

```python
def assign_role(subtask_description):
    subtask_lower = subtask_description.lower()
    
    if any(kw in subtask_lower for kw in ["test", "spec", "verify"]):
        return {"role": "reviewer", "agent": "momus", "category": "reviewer"}
    elif any(kw in subtask_lower for kw in ["document", "readme"]):
        return {"role": "researcher", "agent": "librarian", "category": "writing"}
    elif any(kw in subtask_lower for kw in ["review", "audit", "security"]):
        return {"role": "consultant", "agent": "oracle", "category": "consultant"}
    elif any(kw in subtask_lower for kw in ["analyze", "investigate"]):
        return {"role": "consultant", "agent": "oracle", "category": "consultant"}
    elif any(kw in subtask_lower for kw in ["search", "find", "explore"]):
        return {"role": "searcher", "agent": "explore", "category": "search"}
    elif any(kw in subtask_lower for kw in ["frontend", "ui", "css"]):
        return {"role": "visual", "agent": "multimodal-looker", "category": "visual-engineering"}
    else:
        return {"role": "executor", "agent": "hephaestus", "category": "deep"}
```

**Example Decomposition**:

```
Task: "implement user authentication with tests and docs"

Auto-Decomposition:
├── worker-1: "implement login API"
│   → role: executor, agent: hephaestus, category: deep
├── worker-2: "create auth UI components"
│   → role: visual, agent: multimodal-looker, category: visual-engineering
├── worker-3: "write unit tests"
│   → role: reviewer, agent: momus, category: reviewer
└── worker-4: "document API endpoints"
    → role: researcher, agent: librarian, category: writing
```

---

## How It Works

### Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. PRE-CONTEXT INTAKE                                           │
│ • Derive team name from request                                 │
│ • Create .omo/state/omo-team/<team>/manifest.json               │
│ • Decompose task into N subtasks                                │
│ • Assign roles based on subtask keywords                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. SPAWN WORKERS (Parallel)                                     │
│ • task(category, load_skills, run_in_background=true)           │
│ • Collect task_ids for each worker                              │
│ • Store in .omo/state/omo-team/<team>/task-ids.json             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. MONITOR PROGRESS                                             │
│ • Poll background_output(task_id, block=false)                  │
│ • Update .omo/state/omo-team/<team>/status.json                 │
│ • Wait for all workers to complete                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. AGGREGATE RESULTS                                            │
│ • Read .omo/state/omo-team/<team>/workers/worker-N/result.md    │
│ • Synthesize into final report                                  │
│ • Present to user                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Worker Lifecycle

1. **Spawn**: Leader calls `task()` with `run_in_background=true`
2. **Execute**: Worker performs assigned subtask
3. **Report**: Worker writes results to `result.md`
4. **Complete**: Leader detects completion via `background_output()`
5. **Aggregate**: Leader synthesizes all worker results

---

## Comparison with OMX-Team

| Feature | OMO-Team | OMX-Team |
|---------|----------|----------|
| **Worker Type** | OpenCode subagent sessions | Codex/Claude CLI sessions |
| **Spawning Method** | `task()` tool | `omx team` CLI + tmux |
| **Session Type** | OpenCode (same as leader) | OMX (GPT models) |
| **State Location** | `.omo/state/omo-team/` | `.omx/state/team/` |
| **Coordination** | Background task polling | tmux panes + mailbox |
| **Requires tmux** | No | Yes |
| **Works in OpenCode** | ✅ Yes | ❌ No |
| **Auto Worker Count** | ✅ Yes (v1.1.0) | ❌ No |
| **Auto Role Assignment** | ✅ Yes | ❌ No |

---

## Troubleshooting

### Workers Not Spawning

**Symptoms**: No workers start, timeout errors

**Solutions**:
1. Verify `OPENCODE=1` environment variable
2. Check `task()` tool is available
3. Ensure `.omo/state/` directory is writable

```bash
# Check environment
echo $OPENCODE

# Check permissions
ls -la .omo/state/
```

### Results Not Written

**Symptoms**: Workers complete but no results

**Solutions**:
1. Check worker prompts include output path
2. Verify `.omo/state/omo-team/<team>/workers/` exists
3. Review `background_output()` for errors

```bash
# Check worker output directory
ls -la .omo/state/omo-team/*/workers/
```

### Timeout Issues

**Symptoms**: Workers timeout before completion

**Solutions**:
1. Increase `--timeout` argument
2. Check for blocking operations in workers
3. Review worker logs for stuck processes
4. **Configure provider-level timeouts** (see below)

```bash
# Increase timeout to 10 minutes
/omo-team 3 "complex task" --timeout=600000
```

### Provider-Level Timeout Configuration

**CRITICAL**: To prevent 30-minute freezes when models hit rate limits, configure provider-level timeouts in `oh-my-opencode.json`:

```json
{
  "provider": {
    "nvidia": {
      "timeout": 60000
    },
    "openai": {
      "timeout": 60000
    },
    "google": {
      "timeout": 60000
    }
  },
  "background_task": {
    "staleTimeoutMs": 60000
  }
}
```

**Why this matters:**
- Default background task timeout is 30 minutes (1,800,000ms)
- Provider timeout forces fallback after 60 seconds
- Background task timeout overrides the hardcoded default
- Without this, delegated agents can freeze for 30 minutes on rate limits

**Known Issue (GitHub #2203):** Background tasks may ignore `fallback_models` configuration. Use provider-level timeouts as a workaround.

### Skill Loading Failures

**Symptoms**: `Skills not found: omo-worker`

**Solutions**:
1. Install [`/omo-worker`](https://github.com/Trochez/omo-worker) skill (see Installation section)
2. Verify skill location: `ls ~/.agents/skills/omo-worker/SKILL.md`
3. Wait for skill registration (async)
4. Retry after a few seconds

---

## Advanced Topics

### Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `/plan` | Plan first, then execute with omo-team |
| `/ralph` | Use Ralph for verification after team completes |
| `/ultraqa` | QA cycle after team execution |
| [`/omo-worker`](https://github.com/Trochez/omo-worker) | **Required** - Worker protocol loaded in each spawned worker |

### Required Dependency: omo-worker

**[`/omo-worker`](https://github.com/Trochez/omo-worker) is required** for `/omo-team` to function. Each spawned worker loads this skill to know:
- How to parse task context
- Where to write results
- How to report completion
- How to handle errors

**Install omo-worker:**
```bash
git clone https://github.com/Trochez/omo-worker.git /tmp/omo-worker
mkdir -p ~/.agents/skills/omo-worker
cp /tmp/omo-worker/SKILL.md ~/.agents/skills/omo-worker/
cp /tmp/omo-worker/README.md ~/.agents/skills/omo-worker/
```

**GitHub Repo**: https://github.com/Trochez/omo-worker

### Custom Decomposition Strategies

You can influence how tasks are decomposed:

```bash
# Hint at decomposition
/omo-team 3 "analyze API, database, and frontend separately"

# Workers will be assigned:
# - worker-1: API analysis
# - worker-2: Database analysis
# - worker-3: Frontend analysis
```

### Manual Role Override

If auto-assignment isn't optimal, use explicit decomposition:

```bash
# Instead of auto-assignment
/omo-team 3 "implement feature X"

# Use explicit roles in task description
/omo-team 3 "worker-1: implement API (executor), worker-2: create UI (visual), worker-3: write tests (reviewer)"
```

---

## API Reference

### Slash Command

```bash
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

### State Files

#### Team Manifest

**Location**: `.omo/state/omo-team/<team>/manifest.json`

```json
{
  "team_name": "auth-analysis",
  "created_at": "2026-04-08T00:00:00Z",
  "worker_count": 3,
  "status": "active",
  "tasks": ["task-1", "task-2", "task-3"],
  "description": "analyze authentication module"
}
```

#### Task IDs

**Location**: `.omo/state/omo-team/<team>/task-ids.json`

```json
{
  "worker-1": "bg_abc123",
  "worker-2": "bg_def456",
  "worker-3": "bg_ghi789"
}
```

#### Worker Result

**Location**: `.omo/state/omo-team/<team>/workers/<worker-id>/result.md`

```markdown
# Worker: worker-1

## Task: Analyze authentication flow

### Summary
Analyzed authentication flow in src/auth/

### Findings
- JWT token validation is secure
- Password hashing uses bcrypt
- Session management needs improvement

### Files Modified
- None (analysis only)

### Recommendations
- Add rate limiting to login endpoint
- Implement token refresh mechanism
```

#### Team Status

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

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│ OMO-TEAM QUICK REF                                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ USAGE:                                                       │
│ /omo-team [N] "task" [--category=CAT] [--skills=S1,S2]      │
│                                                              │
│ EXAMPLES:                                                    │
│ /omo-team "analyze auth module"           # Auto-estimated  │
│ /omo-team 3 "implement API"               # Explicit count  │
│ /omo-team 2 "review code" --category=deep # With category   │
│ /omo-team 2 "fix bug" --skills=git-master # With skills     │
│                                                              │
│ CATEGORIES:                                                  │
│ deep | quick | visual-engineering | ultrabrain              │
│ writing | unspecified-high | unspecified-low                │
│                                                              │
│ AUTO FEATURES:                                               │
│ • Worker count estimation (complexity-based)                │
│ • Role assignment (keyword-based)                           │
│ • Agent mapping (oh-my-opencode agents)                     │
│                                                              │
│ STATE LOCATION:                                              │
│ .omo/state/omo-team/<team>/                                 │
│                                                              │
│ REQUIREMENTS:                                                │
│ OPENCODE=1 (must be in OpenCode session)                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.1.0 | 2026-04-08 | Added automatic worker count estimation, automatic role assignment |
| 1.0.0 | 2026-04-07 | Initial release |

---

## Support

- **Documentation**: `DATASHEET.md` (comprehensive reference)
- **Skill Definition**: `SKILL.md` (implementation details)
- **Worker Protocol**: [`/omo-worker`](https://github.com/Trochez/omo-worker) (required dependency)

---

## License

Open Source - OpenCode Community

---

**Last Updated**: April 15, 2026

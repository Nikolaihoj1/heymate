
# LLM-Optimized Autonomous Web App Orchestrator

## Overview
A command-line driven, hierarchical coding system where a single main orchestrator manages planning, scheduling, and validation across a dynamic number of sub-agents. Designed exclusively for web applications, with standardized modules, contract-driven context, and deterministic quality gates. Built to run autonomously for hours on slower or smaller LLMs by minimizing token consumption, enforcing strict context boundaries, and replacing probabilistic reasoning with machine-readable state management.

## Core Principles
- Single main orchestrator handles all human input, planning, scheduling, and validation
- Dynamic sub-agent batching: 1 to N agents spawned based on context window, token budget, and dependency order
- All state files, prompts, logs, and feedback are strictly LLM-optimized
- Zero conversational filler, zero human-readable formatting, zero explanatory comments in agent-facing files
- Token budget is the primary constraint; every token must serve function
- Code quality is preserved through deterministic validation, not verbose instructions
- Checkpointed, resumable, headless runtime capable of multi-hour autonomous execution
- Web-app specialization with standardized module types and contract-driven context

## Architecture & Execution Model
Human input flows exclusively to the main orchestrator. The orchestrator performs a planning phase, generates a locked execution blueprint, and schedules sub-agents in dependency-safe batches. Sub-agents operate in isolated contexts, write code and state files, and report completion. The orchestrator validates output against contracts and linting rules, merges successful work, saves checkpoints, and schedules the next batch. This cycle repeats until all modules are complete, followed by cross-module integration and final build verification.

Sub-agents never communicate directly. All synchronization occurs through the main agent's state store, preventing race conditions, context bleed, and hallucination cascades. Execution can run sequentially or in parallel batches, strictly bounded by context limits and dependency order.

## Web App Module Standardization
Web applications follow predictable architectural patterns. The main agent splits projects into standardized module types, each with a pre-defined contract template and minimal context requirements.

| Module Type | Responsibility | Context Minimization Strategy |
|-------------|----------------|-------------------------------|
| shared-types | TypeScript/Pydantic interfaces, enums, utilities | Receives only type contracts, versioning rules, and export requirements |
| database | Schema, migrations, queries, seeds | Receives schema contract, relationship graph, and indexing rules |
| auth | Login, sessions, tokens, permissions | Receives auth contract, token format, and permission matrix |
| backend-api | Routes, middleware, controllers, services | Receives route contracts, request/response schemas, and auth rules |
| frontend | UI components, routing, state management | Receives component contracts, API shapes, and design token references |
| infra-deploy | Docker, CI/CD, environment variables, hosting | Receives deployment contract, environment schema, and health check rules |

Sub-agents never receive full codebases. They only receive their module contract, consumed/produced interfaces, integration rules, context budget, and turn limits.

## LLM-Centric State & File Optimization
All machine-facing files are optimized for token efficiency and deterministic parsing. Human readability is explicitly deprioritized in favor of LLM performance.

### File Format Rules
- Use strict YAML or JSON for all machine state
- Flatten nested structures where possible
- Use standardized abbreviations: `st` (status), `ctx` (context), `tok` (tokens), `err` (errors), `dep` (dependencies)
- Remove all comments, examples, and explanatory text from agent-facing files
- Atomic updates only: change what changed, never rewrite full files
- File naming: lowercase, hyphenated, no spaces, versioned if needed
- Extension: `.yaml` or `.json` only

### Example: LLM-Optimized State File
```yaml
id: auth
st: running
ctx: 3200/5000
tok: 3180
dep: [shared-types]
err: []
rev: 3
last_upd: 2024-05-20T14:30:00
```

### Example: Human-Readable (AVOID)
```yaml
# Authentication module state
# This tracks progress for the auth service
status: currently running
context_usage: 3200 out of 5000 tokens
dependencies: shared-types module
errors: none so far
revision: 3
last_updated: 2024-05-20T14:30:00
```

### File Structure
```
/project
  /orchestrator
    plan.yaml
    state.yaml
    checkpoints/
      2024-05-20T14-30-00/
        plan.yaml
        state.yaml
        modules/
        contracts/
  /modules
    /auth
      contract.yaml
      state.yaml
      code/
  /contracts
    shared-types.yaml
    auth.yaml
    backend-api.yaml
```

## Planning Phase & Blueprint
Planning produces a machine-readable execution blueprint. No coding begins until this phase is complete and locked.

### plan.yaml Structure
```yaml
project:
  name: "web-app-demo"
  stack: "nextjs-fastapi-postgres"
  max_tokens_per_session: 150000
  slow_llm_mode: true

modules:
  - id: shared-types
    type: shared-types
    context_budget: 4000
    max_turns: 8
    dependencies: []
    contract: /contracts/shared-types.yaml
    success_criteria:
      - passes typecheck
      - no circular imports
      - exports match contract

  - id: database
    type: database
    context_budget: 6000
    max_turns: 10
    dependencies: [shared-types]
    contract: /contracts/database.yaml
    success_criteria:
      - migrations run clean
      - schema matches contract
      - indexes defined

  - id: auth
    type: auth
    context_budget: 5000
    max_turns: 9
    dependencies: [shared-types]
    contract: /contracts/auth.yaml
    success_criteria:
      - token generation passes unit tests
      - permission matrix implemented
      - no hardcoded secrets

  - id: backend-api
    type: backend-api
    context_budget: 8000
    max_turns: 12
    dependencies: [shared-types, database, auth]
    contract: /contracts/backend-api.yaml
    success_criteria:
      - passes lint and typecheck
      - route handlers match contract
      - auth middleware integrated

  - id: frontend
    type: frontend
    context_budget: 9000
    max_turns: 14
    dependencies: [shared-types, backend-api]
    contract: /contracts/frontend.yaml
    success_criteria:
      - builds without errors
      - components match contract
      - API calls use shared types

dependency_order: [shared-types, database, auth, backend-api, frontend]

context_strategy:
  batching: dependency_safe_parallel
  max_parallel: 5
  compaction_threshold: 0.75
  diff_only: true
  token_tracker: tiktoken

validation_pipeline:
  - lint
  - typecheck
  - contract_verify
  - test_run
  - import_graph_check

recovery:
  max_retries_per_module: 3
  on_failure: compact_context, split_module, retry
  halt_conditions:
    - contract_mismatch
    - typecheck_failure_after_3_retries
    - context_budget_exceeded
```

### Planning Rules
1. Contracts define exact inputs, outputs, interfaces, and dependencies
2. Context budgets are hard limits enforced by the orchestrator
3. Dependency order is resolved via topological sort to ensure safe batching
4. Blueprint is locked after planning. Only iterative refinement within budgets is allowed
5. Batching logic groups modules into context-safe windows. Independent modules run in parallel. Dependent modules run sequentially

## CLI Interface & Commands
A command-line interface serves as the sole human touchpoint. It manages planning, execution, monitoring, recovery, and configuration while delegating all code generation to dynamically spawned sub-agents.

### Command Reference
```
web-orchestrator [command] [options]

Commands:
  init          Initialize project structure, config, and baseline contracts
  plan          Execute planning phase, generate locked blueprint, validate dependencies
  run           Start autonomous execution (foreground or background)
  status        Display current state, progress, context usage, active/queued modules
  resume        Resume from last checkpoint
  stop          Gracefully halt execution, save checkpoint, terminate sub-agents
  logs          Stream execution logs in real-time
  validate      Run validation pipeline on specified or all completed modules
  config        View or update CLI and LLM configuration

Options:
  --config PATH       Path to configuration file (default: ./config.yaml)
  --output FORMAT     Console output format: text, json, compact (default: text)
  --dry-run           Validate plan and configuration without execution
  --max-tokens N      Override global token limit for the session
  --batch-size N      Override maximum parallel sub-agents
  --verbose           Enable debug-level logging and raw state dumps
  --daemon            Run in background mode (requires --run or --resume)
  --pid-file PATH     Custom PID file path for daemon management
  --timeout SECONDS   Global execution timeout before forced stop
```

### Execution Modes & Lifecycle
- **Foreground Mode**: Runs synchronously until completion, failure, or user interrupt. Prints structured progress updates to stdout. Ideal for debugging or short sessions.
- **Daemon Mode**: Runs asynchronously in background. Writes PID file, manages process lifecycle. Logs to `./orchestrator/logs/` with rotation. Required for multi-hour autonomous sessions.
- **State-Driven Flow**: `init` creates structure and contracts. `plan` generates locked blueprint. `run`/`resume` starts orchestrator loop. `stop` triggers graceful teardown. `status`/`logs` provide monitoring.

### Signal Handling & Graceful Shutdown
- `SIGINT`/`SIGTERM`: trigger graceful stop
- `SIGHUP`: reload configuration without stopping execution
- `SIGUSR1`: dump current state for debugging
- `SIGUSR2`: force checkpoint save
- Shutdown sequence: set status to `stopping`, terminate sub-agents, wait for turn completion or hard-kill, save final checkpoint, write shutdown log, exit cleanly.

## Context Management & Token Optimization
Context compression operates safely by preserving contracts, interfaces, and type schemas while stripping all non-essential text.

### Pre-Execution Optimization
- Strip comments, consolidate imports, remove unused types
- Flatten contract definitions to essential fields
- Remove example code, tests, and documentation from state files
- Compress dependency graph to adjacency list format

### Mid-Execution Compression
- Auto-summarize `state.yaml` every 3 turns
- Replace verbose progress logs with token counts, status, and error codes
- Keep only current interface stubs, never full module history
- Truncate from bottom up, preserve top (contracts, rules, interfaces)

### Post-Execution Compression
- Keep only contracts, interfaces, and critical decisions
- Archive full code in `/modules/<id>/code/`
- State files reduced to: `id, st, ctx, tok, err, rev`
- Never compress interface definitions, type schemas, or import graphs

### Compression Rules
- Never remove type hints, function signatures, or contract references
- Never remove import paths or dependency declarations
- Never compress error codes or fix instructions
- Use token-aware truncation: cut only after validation passes
- Maintain diff-only updates to avoid full file rewrites

## Validation & Quality Control
Deterministic validation replaces LLM trust to prevent context degradation and sloppy code.

### Validation Pipeline
1. Lint: `ruff`, `eslint`, `mypy`
2. Typecheck: Pydantic, TypeScript, JSON schema
3. Contract verify: import graph, interface matcher, dependency resolver
4. Test run: `pytest`, `jest`, `vitest`
5. Context check: token counter, diff validator, truncation safe check

### Feedback Format
- Machine-readable JSON/YAML only
- Direct fix instructions, no explanations
- Structured error codes: `TYPE_MISMATCH`, `IMPORT_MISSING`, `CTX_LIMIT`, `TEST_FAIL`
- Example:
```json
{"code": "TYPE_MISMATCH", "file": "backend.py", "line": 15, "expected": "UUID", "found": "str", "fix": "update type hint to UUID"}
```

### Retry Logic
- Max 3 retries per module
- On retry: apply fix instruction, reset turn counter, reduce context budget by 10%
- On failure after 3 retries: halt batch, log error, await human input

## Autonomous Runtime & Checkpointing
The system must survive interruptions, LLM timeouts, and context drift during multi-hour sessions.

### Checkpoint Protocol
1. After planning: save `plan.yaml`, `state.yaml`, contracts, dependency graph
2. After each batch: update `state.yaml`, save new checkpoint
3. On failure: rollback to last valid checkpoint, apply retry logic
4. Resume: load latest checkpoint, continue from active batch

### State Management
- Use `yaml` or `json` with strict schemas
- Track only: `id, st, ctx, tok, err, rev, dep`
- Atomic writes: update state file after each turn
- File watchers or async polling for batch completion detection

## Configuration & Environment

### Config Schema (`config.yaml`)
```yaml
llm:
  endpoint: "https://api.provider.com/v1"
  model: "slow-model-v2"
  api_key: "${LLM_API_KEY}"
  max_tokens_per_prompt: 1200
  temperature: 0.1
  timeout_seconds: 60

execution:
  max_parallel_subagents: 5
  max_turns_per_module: 12
  context_compaction_threshold: 0.8
  diff_only_updates: true
  checkpoint_interval_minutes: 10
  global_timeout_minutes: 480

validation:
  lint: ["ruff", "eslint", "mypy"]
  typecheck: ["pydantic", "typescript", "jsonschema"]
  test: ["pytest", "jest", "vitest"]
  contract_verify: true
  import_graph_check: true

paths:
  project_root: "./"
  modules_dir: "./modules"
  contracts_dir: "./contracts"
  checkpoints_dir: "./orchestrator/checkpoints"
  logs_dir: "./orchestrator/logs"
  config_file: "./config.yaml"
```

### Environment Variables
- `LLM_API_KEY`: Provider authentication
- `LLM_ENDPOINT`: API base URL
- `PROJECT_ROOT`: Override project directory
- `LOG_LEVEL`: `info`, `debug`, `error`
- `CLI_OUTPUT_FORMAT`: `text`, `json`, `compact`

## Implementation Stack Recommendations
| Component | Recommendation |
|-----------|----------------|
| CLI Framework | `click` or `typer` for structured commands, subcommands, and argument parsing |
| Async Execution | `asyncio` with `asyncio.create_task()` for sub-agent management |
| Process Management | `subprocess` or `asyncio.create_subprocess_exec()` for sub-agent isolation |
| File Watching | `watchfiles` or `aiofiles` for state file monitoring |
| Logging | `structlog` or `python-json-logger` for structured, machine-readable logs |
| Configuration | `pydantic-settings` for validated config loading from YAML/env |
| State Management | `pydantic` models for strict YAML/JSON parsing and validation |
| Token Tracking | `tiktoken` or `tokenizers` with real-time counters |
| Checkpointing | Atomic file writes with `.tmp` suffix, then rename |
| Signal Handling | `signal.signal()` with async-safe cleanup routines |

## Implementation Notes
1. Atomic State Updates: Use temporary files and rename operations to prevent partial reads during checkpointing
2. Sub-Agent Isolation: Run each sub-agent in a separate process or container with strict resource limits
3. Token-Aware Truncation: Implement a tokenizer-aware truncation function that preserves contracts, interfaces, and type schemas while stripping non-essential state
4. Diff-Only Merges: Sub-agents submit structured diffs. Orchestrator applies only validated changes to prevent context bloat
5. Deterministic Validation: Linters, type checkers, and contract validators run before any state transition. LLM self-assessment is never trusted
6. Graceful Degradation: If context limits are repeatedly hit, auto-split modules, regenerate contracts, and reset budgets without human intervention
7. CLI Exit Codes: `0` success, `1` general error, `2` validation failure, `3` context limit exceeded, `4` configuration error
8. No Human-Readable Bloat: All state files, prompts, and feedback are strictly LLM-optimized. CLI output is the only human-facing layer

## Next Steps & MVP Plan
| Phase | Goal | Deliverable |
|-------|------|-------------|
| 1. CLI Skeleton | Command structure, argument parsing, config loading | Working CLI with init, status, logs commands |
| 2. Orchestrator Core | State machine, batch scheduling, sub-agent spawning | Autonomous loop with checkpointing |
| 3. Token Tracker | Real-time counting, compression triggers, diff-only merges | Context-aware execution engine |
| 4. Validation Pipeline | Linter, typecheck, contract verify, test runner | Deterministic quality gates |
| 5. Daemon & Signals | Background execution, PID management, graceful shutdown | Production-ready CLI runtime |
| 6. Polish | Logging, exit codes, error handling, documentation | Complete autonomous coding system |

## Final Notes
This architecture enforces strict LLM-centric optimization at every layer. State files, prompts, validation feedback, and code generation are all structured for minimal token consumption while preserving deterministic quality. Context compression operates safely by preserving contracts, interfaces, and type schemas while stripping all non-essential text. The orchestrator acts as a state machine, not a conversational partner, enabling hours-long autonomous execution on slower or smaller LLMs.

Specify which component you want to implement first. I will provide production-ready CLI code, state schemas, token-tracking logic, and async execution patterns tailored to your target stack.

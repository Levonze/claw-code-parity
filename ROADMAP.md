# ROADMAP.md

# Clawable Coding Harness Roadmap

## Goal

Turn claw-code into the most **clawable** coding harness:
- no human-first terminal assumptions
- no fragile prompt injection timing
- no opaque session state
- no hidden plugin or MCP failures
- no manual babysitting for routine recovery

This roadmap assumes the primary users are **claws wired through hooks, plugins, sessions, and channel events**.

## Definition of "clawable"

A clawable harness is:
- deterministic to start
- machine-readable in state and failure modes
- recoverable without a human watching the terminal
- branch/test/worktree aware
- plugin/MCP lifecycle aware
- event-first, not log-first
- capable of autonomous next-step execution

## Current Pain Points

### 1. Session boot is fragile
- trust prompts can block TUI startup
- prompts can land in the shell instead of the coding agent
- "session exists" does not mean "session is ready"

### 2. Truth is split across layers
- tmux state
- clawhip event stream
- git/worktree state
- test state
- gateway/plugin/MCP runtime state

### 3. Events are too log-shaped
- claws currently infer too much from noisy text
- important states are not normalized into machine-readable events

### 4. Recovery loops are too manual
- restart worker
- accept trust prompt
- re-inject prompt
- detect stale branch
- retry failed startup
- classify infra vs code failures manually

### 5. Branch freshness is not enforced enough
- side branches can miss already-landed main fixes
- broad test failures can be stale-branch noise instead of real regressions

### 6. Plugin/MCP failures are under-classified
- startup failures, handshake failures, config errors, partial startup, and degraded mode are not exposed cleanly enough

### 7. Human UX still leaks into claw workflows
- too much depends on terminal/TUI behavior instead of explicit agent state transitions and control APIs

## Product Principles

1. **State machine first** — every worker has explicit lifecycle states.
2. **Events over scraped prose** — channel output should be derived from typed events.
3. **Recovery before escalation** — known failure modes should auto-heal once before asking for help.
4. **Branch freshness before blame** — detect stale branches before treating red tests as new regressions.
5. **Partial success is first-class** — e.g. MCP startup can succeed for some servers and fail for others, with structured degraded-mode reporting.
6. **Terminal is transport, not truth** — tmux/TUI may remain implementation details, but orchestration state must live above them.
7. **Policy is executable** — merge, retry, rebase, stale cleanup, and escalation rules should be machine-enforced.

## Roadmap

## Phase 1 — Reliable Worker Boot

### 1. Ready-handshake lifecycle for coding workers
Add explicit states:
- `spawning`
- `trust_required`
- `ready_for_prompt`
- `prompt_accepted`
- `running`
- `blocked`
- `finished`
- `failed`

Acceptance:
- prompts are never sent before `ready_for_prompt`
- trust prompt state is detectable and emitted
- shell misdelivery becomes detectable as a first-class failure state

### 2. Trust prompt resolver
Add allowlisted auto-trust behavior for known repos/worktrees.

Acceptance:
- trusted repos auto-clear trust prompts
- events emitted for `trust_required` and `trust_resolved`
- non-allowlisted repos remain gated

### 3. Structured session control API
Provide machine control above tmux:
- create worker
- await ready
- send task
- fetch state
- fetch last error
- restart worker
- terminate worker

Acceptance:
- a claw can operate a coding worker without raw send-keys as the primary control plane

## Phase 2 — Event-Native Clawhip Integration

### 4. Canonical lane event schema
Define typed events such as:
- `lane.started`
- `lane.ready`
- `lane.prompt_misdelivery`
- `lane.blocked`
- `lane.red`
- `lane.green`
- `lane.commit.created`
- `lane.pr.opened`
- `lane.merge.ready`
- `lane.finished`
- `lane.failed`
- `branch.stale_against_main`

Acceptance:
- clawhip consumes typed lane events
- Discord summaries are rendered from structured events instead of pane scraping alone

### 5. Failure taxonomy
Normalize failure classes:
- `prompt_delivery`
- `trust_gate`
- `branch_divergence`
- `compile`
- `test`
- `plugin_startup`
- `mcp_startup`
- `mcp_handshake`
- `gateway_routing`
- `tool_runtime`
- `infra`

Acceptance:
- blockers are machine-classified
- dashboards and retry policies can branch on failure type

### 6. Actionable summary compression
Collapse noisy event streams into:
- current phase
- last successful checkpoint
- current blocker
- recommended next recovery action

Acceptance:
- channel status updates stay short and machine-grounded
- claws stop inferring state from raw build spam

## Phase 3 — Branch/Test Awareness and Auto-Recovery

### 7. Stale-branch detection before broad verification
Before broad test runs, compare current branch to `main` and detect if known fixes are missing.

Acceptance:
- emit `branch.stale_against_main`
- suggest or auto-run rebase/merge-forward according to policy
- avoid misclassifying stale-branch failures as new regressions

### 8. Recovery recipes for common failures
Encode known automatic recoveries for:
- trust prompt unresolved
- prompt delivered to shell
- stale branch
- compile red after cross-crate refactor
- MCP startup handshake failure
- partial plugin startup

Acceptance:
- one automatic recovery attempt occurs before escalation
- the attempted recovery is itself emitted as structured event data

### 9. Green-ness contract
Workers should distinguish:
- targeted tests green
- package green
- workspace green
- merge-ready green

Acceptance:
- no more ambiguous "tests passed" messaging
- merge policy can require the correct green level for the lane type

## Phase 4 — Claws-First Task Execution

### 10. Typed task packet format
Define a structured task packet with fields like:
- objective
- scope
- repo/worktree
- branch policy
- acceptance tests
- commit policy
- reporting contract
- escalation policy

Acceptance:
- claws can dispatch work without relying on long natural-language prompt blobs alone
- task packets can be logged, retried, and transformed safely

### 11. Policy engine for autonomous coding
Encode automation rules such as:
- if green + scoped diff + review passed -> merge to dev
- if stale branch -> merge-forward before broad tests
- if startup blocked -> recover once, then escalate
- if lane completed -> emit closeout and cleanup session

Acceptance:
- doctrine moves from chat instructions into executable rules

### 12. Claw-native dashboards / lane board
Expose a machine-readable board of:
- repos
- active claws
- worktrees
- branch freshness
- red/green state
- current blocker
- merge readiness
- last meaningful event

Acceptance:
- claws can query status directly
- human-facing views become a rendering layer, not the source of truth

## Phase 5 — Plugin and MCP Lifecycle Maturity

### 13. First-class plugin/MCP lifecycle contract
Each plugin/MCP integration should expose:
- config validation contract
- startup healthcheck
- discovery result
- degraded-mode behavior
- shutdown/cleanup contract

Acceptance:
- partial-startup and per-server failures are reported structurally
- successful servers remain usable even when one server fails

### 14. MCP end-to-end lifecycle parity
Close gaps from:
- config load
- server registration
- spawn/connect
- initialize handshake
- tool/resource discovery
- invocation path
- error surfacing
- shutdown/cleanup

Acceptance:
- parity harness and runtime tests cover healthy and degraded startup cases
- broken servers are surfaced as structured failures, not opaque warnings

## Immediate Backlog (from current real pain)

Priority order: P0 = blocks CI/green state, P1 = blocks integration wiring, P2 = clawability hardening, P3 = swarm-efficiency improvements.

**P0 — Fix first (CI reliability)**
1. Isolate `render_diff_report` tests into tmpdir — flaky under `cargo test --workspace`; reads real working-tree state; breaks CI during active worktree ops
2. Expand GitHub CI from single-crate coverage to workspace-grade verification — current `rust-ci.yml` runs `cargo fmt` and `cargo test -p rusty-claude-cli`, but misses broader `cargo test --workspace` coverage that already passes locally
3. Add release-grade binary workflow — repo has a Rust CLI and release intent, but no GitHub Actions path that builds tagged artifacts / checks release packaging before a publish step
4. Add container-first test/run docs — runtime detects Docker/Podman/container state, but docs do not show a canonical container workflow for `cargo test --workspace`, binary execution, or bind-mounted repo usage
5. Surface `doctor` / preflight diagnostics in onboarding docs and help — the CLI already has setup-diagnosis commands and branch preflight machinery, but they are not prominent enough in README/USAGE, so new users still ask manual setup questions instead of running a built-in health check first
6. Add branding/source-of-truth residue checks for docs — after repo migration, old org names can survive in badges, star-history URLs, and copied snippets; docs need a consistency pass or CI lint to catch stale branding automatically
7. Reconcile README product narrative with current repo reality — top-level docs now say the active workspace is Rust, but later sections still describe the repo as Python-first; users should not have to infer which implementation is canonical

**P1 — Next (integration wiring, unblocks verification)**
2. Add cross-module integration tests — **done**: 12 integration tests covering worker→recovery→policy, stale_branch→policy, green_contract→policy, reconciliation flows
3. Wire lane-completion emitter — **done**: `lane_completion` module with `detect_lane_completion()` auto-sets `LaneContext::completed` from session-finished + tests-green + push-complete → policy closeout
4. Wire `SummaryCompressor` into the lane event pipeline — **done**: `compress_summary_text()` feeds into `LaneEvent::Finished` detail field in `tools/src/lib.rs`

**P2 — Clawability hardening (original backlog)**
5. Worker readiness handshake + trust resolution — **done**: `WorkerStatus` state machine with `Spawning` → `TrustRequired` → `ReadyForPrompt` → `PromptAccepted` → `Running` lifecycle, `trust_auto_resolve` + `trust_gate_cleared` gating
6. Prompt misdelivery detection and recovery — **done**: `prompt_delivery_attempts` counter, `PromptMisdelivery` event detection, `auto_recover_prompt_misdelivery` + `replay_prompt` recovery arm
7. Canonical lane event schema in clawhip — **done**: `LaneEvent` enum with `Started/Blocked/Failed/Finished` variants, `LaneEvent::new()` typed constructor, `tools/src/lib.rs` integration
8. Failure taxonomy + blocker normalization — **done**: `WorkerFailureKind` enum (`TrustGate/PromptDelivery/Protocol/Provider`), `FailureScenario::from_worker_failure_kind()` bridge to recovery recipes
9. Stale-branch detection before workspace tests — **done**: `stale_branch.rs` module with freshness detection, behind/ahead metrics, policy integration
10. MCP structured degraded-startup reporting — **done**: `McpManager` degraded-startup reporting (+183 lines in `mcp_stdio.rs`), failed server classification (startup/handshake/config/partial), structured `failed_servers` + `recovery_recommendations` in tool output
11. Structured task packet format — **done**: `task_packet.rs` module with `TaskPacket` struct, validation, serialization, `TaskScope` resolution (workspace/module/single-file/custom), integrated into `tools/src/lib.rs`
12. Lane board / machine-readable status API — **done**: Lane completion hardening + `LaneContext::completed` auto-detection + MCP degraded reporting surface machine-readable state
13. **Session completion failure classification** — **done**: `WorkerFailureKind::Provider` + `observe_completion()` + recovery recipe bridge landed
14. **Config merge validation gap** — **done**: `config.rs` hook validation before deep-merge (+56 lines), malformed entries fail with source-path context instead of merged parse errors
15. **MCP manager discovery flaky test** — `manager_discovery_report_keeps_healthy_servers_when_one_server_fails` has intermittent timing issues in CI; temporarily ignored, needs root cause fix

16. **Commit provenance / worktree-aware push events** — clawhip build stream shows duplicate-looking commit messages and worktree-originated pushes without clear supersession indicators; add worktree/branch metadata to push events and de-dup superseded commits in build stream display

**P3 — Swarm efficiency**
13. Swarm branch-lock protocol — detect same-module/same-branch collision before parallel workers drift into duplicate implementation
14. Commit provenance / worktree-aware push events — emit branch, worktree, superseded-by, and canonical commit lineage so parallel sessions stop producing duplicate-looking push summaries

## Suggested Session Split

### Session A — worker boot protocol
Focus:
- trust prompt detection
- ready-for-prompt handshake
- prompt misdelivery detection

### Session B — clawhip lane events
Focus:
- canonical lane event schema
- failure taxonomy
- summary compression

### Session C — branch/test intelligence
Focus:
- stale-branch detection
- green-level contract
- recovery recipes

### Session D — MCP lifecycle hardening
Focus:
- startup/handshake reliability
- structured failed server reporting
- degraded-mode runtime behavior
- lifecycle tests/harness coverage

### Session E — typed task packets + policy engine
Focus:
- structured task format
- retry/merge/escalation rules
- autonomous lane closure behavior

## MVP Success Criteria

We should consider claw-code materially more clawable when:
- a claw can start a worker and know with certainty when it is ready
- claws no longer accidentally type tasks into the shell
- stale-branch failures are identified before they waste debugging time
- clawhip reports machine states, not just tmux prose
- MCP/plugin startup failures are classified and surfaced cleanly
- a coding lane can self-recover from common startup and branch issues without human babysitting

## Short Version

claw-code should evolve from:
- a CLI a human can also drive

to:
- a **claw-native execution runtime**
- an **event-native orchestration substrate**
- a **plugin/hook-first autonomous coding harness**

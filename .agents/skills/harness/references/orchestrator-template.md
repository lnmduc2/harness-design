# Orchestrator Skill Templates (v2)

An orchestrator coordinates the entire agent team. This document provides three templates tailored to each execution mode, alongside guidance for combining modes across stages.

- **Template A: Workflow Orchestration** — Large-scale enumerable fan-outs or iterative verification loops with predetermined control flow.
- **Template B: Persistent Agent Collaboration** — Multi-turn feedback, negotiation, and sustained collaboration where agents must retain historical conversational context.
- **Template C: Sub-agent Delegation** — Independent tasks dispatched concurrently as one-off delegations where only results are collected.
- **Hybrid Mode** — Combining appropriate execution modes stage by stage.

All templates incorporate a Phase 0 context check for existing work, standard `_workspace/` file conventions, error handling policies, and test scenarios.

---

## Template A: Workflow Orchestration

```markdown
---
name: {domain}-orchestrator
description: "Orchestrates the {domain} workflow. Use when requested with {initial trigger keywords}, and also use for follow-up requests such as modifying {domain} results, partial re-runs, updating, refining, re-running, or improving previous results."
---

# {Domain} Orchestrator

## Execution Mode: Workflow Orchestration

This skill coordinates deterministic workflows via the Workflow tool. Invoking this skill
implies user consent to utilize Workflow. Restrict default agent counts to {N} agents; scale up
to large fleets only when the user explicitly requests "exhaustive", "thorough", or "comprehensive" sweeps.

## Agent Roster

| Role | `agentType` | Skill | Deliverable (`schema` Summary) |
|---|---|---|---|
| {collector} | {custom or built-in} | {skill} | {findings: [...]} |
| {verifier}  | {custom or built-in} | {skill} | {status, reason} |

Specify custom agent names defined in `.claude/agents/` or built-in agent types in `{custom or built-in}`.

## Workflow Phases

### Phase 0: Context Audit (Follow-up Request Handling)

1. Check for the existence of `_workspace/`:
   - If missing → Execute anew from the beginning.
   - If present and the request asks for partial revisions → Re-run only the target phase/agent.
   - If present but fresh input was provided → Archive existing `_workspace/` to `_workspace_{ts}/` and execute afresh.
2. For partial re-runs with a prior `runId` recorded in `_workspace/run_meta.json`:
   - Modify only the affected stage in the script and resume execution via `resumeFromRunId`. Unchanged
     `agent()` calls return cached results immediately, dramatically reducing re-run costs.
3. Upon launching fresh runs, record the newly issued `runId` into `_workspace/run_meta.json`.

### Phase 1: Lock Down Task Items (Executed directly by Main Agent)

Before launching the workflow, survey the {file list / research axes / audit targets} and organize them into an array.
Pass this array into the script via `args`.

### Phase 2: Workflow Execution

Pass the following script to the Workflow tool. For detailed implementation patterns, see `workflow-recipes.md` in the harness skill.

- `meta`: `name` is '{domain}-run', `phases` are [{Collection}, {Verification}, {Synthesis}]
- Collection: pipeline(args.items, item => agent(..., {schema: COLLECT}))
- Verification: Adversarial verification per discovered finding → .filter(Boolean) → Forward only confirmed findings to downstream stages
- Synthesis: return structured result object

Wait for the completion notification after dispatching. Do not preemptively assume results before notifications arrive.

### Phase 3: Result Synthesis and Reporting

1. Ingest the structured object returned by the workflow.
2. Explicitly disclose any excluded or omitted items in the final report.
3. Author the final deliverable at `{output-path}/{filename}`.
4. Preserve intermediate artifacts and `run_meta.json` in `_workspace/`.

## Error Handling

| Scenario | Mitigation |
|---|---|
| Individual `agent()` failure (returns `null`) | Filter out via `.filter(Boolean)` and log excluded count via `log()`. Disclose missing items in the final report. |
| Workflow-level failure | Inspect the `journal` for raw return values, fix only the failed stage, and resume via `resumeFromRunId`. |
| Irrecoverable failure (quota exhaustion, expired auth, permission denied) | Do not retry or resume until the root cause is resolved, as retries merely burn quota. Inspect the `journal` and partial artifacts directly to establish actual progress, log omitted items to `_workspace/`, and report to the user (including reset time if rate-limited). Once resolved, resume via `resumeFromRunId`. The main agent backfills only directly verified facts; never extrapolate or guess an agent's subjective reasoning. |
| Empty result set | Do not misinterpret as success. Inspect the `journal` to verify actual return payloads from each agent. |
| Conflicting data | Do not discard; cite conflicting viewpoints alongside their respective sources. |

## Test Scenarios

### Happy Path
1. User provides {input}; pre-investigation locks down {M} items.
2. Workflow collects data across {M} items; {K} items pass adversarial verification, followed by final synthesis.
3. Expected Outcome: `{output-path}/{filename}` is created and run metadata is persisted in `run_meta.json`.

### Error Recovery Path
1. Two items return `null` during the collection phase.
2. `.filter(Boolean)` excludes failed items; `log()` reports "2 items excluded".
3. Final report explicitly notes "Failed to collect 2 items for {item-names}".
```

---

## Template B: Persistent Agent Collaboration

```markdown
---
name: {domain}-orchestrator
description: "Coordinates collaboration across {domain} agents. Use when requested with {initial trigger keywords}, and also use for follow-up requests such as modifications, partial re-runs, updating, refining, re-running, or improving previous results."
---

# {Domain} Orchestrator

## Execution Mode: Persistent Agent Collaboration

No explicit team object is created as in v1. Agents launched with names in a session automatically
belong to a shared collaborative context. TeamCreate and TeamDelete are obsolete v1 artifacts.

## Agent Roster

| Name | `subagent_type` | Role | Skill | Deliverable |
|---|---|---|---|---|
| {teammate-1} | {custom or built-in} | {role} | {skill} | {output-file} |
| {teammate-2} | {custom or built-in} | {role} | {skill} | {output-file} |

Specify custom agent names defined in `.claude/agents/` or built-in types in `{custom or built-in}`.

## Workflow Phases

### Phase 0: Context Audit
As in Template A, inspect `_workspace/` to decide whether to execute from scratch, re-run partially,
or initialize a new run. For partial re-runs, inject previous deliverable paths into agent prompts
so they can ingest and iterate upon existing work.

### Phase 1: Preparation
1. Analyze user input to identify {core objectives and constraints}.
2. Initialize `_workspace/` and persist input to `_workspace/00_input/`.

### Phase 2: Agent Dispatch & Task Registration

1. Launch agents concurrently within a single message turn. Declaring `name` is mandatory for addressing via SendMessage:
   - Agent(name: "{teammate-1}", subagent_type: "{type}", prompt: "{role + task instructions + output path}")
   - Agent(name: "{teammate-2}", subagent_type: "{type}", prompt: "...")
2. Register the shared task list:
   - TaskCreate({title: "{task-1}", assignee: "{teammate-1}"})
   - TaskCreate({title: "{task-3}", depends_on: [task-1]})
   > Allocate 3–6 tasks per teammate. Declare inter-task dependencies explicitly via `depends_on`.

### Phase 3: Collaborative Coordination

- Monitor execution as teammates post completion or idle notifications. Track overarching progress via TaskList.
- Request reviews via SendMessage({to: "{teammate-2}"}, "Review draft at _workspace/01_... by {teammate-1}
  and highlight needed fixes"). Relay feedback to {teammate-1} for revisions.
  Because historical context is retained, instructions can be tightly scoped: "In that draft, revise only Section 2."
- When teammates need to discuss findings, the leader mediates the exchange. Deliverables must always be written to disk.

**Deliverable Storage Paths:**

| Teammate | Output Path |
|---|---|
| {teammate-1} | `_workspace/{phase}_{teammate-1}_{artifact}.md` |
| {teammate-2} | `_workspace/{phase}_{teammate-2}_{artifact}.md` |

### Phase 4: Artifact Freezing

Named agents continue receiving messages even after reporting completion. If an agent edits an artifact already handed off in response to late questions from another teammate, downstream stages will be working off stale, pre-edit data. Cross-checking between teammates catches bugs and should not be blocked; instead, freeze deliverables at every stage boundary so downstream files remain immutable.

1. Verify via TaskList and completion notifications that all teammates assigned to the stage have reported completion.
2. Notify teammates of deliverable freezing via SendMessage: "Phase {phase} deliverables are now frozen. If revisions are required hereafter, do not modify existing files; write a new versioned file (e.g., `{phase}_{teammate}_{artifact}_v2.md`) and notify the leader."
3. Record artifact hashes (e.g., `shasum _workspace/{phase}_* > _workspace/freeze_{phase}.sha` or `md5sum`).
4. Pass deliverable paths and hash file paths into the prompts of agents in subsequent stages.

Repeat this freezing protocol at every stage boundary across multi-phase workflows.

### Phase 5: Integration
1. Confirm via TaskList that all registered tasks are completed.
2. Ingest deliverables via Read and apply {integration / verification logic}.
3. Prior to synthesizing the final deliverable, re-verify hashes (e.g., `shasum -c _workspace/freeze_{phase}.sha`) to confirm frozen artifacts were not mutated. If hashes diverge or new version files exist, inspect changes and decide whether to re-run downstream stages that ingested pre-edit versions.
4. Author the final deliverable at `{output-path}/{filename}`.

### Phase 6: Wrap-up
1. Preserve `_workspace/` to maintain the audit trail for post-hoc verification.
2. Report executive summary to the user.
   There is no explicit team teardown step. Agents conclude automatically upon task completion, or can be halted via TaskStop.

## Error Handling

| Scenario | Mitigation |
|---|---|
| Teammate unresponsive or halted | Check status via SendMessage and re-issue instructions. If failure persists, launch a replacement agent with the same custom type under a new name and supply necessary task context in the prompt. |
| Halt caused by quota exhaustion, expired auth, or permission denied | Do not retry or launch replacements, as the outcome will be identical. Inspect partial artifacts directly to establish progress, document omissions in `_workspace/`, and report to the user (including quota reset time). The leader backfills only directly verified facts; never guess an agent's subjective reasoning. |
| Strict majority of team fails | Alert the user and ask whether to proceed with partial results. |
| Execution timeout | Proceed with results collected thus far, noting incomplete areas in the final report. |
| Contradictory findings | Do not discard; cite conflicting claims side-by-side with source attribution. |
| Task status sync lag | Inspect via TaskList and update directly via TaskUpdate. |

## Test Scenarios

### Happy Path
1. User provides {input}; Phase 2 launches {N} agents and registers {M} tasks.
2. Phase 3 exchanges feedback {K} times; Phase 4 freezes deliverables; Phase 5 integrates results.
3. Expected Outcome: `{output-path}/{filename}` is created.

### Error Recovery Path
1. {teammate-2} becomes unresponsive in Phase 3.
2. Leader checks status via SendMessage and re-prompts. Upon persistent failure, leader launches replacement agent "{teammate-2}b" passing existing artifact paths.
3. Final report documents "Partial re-work in {teammate-2} domain".
```

---

## Template C: Sub-agent Delegation

```markdown
---
name: {domain}-orchestrator
description: "Delegates independent {domain} tasks to subagents. Use when requested with {initial trigger keywords}, and also use for follow-up requests such as modifications, partial re-runs, updating, refining, re-running, or improving previous results."
---

## Execution Mode: Sub-agent Delegation

## Workflow Phases

### Phase 0: Context Audit
Inspect `_workspace/` to decide whether to execute from scratch, re-run partially, or initialize a new run.

### Phase 1: Preparation
Analyze input and initialize `_workspace/`.

### Phase 2: Parallel Execution
Dispatch the Agent tool N times concurrently within a single message turn. Runs in the background by default.

| Agent | `subagent_type` | Input | Deliverable |
|---|---|---|---|
| {agent-1} | {type} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` |
| {agent-2} | {type} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` |

Specify custom agent names or built-in types in `{type}`.

Wait for completion notifications. The main agent avoids re-performing delegated searches.

### Phase 3: Integration
1. Aggregate return values and file artifacts, synthesize findings, and author the final deliverable.

### Phase 4: Wrap-up
Preserve `_workspace/` and report summary findings to the user.

## Error Handling
- Retry a failed agent once. If it fails again, note the omission and proceed.
- Do not retry failures with identical outcomes (quota exhaustion, expired auth, permission denied). Inspect partial deliverables directly, log omissions to `_workspace/`, and report to the user (with quota reset time). Backfill only verified facts; never extrapolate subjective reasoning.
- If over half the agents fail, alert the user and confirm whether to proceed.
```

---

## Combining Execution Modes Stage-by-Stage (Hybrid Mode)

Apply the most appropriate execution mode per stage. Annotate the top of each phase with `**Execution Mode:**`.

```markdown
## Execution Mode: Hybrid

| Stage | Execution Mode | Selection Rationale |
|---|---|---|
| Stage 1 (Pre-investigation) | Main Agent directly or Sub-agent Delegation | Locking down the task item list. |
| Stage 2 (Massive Collection & Verification) | Workflow Orchestration | Processing pre-defined items via fan-out and adversarial verification. |
| Stage 3 (Consensus & Integration) | Persistent Agent Collaboration | Discussing contradictory data to reach team consensus. |
| Stage 4 (Independent Audit) | Sub-agent Delegation | Single independent audit pass conducted by a dedicated QA agent. |
```

**Mode Transition Rules:**
- Workflow Orchestration → Persistent Agent Collaboration: Save structured data returned by Workflow into `_workspace/`, and pass file paths into persistent agent prompts.
- Persistent Agent Collaboration → Workflow Orchestration: Freeze team deliverables (Phase 4 of Template B) and pass the artifact path list into the workflow script via `args`.
- Explicitly document data passing pathways at every mode boundary so downstream stages never search blindly for upstream outputs.

---

## Authoring Principles Applicable to All Templates

1. **Declare execution mode at the top of the document.** In hybrid modes, summarize stage-by-stage modes in a table.
2. **In Workflow Orchestration mode, detail script geometry explicitly.** Declare `phases`, `schema` summaries, and `args` schemas without omissions.
3. **In Persistent Agent Collaboration mode, specify usage of `name`, SendMessage, and TaskCreate.** Detail agent rosters, communication protocols, task dependencies, and stage-boundary artifact freezing procedures.
4. **In Sub-agent Delegation mode, declare Agent call parameters fully.** Clarify that calls are one-off dispatches omitting `name`, and specify `subagent_type`, `prompt`, and deliverable paths.
5. **Use explicit file paths.** Anchor paths to `_workspace/`, adhering strictly to `{phase}_{agent}_{artifact}.{ext}`.
6. **Clarify inter-stage dependencies.** Detail how data crosses boundaries, especially where execution modes change.
7. **Anticipate realistic runtime failures.** Never assume unconditional success; establish concrete handling policies per failure scenario.
8. **Always include test scenarios.** Document at least one happy path and one error/recovery flow.
9. **Eliminate obsolete v1 constructs.** References to TeamCreate, TeamDelete, `team_name`, broadcast messaging, or experimental flags indicate defective authoring.

## Follow-up Request Expressions for `description`

Include the following phrasing in the `description` so the orchestrator triggers reliably for revisions and follow-ups as well as initial runs:

- re-run, execute again, update, modify, refine, revise
- "re-do only {part} of {domain}"
- "based on previous results", "improve output"
- Domain-specific colloquial request verbs (e.g., in a release strategy harness: "launch", "promote", "trending")

Omitting follow-up expressions risks the agent failing to activate the harness when users request subsequent iterations.

---
name: harness
description: "Designs a tailored harness for a project, defining specialized agents and the skills they use. Use when the user requests 'build a harness for this project', 'create a harness', 'design a harness', 'harness engineering', or 'create an agent team'. Also use when building an automation architecture for a new domain or project, or when restructuring/expanding an existing harness. Also handles requests to operate or maintain an existing harness, such as 'check harness', 'audit harness', 'harness status', or 'sync agents/skills'. Retrospecting execution results and incorporating feedback is handled by the harness:evolve skill."
---

# Harness v2 — Agent Team and Skill Design

Design a tailored harness for a project. Define each agent's role and create the skills they follow when performing work.

## Core Principles

1. Place agent definitions in `project/.claude/agents/` and skills in `project/.claude/skills/`. Agents define *who* does the work, and skills define *how* that work is done.
2. Select an execution mode based on the control-flow shape of the work. If the sequence and iteration conditions can be determined deterministically in code, orchestrate via Workflows. If iterative feedback and discussion with the same specialists are required, use persistent agents. If only the final result is needed in a fire-and-forget manner, delegate to subagents. Detailed selection criteria are covered in Phase 2.
3. Select models according to task complexity, duration, autonomy, and response latency. Use `fable` for the highest-difficulty tasks requiring long-horizon autonomous planning and execution; use `opus` for design, code generation, complex analysis, and cross-verification; use `sonnet` for routine tasks like log analysis, format conversion, and simple collection. Detailed criteria are covered in Phase 3.
4. To allow orchestrator skills to be loaded cleanly in new sessions, record only invocation triggers and change history in `CLAUDE.md`.
5. Continuously incorporate lessons learned from execution results into agents, skills, and `CLAUDE.md`. Retrospective analysis and delta discovery are handled by the `harness:evolve` skill.
6. Write generated agent definitions, skills, orchestrators, and `CLAUDE.md` records in the language the user uses in conversation. Do not write deliverables in Korean simply because this skill document and templates in `references/` were originally written in Korean. Translate template titles and example phrases into the target language. If the user specifies a language, follow it; if expanding an existing harness without a specified language, follow the language of the existing files.

## Workflow

### Phase 0: Current State Audit

When the harness skill is invoked, first inspect any existing configuration.

1. Read `project/.claude/agents/`, `project/.claude/skills/`, and `project/CLAUDE.md`.
2. Select the procedure to follow based on the current state:
   - **New Build**: If agent or skill directories do not exist or are empty, execute all phases starting from Phase 1.
   - **Extend Existing Configuration**: If adding agents or skills to an existing harness, execute only the required phases indicated in the table below.
   - **Operations & Maintenance**: If auditing, modifying, or synchronizing an existing harness, proceed directly to the operational procedure in Phase 7.

   | Change Content | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 | Phase 6 |
   | --- | --- | --- | --- | --- | --- | --- |
   | Add Agent | Skip (use Phase 0 results) | Decide which execution mode, team, and phase only | Required | When dedicated skill is needed | Modify Orchestrator | Required |
   | Add/Modify Skill | Skip | Skip | Skip | Required | When connections change | Required |
   | Modify Architecture / Mode | Skip | Required | Affected agents only | Affected skills only | Required | Required |

3. If the existing orchestrator contains `TeamCreate`, `TeamDelete`, or `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`, these are v1 artifacts. They will not function in current execution environments; propose migrating to v2. For detailed steps, follow "Migrating from v1 to v2" in `references/execution-modes.md`.
4. Cross-check the actual list of agents and skills against records in `CLAUDE.md` to identify any discrepancies.
5. Report the audit findings and execution plan to the user for confirmation.

### Phase 1: Domain and Task Analysis

1. Understand the project domain and goals from the user request.
2. Categorize the required tasks into functional types such as generation, verification, editing, and analysis.
3. Analyze the control flow to select the appropriate execution mode:
   - Can the list of items to process be enumerated in advance? (e.g., migrating N files or reviewing M perspectives).
   - Is an iterative loop required to verify and revise artifacts?
   - Do agents need mutual exchange of opinions to improve quality?
   - Is ongoing dialogue with the same specialist required within a single session?
4. Identify any overlaps or conflicts with existing agents and skills identified in Phase 0.
5. Inspect the codebase to identify the tech stack, data models, and primary modules.
6. Calibrate explanation difficulty based on user terminology and technical depth. Avoid using technical terms like assertions or JSON Schema with less technical users without proper context.

### Phase 2: Execution Mode and Team Architecture Design

#### 2-1. Execution Mode Selection

Harness v2 utilizes three Claude Code multi-agent capabilities. Select one of the following modes according to the work's control flow:

| Execution Mode | Primitives Used | Suitable Tasks |
| --- | --- | --- |
| **Workflow Orchestration** | `Workflow` scripts with `agent()`, `pipeline()`, `parallel()`, `phase()` | Tasks where item lists, verification procedures, and loop counts can be determined deterministically in code. Tasks requiring structured schema outputs or dispatching dozens of `Agent` calls. |
| **Persistent Agent Collaboration** | `Agent(name:)`, `SendMessage`, `TaskCreate`, `TaskUpdate` | Tasks where named specialists must maintain conversational context while iterating on feedback, negotiation, and collaborative editing. |
| **Sub-agent Delegation** | One-shot `Agent` calls. Runs in background by default, supports parallel dispatch | Tasks where agents do not need to converse with each other and only a single final result is needed. |

Decision rules:

1. If the list of items, verification criteria, and iteration conditions can be expressed in code beforehand, use Workflow. Keeping control flow out of non-deterministic per-turn model decisions ensures reproducibility.
2. If the logic cannot easily be expressed in code and agents must converse or maintain historical context, use persistent agents.
3. If neither applies and only independent task results are needed, delegate to subagents.
4. If different phases possess different characteristics, configure a hybrid mode. Specify each phase's execution mode in the orchestrator.

Using the `Workflow` tool requires explicit user consent. Consent is inferred if the user explicitly requested a workflow execution or invoked an orchestrator skill instructing `Workflow` calls. Restrict default execution to a small number of agents; scale up to large runs only when the user requests comprehensive or exhaustive sweeps.

> For comparisons of execution modes, concurrency limits, schemas, token budgets, and resume conditions, refer to `references/execution-modes.md`.

#### 2-2. Team Architecture Pattern Selection

1. Decompose the task into specialized functional domains.
2. Select the architecture pattern that best fits the work from the six patterns below. Detailed criteria are documented in `references/team-patterns.md`:
   - **Pipeline**: Processes results sequentially from prior stages. Best suited for `pipeline()`.
   - **Fan-out / Fan-in**: Processes independent tasks in parallel and merges results. Uses `pipeline()` and `parallel()` only when necessary.
   - **Expert Pool**: Dispatches only specialists relevant to the input. Best suited for subagents or persistent agents.
   - **Producer-Reviewer**: One agent generates and another reviews. Uses adversarial verification in workflows or persistent agent pairs.
   - **Supervisor**: A central agent monitors progress and dynamically reallocates tasks. Uses persistent agents and shared task lists.
   - **Hierarchical Delegation**: Higher-tier agents delegate down to sub-workers. Restrict to a maximum of two tiers, and limit workflow nesting to one level.
3. If deliverable accuracy is critical, combine the following quality verification patterns:
   - **Adversarial Verification**: Attach N adversarial verification agents to each identified finding; confirm an item only when a majority evaluates it as `confirmed` with sufficient evidence. Do not count `refuted` or `uncertain` verdicts as confirmed.
   - **Judge Panel**: Independently generate N alternative drafts in parallel, evaluate them with a judge panel, and synthesize the final deliverable from the best elements.
   - **Loop-Until-Dry**: Continue searching until newly discovered items equal 0 for K consecutive rounds.
   - **Multi-Modal Sweeps**: Conduct parallel sweeps across differing criteria (e.g., container, content, entity, timeline).
   - **Completeness Critic**: Task the final agent exclusively with discovering missing or omitted elements.

#### 2-3. Agent Division Criteria

Divide agents based on required domain expertise, parallelizability, conversation context to retain, and reusability. Refer to the detailed decision matrix in "Agent Separation Criteria" in `references/team-patterns.md`.

### Phase 3: Agent Definition Authoring

Define reusable specialist agents intended across multiple sessions as `project/.claude/agents/{name}.md` files. Do not inject recurring roles purely into the `prompt` argument of `Agent` tool calls.

- Defining agents in files enables reuse across subsequent sessions. Invoke them via `subagent_type: "{name}"` in the `Agent` tool or `agentType: "{name}"` in `Workflow`.
- Specifying inter-agent inputs and outputs beforehand ensures stable collaboration.
- Agents specify *who* works; skills specify *how* to work, keeping concerns cleanly separated.

Create custom agent definition files for reusable specialist roles, specifying the file name in `subagent_type` or `agentType`. Do not create separate definition files for one-off tasks that use built-in types such as `general-purpose`, `Explore`, or `Plan`.

#### Overlap Check with Existing Agents

Before creating a new agent file, check for role overlap against existing agents in `project/.claude/agents/`. Do not skip this check even when skipping Phase 1 to add an agent. Repeated harness builds or extensions can lead to duplicate agents with different names performing identical roles. If overlap exists, reuse or extend the existing agent rather than creating a duplicate. Review decision criteria and exceptions (deliberate domain specialization) in "Agent Reuse Design" in `references/team-patterns.md`.

#### Model Selection Criteria

Select a model for each agent based on task complexity, duration, autonomy, and response latency. Specify via `model:` in YAML frontmatter or call parameters (`model`, `opts.model`), and document the rationale in comments within the agent definition or orchestrator:

| Model | Selection Criteria | Representative Tasks |
| --- | --- | --- |
| **fable** | Highest-difficulty tasks requiring autonomous long-horizon multi-step planning and execution | Agent orchestration, strategic planning, long-term autonomous execution, massive source synthesis, abstract ideation refinement |
| **opus** | Specialized tasks with well-defined scope requiring deep reasoning and analysis | System architecture, code generation, complex analysis, cross-verification, research critique, creative writing |
| **sonnet** | Routine tasks with explicit procedures where rapid turnaround is paramount | Log analysis, format conversion, static file auditing, deployment script execution, simple data extraction, general summarization |

- Use `fable` when an agent must adjust subsequent plans based on previous results over long autonomous durations. Use `opus` when conducting deep analysis of a well-scoped problem. For example, critiquing a single research paper fits `opus`, whereas reviewing dozens of papers to formulate strategy and author an integrated report fits `fable`.
- Use `sonnet` as the default for standard, unambiguous tasks.
- Do not blanket-assign all agents to `fable` or `opus` simply because the project is important. Select models based on the nature of the task, not the status of the agent.
- Even if a supervisor agent uses `fable` for coordination, assign appropriately tiered models to downstream workers.

> For detailed model selection criteria, refer to `references/model-selection-guide.md`.

#### Agent Definition File Structure

YAML frontmatter must include `name` and `description`. Add `tools` to restrict tool access and `model` to set the model tier as needed. In read-only review and analysis agents, omit `Edit` and `Write` from `tools` to prevent accidental file modifications.

Conversely, grant both `Write` and `Edit` to agents tasked with modifying deliverables. Without `Edit`, an agent must rewrite entire files even for single-line changes; for large files, agents may abandon the edit or take error-prone shortcuts. Furthermore, because specified tools may occasionally be unavailable at runtime, instruct persistent agents to report their actively available tools in their initial check-in.

The markdown body must specify core role, working principles, input/output rules, error handling, and collaboration protocols. For persistent agents, include a `## Communication Rules` section defining message partners for `SendMessage` and shared task list protocols.

> For definition templates, complete examples, and tool restriction guidelines, refer to "Agent Definition Structure" in `references/team-patterns.md`.

#### QA Agent Guidelines

- Assign QA agents a type that has access to all necessary verification tools. `Explore` is read-only and cannot execute test scripts.
- Do not stop at checking file existence; cross-reference API responses against frontend hook signatures and data contracts.
- Do not defer all checks until the entire task is complete; verify each module as soon as it is finished.
- Follow the detailed guidelines in `references/qa-agent-guide.md`.

### Phase 4: Skill Authoring

Author skills for each agent under `project/.claude/skills/{name}/SKILL.md`. Follow the authoring guidelines in `references/skill-writing-guide.md`.

#### 4-0. Check Overlap with Existing Skills

Before creating a new skill, check for functional overlap against existing skills in `project/.claude/skills/`. If overlap exists, connect or extend the existing skill rather than creating a duplicate. Review guidelines on domain specialization boundaries and generalization limits in "Skill Reuse Design" in `references/skill-writing-guide.md`.

#### 4-1. Directory Structure

```
skill-name/
├── SKILL.md              # Required
│   ├── YAML Frontmatter   # name and description required
│   └── Markdown Body
├── scripts/              # Optional: executable code for deterministic, repeatable tasks
├── references/           # Optional: reference documentation loaded on demand
└── assets/               # Optional: static assets like templates and diagrams
```

#### 4-2. Writing Invocation Conditions in `description`

Claude decides whether to invoke a skill based on its `name` and `description`. The `description` must clearly detail what the skill does, the specific circumstances under which it must be invoked, and negative triggers (similar-looking cases where it should NOT be used).

**Poor example:** `"Skill for processing PDF documents"`

**Good example:** `"Processes PDF files including text/table extraction, merging, splitting, rotation, watermarking, encryption, and OCR. MUST use when the user mentions .pdf files or requests PDF deliverables."`

#### 4-3. Skill Body Authoring Principles

| Principle | Application |
| --- | --- |
| **Explain Why First** | Do not just list `ALWAYS` and `NEVER`; state the rationale. Understanding the reason enables sound judgment in edge cases. |
| **Keep It Concise** | Keep `SKILL.md` under 500 lines. Move non-essential background into `references/`. |
| **Generalize to Principles** | Avoid rules tailored only to single examples. Document criteria applicable across diverse inputs. |
| **Pre-bundle Repeated Code** | If multiple agents write identical test or evaluation scripts, package them into `scripts/`. |
| **Use Imperative Style** | Use consistent, clear imperative instructions so expected actions are unambiguous. |

#### 4-4. Progressive Disclosure

Skills load information and executable code progressively across three stages:

| Information | Loading Point | Recommended Size |
| --- | --- | --- |
| **Metadata** (`name`, `description`) | Always loaded | ~100 words |
| **`SKILL.md` Body** | Upon skill activation | Under 500 lines |
| **`references/`** | Loaded on demand when needed | Unrestricted |
| **`scripts/`** | Executed directly for deterministic tasks | Unrestricted (can execute without reading into context) |

- When `SKILL.md` approaches 500 lines, extract detailed reference material into `references/` and document in the body when to read each file.
- If a reference document exceeds 300 lines, provide a table of contents at the top.
- If instructions differ by domain or framework, split reference files so only the relevant file is loaded.

#### 4-5. Connecting Agents and Skills

- A single agent can use one or more skills.
- Multiple agents can share the same skill.
- Skills specify *how* to do the work; agents specify the *role* executing it.

### Phase 5: Integration and Orchestration

An orchestrator is itself a skill. It binds individual agents and skills into a unified workflow, specifying who collaborates, when, and in what order. Complete templates by mode are in `references/orchestrator-template.md`, and workflow script recipes are in `references/workflow-recipes.md`.

When extending an existing harness, modify the existing orchestrator rather than creating a new one. When adding agents, update the team composition, task assignments, data flow, and add new invocation triggers to the `description`.

#### 5-0. Orchestrators by Execution Mode

**A. Workflow Orchestration**

Define a `Workflow` script in the orchestrator skill. The script consists of `meta`, `phase()`, `pipeline()`, `parallel()`, and `agent()`. Specify custom types via `agentType` and enforce structured outputs via `schema`.

```
[Orchestrator Skill] → Workflow(script)
    ├── phase('Collection'): pipeline(items, ...)   ← Distribute pre-defined items
    ├── phase('Verification'): Adversarial verification or Judge panel
    └── return structured result → Leader compiles integrated final report
```

**B. Persistent Agent Collaboration**

Launch named agents and coordinate via shared task lists and `SendMessage`. `TeamCreate` and `TeamDelete` no longer exist. No explicit team object is created; agents launched with names in a session automatically belong to a shared collaborative context. You can re-message them while retaining conversation context.

```
[Main Agent (Leader)]
    ├── Agent(name: "researcher", ...) / Agent(name: "critic", ...)  ← Parallel launch
    ├── TaskCreate(tasks + dependencies)
    ├── SendMessage({to: "critic"}, "Review the draft from researcher")
    └── Collect and synthesize findings
```

**C. Sub-agent Delegation**

Dispatch N `Agent` calls in parallel in a single turn and aggregate results from completion notifications. Dispatched agents run in the background by default.

Configure a hybrid mode if task characteristics differ by phase. For example, collect sources via Workflow, synthesize consensus via persistent agents, and perform adversarial verification on the result via Workflow. Annotate each phase with `**Execution Mode:**`.

#### 5-1. Data Passing Protocols

| Method | Implementation | Best Suited Mode | When to Use |
| --- | --- | --- | --- |
| **Structured Return** | `Workflow`'s `agent(prompt, {schema})` returns validated JSON | Workflow Orchestration | When downstream stages process data programmatically |
| **Standard Return** | Return message from `Agent` tool | Sub-agent Delegation | When the main agent summarizes results directly |
| **Messages** | Direct exchange via `SendMessage` | Persistent Agents | When real-time coordination and feedback are required |
| **Shared Task List** | State tracking via `TaskCreate`, `TaskUpdate` | Persistent Agents | Managing progress, task dependencies, and dynamic assignment |
| **Files** | Read and write to designated workspace paths | All Modes | Large payloads or when intermediate audits are needed |

Follow these rules when passing data via files:

- Store intermediate artifacts in `_workspace/` under the working directory.
- Use the file naming convention `{phase}_{agent}_{artifact}.{ext}` (e.g., `01_analyst_requirements.md`).
- Store only final deliverables at user-designated output paths. Preserve `_workspace/` for post-hoc verification.

#### 5-2. Error Handling

Establish error handling policies in the orchestrator:

- Retry once; if it fails again, proceed without that result but explicitly log the omission in the final report. Never discard conflicting data; preserve it with source citations.
- Do not retry failures that will produce the same outcome, such as rate limit exhaustion, expired credentials, or permission denial. Retrying only burns remaining quota. Inspect partial artifacts directly to determine true progress, log missing items to a file, and report to the user (including quota reset time if rate-limited).
- When the orchestrator fills gaps left by a failed agent, backfill only directly verified facts. Never extrapolate or guess an agent's subjective judgment (e.g., why a source was omitted), as doing so falsifies audit logs.
- In workflows, failed or skipped `agent()` calls return `null`, and `parallel()` does not throw exceptions. Always apply `.filter(Boolean)` to result arrays and report the count of missing items via `log()`.
- If a persistent agent becomes unresponsive, verify status via `SendMessage` and re-issue instructions. If failure persists, launch the same custom type under a new name and pass necessary context in the prompt.

> For handling specific error types, refer to the "Error Handling" section of the corresponding mode template in `references/orchestrator-template.md`.

#### 5-3. Task Scale

| Task Scale | Persistent Agents | Workflow `Agent` Calls |
| --- | --- | --- |
| Small: < 10 tasks | 2–3 agents | 2–5 calls |
| Medium: 10–20 tasks | 3–5 agents | ~10 calls (automatically queued if concurrency limit exceeded) |
| Large: > 20 tasks | Supervisor + 3–5 workers | Dozens to hundreds of calls (overall ceiling 1,000 calls) |

- Coordination overhead increases with agent count. Keep default team sizes lean.
- When the user specifies a token budget (e.g., `+500k`), check `budget.remaining()` in workflow scripts to modulate execution scale.

#### 5-4. Recording Harness Integration in `CLAUDE.md`

Upon completing configuration, document the harness presence and invocation triggers in the project's `CLAUDE.md`. Keep this minimal, as `CLAUDE.md` is loaded into every session.

````markdown
## Harness: {domain-name}

**Goal:** {One-line core objective of the harness}

**Invocation Condition:** When requested to perform work related to {domain}, invoke the `{orchestrator-skill-name}` skill. Simple informational questions may be answered directly.

**Change History:**
| Date | Change Summary | Target | Reason |
| --- | --- | --- | --- |
| {YYYY-MM-DD} | Initial setup via Harness v2 | All | - |
````

Do not duplicate agent/skill rosters, full directory structures, or granular execution rules here. Manage those in the orchestrator skill, `.claude/agents/`, and `.claude/skills/`. Keep only invocation triggers and change history in `CLAUDE.md`.

#### 5-5. Handling Follow-up Requests

The orchestrator must handle iterative refinements as well as fresh executions:

1. In the `description`, include follow-up phrases such as `re-run`, `resume`, `update`, `modify`, `refine`, `re-do only {subtask} of {domain}`, `based on previous results`, and `improve output`.
2. In Phase 0 of the orchestrator, inspect existing execution context:
   - If `_workspace/` exists and the request asks to modify a subset of tasks, re-run only that specific phase or agent.
   - If `_workspace/` exists but fresh input was provided, archive the existing directory with a timestamp and execute anew.
   - If `_workspace/` does not exist, execute from the beginning.
   - If a previous workflow `runId` exists, resume via `resumeFromRunId`. Unchanged `agent()` calls will use cached results.
3. In agent definitions, document re-invocation handling: read previous output files if available, and modify only the parts specified by user feedback.

### Phase 6: Verification and Testing

Verify the generated harness. Follow detailed methods in `references/skill-testing-guide.md`.

#### 6-1. File and Reference Verification

- Confirm all agent files reside in their correct directories.
- Confirm skills have valid YAML frontmatter containing `name` and `description`.
- Confirm cross-referenced agent names match exactly.
- Confirm no command files were created under `.claude/commands/`.
- Confirm no v1 legacy references (`TeamCreate`, `TeamDelete`, `team_name`, experimental flags) remain in deliverables.

#### 6-2. Mode-Specific Verification

- **Workflow Orchestration**: Confirm `meta` contains only literal values without dynamic calls (`Date.now()`, `Math.random()`), `parallel()` is used only when awaiting all results is necessary, `.filter(Boolean)` is applied, and `phase()` titles match `meta.phases`.
- **Persistent Agent Collaboration**: Verify `SendMessage` sender/receiver paths, task dependencies, and agent counts.
- **Sub-agent Delegation**: Confirm inputs and outputs chain correctly between agents, parallel dispatches are batched in a single turn, and results are gathered without omissions.
- **Hybrid Mode**: Verify execution mode annotations per phase and ensure data flow continuity across mode boundaries.

#### 6-3. Skill Execution Testing

1. Author 2–3 concrete test prompts per skill reflecting realistic user requests.
2. Run with-skill and baseline runs in parallel to benchmark output improvements. If recurring comparisons are needed, encode the A/B evaluation as a workflow.
3. Combine qualitative evaluation with quantitative assertions. If objective metrics cannot be established, rely on human judgment.
4. When issues arise, generalize fixes into principles rather than adding rules for isolated edge cases. Iterate until marginal gains diminish.
5. If multiple agents repeatedly author duplicate logic, extract it into `scripts/`.

#### 6-4. Trigger Condition Verification

1. Author 10 positive test prompts spanning varied tones and degrees of explicitness that should trigger the skill.
2. Author 10 near-miss prompts that resemble the topic but should route to other skills or general tools.

Unrelated prompts fail to test boundary discrimination. For instance, when testing an image generation skill, `Extract the chart from this Excel file as a PNG` is a better near-miss than `Write a Fibonacci function`, because while the output is an image, a spreadsheet tool is more appropriate. Also verify that triggers do not collide with existing skills.

#### 6-5. Dry Run

- Verify logical ordering of phases in the orchestrator.
- Verify data passing pathways have no broken links.
- Confirm every agent's input matches the preceding phase's output format.
- Confirm fallback recovery procedures can be executed upon errors.

#### 6-6. Documenting Test Scenarios

Create a `## Test Scenarios` section in the orchestrator skill documenting at least one happy path and one error/recovery flow.

### Phase 7: Operations, Maintenance, and Evolution

A harness is not a one-and-done artifact.

Retrospecting execution results and incorporating feedback is handled by the `harness:evolve` skill. When the user requests `harness retrospective`, `evolve harness`, or `apply feedback`, invoke `harness:evolve`. That skill analyzes deltas between initial and current state, generalizes feedback, updates agents, skills, and orchestrators, and logs changes in `CLAUDE.md`.

This `harness` skill directly handles routine maintenance and operations:

1. **Current State Audit**: Compare `.claude/agents/`, `.claude/skills/`, and orchestrator configurations, generate an inconsistency list, and notify the user.
2. **Incremental Modification**: Apply one change at a time and immediately run verification.
3. **Change History Logging**: Record date, change summary, target, and rationale in `CLAUDE.md`.
4. **Change Verification**: Verify file structures. If invocation triggers were modified, test triggers. For major changes, execute dry runs and verify that `CLAUDE.md` accurately matches the filesystem.

Propose evolving via `harness:evolve` under the following conditions:

- Identical feedback has emerged 2 or more times.
- Agents repeatedly fail due to the same root cause.
- The user consistently performs tasks manually, bypassing the orchestrator.

## Deliverable Checklist

- [ ] Created definition files in `project/.claude/agents/` for all reusable custom agent types (excluding one-off tasks using built-in types).
- [ ] Created necessary `SKILL.md` and reference documents in `project/.claude/skills/`.
- [ ] Defined data passing methods, error handling, and test scenarios in a single orchestrator skill.
- [ ] Explicitly declared execution mode (Workflow Orchestration, Persistent Agents, or Sub-agents), annotating each phase in hybrid modes.
- [ ] Selected agent `model:` tiers based on complexity, duration, autonomy, and latency, documenting rationale in comments. Avoided blanket high-tier pins.
- [ ] Cleaned out all v1 artifacts (`TeamCreate`, `TeamDelete`, experimental flags).
- [ ] Included `.filter(Boolean)` in workflow scripts, restricted `meta` to literals, and reserved `parallel()` only for cases requiring all results.
- [ ] Created no command files in `.claude/commands/`.
- [ ] Verified lack of role overlap with existing agents and skills before creation. Reused or extended existing components where appropriate.
- [ ] Authored agent definitions, skills, orchestrators, and `CLAUDE.md` records in the user's conversational language (or the language of existing harness files).
- [ ] Documented specific tasks, invocation triggers, and follow-up query phrases in skill `description` fields.
- [ ] Kept `SKILL.md` bodies under 500 lines, extracting detailed references into `references/`.
- [ ] Tested execution with 2–3 realistic request scenarios.
- [ ] Validated invocation conditions with positive trigger prompts and negative near-miss boundary cases.
- [ ] Recorded only invocation triggers and change history in `CLAUDE.md`.
- [ ] Handled initial runs, follow-ups, and partial re-runs in Phase 0 of the orchestrator, including workflow resume options.

## Reference Documents

- **Execution Modes Deep Dive**: `references/execution-modes.md`
- **Model Selection Guide**: `references/model-selection-guide.md`
- **Team Patterns & Agent Definitions**: `references/team-patterns.md`
- **Practical Team Examples**: `references/team-examples.md`
- **Workflow Script Recipes & Gotchas**: `references/workflow-recipes.md`
- **Orchestrator Templates**: `references/orchestrator-template.md`
- **Skill Writing Guide**: `references/skill-writing-guide.md`
- **Skill Testing Guide**: `references/skill-testing-guide.md`
- **QA Agent Guide**: `references/qa-agent-guide.md`

# Team Patterns — 6 Core Patterns, Quality Verification, and Agent Definitions

This document supplements Phase 2-2 and Phase 3 of `SKILL.md`. It explains the recommended execution modes in v2 for each pattern. For comprehensive mode details, see `execution-modes.md`.

---

## Table of Contents

1. [6 Core Patterns](#1-6-core-patterns)
2. [Quality Verification Patterns (v2 Additions)](#2-quality-verification-patterns-v2-additions)
3. [Composite Patterns](#3-composite-patterns)
4. [Selecting Agent Types](#4-selecting-agent-types)
5. [Agent Definition Structure](#5-agent-definition-structure)
6. [Agent Separation Criteria](#6-agent-separation-criteria)
7. [Agent Reuse Design](#7-agent-reuse-design)
8. [Differentiating Skills and Agents](#8-differentiating-skills-and-agents)

---

## 1. 6 Core Patterns

### 1-1. Pipeline

Connects tasks in a predetermined sequential order. The output of the preceding agent becomes the input of the succeeding agent.

```
[Analysis] → [Design] → [Implementation] → [Verification]
```

**Best Suited For:** When each stage strictly depends on the deliverable of the preceding stage.

**Example:** Novel writing — Worldbuilding → Character Development → Plot Outline → Chapter Drafting → Editing.

**Caveat:** A bottleneck in any single stage delays the entire workflow. Design stages to be as self-contained and modular as possible.

**Recommended v2 Execution Mode:** If stages and items can be enumerated in advance, **Workflow Orchestration** using `pipeline()` is strongly recommended. Each item transitions to the next stage the moment its current stage completes. Because there is no need to wait for all items in a stage to finish, overall latency is minimized. If human review is required between stages, dispatch subagents sequentially.

### 1-2. Fan-out / Fan-in

Processes independent tasks concurrently and aggregates the results into a unified output.

```
         ┌→ [Specialist A] ─┐
[Divide] → ├→ [Specialist B] ─┼→ [Integrate]
         └→ [Specialist C] ─┘
```

**Best Suited For:** Analyzing the same input across multiple perspectives or domains simultaneously.

**Example:** Comprehensive research — Investigating official sources, news coverage, community discussions, and historical background in parallel, then synthesizing into a single report.

**Caveat:** The integration logic determines the final deliverable's quality.

**Recommended v2 Execution Mode:** Prioritize **Workflow Orchestration**. Define the perspective list as an array, pass it through `pipeline()`, enforce structured output with `schema`, and merge results in code. Use `parallel()` only when all results must be gathered prior to global deduplication (which waits for all parallel dispatches to finish before proceeding). For small investigations (2–4 agents) where researchers must share findings in real time, Persistent Agent Collaboration is also well-suited.

### 1-3. Expert Pool

Selects and dispatches only the specialist relevant to the incoming input and context.

```
[Classifier] → { Specialist A | Specialist B | Specialist C }
```

**Best Suited For:** When different input categories require completely different handling procedures.

**Example:** Code review — Dispatching only the relevant specialist (e.g., Security, Performance, or Architecture) based on the nature of the pull request.

**Caveat:** If the classifier agent miscategorizes the input, the wrong specialist will be invoked.

**Recommended v2 Execution Mode:** Use **Sub-agent Delegation** to invoke the selected specialist as a one-off call. If follow-up dialogue with that specialist is required, assign a `name` to transition into Persistent Agent Collaboration.

### 1-4. Producer-Reviewer

A generation agent and a verification agent collaborate as an iterative pair.

```
[Producer] → [Reviewer] → (If issues detected) → [Producer] Re-run
```

**Best Suited For:** When deliverable accuracy must be strictly verified against objective criteria.

**Caveat:** Cap retries at 2–3 rounds to prevent infinite review loops.

**Recommended v2 Execution Mode:** If verification criteria can be expressed in code, use **Workflow Orchestration**. Execute parallel verification for each finding using the Adversarial Verification pattern described below. If subjective judgment must be negotiated, launch the author and reviewer as a named **persistent agent pair** and exchange feedback via `SendMessage`. Because the author retains previous conversational context, it can address targeted critiques precisely.

### 1-5. Supervisor

A central agent monitors state and dynamically reallocates tasks based on progress.

```
            ┌→ [Worker A]
[Supervisor] ─┼→ [Worker B]    ← Dynamically assigned based on progress
            └→ [Worker C]
```

**Best Suited For:** When workload volume is unpredictable, or task allocation must be determined dynamically during runtime.

**Difference from Fan-out:** Fan-out pre-determines task allocation before execution begins. A supervisor dynamically redistributes work based on ongoing progress.

**Caveat:** Decomposing tasks too granularly forces frequent supervisor reassignments, creating a bottleneck.

**Recommended v2 Execution Mode:** Use **Persistent Agent Collaboration**. The main leader agent registers tasks via `TaskCreate`, receives completion notifications from workers, and reassigns tasks via `SendMessage` and `TaskUpdate`. If the task list can be finalized upfront and allocation logic can be scripted, switch to Workflow Orchestration, as a supervisor's subjective per-turn judgment is redundant in deterministic flows.

### 1-6. Hierarchical Delegation

Delegated agents further delegate subtasks to subordinate workers. Used to decompose complex problems across multiple tiers.

```
[Director] → [Team Lead A] → [Workers A1, A2]
           → [Team Lead B] → [Worker B1]
```

**Best Suited For:** Problems that decompose naturally into macro-level and micro-level operations.

**Caveat:** Nesting deeper than two levels degrades execution latency and risks losing crucial context. **Restrict delegation to a maximum of two tiers.**

**Recommended v2 Execution Mode:** Orchestrate via nested workflows. A parent workflow dispatches child workflows via `workflow(nameOrRef, args)`. Because nested workflow calls support only one level of nesting, deep runaway hierarchies are prevented. Alternatively, flatten the hierarchy into `phase()` steps within a single workflow.

## 2. Quality Verification Patterns (v2 Additions)

Workflow verification patterns designed to filter out plausible-but-wrong outputs when deliverable accuracy is critical.

| Pattern | Structure | When to Use |
|---|---|---|
| **Adversarial Verification** | Dispatch N independent adversarial verification agents per discovered finding. Confirm an item only if a strict majority evaluate it as `confirmed` with sufficient evidence. | When findings and claims must be indisputably reliable (audits, security reviews, investigations). |
| **Perspective Diversity Verification** | Rather than giving N adversarial agents identical prompts, assign distinct angles (e.g., accuracy, security, reproducibility). | When errors stem from varied failure modes. Segmenting criteria uncovers bugs missed by monolithic repeated reviews. |
| **Judge Panel** | Independently generate N alternative drafts in parallel using differing approaches. A judge panel evaluates them, and the final deliverable synthesizes the strongest elements. | Open-ended architectural or strategic designs with multiple viable solutions. Comparing diverse alternatives is far superior to iteratively revising a single draft. |
| **Loop-Until-Dry** | Repeat searches until newly discovered items equal 0 for K consecutive rounds. Track rejected items in a `seen` set to prevent duplicate proposals. | Open-ended discovery tasks where the total number of issues is unknown (bugs, edge cases, vulnerability scanning). Fixed iterations risk truncating discovery prematurely. |
| **Multi-Criteria Sweep** | Dispatch parallel agents sweeping across distinct dimensions (e.g., container, content, entity, timeline). | When a single search criterion cannot comprehensively surface all target items. |
| **Completeness Critic** | Conclude with a dedicated agent tasked solely with asking: "Which search paths were omitted? Which claims lack verification? Which files were left uninspected?" | Immediately prior to final synthesis. Newly identified omissions are queued into follow-up sweeps. |
| **Disclosure of Excluded Count** | When scoping via `top-N` or sampling, always record the count of omitted/excluded items via `log()`. | All fan-out executions. Concealing excluded volume misleads users into assuming exhaustive sweeps were completed. |

**Scale Calibration Principle:** Match the execution scale to the user's requested audit depth. A quick request ("Check if there are bugs") warrants lean discovery and a single verifier. A rigorous request ("Perform an exhaustive security audit") warrants multiple discoverers, 3–5 adversarial verifiers per finding, and an explicit synthesis phase. When ambiguous, perform thorough reviews for audits and investigations, and lean runs for quick checks.

> Implementation recipes are available in `workflow-recipes.md`.

## 3. Composite Patterns

Production harnesses combine multiple patterns into composite topologies:

| Composite Pattern | Structure | Example |
|---|---|---|
| **Fan-out/Fan-in + Producer-Reviewer** | Generate multiple drafts concurrently, then verify each independently | Multilingual localization — Translate into 4 languages in parallel, followed by native-speaker verification per language |
| **Pipeline + Fan-out/Fan-in** | Process sequential phases, with internal stages executing in parallel | Analysis (Sequential) → Implementation (Parallel across modules) → Integration Testing (Sequential) |
| **Supervisor + Expert Pool** | A supervisor routes tasks to dynamically selected domain specialists | Customer support triage — Supervisor classifies inquiries and assigns them to billing, technical, or account specialists |
| **Explore + Adversarial Verify + Synthesize** | Multi-criteria search → Per-item adversarial verification → Completeness audit → Final report | Comprehensive codebase security audit, deep forensic investigation |

In composite patterns, select the appropriate execution mode for each phase. This is known as a hybrid mode. The core decision criterion is whether control flow for that phase can be pre-scripted deterministically in code.

## 4. Selecting Agent Types

When invoking agents, specify the type via `subagent_type` in the `Agent` tool or `agentType` in `Workflow`.

### Built-in Types

| Type | Available Tools | Best Suited For |
|---|---|---|
| `general-purpose` | All tools (including WebSearch, WebFetch) | Web research, general execution, file modifications |
| `Explore` | Read-only (excludes Edit, Write) | Locating code paths in a codebase. Do not use for audits or code reviews. |
| `Plan` | Read-only (excludes Edit, Write) | Architecture design, implementation planning |

### Custom Types

Define custom types under `.claude/agents/{name}.md`. Invoke via `subagent_type: "{name}"` or `agentType: "{name}"`. Declare allowed tools and model tier in the YAML frontmatter.

### Selection Decision Table

| Situation | Recommendation | Rationale |
|---|---|---|
| Complex role reused across multiple sessions | **Custom Type** | Consistently enforces specialist role and principles via versioned files. |
| Simple one-off research adequately guided by prompt alone | `general-purpose` + Detailed Prompt | Uses built-in type directly without unnecessary file overhead. |
| Locating code paths without modifying files | `Explore` | Prevents accidental file edits via lack of mutation tools. |
| Drafting architecture or plans | `Plan` | Focuses strictly on analysis without mutating code. |
| Simple one-off file editing | `general-purpose` + Detailed Prompt | Handled directly via standard built-in tooling. |
| Implementation requiring domain rules or recurring reuse | **Custom Type** | Grants full necessary toolset while enforcing domain invariants. |
| Parallel editing across multiple files | Custom Type + `isolation: 'worktree'` | Prevents file mutation race conditions across parallel agents. Incur worktree overhead only when parallel writes occur. |

**Principle:** Create custom types in `.claude/agents/{name}.md` for reusable specialist roles. Do not create separate definition files for one-off tasks using built-in types.

**Model Policy:** Select model tiers based on task complexity, duration, required autonomy, and latency needs. Use **fable** for long-horizon autonomous planning, execution, and agent orchestration. Use **opus** for architecture, code generation, complex analysis, cross-verification, and creative writing. Use **sonnet** for structured tasks requiring minimal subjective judgment (log parsing, format conversion, static checks, deployment scripts, simple collection). When uncertain, default to Sonnet. Document rationale in comments rather than blanket-assigning high tiers. See `model-selection-guide.md` for complete guidelines.

## 5. Agent Definition Structure

```markdown
---
name: agent-name
description: "Describe the role in 1-2 sentences and enumerate key trigger phrases."
# tools: Read, Grep, Glob, Bash        ← Optional: Restrict tools (e.g., read-only reviewers)
#                                         Provide both Write and Edit for agents that modify deliverables
# model: sonnet                         ← Select per task profile (model-selection-guide.md), document rationale in comments
---

# Agent Name — One-line Role Summary

You are a specialist in [role] within [domain].

## Core Roles
1. Role 1
2. Role 2

## Working Principles
- Principle 1 (Include the rationale. Understanding the reason ensures consistent judgment in edge cases.)
- Principle 2

## Input / Output Rules
- Input: [Source and format — file paths, args, prior stage deliverables]
- Output: [Destination and format — _workspace/ paths or structured return objects]
- Format: [File format, schema, structure]

## Communication Rules (Persistent Agent Collaboration)
- Initial Check-in: Report active available tools to the leader (to confirm tools declared in `tools` were loaded)
- Message Ingestion: [Who sends messages and of what type]
- Message Dispatch: [Who receives messages and under what conditions]
- Shared Tasks: [What task types are registered or updated]

## Re-invocation Protocol
- Inspect previous artifacts if available, and incorporate required improvements.
- Modify only the requested sections when user feedback is provided.

## Error Handling
- [Action on failure]
- [Action on missing or ambiguous inputs]

## Collaboration Protocols
- Relationships with neighboring agents
```

**Workflow-Specific Agents:** Replace "Communication Rules" with a "Structured Output" section. Specify the exact JSON return schema, and state explicitly that the final output must be **raw structured data**, not conversational commentary addressed to the user.

**Tool Restriction Gotchas:**

- Always grant both `Write` and `Edit` to agents tasked with modifying deliverables. Granting only `Write` forces agents to rewrite entire files even for single-line diffs. On large files, agents often give up or write to secondary scratchpads, leaving the original deliverable unpatched.
- Lazy-loaded tools like `TaskCreate` and `TaskUpdate` may occasionally not be injected at runtime even when listed in `tools` (observed in Claude Code 2.1.226; behavior may vary by environment). Because `SendMessage` is typically granted, the boundary is not always explicit. Instruct persistent agents to report their active tool list upon their initial check-in. If expected tools are missing, have the orchestrator manage task updates directly or remove restrictions from the `tools` declaration.

## 6. Agent Separation Criteria

| Criterion | Separate into Distinct Agents | Consolidate into Single Agent |
|---|---|---|
| Expertise | Specialized domains differ significantly | Functional domains heavily overlap |
| Concurrency | Tasks can execute independently in parallel | Tasks must be processed strictly sequentially |
| Context Volume | High context footprint per individual domain | Low context footprint with concise tasks |
| Reusability | Component can be reused by other teams/workflows | Specific to this single team only |

## 7. Agent Reuse Design

Before creating a new agent, cross-reference roles with existing agents in `.claude/agents/`. Successive harness expansions can cause duplicate agents with different names to proliferate.

| Relationship with Existing Agent | Action |
|---|---|
| Existing agent fully subsumes the new role | Do not create a new agent; invoke the existing one. |
| Partial overlap; generalizing existing agent covers new role | Generalize and extend the existing agent. |
| Partial overlap, but domain specialization was deliberate | Create a dedicated new agent. |
| Role and domain are entirely distinct | Create a new agent. |

**Principle:** Single-responsibility agents maximize reusability and minimize redundancy. If extending an agent causes it to handle multiple disparate responsibilities, evaluate whether to split it instead.

**Extending Existing Agents:** Expanding an agent may alter the expectations of existing orchestrators that call it. Before extending, identify which orchestrators invoke the agent, reflect the expanded scope in its `description`, and verify via dry runs (Phase 6-5 of `SKILL.md`) that existing workflows remain unaffected.

## 8. Differentiating Skills and Agents

| Attribute | Skill | Agent |
|---|---|---|
| Definition | Procedural workflows, rules, and toolsets | Specialist identity, role, and behavioral principles |
| Directory | `.claude/skills/` | `.claude/agents/` |
| Invocation Trigger | User requests matching triggers in `description` | Dispatched directly via `Agent` or `Workflow` tools |
| Concern | "How to do it" | "Who does it" |

### Methods for Connecting Skills to Agents

| Method | Implementation | Best Suited For |
|---|---|---|
| **Skill Tool Call** | Declare `Invoke /skill-name via Skill tool` in agent prompt | Standalone workflows that can also be called directly by users |
| **Inline Prompt** | Embed skill instructions directly into agent definition body | Concise rules under 50 lines used exclusively by that single agent |
| **Reference File Ingestion** | Read files from a skill's `references/` via `Read` on demand | Extensive domain rules needed only under specific edge conditions |

Skills shared across multiple agents should be invoked via the `Skill` tool. Compact, agent-specific rules should be inlined directly, and extensive references should be loaded on demand.

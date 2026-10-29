---
name: evolve
description: "Harness evolution skill. Collects and generalizes feedback on the execution results of an active harness, applies it to agents/skills/orchestrators, captures the delta against the initial configuration, and updates the change history. MUST use this skill for improvement requests based on execution experience with an existing harness, such as 'harness retrospective', 'evolve harness', 'apply harness feedback', 'improve harness', 'the result was disappointing, fix the harness', 'incorporate this feedback into the harness', 'organize harness lessons', etc. Building a new harness, structural redesign, or adding agents is handled by the harness skill."
---

# Harness Evolve — Harness Evolution Mechanism

A harness is not a static fixture, but an evolving system. This skill captures the delta of "what worked and what didn't" and feeds it back into the harness so that subsequent executions are measurably better.

```
Initial Harness ──▶ Real Project Usage ──▶ Current Harness
                                                │
                                                ▼ (Capture delta with evolve)
                                          Generalize feedback → Apply to agents, skills, orchestrator
                                                │
                                                ▼
                                          Update change history → Next execution starts from a better draft
```

## Workflow

### Phase 1: Delta Collection

1. Read `.claude/agents/`, `.claude/skills/`, and `CLAUDE.md` (change history table).
2. If it is a git repository, inspect the commit history of harness files (`git log --oneline -- .claude/ CLAUDE.md`) to understand what changed, when, and why compared to the initial setup.
3. If `_workspace/` exists, scan the intermediate artifacts of recent executions to identify actual execution traces:
   - Do the artifacts actually exist at the paths defined by the orchestrator? (If not, the workflow was bypassed or contains dead code).
   - Does the artifact quality adhere to the format and standards specified by the skills?
4. Solicit user feedback (skip if feedback has already been provided):
   - "Are there any areas for improvement in the results?"
   - "Is there anything you would like to change about the agent configuration or workflow?"
   - If there is no feedback, do not force it. However, if the observation signals below are present, proactively propose improvements.

**Observation-Based Evolution Signals (propose even without explicit user feedback):**
- Evidence of identical modification requests repeated 2 or more times.
- Patterns where agents repeatedly fail or retry.
- Evidence of the user bypassing the orchestrator to perform tasks manually (suspected orchestrator trigger failure → candidate for description expansion).
- v1 legacy artifacts (`TeamCreate`, `TeamDelete`, experimental flags) remain in the orchestrator → guide migration using the harness skill's migration procedure.

### Phase 2: Feedback Classification & Target Mapping

| Feedback Type | Modification Target | Example |
|---|---|---|
| Deliverable Quality | Target agent's skill | "The analysis is too superficial" → Add depth criteria to skill |
| Agent Role | Agent definition `.md` | "Security review is also needed" → Guide agent addition via harness skill |
| Workflow Sequence | Orchestrator skill | "Verification should happen first" → Reorder Phases |
| Team Composition | Orchestrator + Agents | "These two can probably be merged" → Merge agents |
| Missing Trigger | Skill description | "It doesn't trigger with this phrase" → Expand description |
| Inappropriate Execution Mode | Orchestrator | "It's the same fan-out every time, but it's slow" → Switch to workflow mode |
| Scale / Cost | Orchestrator | "Consumes too many tokens" → Reduce default scale, integrate budget checks |

**Scope Determination:** If new agent creation/deletion or architectural redesign is required, do not handle it directly in this skill; instead, guide the user to the harness skill (Phase 0: existing configuration expansion procedure). Evolve focuses on the **tuning and adjustment of existing configurations**.

### Phase 3: Generalization & Application

1. **Generalize the feedback** — Narrow modifications tailored only to a specific case cause overfitting. Rather than changing "The introduction was too long in this report" into "Keep the introduction under 10% of total length", identify the underlying principle (e.g., absence of length allocation guidelines in the skill) and fix it at the principle level.
2. **Document the Why alongside the change** — Provide rationale alongside modified instructions. Knowing the reason enables agents to make sound judgements even in edge cases.
3. Apply changes one at a time, running Phase 4 immediately after each change.
4. **Prevent regressions:** If a modification reverses a previous edit documented in the change history, inform the user of the conflict and request confirmation. (e.g., if a previous change reduced length because it was "too long", and now it is flagged as "too short", the proper resolution is finding a criterion that satisfies both constraints).
5. **Preserve the language of existing files** — Write statements reflected in agents, skills, orchestrators, and `CLAUDE.md` in the language already used in the target files. Do not mix other languages into a harness just because this skill document is in English.

### Phase 4: Change History Update & Verification

1. Record in the **Change History** table of `CLAUDE.md`:

```markdown
**Change History:**
| Date | Change Summary | Target | Reason |
|---|---|---|---|
| 2026-07-19 | Add tone guide | skills/content-creator | Feedback: "Too rigid/formal" |
```

2. Verify the structure of modified files (frontmatter, reference consistency).
3. If description was modified, verify triggers (at least 3 should-trigger and 3 near-miss cases each).
4. Perform a final check of consistency between `CLAUDE.md` and actual files.

### Phase 5: Evolution Report

Report to the user:
- Summary of captured deltas (initial configuration → current).
- Changes applied this time and their generalization rationale.
- Feedback decided not to be implemented and the reason (if any).
- Expected improvements in subsequent executions.

## Principles

- **Deltas are assets** — As change history accumulates, future harness builds in the same domain begin from a draft much closer to production readiness. Never delete history.
- **No changes without generalization** — Adding a rule 1:1 for every single incident turns skills into an unmanageable pile of rules. Condense into principles.
- **One at a time** — Applying multiple changes at once makes it impossible to isolate which change produced which effect.

# Practical Team Examples

Each example illustrates the decision criteria for selecting an execution mode tailored to the task and demonstrates the baseline syntax of Harness v2. For formal mode definitions, see `execution-modes.md`; for workflow script authoring recipes, see `workflow-recipes.md`.

---

## Example 1: Comprehensive Research Team — Workflow Orchestration

**Topology:** Fan-out / Fan-in + Adversarial Verification

**Selection Rationale:** The research axes can be enumerated in advance, and the verification procedure for each claim can be scripted deterministically in code.

```
[Main] Pre-investigation: Lock down research axes (official docs, news media, community discourse, background)
     → Workflow(script, args: {axes, topic, ws})
         phase 'Research': pipeline(axes, axis => agent(..., {schema: FINDINGS}))
         phase 'Verification': Dispatch adversarial verification agent per claim → Pass only claims confirmed by majority
         phase 'Synthesis': Dispatch 1 completeness critic → Perform supplementary research if gaps exist
     → Main compiles final integrated report from returned structured data
```

Define research principles and structured output schemas in `.claude/agents/researcher.md`, and skepticism-first verification principles in `.claude/agents/fact-checker.md`. Specify both agents via `agentType` in the workflow.

Never arbitrarily discard contradictory information; record competing viewpoints alongside their source citations within the return schema.

## Example 2: Sci-Fi Novel Writing Team — Persistent Agent-centric Hybrid Mode

**Topology:** Pipeline + Producer-Reviewer

**Selection Rationale:** Worldbuilding, character profiles, and plot arcs must be adjusted in real time to avoid continuity breaks, requiring specialists to retain historical conversation context. In contrast, reviewers evaluate independent perspectives and only need to deliver verdicts, making one-shot subagents sufficient.

```
Stage 1 (Persistent Agents): Launch Agent(name: "worldbuilder") + Agent(name: "character-designer")
                + Agent(name: "plot-architect") in parallel
                → TaskCreate(Worldbuilding, characters, plot; explicit inter-dependencies declared)
                → Leader mediates: worldbuilder finalizes societal structure
                  → Relay to character-designer via SendMessage
                  → If character professions conflict with world lore, request
                    adjustments from worldbuilder via SendMessage
                  Because historical context is retained, instructions can be targeted:
                  "In the caste hierarchy established earlier, revise only the merchant guild."
Stage 2 (Sub-agent): Dispatch prose-stylist once → Draft chapter by reading 3 artifacts from _workspace/
Stage 3 (Sub-agents in Parallel): science-consultant + continuity-manager review independently
Stage 4 (Persistent Agents): A one-shot prose-stylist cannot receive SendMessage calls.
                  If launched with a name in Stage 2, instructions can be sent to incorporate review feedback.
                  If repeated revisions are anticipated, assign a name from the outset.
```

Assign a `name` at initial dispatch to any agent that may receive follow-up revision instructions. Agents invoked as one-shot calls cannot retain or resume conversational context.

## Example 3: Comprehensive Code Review — Workflow Orchestration

**Topology:** Fan-out / Fan-in + Adversarial Verification

**Selection Rationale:** Review perspectives (security, performance, architecture, testability) can be determined upfront, and verifying discovered issues can be expressed programmatically in code.

```javascript
// Perspective review → Adversarial verification per discovered finding
// Does not wait for all reviews to finish: as soon as security review finishes,
// findings in the security domain are verified immediately even while performance review is running.
const FINDINGS = { type: 'object', required: ['findings'], properties: {
  findings: { type: 'array', items: { type: 'object',
    required: ['title', 'file', 'evidence'], properties: {
      title: { type: 'string' }, file: { type: 'string' }, evidence: { type: 'string' } } } } } }
const VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }
const results = await pipeline(
  [
    { key: 'security', prompt: 'Review codebase from a security perspective.' },
    { key: 'perf', prompt: 'Review codebase from a performance perspective.' },
    { key: 'arch', prompt: 'Review codebase from an architectural perspective.' },
    { key: 'test', prompt: 'Review codebase from a testability perspective.' },
  ],
  d => agent(d.prompt, { phase: 'Review', schema: FINDINGS }),
  r => parallel((r?.findings ?? []).map(f => () =>
    agent(`Verify the following finding. Evaluate as confirmed if supported by rigorous evidence, refuted if clearly disproven, or uncertain if ambiguous: ${JSON.stringify(f)}`,
      { phase: 'Verification', schema: VERDICT })
      .then(verdict => ({ ...f, verdict }))))
)
const confirmed = results.flat().filter(Boolean)
  .filter(f => f.verdict?.status === 'confirmed')
```

In v1, reviewers operated as a persistent team sharing findings via `SendMessage`. When each finding can be verified self-sufficiently (as shown above), embed sufficient context in the prompt and verify immediately. If cross-perspective comparison is required, aggregate all review findings before passing through a synchronization barrier. Reserve persistent agents for scenarios requiring real-time debate (e.g., negotiating system architecture tradeoffs).

## Example 4: Large-scale Code Migration — Persistent Supervisor Collaboration or Workflow Orchestration

**Topology:** Fan-out / Fan-in if batches can be pre-partitioned; Supervisor if batches must be dynamically split during runtime.

**Selection Rationale:** Select execution mode based on whether migration batches can be partitioned in advance.

If batches can be determined upfront, use Workflow:

```javascript
const MIGRATION_RESULT = { type: 'object', required: ['worktreePath', 'changedFiles'], properties: {
  worktreePath: { type: 'string', minLength: 1 },
  changedFiles: { type: 'array', minItems: 1, uniqueItems: true,
    items: { type: 'string', minLength: 1 } } } }
const MIGRATION_VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }
const migrated = await pipeline(args.batches,   // Lock down batches after estimating complexity during pre-investigation
  b => agent(`Migrate the following batch of files, returning the absolute path to the isolated worktree and list of changed files: ${b.files.join(', ')}`,
    { agentType: 'migrator', isolation: 'worktree', schema: MIGRATION_RESULT }),
  (r, b) => r && agent(
    `In isolated worktree ${r.worktreePath}, cross-reference migration targets against changed files to detect omissions or syntax errors. Original targets: ${b.files.join(', ')}. Actual changes: ${r.changedFiles.join(', ')}`,
    { agentType: 'qa-inspector', schema: MIGRATION_VERDICT })
    .then(verdict => ({ ...r, verdict })))
const confirmed = migrated.filter(Boolean)
  .filter(r => r.verdict?.status === 'confirmed')
return { confirmed }
```

Always pass the isolated worktree path containing migration results to the adversarial verification agent. When the workflow completes, the main agent inspects `worktreePath` for each `confirmed` result, selectively cherry-picking or merging changes into the target branch. Resolve any merge conflicts prior to merging subsequent worktrees, and proceed only after integration tests pass. Do not merge `refuted` or `uncertain` results; log omitted rationale in the report.

If tasks must be dynamically split during execution based on runtime progress, use Persistent Agents:

```
Leader registers batches via TaskCreate (including depends_on)
→ Launch Agent(name: "migrator-1"), Agent(name: "migrator-2"), Agent(name: "migrator-3") in parallel
→ Inspect results as completion notifications arrive
→ For failed tasks, diagnose cause via SendMessage and reallocate via TaskUpdate
→ Run integration tests once all batches complete
```

## Example 5: Webtoon Production — Persistent Producer and One-shot Reviewer

**Topology:** Producer-Reviewer

**Selection Rationale:** Only one creator and one reviewer are required. Review feedback is returned to the creator for at most two revision rounds, making a lightweight hybrid mode ideal.

```
Stage 1: Agent(name: "artist") → Generate panels → _workspace/panels/
Stage 2: Dispatch Agent(subagent_type: "webtoon-reviewer", prompt: "Review panel drafts") once → Verdict: PASS / FIX / REDO
       → _workspace/review_report.md
Stage 3: Instruct regeneration only for panels receiving REDO via SendMessage({to: "artist"})
       Limit to 2 revision loops. Because artist retains previous context,
       instructions can be narrowly scoped: "Revise only the composition of Panel 3."
Retry Policy: If panels fail after 2 revisions, flag unresolved state and root cause to the user.
              If over 50% of panels receive REDO, suggest prompt refinement to the user.
```

---

## Deliverable Storage Conventions

- **Agent Definitions:** Author under `project/.claude/agents/{name}.md`. Include core role, work principles, input/output rules, re-invocation protocol, error handling, and collaboration rules. Add communication rules for persistent agents, and structured output schemas for workflow agents.
- **Skills:** Author under `project/.claude/skills/{name}/SKILL.md`, supplemented by `references/` and `scripts/` as needed.
- **Orchestrator:** Explicitly declare the execution mode. Use templates from `orchestrator-template.md`.
- **Intermediate Artifacts:** Store using the convention `_workspace/{phase}_{agent}_{artifact}.{ext}`, preserving artifacts even after verification completes.

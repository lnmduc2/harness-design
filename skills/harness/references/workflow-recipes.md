# Workflow Script Recipes — Boilerplate Patterns and Gotchas

This document details script patterns that can be embedded into orchestration skills when designing a harness in Workflow Orchestration mode (Mode A). Scripts are authored in pure JavaScript and passed via the `script` parameter of the `Workflow` tool.

---

## Table of Contents

1. [Baseline Boilerplate](#1-baseline-boilerplate)
2. [Fan-out + Adversarial Verification](#2-fan-out--adversarial-verification)
3. [Judge Panel](#3-judge-panel)
4. [Loop-Until-Dry](#4-loop-until-dry)
5. [Token Budget-Integrated Loops](#5-token-budget-integrated-loops)
6. [Custom Agent Types + Structured Output](#6-custom-agent-types--structured-output)
7. [Gotchas and Verification Checklist](#7-gotchas-and-verification-checklist)

---

## 1. Baseline Boilerplate

A complete, executable script must begin with a `meta` literal block. Phase titles passed to `phase()` calls must match the `title` fields in `meta.phases` exactly.

```javascript
export const meta = {
  name: 'domain-task',
  description: 'One-line description (displayed in the authorization dialog)',
  phases: [
    { title: 'Collection', detail: 'Investigate across perspectives in parallel' },
    { title: 'Verification', detail: 'Adversarially verify evidence per discovered finding' },
  ],
}

phase('Collection')
const raw = await pipeline(args.items, item =>
  agent(`...${item}...`, { label: `Collect:${item}`, phase: 'Collection', schema: COLLECT_SCHEMA }))

phase('Verification')
// ...

return { result }   // Final return value is delivered to the main agent
```

**Principles:**
- **Use `pipeline()` by default; reserve `parallel()` only when a strict synchronization barrier is required.** A synchronization barrier is necessary only when the downstream stage must compare the **entire** batch of previous results against each other (e.g., global deduplication, early termination when aggregate count is zero, or prompts explicitly asking to "compare with all other findings").
- Do not hardcode task items directly into the script. Have the main agent lock down items via pre-investigation and pass them via `args.items`.
- If timestamps are required, pass them via `args.now`. Calling `Date.now()` directly in the script is forbidden.

## 2. Fan-out + Adversarial Verification

Collects findings across distinct perspectives, then verifies each finding individually. Downstream verification does not wait for all perspectives to finish; as soon as a single perspective completes, its findings are verified immediately.

```javascript
export const meta = {
  name: 'review-fanout-verify',
  description: 'Review across perspectives then verify evidence adversarially for each finding',
  phases: [{ title: 'Review' }, { title: 'Verification' }],
}

const FINDINGS = { type: 'object', required: ['findings'], properties: {
  findings: { type: 'array', items: { type: 'object',
    required: ['title', 'file', 'evidence'], properties: {
      title: { type: 'string' }, file: { type: 'string' }, evidence: { type: 'string' } } } } } }
const VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }

const results = await pipeline(
  args.dimensions,   // e.g., [{key:'security', prompt:'...'}, {key:'perf', prompt:'...'}]
  d => agent(d.prompt, { label: `Review:${d.key}`, phase: 'Review', schema: FINDINGS }),
  review => parallel((review?.findings ?? []).map(f => () =>
    agent(`Verify the following finding. Evaluate as confirmed if supported by rigorous evidence, refuted if clearly disproven, or uncertain if ambiguous: ${JSON.stringify(f)}`,
      { label: `Verify:${f.file}`, phase: 'Verification', schema: VERDICT })
      .then(v => ({ ...f, verdict: v }))))
)
const confirmed = results.flat().filter(Boolean)
  .filter(f => f.verdict?.status === 'confirmed')
log(`${confirmed.length} of ${results.flat().filter(Boolean).length} findings passed verification.`)
return { confirmed }
```

To make verification more rigorous, dispatch three adversarial verifiers per finding. Pass an item only if two or more evaluate it as `confirmed` with sufficient evidence. Giving each agent distinct evaluation angles (e.g., accuracy, security, reproducibility) surfaces far more varied issues than repeatedly testing against a single monolithic prompt.

## 3. Judge Panel

Used for open-ended design or planning tasks where multiple valid approaches exist. Generates N independent drafts in parallel, evaluates them via a judge panel, and synthesizes the winning draft with strong elements from alternative proposals.

```javascript
export const meta = {
  name: 'design-judge-panel',
  description: 'Generate N independent drafts, evaluate via judges, and synthesize final draft from highest-scoring proposal',
  phases: [{ title: 'Drafts' }, { title: 'Judging' }, { title: 'Synthesis' }],
}

const angles = args.angles?.length
  ? args.angles
  : ['Prioritizing MVP velocity', 'Prioritizing risk mitigation', 'Prioritizing user experience']
const judgeCount = args.judgeCount ?? 3

phase('Drafts')
const drafts = (await parallel(angles.map(a => () =>
  agent(`Draft an independent proposal from the perspective of ${a}: ${args.brief}`,
    { label: `Draft:${a}`, phase: 'Drafts' }))))
  .filter(Boolean)
log(`Generated ${drafts.length} of ${angles.length} drafts.`)
if (!drafts.length) return { error: 'No drafts completed successfully.', drafts: [] }

phase('Judging')   // Synchronization barrier required: all drafts must be compared simultaneously
const SCORE = { type: 'object', required: ['scores'], properties: {
  scores: { type: 'array', items: { type: 'object',
    required: ['index', 'score', 'strengths'], properties: {
      index: { type: 'integer' }, score: { type: 'number' }, strengths: { type: 'string' } } } } } }
const judged = (await parallel(Array.from({ length: judgeCount }, (_, j) => () =>
  agent(`Score the following ${drafts.length} drafts:\n${drafts.map((d, i) => `[${i}] ${d}`).join('\n---\n')}`,
    { label: `Judge:${j}`, phase: 'Judging', schema: SCORE })))).filter(Boolean)
log(`Received scores from ${judged.length} of ${judgeCount} judges.`)
if (!judged.length) return { error: 'No judging evaluations completed.', drafts }

phase('Synthesis')
const ranked = drafts.map((_, i) => {
  const scores = judged.flatMap(r =>
    r.scores.filter(x => x.index === i).map(x => x.score)).filter(Number.isFinite)
  return scores.length
    ? { index: i, average: scores.reduce((sum, score) => sum + score, 0) / scores.length }
    : null
}).filter(Boolean)
if (!ranked.length) return { error: 'No valid per-draft scores obtained.', drafts, judged }
const winner = ranked.reduce((best, item) =>
  item.average > best.average ? item : best).index
const finalDraft = await agent(
  `Author the final proposal based on the highest-scoring draft, incorporating the strengths of alternative drafts where beneficial.\nWinning Draft:\n${drafts[winner]}\nJudge Evaluations:\n${JSON.stringify(judged)}`,
  { phase: 'Synthesis' })
if (!finalDraft) {
  log('Failed to author final proposal.')
  return { error: 'Failed to author final proposal.', drafts, judged, winner }
}
return finalDraft
```

## 4. Loop-Until-Dry

Used when the total number of items to discover cannot be predicted in advance. Continues searching until newly discovered items equal 0 for K consecutive rounds.

Sections 4 through 6 provide code snippets intended to fit into the boilerplate from Section 1. When running as standalone scripts, include a `meta` block at the top declaring phase titles and schemas.

```javascript
const finders = args.finders ?? []
const dryLimit = args.dryRuns ?? 2
const findingKey = finding => JSON.stringify([
  finding.file ?? '', finding.title ?? '', finding.evidence ?? ''
])
const seen = new Set(), confirmed = []
if (!finders.length) return { confirmed, error: 'No discovery criteria specified.' }
let dry = 0
while (dry < dryLimit) {
  const found = (await parallel(finders.map(f => () =>
    agent(f.prompt, { phase: 'Discovery', schema: FINDINGS })))).filter(Boolean).flatMap(r => r.findings)
  const fresh = found.filter(b => !seen.has(findingKey(b)))   // Deduplicate against seen, NEVER against confirmed.
  if (!fresh.length) { dry++; continue }
  dry = 0
  fresh.forEach(b => seen.add(findingKey(b)))
  const judged = await parallel(fresh.map(b => () =>
    agent(`Verify the following finding. Evaluate as confirmed if supported by rigorous evidence, refuted if clearly disproven, or uncertain if ambiguous: ${JSON.stringify(b)}`,
      { phase: 'Verification', schema: VERDICT })
      .then(v => ({ b, ok: v?.status === 'confirmed' }))))
  confirmed.push(...judged.filter(Boolean).filter(x => x.ok).map(x => x.b))
  log(`Total confirmed findings: ${confirmed.length}; newly discovered this round: ${fresh.length}.`)
}
return { confirmed }
```

Deduplicating against `confirmed` causes rejected findings to be rediscovered repeatedly, trapping the loop in an infinite cycle. Always track duplicates in a `seen` set containing every item ever surfaced.

## 5. Token Budget-Integrated Loops

In sessions where the user specifies a token budget (e.g., "+500k"), dynamically modulate the scope of exploration based on remaining quota. In unmetered sessions without `budget.total`, `budget.remaining()` returns `Infinity`, so always check `budget.total` in loop conditions.

```javascript
const findings = []
while (budget.total && budget.remaining() > 50_000) {
  const r = await agent('Execute next exploration pass.', { schema: FINDINGS })
  if (r) findings.push(...r.findings)
  log(`Discovered ${findings.length} findings; remaining token budget: ${Math.round(budget.remaining() / 1000)}k.`)
}
// Upfront scaling pattern: const FLEET = budget.total ? Math.floor(budget.total / 100_000) : 3
```

## 6. Custom Agent Types + Structured Output

Consume custom `.claude/agents/{name}.md` definitions directly within workflows:

```javascript
const r = await agent(
  `Inspect _workspace/02_draft.md and return identified findings.`,
  { agentType: 'qa-inspector',      // .claude/agents/qa-inspector.md
    schema: FINDINGS,               // Enforces schema adherence even for custom agent types
    effort: 'high' })               // High reasoning effort is recommended during verification phases
```

- Omitting `agentType` dispatches standard built-in workflow subagents.
- Select `model` according to each phase's workload profile: `sonnet` for routine collection and transformation, `opus` for rigorous verification and architecture, and `fable` for autonomous multi-phase planning. See `model-selection-guide.md` for complete criteria.
- Specify `isolation: 'worktree'` if parallel agents modify files concurrently. Because setting up worktrees incurs overhead, apply isolation only when concurrent write collisions are realistically possible.

## 7. Gotchas and Verification Checklist

Verify workflow scripts during Phase 6-2 of the harness build against this checklist:

| Gotcha | Failure Mode | Prevention |
|---|---|---|
| Variables or expressions in `meta` | Script fails to parse. | Use only static literals in `meta`. |
| TypeScript syntax (e.g., `: string[]`) | Script fails to parse. | Author in pure, valid JavaScript. |
| Calling `Date.now()`, `Math.random()`, or `new Date()` | Runtime throws error due to deterministic resume protection. | Pass timestamps and seeds via `args`. If randomness is needed, vary prompts deterministically by item index. |
| Missing `.filter(Boolean)` | Downstream stages crash when processing `null` returned by failed agents. | Always apply `.filter(Boolean)` to results from `parallel()` and `pipeline()`. |
| Unnecessary synchronization barriers | Fast agents are stalled waiting for slower agents, inflating overall latency. | Refactor into `pipeline()` unless results must be compared against the entire batch. |
| Phase title mismatch | Progress display fails to group properly. | Ensure `phase()` string calls match `title` fields in `meta.phases` verbatim. |
| Global `phase()` inside parallel lambdas | Concurrent phase declarations collide. | Pass `opts.phase` explicitly inside child calls. |
| Deduplicating against `confirmed` in loops | Rejected findings resurface every iteration, creating infinite loops. | Deduplicate against a comprehensive `seen` set. |
| Missing `budget.total` check in budget loops | Loops run until reaching the global per-workflow agent call ceiling in unmetered sessions. | Guard conditions with `while (budget.total && ...)`. |
| Truncating scope without disclosure | Processing only top N items while reporting "all tasks complete". | Explicitly report uninspected/omitted counts via `log()`. |
| Diagnosing outcomes without inspecting the journal | Misidentifying empty cached results as successful completions. | Always inspect the `journal` within the transcript directory when diagnosing completed workflows. |
| Writing script to disk before calling `Workflow` | Adds unnecessary procedural friction. | Pass script source directly via `script`. Files are archived automatically and can subsequently be modified via `Edit` and re-invoked via `scriptPath`. |

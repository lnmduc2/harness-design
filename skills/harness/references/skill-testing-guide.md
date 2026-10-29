# Skill Testing and Iterative Improvement Guide

This document describes methods for verifying deliverable quality and iteratively improving skills created by a harness. It serves as reference material supplementing Phase 6 of `SKILL.md`.

---

## Table of Contents

1. [Testing Framework Overview](#1-testing-framework-overview)
2. [Test Prompt Authoring](#2-test-prompt-authoring)
3. [Execution Testing: With-skill vs. Baseline](#3-execution-testing-with-skill-vs-baseline)
4. [Workflow-Based A/B Testing (v2)](#4-workflow-based-ab-testing-v2)
5. [Quantitative Evaluation via Assertions](#5-quantitative-evaluation-via-assertions)
6. [Utilizing Specialized Agents](#6-utilizing-specialized-agents)
7. [Iterative Improvement Loop](#7-iterative-improvement-loop)
8. [`description` Trigger Verification](#8-description-trigger-verification)
9. [Workspace Directory Structure](#9-workspace-directory-structure)

---

<a id="1-testing-framework-overview"></a>

## 1. Testing Framework Overview

Verify skill quality by combining **qualitative evaluation (human inspection of deliverables)** with **quantitative evaluation (automated grading via assertions)**:

| Evaluation Mode | Method | Best Suited For |
|---|---|---|
| **Qualitative Evaluation** | The user directly inspects and evaluates deliverables. | Writing tone, visual design, and creative outputs where quality assessment demands human judgment. |
| **Quantitative Evaluation** | Deliverables are scored automatically against objective assertions. | File creation, structured data extraction, and code generation where outcomes are objectively verifiable. |

Follow the cycle: **Author → Execute Test → Evaluate → Improve → Re-test**.

## 2. Test Prompt Authoring

### Authoring Principles

Author test prompts as **concrete, natural language requests typical of real users**. Abstract or synthetic prompts fail to expose whether a skill functions effectively in real-world scenarios.

**Poor Examples:** `"Process this PDF"`, `"Extract the data"`

**Good Example:**

```
"In 'Q4_Revenue_Final_v2.xlsx' located in the Downloads folder, add a Profit Margin (%) column
calculated using Column C (Revenue) and Column D (Cost). Then sort the rows in descending
order based on the Profit Margin."
```

### Diversifying Test Prompts

- Blend formal business phrasing with casual colloquial tones.
- Mix explicit instructions with implicit contextual prompts (e.g., explicitly specifying a file format vs. requiring the agent to infer it from context).
- Mix simple tasks with complex workflows; incorporate everyday realities such as abbreviations, typos, and informal phrasing.

### Scoping Use Cases

Start with 2–3 prompts. Always include at least one core happy path and one edge case/exception, adding a multi-step composite task case as needed.

<a id="3-execution-testing-with-skill-vs-baseline"></a>

## 3. Execution Testing: With-skill vs. Baseline

### 3-1. Comparing Two Executions

For each test prompt, launch two subagents **concurrently in a single message turn**:

- **With-skill Execution**: Ingests and follows the skill, saving deliverables under `_workspace/iteration-N/eval-{id}/with_skill/outputs/`.
- **Baseline Execution**: Processes the same prompt without access to the skill, saving deliverables under `_workspace/iteration-N/eval-{id}/without_skill/outputs/`.

### 3-2. Establishing the Baseline

| Situation | Baseline Setup |
|---|---|
| Creating a new skill | Execute the identical prompt without the skill. |
| Improving an existing skill | Execute against an archived copy of the pre-modification skill. |

### 3-3. Recording Latency and Token Usage

Extract and persist `total_tokens` and `duration_ms` **immediately** upon receiving subagent completion notifications. These metrics are only available at notification arrival and cannot be reconstructed retroactively.

<a id="4-workflow-based-ab-testing-v2"></a>

## 4. Workflow-Based A/B Testing (v2)

When testing three or more cases or performing recurring verification runs, codify the A/B evaluation as a workflow script. Execute only with explicit user agreement.

```javascript
export const meta = {
  name: 'skill-ab-test',
  description: 'Executes with-skill and baseline runs, then conducts blinded grading',
  phases: [{ title: 'Execution' }, { title: 'Grading' }],
}
const RUN_RESULT = { type: 'object', required: ['saved', 'files'], properties: {
  saved: { type: 'boolean' },
  files: { type: 'array', minItems: 2, items: { type: 'string', minLength: 1 } } } }
const SLOT_GRADE = { type: 'object', required: ['expectations', 'summary'], properties: {
  expectations: { type: 'array', items: { type: 'object',
    required: ['text', 'passed', 'evidence'], properties: {
      text: { type: 'string' }, passed: { type: 'boolean' }, evidence: { type: 'string' } } } },
  summary: { type: 'object', required: ['passed', 'failed', 'total', 'pass_rate'], properties: {
    passed: { type: 'integer' }, failed: { type: 'integer' }, total: { type: 'integer' },
    pass_rate: { type: 'number' } } } } }
const GRADE = { type: 'object', required: ['A', 'B', 'comparison'], properties: {
  A: SLOT_GRADE,
  B: SLOT_GRADE,
  comparison: { type: 'object', required: ['preferred', 'reason'], properties: {
    preferred: { type: 'string', enum: ['A', 'B', 'tie'] },
    reason: { type: 'string' } } } } }

const results = await pipeline(
  args.evals,   // [{id, prompt, assertions, skillPath, withSkillSlot: 'A' | 'B'}]
  e => {
    const withSkillSlot = e.withSkillSlot === 'B' ? 'B' : 'A'
    const baselineSlot = withSkillSlot === 'A' ? 'B' : 'A'
    return parallel([
      () => agent(
        `${e.prompt}\n\nFirst read and follow ${e.skillPath}. Save deliverables to ${args.ws}/${e.id}/with_skill/outputs/, and save identical copies to ${args.ws}/${e.id}/blind/${withSkillSlot}/outputs/. Return all saved file paths.`,
        { label: `WithSkill:${e.id}`, phase: 'Execution', schema: RUN_RESULT }),
      () => agent(
        `${e.prompt}\n\nSave deliverables to ${args.ws}/${e.id}/without_skill/outputs/, and save identical copies to ${args.ws}/${e.id}/blind/${baselineSlot}/outputs/. Return all saved file paths.`,
        { label: `Baseline:${e.id}`, phase: 'Execution', schema: RUN_RESULT }),
    ])
  },
  (runs, e) => {
    const completed = (runs ?? []).filter(Boolean)
    if (completed.length !== 2 || completed.some(r => !r.saved || r.files.length < 2)) {
      log(`${e.id}: Skipping grading because deliverables for both with-skill and baseline are not fully ready.`)
      return null
    }
    return agent(
      `First verify that outputs exist in both ${args.ws}/${e.id}/blind/A/outputs/ and ${args.ws}/${e.id}/blind/B/outputs/. If either directory is empty, do not grade; report failure reasons instead. If both exist, grade A and B independently, then declare the preferred output as A, B, or tie. To preserve blindness, do NOT open eval_metadata.json, with_skill/, or without_skill/. Assertions: ${JSON.stringify(e.assertions)}`,
      { label: `Grade:${e.id}`, phase: 'Grading', schema: GRADE })
      .then(grade => {
        if (!grade) log(`${e.id}: Failed to receive blind grading result; skipping evaluation case.`)
        return grade
      })
  }
)
const mapped = results.map((grade, index) => {
  if (!grade) return null
  const e = args.evals[index]
  const withSkillSlot = e.withSkillSlot === 'B' ? 'B' : 'A'
  const baselineSlot = withSkillSlot === 'A' ? 'B' : 'A'
  return {
    evalId: e.id,
    withSkill: grade[withSkillSlot],
    baseline: grade[baselineSlot],
    comparison: {
      preferred: grade.comparison.preferred === 'tie'
        ? 'tie'
        : grade.comparison.preferred === withSkillSlot ? 'with_skill' : 'baseline',
      reason: grade.comparison.reason,
    },
  }
}).filter(Boolean)
return { results: mapped }
```

The main agent alternates assigning `withSkillSlot` between A and B across test cases. The grader inspects only the neutral `blind/A/` and `blind/B/` paths, ensuring it remains agnostic to which slot utilized the skill. Only runs where both deliverables succeed are graded; upon completion, the main agent maps A/B verdicts back to with-skill vs baseline conditions and persists them to `grading.json`. Grading commences immediately as individual test runs complete without waiting for the entire batch. Re-running via `resumeFromRunId` after editing the skill utilizes cached results for unchanged evaluation cases.

<a id="5-quantitative-evaluation-via-assertions"></a>

## 5. Quantitative Evaluation via Assertions

### 5-1. Authoring Assertions

**Effective assertions** are objectively falsifiable, self-describing in what they inspect, and specifically measure capabilities enhanced by the skill. Avoid trivial assertions that pass unconditionally without the skill (e.g., "Output is generated") or assertions requiring subjective human preference (e.g., "Well written").

### 5-2. Programmatically Verifiable Conditions

Encode assertions that can be checked programmatically into test scripts. Programmatic validation is faster, more reliable, and remains reusable across iteration rounds.

### 5-3. Eliminating Non-discriminating Assertions

Assertions where both with-skill and baseline runs pass 100% of the time fail to measure marginal utility. Remove non-discriminating assertions or tighten them into criteria demanding enough to differentiate quality.

### 5-4. Grading Output Schema

Adhere strictly to the `grading.json` format (`text`, `passed`, `evidence`, and `summary`) defined in `skill-writing-guide.md`.

## 6. Utilizing Specialized Agents

| Role | Responsibility | When to Use |
|---|---|---|
| **Grader** | Validates pass/fail for each assertion, records evidence citations, cross-verifies factual claims, and evaluates assertion quality. | Every iteration round |
| **Blinded Comparator** | Anonymizes and shuffles deliverables A and B, performing rigorous comparative quality analysis. | When confirming that a new version delivers measurable improvement. |
| **Analyzer** | Identifies statistical patterns across runs (non-discriminating assertions, high-variance test cases, latency vs token trade-offs). | After accumulating 3 or more iteration rounds. |

<a id="7-iterative-improvement-loop"></a>

## 7. Iterative Improvement Loop

### 7-1. Improvement Principles

1. **Generalize feedback.** Tailoring a fix to an isolated test run causes overfitting. Extract principles that apply broadly across varied inputs.
2. **Prune unhelpful instructions.** Inspect agent tool execution traces; if skill rules induce unnecessary actions or detours, prune them.
3. **Explain the Why.** Identify why a piece of feedback matters, and embed that rationale clearly into the skill.
4. **Pre-bundle recurring tools.** If agents repeatedly write identical helper scripts during runs, package those scripts into `scripts/`.

### 7-2. Iteration Sequence

```
1. Modify the skill.
2. Re-execute all evaluation cases in a new `iteration-{N+1}/` workspace directory.
3. Present comparative metrics against previous iterations to the user.
4. Solicit user feedback and repeat the cycle.
```

**Termination Conditions:** Conclude when the user is satisfied, when no further feedback remains, or when marginal improvements plateau.

### 7-3. Fresh-Eyes Review

When refining a skill, author an initial draft. Then, inspect the draft from the perspective of an external reviewer reading the instructions for the first time. Iterate progressively rather than attempting perfection in a single pass.

<a id="8-description-trigger-verification"></a>

## 8. `description` Trigger Verification

### 8-1. Authoring Evaluation Prompts

Author 20 evaluation prompts: 10 `should-trigger` prompts and 10 `should-NOT-trigger` prompts.

**Authoring Criteria:**

- Use concrete, realistic natural language typical of real users.
- Include specific details such as file paths, operational context, column headers, or company names.
- Focus on ambiguous **boundary cases** rather than obvious extremes.

**`should-trigger`:** Include diverse ways of expressing identical intent, requests that omit explicit file extensions but clearly need the skill, rare domain use cases, and prompts overlapping with adjacent skills where this skill must take precedence.

**`should-NOT-trigger`:** **Near-miss boundary cases are critical.** Include prompts that share topical keywords with the skill, but where standard tools or alternative skills are actually more appropriate. Completely unrelated prompts provide zero testing value.

### 8-2. Collision Checks with Existing Skills

1. Collect the `description` fields of all existing skills.
2. Verify that `should-trigger` prompts for the new skill do not accidentally trigger existing skills.
3. If collisions occur, sharpen boundaries and exclusions within the `description`.

### 8-3. Automated Optimization (Advanced / Optional)

1. Partition 20 evaluation prompts into a 60% Train set and a 40% Test set.
2. Measure trigger accuracy of the current `description`.
3. Analyze failure modes and refine description phrasing.
4. Select the best performing `description` based on **Test set** accuracy. Selecting based on Train set results induces overfitting.
5. Repeat up to five optimization cycles.

> Execute automated checks via headless CLI scripts (`claude -p`). Because this consumes significant tokens, reserve this step for the final polishing of mature skills.

<a id="9-workspace-directory-structure"></a>

## 9. Workspace Directory Structure

```
_workspace/
├── iteration-1/
│   ├── eval-descriptive-name/
│   │   ├── eval_metadata.json
│   │   ├── with_skill/    (outputs/ + timing.json + grading.json)
│   │   ├── without_skill/ (outputs/ + timing.json + grading.json)
│   │   └── blind/         (A/outputs/ + B/outputs/)
│   └── benchmark.json
├── iteration-2/
└── evals/evals.json
```

**Retention Rules:**

- Name `eval` directories descriptively rather than using opaque numeric indices (e.g., `eval-multi-page-table-extraction`).
- Create independent directories per iteration; never overwrite previous `iteration` directories.
- Never delete `_workspace/`, as it provides the immutable system of record for post-hoc verification and audit history.

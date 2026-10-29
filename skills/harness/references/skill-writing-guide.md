# Skill Authoring Guide

This document provides detailed guidelines for authoring high-quality skills within a harness. It serves as reference material supplementing Phase 4 of `SKILL.md`.

---

## Table of Contents

1. [Authoring the `description`](#1-authoring-the-description)
2. [Body Writing Style](#2-body-writing-style)
3. [Defining Output Formats](#3-defining-output-formats)
4. [Example Authoring Patterns](#4-example-authoring-patterns)
5. [Progressive Disclosure Patterns](#5-progressive-disclosure-patterns)
6. [Script Bundling Decision Criteria](#6-script-bundling-decision-criteria)
7. [Standard Data Schemas](#7-standard-data-schemas)
8. [What NOT to Include in Skills](#8-what-not-to-include-in-skills)
9. [Skill Reuse Design](#9-skill-reuse-design)

---

<a id="1-authoring-the-description"></a>

## 1. Authoring the `description`

The skill fields exposed to Claude during discovery are `name` and `description`. Concrete invocation conditions reside in `description`. Claude inspects both fields to decide which skill to activate.

### How Skills Are Invoked

Claude tends not to invoke skills for simple tasks that can be handled straightforwardly using basic tools. For example, a simple prompt like "Read this PDF" might not activate a skill even with a well-crafted `description`. The more multi-stage, complex, or demanding of domain expertise a task is, the higher the likelihood that Claude activates the skill.

### Authoring Principles

1. State both **what the skill does** and the **specific circumstances under which it must be invoked**.
2. If similar-looking requests exist where the skill should NOT be invoked, explicitly delineate scope boundaries and negative triggers.
3. Because Claude is conservative when deciding whether to activate skills, express mandatory trigger conditions assertively (e.g., "MUST use when...").
4. In orchestrator skills that coordinate multi-step work, always include **follow-up keywords** (re-run, modify, revise, update, improve previous result).

### Good Example

```yaml
description: "Reads PDF files and extracts text/tables, performs merging, splitting,
  rotation, watermarking, encryption/decryption, and OCR. MUST use when the user mentions
  .pdf files or requests PDF deliverables. Especially essential for tasks requiring
  conversion, editing, or structural extraction rather than simple reading."
```

### Poor Examples

- `"A skill for processing data"` — Far too ambiguous; provides no indication of supported file types or operations.
- `"PDF-related tasks"` — Fails to specify capabilities or activation triggers.

<a id="2-body-writing-style"></a>

## 2. Body Writing Style

### Explain Why First

Large language models (LLMs) make sound judgments in edge cases when they understand the rationale behind a rule. Providing the reason why a rule must be followed is far more effective than simply enumerating imperative prohibitions.

**Poor Example:**

```markdown
Always use pdfplumber for table extraction. Never use PyPDF2 for tables.
```

**Good Example:**

```markdown
Use pdfplumber when extracting tables. PyPDF2 is designed for raw text extraction
and fails to preserve table row and column geometry. In contrast, pdfplumber recognizes
cell boundaries and yields structured data.
```

### Generalize into Broadly Applicable Principles

When issues are discovered during testing or feedback, do not tack on narrow instructions tailored only to that single example. Condense the fix into a principle applicable across diverse scenarios.

**Overfitted Fix:** `If a column is named "Q4 Revenue", convert that column to a number.`

**Generalized Fix:** `When column names contain numerical indicator terms like "Revenue", "Amount", or "Quantity", convert the column to numeric types. Leave non-convertible values in their raw state.`

### Use Imperative Voice

Write instructions in clear, active imperative terms (e.g., "Extract...", "Validate...", "Do not mutate...") rather than passive conversational suggestions. Skills are operational procedure manuals for AI agents.

### Conserve Context Window

A model's context window is a finite, valuable resource shared across skill instructions and task context. Evaluate whether every sentence justifies its token cost against these criteria:

- "Does Claude already know this standard concept?" → Delete.
- "Will Claude make mistakes without this instruction?" → Retain.
- "Is a concrete example clearer than a lengthy paragraph?" → Replace with an example.

<a id="3-defining-output-formats"></a>

## 3. Defining Output Formats

When deliverable formatting is critical, explicitly provide the structure agents must follow.

```markdown
## Report Structure
Adhere strictly to the following template:

# [Title]
## Executive Summary
## Key Findings
## Recommendations
```

Format specifications are most effective when paired with concise real-world examples.

**Downstream Workflow Artifacts:** If a downstream phase processes the output programmatically via code, specify a **JSON schema** instead of markdown templates. Explicitly state that the final output must be raw data passed to downstream stages, rather than conversational text addressed to the user. Because `agent()` in `Workflow` enforces schemas strictly, ensure the schema in the skill matches the workflow script.

<a id="4-example-authoring-patterns"></a>

## 4. Example Authoring Patterns

Concrete input/output examples convey constraints far more clearly than abstract rules.

```markdown
## Commit Message Formatting

**Example 1:**
Input: Added user authentication using JWT tokens
Output: feat(auth): implement JWT authentication

**Example 2:**
Input: Fixed bug where password visibility button does not toggle on login page
Output: fix(login): correct toggle behavior for password visibility button
```

<a id="5-progressive-disclosure-patterns"></a>

## 5. Progressive Disclosure Patterns

### Pattern 1: Splitting by Domain Area

```
bigquery-skill/
├── SKILL.md (Overview + domain selection routing)
└── references/
    ├── finance.md (Revenue and billing metrics)
    ├── sales.md (Opportunities and pipeline)
    └── product.md (API usage and features)
```

If the user inquires about revenue metrics, load only `finance.md`.

### Pattern 2: On-Demand Deep-Dives

```markdown
## Document Editing
For standard edits, modify the XML directly.
**If change tracking (redlining) is required**, refer to [REDLINING.md](references/redlining.md).
```

### Pattern 3: Table of Contents for Large References

Include a table of contents at the top of any reference file exceeding 300 lines.

<a id="6-script-bundling-decision-criteria"></a>

## 6. Script Bundling Decision Criteria

Inspect execution logs from testing agents. When the following patterns recur, bundle scripts or procedures directly into the skill:

| Recurring Pattern | Action |
|---|---|
| Multiple test runs repeatedly write identical helper scripts. | Bundle the script into `scripts/`. |
| Runs repeatedly execute the same `pip install` or `npm install` command. | Document explicit dependency setup steps in the skill. |
| Runs consistently follow identical multi-step procedural sequences. | Codify the sequence as a standard workflow in the skill body. |
| Runs encounter similar runtime errors and repeatedly devise identical workarounds. | Document the known issue and mitigation procedure in the skill. |

Always execute and verify bundled scripts directly before shipping.

## 7. Standard Data Schemas

Use the following standard schemas to ensure consistent data exchange across skills, as well as during automated testing and evaluation.

### `eval_metadata.json`

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "User task prompt here",
  "assertions": [
    "Deliverable contains X",
    "File generated in format Y"
  ]
}
```

### `grading.json`

```json
{
  "expectations": [
    {
      "text": "Deliverable contains 'Seoul'",
      "passed": true,
      "evidence": "Verified 'Extracted Seoul regional data' in Step 3"
    }
  ],
  "summary": { "passed": 1, "failed": 0, "total": 1, "pass_rate": 1.0 }
}
```

**Field Name Caution:** Use exact field names: `text`, `passed`, and `evidence`. Do not rename them to `name`, `met`, or `details`.

### `timing.json`

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

Extract and persist `total_tokens` and `duration_ms` immediately upon receiving subagent completion notifications. These metrics are only available at notification time and cannot be retrieved later.

<a id="8-what-not-to-include-in-skills"></a>

## 8. What NOT to Include in Skills

- Auxiliary documentation such as `README.md`, `CHANGELOG.md`, or `INSTALLATION_GUIDE.md`.
- Transient metadata relevant only during skill development (scratchpad test results, iteration notes).
- End-user guides (skills are operational instruction sets for AI agents, not human user manuals).
- Common knowledge that standard foundation models already possess.
- Legacy v1 artifacts (`TeamCreate`, `TeamDelete`, experimental flags, blanket model pins).

## 9. Skill Reuse Design

Before creating a new skill, compare functionality against existing skills in `.claude/skills/`. Successive harness expansions can cause duplicate skills with different names to proliferate.

| Relationship with Existing Skill | Action |
|---|---|
| Existing skill fully subsumes the new capability | Do not create a new skill; attach the existing skill to the agent. |
| Partial overlap; generalizing existing skill covers new capability | Generalize and extend the existing skill. |
| Partial overlap, but domain specialization was deliberate | Create a dedicated new skill. |
| Functional domain is entirely distinct | Create a new skill. |

**Principle:** Single-responsibility skills maximize reusability and minimize duplication. If a skill accumulates multiple disparate roles, evaluate whether to split it.

### Boundary of Generalization

Because generalization can continue indefinitely, stop at the defined boundary of the skill's core responsibility. Retain intentional domain specialization while stripping accidental dependencies.

For instance, generalizing a "Fintech Risk Assessment PDF" skill:

| Stripped Dependency | Resulting Scope |
|---|---|
| Fintech | "Assessment Report PDF". If the core responsibility is generating evaluation reports, stop here. |
| Assessment | "PDF Form Authoring". If an identical utility skill already exists, reuse it rather than generalizing. |

If the intended purpose was strictly "Fintech Risk Assessment" from inception, retain it as a domain-specialized skill rather than prematurely generalizing.

Extending an existing skill may alter the behavior of other agents relying on it. Before expanding a skill, identify all agents that consume it, and update the expanded scope in the skill's `description`.

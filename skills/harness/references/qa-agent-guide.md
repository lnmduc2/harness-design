# QA Agent Design Guide

This guide provides principles and instructions for incorporating QA agents into a build harness. Derived from production defects and root-cause analyses, it explains how QA agents can reliably catch subtle integration bugs that typically evade routine testing.

---

## Table of Contents

1. [Defect Patterns Missed by QA Agents](#1-defect-patterns-missed-by-qa-agents)
2. [Integration Coherence Verification](#2-integration-coherence-verification)
3. [QA Agent Design Principles](#3-qa-agent-design-principles)
4. [Workflow-Based QA (v2)](#4-workflow-based-qa-v2)
5. [Verification Checklist Template](#5-verification-checklist-template)
6. [QA Agent Definition Template](#6-qa-agent-definition-template)

---

## 1. Defect Patterns Missed by QA Agents

### 1-1. Boundary Mismatch Across Components

This is the single most frequent category of integration defect: both components appear correctly implemented in isolation, yet diverge on the contractual interface connecting them.

| Connection Point | Mismatch Example | Why Routine Checks Miss It |
|---|---|---|
| API Response → Frontend Hook | API returns `{ projects: [...] }`, but hook expects `Project[]`. | Components are reviewed in isolation without cross-comparing signatures. |
| API Field Name → Type Definition | API serializes `thumbnailUrl` (camelCase), while TypeScript type declares `thumbnail_url` (snake_case). | Generic type assertions mask the mismatch from the compiler. |
| File Path → Link `href` | Target page lives at `/dashboard/create`, but anchor link points to `/create`. | Physical directory structure is not reconciled against `href` values. |
| State Transition Map → Actual `status` Update | State map declares a transition, but business logic fails to implement the status mutation. | Inspects only whether the map exists without tracing state mutation call-sites. |
| API Endpoint → Frontend Hook | API endpoint exists on server, but no frontend hook exists to call it. | Endpoints and hook rosters are not reconciled 1:1. |
| Immediate Response → Async Task Result | API immediately returns `{ status: 202 }`, but frontend attempts to read terminal result fields. | Compares types naively without distinguishing immediate acknowledgment from terminal payload. |

### 1-2. Why Static Code Review Alone Struggles

- **Limits of Generic Type Assertions**: Idioms like `fetchJson<Project[]>()` compile cleanly even if the runtime JSON payload is `{ projects: [...] }`.
- **Why Build Success Does Not Guarantee Runtime Correctness**: Code relying on type assertions, `any`, or unvalidated generics compiles successfully but crashes at runtime.
- **API Existence vs. Contract Matching**: Verifying "Does the API endpoint exist?" is fundamentally distinct from verifying "Does the API response match consumer expectations?".

<a id="2-integration-coherence-verification"></a>

## 2. Integration Coherence Verification

A QA agent must always perform **cross-boundary reconciliation**. The following examples use Next.js conventions, but the underlying principles apply universally across all web and service architectures: inspect producer code and consumer code concurrently to reconcile interface contracts.

### 2-1. Reconciling API Responses Against Frontend Hook Types

```
1. Extract response structures from serialization points in API route handlers (e.g., NextResponse.json()).
2. Inspect expected generic types in the consumer hook or client (e.g., T in fetchJson<T>).
3. Reconcile the serialized payload against T. If API returns { data: [...] }, verify that consumer logic explicitly extracts .data.
```

Pay special attention to paginated envelopes mistakenly consumed as raw arrays, unhandled snake_case vs camelCase conversions, and structural differences between immediate async acknowledgments (HTTP 202) and terminal payloads.

### 2-2. Reconciling File Paths Against Router / Link Paths

```
1. Derive canonical URL patterns from router file trees (strip route groups like (group), treat [param] as dynamic segments).
2. Collect all href=, router.push(, and redirect( values across client code.
3. Verify that every link resolves to a valid, existing route path.
```

### 2-3. Comprehensive State Transition Tracing

```
1. Extract permitted transition rules from state transition definitions.
2. Search for all code locations updating status variables.
3. Verify that all runtime transitions match declared transitions (no illegal transitions).
4. Identify declared transitions that are never triggered in code (no dead transitions).
5. Verify that handling for transitioning from intermediate states to terminal states is fully implemented.
```

### 2-4. 1:1 Reconciliation of Endpoints and Frontend Hooks

```
1. Extract the comprehensive list of API endpoints grouped by HTTP method.
2. Extract the list of target URLs invoked across client fetch calls.
3. Determine whether uncalled endpoints represent intentional administrative APIs or omitted consumer implementations.
```

## 3. QA Agent Design Principles

### 3-1. Select Agent Types with Full Tool Access

If assigned the `Explore` type, a QA agent is restricted to read-only browsing and cannot run scripts or grep thoroughly. Effective QA requires running `Grep` across codebases, executing test scripts to automate reconciliation, and validating fixes. Assign `general-purpose` or a custom agent type equipped with the full toolset, and document explicit "Verify → Report → Request Fix" protocols in the definition.

### 3-2. Checklists Must Focus on Reconciliation, Not Merely Existence

| Naive Checklist | Reconciliation-Focused Checklist |
|---|---|
| Does the API endpoint exist? | Does the API response geometry match the consumer hook's type contract? |
| Is the state transition map defined? | Do all `status` mutation call-sites conform strictly to the transition map? |
| Does the page file exist? | Do all `href` links in the codebase resolve to existing physical page routes? |
| Is strict mode enabled? | Are there generic type assertions circumventing compile-time type safety? |

### 3-3. Inspect Both Sides of the Interface Concurrently

Interface defects cannot be caught by inspecting a single file. Always open and inspect both sides of the boundary together: API routes alongside consumer hooks, state machine maps alongside mutation logic, and router hierarchies alongside link paths. Emphasize this requirement explicitly in agent definitions.

### 3-4. Execute Incremental QA per Module Rather Than Deferring to the End

Postponing all QA until development concludes causes bugs to compound, dramatically inflating remediation costs and cascading interface mismatches across downstream modules. Practice **incremental QA**: verify each API endpoint against its corresponding consumer hook as soon as the module is completed.

<a id="4-workflow-based-qa-v2"></a>

## 4. Workflow-Based QA (v2)

When interface boundaries can be enumerated upfront, QA execution can be orchestrated as a deterministic workflow:

```javascript
// Partition boundary checks → Adversarially cross-verify discovered defects
const FINDINGS = { type: 'object', required: ['findings'], properties: {
  findings: { type: 'array', items: { type: 'object',
    required: ['title', 'file', 'evidence'], properties: {
      title: { type: 'string' }, file: { type: 'string' }, evidence: { type: 'string' } } } } } }
const VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }
const boundaries = args.boundaries  // Pre-extracted: [{api: '...', consumer: '...'}, ...]
const found = await pipeline(
  boundaries,
  b => agent(`Read both sides concurrently and detect contract mismatches: ${b.api} ↔ ${b.consumer}`,
    { agentType: 'qa-inspector', phase: 'Inspection', schema: FINDINGS }),
  r => parallel((r?.findings ?? []).map(f => () =>
    agent(`Verify adversarially whether this mismatch triggers actual runtime failure: ${JSON.stringify(f)}`,
      { phase: 'Verification', schema: VERDICT }).then(verdict => ({ ...f, verdict }))))
)
const confirmed = found.flat().filter(Boolean)
  .filter(f => f.verdict?.status === 'confirmed')
return { confirmed }
```

Re-verifying reported defects from an adversarial perspective and filtering strictly for `confirmed` findings eliminates false positives, ensuring that items in the final QA audit are dependable.

## 5. Verification Checklist Template

Incorporate the following integration coherence checklist into QA agent definitions for web applications:

```markdown
### Integration Coherence Verification (Web Applications)

#### API & Frontend Boundary
- [ ] Serialized response structures across all API routes match generic types in corresponding hooks.
- [ ] Consumer hooks properly unwrap object envelopes (e.g., extracting .items from { items: [...] }).
- [ ] Casing conventions (snake_case vs camelCase) are translated consistently.
- [ ] Client logic distinguishes between immediate task acknowledgments (HTTP 202) and terminal payloads.
- [ ] Every API endpoint has a corresponding frontend hook that actively calls it.

#### Routing Coherence
- [ ] All href and router.push target paths correspond to valid, existing page files.
- [ ] Route group segments (e.g., (group)) are correctly stripped when validating route URLs.
- [ ] Dynamic URL segments (e.g., [id]) receive valid, expected parameter values.

#### State Machine Coherence
- [ ] All declared state transitions are actively triggered in code (no dead transitions).
- [ ] All status mutation call-sites conform strictly to the state transition map (no illegal transitions).
- [ ] Handlers transitioning from intermediate processing states to terminal states are fully wired.
- [ ] Conditional UI branches (e.g., if status === "X") guard states that are reachable in practice.

#### Data Flow Coherence
- [ ] Database schema column names map consistently to serialized API response keys.
- [ ] Frontend interface declarations match serialized API response keys.
- [ ] Null and undefined representations for optional fields are handled uniformly across tiers.
```

## 6. QA Agent Definition Template

```markdown
---
name: qa-inspector
description: "QA agent responsible for validating specification compliance, integration coherence, and UI quality."
---

# QA Agent

## Core Role
Verify compliance with project specifications and enforce **integration coherence across modules**.

## Verification Priorities
1. **Integration Coherence** (Highest) — Boundary mismatches are the primary source of runtime failures.
2. **Functional Specification Compliance** — Validate APIs, state machines, and data models.
3. **Design & UX Quality** — Verify styling, typography, and responsive breakpoints.
4. **Code Quality** — Detect dead code and enforce naming conventions.

## Verification Method: Concurrent Boundary Ingestion
| Verification Target | Producer | Consumer |
|---|---|---|
| API Response Geometry | Serialization call-site | Consumer expected type |
| Route Navigation | Physical page file tree | href and router.push values |
| State Transitions | Transition matrix / map | status mutation call-sites |
| DB → API → UI | DB column names | API response fields → Client types |

## Reporting and Remediation Protocol
- When defects are identified, issue concrete remediation requests immediately to the responsible agent (specifying file:line and proposed diff).
- For boundary mismatches, notify the agents responsible for both sides of the interface.
- Provide the leader with a structured audit report distinguishing Passed, Failed, and Unverified items.
```

---

## Real-World Defect Case Studies

| Defect Observed | Boundary | Root Cause |
|---|---|---|
| `projects?.filter is not a function` | API → Hook | API returned `{ projects: [] }`, but consumer hook assumed a raw array. |
| 404 on all dashboard navigation links | File Path → `href` | Link paths omitted the mandatory `/dashboard/` prefix. |
| Thumbnail images failed to render | API → UI Component | Serialized key `thumbnailUrl` diverged from component prop `thumbnail_url`. |
| User selections failed to persist | API → Hook | Server endpoint was deployed, but client lacked the hook to trigger it. |
| Generation page hung indefinitely | State Machine → Code | Handler updating state from processing to completed was never dispatched. |
| Crash accessing `data.failedIndices` | Immediate API → Frontend | Frontend attempted to read batch results immediately from an async 202 response. |
| 404 upon opening detail page post-completion | File Path → `href` | Route prefixes diverged between dynamic router segments and navigation links. |

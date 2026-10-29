# Claude Model Selection Guide

Select models for agents based on four criteria: **complexity, estimated duration, required autonomy, and response latency**. Specify the model via `model:` in the YAML frontmatter of an agent definition file, the `model` argument in the `Agent` tool, or `opts.model` passed to `agent()` in `Workflow`. Document the rationale for your choice in comments within the agent definition or orchestration skill.

## Summary Decision Table

| Model | Selection Criteria | Representative Tasks |
|---|---|---|
| **fable** | Long-horizon autonomous model that self-plans from complex objectives and connects multiple stages to produce final deliverables | High-difficulty reasoning, creative tasks, agent orchestration, and highly challenging tasks requiring planning through long-term autonomous execution |
| **opus** | Reasoning-focused model that deeply analyzes specialized, complex problems to deliver rigorous judgments and conclusions | Architecture and design, code generation, complex analysis, cross-verification, critique, and advanced creative work |
| **sonnet** | Balanced general-purpose model that reliably handles the vast majority of routine writing, coding, analysis, and research tasks | General writing/coding/research, log parsing, format conversion, static file auditing, deployment script execution, and simple data extraction |

## 1. Fable

**Primary Role:** Suited for highly demanding tasks that require autonomous long-horizon planning and execution. Rather than merely answering immediate prompts, it understands broad objectives, plans multiple stages, and connects them sequentially to deliver complete end products.

### Well-suited Tasks

**① Multi-stage Long-horizon Workflows** — Tasks that cannot be resolved in a single response turn and require executing multiple phases in succession:
- Market research → Competitor analysis → Strategy formulation → Report authoring
- Requirements analysis → Service architecture design → Development planning → Deliverable implementation
- Reviewing multiple documents to formulate synthesized conclusions and actionable execution roadmaps
- Designing the end-to-end lifecycle of long-term projects

Crucially, these are tasks where stages are not isolated from one another, but where **outputs from earlier stages directly inform and reshape subsequent stages**.

**② Extensive and Complex Document Review** — Reviewing large volumes of complex documentation (research papers, industry reports, meeting transcripts, technical specifications) concurrently. Suitable when tasks extend beyond mere summarization to:
- Analyzing commonalities and divergences across disparate sources
- Identifying conflicting claims and counter-arguments
- Disentangling foundational evidence from tangential data
- Synthesizing vast amounts of material into a coherent, structured deliverable

**③ Transforming Vague Ideas into Concrete Deliverables** — Tasks where requirements are not finalized and only high-level conceptual ideas are provided, requiring the model to determine necessary details autonomously:
- Expanding a high-level concept ("a service with this vibe") into a formal business proposal
- Designing the structure and content of presentation decks from broad topic prompts alone
- Refining incomplete product ideas into concrete feature specifications and development plans
- Producing polished reports or content from scratch without pre-existing drafts

Choose Fable when **the capacity to exercise judgment and fill in missing blanks** is more critical than strictly following rigid procedural instructions.

**④ Problems Requiring Deep Reasoning Combined with Planning, Coordination, and Long-term Execution** — Fable is not categorically deeper in raw reasoning than Opus on isolated prompts; rather, Fable is uniquely suited when deep reasoning must be coupled with multi-step planning, coordination, and sustained autonomous execution.

### When to Select Fable

- Achieving results requires navigating multiple interdependent stages.
- The user cannot easily specify every operational step upfront.
- The model must autonomously formulate its own sequence of operations.
- Polished, comprehensive deliverables must be synthesized from vast source materials.
- Project-level outcomes are needed rather than brief answers.

## 2. Opus

**Primary Role:** A premier reasoning model tailored for deeply analyzing and solving difficult problems. While Fable excels at long-term planning and autonomous execution, Opus excels at deeply penetrating bounded, complex problems with rigorous logical analysis.

### Well-suited Tasks

**① Complex Research and Analysis** — Tasks extending beyond information retrieval to evaluating competing evidence and reaching rigorous logical conclusions:
- Conducting deep investigations into complex industries or markets
- Comparative meta-analysis across multiple research findings
- Evaluating trade-offs of public policies or corporate business strategies
- Analyzing causes and effects based on empirical data and literature
- Synthesizing divergent perspectives into definitive conclusions

**② Long Technical Document Review** — Ingesting and analyzing documents characterized by dense terminology and intricate structures (software specifications, system architecture documents, research papers, technical standards, PRDs, complex API documentation). Suited for **dissecting logical architecture and technical implications** rather than merely producing condensed summaries.

**③ Research Methodology Critique** — Evaluating not only the conclusions of research, but also the validity and rigor of the methods used to obtain them:
- Checking whether study design is sound, sample sizes are adequate, and selection bias exists
- Verifying whether measurement methods and statistical conclusions are statistically valid
- Evaluating whether conclusions exaggerate empirical evidence or admit plausible alternative interpretations

**④ Multi-step Autonomous Coding** — Progressing beyond generating one or two lines of code to understanding project goals and executing interconnected engineering tasks:
- Analyzing existing codebases and pinpointing root causes of elusive bugs
- Executing cross-file code refactoring, feature implementation, and test creation
- Improving code architecture and determining necessary next steps autonomously throughout development

### When to Select Opus

- The problem is technically demanding, but **the scope is relatively well-bounded**.
- Deep logical analysis, critique, and rigorous verification are paramount.
- Precise comprehension of dense technical documentation or research papers is required.
- Identifying subtle logical flaws or edge cases in research or designs.
- Solving complex coding or algorithmic challenges.

## 3. Sonnet

**Primary Role:** A balanced, reliable general-purpose model suited for executing the vast majority of routine day-to-day tasks. Unless a task demands exceptionally deep reasoning or long-horizon autonomous planning, Sonnet is versatile across writing, coding, research, and analysis.

### Well-suited Tasks

**① Writing and Content Production** — Emails, blog posts, report drafts, marketing copy, editing and proofreading, speech manuscripts, social media content, and brainstorming.

**② Routine Coding Tasks** — Writing standard functions, bug fixing, building simple tools, explaining existing code, refactoring, writing unit tests, and authoring data processing scripts.

**③ Analysis and Research** — Topic overviews, product/service comparisons, document content analysis, basic competitor research, key argument extraction, pros-and-cons evaluation, and tabulating results into clean summaries.

**④ Bounded Multi-step Problem Solving** — Executing tasks with clear, well-defined boundaries across several stages (e.g., summarizing source data then drafting a presentation, analyzing requirements then generating code snippets, analyzing datasets then explaining trends). Distinct from open-ended autonomous projects assigned to Fable.

**⑤ Repetitive and Operational Tasks** — Parsing log files, format conversion, static file auditing, deployment script execution, and simple data extraction.

### When to Select Sonnet

- Uncertain which model tier is required (treat Sonnet as the default).
- Routine tasks requiring fast, consistent, and cost-effective completion.
- Handling diverse blends of general writing, coding, and inquiry.
- Managing problems of moderate complexity (neither trivial nor profoundly ambiguous).
- Both processing speed and output quality are vital constraints.

## Differentiating Between Fable and Opus

- **Fable**: Plans the end-to-end task and executes autonomously over extended horizons.
- **Opus**: Deeply reasons through and analyzes bounded, complex problems.

For instance, critiquing an individual research paper is ideal for **Opus**, whereas reviewing dozens of papers to formulate a multi-year research strategy and compile the final thesis is ideal for **Fable**. Use Fable when upstream results must dynamically reshape subsequent stages under autonomous guidance. Use Opus when conducting deep analysis of a single, well-scoped problem.

## Harness Application Rules

1. **Judge by the task, not perceived agent importance** — Never select a model simply because "this agent sounds important." Evaluate the actual task's complexity, duration, required autonomy, and latency needs.
2. **Avoid unjustified blanket assignments** — Blanket-assigning all agents to Fable or Opus merely inflates latency and token burn. Select an appropriate tier for each agent and document the rationale in comments.
3. **Differentiate orchestration and execution layers** — Even if the orchestration layer (main session or supervisor agent) uses Fable for high-level planning, assign appropriately tiered models to downstream worker agents. A Fable orchestrator does not imply its workers need to be Fable.
4. **Select models per workflow stage** — In `Workflow` scripts, assign models dynamically per stage via `opts.model`. Use Sonnet for repetitive collection and transformation stages, Opus for rigorous verification, judging, and architectural design, and Fable for overarching planning and multi-stage autonomous synthesis.

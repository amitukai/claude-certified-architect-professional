# CCAR-P Study Guide — Domain 1: Solution Design & Architecture (17%)

**Source grounding:** Sub-objectives a–f below are copied verbatim from Anthropic's published CCAR-P Exam Guide v1.0 blueprint, as listed at `claudecertificationguide.com/ccar-p`. That page confirms the domain weight (17%) and the six sub-objectives but does not (yet) publish worked lesson content — its own prep track is marked "coming soon." The explanations, scenarios, distractor patterns, and mock questions below are built on Anthropic's own published architecture framework (the "Building Effective Agents" engineering guidance — augmented LLM / workflows / agents), not on any exam-leaked material. Terminology matches the blueprint's own wording (workflow, agentic, augmented LLM) intentionally.

**Domain 1 sub-objectives (official blueprint):**
- **1a.** Translate business problems into Claude-based AI solutions
- **1b.** Design end-to-end architectures (input → processing → output → feedback loops)
- **1c.** Select appropriate architectural patterns (workflow, agentic, augmented LLM)
- **1d.** Design multi-agent systems and orchestration strategies
- **1e.** Apply decomposition techniques for complex problem solving
- **1f.** Align solutions to business value pillars (efficiency, transformation, productivity, cost, performance SLAs)

At 17% weight and 63 total items, expect roughly **10-11 Domain 1 questions** on the real exam, mostly scenario-based, several multi-response.

---

## 1a. Translating Business Problems into Claude-Based AI Solutions

### What this really tests
Not "can you use Claude" — it tests whether you can tell when an LLM/agentic solution is the **wrong** answer, and when a workflow beats a full agent. The exam rewards architects who default to the simplest deterministic solution and only reach for Claude when the problem has genuine judgment, unstructured input, or open-ended reasoning at its core.

A problem is a good candidate for a Claude-based solution when it involves:
- Unstructured or semi-structured input (free text, scanned documents, mixed-format tickets)
- Judgment calls that don't reduce to a fixed rule set (intent classification, tone, risk narrative interpretation)
- High variability in the "shape" of each request, so a rules engine would need constant rewrites
- Tolerance for occasional error, paired with a feasible human-review or validation layer

A problem is a **poor** candidate when it involves:
- Deterministic, auditable math (interest calculations, tax computation) — use a calculator/rules engine, optionally called *as a tool* by Claude
- Exact-match lookups over structured data — use a database query, not model inference
- Zero tolerance for any variance with no feasible review step (e.g., final trade execution with no checks)

### Scenario example
*A regional bank's back office manually reviews 4,000 loan-modification requests per month. Each request arrives as a PDF with free-text hardship letters, financial statements, and inconsistent formatting. Analysts currently read each document, classify hardship type, and route to one of six specialist queues.*

The correct architectural instinct: this is a strong Claude fit — the "hardship type" classification is a judgment call over unstructured text, not a lookup. The correct solution frames it as an augmented LLM (document read + classification) feeding a routing workflow, with human analysts reviewing outputs below a confidence threshold — not a "fully autonomous agent that also approves the modification," which conflates a good text-understanding problem with a high-stakes, low-tolerance financial decision that should stay human-approved.

### Distractor patterns to watch for
- **"More AI is more thorough" trap** — an answer option that hands the *entire* loan-approval decision to Claude sounds "more complete" but violates the human-in-the-loop principle for high-stakes irreversible actions (this crosses into Domain 5 territory, and the exam likes to test the boundary between domains).
- **Deterministic-task mislabeling** — an option like "use Claude to calculate the exact modified payment schedule" is a distractor: that's arithmetic, better done by a tool/calculator that Claude *calls*, not something Claude should compute directly in free text.
- **Underestimating fit** — an option that dismisses the whole problem as "not suitable for LLMs, use only rules-based automation" is also wrong when the input actually is unstructured judgment work; the exam tests both over-eagerness and under-eagerness to apply Claude.

---

## 1b. Designing End-to-End Architectures (Input → Processing → Output → Feedback Loops)

### What this really tests
Whether you can architect the **whole lifecycle**, not just the model call. Every well-formed answer on this sub-objective should account for four stages:

| Stage | What belongs here |
|---|---|
| **Input** | Ingestion, normalization, format handling (PDF, email, chat, API payload), initial validation/sanitization |
| **Processing** | Context assembly (retrieval, memory, tool results), the augmented-LLM call(s), any orchestration logic |
| **Output** | Structured response generation, guardrail/output validation, formatting for the downstream consumer |
| **Feedback loop** | Logging, confidence scoring, human review routing, and — critically — a mechanism to feed corrections back into prompts/examples/evaluation datasets over time |

The feedback loop is the piece candidates most often skip, and it's the piece the exam most often tests, because it's what separates a one-off prototype from a production architecture.

### Scenario example
*An enterprise support desk wants a Claude-based system that drafts responses to incoming tickets. Leadership wants continuous improvement without constant manual prompt rewriting.*

A complete architecture: tickets ingested and normalized (input) → relevant KB articles retrieved and assembled into context, Claude drafts a response (processing) → response passes an output-guardrail check for policy compliance before being shown to the agent (output) → agent edits or approves the draft, and every edit is logged and periodically reviewed to update the few-shot examples or system prompt (feedback loop). An answer that stops at "Claude drafts the response and it's sent to the customer" is incomplete — it's missing both the output validation step and the feedback loop, and is a strong exam distractor because it "sounds done."

### Distractor patterns to watch for
- **Feedback-loop omission** — the most common wrong-but-plausible answer is architecturally correct on input/processing/output but silently drops the feedback loop. If an option describes a system with no mechanism for the output quality to influence future behavior, it is incomplete for this sub-objective even if nothing else is wrong.
- **Feedback loop confused with retraining** — a distractor may claim the feedback loop requires fine-tuning the underlying model. For most enterprise Claude architectures, the feedback loop is prompt/example/evaluation-dataset iteration, not model retraining — retraining Claude itself is not something the architect controls.
- **Output validation skipped** — an option that sends the raw model output straight to the end user/system without any guardrail check is a red flag distractor, especially in regulated-content scenarios.

---

## 1c. Selecting Architectural Patterns (Workflow, Agentic, Augmented LLM)

This is the single most heavily tested sub-objective in Domain 1. It maps directly to Anthropic's own published pattern taxonomy. Know it cold.

### The building block: Augmented LLM
An LLM enhanced with **retrieval** (external knowledge), **tools** (ability to take actions), and **memory** (context retention). This is the foundation every pattern below is built from — it is not itself a "workflow" or an "agent," it's the base unit.

### Workflows — predefined, code-orchestrated paths (predictable, auditable, cheaper)

| Pattern | Mechanism | Best-fit scenario |
|---|---|---|
| **Prompt chaining** | Task split into sequential steps; each LLM call's output feeds the next | Draft a document, then a second call translates it; or generate an outline, then expand it |
| **Routing** | An initial classification step directs input to one of several specialized downstream paths | Classify a support ticket, then send billing questions to one prompt/model and technical questions to another |
| **Parallelization (sectioning)** | Independent subtasks run concurrently, results aggregated programmatically | Screening a long contract for multiple, independent risk categories at once |
| **Parallelization (voting)** | The same task run multiple times/ways, outputs compared for consensus | Running a content-moderation check three times and flagging only if a majority agree it's a violation |
| **Orchestrator-workers** | A central LLM call dynamically decomposes a task and delegates subtasks to worker LLM calls, then synthesizes results | A coding task where the orchestrator decides which files need changes, dispatches a worker per file, and merges results |
| **Evaluator-optimizer** | One LLM generates, a second LLM evaluates against criteria and returns feedback in a loop | Iteratively refining a translation until an evaluator model confirms it preserves nuance |

### Agents — the LLM dynamically directs its own process
Used when the number of steps **cannot be predicted in advance**, and the task has clear success criteria plus a feasible feedback loop from the environment (tool results, code execution, search results). The agent plans, acts, checks ground truth, and decides its own next step — rather than following a path a human engineer wired in code.

### The decision rule the exam wants you to apply
Anthropic's own guidance (and the exam's implicit grading logic) favors the **simplest pattern that solves the problem**. Only add workflow complexity, and only escalate from workflow to full agent, when the simpler option demonstrably fails — because every step up costs predictability, latency, and money.

### Scenario example
*A legal team wants a system to review incoming vendor contracts. The number of issues to check for is fixed and known (12 standard risk clauses), and each check is independent of the others.*

Correct pattern: **parallelization (sectioning)** — twelve independent checks run concurrently, then results aggregate into a single report. This is a workflow, not an agent, because the task set is fixed and known in advance; no dynamic planning is required. A full autonomous agent is over-engineering here — more expensive, less predictable, and solving a problem the workflow already solves deterministically.

*Contrast:* if the task were instead "investigate this vendor's overall risk profile using whatever public and internal sources are relevant, and decide what to check," the number and nature of steps can't be predicted up front — that shifts the correct pattern toward an **agent**.

### Distractor patterns to watch for
- **Agent over-selection** — the exam frequently offers "use a fully autonomous agent" as a distractor for problems that are actually fixed, known-step workflows. If the task's steps are enumerable in advance, an agent is usually the wrong (over-engineered, non-simplest) answer.
- **Workflow under-selection** — the reverse trap: a genuinely open-ended, unpredictable-step problem framed as a fixed-step "just chain three prompts" workflow. If the number/order of steps depends on what earlier steps discover, that's agentic territory, not chaining.
- **Pattern-name confusion** — options that swap "orchestrator-workers" for "parallelization" as if identical. Parallelization is *independent* concurrent subtasks; orchestrator-workers is *dynamic, centrally decided* decomposition and delegation. The exam tests this distinction directly.
- **Missing the augmented LLM baseline** — an option jumping straight to a complex multi-step workflow when a single augmented LLM call (with the right tool/retrieval) already solves the problem is a classic over-engineering distractor.

---

## 1d. Designing Multi-Agent Systems and Orchestration Strategies

### What this really tests
Whether you know *when* multiple agents earn their cost (context isolation, parallelizable independent subtasks, distinct specialist skill sets) versus when a single agent or a workflow is simply cheaper and more reliable. Multi-agent systems trade cost and coordination complexity for parallel throughput and clean context separation.

### Core design considerations
- **Orchestration topology** — a lead/orchestrator agent that decomposes and delegates to specialist subagents (mirrors orchestrator-workers, but with full agentic subagents rather than single LLM calls), versus a flatter peer-to-peer handoff model where agents pass a task directly to the next relevant specialist.
- **Context isolation** — each subagent gets its own clean context window scoped to its subtask, preventing context pollution and allowing much larger effective research/work volume than a single agent's context would allow.
- **Communication protocol** — how agents exchange results (structured tool-call results, a shared MCP server, direct message passing). Choice affects auditability and latency.
- **Failure handling and redundancy** — what happens when a subagent fails, times out, or produces a low-confidence result; does the orchestrator retry, escalate to a human, or proceed with partial results.
- **Cost multiplication** — running N subagents multiplies token spend roughly linearly (sometimes more, with overlapping context); the exam expects you to weigh this against the value of parallelism.

### Scenario example
*A research-assistant product needs to answer broad, open-ended questions ("compare the regulatory environment for autonomous vehicles across five countries") that require gathering and synthesizing information from many independent sources.*

Correct pattern: a **lead orchestrator agent** decomposes the question into independent per-country research subtasks, dispatches a **subagent per country** (each with its own clean context to search and summarize), and the lead agent synthesizes the five results into one answer. This is the correct multi-agent design because the subtasks are genuinely independent and parallelizable, and isolating each country's research into its own context avoids one massive, cluttered context window.

*Contrast — where multi-agent is the wrong answer:* a customer asks a single, narrow factual question answerable in one lookup. Standing up multiple specialist agents for that is unnecessary orchestration overhead — a single augmented LLM call is correct, and "add a multi-agent system" is a distractor testing whether you over-apply the pattern you just learned.

### Distractor patterns to watch for
- **Multi-agent as default** — after learning multi-agent design, candidates over-select it on the exam for tasks that are actually sequential/dependent (where subtasks need each other's outputs) rather than independent/parallel — multi-agent orchestration adds the most value specifically when subtasks are independent; sequential dependency is often better served by a single agent or a prompt-chaining workflow.
- **Shared-context assumption** — an option assuming subagents automatically share full context/memory with each other is usually wrong; the value proposition of multi-agent is *isolated* context per subagent, with the orchestrator responsible for synthesis.
- **Ignoring cost tradeoffs** — an option that recommends multi-agent purely for "better quality" without acknowledging the multiplied token cost and added latency is incomplete for a Professional-level answer; the exam wants cost-awareness baked into the recommendation, not bolted on.
- **No failure-handling plan** — an architecture description with no mention of what happens if a subagent fails or returns a low-confidence result is a common gap the exam probes for.

---

## 1e. Applying Decomposition Techniques for Complex Problem Solving

### What this really tests
The ability to break one large, ambiguous task into the *right shape* of smaller units — matching the decomposition style to the actual dependency structure of the problem, not applying one favorite technique everywhere.

### Decomposition styles
- **Sequential decomposition** — steps that must happen in order because each depends on the prior step's output (research → draft → edit → format). Maps to prompt chaining.
- **Parallel/independent decomposition** — subtasks that don't depend on each other and can run concurrently (checking 12 independent contract clauses). Maps to parallelization.
- **Hierarchical decomposition** — a top-level task broken into sub-goals, each of which may itself be decomposed further, typically coordinated by an orchestrator. Maps to orchestrator-workers or multi-agent designs.
- **Conditional/routing decomposition** — the task branches based on a classification decision, and only one branch is actually executed per input. Maps to the routing workflow pattern.

### Scenario example
*An architect must design a system that generates a quarterly business report: gather metrics from three internal systems, write a narrative summary, have a second pass check the narrative against the raw numbers for factual accuracy, then format the final document.*

Correct decomposition: metrics-gathering from three systems is **parallel** (independent sources) → narrative drafting is **sequential** (depends on gathered metrics) → the accuracy check is best modeled as an **evaluator-optimizer** loop (a second pass checking the first) → final formatting is a last sequential step. Recognizing that *one task can combine several decomposition styles* — rather than forcing the whole pipeline into a single pattern — is what separates a Professional-level answer from a Foundations-level one.

### Distractor patterns to watch for
- **One-pattern-fits-all** — an option that forces a task with clearly independent subtasks into a purely sequential chain (needlessly slower, no parallelism exploited) is a common distractor, as is the reverse: forcing genuinely dependent steps into parallel execution where a later step actually needs an earlier step's output.
- **Over-decomposition** — breaking a genuinely simple, single-step task into an elaborate multi-stage pipeline "to be thorough" — this violates the simplicity-first principle and is a frequent wrong-but-tempting answer.
- **Missing the verification step** — decomposition scenarios involving accuracy-sensitive output (financial, medical, legal) often have a distractor that omits any evaluator/check step, going straight from generation to output.

---

## 1f. Aligning Solutions to Business Value Pillars

### The five pillars, as the blueprint frames them
- **Efficiency** — doing the same work with less time/effort (automating a manual review step)
- **Transformation** — enabling a fundamentally new capability that wasn't feasible before (real-time personalized guidance at a scale no human team could match)
- **Productivity** — augmenting human output rather than replacing a process outright (a drafting assistant that lets each analyst handle more cases)
- **Cost** — reducing the direct cost of an operation (cheaper resolution per ticket)
- **Performance SLAs** — meeting a specific, contracted latency/availability/accuracy target (a support system that must respond within 3 seconds)

### Why this matters architecturally
Each pillar pulls architecture decisions in a different direction. A **cost**-optimized design favors cheaper models, aggressive caching, and routing simple requests away from expensive patterns. A **performance-SLA**-driven design favors low-latency patterns (a single augmented LLM call) over slower multi-agent research pipelines, even if the multi-agent version would be marginally more thorough. A **transformation** goal may justify a more expensive, more complex architecture that a pure cost-efficiency lens would reject, because the value is in the new capability, not incremental savings.

### Scenario example
*A telecom company has two initiatives: (1) a real-time customer chat assistant with a contractual 2-second response SLA, and (2) a back-office system that synthesizes overnight network-outage reports from thousands of log entries, with no strict time pressure.*

For (1), the architecture must prioritize the **performance SLA** pillar: a single fast augmented-LLM call or lightweight routing workflow, avoiding multi-agent orchestration or evaluator-optimizer loops that would blow the latency budget, even if those patterns would produce marginally higher-quality answers. For (2), the **efficiency/transformation** pillars dominate and latency is a non-issue, so a more elaborate orchestrator-workers pipeline processing thousands of logs overnight is entirely appropriate — the extra thoroughness has real value and no SLA is threatened.

### Distractor patterns to watch for
- **Ignoring the stated constraint** — the exam will explicitly state an SLA or cost ceiling in the scenario, then offer a "textbook best" architecture (e.g., full multi-agent evaluator-optimizer loop) that violates the *stated* constraint. Always check the scenario's numbers against the option's implied latency/cost before picking "the most sophisticated" answer.
- **Pillar mismatch** — an option justifies an expensive architecture using the wrong pillar (e.g., justifying it as "cost savings" when it's actually a transformation play, or vice versa) — the exam tests whether you can correctly identify *which* pillar a given business goal actually maps to, not just whether you can name the pillars.
- **One-size architecture** — assuming the same architecture is "correct" for every pillar in a multi-initiative scenario, rather than recognizing that different pillars for different initiatives justify genuinely different architectures.

---

## Cross-Cutting Distractor Pattern Reference

| Distractor pattern | What it looks like | How to catch it |
|---|---|---|
| **Over-engineering ("sounds sophisticated")** | The most complex-sounding option (multi-agent, evaluator-optimizer, full autonomous agent) offered for a problem with fixed, known, simple steps | Ask: could the simplest pattern (single augmented LLM, or a basic workflow) solve this? If yes, complexity is the wrong answer. |
| **Under-engineering (false simplicity)** | A single-call or fixed-chain answer offered for a problem whose steps genuinely can't be predicted in advance | Ask: does solving this require the system to decide its *own* next step based on what it just learned? If yes, it needs agentic capability. |
| **Right pattern, wrong justification** | The architecturally correct choice, paired with a wrong or irrelevant reason in the answer text | Check that the *reasoning*, not just the label, matches the scenario. |
| **Ignoring the stated constraint** | Answer violates an explicit SLA, budget, or compliance detail stated in the scenario stem | Re-read the stem for numbers/constraints before comparing options. |
| **Human-in-the-loop omission** | High-stakes/irreversible action handed fully to the model with no review step | Flag any option that removes human review from a scenario involving money, legal, medical, or safety outcomes. |
| **Feedback loop dropped** | An otherwise-complete architecture with no mechanism for output quality to influence future iterations | Check for logging + review + prompt/example iteration explicitly, not just input→output. |
| **Pattern name swap** | Parallelization described as if it were orchestrator-workers, or routing described as if it were chaining | Know the precise mechanism definition for each of the five workflow patterns, not just their names. |

---

## Comparison Charts

### Chart 1 — Workflow Patterns at a Glance

| Pattern | Steps known in advance? | Subtask relationship | Typical use case | Key risk if misapplied |
|---|---|---|---|---|
| Prompt chaining | Yes | Sequential, dependent | Draft → translate; outline → expand | Forcing independent work through unnecessary sequential steps (slower than needed) |
| Routing | Yes | Mutually exclusive branches | Support ticket triage to specialist prompts | Misclassification sends input down the wrong branch entirely |
| Parallelization (sectioning) | Yes | Independent, concurrent | Checking N independent risk clauses at once | Using it when subtasks actually depend on each other |
| Parallelization (voting) | Yes | Same task, repeated | Consensus check on a sensitive classification | Cost multiplies with each repetition; only justified when accuracy stakes are high |
| Orchestrator-workers | Decided dynamically by orchestrator | Dynamically delegated | Multi-file code change where scope isn't known until analysis | Over-applying when the decomposition is actually fixed and could be a simpler workflow |
| Evaluator-optimizer | Yes (loop count may vary) | Generator + critic loop | Iterative refinement against explicit quality criteria | Looping indefinitely without a stopping condition, or using it where one pass is already sufficient |

### Chart 2 — Workflow vs. Agent vs. Augmented LLM

| Aspect | Augmented LLM | Workflow | Agent |
|---|---|---|---|
| What decides the path | N/A — it's the base unit, single call | The engineer, in code, in advance | The LLM, dynamically, at runtime |
| Predictability | Highest | High | Lower |
| Best fit | Single-step task needing retrieval/tools/memory | Task with a fixed, known set/order of steps | Task where step count/order can't be predicted ahead of time |
| Cost/latency profile | Lowest | Low–moderate, scales with number of steps | Highest, variable |
| Oversight/auditability | Easiest to audit | Easy — path is explicit in code | Harder — path emerges from model decisions |
| Example | Answer a question using a search tool | Route ticket → one of 3 specialist prompts | Open-ended research task deciding its own next search |

### Chart 3 — Multi-Agent Orchestration Topologies

| Topology | How it works | Best fit | Watch-out |
|---|---|---|---|
| **Orchestrator + subagents** | Lead agent decomposes task, dispatches independent subagents, synthesizes results | Broad research/analysis tasks with independent, parallelizable subtopics | Synthesis step becomes a bottleneck if subagent outputs conflict |
| **Peer-to-peer handoff** | One agent completes its portion then hands off directly to the next specialist agent | Sequential specialist workflows (e.g., intake agent → diagnosis agent → resolution agent) | No central view of overall progress; harder to recover from a dropped handoff |
| **Single agent (no multi-agent)** | One agent, one context, handles the whole task itself | Narrow tasks, tightly dependent steps, or when cost/latency budget is tight | The default choice — only move to multi-agent when parallel independence or context isolation is demonstrably needed |

---

## Mock Exam — 20 Questions (Domain 1, Professional Difficulty)

*Format matches the real exam: each item states how many responses to select. Difficulty is calibrated to Professional level — expect close, plausible distractors, not obviously-wrong options.*

---

**Q1 (1a).** A pharmaceutical company wants a system to summarize clinical trial adverse-event reports (free-text narratives from investigators) and flag reports that meet a specific, legally-defined severity threshold requiring regulatory reporting within 24 hours. Which architectural framing is most appropriate? *(Select ONE)*

A. A fully autonomous agent that reads reports, determines severity, and files the regulatory submission without review
B. An augmented LLM that summarizes and flags candidate reports against the threshold criteria, routed to a human reviewer for final determination and filing
C. A rules-based keyword search only, with no LLM involvement, since severity thresholds are legally defined
D. A multi-agent system where five agents independently vote on severity, majority result auto-files without review

**Answer: B**
**Explanation:** The narrative interpretation (unstructured text → severity judgment) is a strong Claude fit, but the legally-defined regulatory filing is a high-stakes, irreversible, compliance-bound action — it requires human-in-the-loop review before filing (this is the correct boundary between Domain 1 solution framing and Domain 5 governance). A is wrong because it removes required human oversight from a regulated, irreversible action. C is wrong because it ignores the genuine judgment/unstructured-text component that keyword search can't reliably capture (nuanced clinical language rarely maps to fixed keywords). D is a distractor combining two attractive-sounding but wrong ideas — multi-agent voting is unnecessary complexity for this problem, and "auto-files without review" repeats the same governance violation as A.

---

**Q2 (1a).** A logistics firm asks you to design "a Claude-based system to calculate optimal delivery routes given traffic, weather, and driver hours-of-service regulations." Which statement best evaluates this request? *(Select ONE)*

A. This is an ideal Claude use case since it involves complex multi-variable reasoning
B. Route optimization is a deterministic constrained-optimization problem; Claude is better used to interpret ambiguous inputs (e.g., unusual driver notes) and orchestrate calls to a dedicated routing/optimization engine as a tool, rather than compute the routes itself
C. Claude should not be involved at all in any part of this system
D. Claude should generate several candidate routes via chain-of-thought reasoning and pick the best one directly

**Answer: B**
**Explanation:** Multi-constraint route optimization is precisely the kind of deterministic, mathematically well-defined problem better solved by a purpose-built optimization engine; Claude's correct role is as an interface/orchestrator (interpreting messy inputs, calling the optimizer as a tool, explaining results) rather than performing the optimization itself. A over-applies Claude to a problem better solved deterministically. C under-applies it — there's real value in Claude handling the ambiguous surrounding inputs. D is a subtle trap: asking an LLM to reason its way to "the best route" via chain-of-thought produces a plausible-sounding but not-guaranteed-optimal or verifiable result for a problem that has an actual mathematically correct answer.

---

**Q3 (1a).** Which of the following business problems is the WEAKEST candidate for a Claude-based solution as the primary decision-maker? *(Select ONE)*

A. Classifying inbound emails into one of 8 departments based on free-text content
B. Determining final loan interest rate purely from a fixed formula based on credit score, loan amount, and term
C. Drafting a first-pass response to a customer complaint for agent review
D. Summarizing a lengthy contract into key terms for a paralegal's review

**Answer: B**
**Explanation:** B is pure deterministic arithmetic from fixed inputs — there is no judgment or unstructured-input component, so it should be a direct formula/rules calculation, not an LLM decision (though Claude could present the result). A, C, and D all involve unstructured text and/or judgment with a human still in the loop for the final action, making them appropriate Claude use cases.

---

**Q4 (1b).** An architecture for an AI-assisted claims-processing system includes: document ingestion and normalization, a retrieval step pulling policy terms, an augmented-LLM call producing a draft coverage determination, and delivery of that determination directly to the customer via automated email. What is the most significant architectural gap? *(Select ONE)*

A. No memory component in the augmented LLM
B. No feedback loop or output validation step before the determination reaches the customer
C. No routing pattern used to select which policy to retrieve
D. No use of prompt chaining

**Answer: B**
**Explanation:** The architecture goes straight from model output to customer-facing delivery with no guardrail/validation check and no mechanism for errors to be caught, logged, or fed back into future iterations — a serious gap for a decision with real customer and compliance impact. A, C, and D describe missing implementation choices that may or may not be necessary depending on the scenario, but none represent as fundamental a lifecycle gap as the missing output-validation/feedback-loop stage.

---

**Q5 (1b).** In an end-to-end architecture, what is the PRIMARY purpose of the feedback loop stage? *(Select ONE)*

A. To retrain the underlying Claude model on the enterprise's proprietary data
B. To capture output quality signals (human corrections, confidence scores, downstream outcomes) and use them to iteratively improve prompts, examples, or evaluation datasets
C. To provide the model with more context window space for the next request
D. To reduce API latency on subsequent calls

**Answer: B**
**Explanation:** For most enterprise Claude architectures, architects don't control retraining the base model (A is a distractor conflating "improvement" with fine-tuning, which is generally outside the architect's control and not how iteration typically happens with hosted Claude models). The feedback loop's real purpose is closing the loop between observed output quality and iterative prompt/example/evaluation improvement. C and D describe unrelated technical effects, not the purpose of a feedback loop.

---

**Q6 (1b).** A team designs a system where user input is validated and normalized, sent to an augmented LLM, and the raw model output is returned directly to the calling application with no further processing. Later, they add a step that checks the output against a structured schema and business rules before returning it. What architectural principle does this second change reflect? *(Select TWO)*

A. Output validation reduces the risk of malformed or policy-violating responses reaching production systems
B. It converts the workflow into a multi-agent system
C. It closes a gap in the standard input→processing→output→feedback lifecycle
D. It eliminates the need for a feedback loop entirely

**Answer: A, C**
**Explanation:** Adding schema/business-rule validation on the output is exactly the "output" stage of the standard lifecycle and directly reduces the risk of bad output reaching downstream systems (A and C are both correct and are two ways of describing the same improvement). B is wrong — validating output doesn't create additional autonomous agents. D is wrong and a common trap — output validation and feedback loops are separate stages; adding one does not eliminate the need for the other.

---

**Q7 (1c).** A team needs to translate a 40-page technical manual into five languages, where each translation is independent of the others and the number of steps (one call per language) is known in advance. Which pattern is most appropriate? *(Select ONE)*

A. Orchestrator-workers
B. Parallelization (sectioning)
C. Evaluator-optimizer
D. Fully autonomous agent

**Answer: B**
**Explanation:** Five independent translation tasks with a fixed, known step count is the textbook definition of parallelization (sectioning) — divide the work, run concurrently, aggregate. A (orchestrator-workers) is a close distractor because it also involves delegation, but orchestrator-workers implies the orchestrator *dynamically decides* the decomposition at runtime; here the decomposition (5 languages) is already fixed and known, so plain parallelization is simpler and sufficient. C is wrong — there's no generator/critic relationship described. D is over-engineering a fixed, known-step task.

---

**Q8 (1c).** A support ticket first needs to be classified as "billing," "technical," or "account access," and each category is then handled by a differently-tuned prompt specialized for that category. Which pattern does this describe? *(Select ONE)*

A. Prompt chaining
B. Routing
C. Parallelization (voting)
D. Orchestrator-workers

**Answer: B**
**Explanation:** This is the definition of routing: an initial classification step directs input to one of several specialized, mutually exclusive downstream paths. A is wrong because chaining implies sequential dependent steps, not branching to one of several exclusive paths. C is wrong — voting means running the same task multiple ways for consensus, not branching to different specialized handlers. D is wrong — there's no dynamic task decomposition or delegation to multiple concurrent workers here, just a single branch selection.

---

**Q9 (1c).** Which scenario most clearly justifies escalating from a workflow to a full autonomous agent, according to Anthropic's own design guidance? *(Select ONE)*

A. The task has exactly three fixed steps that always occur in the same order
B. The task's number and sequence of steps cannot be reliably predicted in advance, but the task has clear success criteria and a feasible feedback mechanism from the environment
C. The task is high-volume and needs to run at low latency
D. The task must be extremely auditable for compliance purposes

**Answer: B**
**Explanation:** This is precisely Anthropic's stated rationale for agents over workflows: unpredictable step count/order, combined with clear success criteria and available ground-truth feedback (e.g., tool results), justifies letting the model dynamically direct its own process. A describes a workflow's defining characteristic, not a reason to escalate to an agent — a fixed, known step sequence is better served by a workflow. C and D are actually arguments *against* choosing an agent — agents tend to have higher, more variable latency and are harder to audit than an explicit code-defined workflow.

---

**Q10 (1c).** An architect proposes using an evaluator-optimizer pattern for a system that generates a single, simple auto-reply confirming receipt of a customer email, with no substantive content requiring quality judgment. What is the best critique of this proposal? *(Select ONE)*

A. Evaluator-optimizer requires at least three LLM calls, which this system cannot support
B. This is over-engineering — a single augmented LLM call (or simple template) already fully solves a task with no meaningful quality dimension to iteratively improve, so the added latency and cost of a generator/critic loop is unjustified
C. Evaluator-optimizer can only be used for code-generation tasks
D. Evaluator-optimizer is only valid when combined with routing

**Answer: B**
**Explanation:** The core Domain 1 principle — favor the simplest pattern that solves the problem — directly applies here. A trivial confirmation reply has no substantive quality dimension worth an iterative critic loop, so evaluator-optimizer adds cost and latency without benefit. A, C, and D are all fabricated technical constraints that don't reflect how the pattern actually works.

---

**Q11 (1d).** A company builds a multi-agent system where a lead agent dispatches five subagents to independently research five competitor products, then synthesizes their findings into a comparison report. What is the PRIMARY architectural benefit of giving each subagent its own isolated context, rather than having one agent research all five sequentially in a single context? *(Select ONE)*

A. It guarantees the final report will be shorter
B. It prevents context pollution across unrelated research threads and allows substantially more total research volume than a single context window could hold before synthesis
C. It eliminates the need for the lead agent to synthesize results
D. It guarantees each subagent produces an identical writing style

**Answer: B**
**Explanation:** Context isolation is the core value proposition of multi-agent research designs — each subagent researches its own product without irrelevant details from the other four cluttering its context, and the total research volume across five clean contexts can far exceed what a single context window could hold. A is not a guaranteed or primary benefit. C is wrong — synthesis is still required and is the lead agent's job. D is wrong and not a benefit of isolation at all — style consistency isn't inherent to context isolation.

---

**Q12 (1d).** A customer-service scenario requires: intake and categorization of a request, followed by an eligibility check against account records, followed by execution of the approved action — where each stage depends entirely on the previous stage's output and there's no parallel work available. Which orchestration topology is most appropriate? *(Select ONE)*

A. Orchestrator + independent parallel subagents
B. Peer-to-peer handoff between sequential specialist agents (or a single agent/workflow performing the sequential steps)
C. Five-agent majority-voting ensemble
D. A fully decentralized swarm with no defined handoff protocol

**Answer: B**
**Explanation:** Because each stage strictly depends on the prior stage's output, there's no independent, parallelizable work here — the orchestrator+parallel-subagent topology (A) is designed for independent subtasks, which doesn't match this scenario. A sequential handoff model (or even a single agent/workflow, since the steps are fixed and known) fits the dependency structure. C is irrelevant — there's no repeated task needing consensus. D introduces uncontrolled complexity with no benefit for a linear dependency chain.

---

**Q13 (1d).** Which of the following is the most complete list of factors an architect must weigh before choosing a multi-agent design over a single-agent design? *(Select TWO)*

A. Whether the subtasks are genuinely independent and parallelizable, versus sequentially dependent
B. The multiplied token/cost overhead of running multiple concurrent agents versus the value gained from parallelism and context isolation
C. Whether the company's brand color scheme is included in the system prompt
D. Whether the multi-agent framework has a trendy name

**Answer: A, B**
**Explanation:** These are the two substantive architectural tradeoffs the exam expects: task independence/parallelizability (A) and cost/value tradeoff (B). C and D are irrelevant distractors with no bearing on the architectural decision.

---

**Q14 (1d).** A multi-agent orchestration design has no defined behavior for what happens when a subagent times out or returns a low-confidence result. From an architecture standpoint, what is the correct characterization of this design? *(Select ONE)*

A. It is complete, since agents are expected to always succeed
B. It has a failure-handling gap that should be addressed before production deployment — e.g., retry logic, escalation to a human reviewer, or proceeding with a flagged partial result
C. This gap only matters for single-agent systems, not multi-agent systems
D. Failure handling is exclusively a Domain 4 (evaluation/testing) concern and has no bearing on Domain 1 architecture design

**Answer: B**
**Explanation:** Robust architecture design must account for failure modes as part of the initial design, not as an afterthought — this is a core Domain 1 "design end-to-end architecture" expectation. A is wrong — agents (and any distributed system) can and do fail or time out. C is backwards — failure handling matters more, not less, in multi-agent systems, since more independent components means more potential failure points. D is a trap: while failure diagnosis is also tested in Domain 4, designing for failure handling as part of the initial architecture is squarely a Domain 1 concern too — domains overlap and the exam does test that overlap.

---

**Q15 (1e).** A task requires: (1) pulling data from three unrelated internal systems, (2) writing a narrative summary from that combined data, (3) a second-pass check of the narrative against the raw data for factual errors, (4) final formatting. Which decomposition correctly matches each stage to its dependency structure? *(Select ONE)*

A. All four stages should be run in parallel to save time
B. Stage 1 is parallel/independent; stage 2 is sequential (depends on stage 1); stage 3 is an evaluator-optimizer-style check (depends on stage 2's output); stage 4 is sequential (depends on stage 3)
C. All four stages must be handled by a single fully autonomous agent with no structured decomposition
D. Stages 1 through 4 should each be handled by a separate, independently voting five-agent ensemble

**Answer: B**
**Explanation:** This correctly matches each stage to its actual dependency structure: independent data pulls can run in parallel, the narrative draft depends on the gathered data (sequential), the accuracy check is a generator/critic relationship (evaluator-optimizer), and formatting depends on the corrected narrative (sequential). A is wrong — stages 2-4 all have real dependencies on prior stages and cannot run in parallel. C over-engineers a task whose steps are actually fixed and knowable in advance — no dynamic agentic planning is needed. D applies unnecessary, costly voting ensembles to steps that don't need consensus-based repetition.

---

**Q16 (1e).** What is the most accurate description of "over-decomposition" as an architectural anti-pattern? *(Select ONE)*

A. Breaking a task into more, smaller stages than its actual complexity or dependency structure warrants, adding latency, cost, and coordination overhead without a corresponding benefit
B. Using too few LLM calls for a task that requires many independent parallel subtasks
C. Failing to use any decomposition at all, regardless of task complexity
D. Using routing when parallelization would have been correct

**Answer: A**
**Explanation:** Over-decomposition specifically means adding unnecessary structural complexity/stages beyond what the task's actual dependency structure requires. B describes under-decomposition (the opposite problem). C describes a lack of decomposition, not over-decomposition. D describes a pattern-selection error, not a decomposition-granularity error — a subtly different failure mode.

---

**Q17 (1e).** A scenario states a task is "simple: extract the shipping address from this one email." Which architectural response best reflects sound decomposition judgment? *(Select ONE)*

A. Decompose into: classify email type → route to address-extraction prompt → run an evaluator-optimizer loop → run a five-way voting ensemble → format final output
B. A single augmented LLM call (or simple extraction prompt) is sufficient; no further decomposition is warranted for a single, simple extraction task
C. Deploy a full multi-agent orchestration system to future-proof the design
D. Decomposition is mandatory for every task regardless of complexity, per best practice

**Answer: B**
**Explanation:** The core "simplest solution that solves the problem" principle applies directly — a single, simple extraction task needs a single call, not an elaborate pipeline. A is a textbook over-decomposition distractor stacking unnecessary patterns onto a trivial task. C is over-engineering under the guise of "future-proofing," which the exam treats as an unjustified complexity increase absent stated future requirements. D is a false absolute — decomposition is a tool applied when task structure warrants it, not a mandatory step for every task.

---

**Q18 (1f).** A company's real-time fraud-detection chat interface has a contractual SLA requiring a response within 1.5 seconds. An architect proposes a design using an orchestrator dispatching three subagents plus an evaluator-optimizer refinement loop, arguing it will produce the most thorough possible fraud assessment. What is the correct critique? *(Select ONE)*

A. The design is correct because thoroughness is always the top priority in fraud detection
B. The design very likely violates the stated performance SLA — multi-agent orchestration plus an iterative refinement loop introduces latency incompatible with a 1.5-second requirement, and a faster single-call or lightweight pattern should be prioritized to meet the explicit constraint
C. SLAs are a Domain 6 concern only and should not affect Domain 1 architecture choices
D. The design is correct as long as the subagents run on the fastest available Claude model

**Answer: B**
**Explanation:** This tests the "ignoring the stated constraint" distractor pattern directly — a technically more thorough design that violates an explicit, contractually-stated SLA is the wrong answer; the correct architecture must satisfy the stated performance constraint, even if it means sacrificing some thoroughness. A ignores the explicit constraint given in the stem. C is wrong — aligning architecture to SLAs is explicitly named in Domain 1's own sub-objective 1f ("performance SLAs" is one of the business value pillars Domain 1 tests). D is a distractor — even the fastest model can't overcome the inherent added latency of multi-agent orchestration plus an iterative refinement loop within 1.5 seconds; model speed alone doesn't fix an architecturally latency-heavy design.

---

**Q19 (1f).** A nonprofit wants a system that lets caseworkers handle triple the client volume by having Claude draft case notes and eligibility summaries for caseworker review, without replacing the caseworkers' judgment. Which business value pillar does this initiative primarily target? *(Select ONE)*

A. Cost reduction, since caseworker headcount will be reduced
B. Productivity — augmenting each caseworker's output/throughput while keeping human judgment central to the process
C. Transformation, since it enables an entirely new capability that did not exist before
D. Performance SLA, since it is primarily about meeting a specific latency target

**Answer: B**
**Explanation:** The described initiative is explicitly about augmenting human throughput (caseworkers handling more volume) while preserving human judgment — this is the definition of the productivity pillar, not cost (the scenario doesn't state headcount reduction — inferring it is a distractor trap), not transformation (case management itself isn't a new capability, just faster/higher-volume), and not an SLA (no latency target is mentioned).

---

**Q20 (1f).** Two initiatives are being scoped: Initiative X needs to process overnight batch reports with no time pressure and prioritizes maximum thoroughness; Initiative Y is a live chat assistant with a strict 2-second SLA. Which statement best reflects correct architectural alignment to business value pillars across both initiatives? *(Select ONE)*

A. Both initiatives should use the identical architecture, since consistency across the company is more important than tailoring to each initiative's constraints
B. Initiative X can justify a more elaborate, higher-latency pattern (e.g., orchestrator-workers or evaluator-optimizer) since thoroughness is prioritized and latency is a non-issue; Initiative Y should prioritize a fast, low-latency pattern (e.g., a single augmented LLM call or lightweight routing) to meet its performance SLA, even at some cost to thoroughness
C. Initiative Y should always take architectural priority over Initiative X regardless of stated constraints
D. Since both initiatives use Claude, the same business value pillar (efficiency) applies equally to both and should drive both designs

**Answer: B**
**Explanation:** This is the core Domain 1f lesson: different initiatives can and should map to different pillars, and architecture should be tailored to each initiative's actual stated constraints and priorities, not standardized for its own sake. A enforces false consistency at the expense of fitting each initiative's real constraints. C asserts an unjustified priority ranking not supported by anything in the scenario. D incorrectly assumes a single pillar applies uniformly — X is really about thoroughness/quality (arguably transformation or productivity) and Y is explicitly about a performance SLA; treating both as "efficiency" ignores the actual stated goals of each.

---

## Quick-Reference Summary Table (All Sub-Objectives)

| Sub-objective | One-line test | Golden rule |
|---|---|---|
| 1a | Should this even be a Claude solution? | Judgment/unstructured input → good fit; deterministic math/lookup → tool or rules engine, human stays in loop for irreversible high-stakes actions |
| 1b | Did you architect the whole lifecycle? | Input → Processing → Output (with validation) → Feedback loop — never stop at "the model returns an answer" |
| 1c | Did you pick the simplest sufficient pattern? | Known/fixed steps → workflow (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer); unpredictable steps + clear success criteria + feedback from environment → agent |
| 1d | Does multi-agent actually earn its cost here? | Independent, parallelizable subtasks + need for context isolation → multi-agent; sequential dependency → single agent/workflow; always plan for subagent failure |
| 1e | Does your decomposition match the real dependency structure? | Match style to structure: parallel for independent work, sequential for dependent work, hierarchical/orchestrated for nested sub-goals, routing for branching — don't force one style everywhere |
| 1f | Did you architect to the stated business constraint, not just "best practice"? | Efficiency / transformation / productivity / cost / performance SLA each pull design differently — check the scenario's explicit numbers (SLA, budget) before picking the most sophisticated-sounding option |

---

*Prepared as independent study material for CCAR-P exam preparation. Domain weight, sub-objective wording, and exam format facts are sourced from Anthropic's published Exam Guide v1.0 as summarized at claudecertificationguide.com/ccar-p. Explanatory content, scenarios, and mock questions are original material built on Anthropic's publicly published agentic-architecture guidance and are not sourced from, or claimed to be, official exam questions.*

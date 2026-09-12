# CCAR-P Study Guide — Domain 2: Claude Models, Prompting & Context Engineering (13%)

**Source grounding:** Sub-objectives 2a–2e below are confirmed by two independent sources — `claudecertificationguide.com/ccar-p` and `claudearchitectcertification.com/certifications/ccar-p` — which agree on the domain's 13% weight and its five sub-objectives. Neither site publishes full worked lesson content for this domain (one marks its prep track "coming soon," the other links out to short concept pages rather than exam-length material). Where those concept pages gave verifiable, stable mechanics (e.g., how prompt caching works structurally), I've used them. Where sources disagreed with each other on volatile specifics — exact model version numbers, exact pricing, exact cache TTLs — I did **not** import those numbers, because a web check turned up materially conflicting figures across sources (some clearly low-quality SEO content). Model-tier names below follow Anthropic's confirmed current lineup rather than the noisiest web results. Treat any exact price or TTL you see elsewhere as something to verify against `docs.claude.com` before an exam attempt or production decision — the underlying *architectural principles* below are what's actually stable and testable.

**Domain 2 sub-objectives (official blueprint):**
- **2a.** Select appropriate Claude models based on trade-offs
- **2b.** Design system prompts, templates, and guardrails
- **2c.** Apply prompt engineering techniques (zero-shot, few-shot, chain-of-thought)
- **2d.** Optimise context windows and manage token usage
- **2e.** Implement prompt reuse strategies (caching, modular prompts, Skills)

At 13% weight and 63 total items, expect roughly **8 Domain 2 questions** on the real exam.

---

## 2a. Selecting Appropriate Claude Models Based on Trade-offs

### What this really tests
Whether you treat model selection as an *engineering decision with three competing axes* — capability, latency, and cost — rather than reflexively picking "the smartest model" or "the cheapest model." Anthropic's current lineup runs roughly: a **fast/cheap tier** (Haiku) for high-volume, low-complexity work; a **balanced default tier** (Sonnet) that covers most production traffic; a **higher-capability tier** (Opus) for complex, long-horizon reasoning and agentic work; and a **top/frontier tier** (Fable) reserved for work the other tiers demonstrably can't handle. The exam's core principle: **start at the cheapest tier that plausibly covers the task, benchmark on real data, and upgrade only on a measured capability gap** — never on intuition alone.

A second, frequently tested idea: within a single model, there are often cheaper levers than switching tiers entirely — e.g., adjusting how much reasoning effort/extended thinking a call uses. Tuning that dial is usually cheaper to reason about and re-test than a full tier migration. A third idea: model versions are pinned snapshots, not evergreen pointers — behavior doesn't silently change under a deployed application, so adopting a newer version is a deliberate migration (re-test required), not an automatic upgrade.

### Scenario example
*A company runs two Claude-based systems: (1) classifying 50,000 inbound support emails per day into one of six categories, and (2) an internal engineering agent that autonomously investigates and fixes complex, multi-file bugs over long sessions.*

Correct alignment: system (1) is high-volume and low-complexity — the fast/cheap tier is very likely sufficient, and running it on a higher-capability tier would be a pure cost/latency loss with no measurable quality gain. System (2) is long-horizon, complex, and benefits from the deepest reasoning available — a higher-capability tier is justified here, and only here. An architect who proposes the same tier for both is not applying a trade-off analysis at all.

### Distractor patterns to watch for
- **"Most capable = best" trap** — an option recommending the top-tier model "to maximize quality" for a simple, high-volume, low-stakes task ignores the cost/latency cost and the fact that quality on simple tasks plateaus well below the top tier.
- **"Cheapest is always right" trap** — the mirror-image error: defaulting to the cheapest tier for a task that has a demonstrated capability gap (e.g., the cheap tier's classification accuracy is measurably too low) is also wrong; cost minimization without a benchmark is not a valid trade-off analysis.
- **Reflexive tier upgrade** — jumping straight to a bigger model when the actual fix is tuning reasoning effort, improving the prompt, or fixing a retrieval issue is a common distractor; the exam wants "cheapest fix first" reasoning, not "biggest model first."
- **Assuming automatic upgrades** — an option assuming a model "gets smarter over time automatically" once deployed misunderstands how pinned model versions work and is a factual trap, not just a design-judgment one.

---

## 2b. Designing System Prompts, Templates, and Guardrails

### What this really tests
Whether you treat the system prompt as the architectural contract for the whole interaction — role, scope, tone, output format, and boundaries — and whether you understand that a system-prompt instruction is necessary but never sufficient as a safety mechanism on its own.

- **System prompts** define persona, scope, output contract (format, length, structure), and behavioral boundaries (what the assistant will and won't do).
- **Templates** are parameterized, reusable prompt structures — the same underlying scaffold instantiated with different variables across many similar tasks, so behavior stays consistent without hand-writing a new prompt per case.
- **Guardrails** are the layered controls that keep the system safe and on-scope: input validation/sanitization, output validation (schema or policy checks *after* generation), explicit escalation/refusal instructions, and — critically — verification that happens outside the model's own text generation, not just an instruction telling the model to behave.

### Scenario example
*An enterprise deploys a Claude-based HR assistant. It must answer benefits questions using only the company's official policy documents, refuse to give individualized legal or medical advice, and escalate anything about a harassment complaint directly to a human HR rep.*

A well-designed system prompt states the assistant's scope (benefits Q&A only, grounded in provided policy documents), explicit refusal/escalation instructions for out-of-scope topics, and the required output format. But the *architecture* — not just the prompt — must also include an output-side guardrail that checks responses against sensitive-topic patterns before they reach the employee, because relying solely on "the system prompt says to escalate harassment topics" gives no guarantee the model will catch every phrasing of a harassment disclosure. The correct answer pairs the system prompt with an independent verification layer.

### Distractor patterns to watch for
- **System prompt as the only safety layer** — an option describing "comprehensive guardrails" that consists entirely of instructions inside the system prompt, with no independent output-side check, is architecturally incomplete — this is one of the most common Domain 2 distractors and mirrors the Domain 5 governance boundary.
- **Over-rigid templating** — a system prompt so long and rule-dense that it fights the model's natural behavior (excessive constraints, contradictory instructions) rather than working with it — an exam distractor may present this as "maximally safe" when it actually degrades reliability and consistency.
- **Guardrails confused with model choice** — an option claiming "using the most capable model" is itself a guardrail; capability and safety-control are separate concerns — a highly capable model with no output validation is still unguarded.

---

## 2c. Applying Prompt Engineering Techniques (Zero-Shot, Few-Shot, Chain-of-Thought)

### What this really tests
Matching the *technique* to the actual failure mode you're trying to prevent — not applying the most elaborate-sounding technique to every task.

| Technique | Mechanism | Best fit | Cost |
|---|---|---|---|
| **Zero-shot** | Direct instruction, no worked examples | Simple, well-understood tasks where the model's default interpretation is already reliable | Lowest token cost, fastest |
| **Few-shot** | Instruction plus a small number of input/output example pairs | Tasks with a subtle or unusual output format/style the model wouldn't reliably infer on its own (a specific JSON schema, a house tone) | Extra tokens for the examples; consistency payoff |
| **Chain-of-thought** | Prompting the model to reason step-by-step before producing a final answer | Multi-step reasoning, calculation-adjacent, or multi-constraint tasks where jumping straight to an answer causes errors | Extra tokens and latency for the reasoning; must be separated from user-facing output if the reasoning shouldn't be shown |

### Scenario example
*A team needs Claude to output a strictly-formatted JSON object with five specific fields for every request, and separately needs Claude to work through a multi-step eligibility determination involving several interacting policy rules before giving a final yes/no answer with justification.*

For the JSON-formatting task: **few-shot** is the right technique — two or three example input/output pairs anchor the exact structure far more reliably than a text description of the schema alone (zero-shot risks format drift; chain-of-thought adds cost with no benefit for a formatting-only task). For the eligibility determination: **chain-of-thought** is correct — walking through each rule before concluding measurably reduces errors on multi-constraint reasoning, and the intermediate reasoning can be kept separate from the final user-facing justification.

### Distractor patterns to watch for
- **Chain-of-thought on trivial tasks** — proposing step-by-step reasoning for a simple lookup or single-fact extraction adds latency and cost with no accuracy benefit; the exam tests whether you recognize when CoT is unearned complexity.
- **Zero-shot where format precision matters** — using a plain instruction with no examples for a task requiring an exact, unusual output structure is a common distractor when format drift is the actual failure mode described in the scenario.
- **Few-shot example bloat** — stacking far more examples than needed "to be thorough" wastes context/tokens without improving consistency once the pattern is already well anchored by 2–3 examples.
- **Reasoning leakage** — an option that shows the chain-of-thought reasoning directly to the end user without separating it from the final answer, when the scenario calls for a clean customer-facing response — this conflates a prompting technique with an output-formatting requirement.

---

## 2d. Optimising Context Windows and Managing Token Usage

### What this really tests
Whether you actively manage what goes into context — rather than assuming "a bigger context window" is a substitute for relevance filtering. Two separate but related skills: (1) fitting within a token budget, and (2) putting only what's actually useful into that budget.

- **Token budget management**: trimming irrelevant conversation history, summarizing older turns instead of carrying full transcripts forward indefinitely, and retrieving only the relevant chunks of a large corpus rather than loading entire documents when a small fraction is relevant.
- **Context quality, not just quantity**: stuffing a context window to its ceiling with everything potentially relevant is not automatically better — irrelevant or poorly-ordered content can degrade the model's ability to attend to what actually matters, and it always costs money and latency. A large context window is a *capacity*, not an obligation to fill it.
- **Note on scope vs. Domain 3**: this sub-objective is about *managing* the token budget and context relevance within a given interaction. The deeper question of progressive-disclosure vs. monolithic context strategy *for tool/integration design* is tested under Domain 3 (Integration) — the exam does test this boundary, so don't conflate "trim my conversation history" (Domain 2) with "how should an agent discover which of 200 available tools to load" (Domain 3).

### Scenario example
*A customer-support chat assistant's conversation history grows across a long support session, and after 40 turns the team notices response quality degrading and latency increasing.*

The correct fix isn't switching to a model with a larger context window and loading the entire transcript every turn — it's actively managing the budget: summarizing turns beyond a recent window into a compact running summary, retrieving only the KB passages relevant to the customer's *current* question rather than every article ever referenced in the conversation, and dropping resolved sub-threads that no longer affect the current issue.

### Distractor patterns to watch for
- **"Bigger window solves it" trap** — proposing a model upgrade purely for a larger context ceiling, without addressing what's actually being put into that context, treats a relevance problem as a capacity problem.
- **Full-history-always** — always appending the complete raw conversation history turn-by-turn as the default strategy, regardless of length, ignoring both cost and the attention-degradation risk of an overstuffed context.
- **Retrieval dumping** — pulling in entire source documents when only a small, identifiable portion is relevant to the current query, instead of targeted retrieval.

---

## 2e. Implementing Prompt Reuse Strategies (Caching, Modular Prompts, Skills)

### What this really tests
Whether you design for *reuse* of stable content across many calls/agents, rather than re-sending or re-authoring the same material repeatedly.

- **Prompt caching**: marking stable, frequently-reused content (a long system prompt, tool definitions, a large reference document, few-shot examples) so that repeated calls reusing that same content don't pay full processing cost each time. The architectural implication: **structure prompts with the most stable, shared content first/most static, and the mutable, per-request content last** — because caching keys off a shared, unchanged prefix. Caching pays off when the same large block is reused frequently within its active window; it does not help content that changes on every call.
- **Modular prompts**: decomposing a system prompt into composable sections (a shared company-policy block, a persona block, a task-specific block) that can be assembled per use case, instead of duplicating an entire prompt with minor variations across many agents or workflows.
- **Skills**: packaging reusable, task-specific instructions/procedures that Claude can load on demand for a given kind of work, rather than permanently loading every possible domain procedure into one always-resident system prompt. This enables progressive loading of only the expertise relevant to the current task.

### Scenario example
*A platform runs five specialized internal agents (HR, IT, Finance, Legal, Facilities) that all share the same company-wide compliance policy block (large and static) but each also need a small, agent-specific instruction set.*

The correct design: the shared compliance block is a single **modular, cacheable** component reused across all five agents (written once, cached once, reused repeatedly) rather than copy-pasted with drift risk into five separate system prompts; each agent's unique procedures are packaged as a **Skill** loaded only when that agent is invoked, instead of every agent's system prompt containing all five domains' full procedures "just in case." This maximizes both cache hit value (the large shared block stays identical across calls) and context efficiency (no agent loads instructions it doesn't need).

### Distractor patterns to watch for
- **Caching volatile content** — expecting cost/latency savings from caching content that's different on every call (e.g., the user's live message) misunderstands what caching actually reuses; only the stable, shared prefix benefits.
- **Monolithic "just in case" prompting** — loading every possible skill/instruction into one giant always-active system prompt instead of modularizing and loading on demand — this wastes tokens on every call regardless of whether that content is relevant, and is a direct violation of the "simplest sufficient design" principle tested across this exam.
- **Duplication instead of modularity** — copy-pasting a shared block into multiple agent prompts (drift risk when the policy changes and one copy isn't updated) instead of treating it as a single reusable, cacheable module.

---

## Cross-Cutting Distractor Pattern Reference

| Distractor pattern | What it looks like | How to catch it |
|---|---|---|
| **"More is better" (model, context, or examples)** | Picking the biggest model, the biggest context window, or the most few-shot examples "to be safe" | Ask: is there a measured gap the simpler/cheaper option actually fails to close? If not, the bigger option is unjustified cost. |
| **Instruction-only safety** | A guardrail described purely as "the system prompt tells it not to X" | Check for an independent, output-side verification step — an instruction alone is not a control. |
| **Technique/task mismatch** | Chain-of-thought on trivial tasks; zero-shot where exact formatting matters | Match the technique to the specific failure mode (reasoning errors → CoT; format drift → few-shot; simple/clear tasks → zero-shot). |
| **Capacity mistaken for a fix** | "Upgrade the model" or "use a bigger context window" proposed as the fix for a relevance/formatting/prompting problem | Ask whether the actual root cause is a budget/capacity problem or a "what's in the prompt" problem — they need different fixes. |
| **Reuse without structure** | Duplicating prompt content across agents instead of modularizing; caching content that changes every call | Check whether stable content is isolated and placed first, and whether what's cached is actually stable across calls. |

---

## Comparison Charts

### Chart 1 — Claude Model Tiers at a Glance (qualitative — verify exact current names/pricing at docs.claude.com)

| Tier | Relative speed | Relative cost | Best fit | Risk if misapplied |
|---|---|---|---|---|
| **Fast/cheap tier** (e.g., Haiku family) | Fastest | Lowest | High-volume, low-complexity, well-defined tasks (classification, extraction, simple chat) | Used for tasks needing deep multi-step reasoning → accuracy shortfall |
| **Balanced default tier** (e.g., Sonnet family) | Mid | Mid | The large majority of everyday production traffic — the sensible starting point absent a specific reason to deviate | Over- or under-used as a default without checking whether the task actually needs more or less |
| **Higher-capability tier** (e.g., Opus family) | Slower | Higher | Complex, long-horizon, agentic, or high-stakes reasoning work with a demonstrated need | Used by default for routine tasks → unnecessary cost/latency |
| **Top/frontier tier** (e.g., Fable family) | Slowest | Highest | Reserved for work the other tiers measurably cannot cover | Reached for without first establishing a genuine capability gap at the tier below |

### Chart 2 — Prompt Engineering Technique Comparison

| Technique | Adds examples? | Adds reasoning steps? | Primary benefit | Primary cost |
|---|---|---|---|---|
| Zero-shot | No | No | Lowest cost/latency for tasks the model already handles reliably | Can drift on unusual formats or edge cases |
| Few-shot | Yes | No | Anchors exact output format/style/structure | Extra tokens per call for the examples |
| Chain-of-thought | No (typically) | Yes | Reduces errors on multi-step/multi-constraint reasoning | Extra tokens and latency; reasoning must be separated from user-facing output if hidden |

### Chart 3 — Prompt Reuse Strategy Comparison

| Strategy | What's reused | Mechanism | Best fit | Key limitation |
|---|---|---|---|---|
| **Prompt caching** | Exact, stable content (system prompt, docs, tool defs) | Marks a stable prefix so repeated calls skip full reprocessing cost for that portion | High-frequency reuse of the same large, unchanged block within an active window | No benefit for content that changes every call; requires stable content ordered first |
| **Modular prompts** | Prompt *structure* (composable sections) | Assembles shared + task-specific blocks per use case at authoring time | Many agents/workflows sharing common policy/persona content with small per-case differences | Doesn't by itself reduce token cost — must be paired with caching for cost savings |
| **Skills** | Task-specific procedures/expertise | Loaded on demand for the task at hand, rather than always resident | Systems needing many different specialized procedures without permanently bloating every call's context | Requires a mechanism to correctly select which skill to load per task |

---

## Mock Exam — 20 Questions (Domain 2, Professional Difficulty)

*Format matches the real exam: each item states how many responses to select.*

---

**Q1 (2a).** A team runs a 24/7 pipeline classifying 200,000 short customer messages per day into one of four categories, and separately runs a research agent that autonomously plans and executes multi-hour investigations into complex technical incidents. Which statement reflects correct model-tier alignment? *(Select ONE)*

A. Both systems should run on the highest-capability tier available, since quality should never be compromised anywhere in the architecture
B. The classification pipeline should default to the fastest/cheapest tier that meets a benchmarked accuracy bar; the research agent's long-horizon, complex reasoning justifies a higher-capability tier
C. Both systems should run on the fastest/cheapest tier, since cost control is always the top priority
D. The tier choice should be identical for both because they're part of the same company's infrastructure and should be standardized

**Answer: B**
**Explanation:** This is the core trade-off principle: task complexity and stakes should drive tier selection, evaluated per-system against a benchmark, not a single company-wide default. A ignores cost/latency for a high-volume task that doesn't need top-tier capability. C ignores a genuine capability need for the long-horizon agent. D enforces false consistency instead of matching each system's actual requirements.

---

**Q2 (2a).** A team observes inconsistent output quality on a moderately complex task currently running on the balanced default tier. Before considering a tier upgrade, what is the recommended first step? *(Select ONE)*

A. Immediately migrate to the highest-capability tier available
B. Investigate whether increasing the reasoning effort/extended-thinking allocation within the current model resolves the inconsistency, since this is typically a cheaper and faster lever than a full tier migration
C. Reduce the prompt to zero-shot to simplify the request
D. Switch to the fastest/cheapest tier to see if cost drops

**Answer: B**
**Explanation:** Tuning available reasoning-effort controls within the same model is generally a cheaper, faster-to-test lever than migrating tiers, and the exam consistently rewards "cheapest fix first" reasoning. A skips a cheaper option. C removes a technique without diagnosing the actual problem. D addresses cost, not the stated quality issue.

---

**Q3 (2a).** Which statement about Claude model versions is most accurate regarding production deployments? *(Select ONE)*

A. Model IDs are evergreen aliases that automatically improve over time as Anthropic updates the underlying model
B. A given deployed model version behaves as a fixed, pinned snapshot; adopting a newer version is a deliberate migration requiring re-testing, not a silent automatic upgrade
C. Model behavior updates automatically unless a developer explicitly opts out
D. Only the fastest/cheapest tier receives pinned, stable versions — higher tiers update automatically

**Answer: B**
**Explanation:** Deployed model versions behave as fixed snapshots; upgrading is a deliberate, tested migration rather than something that happens silently underneath a running application. A, C, and D all describe an "automatic silent update" behavior that misrepresents how model versioning actually works.

---

**Q4 (2a).** A team's task-routing benchmark shows the fast/cheap tier scores well below an acceptable accuracy threshold on a nuanced classification task, while the balanced default tier clears the threshold comfortably. What is the correct interpretation? *(Select ONE)*

A. This proves cost should never be a factor in model selection
B. This is exactly the kind of measured capability gap that justifies moving up a tier for this specific task, despite the added cost
C. The team should keep the fast/cheap tier regardless, since cost minimization always wins
D. The benchmark result is irrelevant since model selection should be based on task type alone, not measured performance

**Answer: B**
**Explanation:** This is precisely the scenario where a tier upgrade is justified — a demonstrated, measured capability gap on the actual task, not intuition. A and C both ignore the evidence in favor of an absolute rule. D dismisses the benchmarking principle the exam explicitly rewards.

---

**Q5 (2b).** An architect describes a system's guardrails entirely as: "the system prompt instructs the assistant never to discuss competitor pricing." No other check exists. What is the correct critique? *(Select ONE)*

A. This is a complete and sufficient guardrail since the instruction directly addresses the risk
B. This is architecturally incomplete — a system-prompt instruction alone is not a guaranteed control; an independent output-side check is needed to catch cases where the instruction is not followed
C. The guardrail would only be complete if paired with a larger, more capable model
D. Guardrails are unnecessary here since competitor pricing is a low-risk topic

**Answer: B**
**Explanation:** This is the central Domain 2b lesson: instructions inside the system prompt are necessary but not sufficient — a robust design pairs them with independent verification on the output side. A accepts an incomplete design. C confuses model capability with safety-control design — they're separate concerns. D dismisses the risk without justification from the scenario.

---

**Q6 (2b).** Which best describes the distinct roles of "system prompt," "template," and "guardrail" in a well-designed architecture? *(Select ONE)*

A. They are three names for the same thing and can be used interchangeably
B. The system prompt defines role/scope/tone and the output contract; a template is the reusable, parameterized structure instantiated per request; a guardrail is a control (often independent of the prompt text) that verifies behavior stays within bounds
C. Templates and guardrails are only relevant for multi-agent systems, not single-agent architectures
D. Guardrails are a subset of the system prompt and cannot exist independently of it

**Answer: B**
**Explanation:** This correctly separates the three distinct architectural roles. A collapses meaningful distinctions the exam tests directly. C is an unsupported restriction — both concepts apply regardless of architecture size. D repeats the "guardrail = prompt instruction" misconception directly addressed in 2b's core lesson — guardrails are strongest specifically when they exist independently of the prompt text.

---

**Q7 (2b).** A system prompt for a regulated-industry assistant is expanded to include 40 dense, sometimes overlapping behavioral rules, on the theory that more explicit rules always produce safer, more reliable behavior. What is the most likely architectural risk? *(Select ONE)*

A. There is no risk — more explicit instructions always strictly improve reliability
B. Overloaded, potentially contradictory instructions can degrade consistency and fight the model's natural behavior, which is a different failure mode than having too few controls, and is best addressed by output-side verification rather than ever-more system-prompt rules
C. The only risk is increased token cost; behavior will be unaffected
D. This approach is guaranteed to be safer than any output-side guardrail, since it addresses the issue at the source

**Answer: B**
**Explanation:** Over-dense, potentially self-contradictory system prompts are a real reliability risk distinct from (and not solved purely by) adding more prompt-side rules — this is why the domain pairs system-prompt design with independent guardrails. A denies a real risk the exam tests. C understates the risk to just cost. D repeats the "prompt text alone is sufficient" misconception.

---

**Q8 (2c).** A task requires Claude to output a strict five-field JSON object on every call, and the team observes the model occasionally omits a field or uses inconsistent key names when given only a plain-text instruction describing the schema. What is the most appropriate fix? *(Select ONE)*

A. Add chain-of-thought reasoning instructions so the model reasons about the schema before answering
B. Provide 2–3 few-shot examples showing exact input/output pairs with the correct schema, to anchor the precise structure
C. Switch to a purely zero-shot approach with an even more detailed text description of the schema
D. Upgrade to the highest-capability model tier

**Answer: B**
**Explanation:** Format drift from a text-only schema description is the textbook case for few-shot examples — concrete input/output pairs anchor exact structure far more reliably than more prose description. A adds cost with no evidence it addresses a formatting (not reasoning) problem. C repeats the failing approach with more of the same. D treats a prompting-technique problem as a capability problem.

---

**Q9 (2c).** Which task most clearly justifies chain-of-thought prompting over zero-shot? *(Select ONE)*

A. Extracting a single named entity from a short, unambiguous sentence
B. Determining eligibility for a benefit that depends on five interacting numeric and categorical rules, where errors have occurred when the model answers directly
C. Formatting a response as a bulleted list
D. Translating a single short sentence into another language

**Answer: B**
**Explanation:** Multi-constraint reasoning with a documented error rate when answering directly is exactly where step-by-step reasoning measurably helps. A, C, and D are all simple, single-step tasks where chain-of-thought would add cost and latency without a corresponding accuracy benefit.

---

**Q10 (2c).** A customer-facing chat product uses chain-of-thought prompting, and the raw reasoning trace — including tentative, later-discarded conclusions — is currently displayed directly to the end customer along with the final answer. What is the architectural issue? *(Select ONE)*

A. There is no issue; showing the full reasoning process is always the most transparent and correct choice
B. The reasoning trace should typically be separated from the customer-facing output; showing tentative/discarded intermediate reasoning to the end user can confuse or mislead them even when the final answer is correct
C. Chain-of-thought should never be used in customer-facing products under any circumstances
D. The fix is to switch to few-shot prompting instead, which has no reasoning trace to manage

**Answer: B**
**Explanation:** This tests the "reasoning leakage" distractor directly — the technique (chain-of-thought) is likely appropriate for the underlying reasoning task, but the *output design* needs to separate internal reasoning from the clean final answer shown to the customer. A ignores a real UX/accuracy-perception risk. C overcorrects into banning a technique that may still be the right choice for the reasoning task itself. D swaps techniques without addressing the actual issue (output separation, not technique choice).

---

**Q11 (2d).** A long-running chat session's context grows across dozens of turns, and the team's current strategy is to append the complete raw transcript to every subsequent call. Latency and cost are rising, and quality is starting to degrade. What is the most appropriate architectural fix? *(Select ONE)*

A. Upgrade to a model with a larger context window so the full transcript always fits
B. Summarize older turns into a compact running summary, retrieve only conversationally-relevant prior content, and drop resolved sub-threads, rather than carrying the full raw transcript forward indefinitely
C. Switch to the fastest/cheapest tier to offset the rising cost
D. Disable context entirely after 10 turns so the model only sees the most recent message

**Answer: B**
**Explanation:** This is the correct token-budget and relevance-management fix — addressing what's actually in context rather than just its ceiling. A treats a relevance problem as a capacity problem and doesn't address the quality-degradation symptom (an overstuffed, poorly curated context can degrade attention regardless of window size). C doesn't address the root cause. D loses genuinely relevant context wholesale rather than curating it.

---

**Q12 (2d).** Which statement best reflects the relationship between context window size and output quality? *(Select ONE)*

A. A larger available context window always produces better output, regardless of what is placed inside it
B. A larger context window increases capacity, but relevance and organization of what's actually placed in context matter independently — an overstuffed or poorly curated context can degrade output quality and always adds cost/latency
C. Context window size has no effect on output quality whatsoever
D. Only the size of the context window matters; the order and relevance of its contents are irrelevant

**Answer: B**
**Explanation:** This is the core 2d lesson — window size is a capacity, not a guarantee, and content relevance/curation is a separate and equally important concern. A, C, and D each collapse this into an oversimplified, incorrect absolute.

---

**Q13 (2d).** A retrieval-augmented system currently loads the entirety of a 300-page policy manual into context for every user question, even though most questions only require 1-2 relevant pages. What is the correct critique? *(Select ONE)*

A. This is optimal because it guarantees the model always has full information available
B. This is inefficient — targeted retrieval of only the relevant sections would reduce token cost and latency and likely improve relevance of the response, versus loading the entire document every time
C. The fix is to switch to a lower-capability model to force more concise responses
D. There is no issue as long as the model's context window is large enough to hold the full document

**Answer: B**
**Explanation:** This is the "retrieval dumping" distractor — loading far more than what's relevant wastes tokens/latency without a corresponding benefit, and targeted retrieval (matching content to the actual query) is the correct fix. A, C, and D each fail to address the real inefficiency, and D repeats the "size negates the need to manage relevance" trap directly.

---

**Q14 (2d).** Which sub-objective distinction does the exam expect an architect to recognize? *(Select ONE)*

A. "Optimizing context windows and managing token usage" (Domain 2) and "progressive discovery vs. monolithic context strategy for tool/integration design" (Domain 3) are the same concept tested in two domains for redundancy
B. Domain 2's context management concerns curating and budgeting what's in a given interaction's context; Domain 3's progressive-disclosure question concerns how an integration/tool-calling architecture decides which of many available tools or data sources to expose at all — related but distinct concerns
C. Only Domain 3 is relevant to context management; Domain 2's context content is not tested
D. Progressive disclosure is a prompt-engineering technique identical to chain-of-thought

**Answer: B**
**Explanation:** This directly tests domain-boundary awareness, something the Professional-level exam is known to probe. A incorrectly treats two related-but-distinct concerns as identical. C and D each misstate what's actually being tested in one domain or the other.

---

**Q15 (2e).** A team wants to reduce repeated processing cost for a long, static system prompt that's sent unchanged on every one of thousands of daily calls, with only the user's message varying per call. What is the most appropriate strategy? *(Select ONE)*

A. Mark the static system prompt content as cacheable, structuring the prompt so the stable content comes first and the variable, per-request user content comes last
B. Rewrite the system prompt to be shorter so there's less to process, and skip caching entirely
C. Cache the user's message instead, since that's what changes most often
D. Move the system prompt into a chain-of-thought reasoning step so it's processed more efficiently

**Answer: A**
**Explanation:** This is exactly what prompt caching is for — a stable, frequently-reused prefix. Ordering stable content first and variable content last is the correct structural pattern for maximizing cache benefit. B discards a real cost-saving lever unnecessarily. C is backwards — caching benefits stable, *unchanging* content, not the part that varies every call. D confuses a reasoning technique with a caching/reuse mechanism entirely.

---

**Q16 (2e).** A company has 12 different internal Claude-based agents, each with its own system prompt that separately restates the same 2,000-word compliance policy nearly verbatim, with only small task-specific sections differing. What is the correct architectural critique? *(Select ONE)*

A. This is optimal because each agent's prompt is self-contained and independently readable
B. This creates drift risk (if the policy changes, all 12 copies must be updated consistently) and forfeits caching/reuse benefits; the compliance block should be a single modular, cacheable component shared across all 12 agents, with only the small task-specific sections varying
C. The fix is to delete the compliance policy from 11 of the 12 agents to reduce duplication
D. Duplication is only a problem if it affects fewer than 5 agents; at 12 agents it's an acceptable trade-off

**Answer: B**
**Explanation:** This is the modular-prompt design lesson directly — shared, stable content should exist once and be reused/cached across use cases, both for consistency (avoiding drift) and for reuse efficiency. A treats a maintenance and efficiency liability as a benefit. C removes required content instead of restructuring it. D applies an arbitrary threshold with no architectural basis.

---

**Q17 (2e).** A system currently loads every possible domain procedure (HR, IT, Finance, Legal, Facilities — five full instruction sets) into a single always-active system prompt for one general-purpose assistant, regardless of which type of question is actually asked. What is the most appropriate improvement? *(Select ONE)*

A. This is already optimal since the assistant always has every possible procedure available immediately
B. Package each domain's procedures as a separate Skill loaded on demand based on the type of question being asked, rather than permanently loading all five domains' content into every call
C. Remove four of the five domains' procedures entirely to simplify the prompt
D. Add a sixth domain's procedures to be even more comprehensive

**Answer: B**
**Explanation:** This is the "monolithic just-in-case prompting" distractor directly — the fix is progressive, on-demand loading (Skills) rather than always-resident content covering every possible task. A defends the wasteful default. C removes needed functionality rather than restructuring how it's loaded. D worsens the exact problem being critiqued.

---

**Q18 (2e).** Which statement correctly distinguishes "modular prompts" from "prompt caching" as reuse strategies? *(Select ONE)*

A. They are identical techniques with different names
B. Modular prompts are about structuring prompt *content* into reusable, composable sections at authoring time; prompt caching is about *runtime cost/latency reduction* for stable content reused across calls — the two are complementary, not the same mechanism
C. Modular prompts only apply to system prompts, while caching only applies to user messages
D. Caching replaces the need for any prompt structure at all

**Answer: B**
**Explanation:** This is the precise distinction the exam tests — modularity is an authoring/reuse-of-structure concern, caching is a runtime cost/performance concern, and a well-designed system typically uses both together (a modular shared block is exactly the kind of content worth caching). A collapses a meaningful distinction. C and D both misstate how each mechanism actually works.

---

**Q19 (2e).** A team caches a block of content that includes the current user's live, unique session ID and timestamp on every call, expecting significant cost savings. Why does this fail to deliver the expected benefit? *(Select ONE)*

A. Caching never provides cost savings under any circumstances
B. Because the cached block changes on every call (due to the unique session ID/timestamp), it never matches a previous cached prefix, so each call effectively misses the cache and pays full processing cost for that block
C. The team should have cached the session ID and timestamp separately from everything else to fix this
D. Caching only works for chain-of-thought prompts, not for system prompts

**Answer: B**
**Explanation:** This directly tests the "caching volatile content" distractor — caching only pays off when the cached prefix is identical across calls; including anything unique per call (session ID, timestamp) breaks the match every time. A overstates the failure into a blanket claim. C still includes volatile content in what's cached and wouldn't fix the underlying issue. D is an invented, unrelated constraint.

---

**Q20 (2e).** An architect proposes combining strategies: a modular, cacheable shared policy block (identical and reused across all agents) placed first in the prompt, followed by an agent-specific Skill loaded only when relevant, followed by the user's live message last. What principle does this design correctly reflect? *(Select ONE)*

A. Stable, shared, cacheable content should be structured first; task-specific content loaded on demand next; and unique, per-call content last — maximizing both cache-hit value and context efficiency
B. All content should always be loaded for every call regardless of relevance, to maximize consistency
C. Order within the prompt has no effect on caching or efficiency
D. Skills and caching are mutually exclusive strategies and cannot be combined in one architecture

**Answer: A**
**Explanation:** This is the correct, integrated application of every 2e concept together: stable/shared content first for caching, on-demand task-specific content next (Skills), and volatile per-call content last — since caching depends on prefix stability. B contradicts the on-demand-loading principle tested throughout this sub-objective. C denies the ordering principle central to how caching actually works. D incorrectly treats complementary strategies as incompatible.

---

## Quick-Reference Summary Table (All Sub-Objectives)

| Sub-objective | One-line test | Golden rule |
|---|---|---|
| 2a | Did you treat model choice as a benchmarked trade-off, not an instinct? | Start cheap, benchmark on real data, upgrade only on a measured capability gap; tune effort within a model before switching tiers; model versions are pinned, not auto-improving |
| 2b | Does your "guardrail" actually guard, or just instruct? | System prompt = role/scope/format contract; guardrail = independent output-side verification — never rely on prompt text alone for safety |
| 2c | Did you match the technique to the actual failure mode? | Zero-shot for simple/clear tasks; few-shot for format/style consistency; chain-of-thought for multi-step reasoning errors — and keep raw reasoning separate from user-facing output |
| 2d | Did you manage relevance, not just capacity? | Bigger context window ≠ automatically better; curate, summarize, and retrieve only what's relevant — don't confuse a relevance problem with a capacity problem |
| 2e | Is stable content reused, or duplicated/re-processed? | Modularize shared content, cache what's identical across calls (stable content first, volatile content last), and load specialized procedures on demand via Skills rather than keeping everything always-resident |

---

*Prepared as independent study material for CCAR-P exam preparation. Domain weight, sub-objective wording, and exam format facts are sourced from Anthropic's published Exam Guide v1.0 as summarized at claudecertificationguide.com/ccar-p and claudearchitectcertification.com/certifications/ccar-p. Explanatory content, scenarios, and mock questions are original material built on Anthropic's publicly available model and prompting guidance, with volatile specifics (exact pricing, exact cache TTLs, exact model version numbers) deliberately omitted or kept qualitative due to conflicting information found across sources — verify current numbers at docs.claude.com before an exam attempt. Not sourced from, or claimed to be, official exam questions.*

# Claude Certified Architect (Professional) - CCAR-P

This repository is a study companion for the Claude Certified Architect (Professional) exam, code `CCAR-P`.

The exam is intended for mid- to senior-level practitioners who design, evaluate, and govern production Claude-based AI systems. It focuses on turning business problems into deployable architectures that use Claude safely, effectively, and at scale.

## Who the exam is for

Anthropic targets the exam at:

- Solution architects
- AI/ML engineers
- Technical leads
- Senior software engineers

Recommended experience:

- 3+ years of systems architecture or platform engineering work
- At least 6 months of hands-on production experience with Claude or similar LLM systems

This experience is recommended rather than mandatory; performance on the exam is what determines the credential.

## Exam facts at a glance

- Exam code: CCAR-P
- Number of items: 63
- Item format: Multiple-choice and multiple-response; each question states how many responses to select
- Time limit: 120 minutes
- Delivery: Pearson VUE, online proctored or at a test centre
- Passing score: Scaled score of 720 on a scale of 100–1,000
- Anthropic exam fee: $175 USD per attempt
- Validity: 12 months from the date the credential is awarded
- Result reporting: Pass/fail with scaled score, plus percent-correct by domain on the score report
- Retakes: Waiting periods of 14, 30, and 90 days after failed attempts; up to four attempts in a rolling twelve months; fee charged per attempt
- Renewal: Free on-time renewal via a non-proctored assessment on the Anthropic Partner Academy; a lapsed credential requires the full exam at the full fee

> All figures above are based on Anthropic’s Exam Guide v1.0 (July 2026).

## Exam blueprint

Domain weights are the guide’s own figures and indicate how much of the exam each area carries.

### Domain 1: Solution Design & Architecture (17%)

- Translate business problems into Claude-based AI solutions
- Design end-to-end architectures (input → processing → output → feedback loops)
- Select appropriate architectural patterns (workflow, agentic, augmented LLM)
- Design multi-agent systems and orchestration strategies
- Apply decomposition techniques for complex problem solving
- Align solutions to business value pillars (efficiency, transformation, productivity, cost, performance SLAs)

### Domain 2: Claude Models, Prompting & Context Engineering (13%)

- Select appropriate Claude models based on trade-offs
- Design system prompts, templates, and guardrails
- Apply prompt engineering techniques (zero-shot, few-shot, chain-of-thought)
- Optimise context windows and manage token usage
- Implement prompt reuse strategies (caching, modular prompts, Skills)

### Domain 3: Integration (19%)

- Evaluate tool/agent configuration for capability bloat
- Analyse authentication and authorisation requirements to identify security gaps
- Evaluate accuracy-latency trade-offs and justify configuration decisions
- Analyse observability challenges and select monitoring strategies at scale
- Design a RAG pipeline with appropriate chunking and indexing strategies
- Apply retrieval strategies matched to data shape and query pattern
- Evaluate connection protocols and select the appropriate integration mechanism (MCP, API/CLI, agent-to-agent)
- Evaluate progressive discovery vs. monolithic context strategy

### Domain 4: Evaluation, Testing & Optimisation (16%)

- Define evaluation metrics (accuracy, latency, cost, safety, security)
- Design evaluation datasets and test frameworks using mixed methodologies
- Conduct A/B testing and iterative improvements
- Diagnose system issues (prompt failure, hallucinations, model mismatch)
- Optimise token usage, latency, and cost-performance trade-offs
- Monitor system performance using logging and observability tools

### Domain 5: Governance, Safety & Risk Management (14%)

- Implement guardrails and safety controls
- Identify risks, limitations, and failure modes of LLM systems
- Apply human-in-the-loop validation strategies
- Ensure compliance with regulations (for example, GDPR, HIPAA, FedRAMP)
- Address ethical AI considerations (bias, fairness, transparency)

### Domain 6: Stakeholder Communication & Lifecycle Management (14%)

- Conduct structured discovery and requirement gathering
- Communicate architectural decisions and trade-offs
- Manage stakeholder feedback loops and expectation alignment (including SLAs)
- Document architectures and provide implementation guidance
- Support lifecycle phases (discovery, design, handoff, monitoring, iteration)

### Domain 7: Developer Productivity & Operational Enablement (7%)

- Configure Claude tools and environments for teams (for example, Claude Code)
- Improve developer workflows using AI-assisted tooling
- Support debugging and operational issue resolution

## What the exam is testing

The Professional level goes beyond foundations and checks whether you can own a production Claude system end to end:

- discovery
- architecture
- integration with enterprise systems
- evaluation and optimization
- governance and risk management
- stakeholder communication and lifecycle support

## Study while the professional track is being built

While the Professional track is still being finalized, Anthropic’s Architect (Foundations) track remains free and open. It overlaps strongly with the Professional exam’s agentic architecture and integration material and is the best free place to begin preparing.

## Official resources and references

- Anthropic documentation: https://docs.anthropic.com/
- MCP specification: https://github.com/modelcontextprotocol

## Notes

- This repository is organized by exam domain study guides.
- The current version of the official page indicates the Professional track is planned and the exam is version 1.0, effective July 2026.
- The content in this README is a concise overview of the official exam guidance, with the exam blueprint retained in its official domain structure and detail.

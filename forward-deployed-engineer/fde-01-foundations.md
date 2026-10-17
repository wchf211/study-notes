# Foundations + Problem → POC (Modules 1–2)

> Core question of the whole course: product engineering asks "can we build this?" The FDE asks "what should we build, for whom, by when, and how do we prove value?" Everything else is the operating model that answers that question inside a customer organization.

## 1. Who Is a Forward Deployed Engineer?

An **FDE** is an embedded engineer who closes the gap between what a model *can* do and a system that works reliably in production inside a customer organization. The role traces to Palantir's work for government and defense clients, where sensitive data and mission-specific workflows made off-the-shelf or remote delivery a poor fit — so engineers worked on site and wrote production code against real systems.

Two jobs at once: deliver for one customer **and** feed recurring patterns back into the core product. The model spread because enterprise AI fails less often on model quality than on data access, access controls, workflow fit, edge cases, and user trust.

Market markers from the course: OpenAI's DeployCo launch (May 2026), Anthropic's new enterprise AI services company, FDE postings up 800%+ in 2025. Typical FDE systems: document Q&A, classification pipelines, knowledge synthesis, agentic workflows. Baseline skills: LLM APIs, agent pipelines, evals, production deployment. The FDE layer on top: customer proximity, organizational navigation, judgment under ambiguity. Frameworks and APIs already exist — the distinctive contribution is choosing the right architecture and deploying it reliably given constraints, failure modes, and human handoff points.

## 2. The FDE Operating Model (6 parts)

1. **Customer discovery** — find the real problem under the stated request.
2. **Project scoping** — a bounded deliverable.
3. **POC delivery** — prove or disprove a hypothesis. A 10-day POC answering the core question beats a six-week better-engineered system. Speed and measurability beat elegance. Know which shortcuts are acceptable and which cause downstream harm.
4. **Stakeholder communication.**
5. **Project handoff** — a system *plus* documentation another team can own and extend.
6. **Field feedback** — patterns across organizations become product intelligence.

Judgment layer: scope, simplify, flag early, keep stakeholder confidence. The timeline belongs to the customer; a missed demo damages trust. Scope to the time available. The first deliverable is the smallest working system demonstrating core value.

Assumed fluency: LLM APIs (streaming, context, token cost/latency), prompt engineering, RAG (embedding, chunking, vector search, reranking, failure diagnosis), agentic frameworks (LangChain, LangGraph, Google ADK, A2A, MCP), building agentic systems, LLM evaluation, LLMOps.

## 3. Running Customer Discovery

Discovery is a **structured technical investigation conducted as a conversation** — not selling, advising, or designing. Customers state a symptom, a pre-decided solution ("we want a chatbot"), or a vague outcome. Restraint matters: proposing an architecture turns the investigation into a pitch. Output: a clear picture of problem, data, stakeholders, constraints — feeding a scope document.

Get three types of people in the room:
- **Executive sponsor** — owns the problem, budget, priorities.
- **Ops / domain expert** — does the work today; most informative about how the system *actually* runs.
- **Technical contact** — systems, data location, real access and feasibility.

If only two are present, note the missing perspective and follow up.

Open by asking for a full account of how the workflow breaks down (trigger, who is involved, what gets stuck, how it recovers), then ask about the **workaround** — what they actually do when the official process breaks. Listen for: data signals (every system/file/source is a scope building block), volume and frequency (converts vague pain into a measurable baseline), ownership gaps (an unowned step is a risk).

Five-column note framework: stated pain | current workflow steps | data available | candidate success metric | blockers to access.

Close by summarizing problem, data, and success back to them, and checking four items: data exists and is accessible in the timeline; a specific measurable success criterion is verifiable before the end; the timeline is realistic; one owner has authority to clear blockers, plus a main contact.

Real AI opportunities show four signals: document-heavy synthesis/extraction; knowledge retrieval at scale; multi-step workflows with conditional logic; repetitive judgment at throughput. Factor inference cost in early for high volume. Establish a baseline first (current time/error cost) — or make establishing the baseline the first step.

Failure patterns: solutioning too early; **HiPPO** (Highest-Paid Person's Opinion) dominating — redirect to the doers without alienating the executive; scope creep in discovery — close with a summary, treat new requirements as a separate conversation after the scope draft; discovery theater (surface questions, vague answers, no data/workflow clarity); skipping the baseline (including projected token cost).

## 4. Scoping and Solution Architecture

Scoping converts the discovery definition into a proposal: what to build, the fitting architecture, and the risks — before building. The **scope document** is a shared agreement/alignment artifact, not a product spec. It covers the next 2–4 weeks, what will be demonstrated, and how success will be measured. A POC is typically two weeks and ends with a demo plus a proceed decision.

Six elements: problem statement (1–2 sentences, in the customer's language); success metric (a specific measurable quality threshold: precision/recall, error-reduction %, speed target, vs. a defined test set); data sources (what, where, who controls access); technical dependencies (internal APIs, storage, auth, external model APIs, retrieval/vector infra, customer systems to read/write); **out of scope** (an explicit not-built list — load-bearing; point to it when mid-build additions arrive); timeline (a date by which something demonstrable is ready).

**Translation ladder** (5 levels): business pain → user workflow → data flow → system integration → AI component. Example: legal firm contract review taking 3 hours per contract against standard terms, bottlenecking onboarding → workflow: lawyer reads/checks/flags/routes → data flow: email PDFs, shared-drive standard terms, spreadsheet approvals → integration: email + document storage + approval workflow → AI component: extract clauses, compare against standards, produce a structured deviation report.

**Minimal AI component principle**: target the highest-effort step; leave routing/email manual in v1. A **scoping spike** is a 1–2 day investigation to confirm data is accessible, in usable format, and clean enough before finalizing scope.

Architecture selection: large-document extraction/comparison/summarization → structured LLM extraction or RAG; Q&A over a large knowledge base → RAG semantic retrieval; multi-step conditional routing/tool calls/handoffs → agentic workflow with defined tools; high-volume classification/labeling/routing → structured LLM constrained output. POC v1 proves the core only.

Token/cost: agentic systems with many tool calls and long context cost the most per transaction; RAG reduces cost via relevant context only; structured extraction is batchable/optimizable. Estimate tokens per transaction × daily volume and confirm viability before finalizing. POC-scale OK but production-unsustainable is a scope-document risk.

**Technical risk register**: risk, likelihood, impact, mitigation. Risks: data quality (the most common mid-build surprise); access delays (security reviews of 2–3 weeks; a POC can stall if not confirmed pre-day-1); model performance gaps (narrow legal/medical/financial domains, terminology/abbreviations/formats); token budget overrun; latency/throughput at full volume. Early risk communication builds trust.

**No-win check**: if there is no realistic path to a demonstrable result given data/access/architecture, pause before committing. Walking away pre-start beats stalling mid-build.

## 5. The POC Sprint

One goal: does the approach work, with this data, in this customer environment? Everything else is out. Set the expectation: by the end we will know whether the approach works. This is **not** a production-ready system or a smaller final product. A POC is a time-boxed experiment answering one question; the output is an answer, not a product. Three valid outcomes: works as scoped → next phase; works with specific adjustments; does not work for this problem/data.

Four stages:
1. **Access / environment setup** — confirm model API credentials, read access, a representative test sample, a comms channel with the technical contact. Ends when code can be written and run against real data or a reliable approximation. If a dependency is missing, identify what can proceed independently vs. what is blocked.
2. **Data exploration / validation** — examine actual format, consistency, and whether the sample reflects the production population. Not optional even if discovery described it. A pipeline built without examining real data needs rework.
3. **Building toward working output** — a first working output (real input → result, even low quality) quickly beats high quality slowly. Then iterate against the scope success metric and the agreed test set, not "looks reasonable."
4. **Evaluation / demo prep** — formal evaluation against the test set, a demo interface, a readout document. A brief async end-of-day update: completed, blocked, next. Share rough intermediates early; silence breeds concern.

Responding to trouble: access delayed → build with synthetic/mocked inputs; document what was mocked, what real data replaces, and what final evaluation uses. Formal evaluation must use real data, never simulated. Data differs → build against the actual data, document assumptions, report the discrepancy with specific impact, not "harder than expected." Version control + documentation from day one: a repo, frequent clear commits, code as if another engineer will maintain it. Record assumptions at the moment they are made.

The POC demo is a **value narrative**, not a tech walkthrough: Before (current workflow, time/cost/error baseline) → During (run on real representative test-set input; typical + strong beats cherry-picked perfect) → After (output → business impact: time saved × weekly volume). Live failure: acknowledge calmly, switch to a recorded backup, explain.

Close: confirm the success metric/threshold is still agreed before the demo; report the primary metric **plus** latency (an operational requirement), cost per transaction at projected volume (budget), and edge-case performance (where it underperforms and how often).

Outcomes: **Proceed** (metric met, integration works, customer confident — agree next steps before leaving); **Pivot** (metric missed but a more specific path exists — define the change before the next attempt); **Stop** (does not work, or cost/latency infeasible at scale — a correct answer at the lowest cost). After Proceed: a POC is not production; the next phase is a more complete system, broader inputs, production integration, real users. The readout document is one page — what was built, what was measured, what measurement showed, recommendation — plus a follow-up document: next-phase build, initial users, success definition.

## Self-test (4 questions)
1. Product engineering asks "can we build this?" What does the FDE ask instead, and why does that change the work?
   → "What should we build, for whom, by when, and how do we prove value?" It shifts the work from building to discovery, scoping, and proof.
2. In discovery, who are the three people you need in the room, and what does each one give you?
   → Executive sponsor (problem, budget, priorities); ops/domain expert (how the work actually runs); technical contact (systems, data location, real access/feasibility).
3. What makes the "out of scope" list load-bearing, and what is the no-win check?
   → It is the artifact you point to when mid-build additions arrive. No-win check: if no realistic path to a demonstrable result exists, pause before committing — walking away pre-start beats stalling mid-build.
4. A POC has three valid outcomes. Name them, and state what a POC's output is.
   → Works as scoped; works with specific adjustments; does not work. Output is an answer, not a product.

# Teaching Lens: How This Course Teaches (Module-by-Module)

> Second lens: not just what the course teaches, but how it organizes explanations — and what is stealable for teaching FDE / agentic systems yourself.

## The Course's Core Pedagogy

Each case study repeats the same three-beat arc — **Decomposing the Problem → System Design → FDE Interview Questions** — so the same system is first broken into subproblems and constraints, then built, then defended under interview pressure. That repetition is the pedagogy: decomposition produces a refined problem statement, design decisions are explicitly traced back to decomposition constraints, and interview questions reward naming the exact component, metric, or failure mode rather than describing a concept in general terms.

Earlier modules build the reusable toolkit (discovery → scope → POC → pilot → handoff → field feedback; stakeholder communication; harness/orchestration/framework choice) and later modules reuse it in three contrasting architectures (RAG, multi-agent, fine-tuning), so the learner compares architectures against the same FDE criteria instead of learning each in isolation. Company guides then re-apply the same toolkit to four different hiring loops, showing how one profile maps to different formats.

## What Is Stealable

1. **Start every case with the full case study in a box, then decompose it in public** — clarifying questions by category, subproblems, constraints, refined problem statement — so learners see the thinking, not just the answer.
2. **Make every design decision traceable to a constraint surfaced in decomposition.** "Every component exists because the case required it" is a checkable standard.
3. **Use the translation ladder** (business pain → workflow → data flow → integration → AI component) and the **minimal-AI-component principle** to keep scope honest.
4. **Teach evaluation as a design decision, not an afterthought.** Define what good looks like, how it is measured, the production threshold before building. Use field-level vs. full-record accuracy, shadow mode, representative test suites across categories (typical / edge / failure / ambiguous) to make evaluation concrete.
5. **Use named failure modes + a monitoring signal + a remediation step** in handoff documentation, and use matching / Show Answer exercises and carousels for retrieval practice — but note the carousel flakiness observed here (see Limitations in README).
6. **Close each interview lesson with the same refrain**: grounded, specific answers beat general descriptions.
7. **End with a personal playbook habit** (30 minutes after each project: what worked, what to ask earlier, what to do differently) to turn field experience into compounding learning.

## What to Watch Out For

- **Carousel fragility**: one RAG interview question (Evaluation Q3) never displayed because the carousel stayed at 2/3; multi-agent and fine-tuning interview lessons only had the first answer per section expanded. When teaching with carousels, keep each question reachable without fragile UI state — a plain list is more robust than a carousel.
- **Density without a running example**: the early toolkit modules (discovery → handoff) are abstract until the case studies arrive. A learner who stops before Module 7 misses the payoff. Front-load one concrete case earlier, or keep a single running example through the toolkit chapters.
- **Interview lessons reward recall of the case**: the interview Q&As are strongest when the reader just finished that case's design. Spaced far apart, they lose context. Keep case + interview questions adjacent — which this course does, and it works.

## Self-test (4 questions)
1. What is the course's three-beat arc per case study, and why does the repetition work?
   → Decomposing the Problem → System Design → FDE Interview Questions. Same system, three passes: break it down, build it, defend it. Design decisions trace back to decomposition constraints; interview answers reward naming exact components/metrics/failure modes.
2. What is the "translation ladder," and what is the minimal-AI-component principle?
   → Business pain → workflow → data flow → integration → AI component. Minimal-AI-component: target the highest-effort step; leave the rest manual in v1.
3. How does the course make evaluation concrete instead of abstract?
   → Evaluation as a design decision before building: define what good looks like, how it is measured, the production threshold. Concrete tools: field-level vs. full-record accuracy, shadow mode, representative test suites across typical/edge/failure/ambiguous cases.
4. What is the personal playbook habit, and why does it compound?
   → After every project, 30 minutes capturing what worked, what to ask earlier next time, what to do differently. Over 3–4 projects it becomes a compounding asset no course can provide.

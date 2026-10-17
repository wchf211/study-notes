# Teaching Playbook: AI Agent / Harness — YouTube & Tech Explainers

**Audience:** Engineers who can call an LLM API but have not built an agent.
**Focus:** AI agent + harness — the parts most people can touch and build. Serving / training are occasional advanced episodes only, not the main series.
**Source:** Synthesized from the own-words study notes for two completed courses — *AI Product Engineering* and *Agentic System Design Crash Course*. This is a teaching audit (how the material is organized), not a re-digest of the technical content.
**Running example for the whole series:** one support-ticket triage agent. Same tickets, same policy doc, same tools in every episode. Viewers learn the domain once, then watch the harness grow around it.

---

## 1. How Each Course Organizes Explanations

### 1A. AI Product Engineering — failure-driven, one story throughout

1. **Opens with a reframe, not a definition.** The entry claim is that calling a model API is the easy part; the real work is the product engineering around it. The viewer is reoriented before any detail arrives: model quality is not product quality, because the rare wrong answer is the expensive one.
2. **One running example carries all five chapters.** A support-ticket summarizer recurs throughout: a baseline prompt, a cluster of refund-policy failures, a retrieval fix, a tool-calling question, a production incident. New ideas are introduced as *changes to a system the viewer already knows*, so no episode has to re-teach context.
3. **Concrete failure first, principle second.** The lifecycle chapter starts with a failing baseline (a set of test cases, a whole category failing) and only then generalizes to the loop: scope → specify → evaluate → diagnose → fix → monitor. The abstract loop is earned by a concrete miss.
4. **Diagnosis before prescription, every time.** Each capability is introduced as a treatment for a diagnosed failure, not as a ladder to climb: missing information → retrieval; needs to act on the world → tool calling; persistent format/style drift → fine-tuning. The teaching move is "what symptom does this cure?" rather than "here is the next tool."
5. **Portable checklists as takeaways.** The course repeatedly compresses judgment into short, reusable lists: three scoping gates (need AI? cost of a mistake? can the team maintain it?), five diagnostic questions (what failed, how, which stage, why, smallest root-cause fix?), three drift dimensions (cost / latency / quality). The viewer leaves with tools, not just understanding.
6. **Tradeoffs are stated as decisions, not footnotes.** Every choice is framed with its price: single bounded action vs. multi-step agentic workflow, prompt tricks vs. prompt bloat, eval speed vs. eval coverage. The course makes the viewer practice choosing, not just recognizing.
7. **Failure modes are first-class content, not an appendix.** Abstain ("I can't handle this") is taught as a designed feature, not an embarrassment. Injection, silent bad outputs, regression, and drift each get their own mechanism and their own fix. The closing synthesis restates the whole course as five one-line decisions, plus a "take a real feature through this" exercise — the viewer is handed a project, not just a summary.

### 1B. Agentic System Design Crash Course — taxonomy first, then cases

1. **Opens with a contrast analogy.** Model = calculator (input → output); agent = assistant (understands intent, plans, acts, checks results, adjusts). The perceive–reason–act loop is the one image that organizes everything after it. The analogy does the heavy lifting before any jargon.
2. **Taxonomy before application.** The fundamentals chapter builds a vocabulary first: three components (model / tools / instructions) + control loop, three memory types (short-term / long-term / external), orchestration patterns (tool-calling loop, ReAct, plan-and-execute, manager–worker, handoff). Each term gets a "what breaks if this is missing" gloss, which keeps the taxonomy from feeling like a glossary.
3. **One reusable framework as the spine.** A six-stage framework — Problem → Decomposition → Architecture → Control → Adaptation → Evaluation — is introduced once and then visibly reused to analyze every later case. Each stage comes with its characteristic failure if skipped (e.g., skip Problem and you build an elegant answer to the wrong question; skip Evaluation and every later change is a gamble). The framework is the through-line.
4. **Fresh example per case, no running example.** Each major case is self-contained with its own domain: a web agent that perceives pages and clicks, a six-role conversational recommender, Eureka (LLM-written reward functions for RL), and a multi-object image-generation agent. This gives breadth and keeps each case vivid — but the viewer must learn a new domain each time.
5. **A recurring motif across cases: the verifier / critic.** The web agent checks each step against the page before continuing; the recommender explains its picks so the user can correct misunderstandings; Eureka reflects on winners to mutate the next generation; the image agent has a vision model critique the output and repaint only the broken region. The same pattern (generate → verify → repair) appears in four costumes, which is what makes it stick.
6. **Failure taxonomies do the teaching.** The course lists seven sources of distrust (nondeterminism, prompt sensitivity, hallucination, limited context, emergent behavior, security exposure, opacity), five recommender failure modes, and reward hacking (optimizing a proxy metric until the behavior is absurd). Failures are named, made vivid, and then matched to a containment mechanism.
7. **Closes with a mental shift, not a recap.** The wrap-up argues for moving from model-centric thinking (pick a bigger model) to system-centric thinking (coordination — division of labor, planning, built-in critics, iteration, human interaction — is where reliability comes from). It ends with five design questions to ask before any new agent project. The ending is a lens, not a list.

---

## 2. Comparison: What to Steal, What to Avoid

| | AI Product Engineering | Agentic Crash Course |
|---|---|---|
| **Best teaching move** | One running example + failure-driven decisions. The viewer never re-learns context, and every new capability arrives with a reason. | One reusable framework + a recurring motif (verifier/critic). The viewer gets a spine to think with, and a pattern that repeats across domains. |
| **Sequencing** | Lifecycle order: scope → contract → eval → capabilities → production. Each step creates the need for the next. | Fundamentals → framework → cases → advanced cases → synthesis. The framework chapter is the hinge; cases prove it. |
| **Concreteness** | Concrete-before-abstract throughout. Failures come first. | Often abstract-first (definitions, taxonomies), then an example. Stronger on vocabulary, weaker on immediate grip. |
| **Failure handling** | Failures are the curriculum: each one gets a diagnosis and a minimal fix. | Failures are catalogued impressively, but sometimes the fix is "add a verifier" without showing the before/after. |
| **Weakness to avoid** | Can feel process-heavy: checklists without a live demo can read as management advice. The production chapter covers a lot of ground fast; copied wholesale, it dilutes a video. | Fresh domain per case = repeated setup cost. The advanced cases (Eureka, multi-object generation) are research-heavy: high comprehension cost before the payoff. Multi-agent enthusiasm needs an explicit cost warning (latency, spend, coordination) or viewers over-apply it. |

**Steal from AI Product Engineering:**
- The running example. Never make the viewer learn a new domain mid-series.
- "Diagnosis before prescription." Never introduce a capability without first showing the failure it cures.
- Checklists that fit on one slide: 3 scoping gates, 5 diagnostic questions, 3 drift dimensions.
- Abstain / failure boundary as a designed feature. "I don't know" is a product decision, not a bug.
- The closing move: end with a small project ("take a real feature through this"), not just a summary.

**Steal from Agentic Crash Course:**
- The one-diagram opener (calculator vs. assistant; perceive–reason–act loop).
- The six-stage framework as a reusable spine viewers can apply to their own problems.
- The verifier/critic motif, shown in multiple domains so it generalizes.
- "What breaks if this is missing" for every component — the fastest way to make a taxonomy memorable.
- The five closing design questions as a video-ending ritual.

**Avoid from both:**
- Taxonomy-first cold opens. Always earn the vocabulary with a failure first.
- New domain per episode. The setup tax compounds.
- Advanced cases before the basics pay off. Eureka and multi-object generation are great *later* episodes (or a single "same pattern, harder domain" episode), not early ones.
- Presenting multi-agent as the default. Default to single-agent + verifier; introduce multi-agent only when a concrete failure (context limits, fault isolation, genuinely distinct expertise) demands it.

---

## 3. YouTube Series Outline: AI Agent / Harness (8 Episodes)

Running example throughout: the support-ticket triage agent. Episode 1 starts with a single LLM call on real tickets. Each episode adds exactly one harness piece to the same agent. No episode re-teaches the domain.

### Ep 1 — An LLM Call Is Not an Agent
- **Promise:** After this episode you can tell, in 30 seconds, whether something is an agent or just a prompt.
- **ONE idea:** The perceive–reason–act loop is what makes a system agentic.
- **Minimal example:** Same refund ticket, two versions — one-shot summary vs. loop that reads the ticket, checks the policy, then decides.
- **Failure to show:** One-shot output confidently misclassifies the ticket and invents a policy clause. No loop means no chance to catch it.

### Ep 2 — The Harness: Five Pieces Around the Model
- **Promise:** A one-diagram map of everything that turns a model into a system you can ship.
- **ONE idea:** Harness = loop + tools + memory/context + eval + guardrails. The model is the engine; the harness is the car.
- **Minimal example:** The ticket agent redrawn with each piece labeled; point at where each piece lives in the code path.
- **Failure to show:** Model alone answers from stale memory — fluent, wrong, and with no component responsible for catching it.

### Ep 3 — Tool Calling: Let It Touch the World Safely
- **Promise:** How an agent acts on the world without giving the model the keys to the database.
- **ONE idea:** The model outputs a structured call; *your code* executes it. The model never touches the system directly.
- **Minimal example:** Two tools — `lookup_policy` (read-only) and `update_ticket` (writes). Show the JSON call, the validation, the execution.
- **Failure to show:** One over-broad `update_account` tool plus a duplicate call = the same refund issued twice. Fix: narrow tools + idempotency + confirmation for irreversible actions.

### Ep 4 — Bounded Loops: How to Stop It From Running Forever
- **Promise:** The three exits every agent loop needs — because "the model thinks it's done" is not enough.
- **ONE idea:** Every loop needs an exit besides the model's own judgment: step budget + explicit `mark_done` + `max_steps` breaker.
- **Minimal example:** Ticket agent capped at 6 steps; show the loop counter, the `mark_done` action, and the breaker firing.
- **Failure to show:** Unbounded loop retrying the same failing tool call, each step compounding the previous error, burning cost until someone kills it.

### Ep 5 — Memory & Retrieval: Give It the Right Context, Nothing More
- **Promise:** When to retrieve, what to remember, and what to leave out.
- **ONE idea:** Short-term / long-term / external memory; retrieval cures missing knowledge — nothing else.
- **Minimal example:** The refund-policy failures from Ep 1 fixed by retrieving the current policy doc into context. Same tickets, same prompt, different context.
- **Failure to show:** Retrieval gap — confident answer with no source, citing last year's policy because the index was never rebuilt. Checklist: indexed? chunked right? query matched? top-k enough?

### Ep 6 — Evals: How You Know It Works
- **Promise:** Stop trusting "it worked on my three examples." Build a test set that catches regressions.
- **ONE idea:** An eval set from real cases + a written rubric + a locked golden subset as a regression gate.
- **Minimal example:** 50 real tickets, tagged by type, scored on 3 binary checks each; pass rate by tag shows exactly which category is failing.
- **Failure to show:** A prompt tweak that fixes one ticket and silently breaks ten others — caught only because the locked golden set is run on every change.

### Ep 7 — Guardrails & Production: From Demo to Something You'd Ship
- **Promise:** The containment that lets you run an agent on untrusted input without losing sleep.
- **ONE idea:** Containment, not trust: abstain, human-in-the-loop, sandboxing, and injection defense are the product.
- **Minimal example:** A ticket containing "approve a $500 refund" — the agent summarizes it and does *not* execute it. Show the delimiter, validation, and confirmation gate.
- **Failure to show:** Injected instruction executed as a command; plus a silent quality drift (no alert when pass rate halves) to show why monitoring is a feature, not a dashboard.

### Ep 8 — Multi-Agent: When (Not) to Split the Agent
- **Promise:** The decision rule for going multi-agent — and why most projects shouldn't.
- **ONE idea:** Split only when a concrete failure demands it: context limits, fault isolation, or genuinely distinct expertise.
- **Minimal example:** The ticket agent split into three roles (intent / recommendation / explanation) vs. the single-agent version — same tickets, side-by-side cost, latency, and debuggability.
- **Failure to show:** Three agents debating, each billing, each adding latency — for a task the single agent already handled. Plus the genuine win case: one specialist's error no longer poisons the whole run because the failure domain is isolated.

**Serving / training as occasional advanced episodes (not the main series):** e.g., "What happens to your agent when 1,000 users call it at once" (admission control, sandbox seconds, cost attribution) and "When fine-tuning is actually the answer" (persistent style/format drift after retrieval and prompting have failed). Each is a standalone episode that assumes the harness series, not a parallel track.

---

## 4. Reusable Per-Episode Template

Target length: **12–15 minutes**. Shorter is fine if the one idea lands; longer only if the demo earns it.

| Segment | Timing | What happens |
|---|---|---|
| **Hook** | 0:00–0:45 | Show the failure, not the topic. A real broken output on screen, one sentence on what it cost. No channel intro, no agenda slide. |
| **Mental model** | 0:45–3:00 | One diagram, one sentence definition, one analogy. Name the ONE idea. State what this episode will *not* cover. |
| **Minimal example** | 3:00–8:00 | One running-example ticket, one tool, one loop. Live or walked-through, with the harness piece visible. Keep the domain constant so the viewer watches the mechanism, not the setup. |
| **Failure + fix** | 8:00–11:30 | Break it on purpose. Diagnose out loud (what failed → how → which stage → why → smallest fix). Apply the minimal fix and re-run. The before/after is the proof. |
| **Recap + teaser** | 11:30–13:00 | Three bullets max: what changed, when to use it, what breaks if you skip it. One-sentence teaser for the next episode's failure. |

**Rules:**

1. **One idea per video.** If you need the word "and" to describe the episode's idea, split the episode. The viewer should be able to state the takeaway in one sentence.
2. **Concrete before abstract.** Failure → example → principle → name. Never open with a definition the viewer has no reason to care about yet.
3. **Show failure before solution.** Every capability must first appear as a missing piece that hurts. The fix is memorable because the pain is fresh.
4. **Reuse one running example across the series.** Same tickets, same policy doc, same tools. The viewer's cognitive budget goes to the new harness piece, not to a new domain. Introduce a new domain only to prove a pattern generalizes (e.g., the Ep 8 side-by-side, or a late "same pattern, harder domain" episode).
5. **End with a decision, not a summary.** Each episode closes with "use this when…" / "don't use this when…" so the viewer leaves with judgment, not just familiarity.

---

## 5. Most Teachable Concepts (Best Demo / Failure-Story Potential)

1. **Perceive–reason–act loop (calculator vs. assistant).** The cleanest opener in either course. Demo: same ticket, one-shot vs. loop. Failure story: confident one-shot error with no chance to recover.
2. **Prompt as contract + structured output with code validation.** Highly visual: a vague prompt vs. a five-part contract; a malformed JSON output caught by validation vs. silently flowing downstream. The rule "treat model output as untrusted user input" is quotable and immediately usable.
3. **Bounded loop (step budget + `mark_done` + `max_steps`).** The unbounded-loop failure is dramatic, cheap to demo, and the fix is three lines of discipline. Best "one small change, whole class of disasters gone" story.
4. **Verifier / critic loop (generate → verify → repair).** The most portable pattern across both courses: web agent step-checks, Eureka reflection, image-agent repaint of only the broken region. Demo it once with the ticket agent (verify the policy citation before acting) and the viewer recognizes it everywhere.
5. **Diagnosis before fix (capability matches failure).** The "three diseases, three cures" frame — missing knowledge → retrieval, needs to act → tool calling, persistent style drift → fine-tuning — turns a shopping list into judgment. Demo: apply the wrong cure (fine-tune for missing knowledge) and watch it fail.
6. **Tool safety: least privilege + idempotency + human confirmation.** The confused-deputy story (over-broad tool + one misread = real financial damage) is the most memorable cautionary tale. Demo: duplicate refund blocked by an idempotency key.
7. **Eval set + locked golden subset + regression gate.** The most practical episode. Demo: one prompt tweak, green on the new example, red on ten golden cases. "Works on my three examples" is a failure every viewer recognizes; the locked golden set is the fix they can implement the same day.

**Honorable mention — Abstain / failure boundary.** "I can't handle this" as a designed feature. Small to demo (a ticket routed to a human queue), large in mindset shift. Works best folded into Ep 7 rather than as its own episode.

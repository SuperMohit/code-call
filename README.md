# Code Call — Technical Spec

**Version:** 0.1 (draft for build/handoff)
**One-line:** 

A live, voice-based meeting simulator where an engineer presents/defends work to a room of AI personas grounded in their actual codebase — improving both communication *and* codebase understanding, with a review-mode redo loop and a per-area mastery map. I see many generic tools to improve communication and others to improve codebase understanding. This merges both.

---

## 1. Goal & scope

The product is **dual-track**. It is not just a speaking coach:

1. **Communication** — practice technical meetings (standup, demo, architecture review, customer call, roadmap discussion, bug triage) in a risk-free room.
2. **Codebase understanding** — each session is anchored to a topic in the user's repo; the conversation surfaces knowledge gaps and misconceptions, building a map of where the user is fluent vs. shaky.

A session is a **live ~15-minute discussion**, voice-based, multi-party. Redo/correction happens in **review afterward**, not via live interruption.

**Non-goals (v1):** writing/reviewing code for the user, replacing human mentorship, async/text-only practice.

---

## 2. Core model

### 2.1 A session = one topic

Every session anchors to a single **topic**. Topics come in types, and the type determines what context is pre-fetched, which scenario runs, and how scoring is weighted:

| Topic type | Example | KB material pulled | Scenario | Tests |
|---|---|---|---|---|
| **Subsystem** | "the database design" | schema, models, migrations, query patterns | Architecture review | Structure & trade-offs of one module |
| **Feature** | "the checkout flow" | all subsystems the feature touches (cross-cutting) | Demo / cross-subsystem review | Holding an end-to-end flow |
| **Bug** | a failing endpoint | failing code + stack trace + blast radius | Debugging review | Diagnosis: root cause, fix, risk |
| **Library / integration** | "should we add Stripe / swap the ORM" | current code **+ external library docs** | Roadmap / decision | Judgment: trade-offs, fit, migration cost |

Note two structural consequences:

- **Library/integration breaks the pure-repo assumption** — the KB must hold external library docs as well as code (see §4).
- **Feature and bug are cross-cutting** — they anchor to a *subgraph* (set of nodes a feature touches) or a *point* (bug site), not a single node. Same code map underneath, different selection shape.

### 2.2 Three measurement tracks

Kept separate on purpose — they behave differently and must not be averaged:

- **Communication (general).** Clarity, conciseness, business framing, jargon fit, holding the room. Transfers across topics → single trend line.
- **Technical understanding (per-area).** Logged against the code map → heatmap, not a single number.
- **Delivery.** Pacing, filler, confidence (from the speech layer).

The scorer must distinguish three outcomes when a persona probes a decision, because they look identical on the surface but mean different things:

1. **Explained badly but knew it** → communication track only.
2. **Couldn't answer** → knowledge *gap*, logged on the area.
3. **Answered wrong** → *misconception*, logged on the area.

Only (2) and (3) feed the codebase map. This separation is the reason the product works as a dual tool.

---

## 3. Agents

Two kinds: **persona agents** (in the room, talk to the user) and **orchestration agents** (behind the glass, run the session).

### 3.1 Persona agents (the room)

Each is the same engine with different settings. A persona is defined by four knobs:

- **Stance** — skeptical, supportive, non-technical, budget-focused, etc.
- **Lane** — what it cares about and what KB material it pulls.
- **Satisfied-by** — what makes it stop pushing.
- **Push intensity** — how hard it presses.

Seed roster:

| Persona | Lane | Pulls from KB |
|---|---|---|
| Skeptical QA Lead | edge cases, consistency, "what breaks", test gaps | test files, recent bugs |
| Senior Engineer | the technical decision itself, trade-offs, alternatives ruled out | the code under discussion, related patterns |
| Non-technical Customer | what it does for them; punishes jargon | feature/product-level docs |
| Budget-focused Exec | cost, timeline, business impact | roadmap, timelines |
| Engineering Manager | blockers, dependencies, risk, team impact | project/planning context |
| Friendly Peer | low-pressure warm-up; fills out the room | light context |

### 3.2 Orchestration agents (behind the glass)

- **Director** — facilitator, not a traffic cop. Reacts to the *shape of the whole meeting* (personas react only to the last thing said). Anchors the session to the topic, then may traverse to a connected area (§5.3). Behavior is **part-visible**:
  - *Visible* (speaks as itself): pulling the human in ("Let's hear your take on the timeline"), steering back on topic, time/coverage calls ("five minutes left").
  - *Invisible* (taps a persona, persona speaks in its own voice): ensuring a concern gets raised, reviving a stalled room, piling on a weak answer.
  - **Rule:** facilitation moves are visible; content moves are invisible. Invisible nudges must respect persona fit — the Director says "raise your concern about X now," never "say this line." Personas always own their own voice and KB context.
- **Scorer** — tracks the three tracks live against KB ground truth; produces the gap/misconception/badly-explained split.
- **Coach** — the only agent that breaks the fourth wall and addresses the user *as the user*, in review. Runs the redo loop (§6.3).
- **Scenario Builder** — given a topic, pre-fetches relevant context and *casts the room*: picks which personas attend and what each walks in knowing.

### 3.3 The room dynamics (turn-taking)

Emergent, not scripted round-robin. Each persona computes an **urge to speak** every beat (how provoked it is by what was just said); highest urge speaks, others yield.

- While the human is speaking, all urges are **suppressed** — the room waits.
- On silence, the highest-urge persona fills the gap (dead air gets filled by a challenge — this is part of the pressure).
- When the human directs a clarifying question at a persona, that persona's urge **overrides**; others yield.
- **Interrupt:** if the human rambles/dodges, a persona's urge can exceed a barge-in threshold and it cuts in rather than waiting for silence.
- **Barge-in (voice):** if the human starts talking while a persona is mid-sentence, the agent backs off.

The Director sits on top of this only for global concerns (coverage, pacing, pulling the human in).

---

## 4. Knowledge base (Qdrant)

### 4.1 Sources (multi-source, payload-tagged)

- **Code chunks** — from the repo.
- **NL summaries** — a natural-language description per code chunk (personas query in English; NL→code retrieval is much better with an NL representation present).
- **External library docs** — for integration/library topics.
- **Project context** — roadmap, planning docs, recent PRs, bug/trace data.

### 4.2 Embedding & chunking

- **Chunk by structure, not line count.** AST-based: split on functions/classes; keep file path + signature with each chunk.
- **Embed two things per chunk:** the code and its NL summary (Qdrant named/multi-vectors on one point).
- **Model:** VoyageCode3 (code-specialized, strong default) or Qodo-Embed-1 (self-hosted/SOTA-small). Avoid general-text embedders for code.

### 4.3 Qdrant features in use

- **Hybrid search (dense + sparse, RRF fusion).** Dense for semantics, sparse (BM25/SPLADE/miniCOIL) for exact identifiers, function names, error strings that dense misses. Essential for code.
- **Named / multi-vectors** — code embedding + NL-summary embedding per point.
- **Payload + filtering** — tag every chunk: `file_path`, `symbol_type`, `language`, `subsystem`, `pr_id`, `source_type` (code/doc/roadmap/trace), `tenant_id`. Filters applied during HNSW traversal (high recall, low latency). This is the mechanism for per-persona pre-fetch.
- **Multitenancy** — repos are private; isolate per user/repo via `tenant_id` payload filter or per-collection. Non-optional.
- **Quantization** — only when a repo grows large (can cut RAM dramatically). Premature otherwise.

---

## 5. Code map & coverage

### 5.1 Building the map

- **Nodes = subsystems** (auth, data layer, caching, API, infra). Derived from directory structure + dependency/import graph, or by clustering embeddings in Qdrant.
- **Edges = real dependencies** between subsystems (from the import graph).

### 5.2 Coverage state

Tracked **per node and per edge**:

- Node coverage: tested? fluent / shaky / untested; last tested; open gaps & misconceptions.
- **Edge coverage:** whether the *seam* between two subsystems has been tested. Getting both nodes individually but fumbling their interaction is its own gap type, logged on the **edge** — the heatmap has hot edges, not just hot nodes. Seams are usually the most valuable thing to study.

### 5.3 Anchoring & traversal

- Session **anchors** to the topic; pre-fetch is deep there.
- The Director may **traverse only along real graph edges** (cache → auth because the code actually connects them — never to an unrelated subsystem).
- **Gated on competence:** only cross once the anchor area is handled solidly. Weak answer → stay and drill. Strong answer → "now, how does this interact with auth?"
- **Capped:** one, maybe two hops per session. Goal is testing seams, not a tour.

---

## 6. Session lifecycle

### 6.1 Before the call

1. Selection policy picks the topic (§7).
2. Scenario Builder resolves the topic type → scenario + rubric weighting + persona cast.
3. **Pre-fetch:** each persona fires its seed queries, payload-filtered to its lane and the focus area; cache top chunks into that persona's context. Do **not** over-fetch — an omniscient persona that has "read every file" is unrealistic.

### 6.2 During the call (~15 min, live voice)

- Personas converse autonomously (§3.3); Director facilitates globally (§3.2).
- **Mid-call retrieval:** only when the user says something that triggers a follow-up — keeps personas fast without making them omniscient.
- **Seam-scoped retrieval on traversal:** when the Director gates a jump to auth, fire auth's seed queries filtered to *its boundary with the cache* (the cross-calls), not all of auth.

### 6.3 Review (the redo loop)

The Coach runs this:

1. **Attempt** — user explained X during the call.
2. **Score** against the rubric (visible tracks), point to the weakest moment (scrub to the timestamp / clip).
3. **Retry** — user re-enters the conversation at that beat; personas resume from there. On each retry the persona **varies its follow-up / angle** so a rehearsed line gets exposed, not rewarded (anti-memorization).
4. **Reveal late** — the model answer (how a strong lead would say it) unlocks only after a real attempt, or via an explicit "I'm stuck." Then the user says it again **in their own words**. Suggest-then-repeat trains parroting; attempt-then-compare trains skill.
5. **Termination:** loop exits when all rubric dimensions clear threshold; the rubric must be **visible and improving across attempts** so the gate never feels arbitrary.

---

## 7. Selection policy (what to practice next)

Weighted by:

- **Weak / uncovered areas first** — spaced repetition over the heatmap (nodes *and* edges).
- **Recent churn** — new PRs surface areas not yet tested.
- **Complexity** — the gnarly parts the user most needs to understand.

The chosen topic also suggests the scenario type (data layer → architecture review; user-facing feature → demo; infra/scaling → roadmap; failing code → bug triage).

---

## 8. Speech stack

**Pipeline architecture (STT → custom agent LLMs → TTS)** — *not* a monolithic speech-to-speech model, because the design needs full control of the LLM layer (personas, Director, Scorer are custom agents). Accept ~200–400ms more latency than S2S for that control.

- **STT:** Deepgram Nova-3 (sub-300ms streaming).
- **TTS:** ElevenLabs (distinct, consistent per-persona voices) or Cartesia/Inworld (lowest latency). A different voice per persona is a hard requirement.
- **Must-haves:** VAD + barge-in (human speaks → persona yields), TTS streaming (audio starts before the full sentence is generated).

---

## 9. Outputs (the payoff)

Per session and cumulative:

- **Communication trend** — single line over time across the three delivery/clarity dimensions.
- **Codebase heatmap** — nodes and edges colored fluent / shaky / untested.
- **Study list** — the specific files/concepts where the user had gaps or misconceptions, pointed at the actual code (Qdrant retrieved it), plus library docs for integration topics.

---

## 10. Open decisions

- Exact urge-to-speak scoring function and barge-in threshold (tuning, not architecture).
- How much context each persona caches at pre-fetch (realism vs. coverage).
- Whether edge-traversal is ever surfaced to the user or always invisible.
- Rubric weights per topic type (needs calibration on real sessions).
- Voice provider final pick (latency vs. voice variety trade-off).

---

## 11. Suggested v1 scope

Prove the engine on the narrowest slice that exercises every part:

- **One repo, one tenant.** Full ingest pipeline (AST chunking → dual embeddings → Qdrant with hybrid + payload).
- **Two topic types** that stress opposite ends: a **subsystem/architecture review** (adversarial, technical-defense heavy) and a **demo or library decision** (business-framing heavy). This validates that the rubric reweighting works.
- **Three personas** (Senior Eng, QA Lead, one of Exec/Customer) + Director + Scorer + Coach.
- **Live voice**, ~15 min, with barge-in.
- **Review redo loop** with the late-reveal sequencing.
- **Code map with node-level coverage**; defer edge/seam tracking and multi-hop traversal to v2 once the single-area loop is solid.

Everything downstream (more topic types, more personas, edge traversal, selection policy sophistication) is additive on this core.

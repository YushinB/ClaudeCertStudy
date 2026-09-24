# CCAR-P Study Notes

> **Root of trust: the official Exam Guide** (`docs/Claude Certified Architect - Professional Exam Guide.pdf`).
> Secondary: `docs/cca-p-exam-guide.pdf` (study-plan guide). When any other source disagrees with the Exam Guide, **the Exam Guide wins**.
> **Target:** pass with **800+/1000** (cut score is 720).

### Exam facts (verified against the Exam Guide, §5 "Exam Details at a Glance")

| Item | Value | Verified |
|---|---|---|
| Exam code | CCAR-P | ✅ §5 |
| Items | 63 | ✅ §5 |
| Format | Multiple-choice and multiple-response (each item states how many to select) | ✅ §5 |
| Time | 120 minutes (~1.9 min/item) | ✅ §5 |
| Passing score | Scaled 720 on 100–1,000 | ✅ §5 |
| Scoring model | Criterion-referenced (formal standard-setting study) | ✅ Scoring section |
| Fee / validity | $175 USD / 12 months | ✅ §5 |
| Score report | Pass/fail + scaled score + % correct by domain | ✅ §5 |
| Unanswered = wrong → always answer | Not stated explicitly in the Exam Guide | ⚠️ Treat as a safe habit, not a cited rule |
| No 6-scenario bank (that is the Foundations exam) | Study by domain and by decision-making | Study rule |

### Content outline (Exam Guide §6 — Blueprint)

| # | Domain | Weight | ≈ Items | Study order |
|---|---|---|---|---|
| 3 | Integration | 19% | ~12 | 1 |
| 1 | Solution Design & Architecture | 17% | ~11 | 2 |
| 4 | Evaluation, Testing & Optimization | 16% | ~10 | 3 |
| 5 | Governance, Safety & Risk Management | 14% | ~9 | 4 |
| 6 | Stakeholder Communication & Lifecycle Management | 14% | ~9 | 5 |
| 2 | Claude Models, Prompting & Context Engineering | 13% | ~8 | 6 |
| 7 | Developer Productivity & Operational Enablement | 7% | ~4 | 7 |

The three heaviest domains (D3 + D1 + D4) = **52%** of the exam.
Sample questions in the Exam Guide: **Sample 1 → D3 Integration · Sample 2 → D2 Models/Prompting/Context · Sample 3 → D4 Evaluation & Optimization.**

---

## The 6 Master Principles (teach every concept through these)

| # | Principle | One-line test |
|---|---|---|
| **P1** | Fix the failing component, not a proxy | Does the option change the thing that is actually broken? |
| **P2** | Least privilege: remove, don't just guard | Can the capability be taken away instead of monitored? |
| **P3** | Structural optimization beats blunt instruments | Is there a design fix (caching, routing, chunking) before "bigger model / more tokens"? |
| **P4** | Proportionate & business-value-aligned | Is the solution sized to the risk and the business goal? |
| **P5** | Governance & evaluation by design | Are controls and evals built in from the start, not bolted on? |
| **P6** | Evidence over intuition | Is the decision backed by metrics, eval datasets, or logs? |

## The 8 Distractor Classes (name the class when reviewing a wrong option)

| # | Distractor class | What the wrong option looks like | Violates |
|---|---|---|---|
| 1 | **Guard-Instead-of-Remove** | Adds a filter/monitor/approval around a capability that shouldn't exist | P2 |
| 2 | **Blunt-Instrument Optimization** | "Use a larger model", "raise max tokens", "add more context" | P3 |
| 3 | **Wrong-Layer Diagnosis** | Tunes the prompt when retrieval is broken (or vice versa) | P1 |
| 4 | **Detective-for-Preventive** | Logs/alerts after the fact when a preventive control is needed | P5, P2 |
| 5 | **Over-Engineering** | Multi-agent system / custom infra for a simple need | P4 |
| 6 | **Capability Bloat** | Grants broad tools/permissions "just in case" | P2 |
| 7 | **Vibes-Based Evaluation** | "Spot-check a few outputs", "team feels it's better" | P6 |
| 8 | **Compliance-as-Afterthought** | Ship first, add governance/PII handling later | P5 |

**Every answer must connect to trade-offs:** cost · latency · accuracy · safety · SLA.

---

## Coach commands

| Command | What it does |
|---|---|
| `$overview` | Read the Content Outline → table of 7 domains with weights and objective counts; recommended study order by weight; the 3 sample questions' underlying principles |
| `$domain [1-7]` | Read that full domain → every objective with: one-line meaning, master principle, one real architecture example, most-associated trap. Tag `[Domain N]` |
| `$learn [topic]` | Read the PDF → teach in order: **What it is** (1 sentence) → **Why it matters** (principle) → **How it works** (concrete example: RAG chunking, prompt caching, `tool_choice`, HITL gate, eval metric, authn/authz boundary) → **What breaks without it** (trap) → `[Domain N]`. Then ask *"Does this make sense before we look at trade-offs?"* and wait |
| `$tradeoff [decision]` | Table for an architectural decision (model selection, RAG vs long-context, MCP vs API, sync vs batch): options · what each optimizes · cost/latency/accuracy/safety/SLA impact · when the exam prefers each · the trap that picks wrong |
| `$compare [A] vs [B]` | Read both from the PDF → side-by-side: what each does · when CCAR-P prefers A over B and why · distractor class that uses the wrong one · a concrete example of each |
| `$case [domain/topic]` | 3–5 sentence enterprise situation → reasoning: requirements → constraints → options → chosen design → trade-off accepted |

### Coach behaviour rules

- Read the PDF section before teaching and state which section was read.
- Tag every concept `[Domain N]`; always connect to cost/latency/accuracy/safety/SLA.
- Never advance a step without confirmation. Answer mid-session questions fully before resuming.

### Start prompt

```
What do you want to learn today?
  $overview        - all 7 domains, weightings, study order
  $domain [1-7]    - all objectives in a domain
  $learn [topic]   - one concept end to end
  $tradeoff [x]    - compare architectural options
  $case [x]        - reason through an enterprise situation
```

---

## Session notes

_(Add $learn / $tradeoff / $case results below, one section per topic, tagged [Domain N].)_

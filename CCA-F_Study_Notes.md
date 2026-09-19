# CCA-F Study Notes

> **Root of trust: the official Exam Guide** (`docs/Claude_Certified_Architect_-_Foundations_-_Exam_Guide.pdf`).
> Every section cites the **Task Statement** it comes from. When a practice site, NotebookLM, or any other source disagrees with the Exam Guide, **the Exam Guide wins**. Sections marked ⚠️ are questions where a practice site's answer key conflicts with the Exam Guide or the question is flawed.

### Exam Guide Task Statement index

| Domain | Task Statements |
|---|---|
| 1. Agentic Architecture & Orchestration | 1.1 Agentic loops · 1.2 Coordinator-subagent orchestration · 1.3 Subagent invocation & context passing · 1.4 Workflow enforcement & handoff · 1.5 Agent SDK hooks · 1.6 Task decomposition · 1.7 Session state, resumption, forking |
| 2. Tool Design & MCP Integration | 2.1 Tool interfaces & descriptions · 2.2 Structured error responses · 2.3 Tool distribution · 2.4 MCP servers integration · 2.5 Built-in tools (Read/Write/Edit/Bash/Grep/Glob) |
| 3. Claude Code Configuration & Workflows | 3.1 CLAUDE.md hierarchy · 3.2 Slash commands & skills · 3.3 Path-specific rules · 3.4 Plan mode vs direct · 3.5 Iterative refinement · 3.6 CI/CD |
| 4. Prompt Engineering & Structured Output | 4.1 Explicit criteria · 4.2 Few-shot · 4.3 Tool use & JSON schemas · 4.4 Validation/retry loops · 4.5 Batch processing · 4.6 Multi-pass review |
| 5. Context Management & Reliability | 5.1 Conversation context · 5.2 Escalation & ambiguity · 5.3 Error propagation · 5.4 Large codebase context · 5.5 Human review & calibration · 5.6 Provenance & uncertainty |

### Study Focus (based on Practice Test 1, Q1–Q60)

**Missed questions by domain** (excluding disputed keys where my answer matched the Exam Guide):

| Domain | Missed | Weak Task Statements | Priority |
|---|---|---|---|
| 4. Prompt Engineering & Structured Output | 8 of ~13 | 4.2 few-shot · 4.3 schema design (nullable, enum+other, normalization) · 4.4 semantic validation · 4.5 batch | 🔴 Highest |
| 5. Context Management & Reliability | 8 of ~21 | 5.1 conversation context · 5.2 escalation wording · 5.4 degradation/scratchpad · 5.5 segment accuracy · 5.6 provenance | 🔴 High |
| 1. Agentic Architecture & Orchestration | 7 of ~17 | 1.1 agentic loop · 1.3 Task tool / goals not procedures · 1.6 decomposition · 1.7 resume vs fresh | 🟠 Medium |
| 2. Tool Design & MCP Integration | 3 of ~11 | 2.1 split generic tools · 2.2 isError · 2.5 structure-first exploration | 🟢 Lower |
| 3. Claude Code Configuration & Workflows | not tested yet | 3.1–3.6 (CLAUDE.md, slash commands, path rules, plan mode, CI/CD) | ⚪ Read the Exam Guide before the next test |

Also not yet covered: **2.3** tool distribution, **2.4** MCP server config, **4.1** explicit criteria, **4.6** multi-pass review, **5.3** error propagation.

### Best Practices Cheat Sheet (recurring patterns)

1. **Guarantee → code, not prompts.** "Must / always / compliance" → hooks, `tool_choice` forcing, schema checks. Prompts are probabilistic. (1.4, 1.5, 4.3)
2. **Fix the root cause at the right layer.** Weak tool description → rewrite it; generic tool → split it; missing data in handoff → add structured fields. Avoid extra classifiers, routers, regex, or post-processing. (2.1, 4.3)
3. **Structured data beats text.** Errors (`isError`, `retryable`), claim→source mappings, dates, case facts, and `calculated_total` all belong in fields, never embedded in prose. (2.2, 4.4, 5.1, 5.6)
4. **Preserve, don't reconstruct or hide.** Keep provenance and conflicting values with attribution. Never auto-correct, silently map to "closest", return empty on error, or delete old data. (4.3, 5.6)
5. **Schema design:** nullable when data may be absent (+ "return null if not stated"), enum + "other" + detail for growing categories, format normalization rules in the prompt. (4.3)
6. **Few-shot = consistency and judgment calls.** Use for format, splitting and granularity, varied document layouts. Not for deterministic rules (use checks). (4.2)
7. **Retry only if the answer exists in the input.** Format errors → retry with feedback; missing information → retries can't help. (4.4)
8. **Context hygiene:** trim tool outputs to relevant fields; scratchpad for long sessions; summary + fresh subagent when degrading; always send full history (the API is stateless). (5.1, 5.4)
9. **Resume vs. fresh vs. fork:** context still valid → `--resume` (tell it what changed); stale tool results → new session + injected summary + fresh calls; compare alternatives → `fork_session`. (1.7)
10. **Orchestration:** hub-and-spoke; coordinator passes context explicitly in prompts; needs `Task` in `allowedTools`; give goals + quality criteria, not procedures; parallelize only independent work, from the coordinator; skip subagents for simple requests. (1.2, 1.3)
11. **Agentic loop is model-driven:** tool result → appended to conversation → model decides next action. Not decision trees or pre-planned sequences. (1.1)
12. **Exploration:** dependent steps → dynamic decomposition; entry points → follow imports; find all names before searching callers; abstraction before implementations. (1.6, 2.5)
13. **Escalation:** explicit human request → escalate immediately; frustrated but resolvable → offer to fix now or escalate; policy gap or no progress → escalate; never sentiment scores. Hand off a structured summary (customer ID, root cause, amount, recommended action). (1.4, 5.2)
14. **Human review & batch:** calibrated field-level confidence + validate per segment before automating; Batch API only when latency-tolerant (interval + 24h ≤ SLA), resubmit only failed `custom_id`s. (4.5, 5.5)
15. **Exam technique:** read every constraint ("evolving", "explicitly", "guaranteed", "both goals"); prefer the option that mirrors Exam Guide wording; one-size-fits-all or arbitrary numbers are usually traps.

### Disputed practice-site answers (follow the Exam Guide)

| Question | Site's key | Exam Guide says choose | Task Statement |
|---|---|---|---|
| Q44 Query routing, "uneven & evolving" | Fast-track factual queries | Coordinator analyzes each query dynamically | 1.2 |
| Q49 Customer demands "a real person NOW" | Ask one targeted question | Escalate immediately, no investigation first | 5.2 |
| Q57 Multi-issue session near context limit | Narrative summary of earlier turns | Structured issue data in a separate context layer | 5.1 |
| Q60 Returning customer after 4 hours | "Start… inject summary" (vs. near-identical "Create… with summary") | Same concept; prefer "inject" wording | 1.7 |

## Task Decomposition: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Unknown scope, each step depends on the last (debugging, root cause) | **Dynamic, adaptive** subtasks |
| Known, repeatable steps (e.g., review every file against the same checklist) | **Fixed pipeline / prompt chaining** |
| Independent subtasks with no shared dependency | **Parallel subagents** |

**Quick check:** If you'd need the result of one subtask to decide what the next one is, don't run them in parallel.


**Source check (official Exam Guide, Task Statement 1.6):**
- "When to use fixed sequential pipelines (prompt chaining) versus dynamic adaptive decomposition based on intermediate findings"
- "The value of adaptive investigation plans that generate subtasks based on what is discovered at each step"
- "Selecting task decomposition patterns appropriate to the workflow: prompt chaining for predictable multi-aspect reviews, dynamic decomposition for open-ended investigation tasks"

### Example question (missed)

**Q:** An agent must investigate intermittent 500s on an API endpoint in a 200+ file codebase, tracing through routing, middleware, business logic, and the database. Components involved are unknown. Best decomposition?

- ❌ Fixed sequence upfront: too rigid; can't follow unexpected leads.
- ❌ Parallel worker per layer (my answer): layers depend on each other, splitting by layer is a guess, and clues between layers get lost.
- ❌ Full plan before any exploration: can't map code paths you haven't seen yet, and the docs may be out of date.
- ✅ **Dynamically generate subtasks based on each discovery**: start at the route and follow the actual call chain.

---

## File Editing Tools: Exam Rule of Thumb

| Situation | Best tool |
|---|---|
| Small, targeted change with a unique anchor | **Edit** |
| Edit can't find a unique match (repetitive file) | **Read, then Write the full file** |
| Same change needed at every occurrence | **Edit with replace_all** |
| Creating a new file | **Write** |

**Tips:** `replace_all` is a trap unless the question says "every occurrence." A Bash workaround is usually wrong when a built-in tool can do the job.


**Source check (official Exam Guide, Task Statement 2.5):**
- "Read/Write for full file operations; Edit for targeted modifications using unique text matching"
- "When Edit fails due to non-unique text matches, using Read + Write as a fallback for reliable file modifications"

### Example question (answered correctly)

**Q:** An agent must insert a helper function between two existing functions in a 150-line module. Edit fails because the file's repetitive docstrings and patterns mean old_string never matches uniquely. Most reliable approach?

- ❌ Append with a Bash heredoc: lands at the end of the file, not between the two functions.
- ✅ **Read the file, insert the function in the right place, Write the updated file**: no text matching needed, and cheap for a small file.
- ❌ Edit with a 30+ line old_string: fragile, and may still not be unique.
- ❌ Edit with replace_all: inserts the function at every match and breaks the module.

---

## Claude Code Session Management: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Resume a **specific** earlier session (you know its name) | `claude --resume <name>` |
| Not sure which session; want to pick from a list | `claude --resume` (interactive picker) |
| Continue the **most recent** conversation (nothing done since) | `claude --continue` |
| Old context is stale or irrelevant | Start fresh (optionally with a summary) |

**Tips:** "Specific session" + "worked on other things since" = `--resume`, not `--continue`. A UUID option is a distractor when a simple name is available.


**Source check (official Exam Guide, Task Statement 1.7):**
- "Named session resumption using --resume <session-name> to continue a specific prior conversation"
- "Using --resume with session names to continue named investigation sessions across work sessions"

### Example question (answered correctly)

**Q:** An engineer built up 2 hours of context yesterday in a session named "auth-deep-dive". Since then she has worked on three other codebases. How should she continue that investigation?

- ❌ Start fresh and re-read the files: throws away 2 hours of built-up context.
- ❌ `--continue`: loads the most recent conversation, which is now one of the other codebases.
- ❌ `--session-id` with a UUID from the transcript file: awkward and error-prone. It's for setting an ID, not the simple way to resume.
- ✅ **`--resume auth-deep-dive`**: loads exactly that session by name.

---

## Resuming Subagents After Changes: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Prior context mostly valid, a few **known** changes since | **Resume + tell it exactly what changed** |
| Nothing changed since the interruption | Resume as-is |
| Prior context largely stale, or the transcript is huge or noisy | Fresh agent + concise summary of findings |
| Pasting a whole old transcript into a new prompt | Almost never (wastes context) |

**Tips:** Resuming keeps the context; the agent doesn't know about changes made while it was away. "The understanding still holds, so no need to mention it" is a trap: stale file or function names lead to wrong reads and wrong conclusions.


**Source check (official Exam Guide, Task Statement 1.7):**
- "The importance of informing the agent about changes to previously analyzed files when resuming sessions after code modifications"
- "Choosing between session resumption (when prior context is mostly valid) and starting fresh with injected summaries (when prior tool results are stale)"
- "Informing a resumed session about specific file changes for targeted re-analysis rather than requiring full re-exploration"

### Example question (missed)

**Q:** An exploration subagent spent 30 minutes on a legacy payment system (47 files read, data flows documented). The connection dropped, and meanwhile a teammate merged a PR renaming two utility functions. The engineer wants to continue. Most effective approach?

- ✅ **Resume from the transcript and tell it about the renamed functions**: keeps 30 minutes of context and fixes the one stale detail.
- ❌ Resume without mentioning the changes (my answer): the agent still believes the old names, so it will search for functions that no longer exist or report wrong call paths.
- ❌ Fresh subagent with the full prior transcript in the prompt: bloats the context and still doesn't mention the renames.
- ❌ Fresh subagent with a summary: loses detail it already had, and is unnecessary when only two functions changed.

---

## Context Degradation & Switching Topics: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Long session; answers turn **generic** ("typical patterns") instead of naming specific classes found earlier | Sign of context degradation: **summarize key findings, move the new task to a fresh subagent with that summary** |
| New subtask is a different area that only needs a few facts from the old work | **Subagent + concise summary** (keeps its context clean and focused) |
| Old context is irrelevant to the new task | `/clear` or a fresh session |
| Context still sharp and the task is the same topic | Continue in the current context |

**Tips:** The fix for degraded context is a *clean context seeded with a distilled summary*, not better prompts in the same bloated context. Watch for two traps: `/clear` throws away findings you still need, and an isolated subagent with "synthesize later" puts the integration burden back on the degraded main context.


**Source check (official Exam Guide, Task Statement 5.4):**
- "Context degradation in extended sessions: models start giving inconsistent answers and referencing 'typical patterns' rather than specific classes discovered earlier"
- "Summarizing key findings from one exploration phase before spawning sub-agents for the next phase, injecting summaries into initial context"

### Example question (missed)

**Q:** After 25 minutes exploring a game engine's rendering subsystem, the agent is asked how physics integrates with rendering for collision debug overlays. Recent answers cite "typical rendering patterns" instead of the specific VulkanPipeline and FrameGraph classes it found. Most effective approach?

- ✅ **Summarize the key rendering findings, then spawn a physics subagent with that summary in its initial context**: it starts clean, keeps the specific facts, and can connect physics to the real classes.
- ❌ Continue in the current context with targeted prompts naming the classes (my answer): treats the symptom. The context is still bloated, so detail keeps fading as physics exploration adds more.
- ❌ `/clear` and start fresh from CLAUDE.md paths: loses the rendering knowledge the integration question depends on.
- ❌ Independent physics subagent, then manually synthesize: the subagent lacks rendering context, so it can't find the integration points, and synthesis falls back on the degraded main context.

---

## Finding All Usages (Aliases / Wrappers): Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Function is re-exported or **renamed** by wrappers | **Read the definition + wrapper modules first to list every exposed name, then Grep for each name** |
| Function has one name, no aliases | Grep for that name |
| Need to understand *intended* usage (not find every caller) | Docs can help, but they're never a complete source |
| Many files import a module | Don't read every importing file; target Grep at the exact names |

**Tips:** Before a destructive change (removing or renaming), completeness beats speed. Pattern: **discover all names (Read) → search each one (Grep)**. Grepping only the original name misses callers of the aliases, and docs are often incomplete or stale.


**Source check (official Exam Guide, Task Statement 2.5):**
- "Tracing function usage across wrapper modules by first identifying all exported names, then searching for each name across the codebase"
- "Selecting Grep for searching code content across a codebase (e.g., finding all callers of a function…)"

### Example question (answered correctly)

**Q:** Before removing `calculateTax` from a core library, the agent must find every caller. Wrapper modules re-expose it under other names (e.g., `computeOrderTax` in the orders module). Most reliable strategy?

- ❌ Grep for all importers, then read each file: slow and noisy (many importers never use this function), and reading by hand is error-prone.
- ❌ Grep only the original name: misses every caller that uses `computeOrderTax` and other aliases.
- ❌ Search the project docs: docs show intended usage, not a complete, current list of callers.
- ✅ **Read the library and wrappers to list every exposed name, then Grep for each name**: complete and efficient.

---

## Long Sessions & Persistent Memory: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Long exploration; agent forgets or contradicts earlier findings, users repeat context | **Scratchpad file of key findings, re-read for later questions** |
| Switching to a new area mid-session | Summary + fresh subagent (see Context Degradation) |
| Want to "fix" it with a bigger context window | Trap: a bigger window doesn't stop detail fading in long contexts |
| Want to wipe context on a timer | Trap: throws away the knowledge you're trying to keep |
| Pre-summarize every file upfront | Trap: expensive, and loses detail before you know what matters |

**Tips:** Persist findings *outside* the context window (a file), written *as you discover them*, and consult it on demand. Summaries made before exploration can't know what will matter. Pairs with the previous question: *scratchpad = ongoing memory within the task; summary + subagent = handoff to a new task.*


**Source check (official Exam Guide, Task Statement 5.4):**
- "The role of scratchpad files for persisting key findings across context boundaries"
- "Having agents maintain scratchpad files recording key findings, referencing them for subsequent questions to counteract context degradation"

### Example question (missed)

**Q:** In 30+ minute exploration sessions, the agent gives inconsistent answers about code structure it discussed earlier, and engineers keep repeating context about modules already explored. Most effective fix?

- ✅ **Agent keeps a scratchpad file of key findings and refers to it for later questions**: findings survive context growth and stay consistent.
- ❌ Higher-capacity model tier with more context: more room doesn't solve early details fading or getting lost in a long context.
- ❌ Automatically clear context every 15 minutes: destroys the explored knowledge, so users repeat even more.
- ❌ Summarize every source file before exploration (my answer): costly upfront work, the summaries are generic and drop details found later, and they don't record the *findings* discovered during exploration.

---

## Tool Descriptions & Tool Selection: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Agent ignores a working tool and uses generic tools instead | **Improve the tool description**: when to use it, why it beats the alternatives, inputs and outputs |
| Agent confuses two similar tools | Sharpen the descriptions to spell out how they differ and when to use each |
| Tempted to remove other tools to force usage | Trap: over-restrictive, breaks legitimate uses |
| Tempted to add a classifier or router in front | Trap: overengineered; fix the root cause first |
| "Accept it as expected behavior" | Trap: ignores a fixable problem |

**Tips:** The model picks tools **based on their descriptions**. A one-line description gives it no reason to prefer the tool. A good description covers *what it does, when to use it (and when not to), why it's better than the alternatives* (e.g., AST-aware and updates all references), *input format, and output*. Fix the prompt or description before adding infrastructure.


**Source check (official Exam Guide, Task Statement 2.1):**
- "Tool descriptions as the primary mechanism LLMs use for tool selection; minimal descriptions lead to unreliable selection among similar tools"
- "The importance of including input formats, example queries, edge cases, and boundary explanations in tool descriptions"
- "Writing tool descriptions that clearly differentiate each tool's purpose, expected inputs, outputs, and when to use it versus similar alternatives"

### Example question (answered correctly)

**Q:** An MCP server with refactoring tools (`extract_function`, `rename_variable`, `inline_function`) is connected and working, but the agent still refactors with Write and sed. Each tool's description is minimal, e.g., "extract_function: extracts a function from code." Most effective way to improve adoption?

- ❌ Request classifier that routes refactoring requests to MCP: extra infrastructure that doesn't fix why the agent doesn't pick the tools.
- ❌ Remove the Write tool for refactoring sessions: blunt, and breaks legitimate file writes.
- ✅ **Improve the tool descriptions: when each tool beats text manipulation, plus expected inputs and outputs**: fixes the root cause.
- ❌ Accept it, since sed is more predictable: text-based refactoring is error-prone (misses references, scope), and the problem is fixable.

---

## Context Filling Up Mid-Investigation: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Mid-task, accuracy dropping, work remaining (more files, unfinished trace) | **Subagent explores the remaining work, seeded with a summary of patterns found so far** |
| Switching to a *different* area after long exploration | Summary + subagent (same idea) |
| Long session, need consistent recall of findings | Scratchpad file |
| `/clear` and re-read "critical files" | Trap: loses findings, and re-reading refills context |
| "Just use Grep to load less" | Trap: helps prevention, but doesn't fix context that's *already* degraded and doesn't trace data flow |

**Tips:** Degraded context + remaining exploration → **delegate to a subagent with a distilled summary**. The subagent's clean context does the heavy reading, and only its condensed results come back to the main agent, which keeps coordinating the overall investigation.

**Summary + subagent vs. summary file + fresh context:** Both carry findings forward. The subagent version also keeps the main conversation intact to combine results, while starting fresh abandons the coordinating context.


**Source check (official Exam Guide, Task Statement 5.4):**
- "Subagent delegation for isolating verbose exploration output while the main agent coordinates high-level understanding"
- "Summarizing key findings from one exploration phase before spawning sub-agents for the next phase, injecting summaries into initial context"

### Example question (answered correctly)

**Q:** Investigating error handling across 15 files of a legacy payment module. After 8 files, the agent is less accurate, forgetting patterns it found, and hasn't yet found all test files or traced the full data flow. Most effective approach?

- ✅ **Spawn a subagent for the remaining files, with a summary of discovered patterns as its initial context**: clean context for the remaining work, keeps prior findings, main agent combines the results.
- ❌ Write a summary to a file and start fresh: reasonable, but it drops the main coordinating context. The subagent is the more effective pattern.
- ❌ `/clear` and re-read only critical files: loses findings and repeats work.
- ❌ Grep for function names in the remaining files: reduces loading, but doesn't fix the forgetting already happening and can't trace the full data flow.

---

## Forking Sessions to Explore Alternatives: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Shared prior analysis, then **two or more alternative directions** to compare | **fork_session**: one branch per approach, each keeps the full prior context |
| Continue one line of work from a specific session | `--resume <name>` |
| Prior context stale or too large | Fresh session + summary |
| Explore alternatives one after another in one thread | Trap: approach A's details bias approach B and fill up the context |
| Manually recreating context in a new session | Trap: lossy, slow, error-prone |

**Tips:** Fork = *same starting point, independent branches*. Each branch inherits the full analysis without copying it by hand, and the branches don't contaminate each other. That makes the comparison fair.


**Source check (official Exam Guide, Task Statement 1.7):**
- "fork_session for creating independent branches from a shared analysis baseline to explore divergent approaches"
- "Using fork_session to create parallel exploration branches (e.g., comparing two testing strategies or refactoring approaches from a shared codebase analysis)"

### Example question (answered correctly)

**Q:** Yesterday the agent analyzed a legacy auth module and found two refactoring approaches (extract a microservice vs. refactor in place). Today the engineer wants specific code changes proposed for each before deciding. Most effective structure?

- ❌ Two fresh sessions with a manual summary: loses detail from yesterday's analysis.
- ❌ Resume for approach 1, new session with recreated context for approach 2: the two approaches start from unequal context, and recreating it by hand is lossy.
- ✅ **fork_session into two branches from yesterday's analysis, one approach per fork**: both get the full context, kept separate.
- ❌ Resume and explore both one after another in the same thread: the approaches bleed into each other and the context fills up.

### Example question 2 on forking (missed)

**Q:** The agent analyzed a complex service module (23 source files, request flows, error handling patterns). A developer wants to develop two testing strategies independently to compare trade-offs: end-to-end tests with mocked external services vs. snapshot tests. How should you manage sessions?

- ❌ Continue in the original session, one strategy after the other: not independent. The first strategy biases the second, and the context fills up.
- ❌ Two fresh sessions that each re-read the source files (my answer): wastes the analysis already done (23 files, traced flows, patterns). Re-reading costs time and tokens, and the two sessions may reach *different* understandings, so the comparison isn't fair.
- ✅ **Resume the analysis session with fork_session, one branch per strategy**: both inherit the complete analysis at no cost and develop independently.
- ❌ Export key findings to a file for two new sessions: a summary loses detail (exact flows, edge cases) that matter for writing tests.

**Lesson:** "Start fresh and re-read" is almost never right when a finished, valid analysis exists. **Reuse the context (resume/fork), don't rebuild it.**

---

## Exploring a Large Unfamiliar Subsystem: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Must understand an architecture spanning many files (thousands of lines) before changing it | **Structure first: imports and class hierarchy → Read the base class/interface → trace only the relevant implementations** |
| Need to find specific call sites or names | Grep (then Read targeted sections) |
| Tempted to Read every file one by one | Trap: floods the context, and detail fades before you reach the end |
| Keyword Grep + tiny line ranges | Trap: finds fragments with no understanding of the design (interfaces, base classes, decorators) |
| Glob by filename, read the largest first | Trap: file size ≠ importance; the naming guess may miss decorators and middleware |

**Tips:** Understand the **abstraction before the implementations**. The base class/interface tells you the contract every implementation follows, so you only need to read the implementations relevant to your change. This is *incremental, top-down exploration*: map the structure, read the core, then drill into the specifics.


**Source check (official Exam Guide, Task Statement 2.5 (closest match; base-class-first detail is not stated verbatim)):**
- "Building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront"

### Example question (missed)

**Q:** Before adding a new cache invalidation trigger, the agent must understand the caching layer. Grep shows it spans 15 files (decorators, middleware, service classes, ~8,000 lines). Most effective next step while managing context?

- ✅ **Analyze imports and class hierarchies to find the base cache class, Read it to understand the interface, then trace the specific invalidation implementations**: builds a mental model cheaply, and reads only what matters.
- ❌ Grep "invalidate" / "expire" and Read only those lines: scattered snippets miss the design and the hooks (where a new trigger belongs).
- ❌ Glob for cache.py / caching/ and read the largest files first: size-based priority is arbitrary, and filename patterns may miss relevant files.
- ❌ Read all 15 files one after another (my answer): ~8,000 lines fill the context, early details fade, and most of the content is irrelevant to invalidation.

---

## Mapping a Flow in a Huge Codebase (800+ files): Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Understand a flow (auth, payments, requests) across a huge multi-service codebase | **Grep for entry points → Read those files → follow imports and calls incrementally** |
| Subsystem already located, need its design | Base class/interface first, then implementations (previous section) |
| Read every file matching broad keywords | Trap: "auth"/"token" match hundreds of files and blow the context |
| Ask the user which files matter | Trap: a new engineer doesn't know. The agent should discover it |
| Parallel subagents per service | Trap: the flow crosses services, so splitting by service breaks the trace |

**Tips:** **Start narrow at the entry points, then expand by following the real code path.** Each read decides the next one (same idea as dynamic decomposition in the first section). Docs like CLAUDE.md or README can help, but aren't a complete strategy.


**Source check (official Exam Guide, Task Statement 2.5):**
- "Building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront"

### Example question (answered correctly)

**Q:** A new engineer wants to understand the authentication and authorization architecture before making security improvements. The codebase has 800+ files across multiple services. Most effective exploration strategy, given the built-in tools and context limits?

- ✅ **Grep for auth entry points, read those files, follow imports and calls to map the flow incrementally**: targeted, and context grows only with what's relevant.
- ❌ Read CLAUDE.md and README, then ask the engineer for 10–15 key files: the engineer is new and doesn't know. Pushes discovery onto the user.
- ❌ Read every file containing "auth", "login", "permission", "token": far too many matches, noisy, blows the context.
- ❌ Parallel subagents per service, then combine: auth crosses service boundaries. Splitting by service loses the connected flow (same trap as Q1).

### Example question 2 on tool descriptions (answered correctly)

**Q:** A local MCP server offers `analyze_dependencies`, `find_dead_code`, `calculate_complexity`. The agent still uses Grep for dependency questions, even when users say "code dependencies". The description only says: "returns a dependency graph by analyzing imports." Most effective way to improve tool selection?

- ✅ **Expand the tool descriptions and outputs (e.g., "Builds dependency graph with list_imports, direct_circular_deps") to clearly show how it differs from Grep**: the agent sees what it can do that Grep can't (graph, cycles).
- ❌ Add routing rules to the system prompt: a brittle workaround. Descriptions are where tool selection happens, and routing rules don't cover every phrasing.
- ❌ Remove Grep: Grep is still needed for plenty of other tasks.
- ❌ Split into granular tools: more tools with the *same* weak descriptions still overlap with Grep, and it adds complexity.

**Lesson (2nd time seen):** Tool not chosen → **fix the description first**. State what it returns and what makes it *different from the generic tool*. Routing rules, removing tools, and restructuring tools are distractors.

---

## Structured Extraction with Conflicting Values: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Document has several legitimate values for one field (original vs. amendment, versions, revisions) | **Redesign the schema: capture every value with source location + effective date** |
| Model picks one value inconsistently | Schema forces it to record *all* candidates, so nothing is silently dropped |
| "Always take the latest" prompt rule | Trap: dates can be ambiguous, amendments can be partial or later reversed, and you lose the audit trail |
| Classifier deletes superseded sections first | Trap: an error-prone extra step that permanently destroys evidence |
| Pattern-match afterwards and send to manual review | Trap: doesn't fix extraction, just offloads it to humans |

**Tips:** When the *data itself* has more than one valid value, the fix is in the **output structure**, not in prompt rules or pre/post-processing. Record provenance (where it came from, when it applies) so later logic or a reviewer can decide what's in effect. *Fix at the schema level before adding pipeline stages.*


**Source check (official Exam Guide, Task Statement 4.4, 5.6):**
- 4.4: "adding 'conflict_detected' booleans for inconsistent source data"
- 5.6: "Completing document analysis with conflicting values included and explicitly annotated, letting the coordinator decide how to reconcile before passing to synthesis"

### Example question (missed)

**Q:** Contracts often contain amendments (original clause "30-day payment terms", Amendment 1 changes it to "45 days"). The model extracts one or the other, inconsistently, with no indication of which applies. Most effective way to improve accuracy?

- ✅ **Redesign the schema so amended fields capture multiple values, each with source location and effective date**: nothing lost, provenance kept, and the value in effect can be determined reliably.
- ❌ Prompt: always extract the most recent amendment: fragile (ordering and dates are ambiguous, partial amendments) and throws away history.
- ❌ Pre-classifier removes superseded sections: another model step that can misclassify and delete needed terms.
- ❌ Pattern-match for amendments afterwards and flag for manual review (my answer): catches the symptom *after* the inconsistent output, doesn't improve extraction, and sends a frequent case to humans (doesn't scale).

---

## Allocating Limited Human Review: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Semantic errors pass schema validation; reviewers can check only a fraction | **Field-level confidence scores + review thresholds calibrated on a labeled validation set** |
| Want to measure overall accuracy / find error patterns | Random sampling (good for *monitoring*, poor for *catching* errors) |
| Review by document formatting anomalies | Trap: a proxy guess; errors like "30 minutes" in a quantity field happen in normal documents too |
| Review only empty or "not found" fields | Trap: those are *visible* gaps; the problem is *wrong values that look valid* |
| Using raw model confidence without calibration | Trap: raw confidence isn't reliable; calibrate against labeled ground truth |

**Tips:** Send humans to where errors are **most likely**, using a signal *validated against real labels*. Field-level beats document-level (more precise). Calibration turns "the model says 0.7" into "below 0.7, the error rate is X%," so the threshold can be set to fit the 20% capacity.


**Source check (official Exam Guide, Task Statement 5.5):**
- "Field-level confidence scores calibrated using labeled validation sets for routing review attention"
- "Having models output field-level confidence scores, then calibrating review thresholds using labeled validation sets"
- "Stratified random sampling for measuring error rates… and detecting novel error patterns" (= monitoring, not targeting)

### Example question (answered correctly)

**Q:** After deployment, 12% of extractions have semantic errors that pass JSON schema validation (e.g., "30 minutes" in an ingredient quantity field). Reviewers can check only 20% of extractions. How to allocate reviewer attention most effectively?

- ❌ Review all documents with formatting anomalies: a weak proxy; misses errors in normally formatted documents.
- ✅ **Field-level confidence scores, with review thresholds calibrated on a labeled validation set**: targets the likely errors, with a threshold tuned to capacity.
- ❌ Random 20% sample: catches only ~20% of errors. Useful for tracking accuracy, not for targeting.
- ❌ Prioritize empty or "not found" fields: those are obvious gaps. Semantic errors are filled-in wrong values.

---

## Internal Consistency Checks in Extraction: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Extracted values should agree mathematically (line items vs. total, subtotal + tax = total) | **Schema adds a `calculated_total` (sum of extracted line items) next to the extracted total, plus an `is_total_consistent` flag → human review when false** |
| Showing the model examples of correct sums | Trap: few-shot examples don't *guarantee* consistency and can't detect when it fails |
| Automatically "fixing" line items to match the total | Trap: silently corrupts data. You don't know which value is wrong |
| Second validation model to reconcile | Trap: extra cost and complexity, and it can also be wrong. Simple arithmetic is deterministic |

**Tips:** For checks that are *deterministic* (math, date ordering, required relationships), build **self-verification into the output schema**: extract both sides, compute the check, flag mismatches. Never auto-correct without evidence; **flag and route to a human**. This also gives you a measurable error signal.


**Source check (official Exam Guide, Task Statement 4.3, 4.4):**
- 4.3: "strict JSON schemas via tool use eliminate syntax errors but do not prevent semantic errors (e.g., line items that don't sum to total, values in wrong fields)"
- 4.4: "Designing self-correction validation flows: extracting 'calculated_total' alongside 'stated_total' to flag discrepancies"

### Example question (missed)

**Q:** An invoice pipeline extracts line items, tax, and grand totals. In 8% of documents, extracted line items don't sum to the extracted total (the downstream accounting system catches the mismatches). Most effective improvement?

- ❌ Few-shot examples where line items sum correctly (my answer): shows the model what to do but enforces nothing. Mismatches still happen and still go undetected in the pipeline.
- ❌ Extract independently, then a separate validation model reconciles: another probabilistic step, more cost, and it may "reconcile" the wrong way.
- ❌ Post-processing auto-adjusts line items to match the total: hides the error. The total could be the wrong value, and data gets corrupted silently.
- ✅ **`calculated_total` from independently extracted line items, compared with the extracted total, with an `is_total_consistent` flag for human review**: catches every mismatch deterministically, keeps the original data, sends only the flagged 8% to humans.

---

## tool_choice: Forcing Tool Calls: Exam Rule of Thumb

| `tool_choice` | Behavior | Use when |
|---|---|---|
| `{"type": "auto"}` (default) | Claude decides: a tool or plain text | General agents |
| `{"type": "any"}` | Must call *some* tool, but Claude picks which | Any tool is fine, just no plain-text reply |
| `{"type": "tool", "name": "X"}` | **Must call exactly tool X** | **A specific step must always happen (e.g., extraction first)** |
| `{"type": "none"}` | No tool calls | Force a text-only answer |

**Tips:** Guarantees come from **API parameters, not prompt instructions**. "any" + a prompt saying "call X first" still lets Claude pick another tool. Tool *order* in the list does **not** set priority. For multi-step pipelines: **force the tool on the first turn**, then handle later steps (enrichment, answering) in following turns with normal `auto`.


**Source check (official Exam Guide, Task Statement 4.3):**
- "The distinction between tool_choice: 'auto' (model may return text instead of calling a tool), 'any' (model must call a tool but can choose which), and forced tool selection (model must call a specific named tool)"
- "Forcing a specific tool with tool_choice: {'type': 'tool', 'name': 'extract_metadata'} to ensure a particular extraction runs before enrichment steps"

### Example question (answered correctly)

**Q:** The pipeline has an `extract_metadata` tool with a JSON schema for paper details. In testing, the agent sometimes skips it and answers directly. Most reliable way to make sure metadata extraction always happens first?

- ✅ **`tool_choice: {"type": "tool", "name": "extract_metadata"}`, then handle enrichment in later turns after receiving the metadata**: guaranteed at the API level.
- ❌ `"any"` + a system prompt saying call extract_metadata first: forces *a* tool, not *this* tool. The prompt part is only a request.
- ❌ `"auto"` + put extract_metadata first in the list: tool order doesn't set priority, and auto still allows a direct answer.
- ❌ `"auto"` with tools so Claude "always prioritizes" metadata: auto guarantees nothing. This is the behavior that's already failing.

---

## Handling Batch Failures (Message Batches API): Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Some requests in a batch failed; results identify them by `custom_id` | **Resubmit only the failed ones, with a fix that targets the actual error** |
| `context_length_exceeded` (the *input* is too big) | **Chunk the documents into smaller pieces → process → combine the partial extractions** |
| Output cut off (`stop_reason: max_tokens`) | Increase `max_tokens` (it controls *output* length only) |
| Rerun the whole batch (caching, bigger model) | Trap: pays again for 97% that already succeeded |
| Prompt caching as a fix | Trap: lowers cost for repeated prefixes, but doesn't make oversized inputs fit |

**Tips:** Two principles: **(1) retry only what failed** (`custom_id` exists for exactly this), **(2) match the fix to the error type**. `max_tokens` = output limit, context window = input + output limit. Don't mix them up.


**Source check (official Exam Guide, Task Statement 4.5):**
- "custom_id fields for correlating batch request/response pairs"
- "Handling batch failures: resubmitting only failed documents (identified by custom_id) with appropriate modifications (e.g., chunking documents that exceeded context limits)"

### Example question (missed)

**Q:** A daily batch of 10,000 documents finishes. 300 (3%) failed with `context_length_exceeded`. The results file identifies each failure by `custom_id`. Most cost-effective way to handle the failures?

- ❌ Reprocess the entire batch with prompt caching (my answer): reruns the 9,700 successes for nothing, and caching doesn't shrink the 300 oversized inputs, so they fail again.
- ❌ Resubmit all 10,000 with a larger-context model tier: pays for 9,700 successes again at a higher price.
- ❌ Increase `max_tokens` for the 300: `max_tokens` limits *output*. The problem is *input* length, and raising it can make things worse.
- ✅ **Resubmit only the 300 failed documents, chunked into smaller pieces, then combine the partial extractions**: fixes the actual cause, pays only for the failures.

---

## Few-Shot Examples for Output Consistency: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Valid schema, but values vary in **format/normalization** ("cotton blend" vs. "Cotton/Polyester mix") or fields get skipped | **Few-shot: 2–3 complete input→output pairs showing the standard format** |
| Values must satisfy a **deterministic rule** (sums, totals) | Schema-level check + flag (few-shot alone is a trap there) |
| "temperature = 0" | Trap: reduces sampling variation, but the model still doesn't know *which* format you want |
| "Bigger model" | Trap: the issue is unclear expectations, not capability |
| "Make the field required" | Trap: forces *a* value (maybe invented or badly formatted), but doesn't standardize format |

**Tips:** A schema defines **structure** (field exists, type = string). It can't define **conventions** (how to write materials, what to do with blends). Examples *show* the convention, and complete examples also show that the field should be filled whenever the info is present.

**Contrast with the invoice-totals question:** few-shot was wrong there because a sum mismatch needs *guaranteed detection*. Here the goal is *consistent style*, which is exactly what examples teach.


**Source check (official Exam Guide, Task Statement 4.2):**
- "Few-shot examples as the most effective technique for achieving consistently formatted, actionable output when detailed instructions alone produce inconsistent results"
- "Using few-shot examples to demonstrate correct handling of varied document structures (inline citations vs bibliographies, methodology sections vs embedded details)"
- "Adding few-shot examples showing correct extraction from documents with varied formats to address empty/null extraction of required fields"
- "Creating 2-4 targeted few-shot examples for ambiguous scenarios…"

### Example question (missed)

**Q:** An extraction system parses e-commerce product descriptions into JSON (dimensions, weight, materials). Despite a well-defined schema, "materials" comes out inconsistently: "cotton blend" sometimes, "Cotton/Polyester mix" other times, and it's occasionally omitted even though the source clearly states it. Most effective way to improve consistency?

- ❌ temperature 0: less randomness, but no definition of the correct format. It may consistently produce the wrong style.
- ❌ More capable model: extraction isn't too hard for the model. It's missing a clear convention.
- ✅ **Few-shot: 2–3 complete input→output pairs with standardized material formats**: shows the exact normalization and that the field must be filled.
- ❌ Make "materials" required (my answer): fixes only the omissions (and may push the model to invent values when material is truly absent). It doesn't fix the inconsistent formats at all.

---

## Preventing Hallucinated Values in Extraction: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Schema already nullable, but model fills in plausible made-up values for info not in the source | **Explicit prompt instruction: return `null` when information is not directly stated in the source** |
| Schema doesn't allow null at all | Make fields nullable first (you can't return null if the schema forbids it) |
| "Make fields required / non-nullable" | Trap: the *opposite* fix. It forces the model to invent values when info is missing |
| "Second LLM call to verify" | Trap: extra cost and latency, and the verifier can be wrong. Fix the first call before adding another |
| "Bigger model" | Trap: the model hasn't been told null is expected, which isn't a capability problem |

**Tips:** Nullable in the schema only makes null *allowed*. The model still tends to be "helpful" and fill gaps. You have to **say explicitly** that null is the correct answer when the information is missing, and that inferring or guessing isn't acceptable. Pairs with the materials question: *required fields push the model to invent values*.


**Source check (official Exam Guide, Task Statement 4.3):**
- "Designing schema fields as optional (nullable) when source documents may not contain the information, preventing the model from fabricating values to satisfy required fields"
- Also 4.2: "The effectiveness of few-shot examples for reducing hallucination in extraction tasks"

### Example question (answered correctly)

**Q:** An event metadata extractor (date, location, organizer, attendee_count) uses a schema where every field is nullable. The model often outputs plausible but wrong values for fields not in the article, e.g., attendee_count = "500" with no attendance info in the source. Most effective way to reduce these false extractions?

- ❌ More capable model: hallucination isn't mainly about capability. The model lacks a clear instruction about missing data.
- ❌ Make all fields required (non-nullable): makes it worse by *forcing* invented values when data is absent.
- ❌ Second LLM call to verify values exist in the source: expensive extra step that is also imperfect. Fix the root cause first.
- ✅ **Prompt instruction: return null for any field not directly stated in the source**: simple, targets the exact behavior, and uses the nullable schema already in place.

### Example question 2 on few-shot (missed): varied document structures

**Q:** With tool use and strict schemas, JSON syntax errors are gone, but 5% of extractions have empty arrays or nulls for required fields (citations, methodology). Spot checks show the source documents *do* contain the info, but in varied formats: inline citations vs. bibliographies, methodology sections vs. details embedded in introductions. Most effective fix?

- ❌ Make the fields optional and flag for manual review: hides a fixable extraction failure and sends it to humans.
- ✅ **Few-shot examples from documents with varied structures, showing how to find citations in different formats and methodology across section types**: teaches the model *where and how* to look.
- ❌ Retry on empty required fields: the same prompt on the same document gives the same miss. Retrying doesn't add understanding.
- ❌ Regex post-processing for citation patterns and methodology keywords (my answer): brittle against varied formats (the very problem), and keyword matches are noisy. It bypasses the model instead of fixing it.

**Lesson:** When the info **is in the source** but the model **misses it because the format varies**, show examples covering those variations. Tool use and schemas fix *syntax*. Examples fix *recognition*. Regex can't handle the variety that causes the failure.

---

## Retry with Error Feedback: When It Works / When It Doesn't: Exam Rule of Thumb

| Failure type | Will retry + error feedback fix it? |
|---|---|
| **Format/structure** errors (wrong type, nested vs. flat, "1,234" vs. integer, datetime vs. date) | **Yes**: the info is there, the model just needs to reshape it |
| **Missing information**: data not in the input (e.g., full author list only in an external document) | **No**: no number of retries can create data that isn't there |
| Same prompt retried *without* feedback | Usually no, same miss again |
| Model doesn't recognize info because layouts vary | Better fixed with few-shot examples than retries |

**Tips:** Before choosing retries, ask: **"Is the correct answer actually in the input?"** If yes → retry with feedback works. If no → retries waste tokens and may push the model to *hallucinate*. Fix it by providing the missing source, allowing null/partial values, or flagging for review.


**Source check (official Exam Guide, Task Statement 4.4):**
- "The limits of retry: retries are ineffective when the required information is simply absent from the source document (vs format or structural errors)"
- "Identifying when retries will be ineffective (e.g., information exists only in an external document not provided) versus when they will succeed (format mismatches, structural output errors)"

### Example question (answered correctly)

**Q:** An extraction system retries on validation failure, appending the specific validation error to the prompt, which fixes most failures within 2–3 attempts. For which failure pattern would more retries be LEAST effective?

- ❌ Keywords as a nested object by category instead of a flat string array: structural, easy to fix with feedback.
- ❌ Citation counts as "1,234" instead of integers: type/format, easy to fix with feedback.
- ✅ **"et al." for co-authors when the full list exists only in an external document not in the input**: the information isn't there, so retrying can't recover it (and may trigger made-up names).
- ❌ ISO datetime "2023-03-15T00:00:00Z" instead of YYYY-MM-DD: format, easy to fix with feedback.

---

## Normalizing Inconsistent Source Formats: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Source uses mixed formats ("$12" vs "12.00", dietary icons vs text) and output must be uniform | **Strict output schema + explicit normalization rules in the prompt** (e.g., price → number 12.00, dietary → tags from a fixed list) |
| Normalization needs *understanding* (an icon means "vegan", a symbol means "spicy") | The model should normalize during extraction, since code can't interpret meaning |
| Extract as-is, normalize in post-processing code | Trap: code has to handle every variant, and semantic cases (icons, wording) can't be parsed reliably |
| One call per field | Trap: more cost and latency, loses cross-field context, doesn't define the format |
| Multiple attempts + majority vote | Trap: expensive, and the "most common" format isn't necessarily the correct one |

**Tips:** Tell the model **exactly what the output must look like** (schema = structure, prompt rules = conventions). Normalize *at extraction time*, where the model can understand context. Add examples (few-shot) if rules alone aren't enough.


**Source check (official Exam Guide, Task Statement 4.3):**
- "Including format normalization rules in prompts alongside strict output schemas to handle inconsistent source formatting"

### Example question (missed)

**Q:** A pipeline turns restaurant menus into JSON (item names, descriptions, prices, dietary tags). Menus format things inconsistently: prices as "$12" vs "12.00", dietary info as icons vs text. Most reliable approach?

- ❌ Separate extraction call per field: many calls, loses context, doesn't define the target format.
- ❌ Several attempts per document, pick the most common format: costly, and the majority can be consistently wrong.
- ✅ **Strict output schema + format normalization rules in the prompt**: one pass, consistent output, and the model applies meaning (icon → "vegetarian").
- ❌ Extract as-is, normalize in post-processing code (my answer): raw output stays inconsistent, code must anticipate every variant, and icon or wording meaning can't be handled by simple code, so normalization breaks on new formats.

---

## Enums with Open-Ended Categories: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Enum fails validation because real data has uncommon or **ever-growing** categories | **Add `"other"` to the enum + a `*_detail` free-text field for the specific value** |
| Categories are truly fixed and complete | Plain enum is fine |
| Force-map unknowns to the "closest" enum value | Trap: silently wrong data ("converted warehouse" → "house"), loses information |
| Switch to free-form string + post-processing | Trap: gives up the enum's consistency for *all* values, and normalization code must chase every variant |
| Keep expanding the enum as new types show up | Trap: never-ending maintenance; new types keep failing until the next update |

**Tips:** `enum + "other" + detail` is the standard **escape-hatch pattern**. You keep clean, predictable categories for the common cases, capture new values without validation failures or data loss, and can review the "other" details later to decide which ones become real enum values. Long-term, it **degrades gracefully** instead of breaking.


**Source check (official Exam Guide, Task Statement 4.3):**
- "Schema design considerations: required vs optional fields, enum fields with 'other' + detail string patterns for extensible categories"
- "Adding enum values like 'unclear' for ambiguous cases and 'other' + detail fields for extensible categorization"

### Example question (missed)

**Q:** Tool use with a JSON schema defines `property_type` as enum ['house', 'apartment', 'condo', 'townhouse']. 8% of extractions fail validation. Listings mention many uncommon types ("studio", "loft", "duplex", "mobile home", "tiny house", "converted warehouse"), and new types keep appearing. Most effective long-term solution?

- ❌ Free-form string + post-processing normalization: loses the enum's guarantees and moves the endless variety into code.
- ❌ Few-shot examples mapping unexpected types to the closest enum value (my answer): turns failures into *silent errors* ("loft" → "apartment"? "mobile home" → "house"?), loses the real type, and can't predict future types.
- ✅ **Add "other" to the enum + a `property_type_detail` string for specifics**: never fails validation, keeps the actual value, and keeps structured categories for common types.
- ❌ Keep expanding the enum + monitoring: reactive, with constant schema changes. New types fail until someone adds them.

---

## Validating Before Automating (Reducing Human Review): Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Aggregate accuracy looks high; planning to skip human review for high-confidence cases | **Break accuracy down by segment (document type, field) first**. Aggregates can hide weak segments |
| Choosing the best confidence threshold | Useful *after* confirming the confidence signal is reliable in every segment |
| Pilot sending unreviewed output straight to production | Trap: uses real downstream systems to find errors you could have found in existing labeled data |
| Check whether 97% meets downstream needs | Important, but still based on the *aggregate* number, which may not hold for each segment |

**Tips:** "97% overall" can be 99.5% on common invoices and 70% on rare contracts or one tricky field. **Aggregate metrics hide segment-level failures** (a Simpson's-paradox-style trap). You already have 3 months of 100% human-reviewed (labeled) data, so analyze it by segment *before* removing the safety net. Automate only segments that are consistently reliable.


**Source check (official Exam Guide, Task Statement 5.5):**
- "The risk that aggregate accuracy metrics (e.g., 97% overall) may mask poor performance on specific document types or fields"
- "Analyzing accuracy by document type and field to verify consistent performance across all segments before reducing human review"

### Example question (missed)

**Q:** A system has run with 100% human review for 3 months. Extractions with model confidence >90% have 97% accuracy overall. To reduce reviewer workload, you plan to automate high-confidence extractions. What validation step is most critical before deploying?

- ❌ Two-week pilot sending 25% of high-confidence extractions directly downstream and watching error reports: exposes production to errors, and error reports are delayed and incomplete. The labeled data already answers the question.
- ✅ **Analyze accuracy by document type and field to confirm high-confidence extractions perform consistently across all segments, not just in aggregate**: finds hidden weak spots before automating.
- ❌ Check that 97% meets requirements for all downstream systems: still relies on the aggregate figure that may not hold per segment.
- ❌ Compare accuracy at 85/90/95% thresholds to find the best cutoff (my answer): tunes one *global* threshold on aggregate numbers. A threshold that looks best overall can still let a weak document type or field through. Segment reliability comes first.

---

## Batch Scheduling vs. SLA (Message Batches API): Exam Rule of Thumb

**Formula:** worst-case latency = **batch interval (max wait before submission) + max processing time (24h)** + overhead/buffer ≤ SLA

| Batching strategy (SLA 30h, processing up to 24h) | Worst case | Verdict |
|---|---|---|
| Every 4h | 4 + 24 = **28h** | ✅ Meets SLA with a 2h buffer for submission delays and retries |
| Every 6h | 6 + 24 = **30h** | ❌ Zero margin, so any delay breaks a 99.9% target |
| End of day | Early docs wait ~8–10h+ → **32h+** | ❌ Violates SLA |
| Real-time API | Fast | ❌ Meets SLA but gives up the 50% discount (not cost-optimal) |

**Tips:** Plan for the **worst case** (the document that arrives right after a batch was submitted, and a batch that takes the full 24h), not the average. A high reliability target (99.9%) means **leave a buffer**. Don't design to exactly the limit. Batches API: 50% cheaper, up to 24h, no latency guarantee → only for non-urgent work.


**Source check (official Exam Guide, Task Statement 4.5):**
- "The Message Batches API: 50% cost savings, up to 24-hour processing window, no guaranteed latency SLA"
- "Calculating batch submission frequency based on SLA constraints (e.g., 4-hour windows to guarantee 30-hour SLA with 24-hour batch processing)"
- "Matching API approach to workflow latency requirements…"

### Example question (missed)

**Q:** Documents arrive continuously during business hours. You want the Message Batches API (50% discount, up to 24h processing). SLA: results within 30 hours of document arrival, with 99.9% reliability. Which batching strategy is most appropriate?

- ✅ **Submit batches every 4 hours**: worst case 4 + 24 = 28h, which leaves a safety buffer.
- ❌ Every 6 hours: 6 + 24 = 30h exactly, no room for any delay at a 99.9% target.
- ❌ One batch at end of day (my answer): a document arriving in the morning waits ~8–10h before submission, then up to 24h of processing = 32–34h > 30h SLA.
- ❌ Real-time API for everything: meets the SLA but loses the 50% savings, which was the goal.

### Example question 3 on few-shot (answered correctly): array granularity & explicit mentions

**Q:** Schema has `skills: string[]`. Monitoring shows: (1) compound phrases like "Python and SQL" sometimes kept as one entry, sometimes split; (2) implied but unstated skills sometimes appear; (3) similar documents produce very different array lengths (5–10 vs 40+). The prompt says only "Extract all skills mentioned." Most effective improvement?

- ❌ Richer schema {skill, confidence, source_quote}: adds metadata, but doesn't define splitting, explicitness, or granularity.
- ✅ **Few-shot examples showing compound-phrase handling, explicit-mention criteria, and the right entry granularity**: one fix addresses all three issues by *showing* the convention.
- ❌ Post-extraction taxonomy mapping + dedup: patches symptoms afterwards and can't remove hallucinated implied skills or fix inconsistent granularity at the source.
- ❌ Hard constraints "10–20 max, one per entry, only explicit": an arbitrary cap cuts real skills (or pads short docs), and rules alone are less clear than demonstrated examples.

**Lesson (few-shot, 3rd time):** Vague instruction + inconsistent *judgment calls* (how to split, what counts, how detailed) → **show examples** that demonstrate those decisions. Arbitrary numeric caps are a trap.

### Example question 2 on Batch API (answered correctly): mixed urgency workloads

**Q:** Two document types share one JSON schema: standard monthly reports (archived after processing) and urgent exception reports (must trigger business alerts within 30 minutes of receipt). Minimize API cost while meeting latency. How to architect the pipeline?

- ✅ **Route standard reports to the Batch API (50% savings), urgent exception reports to the real-time Messages API**: each workload uses the cheapest option that meets its latency needs.
- ❌ Everything real-time: meets latency but pays full price for reports that aren't time-sensitive.
- ❌ Everything in hourly batches, flag urgent ones later: batch processing can take up to 24h with no latency guarantee, so 30-minute alerts will be missed.
- ❌ Everything to Batch with custom_ids, alert when results arrive: same problem. "Delayed alerts" break the 30-minute requirement.

**Lesson:** **Route by latency requirement.** Batch API = 50% cheaper, up to 24h, *no latency SLA* → only for work that can wait. Anything with a short deadline (minutes) → real-time API. Same schema doesn't mean same pipeline.

---

## Generic vs. Purpose-Specific Tools: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| One generic tool with a free-text `instruction` param returns the wrong *kind* of output (summary vs. data vs. verification) | **Split into purpose-specific tools, each with a defined input/output contract** (`extract_data_points`, `summarize_content`, `verify_claim_against_source`) |
| Agent ignores a *working, well-scoped* tool | Improve its description (see Tool Descriptions section) |
| Single tool + `analysis_type` enum | Better than free text, but still one tool with one loose output shape. Callers can't rely on a per-mode contract |
| Coordinator pre-classifies requests | Trap: adds a layer that also guesses, and output shapes are still undefined |
| Longer description mapping phrasings → formats | Trap: can't list every phrasing, and free text stays ambiguous |

**Tips:** Ambiguity comes from the **interface design**. A free-text instruction lets each call mean anything, and an undefined output shape lets each result look like anything. Purpose-specific tools make the **intent explicit through tool selection** and the **result predictable through the output contract**, so downstream agents can rely on it.

**Contrast with the refactoring and dependency MCP questions:** there the tools were already well-scoped, and only their *descriptions* were weak → fix the description. Here the *tool itself* is too generic → redesign (split) it.


**Source check (official Exam Guide, Task Statement 2.1):**
- "Splitting generic tools into purpose-specific tools with defined input/output contracts (e.g., splitting a generic analyze_document into extract_data_points, summarize_content, and verify_claim_against_source)"
- "How ambiguous or overlapping tool descriptions cause misrouting"

### Example question (missed)

**Q:** A document analysis agent has one `analyze_document` tool (document + free-text instruction). "Extract the key financial metrics" often returns narrative summaries, and "summarize the methodology" sometimes returns raw data tables. The synthesis agent reports 35% of results need re-requests with clarified instructions. Most effective way to improve reliability?

- ❌ Richer tool description with examples mapping phrasings to output formats (my answer): still a single free-text interface, and you can't list every possible phrasing. The output shape is still not guaranteed.
- ❌ Keep one tool, add an `analysis_type` enum: clearer intent, but one tool still returns loosely defined output across very different jobs.
- ✅ **Split into `extract_data_points`, `summarize_content`, `verify_claim_against_source`, each with defined input/output contracts**: intent is set by which tool is chosen, and results have predictable structure.
- ❌ Coordinator pre-classifies each request: an extra step that can also misclassify, and doesn't define the outputs.

---

## Coordinator Can't Delegate: The Task Tool: Exam Rule of Thumb

| Symptom | Most likely cause |
|---|---|
| Coordinator *says* "I'll ask the X agent…" but never actually invokes it; subagent definitions are correct | **`Task` (the subagent-spawning tool) is missing from the coordinator's `allowedTools`** |
| Coordinator never mentions subagents at all | It may not know about them (definitions/descriptions missing) |
| Tool call cut off mid-JSON | `max_tokens` too low (you'd see truncation, not a clean sentence) |

**Tips:** In the Claude Agent SDK, subagents are defined via agent definitions, but they are **launched by calling the `Task` tool** (called `Agent` in newer SDK versions). Definitions describe *what* subagents exist; `allowedTools` decides *whether* the coordinator can call the tool that starts them. **Knowing about a subagent ≠ being able to invoke it.** Narrating an action without a tool call = the tool isn't available.


**Source check (official Exam Guide, Task Statement 1.3):**
- "The Task tool as the mechanism for spawning subagents, and the requirement that allowedTools must include 'Task' for a coordinator to invoke subagents"

### Example question (missed)

**Q:** A coordinator has AgentFunctions (agent definitions) configured for four specialized subagents, with appropriate descriptions and restrictions. In testing, it sometimes fails to delegate: it writes "I'll ask the web search agent to find sources" without actually invoking it. Most likely cause?

- ❌ System prompt hides the available subagent types (my answer): the coordinator clearly *knows* about the web search agent, since it names it. Knowledge isn't the problem; invocation is.
- ❌ `max_tokens` too low, truncating the Task call: truncation produces a cut-off response or incomplete tool call, not a clean sentence describing delegation.
- ❌ Context descriptions between coordinator and tools insufficient: vague, and it doesn't explain describing without calling.
- ✅ **`allowedTools` doesn't include "Task"**: the coordinator can describe delegation but has no tool to spawn subagents, so it just narrates.

---

## Passing Context Between Agents (Provenance / Dates): Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Synthesis agent misreads findings (e.g., calls 2022 vs. 2024 stats "contradictory" instead of a trend) | **Require subagents to include metadata (publication or data-collection date, source) in their structured outputs** |
| "Always trust newest, move older to an appendix" | Trap: throws away the trend, and newest isn't always most reliable. It's a blanket rule, not understanding |
| Auto-discard older data | Trap: destroys historical context that is the actual insight (growth 18% → 35%) |
| Restrict search to recent results | Trap: loses useful older data, and doesn't help with internal documents |

**Tips:** Subagents pass **results, not context**. The synthesis agent only knows what's in the handoff. If dates, sources, and scope aren't in the structured output, the downstream agent *can't* reason about them. Fix it by **enriching the handoff contract**, not by adding blanket rules or deleting data. (Same idea as the amendments question: keep provenance, don't discard.)


**Source check (official Exam Guide, Task Statement 5.6):**
- "Temporal data: requiring publication/collection dates in structured outputs to prevent temporal differences from being misinterpreted as contradictions"
- "Requiring subagents to include publication or data collection dates in structured outputs to enable correct temporal interpretation"

### Example question (missed)

**Q:** Researching "renewable energy adoption", the web search agent returns 2024 statistics (35%) and the document analysis agent extracts internal report data (2022: 18%). The synthesis agent wrongly flags them as contradictory instead of recognizing growth over time. What change best lets the synthesis agent interpret temporal differences correctly?

- ✅ **Require subagents to include publication or data-collection dates in their structured outputs**: gives the synthesis agent the information to see "18% (2022) → 35% (2024) = growth".
- ❌ Conflict-resolution agent that discards older data: loses the trend and adds complexity.
- ❌ Always treat the most recent data as authoritative, older findings in an appendix (my answer): a blanket rule that doesn't give the agent dates to reason with. It still can't tell *which* is newer without date metadata, hides the growth insight, and misfires when newer data is lower-quality or measures something different.
- ❌ Web search limited to the past 6 months: loses valid data, and doesn't fix the missing dates in internal documents.

---

## Crash Recovery in Multi-Agent Pipelines: Exam Rule of Thumb

| Approach | Fidelity | Context efficiency | Verdict |
|---|---|---|---|
| **Each agent persists a structured report to a known location; coordinator loads them and injects relevant state into agent prompts** | High (exact structured findings) | High (only relevant state injected) | ✅ Best balance |
| Each agent keeps and reloads its own state file independently | High per agent | OK | ❌ No central coordination: agents can't see each other's progress, and the coordinator can't plan what's left |
| Shared vector store + semantic search | ❌ Retrieval is fuzzy and may miss or mix findings | OK | ❌ Wrong tool for exact state restoration |
| Replay the coordinator's full conversation log | High | ❌ Huge, noisy, fills context | ❌ Inefficient |

**Tips:** Resuming needs **exact, structured checkpoints** (which documents are done, extracted data, partial patterns) plus a **coordinator that decides who needs what**. Structured reports = fidelity. Coordinator-selected injection = efficiency. Semantic search is for *finding relevant information*, not for *restoring precise workflow state*.

### Example question (answered correctly)

**Q:** A multi-agent research pipeline crashed after 12 of 28 documents. The web search agent had found sources, the document analysis agent had partially finished extraction, and the synthesizer had begun pattern identification. Resume without repeating work or losing fidelity. What state management approach best balances fidelity and context efficiency?

- ✅ **Each agent persists a structured report to a known location; on resume the coordinator loads them and injects relevant state into agent prompts**: exact state, centrally orchestrated, minimal context.
- ❌ Each agent keeps its own state file and reloads it independently: decentralized, and cross-agent dependencies and remaining-work planning are lost.
- ❌ Shared vector store with semantic search: approximate retrieval can miss or blur details. Not a reliable checkpoint.
- ❌ Persist the coordinator's full conversation log and give it to agents: fills the context with everything, which is inefficient.

**Source check (official Exam Guide):**
- *Task Statement 5.4* lists this exact pattern: "Structured state persistence for crash recovery: each agent exports state to a known location, and the coordinator loads a manifest on resume" and "Designing crash recovery using structured agent state exports (manifests) that the coordinator loads on resume and injects into agent prompts."
- *Task Statement 1.2*: hub-and-spoke, where the coordinator manages all inter-subagent communication and information routing (why independent per-agent reloads are wrong).
- *Out-of-Scope*: "Embedding models or vector database implementation details". Vector stores can show up as a distractor, but the exam won't test how they work.
- *Lost in the middle* (Domain 5 context management): long histories risk dropped findings (why replaying full logs is wrong).

---

## Handling Conflicting Findings & Uncertainty in Synthesis: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Credible sources disagree (e.g., analysts say $50B, a peer-reviewed study says ~$35B with a 95% CI) and the report either picks one arbitrarily or hedges vaguely | **Explicit report sections: well-established findings vs. contested analysis, keeping each source's original characterization** (who said it, method, confidence interval) |
| Numeric confidence calibration layer (0.0–1.0 weights) | Trap: invents false precision, blends different methods into one number, and loses the *why* |
| Subagents only pass findings that meet quality/diversity criteria | Trap: silently drops the conflict before synthesis sees it |
| Verification agent: accept only claims corroborated by 2+ sources | Trap: a single strong source (peer-reviewed study) gets rejected, and conflicts still aren't *presented* |

**Tips:** Don't **resolve** a real disagreement by picking or averaging, and don't **blur** it with vague hedging. **Present it transparently**, with attribution and each source's own wording and method, so the reader can judge. Hedge only where there's actual disagreement; state confirmed findings plainly.

**Source check (official Exam Guide, Task Statement 5.6):** "handle conflicting statistics from credible sources: annotating conflicts with source attribution rather than arbitrarily selecting one value", and "structuring reports with explicit sections distinguishing well-established findings from contested ones".

### Example question (answered correctly)

**Q:** Final reports handle uncertainty inconsistently: conflicting subagent findings are sometimes merged into one confident statement, other times over-hedged. Web search returns "Industry analysts estimate $50B market size (methodology…)", and document analysis returns "peer-reviewed study estimates ~$35B (95% CI)". The coordinator either picks one arbitrarily or writes vague ranges "depending on factors". What systematic approach best addresses this?

- ❌ Confidence calibration layer normalizing uncertainty to 0.0–1.0 and weighting: false precision; merges different methods into a number that loses meaning and attribution.
- ❌ Subagents only report findings meeting coverage/diversity/quality criteria: filters out information, hiding the conflict instead of handling it.
- ✅ **Synthesis agent uses explicit sections separating confirmed findings from contested analysis, preserving original source characterizations**: consistent structure, transparent conflicts, no arbitrary picking, no blanket hedging.
- ❌ Verification subagent accepting only claims corroborated by 2+ independent sources: discards valid single-source evidence and still doesn't explain disagreements.

---

## Delegating to Subagents: Goals vs. Procedures: Exam Rule of Thumb

| Delegation style | Result |
|---|---|
| **Research goals + quality criteria** (coverage breadth, source diversity, content type); subagent decides *how* | ✅ Adapts to unexpected situations while still meeting the bar |
| Step-by-step procedures (exact queries, fixed sequence) | ❌ Brittle: breaks when reality doesn't match the script |
| Step-by-step + fallback directives ("if search fails, try alternatives") | ❌ Patches one known failure; the next unexpected case still breaks. Still procedural |
| Vague goal only ("research it thoroughly") | ❌ Adaptable but no quality bar → inconsistent results |
| Task classification → different instruction sets | ❌ More procedures, still can't foresee every situation |

**Tips:** Tell the subagent **what "done well" looks like (goal + criteria)**, not **how to do it (steps)**. The right answer sits between the extremes: *not* micromanaged procedures, *not* vague goals. Adding more rules, fallbacks, or categories is still procedural thinking.

**Source check (official Exam Guide, Task Statement 1.3 skills):** "Designing coordinator prompts that specify research goals and quality criteria rather than step-by-step procedural instructions, to enable subagent adaptability."

### Example question (missed)

**Q:** The coordinator gives the web search subagent detailed step-by-step instructions: exact queries, source quality criteria (coverage breadth, source diversity), content type classifications. Instructions sometimes fail when the subagent hits unexpected situations. Most effective way to improve subagent adaptability?

- ✅ **Specify research goals and quality criteria (coverage breadth, source diversity, content type) instead of procedural instructions, letting the subagent determine execution**: keeps the quality bar, frees the subagent to adapt.
- ❌ Remove all procedural detail, delegating "research it thoroughly": loses the quality criteria, so results become inconsistent.
- ❌ Add explicit fallback directives to the detailed instructions (my answer): handles only the failure you predicted ("searches fail"). Other unexpected situations still break a rigid script. It adds procedure instead of fixing the procedural approach.
- ❌ Coordinator classifies tasks as analytical vs. exploratory and uses different instruction sets: more rigid scripts, and can't cover every unexpected situation.
- Related (Task Statement 1.1): "model-driven decision-making (Claude reasons about which tool to call next based on context)" vs. "pre-configured decision trees or tool sequences". Goal-based delegation is the model-driven side.

---

## Rendering Mixed Content Types in Synthesis: Exam Rule of Thumb

| Content type | Render as |
|---|---|
| Financial / numeric comparisons (revenue, margins, growth) | **Tables** |
| News, developments, context | **Prose / narrative** |
| Lists of discrete items (technology areas) | **Structured lists** |

| Option | Verdict |
|---|---|
| **Synthesis agent renders each content type appropriately** | ✅ Fixes the actual problem: presentation at the output stage |
| Standardize all subagents to prose | ❌ Destroys the structure of financial data |
| Standardize all subagents to claim/evidence/source/confidence JSON | ❌ Forces narrative and tabular data into one ill-fitting shape |
| Format conversion layer to a common intermediate representation | ❌ Extra complexity, and a "common format" flattens the differences that matter |

**Tips:** The loss happens at **synthesis**, where everything is flattened into bullet points. Fix it where it breaks: **let output format follow content type**. Don't force every source into one format ("one-size-fits-all" is the trap in all three wrong options).

**Source check (official Exam Guide, Task Statement 5.6 skills):** "Rendering different content types appropriately in synthesis outputs: financial data as tables, news as prose, technical findings as structured lists, rather than converting everything to a uniform format."

### Example question (answered correctly)

**Q:** A research system adds specialized agents: a financial API agent (structured JSON: revenue, margins, growth), a news monitoring agent (prose summaries), a patent analysis agent (structured lists of technology areas). The synthesis agent turns everything into bullet points, so financial comparisons lose tabular clarity and news loses narrative flow. What change would most improve briefing quality?

- ❌ Standardize all subagent outputs to prose with inline citations: financial comparisons become hard-to-read text.
- ❌ Format conversion layer to a common intermediate representation: more infrastructure, and flattening into one format is the problem.
- ❌ Standardize all outputs to claim/evidence/source/confidence JSON: doesn't fit tables or narratives, and still doesn't define rendering.
- ✅ **Synthesis agent renders each content type appropriately: financial data as tables, news as prose**: fixes it where the quality is lost.

---

## Information Flow Between Subagents (Hub-and-Spoke): Exam Rule of Thumb

| Pattern | Verdict |
|---|---|
| **Subagent A returns output → coordinator → coordinator includes relevant findings in the prompt when invoking subagent B** | ✅ Standard hub-and-spoke flow |
| Subagent A directly invokes subagent B | ❌ Subagents don't call each other; all routing goes through the coordinator |
| Event-driven message queue / pub-sub between agents | ❌ Not the Claude Agent SDK pattern; bypasses coordinator control |
| Shared memory store both agents read/write | ❌ Implicit shared state; subagents have isolated context and get context explicitly |

**Tips:** Subagents have **isolated context**. They don't inherit the coordinator's history and don't share memory. **The only way context reaches a subagent is through its prompt**, and the coordinator decides what goes in. Routing everything through the coordinator gives observability, consistent error handling, and controlled information flow.

**Source check (official Exam Guide, Task Statements 1.2 & 1.3):**
- 1.2: "Hub-and-spoke architecture where a coordinator agent manages all inter-subagent communication, error handling, and information routing"; "Routing all subagent communication through the coordinator for observability, consistent error handling, and controlled information flow."
- 1.3: "subagent context must be explicitly provided in the prompt—subagents do not automatically inherit parent context or share memory between invocations"; "Including complete findings from prior agents directly in the subagent's prompt."

### Example question (answered correctly)

**Q:** The web search agent gathered relevant sources; the document analysis agent now needs to examine them. How does information typically flow between these two subagents?

- ❌ Web search agent directly invokes the document analysis agent: breaks hub-and-spoke, since subagents don't spawn each other.
- ✅ **Coordinator receives the web search output and includes relevant findings in the prompt when invoking the document analysis agent**
- ❌ Event-driven message queue with subscriptions: not how the SDK coordinates, and it bypasses the coordinator.
- ❌ Shared memory store: subagents don't share memory, so context must be passed explicitly.

### Example question 2 on subagent context (answered correctly): "no findings provided"

**Q:** After the web search and document analysis agents finish, the coordinator invokes the synthesis agent, which says it can't complete the task because no research findings were provided. Most likely cause?

- ❌ Subagents need a shared API connection for automatic context sharing: no such mechanism exists; context isn't shared automatically.
- ❌ Synthesis agent's context window too small: that would cause truncation or overflow errors, not "nothing was provided".
- ✅ **The coordinator didn't include the previous agents' outputs in the synthesis agent's prompt**
- ❌ Synthesis agent needs tools to fetch other agents' conversation histories: subagents don't read each other's histories; the coordinator passes findings explicitly.

**Lesson:** A subagent saying "no data" = **the coordinator forgot to pass it in the prompt**. Isolated context means nothing arrives unless it's explicitly included (Exam Guide 1.3: "Including complete findings from prior agents directly in the subagent's prompt").

---

## Preserving Source Attribution Through Synthesis: Exam Rule of Thumb

| Option | Verdict |
|---|---|
| **Subagents output structured claim-source mappings; synthesis agent must preserve and merge them** | ✅ Attribution travels *as data* through every step |
| Coordinator injects source-ID prefixes into text, then parses them back out at report time | ❌ Fragile text tagging: the synthesis agent rewrites, merges, and reorders text, so prefixes get dropped or attached to the wrong claims |
| Semantic similarity matching afterwards to reconstruct sources | ❌ Guesswork after the link was already lost; paraphrased or merged claims match poorly |
| Keep full transcripts + a citation-resolution agent to analyze logs | ❌ Heavy, costly, and still reconstructing instead of preserving |

**Tips:** Attribution breaks when claims are **compressed or merged without the claim→source link**. Fix it by making the link **structured data** (e.g., `{claim, sources:[{url, doc, excerpt}]}`) that each step must carry forward, not by tagging free text or reconstructing afterwards. **Preserve > reconstruct.**

**Source check (official Exam Guide, Task Statement 5.6):** "How source attribution is lost during summarization steps when findings are compressed without preserving claim-source mappings"; "structured claim-source mappings that the synthesis agent must preserve and merge"; "Requiring subagents to output structured claim-source mappings (source URLs, document names, relevant excerpts) that downstream agents preserve through synthesis." Also 1.3: "Using structured data formats to separate content from metadata (source URLs, document names, page numbers)… to preserve attribution."

### Example question (missed)

**Q:** Final reports often contain claims without proper source attribution. The web search and document analysis agents attach citations correctly, but the synthesis agent loses track of which sources support which conclusions when combining findings. Most effective architectural change?

- ✅ **Require all subagents to output structured claim-source mappings that the synthesis agent must preserve and merge**
- ❌ Coordinator injects source-identifier prefixes into text before each handoff, then parses them at report generation (my answer): mixes metadata into free text. The synthesis agent paraphrases and combines sentences, so prefixes get lost, duplicated, or misattached, and regex-style parsing can't recover them reliably. Content and metadata should be *separate structured fields*.
- ❌ Semantic similarity matching against original sources afterwards: approximate reconstruction of a link that should never have been lost.
- ❌ Full transcripts + citation-resolution agent: expensive extra agent and log analysis, still reconstructing.

---

## When NOT to Spawn a Subagent: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Simple request, and the coordinator **already has the needed info in its context** | **Coordinator answers directly**, with no subagent |
| Complex analysis, or verbose exploration that would clutter the coordinator | Spawn a subagent |
| Re-sending 80K+ tokens to a new subagent on every follow-up | Trap: slow and costly duplication |
| Prompt caching on the subagent | Trap: cuts some cost/latency, but still spawns an unnecessary agent |
| Pre-generate summaries at many granularities | Trap: wasted work for queries that may never come, and still can't predict the exact question |
| Reduced-context subagent that asks the coordinator on demand | Trap: adds round-trips and complexity; slower, not faster |

**Tips:** Subagents have a cost: startup, context transfer, extra latency. Use them for **isolation or complexity**, not by habit. The coordinator should **analyze the query and decide** whether delegation is needed.

**Source check (official Exam Guide, Task Statement 1.2):** "The role of the coordinator in… deciding which subagents to invoke based on query complexity"; "Designing coordinator agents that analyze query requirements and dynamically select which subagents to invoke rather than always routing through the full pipeline."

### Example question (answered correctly)

**Q:** Follow-up queries like "summarize what we learned about market trends" take 40+ seconds. The coordinator spawns the synthesis subagent for each summary, passing 80K+ tokens of findings, which the coordinator already has in its context. Most effective way to improve response time?

- ❌ Pre-generate and cache summaries at multiple granularities: speculative work, and can't match every follow-up question.
- ❌ Reduced-context subagent that requests findings on demand: more back-and-forth, more latency.
- ✅ **Coordinator handles straightforward summaries directly with its existing context; spawn subagents only for complex analysis**
- ❌ Prompt caching on the synthesis subagent: an optimization of an unnecessary step. The spawn overhead remains.

### Example question 2 on attribution (answered correctly): final report handoff

**Q:** Web search found 25 sources (120K tokens raw), document analysis extracted insights (15K), synthesis produced a narrative draft (3K). The coordinator must pass context to the report generation agent for final output with proper citations. Best balance of completeness and efficiency?

- ✅ **Pass the synthesis draft + a structured source index mapping key claims to source URLs and relevant excerpts**: small (draft + index), and complete enough for accurate citations.
- ❌ Full accumulated context from all agents (~138K): complete but very inefficient, with lost-in-the-middle risk.
- ❌ Condensed summary with sources attributed *by name only*: too little to cite properly (no URLs, no excerpts, no claim-level mapping).
- ❌ Only the draft, with a post-processing pipeline matching claims to sources afterwards: reconstruction after the link is lost (same trap as semantic matching).

**Lesson:** Handoffs should carry **the distilled content + a structured claim→source index** (URL, excerpt). Not everything (bloat), not names only (too thin), not reconstructed later.

---

## Parallelizing Work While Keeping Observability: Exam Rule of Thumb

| Option | Verdict |
|---|---|
| **Coordinator spawns parallel subagents (each gets a subset of independent items), then aggregates before synthesis** | ✅ Faster, and all delegation stays visible to the coordinator |
| A subagent spawns its own subagents | ❌ Nested delegation hides work from the coordinator; breaks hub-and-spoke monitoring and debugging |
| Recursive agent hierarchy down to single items | ❌ Deep trees, overhead, hard to trace |
| Message queue + worker pool | ❌ Outside the coordinator pattern; async infrastructure reduces visibility |

**Tips:** Only the **coordinator delegates**; subagents don't spawn subagents. Parallelize by emitting **multiple Task calls in one coordinator response**. Parallel is right **when subtasks are independent** (each precedent can be analyzed on its own).

**Contrast with Q1 (500-error trace):** parallel workers were wrong there because each step *depended on* the previous finding. Here the 12 precedents are independent, so parallel is right.

**Source check (official Exam Guide):** 1.3: "Spawning parallel subagents by emitting multiple Task tool calls in a single coordinator response rather than across separate turns." 1.2: "Routing all subagent communication through the coordinator for observability, consistent error handling, and controlled information flow."

### Example question (missed)

**Q:** The document analysis subagent processes a legal case's cited precedents sequentially; a landmark case with 12 precedents takes 3+ minutes. Most effective way to reduce latency while preserving the coordinator's ability to monitor and debug?

- ❌ Recursive agent hierarchy subdividing to single-precedent granularity: deep nesting, overhead, poor traceability.
- ❌ Let the document analysis subagent spawn its own specialized subagents dynamically (my answer): speeds things up but hides the sub-delegation from the coordinator, which loses monitoring and debugging (breaks hub-and-spoke). Subagents typically aren't given the Task tool.
- ❌ Message queue with an async worker pool: extra infrastructure outside the coordinator, less visibility.
- ✅ **Coordinator spawns parallel document analysis subagents, each with a subset of precedents, then aggregates before synthesis**

---

## Dynamic Query Routing by Complexity: Exam Rule of Thumb ⚠️ (answer key disputed)

| Option | Verdict per official Exam Guide |
|---|---|
| **Coordinator analyzes each query dynamically and selectively routes subagents by complexity and characteristics** | ✅ Matches Task Statement 1.2 almost word for word; adapts as the query mix changes |
| Fast-track factual questions around subagents, analytical ones through the full pipeline | ⚠️ Practice site marks this correct, but it's a fixed two-way split. It still sends every "analytical" query through the *full* pipeline, which is what 1.2 warns against, and it can't adapt to new query types |
| Pattern-based routing to predefined subagent combinations | ❌ Rigid; breaks as usage evolves |
| Train a complexity classifier on labeled historical data | ❌ Over-engineered; history doesn't cover *evolving* new applications |

**Tips:** Key words **"uneven and evolving"** → avoid anything fixed (patterns, predefined combos, trained-on-history classifiers, binary fast-tracks). Prefer **model-driven, per-query decisions by the coordinator**.

**Source check (official Exam Guide, Task Statement 1.2):** "The role of the coordinator in… deciding which subagents to invoke based on query complexity"; "Designing coordinator agents that analyze query requirements and dynamically select which subagents to invoke rather than always routing through the full pipeline."

### Example question (my answer marked wrong, but supported by the Exam Guide)

**Q:** In production, complex queries go to specialized subagents. The query distribution is uneven and evolving as users discover new applications. Most effective approach to optimize for query complexity?

- ❌ Trained query complexity classifier on labeled history: heavy, and stale as usage evolves.
- ✅ (per Exam Guide) **Coordinator analyzes each query dynamically and selectively routes subagents based on complexity and routing characteristics** (my answer)
- ⚠️ Fast-track factual queries bypassing subagents; analytical ones through the complete pipeline (practice site's key): a static binary rule, and still "always the full pipeline" for analytical queries.
- ❌ Pattern-based factual/analytical routing to predefined combinations: rigid.

**Note:** On the real exam, choose the option that mirrors the Exam Guide wording: *dynamic coordinator analysis and selective subagent invocation*.

**Site's rationale reviewed:** it assumes the distribution is "mostly factual/simple queries", but that condition isn't in the question stem (the stem says "uneven and evolving"). The explanations for the wrong options are templated and circular. If a variant of this question explicitly says *most queries are simple factual lookups*, fast-track becomes defensible. As written, dynamic coordinator routing fits the stem and the Exam Guide better.

### Example question 3 on attribution (answered correctly): metadata lost in summarization

**Q:** The synthesis agent receives summarized findings from web search and document analysis, then passes a consolidated summary to the report generator. Reports make factual claims without proper citations because source metadata was lost during summarization. Most effective approach?

- ❌ Report generator queries the web search agent to re-locate sources: subagent-to-subagent calls (breaks hub-and-spoke), plus reconstructing after the fact. Document analysis sources aren't covered either.
- ✅ **Each agent outputs structured data separating content summaries from source metadata (URLs, document names, page numbers)**: metadata survives every summarization step as its own fields.
- ❌ Synthesis agent embeds inline citations in its summary text: metadata mixed into free text, easily dropped or garbled in later rewriting (same trap as prefix tagging).
- ❌ Skip summarization, pass full raw outputs: keeps sources but bloats the context and loses the point of the pipeline.

**Lesson (attribution, 3rd time):** Keep **content and metadata in separate structured fields** (Exam Guide 1.3 wording almost verbatim). Not inline text, not re-lookup, not raw dumps.

---

## MCP Tool Error Handling (isError): Exam Rule of Thumb

| Option | Verdict |
|---|---|
| **Return the error message in the tool result content with `isError: true`** (ideally with structured metadata: `errorCategory`, `isRetryable`, human-readable description) | ✅ The agent *sees* the failure, knows it's a failure, and can decide what to do |
| Log server-side, return an empty result | ❌ Hides the failure. An empty result looks like "no matching order", so the agent gives wrong answers and never retries transient failures |
| Throw an exception from the tool handler | ❌ Error goes to the framework or logs, not to the model's reasoning; the agent can't recover intelligently |
| Success response with a `status` field | ❌ Signals success while carrying an error; the agent may treat it as valid data |

**Error types the agent must tell apart:**

| Type | Example | Retry? |
|---|---|---|
| Transient | timeout, DB temporarily down | Yes |
| Validation | invalid order ID format | No: fix input |
| Business | policy violation | No: explain to user |
| Permission | not authorized | No: escalate |
| **Valid empty result** | query succeeded, no matches | Not an error |

**Tips:** Errors are **information for the agent**. Don't hide them (empty result), don't send them only to logs (exception), and don't disguise them as success (status field). Generic "Operation failed" is also bad: the agent needs the category to choose retry vs. explain vs. escalate. **Access failure ≠ empty result.**

**Source check (official Exam Guide, Task Statement 2.2):** "The MCP isError flag pattern for communicating tool failures back to the agent"; "Returning structured error metadata including errorCategory (transient/validation/permission), isRetryable boolean, and human-readable descriptions"; "Distinguishing between access failures (needing retry decisions) and valid empty results (representing successful queries with no matches)."

### Example question (missed)

**Q:** The `lookup_order` MCP tool's backend sometimes returns errors ("Order not found", temporary database failures). Correct pattern for communicating these errors to the agent?

- ❌ Log server-side, return an empty result "to avoid confusing the model" (my answer): *creates* confusion. The agent can't tell "DB is down, retry" from "order truly doesn't exist", so it may tell a customer their order doesn't exist during an outage.
- ❌ Throw an exception for the framework to catch and log: the model never gets a usable error to reason about.
- ✅ **Return the error message in the tool result content with `isError: true`**
- ❌ Success response with a "status" field for the error type: mixed signals; the call looks successful.

---

## Escalating to a Human: Handoff Context: Exam Rule of Thumb

**Scenario to picture:** You're a support agent. After a long investigation you found the problem, but you're *not allowed* to fix it (the refund is over your limit). You hand the case to a human colleague who **did not see the conversation**. What note do you give them?

| Handoff option | Verdict |
|---|---|
| **Structured summary: customer ID + root cause + refund amount + recommended action** | ✅ Human can act immediately: who, why, how much, what to do |
| Complete transcript with all tool results | ❌ 25+ turns of noise; human must redo the analysis you already did |
| Original complaint verbatim + tool excerpts | ❌ Raw evidence without your conclusion or a recommended action |
| Diagnosis + refund amount only | ❌ Missing customer ID (can't find the account) and next step |

**Tips:** A handoff should be **self-contained and actionable**: *identify* (customer ID), *explain* (root cause), *quantify* (amount), *recommend* (action). Not everything, not raw evidence, not too little. Same principle as agent-to-agent handoffs: pass distilled, structured context.

**Source check (official Exam Guide, Task Statement 1.4):** "Structured handoff protocols for mid-process escalation that include customer details, root cause analysis, and recommended actions"; "Compiling structured handoff summaries (customer ID, root cause, refund amount, recommended action) when escalating to human agents who lack access to the conversation transcript."

### Example question

**Q:** After 25+ turns investigating a billing dispute, the agent found duplicate charges caused by a payment gateway timeout triggering retry logic. The $847 refund exceeds the $500 authorization limit, so it must call `escalate_to_human`. The human won't have the transcript. What context should it pass?

- ✅ **Structured summary: customer ID, root cause, refund amount, recommended action** (matches Exam Guide 1.4 verbatim)
- ❌ Complete transcript with all tool results: overwhelming, and makes the human re-investigate.
- ❌ Original complaint + tool excerpts showing duplicates: evidence without conclusions or a next step.
- ❌ Diagnosis + refund amount only: missing customer identity and recommended action.

---

## Trimming Verbose Tool Outputs: Exam Rule of Thumb

| Option | Verdict |
|---|---|
| **Keep only task-relevant fields** (items, purchase date, return window, status) from each tool result; drop verbose details | ✅ Cuts tokens, keeps exact structured values |
| Keep going without changing anything | ❌ Context fills with irrelevant data → degradation, lost-in-the-middle, higher cost |
| Move tool responses to a vector DB + semantic retrieval | ❌ Over-engineered for a conversation; approximate retrieval (and vector DB details are out of scope) |
| Replace structured responses with prose summaries | ❌ Loses precision (dates, amounts, statuses become fuzzy text); summaries still carry noise |

**Tips:** Tool results **accumulate** and use tokens out of proportion to their relevance (40+ fields when ~5 matter). Trim to the **fields the current task needs**, keep them **structured**, and do it **before** they pile up. Don't convert exact data to prose.

**Source check (official Exam Guide, Task Statement 5.1):** "Progressive summarization risks: condensing numerical values, percentages, dates… into vague summaries"; "How tool results accumulate in context and consume tokens disproportionately to their relevance (e.g., 40+ fields per order lookup when only 5 are relevant)"; "Trimming verbose tool outputs to only relevant fields before they accumulate in context (e.g., keeping only return-relevant fields from order lookups)."

### Example question (answered correctly)

**Q:** The agent called `lookup_order` several times investigating return requests. Each response has 40+ fields (items, shipping, payment, status history), and tool outputs now make up most of the context. The customer wants to discuss two more orders. Most effective approach before more lookups?

- ✅ **Extract only return-relevant fields (items, purchase date, return window, status) from each existing response, removing verbose details**
- ❌ Proceed without modifying the existing tool output context: context keeps bloating with each lookup.
- ❌ Move tool responses to a vector DB with semantic indexing: over-engineering and fuzzy retrieval for data you can simply trim.
- ❌ Replace structured responses with natural-language summaries: loses exact values needed for return decisions.

---

## Escalation Triggers: Explicit Human Request: Exam Rule of Thumb ⚠️ (answer key disputed)

| Customer situation | Correct behavior (Exam Guide 5.2) |
|---|---|
| **Explicitly demands a human** ("I want a real person NOW") | **Escalate immediately, without first investigating** |
| Frustrated, but issue is straightforward and within agent capability, **no explicit demand** | Acknowledge frustration, offer to resolve; escalate only if they **reiterate** |
| Policy ambiguous or silent on the request | Escalate |
| No meaningful progress possible | Escalate |
| Negative sentiment alone | ❌ Not a reliable trigger |

**Tips:** An **explicit request for a human is itself an escalation trigger**. Don't stall with questions, and don't run tool calls "first". Watch the wording: "*explicitly demands*" → escalate now; "*frustrated but solvable*" → offer to help, escalate if repeated.

**Source check (official Exam Guide, Task Statement 5.2):** "Appropriate escalation triggers: customer requests for a human…"; "The distinction between escalating immediately when a customer explicitly demands it versus offering to resolve when the issue is straightforward"; "**Honoring explicit customer requests for human agents immediately without first attempting investigation**"; "Acknowledging frustration while offering resolution when the issue is within the agent's capability, escalating only if the customer reiterates their preference."

### Example question (practice-site key disagrees with Exam Guide)

**Q:** A customer writes: "This is frustrating. I've explained my issue twice and nothing is being resolved. I want to talk to a real person NOW." The agent hasn't called any tools yet. What should the agent do?

- ✅ (per Exam Guide) **Immediately call `escalate_to_human`**: explicit demand plus already explained twice. Ideally include a short structured summary of what the customer has said.
- ❌ Explain what the agent can do and offer to resolve, escalating only if repeated: that's the rule for frustration *without* an explicit demand. Here the customer already demanded a human (and has effectively repeated themselves).
- ⚠️ Acknowledge frustration + ask one targeted question before escalating (practice site's key): delays an explicit request and makes the customer explain a *third* time. Contradicts "honor… immediately".
- ❌ Call `get_customer` and `lookup_order` first, then escalate (my answer): the Exam Guide explicitly says "without first attempting investigation".

**Note:** On the real exam, choose **immediate escalation** when the customer explicitly asks for a human.

**Site's rationale reviewed:** It argues "the customer said 'twice' but you have no context, so ask one empathetic question to try to resolve it first." This conflicts with the Exam Guide: (1) an explicit human request is itself the trigger, to be honored "immediately without first attempting investigation", and a clarifying question *is* an attempt to investigate; (2) the missing context is solved by the **structured handoff summary**, not by making the customer explain a third time; (3) the explanations for the other options are templated and circular ("doesn't address the requirement"), with no real argument against immediate escalation.

### Example question 2 on escalation (answered correctly): choosing escalation triggers

**Q:** Implementing when the agent should call `escalate_to_human`. Which approach most reliably identifies cases that genuinely need a human?

- ✅ **Escalate when the customer requests a human, when the issue requires policy exceptions, or when the agent cannot make meaningful progress** (the Exam Guide 5.2 trigger list, verbatim)
- ❌ Sentiment analysis with a frustration-score threshold: sentiment is an unreliable proxy for case complexity (Exam Guide 5.2). Angry customers with simple issues get escalated; calm customers with policy gaps don't.
- ❌ Rules engine mapping issue types, segments, and categories to escalation: rigid, can't handle novel situations or policy gaps, and throws away useful model judgment.
- ❌ Escalate after three consecutive failed tool calls: an arbitrary count. It escalates solvable cases after transient errors and misses explicit human requests or policy exceptions.

**Lesson:** The three reliable triggers are **explicit human request, policy exception or gap, no meaningful progress**. Sentiment scores, self-reported confidence, rigid rules, and fixed failure counts are traps.

### Example question 2 on handoffs (answered correctly): mid-process escalation

**Q:** Handling a billing dispute, the agent called `get_customer` and `lookup_order` and found a promotional pricing error that needs manager approval, beyond its authorization. How should the workflow handle this mid-process escalation?

- ❌ Call `escalate_to_human` with only the customer's original message: throws away the investigation, so the human starts over.
- ✅ **Compile a structured handoff (customer details, order info, identified issue) before calling `escalate_to_human`** (Exam Guide 1.4: "Structured handoff protocols for mid-process escalation that include customer details, root cause analysis, and recommended actions")
- ❌ Attempt `process_refund` anyway, escalate only if rejected: acting beyond authorization and relying on the system to stop it. Violates policy (the Exam Guide recommends hooks that *block* such actions and redirect to escalation).
- ❌ Persist full conversation and tool history to a DB, escalate with a reference ID: the human must dig through raw logs; not a distilled, actionable handoff.

**Lesson:** Mid-process escalation = **stop before unauthorized actions** + **structured, actionable handoff** of what you already found.

### Example question 2 on tool errors (answered correctly): partial success + transient failure

**Q:** In a billing dispute, `get_customer` and `lookup_order` succeed, but `process_refund` returns a timeout. The agent can explain the billing and confirm refund eligibility, but can't process the refund. Which approach best balances first-contact resolution with appropriate error handling?

- ❌ Confirm the refund will be processed and close the conversation: false promise; nothing was processed, so the customer is misled.
- ❌ Escalate immediately because the refund can't be completed: gives up the parts the agent *can* resolve; a transient timeout isn't a policy or capability gap.
- ✅ **Explain the billing, confirm eligibility, acknowledge the system issue, and offer escalation or a retry later**: delivers everything possible now, transparent about the failure, gives the customer a clear next step.
- ❌ Automatic exponential-backoff retries, keeping the conversation open until success: the customer waits indefinitely with no communication; the outage may last a long time.

**Lesson:** On a partial failure, **deliver what succeeded, be honest about what failed (transient error), and offer a concrete path forward**. Never claim success, don't abandon progress, don't stall silently.

### Example question 3 on tool errors (answered correctly): structured retryable metadata

**Q:** Logs show inconsistent handling when `lookup_order` fails: the agent sometimes retries, wasting 3–4 turns, before concluding. The MCP tool returns only a plain-text error message. Most effective improvement?

- ❌ New `analyze_error` MCP tool the agent calls to classify errors: an extra tool call per error, and pushes classification away from the source that already knows the error type.
- ❌ Retry with exponential backoff in the MCP server for all errors, returning only successes: business errors (e.g., order not found) never succeed, so it just delays and hides the failure from the agent.
- ❌ Few-shot examples teaching the agent to parse error text: brittle guessing from free text; the server already knows the answer.
- ✅ **Return structured error responses with `retryable: false` for business errors and a customer-friendly explanation for Claude to use**: the agent knows immediately not to retry and what to tell the customer.

**Lesson:** The tool **knows** the error type, so **say it in structured fields** (`errorCategory`, `isRetryable`, message) instead of making the agent guess from text (Exam Guide 2.2: "Including retriable: false flags and customer-friendly explanations for business rule violations"; "returning structured metadata prevents wasted retry attempts").

---

## How the Agentic Loop Decides the Next Action: Exam Rule of Thumb

**The loop:** model requests a tool (`stop_reason: "tool_use"`) → your code runs the tool → **tool result is appended to the conversation** → model is called again and **reasons over the full context** to choose the next tool or give a final answer (`stop_reason: "end_turn"`).

| Option | Verdict |
|---|---|
| **Tool result is added to the conversation; the model reasons about the next action** | ✅ Model-driven decision-making, the core of an agentic loop |
| Orchestration layer auto-routes based on a status field | ❌ Hard-coded routing, not the model reasoning; ignores context (return policy, customer request) |
| Pre-configured decision tree mapping attributes → tool calls | ❌ That's a scripted workflow, not an agent |
| Execute the remaining steps of a sequence planned at the start | ❌ Can't adapt to new information (e.g., "45 days" may exceed the return window) |

**Tips:** In an agent, **each tool result changes what happens next**, and the *model* makes that call with the whole conversation as context. (When a rule must be *guaranteed*, such as a refund limit, enforce it with hooks or gates *around* the model, but the loop itself is still model-driven.)

**Source check (official Exam Guide, Task Statement 1.1):** "How tool results are appended to conversation history so the model can reason about the next action"; "The distinction between model-driven decision-making (Claude reasons about which tool to call next based on context) and pre-configured decision trees or tool sequences."

### Example question (missed)

**Q:** The agent calls `lookup_order` and gets details showing the item was purchased 45 days ago. How does the agentic loop decide whether to call `process_refund` or `escalate_to_human` next?

- ❌ The orchestration layer automatically routes to the next tool based on the order's status field (my answer): that's external hard-coded routing. In the agentic loop, the orchestrator just runs tools and returns results; *the model* decides. A status field alone also can't weigh the 45-day age against the return policy.
- ❌ Pre-configured decision tree mapping order attributes to tool calls: scripted workflow, not model-driven.
- ❌ Execute the remaining steps of a tool sequence planned at the start: ignores the new information from the lookup.
- ✅ **The order details are added to the conversation and the model reasons about which action to take**

### Example question 3 on escalation (missed): frustrated but resolvable, no explicit human demand

**Q:** A customer writes: "I've been going back and forth on this return for days. I just want to speak to someone who can actually help me." `lookup_order` confirms the return is straightforward: within policy, eligible for immediate processing. What should the agent do?

- ❌ Process the refund and then inform them: acts without consent; ignores what the customer said about wanting someone to help.
- ❌ Call `escalate_to_human` immediately (my answer): "someone who can actually help me" expresses frustration, **not an explicit demand for a human**, and the agent *can* resolve it now. Escalating adds more delay to a customer already waiting days.
- ✅ **Acknowledge frustration, tell them it's resolvable now, and offer to complete it or escalate**: fastest resolution while respecting their choice.
- ❌ Ask what hasn't worked before deciding: unnecessary; the agent already verified it's resolvable.

**Contrast with Q49 (the key distinction, Exam Guide 5.2):**

| | Q49 | This question |
|---|---|---|
| Wording | "I want to talk to a **real person** NOW" | "speak to **someone who can actually help** me" |
| Explicit human demand? | **Yes** | No (frustration) |
| Agent has verified it can resolve? | No | **Yes: within policy, eligible now** |
| Correct action | **Escalate immediately** | **Acknowledge + offer to resolve now or escalate** (escalate if they reiterate) |

**Source:** Exam Guide 5.2: "The distinction between escalating immediately when a customer explicitly demands it versus offering to resolve when the issue is straightforward"; "Acknowledging frustration while offering resolution when the issue is within the agent's capability, escalating only if the customer reiterates their preference."

---

## Guaranteed Compliance: Hooks vs. Prompts: Exam Rule of Thumb

| Option | Guarantee? |
|---|---|
| **Hook intercepts the tool call; if refund > $500 → block and invoke human escalation** | ✅ Deterministic: enforced in code *before* the action runs |
| Few-shot examples at $400/$500/$600 | ❌ Still probabilistic; lowers the failure rate but not to zero |
| Emphatic prompt ("CRITICAL… MUST… NEVER") | ❌ Still just instructions; a non-zero failure rate remains |
| Tool returns an error "please escalate" | ❌ Partly deterministic (blocks the refund) but relies on the model to *choose* to escalate; the rule says escalation must be **automatic** |

**Tips:** "Must", "guaranteed", "compliance", "cannot be left to model discretion" → **enforce in code (hooks, gates, prerequisites)**, not prompts. Prompts and examples guide behavior; hooks **guarantee** it. The hook should both **block** the violating action and **redirect** to the required workflow.

**Source check (official Exam Guide):** 1.4: "When deterministic compliance is required (e.g., identity verification before financial operations), prompt instructions alone have a non-zero failure rate." 1.5: "Implementing tool call interception hooks that block policy-violating actions (e.g., refunds exceeding $500) and redirect to alternative workflows (e.g., human escalation)."

### Example question (answered correctly)

**Q:** Compliance requires refunds over $500 to automatically escalate to a human; this can't be left to model discretion. Despite clear system prompt instructions, the agent processes high-value refunds directly 3% of the time. How to achieve guaranteed compliance?

- ✅ **Hook intercepts tool calls; when the refund amount exceeds $500, block it and invoke human escalation**
- ❌ Few-shot examples at various amounts: better guidance, no guarantee.
- ❌ Emphatic system prompt language: still prompt-based, a non-zero failure rate.
- ❌ Refund tool returns an "exceeds limit, please escalate" error: blocks the refund, but escalation still depends on the model's discretion, not automatic.

---

## Multi-Issue Sessions Near Context Limits: Exam Rule of Thumb ⚠️ (answer key disputed)

| Option | Verdict per official Exam Guide (5.1) |
|---|---|
| **Extract and persist structured issue data (order IDs, amounts, statuses) into a separate context layer** | ✅ Exam Guide 5.1 **verbatim**; exact facts for every issue stay available regardless of history trimming |
| Summarize earlier turns into narrative, keep full history only for the active issue | ⚠️ Practice site's key. The guide lists this as a **risk**: "Progressive summarization risks: condensing numerical values, percentages, dates… into vague summaries." Narrative loses refund amounts, IDs, statuses |
| Sliding window of the most recent 30 turns | ❌ Drops the refund issue (turns 1–15) entirely, which is exactly what the customer is asking about |
| Re-fetch via MCP tools on demand | ❌ Tools return current system data, not what was discussed or promised in the conversation; extra calls |

**Tips:** For long, multi-issue sessions, keep a **persistent "case facts" block**: structured transactional facts (order numbers, amounts, dates, statuses) per issue, **included in every prompt, outside the summarized history**. Summaries may compress conversation *flow*, but **never** the facts.

**Source check (official Exam Guide, Task Statement 5.1):**
- Knowledge: "Progressive summarization risks: condensing numerical values, percentages, dates, and customer-stated expectations into vague summaries."
- Skills: "Extracting transactional facts (amounts, dates, order numbers, statuses) into a persistent 'case facts' block included in each prompt, outside summarized history"; "**Extracting and persisting structured issue data (order IDs, amounts, statuses) into a separate context layer for multi-issue sessions.**"

### Example question (my answer marked wrong, but it matches the Exam Guide verbatim)

**Q:** A customer raises three issues in one session: refund inquiry (turns 1–15), subscription question (16–30), payment method update (31–45). At turn 48 they ask, "What happened with my refund?" The conversation is approaching context limits. Which strategy best maintains the agent's ability to address all issues?

- ⚠️ Narrative summary of earlier turns, full history only for the active issue (practice site's key): the refund details (amount, order ID, status) risk becoming vague, which is the risk the guide warns about.
- ❌ Sliding window of the last 30 turns: loses turns 1–15, the refund.
- ❌ Re-fetch via MCP tools on demand: doesn't restore conversation-specific context; extra calls.
- ✅ (per Exam Guide) **Extract and persist structured issue data (order IDs, amounts, statuses) into a separate context layer** (my answer)

**Note:** On the real exam, choose the **structured case-facts / separate context layer** option. It is the Exam Guide's own wording.

**Site's rationale reviewed:** It calls progressive narrative summarization "the classic pattern" for long multi-issue chats. Problems: (1) the Exam Guide names *progressive summarization* as a **risk** (vague numbers, dates, statuses) and names the structured separate context layer as the **skill**; (2) at turn 48 the customer returns to the **refund**, so the "active issue" would be the one already compressed into narrative, with its amount, order ID, and status possibly lost; (3) "resolved topics" is assumed, not stated; (4) the rejection of option C is templated and gives no real reason.

---

## Stateless API: Passing Conversation History: Exam Rule of Thumb

| Symptom | Most likely cause |
|---|---|
| Agent "forgets" earlier turns and asks for info already given, as if the exchange never happened | **Conversation history isn't being passed in subsequent API requests** |
| Remembers recent turns but loses old details in a very long chat | Context degradation / lost in the middle / trimming |
| Subagent says "no findings provided" | Coordinator didn't include prior outputs in its prompt |

**Tips:** The Messages API is **stateless**. Claude has no memory between requests. **Your application must send the full `messages` array (all prior user and assistant turns plus tool results) with every call.** No "remember" instruction and no memory setting exists; there is no "two-turn default limit". Forgetting *everything* earlier points to missing history, not a model limit.

**Source check (official Exam Guide, Task Statement 5.1):** "The importance of passing complete conversation history in subsequent API requests to maintain conversational coherence." Also 1.1: "How tool results are appended to conversation history so the model can reason about the next action."

### Example question (missed)

**Q:** The agent verifies identity through a multi-step process before resetting passwords. In testing, after the customer answers the third verification question, the agent asks for their name again, as if the earlier exchange never happened. Most likely cause?

- ❌ The verification tool clears the agent's internal state: tools don't hold or clear the model's memory; there's no internal state to clear.
- ❌ The prompt lacks instructions to remember across exchanges: memory isn't an instruction; the model only sees what's in the request.
- ❌ Claude's memory is limited to two turns by default, needing configuration (my answer): **no such limit or setting exists**. Context is whatever messages you send, up to the context window.
- ✅ **The conversation history isn't being passed in subsequent API requests**

### Example question 4 on tool errors (answered correctly): transient vs. business errors + customer-facing quality

**Q:** `process_refund` returns technical errors ("503 Service Unavailable", "Connection timeout"; transient, 5%) and business errors ("Order exceeds 30-day return window", "Item already refunded"; permanent, 12%). The agent wastes 3–4 turns retrying business errors. Both return plain text. Most effective way to reduce wasted retries *while improving customer-facing response quality*?

- ❌ Add a mandatory `check_refund_eligibility` tool before `process_refund`: an extra call on every refund, only prevents *some* business errors, doesn't fix how errors are communicated.
- ❌ Tool-level auto-retry for technical errors only, business errors passed through without retries: strongest distractor. It helps transient errors, but business errors still reach Claude as **plain text**, so Claude may still retry, and there's **no customer-friendly explanation**. Misses half the question.
- ❌ Few-shot examples for parsing error text: guessing from free text.
- ✅ **Structured error responses with `retryable: false` for business errors and a customer-friendly explanation for Claude to use**: stops wasted retries *and* gives Claude a good message for the customer.

**Lesson:** Read **both** goals in the question. "Reduce retries" + "customer-facing quality" → **structured metadata (retryable flag) + human-readable explanation** (Exam Guide 2.2).

---

## Returning Customer / Stale Tool Results: Exam Rule of Thumb ⚠️ (flawed question: two near-identical options)

| Option | Verdict |
|---|---|
| **Start a new session, inject a structured summary (issue type, resolution steps, current status), then make fresh tool calls** | ✅ Summary keeps what matters; fresh calls replace stale data ("Pending Refund" may have changed in 4 hours) |
| "Create a new session **with** a structured summary…, then make fresh tool calls" | ⚠️ Practically the same idea; marked wrong only by wording. The Exam Guide uses "**injected** summaries", so pick the option that matches its wording |
| Resume and auto re-call **all** previous tool results | ❌ Keeps 32 turns of bloat and wastes calls re-running everything |
| Resume full conversation + prompt to prioritize the pending refund | ❌ Stale "Pending Refund" status may be wrong now; context bloat |

**Tips:** Old tool results are **stale**. When prior tool data is outdated, **start fresh + inject a structured summary + re-fetch only what's needed**. Resume only when prior context is still mostly valid. When two options look identical, choose the one whose wording mirrors the Exam Guide.

**Source check (official Exam Guide, Task Statement 1.7):** "Why starting a new session with a structured summary is more reliable than resuming with stale tool results"; "Choosing between session resumption (when prior context is mostly valid) and starting fresh with injected summaries (when prior tool results are stale)."

### Example question (my answer marked wrong, but nearly identical to the key)

**Q:** A customer returns 4 hours later about the same billing dispute. The previous 32-turn session has lookup results saying "Status: Pending Refund" and many tokens from prior lookups. The agent must be ready to answer fully. Most reliable approach?

- ⚠️ Create a new session with a structured summary (issue type, resolution steps, current status), then fresh tool calls (my answer): same approach as the key; the only difference is "with" vs. "inject".
- ❌ Resume with state and auto re-call all previous tool results: bloat and unnecessary calls.
- ❌ Resume full conversation + prompt to prioritize the pending refund: relies on stale status.
- ✅ **Start a new session, inject a structured summary of the previous interaction, then make fresh tool calls as needed**

**Site's rationale reviewed:** The explanation for C (stale data after 4 hours → new session + structured summary + fresh tool calls) applies **word for word to B too**. The rejection of B is templated ("doesn't address the requirement… data may be stale"), yet B already makes fresh tool calls. The site gives no real difference, which confirms the question is flawed. Takeaway: the *concept* is what matters. If both appear on an exam, prefer the Exam Guide wording "inject".

---

## Practice Test 2 — Q61: Invoice Line Items vs. Grand Total Mismatch: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Extracted values must agree mathematically (line items sum vs. grand total), and mismatches come from **multiple independent causes** (OCR errors *and* model extraction mistakes) | **Add a `calculated_total` field (model sums the extracted line items) alongside the extracted `stated_total`/`grand_total`, with a flag (e.g. `is_total_consistent`) that routes mismatches to human review** |
| Tempted to fix with few-shot examples of correctly-summing invoices | Trap: guidance only, no guarantee — doesn't address either root cause (OCR *or* model error) |
| Tempted to auto-adjust line items proportionally to match the stated total | Trap: **silently corrupts data** — you don't know which value (line items or stated total) is actually wrong |
| Tempted to add a second "validation" model to reconcile the two | Trap: extra cost/latency, and the second model can itself be wrong — arithmetic consistency is **deterministic** and doesn't need an LLM to check |

**Tips:** Same pattern as the original invoice-totals question (Q1 of Practice Test 1) — whenever extracted values must satisfy a deterministic mathematical relationship (sum, subtotal + tax = total), build **self-verification into the schema**: extract both sides independently, compute the check in code, and flag disagreements for a human. Never auto-correct without evidence, and never delegate a deterministic check to a second probabilistic model.

**Source check (official Exam Guide, Task Statement 4.3, 4.4):**
- 4.3: "strict JSON schemas via tool use eliminate syntax errors but do not prevent semantic errors (e.g., line items that don't sum to total, values in wrong fields)"
- 4.4: "Designing self-correction validation flows: extracting 'calculated_total' alongside 'stated_total' to flag discrepancies"

### Example question (Practice Test 2, Q61 — answered correctly)

**Q:** An extraction pipeline processes invoices and extracts line items, subtotals, tax amounts, and grand totals. During evaluation, in 18% of extractions the sum of extracted line item amounts doesn't match the extracted grand total — sometimes due to OCR errors in the source document, sometimes due to extraction mistakes by the model. Downstream accounting systems reject records with mismatched totals. What's the most effective approach to improve extraction reliability?

- ❌ Add few-shot examples demonstrating invoices where extracted line items sum correctly to the stated total, encouraging the model to produce mathematically consistent extractions: guidance only, no guarantee — doesn't fix either the OCR-error cause or the model-mistake cause.
- ❌ Implement post-processing that automatically adjusts line item amounts proportionally when their sum doesn't match the stated total: silently corrupts data — you don't know whether the line items or the stated total is the actually-wrong value.
- ✅ **Add a `calculated_total` field where the model sums extracted line items alongside a `stated_total` field. Flag records for human review when values differ.**: deterministic self-check, preserves the original extracted data, routes only genuine mismatches to review.
- ❌ Extract line items and totals independently, then use a separate validation model to reconcile discrepancies by determining which extracted values are most likely correct: another probabilistic model step, added cost/latency, and it can itself be wrong — arithmetic consistency should be checked deterministically in code, not guessed by another model.

**Glossary (thuật ngữ):**
- *self-verification* = tự kiểm chứng (mô hình tự tính toán và so sánh với giá trị đã trích xuất, không cần mô hình thứ hai để xác minh)
- *deterministic* = tất định (kết quả tính toán luôn cố định, không phụ thuộc vào xác suất hay "phán đoán" của mô hình)
- *calculated_total / stated_total* = tổng do model tự tính bằng cách cộng line items / tổng đã được trích xuất trực tiếp từ tài liệu gốc
- *flag for human review* = gắn cờ đánh dấu để chuyển cho con người xem xét lại
- *silently corrupt data* = âm thầm làm sai lệch dữ liệu (sửa dữ liệu mà không có bằng chứng đâu là giá trị đúng)
- *root cause* = nguyên nhân gốc rễ (ở đây có 2 nguyên nhân độc lập: lỗi OCR và lỗi trích xuất của model)

---

## Practice Test 2 — Q62: Untested Code Paths in a 45-File Legacy Payment Module: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Mid-investigation (45-file legacy module), after only 8 files the agent is already less accurate — forgetting patterns, hasn't located all test files or traced critical flows | **Spawn subagents for specific remaining questions (e.g., "find all test files", "trace refund flow dependencies") while the main/coordinator agent preserves high-level understanding and combines results** |
| Tempted to `/clear` + selectively re-read "critical files" + scratchpad | Trap: `/clear` throws away the 8 files' worth of findings already made, and re-reading refills the context with the same problem |
| Tempted to switch to Grep for function names instead of reading full files | Trap: reduces *new* loading going forward, but does nothing to fix the context that is *already* degraded, and narrow Grep hits can't trace a full data/call flow |
| Tempted to write a full summary report, clear context completely, use the report as the sole reference | Trap: discards the main coordinating context entirely; a report alone can't hold the same working detail as continuing to coordinate live, and this abandons the main agent's role instead of just offloading verbose exploration |

**Tips:** Identical pattern to the "Context Filling Up Mid-Investigation" rule: mid-task, accuracy already dropping, and real work remains (untested paths still to find, flows still to trace) → **delegate the remaining work to subagent(s) with specific, targeted questions**, and let the coordinator keep the high-level thread instead of doing all the reading itself. This isolates the verbose file-reading into a subagent's disposable context while the main agent's own context stays clean enough to keep coordinating.

**Source check (official Exam Guide, Task Statement 5.4):**
- "Subagent delegation for isolating verbose exploration output while the main agent coordinates high-level understanding"
- "Summarizing key findings from one exploration phase before spawning sub-agents for the next phase, injecting summaries into initial context"
- Same knowledge point also covers: "Context degradation in extended sessions: models start giving inconsistent answers... rather than specific classes/patterns discovered earlier"

### Example question (Practice Test 2, Q62 — answered correctly; near-duplicate of the 15-file payment-module question above)

**Q:** An engineer asks your agent to identify untested code paths in a legacy payment processing module spanning 45 files. After reading the first 8 source files, the agent's responses are becoming noticeably less accurate — it's forgetting previously discussed code patterns and hasn't yet located all test files or traced critical payment flows. What's the most effective approach to complete this investigation?

- ✅ **Spawn subagents to investigate specific questions (e.g., "find all test files for payment processing", "trace refund flow dependencies") while the main agent coordinates findings and preserves high-level understanding.**: clean context for the remaining reading, main agent keeps the coordinating thread and combines results — the standard fix for context degrading mid-investigation with work still remaining.
- ❌ Clear context with `/clear`, then selectively re-read only the most critical files discovered so far, writing key findings to a scratchpad file that persists between context resets: `/clear` discards the findings from the 8 files already read, and re-reading them (even selectively) refills the context with the same accumulation problem.
- ❌ Switch to using Grep to search for specific function names instead of reading full files, reducing the content loaded into context for remaining exploration: only slows *future* accumulation — it does nothing to fix the accuracy already lost, and narrow function-name matches can't trace a full payment flow across files.
- ❌ Document all current findings in a summary report, clear context completely, then use that report as the sole reference for continuing the investigation: throws away the main coordinating context; a static report can't substitute for an agent that keeps orchestrating the rest of the investigation and integrating new findings.

**Glossary (thuật ngữ):**
- *context degradation* = suy giảm chất lượng ngữ cảnh (mô hình bắt đầu trả lời chung chung, quên chi tiết cụ thể đã tìm thấy trước đó, khi hội thoại/ngữ cảnh quá dài)
- *subagent delegation* = giao việc cho subagent (một agent con được sinh ra để làm một phần việc cụ thể, tách khỏi ngữ cảnh chính)
- *coordinator / main agent* = agent điều phối / agent chính (agent giữ vai trò tổng hợp, không tự đọc hết mọi chi tiết mà giao việc và tổng hợp kết quả)
- *high-level understanding* = hiểu biết tổng quan/cấp cao (nắm được bức tranh lớn, không sa vào từng chi tiết nhỏ)
- *isolating verbose output* = cô lập lượng output dài dòng (đẩy phần đọc/khám phá tốn nhiều ngữ cảnh sang subagent, để ngữ cảnh chính luôn gọn)
- *scratchpad file* = tệp ghi chú tạm (dùng để lưu phát hiện chính, đọc lại khi cần, khác với việc xóa sạch ngữ cảnh)

---

## Practice Test 2 — Q63: Coordinator Narrates Delegation But Never Invokes Subagents: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Coordinator has correct AgentDefinitions for all subagents (descriptions, prompts, tool restrictions), correctly *reasons* about when to delegate ("I'll ask the web search agent..."), but **no subagent execution ever occurs**, and it proceeds as if delegation happened — no errors in logs | **The coordinator's `allowedTools` doesn't include `"Task"`, so it can reason about delegation but has no tool to actually spawn subagents** |
| Tempted to blame subagent context isolation (task descriptions not reaching subagents) | Trap: irrelevant here — no subagent ever *runs*, so there's no context-forwarding problem to fix; this diagnosis assumes execution started, which it didn't |
| Tempted to blame the system prompt not listing available subagent types | Trap: contradicted by the scenario itself — the coordinator clearly *knows* about the web search agent (it names it explicitly), so knowledge isn't the gap |
| Tempted to blame `max_tokens` truncating the Task call | Trap: truncation would produce a cut-off/incomplete tool call or garbled output, not a clean, complete sentence describing delegation with zero errors in the logs |

**Tips:** Same lesson as before: **knowing about a subagent (via AgentDefinitions/descriptions) is not the same as being able to invoke one.** `AgentDefinitions` describe *what* subagents exist; `allowedTools` decides *whether* the coordinator can call the `Task` tool that actually spawns them. When a coordinator narrates an action ("I'll ask X agent...") without a matching tool call ever appearing in the trace, and logs show no errors, the near-universal cause is a missing tool permission, not a reasoning, context-passing, or token-limit problem.

**Source check (official Exam Guide, Task Statement 1.3):**
- "The Task tool as the mechanism for spawning subagents, and the requirement that allowedTools must include 'Task' for a coordinator to invoke subagents"

### Example question (Practice Test 2, Q63 — answered correctly; near-duplicate of the earlier "Coordinator Can't Delegate" question)

**Q:** The coordinator agent has AgentDefinitions configured for all four specialized subagents, each with appropriate descriptions, prompts, and tool restrictions. During testing, you notice the coordinator correctly reasons about when to delegate — it generates messages like "I'll ask the web search agent to find sources on this topic" — but no subagent execution ever occurs. The coordinator then proceeds as if the delegation happened and continues with incomplete information. Logs show no errors. What is the most likely cause?

- ❌ Subagent context isolation means task descriptions from the coordinator don't automatically reach subagents; you need to configure explicit context forwarding in ClaudeAgentOptions: irrelevant — no subagent ever executes at all, so there's nothing for context forwarding to fix.
- ❌ The AgentDefinitions are configured correctly, but the coordinator's system prompt doesn't explicitly list the available subagent types, preventing the model from knowing they can be invoked: contradicted by the scenario — the coordinator clearly names the web search agent, so it does know about it.
- ✅ **The coordinator's `allowedTools` configuration doesn't include "Task", so while it can reason about delegation, it cannot invoke the tool required to spawn subagents.**: matches the exact symptom — correct reasoning, no execution, no errors (there's no error because the model simply never attempts a disallowed tool call in a way that surfaces as one; it just narrates instead).
- ❌ The coordinator's `max_tokens` setting is too low, causing the Task tool invocation to be truncated before the subagent type parameter can be specified: would show up as a truncated/malformed call or an error, not a clean narrative sentence with zero logged errors.

**Glossary (thuật ngữ):**
- *AgentDefinitions* = định nghĩa agent (mô tả subagent nào tồn tại: mô tả, prompt, giới hạn công cụ — nhưng không tự cấp quyền gọi)
- *allowedTools* = danh sách công cụ được phép dùng (cấu hình quyết định coordinator được gọi công cụ nào, bao gồm cả "Task")
- *Task tool* = công cụ Task (cơ chế thực sự dùng để "sinh ra"/gọi chạy một subagent — không có trong allowedTools thì không gọi được, dù agent có biết về subagent đó)
- *narrate without invoking* = nói ra ý định nhưng không thực sự gọi (mô hình mô tả hành động bằng lời nhưng không có tool call tương ứng xảy ra)
- *truncation* = bị cắt ngắn (do giới hạn max_tokens, khiến tool call không hoàn chỉnh — sẽ để lại dấu vết lỗi/không hoàn chỉnh, khác với tình huống câu hỏi này)

---

## Practice Test 2 — Q64: Query Routing by Complexity, "Diverse and Evolving" (confirms the disputed Q44 pattern)

**Note:** This question is a near-duplicate of the disputed Q44 in Practice Test 1 ("Dynamic Query Routing by Complexity"), but here the platform's own answer key agrees with the Exam Guide (dynamic coordinator analysis = correct, fast-path bypass = explicitly marked wrong/"Sai"). This confirms the Exam Guide reading was right and the Q44 site's key was the outlier.

| Situation | Best approach |
|---|---|
| Query distribution is **diverse and evolving** as users discover new applications; some queries are simple (single fact), others complex (comparative research) | **Coordinator analyzes each query and dynamically decides which subagents to invoke, based on its own assessment of that query's requirements** |
| Simple fact queries currently traverse all subagents sequentially (40+ seconds, high token cost) while only complex queries actually need the full pipeline | Confirms the coordinator should skip unnecessary subagents per-query — but by *reasoning*, not by a fixed rule |
| Tempted to build pattern-based routing (single-fact vs. comparative vs. analytical → predefined subagent combination) | Trap: rigid categories decided in advance; breaks as users discover new applications the categories don't anticipate |
| Tempted to hard-code a fast-path that bypasses subagents entirely for "factual" questions, full pipeline for everything else | Trap (marked wrong here too): a static binary split — still routes *every* "other" query through the *complete* pipeline regardless of whether it actually needs all four subagents, and can't adapt as the query mix evolves |
| Tempted to train a query-complexity classifier on labeled historical data | Trap: extra infrastructure, and a classifier trained on historical data can't anticipate *new* query types as usage evolves |

**Tips:** The recurring exam signal is the phrase **"diverse/uneven and evolving."** Any solution that hard-codes categories, patterns, or a classifier trained on the past will break as new query shapes appear. The Exam Guide's answer is always the same shape: **the coordinator itself analyzes each query, per request, and dynamically chooses which subagents to invoke** — no predefined buckets, no static fast-track, no offline-trained classifier.

**Source check (official Exam Guide, Task Statement 1.2):**
- "The role of the coordinator in... deciding which subagents to invoke based on query complexity"
- "Designing coordinator agents that analyze query requirements and dynamically select which subagents to invoke rather than always routing through the full pipeline"

### Example question (Practice Test 2, Q64 — answered correctly; confirms the Q44 dispute resolution)

**Q:** In production, you observe that simple fact-checking queries (e.g., "What year was the Paris Climate Agreement signed?") traverse all four subagents sequentially, consuming 40+ seconds and significant tokens per query. Complex comparative research benefits from the full pipeline. Your query distribution is diverse and evolving as users discover new applications. What's the most effective approach to optimize for varying query complexity?

- ❌ Implement pattern-based routing that categorizes queries by structure (single-fact vs. comparative vs. analytical) and maps each category to a predefined subagent combination: rigid, predefined categories can't anticipate new query shapes as usage evolves.
- ✅ **Have the coordinator analyze each query and dynamically decide which subagents to invoke based on its assessment of query requirements.**: adapts per query and as the distribution evolves — matches the Exam Guide's wording almost verbatim.
- ❌ Create a fast-path for factual questions that bypasses subagents entirely, routing all other queries through the complete pipeline to ensure research thoroughness: a static binary rule; still forces *every* "other" query through the *full* pipeline even when it doesn't need all four subagents, and can't adapt to new query types.
- ❌ Train a query complexity classifier on labeled historical data to predict optimal subagent combinations, retraining periodically as query patterns evolve: extra infrastructure and lag — a classifier is only as good as its (necessarily historical) training data, so it always trails genuinely new applications users discover.

**Glossary (thuật ngữ):**
- *dynamic routing / dynamically decide* = định tuyến động (coordinator tự đánh giá từng truy vấn tại thời điểm xử lý, không theo luật cố định lập sẵn)
- *pattern-based routing* = định tuyến theo mẫu cố định (phân loại truy vấn theo cấu trúc đã biết trước rồi ánh xạ sang một tổ hợp subagent cố định — cứng nhắc, không thích ứng được)
- *fast-path* = đường tắt (bỏ qua toàn bộ subagent cho một nhóm truy vấn được coi là "đơn giản" — vẫn là luật tĩnh, không phải suy luận theo từng truy vấn)
- *query complexity classifier* = bộ phân loại độ phức tạp truy vấn (một mô hình huấn luyện riêng để đoán tổ hợp subagent tối ưu — có độ trễ vì luôn dựa trên dữ liệu lịch sử)
- *diverse and evolving* = đa dạng và luôn thay đổi (từ khóa tín hiệu trong đề: bất kỳ giải pháp cố định/định trước nào đều là bẫy)

---

## Practice Test 2 — Q65: Uniform MCP Error Responses Cause Inconsistent Agent Behavior (near-duplicate of the isError lesson)

| Situation | Best approach |
|---|---|
| `lookup_order` failures all return the same uniform response (`isError: true`, generic text "Operation failed"), so the agent sometimes over-retries (order truly doesn't exist), sometimes escalates prematurely (a transient network blip), sometimes asks the user for clarification (when it's really a backend permission error) | **Enhance error responses with structured metadata: `errorCategory` (transient/validation/permission), an `isRetryable` boolean, and a description of what actually caused the failure** |
| Tempted to fix it with few-shot examples teaching the agent to interpret error-message *patterns* | Trap: the underlying text is already generic/uniform ("Operation failed") — there's no distinguishing pattern in the text for examples to teach the model to recognize |
| Tempted to add a separate `analyze_error` MCP tool the agent calls after any failure | Trap: an extra round-trip/tool call per failure, and it pushes classification *away* from the source (the original tool) that already knows exactly why it failed |
| Tempted to add retry-with-exponential-backoff inside the MCP server for *all* errors, only returning to the agent once retries are exhausted | Trap: retries help transient errors but are wasted effort (and added latency) on validation/permission errors that will never succeed no matter how many times they're retried — and it still doesn't tell the agent *why* it ultimately failed |

**Tips:** Same core lesson as `lookup_order`/`isError` earlier: the tool (or its backend) already knows *why* a call failed. The fix belongs at the **source of the error**, encoded as **structured fields** the agent can branch on programmatically (`errorCategory`, `isRetryable`) — not as free text for the agent to guess-parse, not as an extra tool hop, and not as blanket retry logic that ignores the error's actual type.

**Source check (official Exam Guide, Task Statement 2.2):**
- "The MCP isError flag pattern for communicating tool failures back to the agent"
- "Returning structured error metadata including errorCategory (transient/validation/permission), isRetryable boolean, and human-readable descriptions"
- "Distinguishing between access failures (needing retry decisions) and valid empty results"

### Example question (Practice Test 2, Q65 — answered correctly; near-duplicate of the earlier lookup_order/isError question)

**Q:** Production logs reveal inconsistent error handling: when `lookup_order` fails, the agent sometimes retries 5+ times (wasteful when the order ID doesn't exist), sometimes escalates immediately (premature for temporary network issues), and sometimes asks users for clarification (inappropriate when the issue is a backend permission error). Investigation shows your MCP tool returns uniform error responses: `{"isError": true, "content": [{"type": "text", "text": "Operation failed"}]}`. The agent cannot distinguish between error types. What's the most effective improvement?

- ❌ Add few-shot examples to the system prompt demonstrating how to interpret error message patterns and select appropriate responses for each: there's no pattern to interpret — every failure returns the exact same generic text, so examples have nothing distinguishing to teach from.
- ❌ Create an `analyze_error` MCP tool the agent calls after any failure to determine the error category and recommended action: an extra tool call per failure, and moves classification away from the original tool/backend that already has the real answer.
- ✅ **Enhance error responses with structured metadata: include `errorCategory` (transient/validation/permission), `isRetryable` boolean, and a description of what caused the failure.**: fixes the problem at its source — the agent now has explicit, structured signals to decide retry vs. escalate vs. clarify, instead of guessing from identical text.
- ❌ Implement retry logic with exponential backoff in your MCP server for all errors, returning to the agent only after retries are exhausted: wastes time/latency retrying errors that can never succeed (validation, permission), and still leaves the agent without any explanation once retries run out.

**Glossary (thuật ngữ):**
- *uniform error response* = phản hồi lỗi đồng nhất (mọi loại lỗi đều trả về cùng một nội dung chung chung, khiến agent không phân biệt được)
- *errorCategory* = loại lỗi được phân loại rõ ràng (transient = tạm thời, validation = lỗi dữ liệu đầu vào, permission = lỗi quyền truy cập)
- *isRetryable* = cờ boolean cho biết lỗi này có nên thử lại hay không
- *structured metadata* = siêu dữ liệu có cấu trúc (thông tin đặt trong các trường rõ ràng, máy đọc được, thay vì chỉ là văn bản tự do)
- *push classification away from the source* = đẩy việc phân loại ra xa khỏi nơi biết rõ nguyên nhân nhất (ở đây là chính MCP tool/backend — nơi lẽ ra nên trả lỗi có phân loại ngay từ đầu)

---

## Practice Test 2 — Q66: Iterative Research Loops for Closing Analysis Gaps

| Situation | Best approach |
|---|---|
| Fixed, single-pass pipeline (search → analysis → synthesis); the analysis agent explicitly identifies a specific, well-defined gap in the retrieved sources (e.g., "discusses X but lacks Y"), but under the current rigid pipeline this insight goes nowhere because the search phase has already finished | **Have the analysis agent report the specific gap to the coordinator as an explicit finding; the coordinator triggers a targeted follow-up search for exactly that gap and re-invokes analysis, looping until coverage is sufficient** |
| Coordinator itself scans/interprets the analysis agent's output looking for implicit "gap indicators" to decide whether to re-search | Trap: puts the burden of *detecting* the gap on the coordinator, which didn't do the analysis — fragile, since it depends on the coordinator successfully inferring something from free-form output instead of the analysis agent stating it as an explicit, structured finding it already has |
| Attach confidence scores per section in the final synthesis and flag low-coverage areas for manual (human) review | Trap: treats the symptom only at the very end, after synthesis, and routes a *recoverable* problem to a human instead of using the gap the system already identified to automatically go get the missing information |
| Add an upfront research-planning agent that decomposes topics into sub-questions before the search phase even starts | Trap: front-loaded, one-shot planning cannot anticipate a gap ("lacks token refresh patterns") that is only discoverable *after* documents are retrieved and analyzed — the whole premise here is that the gap surfaces during analysis, not before search |

**Tips:** This combines two ideas already in this cheat sheet: (1) **dynamic, adaptive decomposition** — generate the next subtask (a targeted search) based on what was actually discovered, rather than planning everything upfront (Task Statement 1.6); and (2) **hub-and-spoke discipline** — the agent that discovers something should report it explicitly and in structured form, and the *coordinator* is the one that decides to re-invoke earlier pipeline stages (Task Statement 1.2/1.3), not the other way around. Also apply the "is the missing information actually recoverable?" test from the retry-with-feedback rule: here it *is* recoverable (more targeted search can find token-refresh docs), so the system should loop to get it automatically — pushing straight to human review would be appropriate only if the information genuinely couldn't be found by searching more.

**Source check (official Exam Guide, Task Statement 1.6, 1.2/1.3):**
- 1.6: "The value of adaptive investigation plans that generate subtasks based on what is discovered at each step"; "dynamic adaptive decomposition based on intermediate findings"
- 1.2/1.3: coordinator manages all inter-subagent routing and re-invocation decisions; subagents report structured findings that the coordinator acts on, rather than the coordinator inferring intent from unstructured output

### Example question (Practice Test 2, Q66 — answered correctly)

**Q:** Users report that final reports sometimes lack depth on specific subtopics. Investigation shows that the document analysis agent frequently identifies gaps — for instance, noting "the retrieved sources discuss API authentication but lack details on token refresh patterns" — but under the current strict pipeline, this insight isn't actionable since search has already completed. What's the most effective architectural change?

- ❌ Have the synthesis agent attach confidence scores to each section and flag areas with insufficient coverage for manual review: only reacts after the fact with a human safety net, and ignores that the analysis agent already pinpointed exactly what's missing and more search could resolve it automatically.
- ✅ **Have the analysis agent report specific gaps to the coordinator, which triggers targeted searches and re-invokes analysis until sufficient.**: turns the one-pass pipeline into an adaptive loop — the party that found the gap reports it explicitly, and the coordinator acts on that explicit signal by fetching exactly what's missing and re-checking, until coverage is actually sufficient.
- ❌ Add a research planning agent before the search phase that decomposes topics into specific sub-questions: happens too early — this particular gap (missing token-refresh details) can only be discovered once documents have already been retrieved and analyzed, so no amount of upfront planning would have anticipated it.
- ❌ Have the coordinator review analysis output for gap indicators and re-invoke search with gap-informed queries when gaps are detected: close, but wrong emphasis — it makes the *coordinator* respons­ible for spotting the gap by reviewing/interpreting the analysis output, instead of having the analysis agent (which actually did the work and already noticed the gap) report it explicitly; it also lacks the "re-invokes analysis until sufficient" loop, so it's only a single reactive pass rather than an iterative fix.

**Glossary (thuật ngữ):**
- *rigid / strict pipeline* = pipeline cứng nhắc, chạy một lượt cố định (search → analysis → synthesis) không có vòng lặp quay lại
- *feedback loop / iterative loop* = vòng lặp phản hồi (kết quả của một bước được dùng để quyết định có cần chạy lại bước trước đó hay không, lặp đến khi đạt yêu cầu)
- *targeted search* = tìm kiếm có mục tiêu cụ thể (tìm đúng vào phần thông tin còn thiếu, thay vì tìm kiếm lại từ đầu)
- *explicit structured finding* = phát hiện được báo cáo rõ ràng dưới dạng có cấu trúc (agent tự nêu ra gap của mình, thay vì để bên khác phải suy đoán)
- *gap indicator* = dấu hiệu cho thấy thiếu thông tin (ở đây, việc để coordinator tự "soi" ra dấu hiệu này kém tin cậy hơn việc phân tích-agent tự báo cáo thẳng)
- *recoverable information* = thông tin có thể tìm lại được (khác với thông tin thực sự không tồn tại trong nguồn — nếu recoverable thì nên thử tìm thêm, không nên đẩy thẳng cho con người)

---

## Practice Test 2 — Q67: Integrating Jira Through an Existing MCP Server

| Situation | Best approach |
|---|---|
| Team currently copy-pastes Jira ticket content manually into conversations; wants the agent to access this **standard** ticket data (tickets, comments, metadata) directly | **Integrate an existing Jira MCP server that already exposes tickets, comments, and metadata through discoverable tool interfaces** |
| Tempted to call Jira's REST API directly via Bash + curl, handling auth headers and parsing JSON inline | Trap: reinvents an integration that already exists as a maintained MCP server — no discoverable tool schema for the model to reason about, ad hoc auth/credential handling in shell commands, and brittle JSON parsing done inline instead of by a proper tool contract |
| Tempted to export Jira tickets to markdown files in the repo and have the agent Read them | Trap: a static snapshot that goes stale immediately — new comments, status changes, and updates on the live ticket won't show up, and the "export" step is still manual work, just moved earlier in the pipeline |
| Tempted to build a brand-new custom MCP server wrapping Jira's API, purpose-built for this team's code review workflow | Trap: unnecessary reinvention — the need described is **standard** ticket/comment/metadata access, which an existing, already-available Jira MCP server already covers; building a custom server only makes sense when the need is genuinely specific to a workflow that no existing integration serves |

**Tips:** When a team needs an agent to reach a common external system (Jira, GitHub, Slack, a database) for **standard** data access, prefer an **existing, already-built MCP server** for that system over: (a) raw API calls via Bash/curl (loses discoverability, structured error handling, and a stable tool contract), (b) static file exports (go stale, still require a manual step), or (c) building a custom MCP server from scratch (unnecessary engineering effort when a standard integration already does the job — reserve custom MCP servers for genuinely workflow-specific needs an existing server doesn't cover).

**Source check (official Exam Guide, Task Statement 2.4):** MCP servers integration — preferring standard, already-available MCP server integrations for common external systems over ad hoc API calls, static exports, or unnecessary custom server development, so the agent gets a discoverable, structured tool interface with proper authentication handling built in.

### Example question (Practice Test 2, Q67 — answered correctly)

**Q:** [Context: a team currently copy-pastes Jira ticket content into conversations manually.] This currently requires manually copy-pasting content into conversations. The team wants the agent to access this standard Jira ticket data directly. What's the most effective approach?

- ❌ Use the Bash tool with curl to call Jira's REST API, including authentication headers and parsing JSON responses inline: reinvents integration work that already exists as a maintained server, with no discoverable tool interface and fragile inline auth/parsing.
- ✅ **Integrate an existing Jira MCP server that exposes tickets, comments, and metadata through discoverable tool interfaces.**: reuses a standard, already-built integration with a proper tool contract — the correct default when the need is standard ticket/comment/metadata access.
- ❌ Export Jira tickets to markdown files in the repository that the agent accesses using the Read tool: a stale, static snapshot that misses live updates, and still requires a manual export step.
- ❌ Build a custom MCP server wrapping Jira's API with tools designed specifically for this team's code review workflow: unnecessary — the described need is standard data access that an existing Jira MCP server already covers; custom-building is over-engineering here.

**Glossary (thuật ngữ):**
- *MCP server* = máy chủ MCP (Model Context Protocol) — một lớp tích hợp chuẩn hóa, cung cấp các "tool" có mô tả rõ ràng để agent gọi vào hệ thống bên ngoài (ví dụ Jira), thay vì agent tự gọi API thô
- *discoverable tool interfaces* = giao diện công cụ có thể "khám phá" được — nghĩa là model có thể thấy công cụ đó tồn tại, biết input/output của nó, thay vì phải đoán qua các lệnh gọi API thủ công
- *ad hoc / inline* = tùy biến ngay tại chỗ, không theo chuẩn (ví dụ: tự viết header xác thực và tự parse JSON ngay trong câu lệnh Bash, thay vì dùng một tool đã được thiết kế sẵn)
- *static snapshot* = bản chụp tĩnh (dữ liệu được xuất ra một lần, không tự cập nhật theo thời gian thực — dễ bị lỗi thời/stale)
- *reinventing the wheel* = làm lại từ đầu một thứ đã có sẵn (ở đây là tự xây dựng lại tích hợp Jira dù đã có MCP server chuẩn cho việc đó)

---

## Practice Test 2 — Q68: Plan Mode vs. Direct Execution for Production Bugs: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Clear stack trace, narrow scope, symptom points to a specific code location (even in an unfamiliar module) | **Direct execution**: read the trace, inspect the code, fix once root cause is confirmed |
| Ambiguous, multi-step, or high-blast-radius task; requirements unclear; many plausible approaches | **Plan mode**: enumerate root causes / design before touching code |
| Tempting hybrid: "explore a bit directly, then switch to plan mode before implementing" | ❌ Usually unnecessary overhead for a well-scoped bug — a trap answer |
| Deciding factor | **Clarity/complexity of the signal and task**, not whether you're personally familiar with the module |

**Tips:** Don't equate "I haven't worked with this module before" with "I need to plan first." The exam consistently keys mode choice off how well-scoped and unambiguous the problem is, not off the engineer's familiarity. A concrete stack trace is itself a strong, narrowing signal — it already tells you where to look, so front-loading a planning phase (enumerating root causes, designing a "comprehensive solution") adds latency without adding value. Reserve plan mode for genuinely open-ended or high-risk work: unclear requirements, broad architectural change, or multiple plausible approaches needing a deliberate before making changes.

**Source check (official Exam Guide, Task Statement 3.4 — Plan mode vs. direct execution):**
- Task Statement 3.4 covers choosing between plan mode and direct execution based on task characteristics (complexity, ambiguity, risk) rather than by rote habit or by the engineer's personal familiarity with the code.

### Example question (Practice Test 2, Q68 — answered correctly)

**Q:** A critical bug is affecting production users. Error logs show exceptions in the OrderProcessing module with a clear stack trace pointing to a specific area, but you haven't worked with this module before. What's the most effective approach?

- ❌ Use plan mode to analyze the error in context of the module's design, enumerate potential root causes, and prioritize fixes systematically.
- ❌ Start with direct execution to gather initial information, then switch to plan mode to design a comprehensive solution before implementing.
- ✅ **Use direct execution to examine the stack trace, read the relevant code, and implement a fix once you identify the root cause.**
- ❌ Enter plan mode to explore the module's architecture and dependencies before attempting any fix.

**Glossary (thuật ngữ):**
- *Plan mode* = chế độ lập kế hoạch (Claude Code phân tích/đề xuất phương án trước khi chỉnh sửa code, chưa thực thi thay đổi)
- *Direct execution* = thực thi trực tiếp (đọc code, sửa và áp dụng thay đổi ngay, không qua giai đoạn lập kế hoạch riêng)
- *Stack trace* = dấu vết ngăn xếp (chuỗi lời gọi hàm dẫn đến lỗi, giúp xác định vị trí xảy ra exception)
- *Root cause* = nguyên nhân gốc rễ (nguyên nhân thực sự gây ra lỗi, không chỉ là triệu chứng bề mặt)
- *Blast radius* = phạm vi ảnh hưởng (mức độ tác động lan rộng của một thay đổi hoặc lỗi)

---

## Practice Test 2 — Q69: High Precision / Low Recall in Automated Review — Splitting Finding from Thresholding: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| One prompt instructs both "only report high-confidence issues" / "err on the side of not commenting" (precision) **and** must still catch subtle bugs (recall) — real bugs (e.g., a race condition) slip through silently | **Split into two stages: a finding stage that optimizes purely for coverage (flag every potential issue with confidence + severity metadata, no suppression), and a separate thresholding stage that filters those findings** |
| Add few-shot examples of bug categories to catch, but keep the same conservative "high-confidence only" filtering instruction | ❌ Trap: still bundles detection and filtering into one instruction — the underlying conflict (be thorough vs. be quiet) is untouched |
| Remove conservative filtering, report everything, then apply a programmatic dedup/category-suppression filter | ❌ Trap: a rigid category-based filter can't reason about evidence the way a calibrated confidence/severity threshold can; risks re-introducing noise or dropping real bugs by category |
| Expand the context window (test files, git history, dependency graph) | ❌ Trap: richer context can help judgment, but doesn't resolve the structural conflict of asking one pass to be both broad and quiet |

**Tips:** When a single instruction set has to be simultaneously wide (find everything, including subtle issues) and narrow (report only what's certain), the model will bias toward whichever directive is more explicit — usually the "stay quiet" one, since it's the safer-sounding instruction. This is a **precision/recall tradeoff problem**, and the fix is architectural: decouple the two objectives into separate passes so each can be tuned independently (e.g., adjust the threshold without ever weakening the finding stage's willingness to notice things). This is the multi-pass review pattern: pass 1 = generate/find with high recall + structured metadata (confidence, severity); pass 2 = filter/threshold using that metadata.

**Source check (official Exam Guide, Task Statement 4.6 — Multi-pass review):**
- 4.6 covers designing multi-stage review pipelines where a first pass optimizes for coverage/recall (flagging all potential issues with confidence and severity metadata) and a second pass applies thresholding/filtering — rather than conflating both objectives into a single prompt instruction, which forces an implicit and uncontrollable precision/recall tradeoff.

### Example question (Practice Test 2, Q69 — answered correctly)

**Q:** After deploying the automated review, you notice high precision but low recall — real bugs are slipping through undetected. Investigation reveals your review prompt instructs Claude to "only report high-confidence issues you are certain about" and "err on the side of not commenting." Developers appreciate the low noise, but a race condition that caused a production outage was visible in a reviewed PR and went unreported. You need to substantially improve bug detection while keeping false positive rates manageable for your team. What is the most effective approach?

- ❌ Remove the conservative filtering instructions and prompt Claude to report all potential issues, then apply a programmatic filter to deduplicate and suppress categories that historically generate false positives.
- ❌ Add detailed few-shot examples demonstrating bug categories Claude should flag — race conditions, null dereferences, error handling gaps — while keeping the high-confidence filtering instruction to maintain current precision levels.
- ❌ Expand the context window by including related test files, recent git history, and the module's dependency graph alongside the diff, giving Claude richer signals to assess issue severity.
- ✅ **Split the review into a finding stage where Claude's goal is coverage — flagging every potential issue with confidence and severity metadata — and a separate stage that thresholds those findings.**

**Glossary (thuật ngữ):**
- *precision* = độ chính xác (trong số các vấn đề được báo cáo, bao nhiêu % là thật — báo cáo sai càng ít thì precision càng cao)
- *recall* = độ bao phủ / độ nhạy (trong số các vấn đề thật sự tồn tại, bao nhiêu % được phát hiện — bỏ sót càng ít thì recall càng cao)
- *precision/recall tradeoff* = sự đánh đổi giữa độ chính xác và độ bao phủ (tăng cái này thường làm giảm cái kia nếu xử lý trong cùng một bước)
- *finding stage* = giai đoạn phát hiện (tối ưu cho việc tìm ra càng nhiều vấn đề tiềm ẩn càng tốt, chưa lọc)
- *thresholding stage* = giai đoạn áp ngưỡng lọc (dùng điểm tin cậy/mức độ nghiêm trọng để quyết định cái gì đủ đáng tin để báo cáo)
- *multi-pass review* = quy trình rà soát nhiều bước (tách biệt giai đoạn tìm kiếm và giai đoạn lọc/xếp hạng thay vì gộp chung một bước)

---

## Practice Test 2 — Q70: Guaranteeing Preview-Before-Execute with a Single-Use Confirmation Token: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| A single tool with a `dry_run: boolean` parameter is supposed to always be previewed before real execution, but the agent bypasses the preview in some % of calls (production monitoring shows this) | **Split into two tools: a `preview_*` tool that returns impact details plus a single-use confirmation token, and an `execute_*` tool that requires that token — cryptographically/structurally binding execution to the specific previewed action** |
| Add detailed instructions + few-shot examples to the tool description telling the agent to always preview first | ❌ Trap: still just guidance in natural language — probabilistic, matches the exact failure already observed |
| Server-side validation: permit the destructive call only if an identical-parameter preview call happened within the last N seconds | ❌ Trap: a time-window + parameter-matching heuristic; gameable (mismatched or similarly-shaped calls inside the window), and doesn't bind execution to the *exact* previewed action |
| Annotate the tool as requiring confirmation; have the orchestration layer prompt the *user* for approval before forwarding calls | ❌ Trap: solves a different problem (human-in-the-loop approval), not a structural guarantee that the *agent* always previews before executing |

**Tips:** Same family as "guarantee → code, not prompts" (see the refund-hook rule), but the enforcement point here is the **tool interface itself**, not a hook around the agent loop. A boolean flag on one tool is not a guarantee — nothing stops the model from setting it to `false` directly. Splitting into two tools bound by a single-use token makes the required sequence *impossible to skip*, because the second call cannot succeed without a token that only the first call produces, tied to that one specific action. This is a stronger, more general version of "fix the root cause at the right layer": don't patch a probabilistic instruction — restructure the interface so the invalid sequence has no valid path through the API.

**Source check (official Exam Guide, Task Statement 2.1 — Tool interfaces & descriptions, related to 1.4/1.5 guaranteed workflow enforcement):**
- 2.1: designing tool interfaces so that a required sequence (preview before a destructive action) is enforced structurally by the interface contract (e.g., a single-use token binding a follow-up call to a specific prior call), rather than relying on tool descriptions or prompted instructions to produce the correct call order.
- Related to 1.4/1.5: "prompt instructions alone have a non-zero failure rate" for guaranteed compliance — the same principle applied at the tool-design layer instead of via a hook.

### Example question (Practice Test 2, Q70 — answered correctly)

**Q:** Your `remove_team_member` tool uses a `dry_run: boolean` parameter for previewing impacts before execution. Production monitoring shows the agent bypasses the preview step in 15% of calls by calling with `dry_run=false` directly. You need to ensure every removal is preceded by a preview that the user explicitly confirms. What is the most reliable approach?

- ❌ Add detailed instructions and few-shot examples to the tool description requiring the agent to always call with dry_run=true first and wait for user confirmation before calling with dry_run=false.
- ❌ Add server-side validation that permits dry_run=false only when a dry_run=true call with identical parameters occurred within the past 60 seconds.
- ❌ Annotate the tool as requiring confirmation and configure the orchestration layer to prompt the user for approval before forwarding any calls to annotated tools.
- ✅ **Replace with two tools: preview_remove_member returns impact details and a single-use confirmation token; execute_remove_member requires that token, binding execution to the specific previewed action.**

**Glossary (thuật ngữ):**
- *dry_run* = chạy thử / xem trước (thực hiện mọi bước trừ hành động thật, để xem tác động trước khi làm thật)
- *single-use confirmation token* = mã xác nhận dùng một lần (chỉ hợp lệ cho đúng một lần thực thi, gắn với đúng hành động đã được xem trước, dùng xong là mất hiệu lực)
- *bind execution to the specific previewed action* = ràng buộc việc thực thi vào đúng hành động đã được xem trước (không cho phép "xem trước A nhưng thực thi B")
- *structural guarantee* = đảm bảo về mặt cấu trúc (không thể vi phạm được do chính thiết kế API, khác với việc chỉ dựa vào hướng dẫn/prompt)
- *heuristic time-window validation* = kiểm tra theo khoảng thời gian mang tính suy đoán (dựa vào việc có lệnh preview trong N giây gần đây — dễ bị lách nếu tham số không khớp chính xác)

---

## Practice Test 2 — Q71: Blind Retry vs. Retry-With-Feedback for Semantic Validation Failures (confirms existing 4.4 pattern)

**Note:** Near-duplicate of the "Retry only if the answer exists in the input" rule already in this cheat sheet (Task Statement 4.4), but this instance sharpens a specific distinction the exam tests: **blind retry (re-run unchanged, hope for a pass) vs. retry-with-feedback (re-run with the document + failed extraction + validation errors so the model can correct with direction)**.

| Situation | Best approach |
|---|---|
| 12% of invoice extractions fail *semantic* validation (line items don't sum to total, vendor IDs don't match valid formats) — JSON syntax is never the problem, since tool use with strict schemas already guarantees syntax | **On validation failure, make a follow-up request including the document, the failed extraction, and the specific validation errors, so the model corrects with feedback** |
| Retry up to N times, accepting the first attempt that happens to pass validation | ❌ Trap: blind retry — the model never learns what was wrong; a later attempt might pass schema checks by chance without being more semantically correct |
| Stricter schema constraints + more detailed field descriptions to prevent bad values upfront | ❌ Trap: schemas (even detailed ones) guarantee syntax, not semantics — line items can still fail to sum to the total even with perfectly typed fields; doesn't address failures already happening |
| Post-processing that auto-corrects common errors (e.g., recalculating totals from line items when sums mismatch) | ❌ Trap: silent auto-correction — you don't know which value is actually wrong (line items vs. stated total), so "fixing" one can corrupt the data without evidence |

**Tips:** The retry-with-feedback mechanism only works because the missing/wrong information is *recoverable from the input the model already has* (the document itself) — it's a case of "format/structural or semantically-checkable errors → retry with feedback works," distinct from cases where the retry can't help because the correct value simply isn't present anywhere in the input (see the existing "et al." / missing-co-author-list example in this cheat sheet, where retrying risks hallucination instead).

**Source check (official Exam Guide, Task Statement 4.4):**
- "Designing self-correction validation flows" and "retrying with specific validation error feedback so the model can correct semantic mistakes" — as opposed to blind re-invocation with no information about what failed.

### Example question (Practice Test 2, Q71 — answered correctly)

**Q:** Your invoice extraction uses tool use with strict JSON schemas. JSON syntax errors never occur, but 12% of extractions fail semantic validation — for example, line item amounts don't match the extracted total, or vendor IDs don't match valid formats. These failures currently route to manual review. What's the most effective approach to reduce manual review volume while maintaining accuracy?

- ❌ Retry the extraction up to 3 times when validation fails, accepting the first result that passes validation.
- ✅ **When validation fails, make a follow-up request with the document, extraction, and validation errors for model correction.**
- ❌ Add stricter schema constraints with detailed field descriptions to prevent the model from generating invalid values initially.
- ❌ Implement post-processing logic that automatically corrects common errors, such as recalculating totals from line items when sums don't match.

**Glossary (thuật ngữ):**
- *blind retry* = thử lại một cách mù quáng (lặp lại yêu cầu y hệt, không cho mô hình biết nó đã sai ở đâu)
- *retry with feedback* = thử lại có phản hồi (gửi kèm lỗi cụ thể để mô hình tự sửa có định hướng)
- *semantic validation* = kiểm tra tính đúng đắn về mặt ý nghĩa (khác với kiểm tra cú pháp — dữ liệu có thể đúng định dạng nhưng sai về logic/nội dung)
- *silent auto-correction* = tự động sửa lỗi âm thầm (thay đổi dữ liệu mà không biết chắc giá trị nào đúng, có thể làm sai lệch dữ liệu)

---

## Practice Test 2 — Q72: Comparing Techniques for Guaranteed JSON Schema Compliance: Exam Rule of Thumb

| Technique | Guarantee level |
|---|---|
| **Tool use: define a tool with the target schema as input parameters; Claude calls it with the extracted data** | ✅ **Strongest — enforced at the API/generation layer**, structural validation happens as part of producing the tool call itself |
| Pre-fill the response with an opening brace `{` to force JSON output, then complete and parse | ❌ Only guarantees the response *starts* as JSON — nothing constrains the rest to match the schema (missing fields, wrong types, malformed JSON mid-way) |
| Prompted instructions ("output only valid JSON matching the schema exactly") + retry logic that re-prompts on parse failure | ❌ Still prompt-based/probabilistic; better than nothing, but accepts that failures *will* happen and reacts after the fact (added latency/cost) instead of preventing them |
| Detailed JSON formatting instructions + schema in the prompt, then parse Claude's text response as JSON | ❌ Pure "prompted JSON" — no enforcement mechanism at all; relies entirely on the model voluntarily following natural-language instructions |

**Tips:** All three wrong options are variations of the same underlying approach — **Claude writes free text, your code then tries to parse it as JSON**. Any variant of that approach is inherently probabilistic: pre-filling only anchors the start, instructions-only relies on compliance, and instructions+retry just adds a safety net around expected failures. **Tool use is categorically different**: it doesn't ask the model to *write* JSON that happens to match a schema, it defines the schema as the tool's actual input parameters, so producing a tool call *is* producing schema-conformant structured data — this is the strongest guarantee technique in the exam's model for structured output.

**Source check (official Exam Guide, Task Statement 4.3 — Tool use & JSON schemas):**
- "Using tool use with strict JSON schemas as the most reliable mechanism for guaranteed schema-conformant structured output, as opposed to prompted JSON generation (with or without response pre-filling or retry logic), which remains probabilistic."

### Example question (Practice Test 2, Q72 — answered correctly)

**Q:** Your system must extract event details from calendar invitations and output JSON that strictly conforms to a schema with fields for title, date, time, location, and attendees. Downstream reject any malformed or non-conformant JSON. What approach provides the most reliable schema compliance?

- ❌ Pre-fill Claude's response with an opening brace to force JSON output, then complete and parse the response.
- ❌ Append instructions like "Output only valid JSON matching the schema exactly" and implement retry logic to re-prompt when JSON parsing fails.
- ❌ Include detailed JSON formatting instructions and the target schema in your prompt, then parse Claude's text response as JSON.
- ✅ **Define a tool with your target schema as input parameters and have Claude call it with the extracted data.**

**Glossary (thuật ngữ):**
- *tool use / function calling* = sử dụng công cụ (thay vì để mô hình viết văn bản tự do, ta định nghĩa một "hàm" với tham số đầu vào cụ thể — mô hình "gọi hàm" bằng cách điền tham số theo đúng cấu trúc)
- *response pre-filling* = mồi trước phản hồi (đặt sẵn một phần đầu của câu trả lời, ví dụ dấu `{`, để hướng mô hình bắt đầu đúng định dạng — nhưng không đảm bảo toàn bộ phần còn lại đúng)
- *prompted JSON* = JSON được yêu cầu qua prompt (mô hình viết chuỗi văn bản dạng JSON theo hướng dẫn, sau đó hệ thống mới parse — không có ràng buộc kỹ thuật nào đảm bảo đúng)
- *schema-conformant* = tuân thủ đúng lược đồ (dữ liệu có đầy đủ trường, đúng kiểu, đúng cấu trúc như đã định nghĩa)
- *guaranteed at the API/generation layer* = được đảm bảo ngay ở tầng API/tạo sinh (ràng buộc xảy ra trong lúc mô hình tạo ra kết quả, không phải kiểm tra lại sau khi đã tạo xong)

---

## Practice Test 2 — Q73: Tool Definitions Consuming Context Budget Near the Window Limit (answered correctly — confirmed)

**Note:** Initially answered by reasoning alone (no "Đúng/Sai" tag was visible on the first screenshot). The user later re-sent the same question with the platform's key revealed, confirming this reasoned answer was correct.

| Situation | Best approach |
|---|---|
| Tool definition ≈2,500 tokens; documents <150K tokens → 98% accuracy; documents 175–190K tokens → 71% accuracy, with the **final third of the document consistently missed**; model's context window = 200K | **Tool definitions consume input context tokens too. Combined with the system prompt and document content, the total approaches the 200K context limit, degrading processing of content near the end of the document** — a sharp cliff near a hard boundary, not a smooth length-based decline |
| "Model distributes attention proportionally; fields mentioned once near the document's end get insufficient focus" | ❌ Trap: backwards from the well-known primacy/recency pattern (beginning and end normally get *more* attention; the *middle* is what's typically neglected/"lost in the middle") — unless context overflow is the real driver, which the given numbers point to |
| "Schemas exceeding 8–10 fields increase decision complexity during parameter generation, reducing accuracy independent of document length" | ❌ Trap: directly refuted by the data — the same 12-field schema gets 98% accuracy on shorter documents, so complexity alone can't be a length-independent cause |
| "Very long documents exceed the model's effective attention span regardless of context limits" | ❌ Trap: the phrase "regardless of context limits" explicitly waves away the exact factor (the 200K window, and numbers that sum close to it) the question's setup is built around |

**Tips:** When a question hands you specific numbers (tool definition size, document token ranges, context window size) that sum to something close to the stated limit, that's the intended signal — the exam wants you to notice that **tool definitions, system prompts, and document content all count against the same context budget**, and that budget pressure explains both the sharp (not gradual) accuracy cliff and which part of the input degrades first (the end, since that's what's nearest the boundary once everything is summed). Be suspicious of any option that explicitly denies the numeric hint the question just gave you (e.g., "regardless of context limits").

**Source check (official Exam Guide, Task Statement 5.4 — Large codebase/document context management):**
- 5.4 covers budgeting the full context window across all contributors — system prompt, tool definitions, and document/tool-result content — and recognizing that approaching the total context limit degrades processing, particularly for content nearest the boundary, rather than treating the document alone against the model's raw context size.

### Example question (Practice Test 2, Q73 — answered correctly)

**Q:** Your extraction system uses tool_use with a JSON schema containing 12 fields and detailed descriptions, totaling approximately 2,500 tokens for the complete tool definition. Processing documents under 150K tokens yields 98% accuracy. For documents between 175–190K tokens, accuracy drops to 71%, with information from the final third consistently missed. The model's context window is 200K tokens. What is the most likely cause?

- ❌ The model distributes attention proportionally across input length, causing fields mentioned only once near the document's end to receive insufficient processing focus.
- ❌ Schemas exceeding 8-10 fields increase decision complexity during parameter generation, reducing extraction accuracy independent of document length.
- ❌ Very long documents exceed the model's effective attention span regardless of context limits, causing accuracy degradation for content farther from the prompt instructions.
- ✅ **Tool definitions consume input context tokens. Combined with system prompts and document content, the total approaches the context limit, degrading end-of-document processing.**

**Glossary (thuật ngữ):**
- *context budget* = ngân sách ngữ cảnh (tổng số token khả dụng trong context window, phải chia sẻ giữa system prompt, tool definitions, và nội dung tài liệu/tool result)
- *tool definition tokens* = token của định nghĩa tool (mô tả tool, schema, tên field... cũng tính vào tổng input context, không phải "miễn phí")
- *accuracy cliff* = vách đá độ chính xác (sự sụt giảm đột ngột, không mượt mà, thường là dấu hiệu chạm giới hạn cứng thay vì suy giảm dần theo khoảng cách)
- *primacy/recency effect* = hiệu ứng đầu-cuối (nội dung ở đầu và cuối ngữ cảnh thường được chú ý nhiều hơn phần giữa — "lost in the middle" mô tả việc phần *giữa* bị bỏ sót, không phải phần cuối)
- *effective attention span* = phạm vi chú ý hiệu quả (khái niệm mô hình "chú ý" tốt trong một khoảng nhất định của input — nhưng câu hỏi này cho thấy nguyên nhân thực sự là ngân sách token, không phải một giới hạn chú ý trừu tượng)

---

## Practice Test 2 — Q74: tool_choice "any" for Guaranteed Structured Output Across Unknown Document Types (confirms existing tool_choice pattern)

**Note:** Near-duplicate/confirmation of the existing "tool_choice: Forcing Tool Calls" rule (Task Statement 4.3) — this instance specifically tests the `"any"` case (as opposed to `"auto"` or a forced-specific-tool), which hadn't yet had its own worked example in this cheat sheet.

| Situation | Best approach |
|---|---|
| Multiple extraction tools exist, each with a schema tailored to a different document type (invoice/contract/receipt); the document type is **not known in advance**; `tool_choice: "auto"` sometimes lets Claude return conversational text instead of calling any tool, breaking downstream parsing | **`tool_choice: "any"` with all extraction tools defined** — guarantees *some* tool call happens (no plain-text escape), while letting Claude itself pick the correct tool/schema based on the document content it's reading in that same turn |
| Preliminary classification call, then a second call with tool_choice forced to the identified tool | ❌ Trap: works, but adds an unnecessary extra API round-trip (latency + cost) for something achievable in a single call with `"any"` |
| Consolidate all document types into one unified-schema tool and force that tool | ❌ Trap: defeats the reason for separate, type-specific schemas — a universal schema either bloats with irrelevant fields or loses per-type precision |
| Keep `tool_choice: "auto"` + system prompt instructions requiring tool use | ❌ Trap: the classic "guarantee → code, not prompts" failure — `auto` still permits a plain-text reply regardless of prompt wording, which is the exact bug already observed |

**Tips:** Recap of the full `tool_choice` decision table: `"auto"` = Claude may choose text or a tool (no guarantee); `"any"` = Claude must call **some** tool, but picks which one (use when the right tool depends on content Claude discovers **during** that same call, e.g. document type classification); `{"type": "tool", "name": "X"}` = must call exactly tool X (use when you already know which step must run); `"none"` = force plain text. When you don't know which of several tools is correct until Claude actually reads the input, `"any"` is the right guarantee — not a pre-classification step, not consolidating into one tool, and never a prompt-only instruction.

**Source check (official Exam Guide, Task Statement 4.3):**
- "tool_choice: 'any' forces the model to call one of the available tools without specifying which, appropriate when tool selection depends on content the model determines during generation (e.g., document type classification) and a guarantee of structured output is required without knowing the type in advance."

### Example question (Practice Test 2, Q74 — answered correctly)

**Q:** The extraction pipeline receives documents of varying types — some are invoices, others are contracts, and some are receipts. You've defined separate extraction tools, each with its own schema tailored to the document type. During testing, you observe that with tool_choice: "auto", Claude sometimes returns conversational text instead of calling an extraction tool, causing downstream parsing failures. You need guaranteed structured output without knowing the document type in advance. What's the most effective approach?

- ❌ Add a preliminary classification call, then make a second call with tool_choice forced to the identified extraction tool.
- ✅ **Set tool_choice: "any" with all extraction tools defined.**
- ❌ Consolidate all document types into a single unified-schema extraction tool and force that tool.
- ❌ Keep tool_choice: "auto" with system prompt instructions requiring tool use.

**Glossary (thuật ngữ):**
- *tool_choice: "any"* = buộc phải gọi một tool nào đó trong danh sách, nhưng để mô hình tự chọn tool nào phù hợp
- *tool_choice: "auto"* = mặc định, mô hình tự quyết định gọi tool hay trả lời bằng văn bản
- *forced tool selection* = ép buộc gọi đúng một tool cụ thể được chỉ định trước
- *preliminary classification call* = một lượt gọi API riêng chỉ để phân loại trước khi thực hiện lượt gọi chính
- *unified-schema tool* = một tool duy nhất có schema "gộp chung" cho nhiều loại tài liệu khác nhau (thường đánh đổi mất độ chính xác riêng của từng loại)

---

## Practice Test 2 — Q75: Retry-With-Feedback for Pydantic Type Errors (confirms 4.4 pattern again)

**Note:** Near-duplicate of Q71 and the existing "Retry only if the answer exists in the input" rule (Task Statement 4.4) — same lesson, different error type (a format/type mismatch: model returned a string range `"2 to 3"` where a float was expected).

**Q:** Monitoring shows 12% of extractions fail Pydantic validation with specific errors like "expected float for quantity, got '2 to 3'". Retrying these requests without modification produces failures. What's the most effective approach to recover from these validation failures?

- ✅ **Send a follow-up request including the validation error, asking the model to correct its output.**: retry with feedback — the prompt itself confirms blind retry ("without modification") fails, so the fix must include the specific error.
- ❌ Implement a secondary pipeline using a larger model tier to reprocess documents that fail validation: costly, no guarantee a bigger model avoids this exact format error.
- ❌ Set temperature to 0 to eliminate output variability and ensure consistent formatting: reduces randomness, doesn't teach the model the correct format — could be consistently wrong.
- ❌ Pre-process source documents to standardize problematic formats before sending them for extraction: wrong layer — the failure is in the model's output, not the source document's formatting.

---

## Practice Test 2 — Q76: Progressive Summarization Done Right for Long-Running Accumulating Conversations: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Long-running, accumulating conversation (weekly sessions over months, now 85K tokens); assistant gives generic answers instead of referencing the group's specific earlier conclusions; discussions build on prior sessions' conclusions | **Progressive summarization: replace older conversation blocks with concise summaries that explicitly extract key conclusions, decisions, and recurring themes; keep the most recent exchanges verbatim** |
| Add structured XML tags marking significant conclusions throughout the (still 85K-token) history | ❌ Trap: reorganizes the same amount of content but doesn't reduce it — the underlying context-degradation problem (and continued growth every week) remains |
| Rolling window truncation, keep only the most recent 25K tokens | ❌ Trap: outright deletes older conclusions the discussion explicitly needs to build on |
| Semantic embedding index + retrieve only relevant past exchanges, replacing linear format with retrieved segments | ❌ Trap: over-engineered; approximate retrieval risks losing the narrative thread of an ongoing discussion |

**Tips:** This contrasts with the "Trimming Verbose Tool Outputs" rule elsewhere in this cheat sheet, where replacing **precise structured data** (order fields, dates, amounts) with prose summaries is a trap because exactness is required. Here the content is **natural discussion**, not numeric data, so summarization is the right tool — but only if done well: **explicitly extract** the specific conclusions/decisions/themes (not a vague summary) and **keep recent exchanges verbatim** so the live discussion doesn't lose precision. Progressive summarization is "good" or "bad" depending on *what* is being summarized (precise structured values → bad; narrative discussion content, with explicit extraction of key points → good) and whether recent content stays exact.

**Source check (official Exam Guide, Task Statement 5.1 — Conversation context):**
- "Progressive summarization for long-running conversations: replacing older conversation blocks with concise summaries that explicitly extract key conclusions, decisions, and recurring themes, while keeping recent exchanges verbatim to preserve precision for ongoing context."

### Example question (Practice Test 2, Q76 — answered correctly)

**Q:** After three months of weekly sessions, your conversation history has grown to 85,000 tokens. When users ask "What did we conclude about the theme of isolation?", the assistant provides generic literary analysis rather than referencing the group's specific insights from earlier sessions. Discussions often build on previous meetings' conclusions, so maintaining narrative context is important. What's the most effective approach?

- ❌ Add structured XML tags to mark significant discussion conclusions throughout the conversation history.
- ❌ Implement rolling window truncation to keep only the most recent 25,000 tokens.
- ✅ **Implement progressive summarization where older conversation blocks are replaced with concise summaries that explicitly extract key conclusions, decisions, and recurring themes, keeping recent exchanges verbatim.**
- ❌ Use semantic embedding to index the full conversation history and retrieve only relevant past exchanges for each user query, replacing the linear conversation format with retrieved segments.

**Glossary (thuật ngữ):**
- *progressive summarization* = tóm tắt dần dần / lũy tiến (các phần cũ hơn của hội thoại được nén lại thành tóm tắt theo thời gian, càng cũ càng cô đọng)
- *explicitly extract key conclusions* = trích xuất rõ ràng các kết luận chính (khác với tóm tắt mơ hồ, chung chung — bản tóm tắt phải nêu cụ thể kết luận/quyết định/chủ đề)
- *keep recent exchanges verbatim* = giữ nguyên văn các trao đổi gần đây (không tóm tắt phần mới nhất, để giữ độ chính xác cho ngữ cảnh đang diễn ra)
- *rolling window truncation* = cắt cửa sổ trượt (chỉ giữ lại N token gần nhất, xóa hẳn phần cũ hơn)
- *semantic embedding retrieval* = truy xuất bằng embedding ngữ nghĩa (đánh chỉ mục toàn bộ lịch sử rồi chỉ lấy ra đoạn liên quan — có thể làm mất mạch tường thuật liên tục)

---

## Practice Test 2 — Q77: Stateless API, Missing Messages Array History (confirms existing 5.1 pattern)

**Note:** Direct duplicate of the existing "Stateless API: Passing Conversation History" rule (Task Statement 5.1) — same root cause, playlist-preferences framing.

**Q:** You're implementing a feature where users refine their playlist preferences through multiple conversation turns. After deploying, you notice Claude's responses don't reflect what was said earlier in the same conversation — for example, a user says they love jazz, but two messages later Claude asks what genres they enjoy. What is the most likely cause?

- ❌ The Claude API requires a session_id parameter that you haven't configured: no such parameter exists; the API doesn't hold conversation state server-side.
- ✅ **Your application isn't including prior messages in the messages array.**: the API is stateless — forgetting something said 2 messages ago means the history isn't being sent.
- ❌ The model's context window has been exceeded by the conversation length: a few-message conversation is nowhere near a 200K-token limit.
- ❌ Claude requires a vector database connection to maintain conversation memory: no such requirement — memory is just whatever is included in `messages`.

---

## Practice Test 2 — Q78: System Prompt Adherence Drift — Accumulated Responses Diluting Influence, Not "Attention Decay": Exam Rule of Thumb

| Situation | Best approach / correct diagnosis |
|---|---|
| System prompt defines a persona + specific guidelines (always ask about budget, suggest alternatives, confirm timeline); responses follow the guidelines for turns 1–4, but by turn 7 the assistant gives generic advice, skipping budget/timeline questions — and the **whole conversation is only 2,500 tokens** (too short to blame on context length) | **The assistant's own accumulated responses are diluting the system prompt's influence** — once a response drifts generic, that output becomes part of the context, and later turns increasingly pattern-match the assistant's own recent (already-drifted) outputs rather than the original system prompt — a self-reinforcing snowball, not a length/attention effect |
| "The model's attention on system prompt instructions naturally weakens as turns accumulate" | ❌ Trap: sounds intuitive, but frames the cause as an inevitable, content-independent decay unrelated to what's actually in the context — doesn't fit a conversation this short (no "lost in the middle" scenario at 2,500 tokens) and offers no actionable fix |
| "System prompts only establish initial behavior and don't persist across all turns" | ❌ Trap: factually wrong about the API — if the application correctly re-sends the system prompt every call, it is present in every turn |
| "The system prompt is only sent with the first API request" | ❌ Trap: misunderstands the stateless Messages API — every request must include the system prompt again; there's no "first request only" behavior |

**Tips:** When a question gives you a **short** conversation/token count and still shows guideline drift, that's a signal to rule out length-based explanations (context degradation, lost-in-the-middle, attention decay by distance) and instead look for a **compounding/self-reinforcing** mechanism: the assistant's own prior outputs, once they've drifted from instructions, become part of what the model conditions on going forward, and can increasingly outweigh the original system-level guidance. This is different from — and more actionable than — "attention naturally weakens with turn count," because it points to a fix (catch drift early, periodically reinforce key guidelines) rather than accepting an inevitable decay.

**Source check (official Exam Guide, Task Statement 5.1 — Conversation context):**
- 5.1 covers how the assistant's own accumulated conversational responses can progressively dilute or override system-prompt-level guidance in the context, as distinct from context-length-driven degradation (lost in the middle, attention span limits) — the former is a compounding, content-driven effect that can occur even in short conversations, while the latter requires substantial context length to manifest.

### Example question (Practice Test 2, Q78 — answered correctly)

**Q:** Your home renovation planning assistant uses a system prompt defining an expert contractor persona with specific guidelines: always ask about budget, suggest alternatives at multiple price points, and confirm timeline requirements. During testing, responses follow these guidelines for turns 1–4, but by turn 7, the assistant gives generic advice without asking about budget or timeline. The conversation totals only 2,500 tokens. What is the most likely cause?

- ❌ System prompts only establish initial behavior and don't persist across all turns.
- ❌ The system prompt is only sent with the first API request.
- ❌ The model's attention on system prompt instructions naturally weakens as turns accumulate.
- ✅ **The assistant's accumulated responses are diluting the system prompt's influence.**

**Glossary (thuật ngữ):**
- *system prompt persistence* = tính duy trì của system prompt (system prompt phải được gửi lại trong mọi request vì API stateless — không có khái niệm "chỉ áp dụng ở lần đầu")
- *attention decay* = suy giảm attention (một cách giải thích mang tính "tự nhiên, không kiểm soát được" — thường là bẫy nếu không khớp với độ dài ngữ cảnh thực tế)
- *diluting influence* = pha loãng ảnh hưởng (các phản hồi mới, đã trôi khỏi hướng dẫn ban đầu, dần lấn át vai trò định hướng của system prompt trong ngữ cảnh)
- *self-reinforcing drift / snowball effect* = hiệu ứng tự củng cố / lăn tuyết (một khi đã lệch hướng, các lượt sau càng dễ lệch thêm vì mô hình học theo chính output gần nhất của nó)
- *in-context learning from own outputs* = học trong ngữ cảnh từ chính output của mình (mô hình có xu hướng tiếp tục theo pattern nó vừa tạo ra trong cùng hội thoại)

---

## Practice Test 2 — Q79: Sliding Window Loses Old Content → Hybrid Summarize-Older/Keep-Recent-Verbatim (confirms Q76 pattern)

**Note:** Near-duplicate of Q76's "Progressive Summarization Done Right" rule (Task Statement 5.1) — same fix, framed as replacing a lossy sliding window.

**Q:** Users report that during extended conversations, the AI loses track of specific topics, examples, and preferences they mentioned earlier in the session. Your current implementation uses a sliding window that keeps only the most recent 25 message pairs to stay within context limits. What's the most effective approach to maintain awareness of earlier conversation content while managing context size?

- ❌ Implement vector similarity search over the full conversation history, retrieving relevant past messages for each user query: over-engineered; approximate retrieval fragments the linear conversation flow.
- ❌ Increase the window size to 50 message pairs to retain more conversation history before truncation: just delays the same failure to a later point.
- ❌ Add a separate API call each turn to summarize messages being dropped, prepending this running summary to the conversation: unnecessary per-turn API overhead vs. summarizing as part of the architecture only when needed.
- ✅ **Replace the sliding window with a hybrid approach: summarize older messages while keeping recent messages verbatim.**: matches the established pattern — reduce size without losing older content, keep recent content exact.

---

## Practice Test 2 — Q80: The Fix for Guideline Drift — Periodic User-Role Reinforcement (directly pairs with Q78)

**Note:** This is the actionable "fix" half of Q78's diagnosis. Q78 established that guideline drift over turns is caused by the assistant's own accumulated responses diluting the system prompt's influence (a compounding effect, not context-length exhaustion). This question confirms that reading explicitly: conversation length is well within context limits (30K of 200K tokens), ruling out capacity-based explanations, and asks for the correct intervention.

| Situation | Best approach |
|---|---|
| Claude follows system prompt guidelines consistently for the first 10–15 turns, but by turns 25–30 responses deviate (informal tone when formality was specified, skipped required formatting, restricted info types appearing) — conversation is well within context limits (30K/200K tokens), so this is guideline-adherence drift, not context capacity | **Insert user-role messages that reinforce critical guidelines at natural conversation breakpoints, especially before complex requests** — proactively counters the progressive dilution identified in Q78 |
| Move behavioral guidelines from the system prompt into the first user message | ❌ Trap: still a one-time, front-loaded instruction (same position problem as the system prompt); doesn't address turn-based drift, and may even weaken adherence since system prompts typically carry special instruction-following priority |
| Implement post-response validation that regenerates each response until it conforms to guidelines | ❌ Trap: treats the symptom after it occurs, with added cost/latency from repeated regeneration; doesn't prevent the drift from starting, and can be unreliable once the model's own in-context sense of "correct" has drifted |
| Automatically start a new conversation after 20 turns, passing a summary of the prior context | ❌ Trap: solves a different problem (context capacity), explicitly ruled out by the question (only 30K/200K tokens used); disrupts continuity/UX for a problem that isn't about running out of context |

**Tips:** This pairs directly with Q78: if the diagnosis is "the assistant's own accumulated responses are diluting the system prompt's influence" (a compounding, content-driven effect), the fix is **periodic reinforcement of the critical guidelines via user-role messages at natural breakpoints** — not moving the instructions elsewhere (still front-loaded, still one-time), not regenerating after the fact (reactive, costly), and not resetting the session (solves the wrong problem when context capacity isn't the issue).

**Source check (official Exam Guide, Task Statement 5.1 — Conversation context):**
- 5.1 covers reinforcing critical behavioral guidelines via periodic user-role messages at natural conversation breakpoints (especially before complex requests) as the correct countermeasure to system-prompt-influence dilution during extended conversations — distinct from context-capacity fixes (session reset, summarization for length) which address a different failure mode.

### Example question (Practice Test 2, Q80 — answered correctly)

**Q:** During QA testing, you notice that Claude follows your system prompt guidelines consistently in the first 10–15 turns, but by turn 25–30, responses begin deviating — using informal tone when formality was specified, occasionally skipping required formatting, or providing information types the guidelines restrict. Conversation length is well within context limits (typically 30,000 tokens out of 200,000 available). What's the most effective approach to maintain consistent behavior throughout extended conversations?

- ❌ Move behavioral guidelines from the system prompt into the first user message.
- ❌ Implement post-response validation that regenerates each response until it conforms to the specified guidelines.
- ✅ **Insert user-role messages that reinforce critical guidelines at natural conversation breakpoints, especially before complex requests.**
- ❌ Automatically start a new conversation after 20 turns, passing a summary of the prior context to maintain continuity.

**Glossary (thuật ngữ):**
- *guideline drift* = sự trôi dần khỏi hướng dẫn (mức độ tuân thủ guideline giảm dần theo số turn, dù chưa hết context)
- *natural conversation breakpoints* = điểm ngắt tự nhiên của hội thoại (thời điểm chuyển chủ đề, trước yêu cầu phức tạp — nơi thích hợp để chèn lại nhắc nhở)
- *reinforcement message* = tin nhắn củng cố (chèn thêm để nhắc lại các quy tắc/guideline quan trọng, thường ở vai "user")
- *post-response validation/regeneration* = kiểm tra và tạo lại phản hồi (chạy lại nhiều lần cho đến khi output đạt guideline — tốn chi phí, xử lý triệu chứng chứ không phải nguyên nhân)
- *context capacity vs. adherence drift* = dung lượng ngữ cảnh khác với trôi tuân thủ (hai vấn đề khác nhau — dùng sai giải pháp của vấn đề này cho vấn đề kia là bẫy thường gặp)

---

## Practice Test 2 — Q81: Injecting External Webhook Events into an Ongoing Conversation via the System Prompt: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| An external system pushes a real-time event (webhook: package shipped) mid-conversation; the user is actively chatting and will likely follow up soon; you want the next response to naturally reflect the new state, without fabricating a conversational turn | **Add the current status to the system prompt before the next API call** — ambient/environmental state belongs in the system prompt, not attributed to either conversational party; the next real user turn naturally has access to it |
| Append the status update as a prefix to the next user message before calling the API | ❌ Trap: misattributes system-originated data as something the *user* said — wrong role semantics |
| Immediately send an API request with the update as a synthetic user message, generating an unsolicited assistant response | ❌ Trap: produces a reply nobody asked for, which can collide with the user's real in-flight message (race condition) and breaks natural conversational flow |
| Configure the assistant to call a get_order_status tool at the start of every response | ❌ Trap: wasteful — adds a tool call (latency/cost) on every single turn regardless of whether anything changed, instead of updating only when a real event occurs |

**Tips:** When an external event needs to inform Claude's next response without being something anyone "said," the system prompt is the right injection point — it represents ambient context/state, distinct from user and assistant turns. Avoid two temptations: (1) smuggling system-originated facts into a user-role message (misattributes authorship), and (2) proactively firing an assistant response the user didn't request (disrupts the natural turn-taking flow and risks racing the user's actual next message). Also avoid solving a one-off event with an always-on tool call every turn — that trades a rare, event-driven update for constant unnecessary overhead.

**Source check (official Exam Guide, Task Statement 5.1 — Conversation context):**
- Covers injecting external/environmental state updates (e.g., webhook-driven status changes) into the system prompt ahead of the next API call, so the assistant naturally incorporates current state into its next real response — as distinct from fabricating synthetic conversational turns or triggering unsolicited responses.

### Example question (Practice Test 2, Q81 — answered correctly)

**Q:** During a conversation about order tracking, your external system receives a webhook indicating the user's package has shipped. The user is actively chatting and will likely send a follow-up message soon. You want the assistant to naturally incorporate this status change in its next response. What's the most effective approach?

- ❌ Append the status update as a prefix to the next user message before calling the API.
- ❌ Immediately send an API request with the update as a synthetic user message, generating an unsolicited assistant response.
- ✅ **Add the current shipping status to the system prompt before the next API call.**
- ❌ Configure the assistant to call a get_order_status tool at the start of every response.

**Glossary (thuật ngữ):**
- *webhook* = webhook (cơ chế hệ thống bên ngoài chủ động gửi thông báo sự kiện đến ứng dụng của bạn ngay khi có thay đổi, thay vì phải hỏi liên tục)
- *ambient/environmental state* = trạng thái nền/môi trường (thông tin về thế giới bên ngoài mà Claude cần biết, không phải điều ai đó "nói" trong hội thoại)
- *synthetic user message* = tin nhắn user giả lập (tạo ra một tin nhắn "như thể" user gửi, dù thực tế không phải)
- *unsolicited assistant response* = phản hồi không được yêu cầu (assistant tự tạo ra câu trả lời mà không ai chủ động hỏi ở lượt đó)
- *race condition* = tình huống tranh chấp thời điểm (phản hồi tự động có thể chen ngang đúng lúc user đang gửi tin nhắn thật, gây xung đột thứ tự)

---

## Practice Test 2 — Q82: Periodic Reminder Injection for Proficiency-Adaptation Drift (⚠️ self-reasoned, no answer key shown — near-duplicate of Q78/Q80)

**Note:** No answer tag was provided for this question (pasted as plain text, no screenshot with Đúng/Sai). This is my own reasoning applying the confirmed Q78/Q80 pattern, not a verified platform key.

| Situation | Best approach |
|---|---|
| System prompt has verbose proficiency-adaptation guidelines; they're followed correctly through the first ~12 turns, but in conversations exceeding 12 turns (~4,000 tokens of history) the assistant increasingly defaults to intermediate-level explanations regardless of stated level — a turn-based drift, since the guidelines demonstrably work early on (not an ambiguity/clarity problem) | **Inject a condensed reminder of the proficiency requirements periodically (every 4–5 turns)** — matches the confirmed Q78/Q80 fix for progressive dilution of system-level guidance over turns |
| Separate API call after each response to evaluate difficulty match, regenerating misaligned responses | ❌ Trap: reactive, post-hoc, costly on every turn — the same "post-response validation/regeneration" trap ruled out in Q80 |
| Replace verbose guidelines with few-shot examples demonstrating level-specific differences | ❌ Trap: addresses instruction *clarity*, but the guidelines already work correctly for the first 12 turns — the cause is dilution over time, not ambiguity; a reformatted but still one-time, front-loaded instruction still grows more distant as turns accumulate |
| Restructure the system prompt to place the rules in a final section immediately before conversation history begins | ❌ Trap: still a single, one-time placement; the same growing-distance-over-turns problem recurs as the conversation lengthens |

**Tips:** The tell here is that the problem is explicitly turn-count-triggered (fine through 12 turns, degrading past that point), not present from turn 1 — this rules out "the instructions are unclear/verbose" as the cause (ruling out the few-shot fix) and points to the same compounding dilution mechanism from Q78. Any fix that's still a single placement in the system prompt (front section, final section, few-shot or prose) suffers the same fate: it grows more distant from the model's generation point as the conversation grows. Only a **periodic** re-injection counters this by repeatedly refreshing the guidance's presence and weight throughout the extended conversation.

**Source check (official Exam Guide, Task Statement 5.1 — Conversation context):**
- Same principle as Q78/Q80: periodic reinforcement of critical behavioral guidelines throughout an extended conversation counters progressive dilution of system-prompt-level instructions, as distinct from clarity fixes (few-shot, rewording) or one-time repositioning, which don't address the turn-based compounding nature of the drift.

### Example question (Practice Test 2, Q82 — answered by reasoning, unconfirmed)

**Q:** Your conversational AI tutor has a 2,800-token system prompt containing teaching methodology, persona guidelines, and detailed written instructions for adapting explanations to different proficiency levels. User testing reveals that in conversations exceeding 12 turns (approximately 4,000 tokens of conversation history), the assistant increasingly ignores the proficiency-adaptation guidelines, defaulting to intermediate-level explanations regardless of the learner's stated level. What's the most effective approach to ensure consistent adherence to these guidelines throughout extended conversations?

- ❌ After each assistant response, make a separate API call to evaluate whether the difficulty level matched the learner's profile, regenerating responses that don't align.
- ❌ Replace the verbose proficiency guidelines with few-shot examples demonstrating appropriate responses at each proficiency level, showing concrete differences in vocabulary, complexity, and explanation depth.
- ❌ Restructure the system prompt to place the proficiency-adaptation rules in a clearly-marked final section immediately before the conversation history begins.
- ✅ **Inject a condensed reminder of the proficiency requirements into the conversation as a system message every 4-5 turns.** *(my reasoned answer — not confirmed by a shown key)*

**Glossary (thuật ngữ):**
- *proficiency-adaptation guidelines* = hướng dẫn điều chỉnh theo trình độ (quy tắc để assistant thay đổi độ khó/ngôn ngữ giải thích theo trình độ người học)
- *condensed reminder* = lời nhắc cô đọng (bản rút gọn của guideline quan trọng, đủ ngắn để chèn lại định kỳ mà không tốn nhiều token)
- *turn-based drift* = trôi dần theo số turn (khác với lỗi do hướng dẫn không rõ ràng — ở đây hướng dẫn đã đúng ngay từ đầu, chỉ suy yếu dần theo thời gian)
- *one-time front-loaded instruction* = hướng dẫn đặt một lần duy nhất ở đầu (dù đặt ở đầu hay cuối system prompt, nó vẫn chỉ xuất hiện một lần, nên càng hội thoại dài càng "xa" điểm sinh câu trả lời)

---

## Practice Test 2 — Q85: Stratified Random Sampling for High-Confidence Errors (confirms existing 5.5 monitoring pattern)

**Note:** Direct application of the existing "Stratified random sampling for measuring error rates… and detecting novel error patterns" quote already in this cheat sheet (Task Statement 5.5), here specifically for errors *within* the high-confidence group where the confidence signal itself has already failed.

| Situation | Best approach |
|---|---|
| System routes <85% confidence to human review; a quarterly audit finds 12% of >85%-confidence extractions still contain "plausible-but-incorrect" errors from varied sources (comparison tables, appendix confusion, ambiguous phrasing); need to both catch these errors sustainably and measure whether improvements reduce the error rate over time | **Stratified random sampling reviewing a fixed % of high-confidence extractions weekly** — gives a trackable error-rate metric over time *and* surfaces unknown/novel error patterns, since the sample isn't restricted to already-identified error sources |
| Lower the confidence threshold from 85% to 70% | ❌ Trap: doesn't fix the root issue — these errors already occur *above* 85%, so confidence isn't predictive for this failure type; just routes more untargeted volume to review |
| Add a verification pass that re-extracts and flags disagreement between two attempts | ❌ Trap: misses *systematic* misinterpretation — if the model consistently misreads the same ambiguous phrasing or table, both extraction attempts likely produce the same wrong answer, so disagreement-based flagging catches nothing |
| Heuristic rules flagging documents with comparison tables/appendices regardless of confidence | ❌ Trap: only covers currently-known error sources; doesn't generalize to future/unknown error patterns (reactive, non-generalizing patch) |

**Tips:** When the population needing review is already *above* the confidence threshold (i.e., the confidence signal has already been "spent" and failed for this group), you can't stratify further by confidence — you need **random sampling with fixed periodic cadence** to (1) get an unbiased, trackable error-rate estimate over time and (2) catch error types you don't yet know to look for. Contrast with the "Allocating Limited Human Review" rule elsewhere in this cheat sheet, where confidence-calibrated thresholds are right for *targeting* review capacity among a population where confidence still carries signal — stratified random sampling is for *monitoring* a population where it doesn't.

**Source check (official Exam Guide, Task Statement 5.5 — Human review & calibration):**
- "Stratified random sampling for measuring error rates over time and detecting novel error patterns among extractions the confidence-based routing has already classified as high-confidence."

### Example question (Practice Test 2, Q85 — answered correctly)

**Q:** The system routes documents with extraction confidence below 85% to human review. A quarterly audit reveals that 12% of high-confidence extractions (>85%) also contain errors — cases where the model finds plausible-but-incorrect values. Error sources vary: comparison tables showing competitor specs, appendices referencing different product variants, and ambiguous phrasing the model misinterprets. You need a sustainable strategy to catch these high-confidence errors and measure whether improvements reduce the error rate over time. What approach is most effective?

- ✅ **Implement stratified random sampling reviewing a fixed percentage of high-confidence extractions weekly, enabling error rate measurement and novel pattern detection.**
- ❌ Lower the confidence threshold from 85% to 70%, routing a larger volume of extractions to human review.
- ❌ Add a verification pass that re-extracts from each high-confidence document, flagging cases where the two extraction attempts produce different results.
- ❌ Implement heuristic rules that flag documents containing comparison tables or appendices for review regardless of confidence score.

**Glossary (thuật ngữ):**
- *stratified random sampling* = lấy mẫu ngẫu nhiên theo tầng (chia tổng thể thành các nhóm/tầng rồi lấy mẫu ngẫu nhiên trong từng nhóm, đảm bảo đại diện đều)
- *plausible-but-incorrect values* = giá trị hợp lý nhưng sai (kết quả trông có vẻ đúng, hợp logic, nhưng thực chất không chính xác — khó phát hiện hơn lỗi rõ ràng)
- *novel error pattern* = dạng lỗi mới chưa từng biết đến (chưa được liệt kê trong danh sách nguyên nhân lỗi hiện tại)
- *systematic misinterpretation* = hiểu sai một cách hệ thống (mô hình luôn hiểu sai theo cùng một cách với cùng loại nội dung mơ hồ — không phải lỗi ngẫu nhiên)
- *confidence signal exhausted* = tín hiệu confidence đã "hết tác dụng" (điểm tin cậy không còn phản ánh đúng khả năng sai của nhóm dữ liệu này nữa)

---

## Practice Test 2 — Q86: Structured Fact Database for Precision-Dependent Questions Across Papers (confirms structured-layer pattern)

**Note:** Direct variant of the existing "structured case-facts / separate context layer" rule (Q57 pattern) and the cheat sheet principle "structured data beats text" (Task Statement 5.1/5.6) — here applied to a research assistant discussing multiple academic papers over an extended conversation.

| Situation | Best approach |
|---|---|
| Research assistant discusses academic papers; conversations exceed 60K tokens; current approach summarizes paper discussions after 8 turns to fit context; users later ask precision-dependent follow-ups (exact sample sizes, p-values, inclusion criteria) about papers discussed earlier, and answers come back hedged or inaccurate | **Maintain a structured database of key facts extracted from each paper (sample sizes, statistics, methods); retrieve relevant entries into context when a precision-dependent question is detected** — scales across many papers and guarantees exact values via structure, not prose |
| Retrieval that re-injects relevant raw paper sections when the question suggests numerical need | ❌ Trap: detecting "needs numbers" to trigger retrieval is a fuzzy classification step, and raw text still requires the model to re-derive the exact value each time rather than handing over an already-precise fact |
| Separate Claude call with explicit instructions to generate "higher-fidelity" summaries preserving all numerical details | ❌ Trap: still narrative summarization — no structural guarantee that precise values survive condensation, even with better instructions (same risk already flagged: condensing numbers into any summary format risks vagueness) |
| Keep methodology/results source text in context permanently, summarizing only discussion/interpretation | ❌ Trap: doesn't scale — across many papers in an extended conversation, permanently retaining full raw sections reproduces the exact context-growth problem being solved |

**Tips:** Whenever a question needs **exact, precise values** (numbers, dates, statistics, IDs) that must survive across a long, evolving conversation, the answer is almost always: **extract those specific facts into a structured store once, separate from the summarization/narrative flow**, and retrieve only the relevant structured entries on demand. Prose summarization — no matter how carefully instructed — cannot structurally guarantee precision is preserved, and permanently keeping raw source text doesn't scale as the number of documents/topics grows.

**Source check (official Exam Guide, Task Statement 5.1 — Conversation context, related to 5.6 provenance):**
- Parallels the Task Statement 1.4/5.1 principle behind structured case-facts (Q57): "Extracting and persisting structured key-fact data (e.g., sample sizes, statistical values, methodology details) into a separate, retrievable context layer, rather than relying on narrative summarization, to preserve precision for future precision-dependent queries across an extended conversation."

### Example question (Practice Test 2, Q86 — answered correctly)

**Q:** Your research assistant helps users analyze academic papers over extended conversations. User testing reveals a recurring issue: after conversations exceed 60K tokens, users ask follow-up questions requiring precise numerical details from papers discussed earlier — sample sizes, exact p-values, specific inclusion criteria. Your current approach summarizes paper discussions after 8 turns to stay within context limits. Users report that responses to these precision-dependent questions are often hedged or inaccurate. What's the most effective architectural change?

- ❌ Implement retrieval that re-injects relevant paper sections when the user's question suggests they need specific numerical details.
- ❌ Use a separate Claude call with explicit instructions to generate higher-fidelity summaries that preserve all numerical details and statistical values.
- ❌ Keep source text from methodology and results sections in context permanently, while summarizing only the conversational discussion and interpretation portions.
- ✅ **Maintain a structured database of key facts extracted from each paper (sample sizes, statistics, methods) and retrieve relevant entries into context when precision-dependent questions are detected.**

**Glossary (thuật ngữ):**
- *precision-dependent question* = câu hỏi đòi hỏi độ chính xác tuyệt đối (cần con số/giá trị chính xác, không chấp nhận trả lời mơ hồ hoặc ước lượng)
- *structured fact database* = cơ sở dữ liệu sự kiện có cấu trúc (lưu các giá trị quan trọng dưới dạng trường dữ liệu rõ ràng, tách biệt khỏi văn bản tóm tắt)
- *higher-fidelity summary* = tóm tắt "độ trung thực cao hơn" (vẫn là tóm tắt dạng văn xuôi, chỉ được yêu cầu giữ chi tiết hơn — không có đảm bảo cấu trúc)
- *retrieve relevant entries* = truy xuất đúng mục cần thiết (chỉ lấy đúng phần dữ liệu liên quan đến câu hỏi hiện tại, không tải toàn bộ)
- *scale across documents* = mở rộng qua nhiều tài liệu (giải pháp phải hoạt động tốt dù số lượng paper/chủ đề tăng lên theo thời gian)

---

## Practice Test 2 — Q87: Plan Mode for Broad, Multi-File Breaking-Change Migrations (contrasts with Q68's direct-execution case)

**Note:** This pairs directly with Q68 ("Plan Mode vs. Direct Execution for Production Bugs"), but sits on the opposite side of that rule: there, a narrow, high-signal production bug (clear stack trace) called for direct execution; here, a broad, multi-file, multi-module change with several *interacting* breaking changes calls for plan mode first.

| Situation | Best approach |
|---|---|
| Security audit requires a major version migration (v2→v3) of a library with multiple breaking changes that interact (callback→Promise changes control flow at every call site, a restructured type cascades through consumers, three methods removed need replacement logic); the library is imported in 45 files across several modules — scope and impact are not yet understood | **Enter plan mode: explore library usage across modules, map affected code paths, then create a migration strategy before implementing** |
| Paste the migration guide's breaking changes into the prompt and use direct execution to update all 45 files | ❌ Trap: treats interacting breaking changes as a simple mechanical find/replace; jumping to execution across many files without first understanding usage patterns risks incomplete or inconsistent fixes |
| Create a custom slash command encapsulating the migration transformations, then execute it against each file without prior codebase exploration | ❌ Trap: automates the transformation before understanding how the library is actually used — premature automation without exploration, similar to jumping to a fixed plan before knowing the real usage patterns |
| Update the dependency version, run the test suite, and use Claude Code to fix each failure as it appears | ❌ Trap: reactive, test-driven whack-a-mole; doesn't systematically map the interacting breaking changes upfront and can miss issues the test suite doesn't cover |

**Tips:** Contrast with Q68: the deciding factor for plan mode vs. direct execution is **task clarity/complexity and blast radius**, not a fixed rule of thumb like "bugs = direct, migrations = plan." A narrow, well-scoped problem with a clear signal (a stack trace pointing at one spot) favors direct execution even in an unfamiliar module. A broad change touching many files across modules, with multiple breaking changes that interact with each other and an unknown usage footprint, favors plan mode: explore first (map where and how the library is used), then design a migration strategy, then implement — because acting first here risks compounding inconsistent fixes across 45 files before you understand the full scope.

**Source check (official Exam Guide, Task Statement 3.4 — Plan mode vs. direct execution):**
- 3.4 covers choosing plan mode for broad, multi-file changes with significant blast radius and interacting breaking changes — mapping usage and affected code paths before implementing — as opposed to direct execution, which fits narrow, well-scoped, high-signal tasks (see Q68).

### Example question (Practice Test 2, Q87 — answered correctly)

**Q:** A security audit requires updating your authentication library from v2 to v3. The migration guide documents breaking changes: authenticate() now returns a Promise instead of accepting a callback, the User type has restructured fields, and three deprecated methods were removed. Grep shows the library is imported in 45 files across several modules. What's the most effective approach?

- ❌ Paste the migration guide's breaking changes into your prompt and use direct execution to update all usages across the 45 files.
- ✅ **Enter plan mode to explore library usage across modules, map affected code paths, then create a migration strategy before implementing.**
- ❌ Create a custom slash command encapsulating the migration transformations, then execute it against each file without prior codebase exploration.
- ❌ Update the dependency version, run the test suite, and use Claude Code to fix each failure as it appears.

**Glossary (thuật ngữ):**
- *breaking change* = thay đổi phá vỡ tương thích (thay đổi API khiến code cũ không còn hoạt động đúng nếu không sửa)
- *interacting breaking changes* = các thay đổi phá vỡ tương thích có tương tác lẫn nhau (một thay đổi ảnh hưởng đến cách xử lý thay đổi khác, ví dụ callback→Promise làm thay đổi luồng điều khiển tại mọi nơi gọi hàm)
- *blast radius* = phạm vi ảnh hưởng (số lượng file/module bị tác động bởi một thay đổi)
- *migration strategy* = chiến lược di trú (kế hoạch có hệ thống để chuyển đổi codebase sang phiên bản/API mới)
- *premature automation* = tự động hóa quá sớm (viết script/lệnh tự động biến đổi code trước khi hiểu rõ cách code hiện tại đang sử dụng thư viện)

---

## Practice Test 2 — Q88: Iterative Refinement for Interacting Formatting Issues (first Task Statement 3.5 example)

**Note:** First question in this cheat sheet testing Task Statement 3.5 (Iterative refinement), previously flagged as untested.

| Situation | Best approach |
|---|---|
| PDF report generation works correctly for the database query, but has three formatting issues that **interact**: table columns too narrow (truncation), dates not properly formatted, page breaks incorrect — changing column widths affects date rendering, and page breaks depend on content height | **Fix issues in dependency order, one at a time, verifying (testing) after each change**: column width first (with specific measurements) → verify → fix date formatting within the corrected columns → verify → adjust page breaks — testing after each step |
| Show Claude an example of a correctly formatted report and ask it to match that output, rather than listing specific technical issues | ❌ Trap: an end-result example doesn't convey the specific technical causes (exact measurements, page-break logic) or the correct fix order when issues interact; Claude may mimic surface appearance without fixing the underlying mechanism |
| Provide all three issues in a single detailed message with exact specifications, letting Claude address them together in one update | ❌ Trap: since the issues interact, fixing them all at once makes it very hard to attribute a new problem to the specific change that caused it — loses step-by-step verifiability |
| Start fresh with a detailed prompt specifying all formatting requirements upfront | ❌ Trap: discards working progress (correct DB querying) and assumes the interactions between issues can be fully anticipated before actually observing them in practice |

**Tips:** When multiple issues are **interdependent** (fixing one changes how another manifests), the effective iteration pattern is: **fix in dependency order, one change at a time, verify/test after each step**, rather than batching all fixes into one large change or trying to specify everything correctly upfront. This mirrors the general principle that a fixed, comprehensive plan can't anticipate consequences you can only discover by observing the system after each incremental change — especially true when changes causally affect each other.

**Source check (official Exam Guide, Task Statement 3.5 — Iterative refinement):**
- 3.5 covers iterating toward a working solution by addressing interdependent issues in dependency order with verification after each step, rather than batching all changes together or attempting to fully specify a complex, interacting set of requirements in a single upfront pass.

### Example question (Practice Test 2, Q88 — answered correctly)

**Q:** You've asked Claude Code to build a PDF report generation feature. The initial implementation queries the database correctly, but the output has formatting issues: table columns are too narrow causing content truncation, dates display without proper formatting, and page break handling is incorrect. You've noticed these issues interact — changing column widths affects how dates render, and page breaks depend on content height. What's the most effective approach for iterating toward a working solution?

- ✅ **Address the column width issue first with specific measurements, verify it works, then fix date formatting within the corrected columns, then adjust page breaks — testing after each change.**
- ❌ Show Claude an example of a correctly formatted report and ask it to match that output, rather than listing the specific technical issues.
- ❌ Provide all three issues in a single detailed message with exact specifications for each, allowing Claude to address them together in one update.
- ❌ Start fresh with a detailed prompt specifying all formatting requirements upfront.

**Glossary (thuật ngữ):**
- *interacting issues* = các vấn đề có tương tác lẫn nhau (sửa vấn đề này làm thay đổi cách vấn đề khác biểu hiện)
- *dependency order* = thứ tự phụ thuộc (thứ tự sửa lỗi hợp lý dựa trên việc vấn đề nào ảnh hưởng đến vấn đề nào)
- *iterative refinement* = tinh chỉnh lặp lại (sửa từng phần nhỏ, kiểm tra, rồi mới tiếp tục — thay vì làm tất cả cùng lúc)
- *batching changes* = gộp nhiều thay đổi làm một lần (áp dụng nhiều fix cùng lúc — rủi ro khi các fix có tương tác, khó xác định nguyên nhân nếu có lỗi mới)
- *content truncation* = nội dung bị cắt xén (do cột quá hẹp, dữ liệu không hiển thị đầy đủ)

---

## Practice Test 2 — Q89: @references for One-Off Pattern-Following Tasks vs. CLAUDE.md (⚠️ self-reasoned, no answer key shown)

**Note:** No "Đúng/Sai" tag was visible on this question's screenshot (only a selected radio button, no color-coded verdict), so this answer is my own reasoning, not a confirmed platform key.

| Situation | Best approach |
|---|---|
| A one-off task must follow existing code patterns (db transactions, error handling, audit logging); you've already identified the exact 3 files that exemplify these patterns; patterns are already well-documented elsewhere (team wiki) and don't need project-level documentation | **Use @references to include the three modules directly in the prompt, giving Claude concrete code examples of the patterns to follow** |
| Describe the patterns in natural language in the prompt | ❌ Trap: less precise than actual code — prose descriptions can miss specific conventions that concrete code makes explicit |
| Ask Claude to explore the codebase to find and understand the patterns before generating the new module | ❌ Trap: wasted effort — you already know exactly which files exemplify the pattern, so exploration is unnecessary |
| Add documentation of each pattern to CLAUDE.md, establishing them as automatic project conventions | ❌ Trap: explicitly ruled out by the question itself — this is a one-off task and the patterns are already documented in the team wiki; CLAUDE.md is for persistent, project-wide conventions applied across many future tasks, not single-use integration tasks |

**Tips:** When you already know precisely which existing files exemplify a pattern, and the task is a one-off (not a recurring project convention), the most direct and precise approach is to reference those files directly (@references) so Claude sees concrete, exact code — not a prose description (imprecise) and not a fresh exploration (unnecessary, since you already know the answer). Reserve CLAUDE.md for conventions that should apply automatically and persistently across many future tasks — the question's own wording ("one-off," "don't need additional project-level documentation") is a direct signal to rule it out.

**Source check (official Exam Guide, Task Statement 3.1 — CLAUDE.md hierarchy, contrasted with direct file references):**
- 3.1 covers when project-level persistent documentation (CLAUDE.md) is appropriate (recurring conventions applied automatically across tasks) versus when direct, in-prompt file references are more effective for one-off tasks where the exact source-of-truth files are already known.

### Example question (Practice Test 2, Q89 — answered by reasoning, unconfirmed)

**Q:** You're implementing a new payment processing module that must follow your project's established patterns for database transactions, error handling, and audit logging. You've identified three existing modules that exemplify these patterns: db_utils.py, error_handlers.py, and audit_logger.py. This is a one-off integration task — these patterns are well-documented in your team wiki and don't need additional project-level documentation. What's the most effective approach?

- ❌ Describe the patterns from the three modules in natural language in your prompt, explaining the transaction handling approach, error format, and logging conventions Claude should follow.
- ❌ Ask Claude to explore your codebase to find and understand the transaction, error handling, and logging patterns before generating the new module.
- ✅ **Use @references to include the three modules directly in your prompt, giving Claude concrete code examples of the patterns to follow.** *(my reasoned answer — not confirmed by a shown key)*
- ❌ Add documentation of each pattern to your CLAUDE.md file, establishing them as project conventions that Claude will apply automatically.

**Glossary (thuật ngữ):**
- *@references* = cú pháp tham chiếu file trực tiếp trong Claude Code (đưa nguyên văn nội dung file vào prompt bằng cách gõ @tên_file)
- *one-off task* = tác vụ chỉ làm một lần (không lặp lại, không cần thiết lập quy ước lâu dài cho dự án)
- *project-level convention* = quy ước ở cấp dự án (áp dụng tự động, lâu dài cho nhiều task trong tương lai — phù hợp với CLAUDE.md)
- *concrete code example* = ví dụ code cụ thể (nguyên văn, chính xác — khác với mô tả bằng lời có thể bỏ sót chi tiết)

---

## Practice Test 2 — Q90: Selective @imports in CLAUDE.md for Monorepo Shared Standards (answered correctly — confirmed)

| Situation | Best approach |
|---|---|
| Monorepo has multiple shared standards docs, each applicable to only some packages (by domain characteristics: handles user data, is API-facing, etc.); packages have no naming convention indicating applicability; maintainers know their own package's domain requirements; current CLAUDE.md files duplicate ALL standards into every package, most of it irrelevant | **Use @imports in each package's CLAUDE.md to reference only the specific standard files relevant to that package, based on the maintainer's own domain knowledge** |
| Put all standards in the root CLAUDE.md with override instructions like "ignore security-rules.md when working in packages that don't handle user data" | ❌ Trap: fragile — relies on the model correctly applying conditional "ignore X unless Y" negative instructions across many packages; doesn't reduce irrelevant content, just asks the model to filter it at read time |
| Create a shared-standards file that uses @imports to combine all three standards, then have each package's CLAUDE.md import that combined file | ❌ Trap: solves the duplication problem but not the irrelevance problem — every package still pulls in every standard regardless of whether it actually applies |
| Create central claude/rules/ files for each standard with YAML frontmatter paths listing every package directory where that standard applies | ❌ Trap: over-engineered — centralizes and duplicates the domain knowledge that maintainers already have, creating a central file that must be updated every time a package's data-handling/API-facing status changes; more to maintain, not less |

**Tips:** When different packages need different subsets of shared standards, and the people who know which subset applies are the individual package maintainers (not a central config or naming convention), let @imports be scoped per-package and decided locally by each maintainer. This avoids both duplication (each package's CLAUDE.md imports directly, no copy-pasted content) and irrelevance (only the applicable standards are pulled in) — without a fragile central override mechanism or a central mapping file that has to track every package's characteristics and be kept in sync as packages evolve.

**Source check (official Exam Guide, Task Statement 3.1 — CLAUDE.md hierarchy & composition):**
- 3.1 covers composing CLAUDE.md content via @imports for selective inclusion of shared documentation, as opposed to a monolithic root file with conditional exceptions or a centrally-maintained applicability map.

### Example question (Practice Test 2, Q90 — answered correctly, confirmed)

**Q:** Your monorepo contains shared coding standards in /docs/standards/security-rules.md (for services handling user data), testing-patterns.md (for all packages), and api-conventions.md (for API-facing services). Your 15 packages are organized by feature domain (/packages/auth/, /packages/billing/, /packages/notifications/, etc.) without naming conventions indicating which handle user data or expose APIs. Package maintainers are expected to configure their own local development settings, as they understand their package's domain requirements. Currently, all package CLAUDE.md files duplicate all three standards, applying irrelevant guidance. What's the most effective approach?

- ❌ Put all standards in the root CLAUDE.md with override instructions like "ignore security-rules.md when working in packages that don't handle user data."
- ❌ Create a shared-standards.md that uses @imports to combine all three standards, then have each package's CLAUDE.md import that combined file.
- ❌ Create claude/rules/ files for each standard with YAML frontmatter paths listing every package directory where that standard should apply.
- ✅ **Use @imports in each package's CLAUDE.md to reference only the specific standard files relevant to that package, based on the maintainer's domain knowledge.**

**Glossary (thuật ngữ):**
- *@imports* = cú pháp nhúng/tham chiếu file khác vào trong CLAUDE.md (cho phép ghép nhiều nguồn tài liệu mà không copy-paste)
- *monorepo* = một repository chứa nhiều package/dự án con
- *feature domain* = miền chức năng (cách tổ chức package theo nghiệp vụ, ví dụ auth, billing, notifications)
- *YAML frontmatter* = phần metadata dạng YAML đặt ở đầu file (thường dùng để khai báo thuộc tính, phạm vi áp dụng...)
- *override instructions* = hướng dẫn ghi đè/loại trừ (ví dụ "bỏ qua quy tắc X trong trường hợp Y") — dễ sai vì phụ thuộc model diễn giải đúng điều kiện phủ định

---

## Practice Test 2 — Q91: Resume Extraction — Guaranteed JSON Schema Compliance (confirms Q72 pattern, ⚠️ self-reasoned, no answer key shown)

**Note:** Near-duplicate of Q72 (Task Statement 4.3 — Tool use & JSON schemas), same underlying rule, different scenario (extracting candidate information from resumes instead of calendar event details). No "Đúng/Sai" tag was visible on this screenshot (only a selected radio button), so this answer follows from the already-confirmed Q72 rule by reasoning, not a freshly confirmed key.

### Example question (Practice Test 2, Q91 — answered by reasoning, matches confirmed Q72 rule)

**Q:** The system needs to extract candidate information (name, contact details, skills, work experience, education) from uploaded resumes. The extracted data must strictly conform to a predefined JSON schema, as missing required fields or incorrect data types will cause downstream validation failures. What is the most reliable approach to ensure Claude's output consistently matches the schema?

- ❌ Parse Claude's text response with regex patterns to extract JSON objects, using retry logic for malformed responses.
- ❌ Include detailed JSON formatting instructions and a template example in the system prompt, asking Claude to output only valid JSON.
- ❌ Make two separate API calls—first extracting information as text, then asking Claude to format that text as JSON.
- ✅ **Define a tool with an input schema matching your required JSON structure and extract the data from Claude's tool_use response.** *(my reasoned answer, following the confirmed Q72 rule — not independently confirmed by a shown key)*

**Glossary (thuật ngữ):**
- *candidate information extraction* = trích xuất thông tin ứng viên (từ CV/resume — tên, liên hệ, kỹ năng, kinh nghiệm, học vấn)
- *tool_use response* = phản hồi dạng gọi tool của model (chứa dữ liệu có cấu trúc theo đúng input schema đã định nghĩa cho tool đó)

---

## Practice Test 2 — Q92: Error-Type-Specific, Instructive Tool Error Messages (confirms existing isError pattern — answered correctly)

**Note:** Near-duplicate/extension of the existing "MCP Tool Error Handling (isError)" rule (Task Statement 2.2) and Anthropic's tool-use documentation guidance ("Write instructive error messages... include what went wrong and what Claude should try next"). Same core lesson, new comparison set of implementation options.

| Option | Verdict |
|---|---|
| **Return error-type-specific messages with `is_error: true`**, e.g., "order not found — try get_customer to search by phone" for data errors, "Database timeout (transient) — retry should succeed" for infrastructure errors | ✅ Keeps the `is_error` flag (Claude still knows it's a failure) while making the message itself instructive and specific to the error type — exactly what Anthropic's documented recommendation asks for |
| Remove `is_error: true` and return error details as normal tool content, so Claude reasons about it as data | ❌ Trap: strips the failure signal entirely — Claude may treat the error text as valid data instead of recognizing a failed call |
| Add an error classification step in the agentic loop that intercepts tool errors before Claude sees them, tags each as "retry"/"try_alternative"/"escalate" | ❌ Trap: unnecessary indirection — the tool itself already knows the error type; classifying it externally after the fact just adds a layer instead of having the source say so directly |
| Implement retry logic with exponential backoff inside each tool implementation so transient errors are resolved transparently before any failure surfaces to Claude | ❌ Trap: business/permanent errors (e.g., order truly not found) will never succeed no matter how many silent retries — just delays and hides the failure; and even for genuinely transient errors, Claude never learns a retry happened, losing visibility into the loop |

**Tips:** The fix keeps the structural error signal (`is_error: true`) intact and improves only the *content* of the message: name the specific error type and suggest a concrete next action. Don't strip the flag (that hides the failure), don't move classification to an external layer (the tool already knows), and don't silently retry inside the tool (permanent errors never resolve, and even transient retries should be visible to the agent, not hidden from it).

**Source check (official Exam Guide, Task Statement 2.2 — MCP tool error handling; Anthropic tool-use documentation):**
- "Write instructive error messages. Instead of generic errors like 'failed', include what went wrong and what Claude should try next."
- Consistent with the existing isError rule: structured, error-type-specific metadata/messages let the agent choose retry vs. try-alternative vs. escalate, rather than guessing from a generic message or losing the failure signal altogether.

### Example question (Practice Test 2, Q92 — answered correctly)

**Q:** Anthropic's tool use documentation states: "Write instructive error messages. Instead of generic errors like 'failed', include what went wrong and what Claude should try next." A billing dispute agent uses lookup_order, which catches all exceptions and returns a tool_result with is_error: true and the message "execution failed". Monitoring shows two failure modes: the agent retries the identical call until hitting the turn limit, or it immediately calls escalate_to_human without trying alternative tools. Which change follows the documented recommendation and gives Claude the information it needs to select the correct recovery action for each error type?

- ❌ Remove is_error: true and return the error details as normal tool content, so Claude reasons about the response as data rather than treating it as a flagged failure condition that biases retry behavior.
- ❌ Add an error classification step in the agentic loop that intercepts tool errors before Claude sees them, tags each as "retry," "try_alternative," or "escalate," and adds that recommendation to the tool result.
- ✅ **Return error-type-specific messages with is_error: true, e.g., "order not found-try get_customer to search by phone" for data errors and "Database timeout (transient)-retry should succeed" for infrastructure errors.**
- ❌ Implement retry logic with exponential backoff inside each tool implementation so transient errors are resolved transparently within the tool before any failure result is surfaced to Claude in the agentic loop.

**Glossary (thuật ngữ):**
- *instructive error message* = thông báo lỗi mang tính hướng dẫn (nêu rõ vấn đề là gì và bước tiếp theo nên thử, thay vì chỉ nói "thất bại")
- *is_error: true* = cờ đánh dấu tool_result là một lỗi (giữ nguyên để Claude biết đây là thất bại, không phải dữ liệu hợp lệ)
- *transient error* = lỗi tạm thời (ví dụ timeout, mất kết nối tạm thời — thử lại thường sẽ thành công)
- *data error* = lỗi do dữ liệu (ví dụ không tìm thấy đơn hàng — thử lại y hệt sẽ không giải quyết được, cần cách tiếp cận khác)
- *escalate_to_human* = chuyển tiếp cho con người xử lý

---

## Practice Test 2 — Q93: Uniform MCP Error Responses, Reworded (exact near-duplicate of Q65 — answered correctly)

**Note:** Same scenario and same correct answer as Q65 (Task Statement 2.2 — MCP isError / structured error metadata), reworded (`tool_code`/`tool_id` instead of `lookup_order`/order ID, "5 times" instead of "5+ times", JSON error shape `{"status": "error", "content": "{\"type\": \"Error\", \"message\": \"Operation failed.\"}"}`). See Q65 for the full rule table and reasoning — identical lesson: fix uniform/generic error responses with structured metadata (`error_category`, retryability, cause) at the source, not few-shot examples, not a separate `analyze_error` tool, not blanket retry-with-backoff.

### Example question (Practice Test 2, Q93 — answered correctly; near-duplicate of Q65)

**Q:** Production logs reveal inconsistent error handling: when tool_code fails, the agent sometimes retries 5 times (even if the tool_id doesn't exist), sometimes escalates immediately (premature for temporary network issues), and sometimes adds user-friendly explanation (inappropriate when the issue is a backend permission error). Investigation shows four MCP tool returns uniform error responses: {"status": "error", "content": "{\"type\": \"Error\", \"message\": \"Operation failed.\"}"}. The agent learns different types. What's the most effective improvement?

- ✅ **Enhance error responses with structured metadata. Include error_category (transient/retriable/permission), reason, and a description of what caused the failure.**
- ❌ Implement retry logic with exponential backoff in your MCP server for all errors, returning to the agent only after retries are exhausted.
- ❌ Create an analyze_error MCP tool the agent calls after any failure to determine the error category and recommended action.
- ❌ Add a few-shot examples to the system prompt demonstrating how to interpret error message patterns and select appropriate responses for each.


## Practice Test 2 — Q94: Force Metadata Extraction Before Enrichment Tool Calls
### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này được trích ra trực tiếp từ bài thi mẫu và chuẩn kiến thức **Claude Certified Architect – Foundations Exam Guide**:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [625–626].
  * **Domain 4: Prompt Engineering & Structured Output** (Trọng số **20%**) [625–626].
* **Task Statements liên quan:**
  * **Task Statement 2.3:** Distribute tools appropriately across agents and configure tool choice [652–655].
  * **Task Statement 4.3:** Enforce structured output using tool use and JSON schemas [678–681].
* **Trích dẫn chuẩn từ Exam Guide:**
  > *"Using tool_choice forced selection to ensure a specific tool is called first (e.g., forcing extract_metadata before enrichment tools), then processing subsequent steps in follow-up turns"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nhận diện dạng bài phụ thuộc dữ liệu (Data Dependency):** Khi bài toán xuất hiện hai nhóm công cụ — một nhóm trích xuất dữ liệu gốc (như `extract_metadata` để tạo mã DOI) và một nhóm làm phong phú dữ liệu (`lookup_citations`, `verify_doi` bắt buộc phải dùng DOI làm input) — thứ tự gọi công cụ mang tính quyết định.
2. **Kỹ thuật "Forced Tool Selection":** Để đảm bảo 100% mô hình không bao giờ gọi nhầm công cụ enrichment trước, bạn phải bắt buộc chọn công cụ gốc ở lượt 1 bằng cú pháp `tool_choice: {"type": "tool", "name": "extract_metadata"}`.
3. **Cạm bẫy vòng lặp vô tận (Infinite Loop Trap):** Ép buộc gọi một công cụ **chỉ được thực hiện ở lượt gọi API đầu tiên**. Nếu áp dụng cấu hình ép buộc này cho mọi lượt gọi API trong pipeline, hệ thống sẽ rơi vào vòng lặp gọi đi gọi lại công cụ đó mà không bao giờ chuyển sang bước tiếp theo.
4. **Quy tắc bác bỏ các đáp án bẫy:**
   * **Không dựa vào thứ tự mảng:** Claude API không ưu tiên chọn công cụ dựa trên vị trí của nó trong mảng `tools` [647–648].
   * **`tool_choice: "any"` không chỉ định đích danh:** Chế độ `"any"` chỉ bắt buộc mô hình dùng *một công cụ bất kỳ*, không giải quyết được việc ép buộc đúng công cụ cần thiết.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Set tool_choice to {"type": "tool", "name": "extract_metadata"} and process the enrichment requests in subsequent turns after receiving the extracted metadata.**
  * **Phân tích:** Đây là thiết kế chuẩn chuẩn đoán theo tài liệu của Anthropic. Ở lượt (turn) 1, hệ thống ép Claude phải gọi `extract_metadata` để lấy mã DOI. Sau khi backend nhận được DOI và đưa vào lịch sử hội thoại, các lượt thoại tiếp theo sẽ mở lại chế độ mặc định (`"auto"`) để Claude sử dụng DOI đó gọi các công cụ enrichment như `lookup_citations` hay `verify_doi`.

* **Lựa chọn 1 (Sai):** *Set tool_choice to {"type": "tool", "name": "extract_metadata"} for every API call in the pipeline, ensuring Claude always extracts metadata before any enrichment can occur.*
  * **Phân tích:** Việc áp dụng cấu hình ép buộc này cho **mọi đợt gọi API** (for every API call) là một lỗi kiến trúc nghiêm trọng. Claude sẽ bị ép phải chạy đi chạy lại `extract_metadata` ở tất cả các lượt thoại và không bao giờ có thể tiến sang bước gọi các công cụ làm phong phú dữ liệu.

* **Lựa chọn 2 (Sai):** *Set tool_choice to "any" so Claude must use a tool, combined with system prompt instructions prioritizing extract_metadata.*
  * **Phân tích:** `tool_choice: "any"` chỉ đảm bảo rằng Claude sẽ chọn *ít nhất một công cụ bất kỳ* thay vì trả về văn bản tự nhiên. Việc kết hợp với System Prompt vẫn mang tính xác suất và sẽ thất bại khi câu hỏi của người dùng có chứa các từ khóa kích thích mô hình gọi trực tiếp `lookup_citations`.

* **Lựa chọn 3 (Sai):** *Set tool_choice to "auto" and reorder the tool definitions so extract_metadata appears first in the tools array, since Claude prioritizes earlier-listed tools.*
  * **Phân tích:** Đây là một quan niệm sai lầm về cơ chế vận hành của Claude API [647–648]. Mô hình không ưu tiên công cụ dựa trên thứ tự sắp xếp trong mảng `tools`, mà lựa chọn dựa trên sự phù hợp giữa ngữ cảnh hội thoại và phần mô tả (`description`) của từng công cụ [647–648].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Forced Tool Selection** | Ép buộc chọn công cụ | Cấu hình `tool_choice` chỉ định rõ tên một công cụ bắt buộc Claude phải gọi ở lượt API hiện tại. |
| **Enrichment Tools** | Công cụ làm phong phú dữ liệu | Các công cụ gọi API phụ để bổ sung thông tin (ví dụ: tìm trích dẫn, kiểm tra DOI) sau khi đã có dữ liệu cốt lõi. |
| **Data Dependency** | Phụ thuộc dữ liệu | Ràng buộc trong đó Công cụ B bắt buộc phải sử dụng đầu ra (output) của Công cụ A làm đầu vào (input). |
| **Multi-turn Workflow** | Quy trình xử lý đa lượt | Chuỗi tương tác gọi API nhiều lần, trong đó kết quả của lượt trước được nạp vào ngữ cảnh cho lượt sau [632–633, 654]. |
| **`tool_choice: "auto"`** | Chế độ chọn công cụ tự động | Chế độ mặc định, cho phép Claude tự quyết định gọi công cụ hoặc trả về văn bản tự nhiên. |
| **`tool_choice: "any"`** | Chế độ bắt buộc dùng công cụ | Bắt buộc Claude phải sử dụng ít nhất một công cụ trong danh sách, nhưng không cố định công cụ nào. |
| **Infinite Loop Trap** | Bẫy vòng lặp vô tận | Lỗi lập trình khi duy trì ép buộc chọn một công cụ liên tục ở mọi đợt gọi API. |
| **Tool Array Ordering** | Thứ tự mảng công cụ | Vị trí khai báo công cụ trong mảng `tools` (không có giá trị quyết định độ ưu tiên chọn công cụ của Claude) [647–648]. |


## Practice Test 2 — Q95: Turn-Limited Agentic Review for Cross-File Findings

**Đáp án đúng:**  
**Redesign the review as a turn-limited agentic task where the model can read files and search the codebase via tools, following references to verify cross-file findings.** *(Thiết kế lại quy trình review thành một tác vụ tác tử giới hạn số lượt, trong đó mô hình có thể đọc tệp và tìm kiếm codebase thông qua các công cụ, truy vết các điểm tham chiếu để xác minh các phát hiện liên tệp)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [619–621]:

*   **Phân vùng kiến thức chính (Primary Domains):**
    *   **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**).
    *   **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**).
*   **Task Statements liên quan:**
    *   **Task Statement 1.1:** Design and implement agentic loops for autonomous task execution [632–634].
    *   **Task Statement 2.5:** Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively [657–659].
*   **Trích dẫn chuẩn từ tài liệu Exam Guide:**
    *   > *"Building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront"*.
    *   > *"Tracing function usage across wrapper modules by first identifying all exported names, then searching for each name across the codebase"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nhận diện bài toán "Bỏ sót ngữ cảnh do không được nạp" (Missing Context Problem):** Khi mô hình chỉ nhận được `diff` và các tệp bị thay đổi (`changed files`), nó hoàn toàn **không có thông tin** về các tệp không thay đổi (`unchanged files`). Vì vậy, các câu lệnh yêu cầu mô hình "suy luận" hoặc "đoán" (như Chain-of-Thought) đều thất bại vì thiếu dữ liệu đầu vào.
2. **Quy tắc "Agentic Exploration vs Static Ingestion":** 
   * **Nạp tĩnh (Static Ingestion):** Cố gắng nạp tất cả các tệp phụ thuộc vào prompt sẽ gây ra hiện tượng phình to token, phân tán sự chú ý (*Attention Dilution*) và hiệu ứng *"Lost in the Middle"* [688–689].
   * **Tác tử linh hoạt (Agentic Exploration):** Cho phép mô hình dùng công cụ (`Grep` để tìm vị trí gọi hàm trên toàn dự án, `Read` để đọc tệp chứa vị trí đó) để chủ động truy vết khi phát hiện thay đổi [657–659].
3. **Giới hạn an toàn "Turn-limited":** Chèn giới hạn lượt thoại (*turn-limited*) giúp tác tử không bị lặp vô tận (*infinite loop*) hoặc tiêu tốn quá nhiều chi phí API khi kiểm tra các dự án lớn.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng (Lựa chọn 4):** **Redesign the review as a turn-limited agentic task where the model can read files and search the codebase via tools, following references to verify cross-file findings.**
    *   **Phân tích:** Đây là kiến trúc tối ưu nhất theo khuyến nghị của Anthropic [657–659]. Claude xem `diff`, thấy hàm `update_user()` bị đổi tham số, sau đó tự động dùng `Grep` quét toàn bộ codebase xem tệp nào đang gọi `update_user()`, và dùng `Read` mở đúng tệp đó ra kiểm tra xem cú pháp gọi cũ có bị lỗi không. Cách này giúp phát hiện 100% lỗi cross-file mà không cần nạp thừa token [658–659].

*   **Lựa chọn 1 (Sai):** *Use static analysis to build a dependency graph of changed code, then expand the prompt to include all files within two dependency hops of any changed file.*
    *   **Phân tích:** Mở rộng prompt nạp tĩnh tất cả các tệp trong bán kính 2 dependency hops sẽ khiến kích thước prompt bùng nổ (token bloat), dẫn đến hiện tượng trôi chỉ dẫn (*Instruction Degradation*) và làm suy giảm khả năng chú ý của Claude (*Attention Dilution / Lost in the Middle*) [688–689].

*   **Lựa chọn 2 (Sai):** *Run parallel review passes per changed file with direct dependents included in each pass, then aggregate and deduplicate findings using a final summarization call.*
    *   **Phân tích:** Chia nhỏ các đợt review song song kèm tệp phụ thuộc trực tiếp vẫn là phương pháp nạp tĩnh. Cách này vừa tốn kém chi phí gọi API gấp nhiều lần, vừa cồng kềnh quy trình mà vẫn có thể bỏ sót lỗi nếu liên kết phụ thuộc nằm ở hop thứ 2 hoặc xa hơn.

*   **Lựa chọn 3 (Sai):** *Add chain-of-thought instructions asking the model to list all external references in the diff, then reason step-by-step about how each change might affect callers in other files.*
    *   **Phân tích:** Chain-of-Thought không thể tạo ra thông tin mà mô hình không được cung cấp. Nếu các tệp chưa thay đổi không nằm trong prompt, Claude không thể "suy luận" chính xác những tệp đó đang truyền tham số gì, dẫn đến việc mô hình đoán mò (*hallucination*).

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Cross-file Interactions** | Tương tác liên tệp | Mối quan hệ phụ thuộc giữa các tệp khác nhau trong codebase (ví dụ: hàm ở tệp A gọi phương thức ở tệp B). |
| **Turn-limited Agentic Task** | Tác vụ tác tử giới hạn số lượt | Tác tử hoạt động có cờ chặn số lượt vòng lặp tối đa để tránh tiêu tốn token vô hạn. |
| **Incremental Exploration** | Khám phá tăng tiến | Phương pháp tìm kiếm codebase theo từng bước bằng `Grep` và `Read` thay vì nạp toàn bộ code ngay từ đầu [658–659]. |
| **Dependency Hop** | Bước nhảy phụ thuộc | Cấp độ phụ thuộc trực tiếp hoặc gián tiếp giữa các module/tệp trong dự án. |
| **Attention Dilution** | Phân tán sự chú ý | Hiện tượng mô hình giảm độ chính xác khi prompt chứa quá nhiều đoạn văn bản không liên quan [688–689]. |
| **Static Context Injection** | Nạp ngữ cảnh tĩnh | Việc nhồi toàn bộ mã nguồn vào prompt trước khi gọi API thay vì cho phép mô hình tự tra cứu qua công cụ. |


## Practice Test 2 — Q96: Local Retry with Exponential Backoff for Transient Tool Failures

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này được trích ra trực tiếp từ bài thi mẫu và chuẩn kiến thức **Claude Certified Architect – Foundations Exam Guide** [617–618]:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [623–624].
  * **Domain 5: Context Management & Reliability** (Trọng số **15%**).
* **Task Statements liên quan:**
  * **Task Statement 2.2:** Implement structured error responses for MCP tools [648–650].
  * **Task Statement 5.3:** Implement error propagation strategies across multi-agent systems [692–694].
* **Trích dẫn chuẩn từ Exam Guide:**
  > *"Knowledge of: The distinction between transient errors (timeouts, service unavailability), validation errors, and business errors..."*.  
  > *"Skills in: Implementing local error recovery within subagents/tools for transient failures, propagating to the coordinator/agent only errors that cannot be resolved locally..."* [649–650, 694].  
  > *"Knowledge of: Why silently suppressing errors (returning empty results as success) or terminating entire workflows on single failures are both anti-patterns"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Phân loại lỗi HTTP (Transient vs. Permanent Errors):** Lỗi **HTTP 503 Service Unavailable** là lỗi hạ tầng/mạng tạm thời (*transient error*). Loại lỗi này thường chỉ diễn ra trong khoảnh khắc và có khả năng tự phục hồi rất cao sau một khoảng thời gian ngắn.
2. **Nguyên tắc "Local Error Recovery" tại cấp công cụ:** Khi một công cụ gặp lỗi tạm thời từ API bên ngoài, cách xử lý hiệu quả nhất là tự động **thử lại (retry) kèm thuật toán lùi thời gian lũy thừa (exponential backoff)** ngay bên trong mã nguồn triển khai của công cụ (*tool implementation*) [649–650]. Việc này giúp xử lý triệt để sự cố mà không làm phiền đến tác tử LLM, tránh tốn lãng phí token và lượt thoại (turns) [649–650].
3. **Nhận diện và loại bỏ Phản mẫu (Anti-Patterns):**
   * **Nuốt lỗi / Giả lập thành công (Silently Suppressing Errors):** Trả về danh sách rỗng hoặc phản hồi trống khi gặp lỗi là phản mẫu bị cấm hoàn toàn trong kiến trúc Anthropic.
   * **Báo lỗi ngay lập tức:** Trả lỗi về cho Claude ngay ở lần gặp 503 đầu tiên khi chưa thử retry ở cấp công cụ sẽ bắt LLM phải xử lý một sự cố tạm thời không cần thiết [649–650].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Automatically retry the request up to five times with exponential backoff before returning results to the agent.**
  * **Phân tích:** Đây là thiết kế chuẩn theo hướng dẫn của Anthropic cho lỗi tạm thời (*transient failures*) [649–650]. Việc tự động retry với exponential backoff giúp công cụ tự phục hồi các đợt gián đoạn dịch vụ ngắn từ API hãng hàng không mà không làm gián đoạn vòng lặp tác tử (*agentic loop*) [649–650]. Nếu sau các lần thử lại vẫn thất bại, công cụ mới trả về thông báo lỗi có cấu trúc kèm cờ `isError: true` [648–649].

* **Lựa chọn 2 (Sai):** *Log the error internally and return an empty response, letting the model continue without the flight data.*
  * **Phân tích:** Vi phạm nguyên tắc báo lỗi của Anthropic. Việc im lặng trả về phản hồi rỗng (*silently suppressing errors*) khiến Claude không nhận biết được sự cố API và sẽ đưa ra quyết định sai lệch ở các bước tiếp theo do thiếu thông tin.

* **Lựa chọn 3 (Sai):** *Return an error message in the tool result explaining the service is temporarily unavailable.*
  * **Phân tích:** Việc đẩy ngay thông báo lỗi về cho Claude ở lần 503 đầu tiên mà không thử lại (retry) ở cấp triển khai công cụ sẽ bắt tác tử phải dừng luồng xử lý hoặc chuyển hướng không cần thiết cho một lỗi tạm thời có thể tự hết sau vài mili-giây [649–650].

* **Lựa chọn 4 (Sai):** *Return an empty flight list as if the search succeeded but found no matching flights.*
  * **Phân tích:** Đây là phản mẫu nguy hiểm nhất trong thiết kế công cụ (*returning empty results as success*). Trả về danh sách chuyến bay rỗng sẽ khiến Claude báo sai cho người dùng rằng "không có chuyến bay nào phù hợp", trong khi thực tế là do API bị gián đoạn kết nối.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Transient Error** | Lỗi tạm thời / Lỗi ngắt quãng | Lỗi hạ tầng hoặc mạng diễn ra trong thời gian ngắn (như Timeout, 503 Service Unavailable) có thể tự hết khi thử lại. |
| **Local Error Recovery** | Tự khôi phục lỗi cục bộ | Kỹ thuật xử lý lỗi ngay bên trong mã nguồn triển khai của công cụ/subagent trước khi đẩy lỗi lên cấp quản lý cao hơn [649–650, 694]. |
| **Exponential Backoff** | Lùi thời gian chờ lũy thừa | Thuật toán tăng dần khoảng thời gian chờ giữa các lần thử lại (ví dụ: 1s, 2s, 4s, 8s) để tránh làm quá tải API mục tiêu. |
| **Silently Suppressing Errors** | Nuốt lỗi âm thầm | Phản mẫu thiết kế khi công cụ gặp sự cố nhưng lại trả về kết quả rỗng hoặc báo thành công giả lập. |
| **Valid Empty Results** | Kết quả rỗng hợp lệ | Truy vấn thành công nhưng hệ thống không tìm thấy dữ liệu khớp (phân biệt hoàn toàn với trường hợp truy vấn thất bại do lỗi kết nối). |
| **IsRetryable Flag** | Cờ báo khả năng thử lại | Cờ dữ liệu có cấu trúc trả về cho tác tử để cho biết lỗi này có nên tiếp tục gọi lại công cụ hay không. |

---

## Practice Test 2 — Q97: Split Generic Workout Logging into Purpose-Specific Tools

**Đáp án đúng:**  
**Split into `log_cardio_workout` (with `duration_minutes` or `distance_miles` parameters) and `log_strength_workout` (with `reps` and `sets` parameters).** *(Tách thành hai công cụ riêng biệt: `log_cardio_workout` (chứa các tham số `duration_minutes` hoặc `distance_miles`) và `log_strength_workout` (chứa các tham số `reps` và `sets`))*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

*   **Phân vùng kiến thức chính (Primary Domain):** **Domain 2: Tool Design & MCP Integration** (Quản lý & Thiết kế Công cụ, Tích hợp MCP – chiếm **18%** tổng trọng số đề thi) [623–624].
*   **Task Statement:** **Task Statement 2.1: Design effective tool interfaces with clear descriptions and boundaries** (Thiết kế giao diện công cụ hiệu quả với mô tả và ranh giới rõ ràng) [645–648].
*   **Trích dẫn nguyên văn từ Exam Guide Blueprint:**
    *   *Kiến thức (Knowledge of):* 
        > *"How ambiguous or overlapping tool descriptions cause misrouting..."*.
    *   *Kỹ năng (Skills in):* 
        > *"Splitting generic tools into purpose-specific tools with defined input/output contracts (e.g., splitting a generic analyze_document into extract_data_points, summarize_content, and verify_claim_against_source)"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1.  **Nguyên tắc "Tách công cụ dùng chung" (Tool Splitting vs. Generic Tools):**  
    Khi một công cụ dùng chung (`generic tool`) như `log_workout` gánh quá nhiều chức năng thuộc các nhóm dữ liệu khác nhau (cardio vs. strength) dẫn đến tỷ lệ truyền sai tham số cao (23%), giải pháp kiến trúc triệt để nhất của Anthropic là **tách công cụ dùng chung thành các công cụ chuyên biệt** (`purpose-specific tools`).
2.  **Ràng buộc ở cấp Schema (Schema Constraints) > Hướng dẫn Prompt (Prose Instructions):**  
    Việc tách schema thành hai hợp đồng dữ liệu riêng biệt khiến các sự kết hợp sai lệch (như truyền `reps` cho bài tập chạy bộ) trở nên **bất khả thi về mặt cấu trúc** (*structurally impossible*). Mô hình không thể chọn nhầm tham số vì các tham số không hợp lệ hoàn toàn không tồn tại trong schema của công cụ đó [637–638, 647].
3.  **Xử lý chủ động (Proactive Prevention) > Xử lý bị động (Reactive Retry):**  
    Đừng dựa vào việc trả lỗi từ server để bắt mô hình thử lại (*retry*), vì cách làm này gây lãng phí token và gia tăng độ trễ (*latency*). Hãy triệt tiêu nguy cơ sinh lỗi ngay từ khâu thiết kế giao diện công cụ [647, 649–650].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng:** **Split into `log_cardio_workout` (with `duration_minutes` or `distance_miles` parameters) and `log_strength_workout` (with `reps` and `sets` parameters).**
    *   **Phân tích:** Đây là thiết kế chuẩn xác nhất theo hướng dẫn của Anthropic. Việc chia tách thành hai công cụ với tham số rõ ràng giúp loại bỏ hoàn toàn khả năng mô hình truyền nhầm `reps` cho chạy bộ hay `miles` cho đẩy ngực. Mọi cuộc gọi công cụ sẽ đạt độ chính xác 100% ở cấp độ cú pháp schema.

*   **Lựa chọn 1 (Sai):** *Add explicit examples to the tool description showing valid combinations (e.g., "For running: use minutes or miles. For push-ups: use reps") with constraints for each exercise category.*
    *   **Phân tích:** Bổ sung ví dụ vào phần mô tả công cụ (*tool description*) giúp mô hình hiểu ngữ cảnh tốt hơn, nhưng đây vẫn là giải pháp dựa trên chỉ dẫn văn bản tự nhiên mang tính xác suất. Tỷ lệ lỗi 23% có thể giảm nhưng **không thể triệt tiêu hoàn toàn** vì schema của công cụ `log_workout` vẫn cho phép truyền các tham số sai [646–647].

*   **Lựa chọn 2 (Sai):** *Add enum constraints on measurement limiting values to "minutes", "miles", "reps", or "sets" to prevent arbitrary measurement strings.*
    *   **Phân tích:** Việc thêm `enum` chỉ chặn được việc người dùng/mô hình nhập các chuỗi ký tự lạ (như "hours" hay "kilograms"), nhưng **hoàn toàn không ngăn được lỗi kết hợp sai**. Mô hình vẫn có thể chọn `exercise_type: "running"` đi kèm với một giá trị enum hợp lệ trong danh sách là `measurement: "reps"`.

*   **Lựa chọn 4 (Sai):** *Implement server-side validation returning descriptive errors for invalid combinations, allowing the agent to retry with corrections.*
    *   **Phân tích:** Đây là cách xử lý bị động (*reactive error handling*). Mặc dù trả về thông báo lỗi có cấu trúc là một thực hành tốt [648–650], nhưng việc để 23% số lượng cuộc gọi bị thất bại rồi bắt tác tử phải xử lý thử lại (*retry loop*) sẽ làm tăng gấp đôi độ trễ (*latency*) và chi phí API không cần thiết, trong khi hoàn toàn có thể phòng ngừa từ sớm bằng cách tách schema.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Generic Tool** | Công cụ dùng chung cồng kềnh | Công cụ ôm đồm quá nhiều chức năng/tham số của các nhóm tác vụ khác nhau. |
| **Purpose-Specific Tools** | Các công cụ chuyên biệt | Các công cụ được chia nhỏ với hợp đồng đầu vào/đầu ra (schema) được xác định chặt chẽ. |
| **Tool Splitting** | Kỹ thuật tách công cụ | Mẫu thiết kế chia một công cụ cồng kềnh thành các công cụ nhỏ để triệt tiêu lỗi chọn sai tham số. |
| **Schema-Level Constraint** | Ràng buộc ở cấp độ Schema | Việc giới hạn khả năng truyền sai dữ liệu trực tiếp bằng cấu trúc JSON Schema thay vì câu lệnh prompt [637–638]. |
| **Parameter Mismatch** | Lỗi bất đồng bộ / lệch tham số | Hiện tượng tác tử truyền các tham số không tương thích với loại tác vụ (ví dụ: `reps` cho bài tập chạy). |
| **Enum Constraint** | Ràng buộc tập giá trị cố định | Việc giới hạn một trường dữ liệu chỉ được phép nhận các giá trị trong một danh sách `enum` định sẵn. |
| **Reactive Error Handling** | Xử lý lỗi bị động | Cách xử lý đợi lỗi xảy ra trên server rồi mới trả thông báo bắt tác tử gọi lại công cụ (*retry*) [648–650]. |
| **Proactive Error Prevention** | Phòng ngừa lỗi chủ động | Kỹ thuật thiết kế giao diện/schema chuẩn xác để ngăn chặn lỗi xảy ra ngay từ lần gọi đầu tiên. |


## Practice Test 2 — Q98: Clarifying Action-Type Ambiguity Before Execution

**Đáp án đúng:**  
**Ask one clarifying question about action type: play now or configure for later** *(Đặt một câu hỏi làm rõ duy nhất về loại hành động: phát nhạc ngay hay cấu hình cho sau này)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

*   **Phân vùng kiến thức chính (Primary Domains):**
    *   **Domain 5: Context Management & Reliability** (Trọng số **15%**) [623–624].
    *   **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
*   **Task Statements liên quan:**
    *   **Task Statement 5.2:** Design effective escalation and ambiguity resolution patterns (Thiết kế các mẫu xử lý sự mơ hồ và chuyển giao hiệu quả) [689–692].
    *   **Task Statement 1.4:** Implement multi-step workflows with enforcement and handoff patterns [637–639].
*   **So sánh hai dạng mơ hồ (Ambiguity Comparison):**
    1.  *Mơ hồ về Tham số (Parameter Ambiguity - ví dụ câu hỏi đặt địa điểm tổ chức tiệc trước đó):* Ý định cốt lõi đã rõ ràng ("đặt chỗ"), chỉ thiếu tham số (ngày, số khách). Giải pháp tối ưu là nêu giả định hợp lý và đưa ra gợi ý ngay [689–692].
    2.  *Mơ hồ về Loại Hành động (Action Type Ambiguity - câu hỏi hiện tại):* Câu lệnh "Set up my focus music" có thể dẫn đến 3 nhánh luồng công việc hoàn toàn khác nhau (phát nhạc ngay, tạo playlist, hay cài đặt tùy chọn) [689–692]. Giải pháp tối ưu là đặt **1 câu hỏi làm rõ tập trung** vào ranh giới phân nhánh hành động chính (*play now vs configure for later*) trước khi thực thi [689–692].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1.  **Nhận diện loại sự mơ hồ (Intent vs. Parameter):**
    *   Nếu mơ hồ về *loại hành động* (chưa biết người dùng muốn nghe nhạc hay muốn cài đặt) \\(\rightarrow\\) **Đặt 1 câu hỏi định hướng nhánh** (*one targeted clarifying question*) với các lựa chọn ngắn gọn nhất [689–692].
    *   Nếu mơ hồ về *chi tiết tham số* \\(\rightarrow\\) Đưa ra **giả định công khai** và chạy thử nghiệm ngay.
2.  **Tránh bẫy "Đoán mò ý định" (Intent Guessing Trap):** Tự ý thực hiện một hành động phức tạp (phát nhạc ngẫu nhiên hoặc tạo playlist) khi người dùng mới có thể đang muốn thiết lập gu âm nhạc sẽ gây trải nghiệm gượng ép và không chính xác.
3.  **Tránh bẫy "Ma sát nhận thức cao" (High Cognitive Friction Trap):** Bắt người dùng mới phải trả lời ngay một biểu mẫu chi tiết (thể loại, nhịp độ, nghệ sĩ) ngay từ tin nhắn đầu tiên sẽ làm tăng tỷ lệ bỏ dở phiên làm việc (*abandonment rate*) [689–692].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng:** **Ask one clarifying question about action type: play now or configure for later**
    *   *Phân tích:* Đây là thiết kế tương tác tối ưu nhất [689–692]. Câu hỏi làm rõ ngắn gọn với 2 lựa chọn phân nhánh (*Play now* vs *Configure for later*) giúp xác định chính xác mục tiêu của người dùng chỉ sau 1 lượt phản hồi đơn giản, triệt tiêu nguy cơ đoán sai ý định mà không gây ma sát nhận thức [689–692].

*   **Lựa chọn 2 (Sai):** *Play popular focus tracks Immediately and let the user redirect if needed*
    *   *Phân tích:* Đây là phản mẫu đoán mò ý định. Nếu người dùng muốn thiết lập danh sách nhạc riêng hoặc điều chỉnh tùy chọn trước, việc tự ý phát nhạc ngẫu nhiên sẽ làm gián đoạn trải nghiệm và bắt người dùng phải thao tác dừng/sửa đổi [689–692].

*   **Lựa chọn 3 (Sai):** *Create a new "Focus" playlist with curated tracks and notify the user it's ready.*
    *   *Phân tích:* Tự động tạo tài nguyên mới (playlist) trên tài khoản người dùng khi chưa có sự xác nhận sẽ tạo ra các dữ liệu thừa không mong muốn nếu ý định thực sự của họ chỉ là nghe một bản nhạc nhanh.

*   **Lựa chọn 4 (Sai):** *Start preference configuration by asking about genres, temps, and artists they prefer for focus.*
    *   *Phân tích:* Đây là bẫy ma sát nhận thức lớn (*High Cognitive Friction*). Việc ép người dùng trả lời một loạt câu hỏi phỏng vấn chi tiết về thể loại, nhịp độ, nghệ sĩ ngay ở tin nhắn đầu tiên sẽ gây quá tải thông tin và làm tăng tỷ lệ người dùng thoát ứng dụng (*abandonment rate*) [689–692].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Action Type Ambiguity** | Mơ hồ về loại hành động | Tình huống câu lệnh có thể thuộc về các mục tiêu/ý định cốt lõi hoàn toàn khác nhau [689–692]. |
| **Targeted Clarifying Question** | Câu hỏi làm rõ có mục tiêu | Câu hỏi ngắn gọn, tập trung vào 1 điểm mấu chốt để phân nhánh luồng xử lý với ma sát thấp nhất [689–692]. |
| **Intent Guessing** | Đoán mò ý định | Phản mẫu thiết kế khi hệ thống tự ý thực hiện một hành động chính khi chưa rõ người dùng muốn gì. |
| **Cognitive Friction** | Ma sát nhận thức | Mức độ nỗ lực suy nghĩ/thao tác mà người dùng phải bỏ ra trong một lượt hội thoại. |
| **Parameter Ambiguity** | Mơ hồ về tham số | Tình huống ý định chính đã rõ nhưng thiếu các thông số chi tiết (ngày, giờ, số lượng) [689–692]. |
| **Abandonment Rate** | Tỷ lệ bỏ dở phiên | Tỷ lệ người dùng thoát hoặc ngừng tương tác do luồng hội thoại quá phức tạp/nhiều câu hỏi gạn hỏi [689–692]. |

---

## Practice Test 2 — Q99: Explicit Assumptions for Parameter Ambiguity


### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 5: Context Management & Reliability** (Quản lý ngữ cảnh & Độ tin cậy – chiếm **15%** tổng trọng số đề thi) [623–624].
* **Task Statement:** **Task Statement 5.2: Design effective escalation and ambiguity resolution patterns** (Thiết kế các mẫu xử lý sự mơ hồ và chuyển giao hiệu quả) [689–692].
* **Trích dẫn chuẩn từ Exam Guide:**
  > *"Knowledge of: How multiple clarifying questions create conversational friction and high abandonment rates..."* [689–692].  
  > *"Skills in: Instructing the agent to state explicit assumptions based on available context and proceed with recommendations while inviting corrections..."* [689–692].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nguyên tắc "Tối ưu hóa ma sát tương tác" (Reducing Conversational Friction):** Khi người dùng đưa ra câu lệnh mơ hồ về chi tiết/yêu cầu (như *"Can you help me with the report?"*), việc bắt họ trả lời một danh sách gồm 3–4 câu hỏi làm rõ cùng lúc sẽ gây quá tải nhận thức (*cognitive overload*), dẫn đến tỷ lệ bỏ dở phiên cao (40%) [689–692].
2. **Công thức giải quyết mơ hồ chuẩn của Anthropic:**
   \\[\text{Mô hình đưa ra Giả định rõ ràng (Explicit Assumptions)} + \text{Thực thi tạo kết quả ban đầu} + \text{Mời người dùng điều chỉnh (Invite Corrections)}\\]
3. **Phân biệt hai kỹ thuật xử lý sự mơ hồ:**
   * **Nêu giả định công khai (Option đúng trong bài này):** Áp dụng khi câu lệnh thiếu tham số/chi tiết nhưng mô hình có thể dự đoán được các phương án khả thi dựa trên ngữ cảnh [689–692].
   * **Hỏi 1 câu hỏi định hướng:** Chỉ áp dụng khi câu lệnh đứng trước các ranh giới phân nhánh hành động hoàn toàn khác nhau (như *Phát nhạc ngay v.s Cấu hình cài đặt*) [689–692].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Modify the system prompt to instruct the assistant to make reasonable assumptions from available context, state those assumptions explicitly, and offer to adjust if the interpretation is wrong.**
  * **Phân tích:** Đây là thiết kế tương tác tối ưu nhất được Anthropic khuyến nghị [689–692]. Tác tử sẽ tự động chọn một báo cáo gần nhất hoặc một dạng hỗ trợ phổ biến (ví dụ: *"Tôi giả định bạn muốn xem lại báo cáo doanh thu tuần này. Dưới đây là bản kiểm tra... Nếu bạn muốn chỉnh sửa báo cáo khác, hãy báo cho tôi biết"*). Cách này giúp người dùng nhìn thấy kết quả ngay lập tức để phản hồi, triệt tiêu ma sát tương tác và giảm tỷ lệ bỏ cuộc [689–692].

* **Lựa chọn 1 (Sai):** *Limit the assistant to one clarifying question per turn, using conversation history to accumulate answers over multiple exchanges rather than requesting everything upfront.*
  * **Phân tích:** Việc giới hạn 1 câu hỏi mỗi lượt chỉ kéo dài số lượt hội thoại (*conversational turns*) ra nhiều đợt liên tiếp. Người dùng vẫn phải mất 3 lượt trò chuyện mới bắt đầu nhận được kết quả, điều này vẫn tạo ra ma sát tương tác cao [689–692].

* **Lựa chọn 2 (Sai):** *Add a preprocessing step using a smaller model to classify request ambiguity on a 1-5 scale, routing high-ambiguity requests to a clarification dialog and low-ambiguity requests directly to the assistant.*
  * **Phân tích:** Lựa chọn này làm cồng kềnh hạ tầng (*over-engineering*) bằng cách thêm một mô hình phân loại trung gian, nhưng **không hề giải quyết được vấn đề trải nghiệm người dùng**. Với các yêu cầu có độ mơ hồ cao, hệ thống vẫn bắt người dùng trải qua luồng hỏi gạn gây khó chịu [689–692].

* **Lựa chọn 4 (Sai):** *Create a lookup table of common request patterns with predefined default interpretations, having the assistant respond with those defaults without stating the assumptions made.*
  * **Phân tích:** Đưa ra phản hồi mặc định nhưng **ngầm định (không nêu rõ giả định cho người dùng)** là một phản mẫu nguy hiểm (*dangerous anti-pattern*). Người dùng sẽ không hiểu tại sao hệ thống lại xử lý sai báo cáo của họ và không biết cách đưa ra câu lệnh điều chỉnh thích hợp [689–692].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Conversational Abandonment Rate** | Tỷ lệ bỏ dở phiên hội thoại | Tỷ lệ người dùng thoát ứng dụng khi luồng tương tác quá phức tạp hoặc bị hỏi gạn quá nhiều [689–692]. |
| **Explicit Assumptions** | Giả định công khai | Kỹ thuật tác tử tự động chọn giá trị khả thi và thông báo rõ ràng cho người dùng biết [689–692]. |
| **Conversational Friction** | Ma sát hội thoại | Nỗ lực nhận thức mà người dùng phải bỏ ra để trả lời các câu hỏi làm rõ của tác tử [689–692]. |
| **Predefined Default Interpretation** | Diễn giải mặc định định sẵn | Việc hệ thống chọn một phương án mặc định (nếu ngầm định không nêu rõ sẽ dễ gây hiểu lầm) [689–692]. |
| **Multi-question Response** | Phản hồi chứa nhiều câu hỏi | Phản mẫu thiết kế khi tác tử hỏi một lúc 3–4 câu hỏi gạn hỏi thông tin ở một lượt thoại [689–692]. |



## Practice Test 2 — Q100: Direct Execution for a Simple, Well-Scoped Change

**Đáp án đúng:**  
**Use direct execution to make the change** *(Sử dụng chế độ thực thi trực tiếp để thực hiện thay đổi)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này trích ra trực tiếp từ bài thi mẫu và chuẩn kiến thức **Claude Certified Architect – Foundations Exam Guide**:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 3: Claude Code Configuration & Workflows** (Trọng số **20%**).
* **Task Statement:** **Task Statement 3.4: Determine when to use plan mode vs direct execution** (Xác định khi nào nên dùng chế độ lập kế hoạch v.s thực thi trực tiếp) [664–666].
* **Trích dẫn chuẩn từ Exam Guide:**
  > *"Direct execution is appropriate for simple, well-scoped changes (e.g., adding a single validation check to one function)"*.  
  > *"Selecting direct execution for well-understood changes with clear scope (e.g., a single-file bug fix with a clear stack trace, adding a date validation conditional)"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Quy tắc ranh giới giữa Direct Execution và Plan Mode:**
   * **Thực thi trực tiếp (Direct Execution):** Dành cho các tác vụ đơn giản, phạm vi hẹp và rõ ràng (*well-scoped*), chỉ tác động lên 1 tệp hoặc 1 hàm duy nhất (ví dụ: thêm câu lệnh điều kiện `if`, sửa một lỗi đơn lẻ có stack trace rõ ràng) [664–666].
   * **Chế độ Lập kế hoạch (Plan Mode):** Dành cho các tác vụ phức tạp, tái cấu trúc hệ thống (*architectural decisions*), hoặc thay đổi liên quan đến nhiều tệp (*multi-file modifications*) [664–665].
2. **Nhận diện bẫy Over-engineering:** Khi đề bài mô tả một yêu cầu **cực kỳ cụ thể và phạm vi hẹp** (chỉ thêm một câu lệnh điều kiện kiểm tra ngày trong 1 hàm duy nhất thuộc 1 tệp), bất kỳ phương án nào đề xuất mở **Plan Mode** hay **Extended Thinking** đều là bẫy làm cồng kềnh quy trình và lãng phí tài nguyên/token [664–666].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 2):** **Use direct execution to make the change**
  * **Phân tích:** Đây là lựa chọn chính xác tuyệt đối theo nguyên văn hướng dẫn của Anthropic [664–666]. Với một tác vụ đơn giản, đã xác định chính xác vị trí (1 hàm trong 1 tệp) và nội dung cần sửa (thêm lệnh kiểm tra ngày ở tương lai), việc dùng **Direct Execution** giúp Claude Code hoàn thành công việc ngay lập tức mà không làm tốn token hay thời gian hội thoại [664–666].

* **Lựa chọn 1 (Sai):** *Enter plan mode to analyze how the validation might impact other parts of the reservation flow*
  * **Phân tích:** Việc bật Plan Mode để phân tích ảnh hưởng cho một câu lệnh kiểm tra logic cơ bản trong phạm vi 1 hàm là không cần thiết và gây lãng phí thời gian [664–666]. Plan Mode chỉ dành cho các thay đổi kiến trúc lớn tác động liên tệp [664–665].

* **Lựa chọn 3 (Sai):** *Start with extended thinking mode enabled to ensure thorough reasoning about the validation logic*
  * **Phân tích:** Logic kiểm tra một ngày có ở tương lai hay không (`event_date > current_date`) là một phép so sánh đơn giản, không đòi hỏi tư duy sâu mở rộng (*extended thinking*). Bật extended thinking cho tác vụ này sẽ làm gia tăng độ trễ (*latency*) và chi phí API không cần thiết.

* **Lựa chọn 4 (Sai):** *Enter plan mode first to create a detailed implementation strategy before making the change*
  * **Phân tích:** Lập chiến lược triển khai chi tiết qua Plan Mode là phản mẫu (*anti-pattern*) đối với các tác vụ hẹp. Đề bài đã cho sẵn chiến lược (thêm conditional check vào hàm hiện có), do đó bước lập kế hoạch bị thừa [664–666].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Direct Execution** | Thực thi trực tiếp | Chế độ cho phép Claude Code chỉnh sửa code/thực thi lệnh ngay lập tức cho các tác vụ phạm vi hẹp, đơn giản [664–666]. |
| **Plan Mode** | Chế độ lập kế hoạch | Chế độ phân tích codebase và lập chiến lược trước khi sửa code, dùng cho các thay đổi lớn/nhiều tệp [664–665]. |
| **Well-Scoped Change** | Thay đổi có phạm vi rõ ràng | Yêu cầu chỉnh sửa đã xác định chính xác vị trí, chức năng và không có nguy cơ xung đột kiến trúc [664–666]. |
| **Extended Thinking Mode** | Chế độ tư duy mở rộng | Chế độ kích hoạt khả năng suy luận chuỗi dài của mô hình cho các bài toán logic hoặc thuật toán phức tạp. |
| **Multi-File Modification** | Chỉnh sửa trên nhiều tệp | Tác vụ tác động đến nhiều module/tệp cùng lúc (kịch bản bắt buộc nên cân nhắc Plan Mode) [664–665]. |
| **Architectural Decision** | Quyết định kiến trúc | Các lựa chọn thiết kế hệ thống cấp cao (như chuyển monolith sang microservices, đổi thư viện cốt lõi) [664–665]. |

---

## Practice Test 2 — Q101: PostToolUse Hook for Deterministic Code Formatting

**Đáp án đúng:**  
**Configure a Post ToolUse hook with an Edit|Write matcher that automatically runs Prettier on each file Claude modifies.** *(Cấu hình một hook PostToolUse với bộ khớp Edit|Write để tự động chạy Prettier trên mỗi tệp mà Claude chỉnh sửa)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)**:

*   **Phân vùng kiến thức chính (Primary Domains):**
    *   **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
    *   **Domain 3: Claude Code Configuration & Workflows** (Trọng số **20%**) [623–624].
*   **Task Statements liên quan:**
    *   **Task Statement 1.5:** Apply Agent SDK hooks for tool call interception and data normalization (Áp dụng các hook trong Agent SDK để can thiệp cuộc gọi công cụ và chuẩn hóa dữ liệu) [639–641].
    *   **Task Statement 1.4:** Implement multi-step workflows with enforcement and handoff patterns [637–639].
*   **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
    *   *Kiến thức (Knowledge of):* 
        > *"The difference between programmatic enforcement (hooks, prerequisite gates) and prompt-based guidance for workflow ordering"*.
        > *"When deterministic compliance is required ..., prompt instructions alone have a non-zero failure rate"* [637–638].
        > *"The distinction between using hooks for deterministic guarantees versus relying on prompt instructions for probabilistic compliance"* [640–641].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1.  **Quy tắc "Tuân thủ định mệnh (Deterministic) v.s Tuân thủ xác suất (Probabilistic)":**
    *   **Hướng dẫn qua Prompt (Prompt Instructions / Rules / Skills):** Mọi câu lệnh trong `CLAUDE.md`, `.claude/rules/`, hay `SKILL.md` dù có nhấn mạnh bằng chữ hoa (`IMPORTANT: MUST`) đều mang tính **xác suất (probabilistic)** và luôn có tỷ lệ thất bại nhất định [637–638, 640–641].
    *   **Thực thi bằng Lập trình (Programmatic Enforcement / Hooks):** Khi một quy chuẩn yêu cầu độ chính xác tuyệt đối 100% (như định dạng code, kiểm tra an ninh, hoặc xác thực danh tính), cách duy nhất là dùng **Hook / Prerequisite Gate** để thực thi bằng code tự động [637–641].
2.  **Kỹ thuật chặn PostToolUse cho Edit/Write:** Khi Claude thực hiện hành động ghi hoặc sửa tệp (`Edit` hoặc `Write`), hook `PostToolUse` sẽ can thiệp ngay sau đợt gọi công cụ đó để chạy công cụ định dạng code (như Prettier) trực tiếp trên tệp thô, đảm bảo 100% code lưu vào đĩa đều đúng định dạng [639–640].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng (Lựa chọn 1):** **Configure a Post ToolUse hook with an Edit|Write matcher that automatically runs Prettier on each file Claude modifies.**
    *   **Phân tích:** Đây là thiết kế kiến trúc chuẩn xác nhất theo tài liệu của Anthropic [639–641]. Bằng cách sử dụng hook `PostToolUse` gắn với các công cụ chỉnh sửa tệp (`Edit|Write`), hệ thống sẽ định dạng code một cách tự động và mang tính **chắc chắn 100% (deterministic guarantee)**, loại bỏ hoàn toàn các lỗi thiếu dấu phẩy hay sai khoảng cách thụt lề mà không phụ thuộc vào việc Claude có "nhớ" quy tắc hay không [637–641].

*   **Lựa chọn 2 (Sai):** *Split the formatting rules into path-scoped .claude/rules/ files that load when Claude works on matching file types.*
    *   **Phân tích:** Việc chuyển quy tắc vào tệp `.claude/rules/` giúp thu hẹp phạm vi nạp ngữ cảnh theo đường dẫn tệp (*path-scoped rules*) [662–664], nhưng bản chất của nó vẫn là hướng dẫn bằng văn bản tự nhiên (*prompt instructions*). Nó chỉ mang tính xác suất và đề bài đã chứng minh việc bổ sung chỉ dẫn không thể triệt tiêu hoàn toàn 15% lỗi còn lại [637–638].

*   **Lựa chọn 3 (Sai):** *Add a Stop hook with a prompt-based check that evaluates whether generated code follows formatting standards and prompts Claude to fix violations.*
    *   **Phân tích:** Sử dụng hook kiểm tra dựa trên prompt (*prompt-based check*) ở cuối luồng (`Stop hook`) sẽ bắt Claude phải tự đánh giá lại và sinh lại code. Cách này vừa tốn kém token/thời gian, vừa dựa vào sự tự đánh giá mang tính xác suất của LLM, không thể đảm bảo độ tin cậy tuyệt đối bằng việc chạy trực tiếp công cụ Prettier [637–638, 684–685].

*   **Lựa chọn 4 (Sai):** *Extract the formatting rules into a dedicated skill that Claude loads automatically when generating code, with more detailed examples of correct formatting.*
    *   **Phân tích:** Việc đưa quy tắc vào tệp `SKILL.md` kèm ví dụ chi tiết (Few-shot) rất tốt để định hình phong cách viết code [660–662, 674–676], nhưng nó vẫn thuộc nhóm hướng dẫn prompt (*prompt guidance*). Bất kỳ phương pháp prompt nào cũng không thể thay thế cho một hook lập trình cứng khi cần sự tuân thủ tuyệt đối [637–641].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Deterministic Guarantee** | Bảo đảm mang tính định mệnh / Tuyệt đối | Sự đảm bảo chắc chắn 100% nhờ việc thực thi bằng code/hook thay vì dựa vào xác suất sinh văn bản của LLM [637–641]. |
| **Probabilistic Compliance** | Tuân thủ mang tính xác suất | Mức độ tuân thủ dựa trên hướng dẫn prompt, luôn có tỷ lệ sai số hoặc bỏ sót nhỏ [637–638, 640–641]. |
| **PostToolUse Hook** | Hook sau khi dùng công cụ | Lớp can thiệp kỹ thuật kích hoạt ngay sau khi một công cụ (như Edit/Write) thực thi xong [639–640]. |
| **Tool Matcher (`Edit\|Write`)** | Bộ khớp công cụ | Cấu hình lọc để chỉ kích hoạt hook khi các công cụ chỉnh sửa tệp cụ thể được gọi [639–640]. |
| **Path-Scoped Rules** | Quy tắc phân vùng theo đường dẫn | Các tệp quy tắc trong `.claude/rules/` chỉ nạp vào ngữ cảnh khi thao tác với đường dẫn khớp tệp [662–664]. |
| **Programmatic Enforcement** | Thực thi bằng lập trình | Việc dùng mã nguồn, giao thức hoặc hooks để bắt buộc hệ thống tuân thủ quy tắc kinh doanh [637–639]. |


## Practice Test 2 — Q102: Interview Pattern for Unfamiliar Caching Requirements

**Đáp án đúng:**  
**Ask Claude to interview you about the caching requirements before implementing, surfacing considerations like invalidation strategies, cache layers, consistency guarantees, and failure modes.** *(Yêu cầu Claude phỏng vấn bạn về các yêu cầu caching trước khi triển khai, làm nổi bật các cân nhắc như chiến lược xóa cache (invalidation), các lớp cache, bảo đảm tính nhất quán, và các chế độ thất bại (failure modes))*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này trích ra trực tiếp từ bài thi mẫu và chuẩn kiến thức **Claude Certified Architect – Foundations Exam Guide**:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 3: Claude Code Configuration & Workflows** (Trọng số **20%**) [623–624].
* **Task Statement:** **Task Statement 3.5: Apply iterative refinement techniques for progressive improvement** [666–668].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức (Knowledge of):* 
    > *"The interview pattern: having Claude ask questions to surface considerations the developer may not have anticipated before implementing"*.
  * *Kỹ năng (Skills in):* 
    > *"Using the interview pattern to surface design considerations (e.g., cache invalidation strategies, failure modes) before implementing solutions in unfamiliar domains"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nhận diện Mẫu thiết kế "Mô hình phỏng vấn" (The Interview Pattern):** Khi lập trình viên phải thực hiện một tác vụ trong một mảng công nghệ hoàn toàn mới hoặc chưa có nhiều kinh nghiệm thực chiến (*unfamiliar domain* — ví dụ: thiết kế caching cho môi trường production lần đầu), kỹ thuật tinh chỉnh hiệu quả nhất là yêu cầu Claude đóng vai trò người phỏng vấn (*interview pattern*).
2. **Lợi ích của Interview Pattern:** Bằng cách để Claude đặt câu hỏi "phỏng vấn ngược" trước khi viết code, hệ thống giúp khơi gợi và làm lộ diện các biến số, rủi ro kiến trúc mà lập trình viên chưa lường trước được (ví dụ: *Chiến lược làm tươi cache khi sản phẩm thay đổi giá là gì? Xử lý thế nào khi cạn bộ nhớ Redis hoặc bị rớt kết nối?*).
3. **Phân biệt với các kỹ thuật tinh chỉnh khác trong Domain 3:**
   * Nếu bài toán thiếu ví dụ code cụ thể \\(\rightarrow\\) Dùng **Few-Shot / Concrete Examples** [667–668, 674–675].
   * Nếu lập trình viên **mới làm mảng này lần đầu và chưa rõ các câu hỏi kiến trúc cần trả lời** \\(\rightarrow\\) Dùng **Interview Pattern**.
   * Nếu bài toán cần phân tích cấu trúc phụ thuộc của nhiều tệp code cồng kềnh \\(\rightarrow\\) Dùng **Plan Mode** [664–666].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 4):** **Ask Claude to interview you about the caching requirements before implementing, surfacing considerations like invalidation strategies, cache layers, consistency guarantees, and failure modes.**
  * *Phân tích:* Đây là thiết kế kiến trúc chuẩn xác tuyệt đối, khớp nguyên văn kịch bản được Anthropic mô tả trong Task Statement 3.5. Việc yêu cầu Claude phỏng vấn giúp chốt hạ toàn bộ các ràng buộc về tính nhất quán, làm tươi dữ liệu và xử lý sự cố trước khi bắt đầu sinh mã nguồn.

* **Lựa chọn 1 (Sai):** *Write a specification with your known requirements and "TBD" markers for uncertain areas, having Claude propose solutions for each TBD as it implements.*
  * *Phân tích:* Việc viết bản tả với các đánh dấu "TBD" (To Be Determined) rồi để Claude tự đề xuất trong lúc viết code sẽ khiến kiến trúc bị thay đổi lặt vặt liên tục, dễ dẫn đến các giả định thiết kế sai lệch so với nhu cầu thực tế của hệ thống.

* **Lựa chọn 2 (Sai):** *Use plan mode to analyze the current /products endpoint implementation, then provide your caching requirements once Claude explains how the existing code is structured.*
  * *Phân tích:* Plan mode giúp phân tích cấu trúc code hiện tại [664–666], nhưng ở kịch bản này vấn đề cốt lõi không phải là do lập trình viên không hiểu code của `/products`, mà là do họ **thiếu kinh nghiệm về các quy chuẩn kiến trúc của Production Caching** (như cache invalidation, consistency).

* **Lựa chọn 3 (Sai):** *Start with a minimal request: "Add Redis caching to /products with 5-minute TTL." Add features and fix issues through follow-up prompts as problems surface during testing.*
  * *Phân tích:* Đây là phương pháp thử-sai bị động (*trial-and-error*). Trong môi trường production caching, việc thiếu các cơ chế như xử lý rớt kết nối hay xóa cache khi dữ liệu thay đổi có thể gây ra sự cố nghiêm trọng (dữ liệu sai lệch, trôi thông tin) khi lên hệ thống thật mới phát hiện ra.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Interview Pattern** | Mẫu thiết kế phỏng vấn | Kỹ thuật yêu cầu Claude đóng vai người phỏng vấn để đặt câu hỏi, giúp khơi gợi các yêu cầu/rủi ro chưa lường trước. |
| **Unfamiliar Domain** | Lĩnh vực / Mảng công nghệ chưa quen thuộc | Tình huống lập trình viên thực hiện tác vụ ở mảng họ chưa có kinh nghiệm thực chiến (ví dụ: production caching). |
| **Cache Invalidation Strategy** | Chiến lược xóa / Làm tươi bộ nhớ đệm | Cơ chế quyết định khi nào dữ liệu trong cache bị hủy hoặc cập nhật lại để tránh sai lệch dữ liệu. |
| **Consistency Guarantees** | Cam kết tính nhất quán dữ liệu | Mức độ đảm bảo dữ liệu trong cache khớp 100% với dữ liệu gốc trong cơ sở dữ liệu. |
| **Failure Modes / Fallback** | Chế độ xử lý sự cố / Dự phòng | Logic xử lý khi lớp caching bị sập (ví dụ: tự động truy vấn trực tiếp xuống DB gốc mà không làm ngắt ứng dụng). |
| **Iterative Refinement** | Tinh chỉnh lặp lại | Chuỗi kỹ thuật cải thiện dần chất lượng output qua các bước kiểm thử, ví dụ mẫu hoặc phỏng vấn [666–668]. |

---

## Practice Test 2 — Q103: Concrete Input-Output Examples for Edge Cases

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này trích ra trực tiếp từ bài thi mẫu và chuẩn kiến thức **Claude Certified Architect – Foundations Exam Guide** [617–618]:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 3: Claude Code Configuration & Workflows** (Trọng số **20%**) [623–624].
* **Task Statement:** **Task Statement 3.5: Apply iterative refinement techniques for progressive improvement** [666–669].
* **Trích dẫn chuẩn từ Exam Guide Blueprint:**
  * *Kiến thức (Knowledge of):* 
    > *"Concrete input/output examples as the most effective way to communicate expected transformations when prose descriptions are interpreted inconsistently"* [666–667].
  * *Kỹ năng (Skills in):* 
    > *"Providing specific test cases with example input and expected output to fix edge case handling (e.g., null values in migration scripts)"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Kỹ thuật "Test Case với Dữ liệu Cụ thể" (Concrete Input/Output Examples):** Khi mô hình sinh mã nguồn gặp lỗi xử lý các trường hợp biên (*edge cases* như dữ liệu `null`, chuỗi rỗng, định dạng lạ), phương pháp tinh chỉnh lặp lại hiệu quả nhất là cung cấp một **test case cụ thể chứa dữ liệu mẫu đầu vào (example input)** và **kết quả đầu ra mong đợi (expected output)** [666–668].
2. **Loại bỏ Diễn giải Văn bản Tự nhiên Mơ hồ (Prose Descriptions):** Việc mô tả lỗi bằng văn bản tự nhiên dài dòng thường khiến LLM diễn giải không nhất quán hoặc tiếp tục bỏ sót khi được yêu cầu sinh lại mã nguồn [666–667]. 
3. **Tránh Bẫy "Tạo lại từ đầu" (Regenerate Everything):** Yêu cầu mô hình tạo lại toàn bộ script (*regenerate the entire script*) hay viết lại từ đầu (*complete rewrite*) khi chỉ gặp lỗi ở một trường hợp biên cụ thể là một phản mẫu gây lãng phí token, tăng độ trễ và dễ làm mất các phần logic đã chạy đúng trước đó [668–669].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 2):** **Provide a test case with example input containing null values and the expected output, then ask Claude to fix it.**
  * **Phân tích:** Đây là thiết kế tinh chỉnh lặp lại tối ưu nhất theo nguyên văn hướng dẫn của Anthropic trong Task Statement 3.5. Bằng cách cung cấp một ví dụ dữ liệu `null` cụ thể cùng kết quả đầu ra chính xác, Claude sẽ nắm bắt ngay lập tức quy tắc xử lý trường hợp biên mà không phải đoán mò, giúp sửa lỗi nhanh chóng và chính xác [666–668].

* **Lựa chọn 1 (Sai):** *Manually edit the generated code to fix the null handling, then continue working with Claude on other parts.*
  * **Phân tích:** Việc tự sửa thủ công đoạn code bị lỗi triệt tiêu lợi ích tự động hóa của tác tử. Quan trọng hơn, nếu bạn tự sửa mà không phản hồi lại cho Claude, mô hình trong các lượt thoại tiếp theo sẽ không nhận biết được logic xử lý `null` mới, dẫn đến nguy cơ tái phát lỗi (*regressions*) khi làm việc ở các mô-đun khác [643–644, 668].

* **Lựa chọn 3 (Sai):** *Describe the null value problem in detail and ask Claude to regenerate the entire script with improved edge case handling.*
  * **Phân tích:** Mô tả bằng văn bản tự nhiên đơn thuần (*prose description*) thường bị LLM diễn giải thiếu nhất quán so với việc đưa ví dụ code/dữ liệu thực tế [666–667]. Đồng thời, yêu cầu tạo lại toàn bộ script (*regenerate the entire script*) gây lãng phí chi phí token không cần thiết [668–669].

* **Lựa chọn 4 (Sai):** *Add "think harder about edge cases" to your prompt and request a complete rewrite of the migration logic.*
  * **Phân tích:** Các câu lệnh giục mô hình suy nghĩ chung chung như "think harder" mang tính mơ hồ, không cung cấp thêm bất kỳ ngữ cảnh hay ràng buộc kỹ thuật rõ ràng nào [671–673]. Việc yêu cầu viết lại toàn bộ logic chuyển đổi dữ liệu (*complete rewrite*) vừa lãng phí tài nguyên vừa gây ra rủi ro sinh thêm lỗi mới [666–668].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Edge Case Handling** | Xử lý trường hợp biên | Logic lập trình dùng để xử lý các tình huống dữ liệu đặc biệt/hiếm gặp (như `null`, `undefined`, chuỗi rỗng). |
| **Input/Output Example** | Ví dụ đầu vào / đầu ra | Kỹ thuật đưa ra cặp dữ liệu mẫu và kết quả kỳ vọng để hướng dẫn mô hình tinh chỉnh code chính xác [666–668]. |
| **Data Migration Script** | Kịch bản chuyển đổi dữ liệu | Đoạn mã dùng để di chuyển và chuẩn hóa dữ liệu giữa các cơ sở dữ liệu hoặc hệ thống lưu trữ. |
| **Iterative Refinement** | Tinh chỉnh lặp lại | Chuỗi kỹ thuật cải thiện chất lượng output theo từng bước thông qua test case, sửa lỗi mục tiêu hoặc ví dụ mẫu [666–668]. |
| **Prose Description** | Diễn giải bằng văn bản tự nhiên | Việc mô tả yêu cầu/lỗi bằng câu văn thông thường (kém hiệu quả hơn ví dụ cụ thể khi sửa lỗi biên) [666–667]. |

---

## Practice Test 2 — Q104: Parallel Subagent Calls in a Single Coordinator Response

**Đáp án đúng:**  
**Structure the coordinator to emit both Task tool calls (for web search and document analysis) in a single response message.** *(Cấu hình tác tử điều phối để phát ra cả hai cuộc gọi công cụ Task (cho tìm kiếm web và phân tích tài liệu) trong một tin nhắn phản hồi duy nhất)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

*   **Phân vùng kiến thức chính (Primary Domain):** **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
*   **Task Statement:** **Task Statement 1.3: Configure subagent invocation, context passing, and spawning** [635–637].
*   **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
    *   *Kỹ năng trong Task Statement 1.3:*
        > *"Spawning parallel subagents by emitting multiple Task tool calls in a single response message rather than across separate turns"*.
    *   *Kỹ năng thực hành trong Exercise 4:*
        > *"Implement parallel subagent execution by having the coordinator emit multiple Task tool calls in a single response. Measure the latency improvement compared to sequential execution"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1.  **Cơ chế thực thi song song Subagent trong Claude Agent SDK:** Khi một tác tử điều phối (*coordinator*) cần gọi nhiều subagents độc lập với nhau (không phụ thuộc dữ liệu đầu ra của nhau), cơ chế chuẩn để chạy song song là bắt buộc mô hình phát ra (*emit*) nhiều cuộc gọi công cụ `Task` cùng lúc trong **một tin nhắn phản hồi duy nhất (single response message)**.
2.  **Nhận diện nút thắt tuần tự (Sequential Bottleneck):** Nếu coordinator phát ra 1 tool call `Task` ở lượt 1, đợi kết quả, rồi mới phát ra tool call `Task` tiếp theo ở lượt 2, hệ thống đang bị chạy tuần tự qua nhiều lượt (*across separate turns*), làm tăng gấp đôi độ trễ (*latency*).
3.  **Bác bỏ các giải pháp Over-Engineering và Prompting suông:**
    *   Thêm lời giải thích về "lợi ích hiệu năng" trong system prompt không thể thay thế cho việc hướng dẫn cấu trúc emit tool calls.
    *   Tự xây dựng một lớp bất đồng bộ phức tạp (*async orchestration layer*) bên ngoài agent để chạy nhiều thread là giải pháp làm cồng kềnh hạ tầng không cần thiết khi Agent SDK đã hỗ trợ sẵn việc này.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng (Lựa chọn 3):** **Structure the coordinator to emit both Task tool calls (for web search and document analysis) in a single response message.**
    *   *Phân tích:* Đây là phương pháp thiết kế kiến trúc chuẩn xác nhất theo tài liệu của Anthropic. Bằng cách cấu hình coordinator để xuất ra cả hai cuộc gọi công cụ `Task` (`web search` và `document analysis`) trong cùng một tin nhắn phản hồi, SDK sẽ tự động kích hoạt thực thi hai subagents này song song, giảm thiểu tối đa tổng thời gian thực thi.

*   **Lựa chọn 1 (Sai):** *Add detailed instructions to the coordinator's system prompt explaining the performance benefits of parallel execution at the same time.*
    *   *Phân tích:* Diễn giải lý thuyết về "lợi ích hiệu năng" trong system prompt chỉ mang tính định hướng tổng quan. Nếu không hướng dẫn rõ ràng cấu trúc để mô hình phát ra nhiều cuộc gọi `Task` trong cùng 1 response, Claude vẫn sẽ duy trì thói quen gọi từng công cụ theo lượt tuần tự.

*   **Lựa chọn 2 (Sai):** *Switch both subagents to use a Haiku-tier model instead of Sonnet to reduce their individual execution time.*
    *   *Phân tích:* Chuyển sang mô hình Haiku có thể làm giảm thời gian phản hồi của từng subagent đơn lẻ, nhưng **hoàn toàn không giải quyết được nút thắt kiến trúc** là hai tác vụ bị thực thi nối tiếp nhau. Vấn đề cốt lõi ở đây là luồng thực thi tuần tự chứ không phải tốc độ xử lý của từng mô hình.

*   **Lựa chọn 4 (Sai):** *Create an async orchestration layer outside the agent that spawns parallel threads, each running a separate coordinator.*
    *   *Phân tích:* Đây là bẫy làm cồng kềnh hệ thống (*over-engineering*). Việc tạo thêm lớp đa luồng (*multi-threading*) chạy nhiều coordinator riêng biệt bên ngoài làm phá hỏng mô hình quản lý tập trung Hub-and-Spoke và làm mất khả năng tổng hợp kết quả của coordinator chính [632–634, 636].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Parallel Subagent Spawning** | Khởi tạo subagent song song | Kỹ thuật kích hoạt nhiều subagents chạy đồng thời trong cùng một đợt gọi API. |
| **Single Response Message** | Một tin nhắn phản hồi duy nhất | Một lượt thoại (turn) duy nhất mà trong đó Claude phát ra nhiều yêu cầu tool calls cùng lúc. |
| **Task Tool** | Công cụ Task | Công cụ built-in trong Agent SDK được dùng để khởi tạo và giao việc cho subagent [635–636]. |
| **Sequential Bottleneck** | Nút thắt thực thi tuần tự | Sự cố hiệu năng do chạy lần lượt từng subagent qua nhiều lượt thoại riêng biệt. |
| **Independent Tasks** | Các tác vụ độc lập | Các công việc không đòi hỏi đầu ra của tác vụ này làm đầu vào cho tác vụ kia (điều kiện lý tưởng để chạy song song). |
| **Coordinator Agent** | Tác tử điều phối | Tác tử trung tâm đảm nhận việc phân rã công việc, giao việc cho subagents và tổng hợp kết quả [632–634]. |


## Practice Test 2 — Q105: Multi-Pass Review to Prevent Output Truncation

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
  * **Domain 4: Prompt Engineering & Structured Output** (Trọng số **20%**) [623–624].
* **Task Statements liên quan:**
  * **Task Statement 1.6:** Design task decomposition strategies for complex workflows [641–643].
  * **Task Statement 4.6:** Design multi-instance and multi-pass review architectures [684–686].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 1.6 & 4.6:*
    > *"Prompt chaining patterns that break reviews into sequential steps (e.g., analyze each file individually, then run a cross-file integration pass)"* [641–642].
    > *"Multi-pass review: splitting large reviews into per-file local analysis passes plus cross-file integration passes to avoid attention dilution..."* [684–685].
  * *Câu hỏi mẫu tương tự (Sample Question 12):*
    > *"Splitting reviews into focused passes directly addresses the root cause: attention dilution when processing many files at once. File-by-file analysis ensures consistent depth..."* [747–748].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nhận diện Sự cố Tràn token đầu ra (`max_tokens limit` truncation):** Khi chạy tác vụ review tự động trên một Pull Request lớn (30+ tệp) bằng **1 đợt gọi API duy nhất (*single-pass review*)**, lượng dữ liệu phản hồi trả về vượt quá giới hạn token đầu ra tối đa (`max_tokens`), khiến chuỗi JSON bị đứt đoạn giữa chừng và làm hỏng bộ phân tích (*parser*) của pipeline.
2. **Quy tắc "Chia để trị" (Task Decomposition / Multi-pass Architecture):**
   * Giải pháp triệt để và mang tính kiến trúc hệ thống duy nhất là **chia nhỏ PR thành nhiều đợt gọi API** (ví dụ: review theo từng tệp hoặc từng nhóm 3–5 tệp) [641–643, 685].
   * Backend sẽ thu thập kết quả từ từng đợt gọi API nhỏ rồi hợp nhất các mảng phát hiện (`findings arrays`) lại với nhau trước khi trả kết quả cuối cùng [641–643, 747–748].
3. **Bác bỏ các giải pháp tình thế / đối phó:**
   * **Khống chế từ ngữ bằng Prompt:** Yêu cầu mô hình viết dưới 50 từ hay chỉ báo lỗi nặng vẫn có nguy cơ vượt trần `max_tokens` khi gặp PR có quá nhiều lỗi.
   * **Bỏ `tool_use` sang Markdown:** Làm mất đi khả năng định dạng đầu ra có cấu trúc chuẩn (*structured output*) của `tool_use`, khiến code backend càng khó phân tích dữ liệu tự động [676–678].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Split the review into multiple API calls that each analyze a subset of the changed files, then merge the resulting findings arrays.**
  * *Phân tích:* Đây là giải pháp kiến trúc tối ưu nhất được Anthropic khuyến nghị [641–643, 685, 747–748]. Việc phân rã PR 30+ tệp thành nhiều đợt gọi API xử lý các tập hợp tệp nhỏ hơn (*subset of files*) giúp kiểm soát tuyệt đối dung lượng token đầu ra không vượt quá `max_tokens`, bảo đảm chuỗi JSON luôn hoàn chỉnh không bị cắt đứt. Đồng thời, phương pháp này còn giúp nâng cao chất lượng review nhờ tránh được hiện tượng phân tán sự chú ý (*attention dilution*).

* **Lựa chọn 1 (Sai):** *Add retry logic that detects truncated JSON and re-sends the request with instructions to report only critical and high severity findings.*
  * *Phân tích:* Đây là cách xử lý mang tính đối phó. Việc yêu cầu Claude chỉ báo lỗi nghiêm trọng không bảo đảm chuỗi JSON sẽ đủ ngắn khi gặp các PR cực kỳ lớn, đồng thời làm bỏ sót các lỗi tiềm ẩn ở mức độ trung bình và nhỏ trong codebase.

* **Lựa chọn 2 (Sai):** *Increase max_tokens to the model's maximum and instruct Claude to keep finding descriptions under 50 words each.*
  * *Phân tích:* Dù có tăng `max_tokens` lên mức tối đa của mô hình, nếu tổng số lượng lỗi phát hiện trên 30+ tệp vượt quá trần output, phản hồi vẫn sẽ bị đứt đoạn. Bên cạnh đó, các hướng dẫn khống chế số từ bằng prompt tự nhiên (*under 50 words*) chỉ mang tính xác suất (*probabilistic*) và không bảo đảm tuân thủ 100% [637–638, 666–667].

* **Lựa chọn 4 (Sai):** *Switch from tool_use to prompting Claude to return findings as a markdown list.*
  * *Phân tích:* Mất đi tính năng `tool_use` đồng nghĩa với việc mất đi sự bảo đảm về đầu ra có cấu trúc chuẩn (*JSON Schema enforcement*) [676–678]. Bảng danh sách Markdown vẫn có thể bị cắt ngang do tràn `max_tokens`, đồng thời khiến việc phân tích cú pháp tự động trên pipeline CI/CD trở nên thiếu tin cậy.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Output Truncation** | Cắt ngang đầu ra | Hiện tượng văn bản/JSON trả về bị đứt đoạn giữa chừng do chạm trần `max_tokens`. |
| **Single-Pass Review** | Review đơn lượt | Việc nạp toàn bộ danh sách tệp thay đổi vào 1 đợt gọi API duy nhất (dễ gây tràn token và suy giảm chất lượng review) [746–748]. |
| **Multi-Pass Review** | Review đa lượt | Kỹ thuật chia nhỏ tác vụ review thành nhiều lượt gọi API cho từng nhóm tệp nhỏ rồi tổng hợp kết quả [641–643, 685]. |
| **Task Decomposition** | Phân rã tác vụ | Phương pháp chia một công việc lớn thành các sub-task nhỏ hơn để tránh quá tải ngữ cảnh/token [641–643]. |
| **Attention Dilution** | Phân tán sự chú ý | Hiện tượng mô hình giảm độ sâu và độ chính xác khi phải xử lý quá nhiều tệp/thông tin cùng lúc trong prompt. |
| **Structured Output Enforcement** | Bắt buộc đầu ra có cấu trúc | Kỹ thuật dùng `tool_use` và JSON Schema để đảm bảo dữ liệu trả về đúng định dạng máy có thể đọc được [676–678]. |

---

## Practice Test 2 — Q106: Grep Distinctive Error Text Before Reading Files
**Đáp án đúng:**  
**Use Grep to search for distinctive text from the error message (like "SYNC_CONFLICT" or "entity version mismatch"), then Read the matching files to understand context.** *(Sử dụng Grep để tìm kiếm chuỗi văn bản đặc trưng từ thông báo lỗi (như "SYNC_CONFLICT" hoặc "entity version mismatch"), sau đó dùng Read để đọc các tệp khớp kết quả nhằm hiểu ngữ cảnh)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) kết hợp với **Domain 3: Claude Code Configuration & Workflows** (Trọng số **20%**) [623–624].
* **Task Statement:** **Task Statement 2.5: Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively** [655–657].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 2.5:*
    > *"Grep for content search (searching file contents for patterns like function names, error messages, or import statements)"* [655–656].
    > *"Glob for file path pattern matching (finding files by name or extension patterns)"* [655–656].
  * *Kỹ năng trong Task Statement 2.5:*
    > *"Selecting Grep for searching code content across a codebase (e.g., finding all callers of a function, locating error messages)"* [656–657].
    > *"Building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront"* [656–657].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Phân biệt triệt để giữa `Grep` và `Glob`:**
   * **`Grep` (Content Search):** Dùng để tìm kiếm **nội dung bên trong tệp** (chuỗi văn bản, từ khóa, tên hàm, thông báo lỗi, câu lệnh import) [655–657].
   * **`Glob` (Path Search):** Chỉ dùng để tìm kiếm **tên tệp hoặc cấu trúc đường dẫn** theo định dạng mẫu (ví dụ: `**/*.test.ts`, `src/errors/*`) [655–657].
2. **Nguyên tắc Khám phá Tăng tiến (Incremental Exploration):** Bắt đầu bằng `Grep` để tìm chính xác điểm xuất hiện của chuỗi văn bản độc nhất, sau đó mới dùng `Read` mở đúng tệp chứa kết quả đó [656–657].
3. **Quy tắc bác bỏ các giải pháp lãng phí Token:**
   * **Không đọc tài liệu/tệp tràn lan ngay từ đầu:** Đọc README hoặc duyệt từng tệp nguồn trong 12 dịch vụ (`Read` thủ công) sẽ làm cạn kiệt cửa sổ ngữ cảnh (*context exhaustion*) và tốn thời gian không cần thiết [656–657].
   * **Truy vấn phải đủ hẹp (*Targeted Search*):** Tìm kiếm chuỗi văn bản đặc trưng (`SYNC_CONFLICT`) tốt hơn nhiều so với việc tìm tất cả các tệp import module xử lý lỗi chung chung.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 3):** **Use Grep to search for distinctive text from the error message (like "SYNC_CONFLICT" or "entity version mismatch"), then Read the matching files to understand context.**
  * *Phân tích:* Đây là chiến lược khám phá codebase chuẩn xác nhất theo tài liệu Anthropic [655–657]. Bằng cách dùng `Grep` quét chuỗi văn bản độc nhất `"SYNC_CONFLICT"`, hệ thống sẽ định vị chính xác vị trí tệp và dòng code tạo ra lỗi trong 12 dịch vụ chỉ sau 1 lần tìm kiếm, sau đó mới dùng `Read` đọc đúng tệp đó để hiểu ngữ cảnh [656–657].

* **Lựa chọn 1 (Sai):** *Read the project's README and service configuration files to understand the architecture, then systematically Read source files in service directory.*
  * *Phân tích:* Đây là phản mẫu (*anti-pattern*) trong việc khám phá codebase. Đọc thủ công README và các tệp mã nguồn thuộc 12 dịch vụ sẽ ngốn hàng trăm ngàn token, gây phân tán sự chú ý (*Attention Dilution*) và tốn chi phí API rất lớn khi đã có sẵn từ khóa tìm kiếm [656–657, 688–689].

* **Lựa chọn 2 (Sai):** *Use Glob to find files in directories commonly associated with error handling (such as errors/, exceptions/, or handlers/) across services, then Read each matching file.*
  * *Phân tích:* Lựa chọn này vi phạm nguyên tắc sử dụng công cụ: `Glob` chỉ tìm theo cấu trúc tên đường dẫn chứ không đọc được nội dung tệp [655–656]. Ngoài ra, chuỗi lỗi `"SYNC_CONFLICT"` có thể được khai báo trực tiếp trong một file logic nghiệp vụ (như `order_service.py`) chứ không nhất thiết nằm trong thư mục `errors/`, dẫn đến việc `Glob` bỏ sót kết quả [655–657].

* **Lựa chọn 4 (Sai):** *Use Grep to find all files that import the project's error handling module, then Read those files to locate custom error definitions.*
  * *Phân tích:* Mặc dù `Grep` đã được sử dụng đúng công cụ, nhưng **mức độ thu hẹp truy vấn quá kém**. Hầu hết mọi tệp trong 12 dịch vụ đều import module xử lý lỗi, dẫn đến việc `Grep` trả về hàng trăm tệp và bắt Claude phải dùng `Read` từng tệp một cách không cần thiết, làm mất đi lợi thế của chuỗi từ khóa độc nhất `"SYNC_CONFLICT"`.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **`Grep` Tool** | Công cụ Grep | Công cụ tìm kiếm chuỗi ký tự/từ khóa bên trong nội dung tệp thuộc codebase [655–657]. |
| **`Glob` Tool** | Công cụ Glob | Công cụ tìm kiếm tên tệp hoặc đường dẫn tệp theo mẫu wildcard (như `**/*.py`) [655–657]. |
| **Incremental Exploration** | Khám phá tăng tiến | Phương pháp tìm kiếm codebase theo từng bước (Grep \\(\rightarrow\\) Read) thay vì nạp tràn lan ngay từ đầu [656–657]. |
| **Distinctive Text / String Literal** | Chuỗi văn bản đặc trưng / Chuỗi nguyên bản | Từ khóa hoặc thông báo lỗi duy nhất giúp thu hẹp kết quả tìm kiếm xuống mức tối đa. |
| **Codebase Entry Point** | Điểm đầu vào mã nguồn | Vị trí khởi đầu của một luồng logic hoặc hàm/chuỗi lỗi cần truy vết trong dự án [656–657]. |
| **Targeted Query** | Truy vấn có mục tiêu | Câu lệnh tìm kiếm có phạm vi hẹp để tránh làm phình to ngữ cảnh và lãng phí token [656–657]. |


## Practice Test 2 — Q107: Adaptive Test Planning for a Large Legacy Codebase

**Đáp án đúng:**  
**Use Glob and Grep to map codebase structure, identify heavily-coupled modules, create a prioritized plan for high-impact areas, and revise as dependencies are discovered.** *(Sử dụng Glob và Grep để lập bản đồ cấu trúc codebase, xác định các module có độ phụ thuộc cao, tạo kế hoạch ưu tiên cho các vùng có ảnh hưởng lớn và điều chỉnh linh hoạt khi phát hiện ra các phụ thuộc mới)* [642–643, 655–657].

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này nằm trong đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domains):** 
  * **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [623–624].
* **Task Statements liên quan:**
  * **Task Statement 1.6: Design task decomposition strategies for complex workflows** [641–643].
  * **Task Statement 2.5: Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively** [655–657].
* **Trích dẫn chuẩn nguyên văn từ Exam Guide Blueprint:**
  * *Kỹ năng trong Task Statement 1.6:*
    > *"Decomposing open-ended tasks (e.g., 'add comprehensive tests to a legacy codebase') by first mapping structure, identifying high-impact areas, then creating a prioritized plan that adapts as dependencies are discovered"* [642–643].
  * *Kỹ năng trong Task Statement 2.5:*
    > *"Building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront"* [656–657].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nhận diện dạng bài "Phân rã tác vụ mở trên codebase lớn" (Open-ended Task on Large Codebase):** Khi đối mặt với yêu cầu mở rộng (như *"bổ sung test toàn diện cho codebase 200 tệp chưa có test"*), tác tử không bao giờ được cố gắng đọc tất cả các tệp cùng một lúc hay áp dụng lịch trình cố định cứng nhắc [641–643, 656–657].
2. **Quy tắc 3 bước xử lý tác vụ mở:**
   * **Bước 1 (Lập bản đồ nhẹ token):** Dùng `Glob` (tìm cấu trúc thư mục/tệp) và `Grep` (tìm từ khóa/liên kết) để xây dựng bản đồ tổng quan codebase mà không làm phình ngữ cảnh [642–643, 655–657].
   * **Bước 2 (Xác định vùng ảnh hưởng cao):** Định vị các module cồng kềnh, có độ kết nối/phụ thuộc cao (*heavily-coupled modules / high-impact areas*) để đưa vào danh sách ưu tiên viết test trước [642–643].
   * **Bước 3 (Thích ứng linh hoạt):** Tạo kế hoạch ban đầu và cho phép điều chỉnh (*revise*) khi phát hiện thêm các dependency mới trong quá trình thực thi [642–643].
3. **Bác bỏ các cạm bẫy thiết kế (Anti-Patterns):**
   * **Cạn kiệt ngữ cảnh (Context Exhaustion):** Đọc lần lượt toàn bộ 200 tệp (`Read all 200 files`) trước khi viết code là phản mẫu gây tràn token ngữ cảnh [656–657, 688–689].
   * **Lập lịch cứng nhắc (Fixed Schedule):** Phân bổ nguồn lực đều nhau cho mọi thư mục mà không quan tâm đến độ phức tạp hay tầm quan trọng nghiệp vụ là cách làm kém hiệu quả.
   * **Thực thi ngẫu nhiên (Alphabetical Approach):** Viết test theo thứ tự bảng chữ cái hoàn toàn bỏ qua thứ tự ưu tiên của các module cốt lõi [642–643].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Use Glob and Grep to map codebase structure, identify heavily-coupled modules, create a prioritized plan for high-impact areas, and revise as dependencies are discovered.**
  * *Phân tích:* Khớp 100% với hướng dẫn chuẩn trong Task Statement 1.6 và 2.5 của Anthropic [642–643, 655–657]. Sử dụng `Glob` và `Grep` giúp tác tử nhanh chóng nắm bắt cấu trúc dự án 200 tệp một cách tiết kiệm token tối đa, định vị chính xác các module quan trọng có độ phụ thuộc cao để ưu tiên viết test trước, đồng thời giữ cho kế hoạch có tính linh hoạt điều chỉnh khi phát hiện thêm liên kết mới [642–643, 656–657].

* **Lựa chọn 1 (Sai):** *Create a fixed testing schedule upfront based on directory structure, allocating equal effort to each top-level directory regardless of code complexity or business importance.*
  * *Phân tích:* Việc lập lịch cố định (*fixed schedule*) và chia đều công sức cho mọi thư mục cấp cao mà không xem xét độ phức tạp hay tầm quan trọng nghiệp vụ là một phản mẫu. Cách làm này khiến tác tử lãng phí thời gian vào các thư mục phụ ít quan trọng [641–643].

* **Lựa chọn 3 (Sai):** *Systematically read all 200 files to create a complete function inventory before writing any tests, ensuring the testing plan accounts for every function before beginning.*
  * *Phân tích:* Đọc thủ công toàn bộ 200 tệp bằng công cụ `Read` để kiểm kê từng hàm trước khi viết test sẽ dẫn đến sự cố tràn cửa sổ ngữ cảnh (*context window exhaustion*) và ngốn hàng trăm ngàn token không cần thiết [656–657, 688–689]. Chiến lược đúng đắn là khám phá tăng tiến bằng `Glob`/`Grep` [656–657].

* **Lựa chọn 4 (Sai):** *Start writing tests for the first module alphabetically, using test failures and imports to discover related files organically.*
  * *Phân tích:* Viết test theo thứ tự bảng chữ cái (*alphabetically*) là phương pháp ngẫu nhiên, không có chiến lược. Cách này khiến tác tử bị cuốn vào viết test cho các module không quan trọng chỉ vì tên của nó bắt đầu bằng chữ "A", thay vì tập trung vào các khu vực cốt lõi có rủi ro cao của hệ thống [642–643].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Open-Ended Task Decomposition** | Phân rã tác vụ mở | Chiến lược chia nhỏ một công việc rộng, chưa rõ điểm dừng thành các bước định vị, ưu tiên và thực thi linh hoạt [641–643]. |
| **Heavily-Coupled Modules** | Các module có độ phụ thuộc cao | Các module có nhiều liên kết, được gọi bởi nhiều thành phần khác trong hệ thống (cần ưu tiên viết test) [642–643]. |
| **High-Impact Areas** | Các vùng có ảnh hưởng lớn | Những đoạn mã nguồn đóng vai trò cốt lõi hoặc chứa nhiều logic nghiệp vụ quan trọng trong codebase [642–643]. |
| **Adaptive Prioritized Plan** | Kế hoạch ưu tiên có tính thích ứng | Kế hoạch thực thi được xếp theo độ ưu tiên và có khả năng tự điều chỉnh khi phát hiện thêm thông tin mới [642–643]. |
| **Incremental Codebase Mapping** | Lập bản đồ mã nguồn tăng tiến | Kỹ thuật tìm kiếm cấu trúc code theo từng bước dùng `Glob`/`Grep` trước rồi mới `Read` từng tệp cụ thể [656–657]. |
| **Context Window Exhaustion** | Cạn kiệt cửa sổ ngữ cảnh | Hiện tượng nạp quá nhiều tệp code thô làm tràn token ngữ cảnh của mô hình [656–657, 688–689]. |

---

## Practice Test 2 — Q108: MCP Resources for Cross-System Content Catalogs

**Đáp án đúng:**  
**Expose each server's content catalog as MCP resources—issue summaries, documentation hierarchy, database schemas** *(Công khai danh mục nội dung của từng server dưới dạng MCP resources — tóm tắt issue, hệ thống phân cấp tài liệu, sơ đồ cơ sở dữ liệu)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này nằm trong đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [623–624].
* **Task Statement:** **Task Statement 2.4: Integrate MCP servers into Claude Code and agent workflows** [653–655].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 2.4:*
    > *"MCP resources as a mechanism for exposing content catalogs (e.g., issue summaries, documentation hierarchies, database schemas) to reduce exploratory tool calls"* [653–654].
  * *Kỹ năng trong Task Statement 2.4:*
    > *"Exposing content catalogs as MCP resources to give agents visibility into available data without requiring exploratory tool calls"*.
  * *Chủ đề thuộc phạm vi thi (In-Scope Topics):*
    > *"MCP tool and resource design: resources for content catalogs, tools for actions..."*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Phân biệt cốt lõi giữa MCP Tools và MCP Resources:**
   * **MCP Tools:** Dùng cho các **hành động/thao tác** (*actions/side-effects*) hoặc các truy vấn linh hoạt theo tham số.
   * **MCP Resources:** Dùng để cung cấp các **danh mục nội dung/dữ liệu tĩnh hoặc có cấu trúc** (*content catalogs / data views*) như danh sách tóm tắt issue, sơ đồ CSDL, cấu trúc wiki để tác tử có tầm nhìn (*visibility*) ngay từ đầu [653–655, 766].
2. **Nhận diện bài toán "Lãng phí cuộc gọi do khám phá mù" (Exploratory Tool Call Waste):**
   * Khi tác tử phải gọi 8–10 tool liên tiếp chỉ để "ngó nghiêng" xem server có chứa thông tin gì (ví dụ gọi `search_issues`, `run_query` chỉ để dò tên bảng hoặc tên issue), nó sẽ làm ngốn token và tràn context window [653–654, 687].
   * Giải pháp chuẩn của Anthropic là **Expose Content Catalog dưới dạng MCP Resources** giúp LLM "thấy" sẵn bản đồ dữ liệu mà không cần chạy tool khám phá [653–655].
3. **Quy tắc bác bỏ các cạm bẫy thiết kế:**
   * **Gộp Server (Unified Server):** Phá hỏng tính đóng gói module của kiến trúc MCP.
   * **Tool chuẩn bị (`prepare_investigation` tool):** Vẫn bắt LLM gọi Tool (mang tính hành động) thay vì tận dụng cơ chế đọc Resource có sẵn trong chuẩn MCP.
   * **Orchestrator định tuyến từ khóa:** Không giải quyết được các câu hỏi liên hệ thống (*cross-system questions*) yêu cầu thông tin từ cả 3 server cùng lúc.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Expose each server's content catalog as MCP resources—issue summaries, documentation hierarchy, database schemas**
  * *Phân tích:* Khớp 100% với kiến trúc chuẩn của Anthropic [653–655, 766]. Việc expose danh mục dưới dạng MCP Resources cung cấp bức tranh toàn cảnh (*content catalog*) của từng server cho tác tử trước khi thực hiện hành động. Nhờ đó, tác tử không cần thực hiện các cuộc gọi khám phá thử-sai (*exploratory tool calls*), giúp giảm từ 8–10 cuộc gọi xuống các cuộc gọi chính xác, tiết kiệm token và tránh tràn cửa sổ ngữ cảnh [653–655, 687].

* **Lựa chọn 1 (Sai):** *Consolidate all three servers into a unified MCP server with cross-referencing capabilities*
  * *Phân tích:* Việc gộp 3 server độc lập (issue tracker, documentation wiki, database explorer) thành 1 server duy nhất vi phạm nguyên tắc thiết kế phân rã module của MCP [653–654]. Ngoài ra, việc gộp server không giải quyết được gốc rễ vấn đề: LLM vẫn thiếu tầm nhìn về danh mục dữ liệu và vẫn phải thực hiện các cuộc gọi dò tìm nếu dữ liệu không được biểu diễn thành Resources [653–655].

* **Lựa chọn 3 (Sai):** *Add a prepare_investigation tool to each server that accepts a natural language question and returns relevant content summaries*
  * *Phân tích:* Thêm một tool mới vẫn buộc tác tử phải thực thi cuộc gọi tool (*tool call*) để lấy thông tin tổng quan. Cách làm này không tận dụng đúng tính năng cốt lõi của MCP Protocol là tách biệt giữa **Tools (Hành động)** và **Resources (Dữ liệu/Danh mục đọc trực tiếp)** [653–655, 766].

* **Lựa chọn 4 (Sai):** *Add an orchestrator that routes questions to a single server based on keywords*
  * *Phân tích:* Đề bài đặt ra kịch bản là câu hỏi liên hệ thống (*cross-system questions*) như *"Bảng CSDL nào bị ảnh hưởng bởi đợt refactor authentication trong PROJ-1234?"*. Câu hỏi này đòi hỏi truy cập đồng thời cả Issue Tracker (PROJ-1234) và Database Explorer (bảng CSDL). Định tuyến câu hỏi sang duy nhất **1 server** dựa trên từ khóa sẽ làm hệ thống không thể trả lời các câu hỏi liên kết đa hệ thống.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **MCP Resources** | Tài nguyên MCP | Cơ chế của chuẩn MCP dùng để khai báo/đọc danh mục dữ liệu, sơ đồ, tài liệu mà không tạo ra side-effect [653–655, 766]. |
| **MCP Tools** | Công cụ MCP | Các hàm/thao tác thực thi có khả năng thay đổi trạng thái hoặc thực hiện truy vấn linh hoạt theo tham số [653–655, 766]. |
| **Content Catalog** | Danh mục nội dung | Tập hợp thông tin tổng quan (sơ đồ CSDL, danh mục wiki, tóm tắt issue) giúp tác tử hiểu cấu trúc dữ liệu khả dụng [653–655]. |
| **Exploratory Tool Calls** | Cuộc gọi công cụ khám phá | Các lần gọi tool mang tính dò tìm, thử-sai do tác tử thiếu thông tin về dữ liệu hiện có trong hệ thống [653–655]. |
| **Cross-System Questions** | Câu hỏi liên hệ thống | Loại truy vấn phức tạp đòi hỏi thông tin tổng hợp từ nhiều server/dịch vụ khác nhau cùng lúc. |
| **Unified MCP Server** | Server MCP hợp nhất | Phản mẫu thiết kế gom tất cả dịch vụ vào 1 server thay vì giữ cấu trúc MCP độc lập theo miền [653–654]. |

---
## Practice Test 2 — Q109: Coordinator Feedback Loops for Research Coverage Gaps

**Đáp án đúng:**  
**Have the coordinator evaluate synthesis output for gaps, then re-delegate to web search and document analysis with targeted queries before Invoking synthesis again.** *(Cho phép tác tử điều phối đánh giá đầu ra của tác tử tổng hợp để tìm khoảng trống thông tin, sau đó giao việc lại cho các tác tử tìm kiếm web và phân tích tài liệu bằng các truy vấn có mục tiêu trước khi gọi lại bước tổng hợp)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này trích từ bộ kịch bản **Scenario 3: Multi-Agent Research System** trong đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [626–627]:

*   **Phân vùng kiến thức chính (Primary Domain):** **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623, 626–627].
*   **Task Statement:** **Task Statement 1.2: Orchestrate multi-agent systems with coordinator-subagent patterns** [632–635].
*   **Trích dẫn chuẩn từ Exam Guide Blueprint:**
    *   *Kỹ năng chính trong Task Statement 1.2:*
        > *"Implementing iterative refinement loops where the coordinator evaluates synthesis output for gaps, re-delegates to search and analysis subagents with targeted queries, and re-invokes synthesis until coverage is sufficient"*.
    *   *Mô tả thực hành liên quan (Exercise 4):*
        > *"Design and Debug a Multi-Agent Research Pipeline... practice orchestrating subagents, managing context passing, and handling synthesis with coverage gaps"* [714–716].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1.  **Nhận diện Mẫu kiến trúc "Vòng lặp tinh chỉnh lặp lại" (Iterative Refinement Loop):** Khi một tác tử tổng hợp (*synthesis agent*) phát hiện các câu hỏi nghiên cứu chưa được trả lời, cách xử lý chuẩn của hệ thống đa tác tử (*multi-agent architecture*) là quay lại tác tử điều phối (*coordinator*) để thực hiện vòng lặp tinh chỉnh lặp lại.
2.  **Nguyên tắc Mô hình Hub-and-Spoke (Hub-and-Spoke Architecture):**
    *   Tác tử điều phối (*coordinator*) đóng vai trò là "Hub" trung tâm nắm toàn bộ luồng điều khiển, đánh giá khoảng trống thông tin (*coverage gaps*) và phân rã các truy vấn có mục tiêu (*targeted queries*) [632–634].
    *   Các subagent ("Spokes") như `web search` hay `document analysis` chỉ thực hiện công việc được giao và không trực tiếp trao đổi ngữ cảnh với nhau hay tự ý "vượt cấp" [632–634, 651].
3.  **Quy tắc bác bỏ các cạm bẫy thiết kế:**
    *   **Vi phạm phân quyền công cụ (Scope/Tool Misuse):** Cấp trực tiếp công cụ tìm kiếm web cho tác tử tổng hợp là vi phạm nguyên tắc "phân quyền công cụ theo chuyên môn" (*scoped tool access*), khiến tác tử tổng hợp dễ sử dụng sai công cụ.
    *   **Giải pháp thụ động (Passive Limitation):** Ghi chú sự thiếu hụt thông tin vào báo cáo cuối cùng không giúp cải thiện độ hoàn thiện của nghiên cứu.
    *   **Mở rộng tìm kiếm tĩnh ban đầu (Static Broad Search):** Tăng độ rộng truy vấn ngay từ đầu gây lãng phí token và vẫn không thể đảm bảo bao phủ được các khía cạnh ẩn chỉ phát hiện được sau khi đã tổng hợp dữ liệu [634, 641–642].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng (Lựa chọn 1):** **Have the coordinator evaluate synthesis output for gaps, then re-delegate to web search and document analysis with targeted queries before Invoking synthesis again.**
    *   *Phân tích:* Khớp 100% với kỹ năng được yêu cầu trong Task Statement 1.2. Tác tử điều phối nhận tín hiệu báo thiếu hụt từ tác tử tổng hợp, phân tích các khoảng trống, đặt ra các câu hỏi/truy vấn tìm kiếm tập trung (*targeted queries*), giao lại việc cho các subagent thu thập dữ liệu, sau đó mới tổng hợp lại để xuất báo cáo hoàn chỉnh.

*   **Lựa chọn 2 (Sai):** *Give the synthesis agent direct access to web search tools so it can autonomously fill knowledge gaps without returning control to the coordinator.*
    *   *Phân tích:* Vi phạm nguyên tắc thiết kế mô hình Hub-and-Spoke và phân quyền công cụ (*Task Statement 2.3*) [632–633, 650–651]. Trao công cụ `web search` trực tiếp cho tác tử tổng hợp khiến nó hoạt động chồng chéo vai trò, làm mất khả năng quan sát (*observability*) và kiểm soát luồng của tác tử điều phối.

*   **Lựa chọn 3 (Sai):** *Have the report generation agent note which research questions couldn't be answered, so users understand the limitations of the final output.*
    *   *Phân tích:* Đây là giải pháp ghi nhận sự cố thụ động (*passive workaround*). Mặc dù việc minh bạch giới hạn báo cáo là một thực hành tốt khi cạn kiệt nguồn tra cứu, nhưng nó không giải quyết được mục tiêu chính của đề bài là **"cải thiện độ hoàn thiện của nghiên cứu"** (*improve research completeness*).

*   **Lựa chọn 4 (Sai):** *Increase the initial breadth of queries sent to web search and document analysis to reduce the probability of missing relevant information.*
    *   *Phân tích:* Mở rộng truy vấn ngay từ đợt đầu là phương pháp nạp dữ liệu tĩnh không tối ưu. Nó gây bùng nổ số lượng token, phân tán sự chú ý (*attention dilution*) và tốn chi phí API, nhưng vẫn có xác suất cao bỏ sót các góc khuất chuyên sâu mà chỉ có thể phát hiện sau khi hoàn thành lượt tổng hợp ban đầu [634, 641–642, 688].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Iterative Refinement Loop** | Vòng lặp tinh chỉnh lặp lại | Luồng xử lý trong đó coordinator đánh giá kết quả tổng hợp, phát hiện khoảng trống và giao việc tìm kiếm bổ sung trước khi tạo báo cáo cuối cùng. |
| **Coverage Gaps** | Khoảng trống bao phủ thông tin | Các câu hỏi hoặc chủ đề phụ chưa được trả lời đầy đủ do thiếu dữ liệu từ các bước tìm kiếm trước. |
| **Targeted Queries** | Truy vấn có mục tiêu | Các câu lệnh tìm kiếm/truy vấn được thu hẹp phạm vi chính xác vào những phần thông tin còn thiếu. |
| **Hub-and-Spoke Architecture** | Kiến trúc Trục và Nan hoa | Mô hình quản lý tập trung trong đó một tác tử điều phối (Hub) quản lý toàn bộ tương tác giữa các tác tử phụ (Spokes) [632–634]. |
| **Scoped Tool Access** | Phân quyền công cụ theo phạm vi | Việc giới hạn danh sách công cụ cấp cho từng tác tử đúng với chức năng chuyên môn của nó [650–651]. |
| **Synthesis Agent** | Tác tử tổng hợp | Tác tử chuyên trách việc gom nhóm, đối chiếu và hợp nhất các phát hiện từ nhiều nguồn dữ liệu khác nhau [633–635]. |

---

## Practice Test 2 — Q110: Structured Document IDs for Reliable Tool Chaining

**Đáp án đúng:**  
**Structured data containing document IDs and metadata for each result.** *(Dữ liệu có cấu trúc chứa mã định danh tệp (document IDs) và siêu dữ liệu (metadata) cho từng kết quả)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [623–624].
  * **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
* **Task Statements liên quan:**
  * **Task Statement 2.1:** Design effective tool interfaces with clear descriptions and boundaries [645–648].
  * **Task Statement 1.3 & 5.1:** Structured context passing & context management across multi-step workflows [636, 687–689].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kỹ năng trong Task Statement 1.3 & 5.1:*
    > *"Using structured data formats to separate content from metadata (source URLs, document names, page numbers, document IDs) when passing context between agents/tools to enable accurate downstream processing"* [636, 688–689].
    > *"Trimming verbose tool outputs and returning structured data with clear identifiers so downstream tools can reference specific items unambiguously"* [687–689].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Quy tắc "Mã định danh lập trình cho chuỗi công cụ" (Programmatic IDs for Tool Chaining):** Khi một công cụ được dùng trong quy trình nhiều bước (*multi-step workflow*) — tức là kết quả của Công cụ A được nạp lại cho Claude để gọi tiếp Công cụ B — đầu ra của Công cụ A bắt buộc phải trả về **dữ liệu có cấu trúc chứa ID định danh độc nhất (`document_id`)** [636, 687–689].
2. **Khác biệt giữa UI người dùng và API Tác tử:**
   * Văn bản tự nhiên (*human-readable strings*) hoặc đường liên kết Web (*URLs*) được thiết kế cho con người xem trên giao diện.
   * Dữ liệu JSON có cấu trúc chứa `id` và `metadata` được thiết kế dành riêng cho LLM Agent để truyền tham số chính xác vào các hàm API xử lý ở bước sau (ví dụ: `get_document_details(doc_id="doc_123")`) [636, 676–678].
3. **Triệt tiêu sự mơ hồ (Ambiguity Elimination):** Nếu chỉ trả về tiêu đề tệp ("Q2 Budget Proposal"), hệ thống có thể gặp lỗi nếu trong cơ sở dữ liệu có nhiều tệp trùng tên hoặc tiêu đề bị thay đổi. Mã `document_id` cố định sẽ triệt tiêu hoàn toàn rủi ro này.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng:** **Structured data containing document IDs and metadata for each result.**
  * *Phân tích:* Khớp 100% với nguyên tắc thiết kế công cụ của Anthropic [636, 687–689]. Việc trả về JSON/Object có cấu trúc chứa `document_id` và các trường `metadata` giúp Claude nhận biết chính xác từng tài xế/tài liệu và truyền trực tiếp mã `id` đó vào các cuộc gọi công cụ tiếp theo một cách đáng tin cậy.

* **Lựa chọn 2 (Sai):** *URLs that users can click to open the document in their browser.*
  * *Phân tích:* Các đường dẫn URL cho trình duyệt chỉ phục vụ hiển thị trên giao diện người dùng (UI), hoàn toàn không giúp tác tử (Agent) trích xuất mã định danh để gọi các công cụ backend tiếp theo trong quy trình tự động.

* **Lựa chọn 3 (Sai):** *More detailed human-readable descriptions including the size and authors.*
  * *Phân tích:* Các mô tả văn bản tự nhiên chi tiết dành cho người đọc vẫn thiếu mã định danh duy nhất (`doc_id`), đồng thời làm phình to token không cần thiết và bắt Claude phải phân tích chuỗi chữ phức tạp thay vì đọc trực tiếp thuộc tính dữ liệu có cấu trúc [687–688].

* **Lựa chọn 4 (Sai):** *A JSON array of document titles extracted from the search results.*
  * *Phân tích:* Mảng JSON chứa tiêu đề tệp vẫn chưa đủ. Tiêu đề tệp có thể bị trùng lặp, chứa ký tự đặc biệt hoặc không thể dùng làm tham số gọi API backend vốn luôn yêu cầu mã định danh duy nhất (`document_id`).

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Multi-step Workflows** | Quy trình xử lý nhiều bước | Chuỗi tác vụ liên hoàn trong đó kết quả của công cụ trước làm đầu vào cho công cụ sau. |
| **Document ID / Identifier** | Mã định danh tài liệu | Chuỗi ký tự duy nhất đại diện cho một tài nguyên/tệp trong hệ thống backend. |
| **Structured Data** | Dữ liệu có cấu trúc | Dữ liệu được tổ chức dưới dạng các cặp key-value (như JSON Object) dễ dàng cho máy đọc. |
| **Human-readable Format** | Định dạng dành cho người đọc | Văn bản tự nhiên được trình bày để con người dễ hiểu nhưng khó phân tích bằng lập trình. |
| **Tool Chaining** | Chuỗi liên kết công cụ | Kỹ thuật tác tử gọi nối tiếp nhiều công cụ dựa trên dữ liệu thu được ở các bước trước. |
| **Metadata** | Siêu dữ liệu / Dữ liệu tả | Các thông tin bổ sung đi kèm tài liệu (như tác giả, kích thước, ngày tạo, trạng thái). |

---

## Practice Test 2 — Q111: Composite Tool for Large Connector Sets


**Đáp án đúng:**  
**Design a `find_and_execute(description, params)` composite tool that searches and immediately executes the best matching connector.** *(Thiết kế một công cụ tổng hợp composite tool `find_and_execute(description, params)` để tìm kiếm và thực thi ngay lập tức connector phù hợp nhất)* [647, 650–651].

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [623–624].
  * **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [623–624].
* **Task Statements liên quan:**
  * **Task Statement 2.1:** Design effective tool interfaces with clear descriptions and boundaries [645–648].
  * **Task Statement 2.3:** Distribute tools appropriately across agents and configure tool choice [650–652].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 2.3:*
    > *"The principle that giving an agent access to too many tools (e.g., 18+ instead of 4-5) degrades tool selection reliability by increasing decision complexity"* [650–651].
  * *Kỹ năng trong Task Statement 2.1 & 2.3:*
    > *"Splitting generic tools into purpose-specific tools or encapsulating large tool sets into abstraction layers to reduce decision complexity and prevent direct misrouting"* [647, 650–651].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Quy tắc "Giới hạn số lượng công cụ khả dụng" (Tool Set Restriction):**
   * Khi số lượng công cụ khả dụng tăng lên quá lớn (50+ connectors), độ chính xác chọn công cụ của LLM sẽ giảm mạnh (trong bài là rơi xuống 58%) do quá tải quyết định (*decision complexity*) [650–651].
   * Khuyến nghị kiến trúc của Anthropic là **chỉ nên cung cấp 4–5 công cụ khả dụng** cho tác tử tại một thời điểm [650–651].
2. **Mẫu thiết kế công cụ tổng hợp (Composite Tool Pattern):**
   * Bằng cách đóng gói 50+ connectors phía sau một công cụ tổng hợp duy nhất là `find_and_execute(description, params)`, tác tử **không còn nhìn thấy 50+ connectors trực tiếp nữa**.
   * Kỹ thuật này giải quyết triệt để **cả 2 sự cố**:
     1. Tác tử không thể "bỏ qua việc tìm kiếm để gọi trực tiếp" vì các connector lẻ không còn nằm trong danh sách công cụ khả dụng.
     2. Tác tử không thể "chọn nhầm connector sau khi tìm kiếm" vì việc tìm kiếm, khớp cú pháp và thực thi được đóng gói nguyên khối (*atomic execution*) bên trong công cụ composite.
3. **Quy tắc loại trừ cạm bẫy:**
   * **Bổ sung mô tả/Few-shot cho 50+ tools:** Việc làm phình to mô tả cho 50+ công cụ vừa gây lãng phí token cực lớn (*context bloat*), vừa không giải quyết được gốc rễ bài toán quá tải công cụ [650–651, 688].
   * **Thêm động công cụ (Dynamic Adding & Persistence):** Nếu các công cụ đã phát hiện vẫn tồn tại (*persist*), sau nhiều lượt gọi danh sách công cụ sẽ bị phình to lại về mức 50+, khiến sự cố chọn sai tái phát [650–651].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 4):** **Design a `find_and_execute(description, params)` composite tool that searches and immediately executes the best matching connector.**
  * *Phân tích:* Khớp 100% với nguyên tắc thiết kế hạ tầng của Anthropic [647, 650–651]. Công cụ tổng hợp (`composite tool`) `find_and_execute` ẩn toàn bộ 50+ connectors đằng sau một giao diện duy nhất. Tác tử chỉ cần truyền mô tả nhu cầu (`description`) và tham số (`params`), hệ thống backend sẽ tự động tra cứu, chọn connector tối ưu và thực thi ngay lập tức. Giải pháp này giúp đưa danh sách công cụ khả dụng về lại mức tối thiểu, triệt tiêu hoàn toàn khả năng tác tử gọi nhầm hoặc nhảy bước [647, 650–651].

* **Lựa chọn 1 (Sai):** *Design search_connectors to dynamically add matched connectors to the agent's available tools. Connectors start unavailable and persist once discovered.*
  * *Phân tích:* Việc duy trì cố định (*persist*) các công cụ sau khi tìm thấy sẽ làm danh sách công cụ của tác tử phình toàn bộ qua các lượt thoại. Cuối cùng, hệ thống sẽ quay trở lại trạng thái ban đầu là có 50+ công cụ khả dụng, khiến độ chính xác chọn công cụ lại sụt giảm về 58% [650–651].

* **Lựa chọn 2 (Sai):** *Design connectors with built-in compatibility validation that return descriptive errors for mismatched requests.*
  * *Phân tích:* Đây là phương pháp xử lý bị động (*reactive validation*). Việc trả về lỗi khi truyền sai tham số chỉ bắt tác tử phải thực hiện vòng lặp thử lại (*retry loop*), gây tốn kém token và gia tăng độ trễ (*latency*), nhưng hoàn toàn không giải quyết được nguyên nhân gốc rễ là sự quá tải danh sách công cụ ở đầu vào [648–650].

* **Lựa chọn 3 (Sai):** *Enhance all connector descriptions with detailed usage samples, edge cases, and input requirements. Add few-shot examples showing the correct search-then-use workflow.*
  * *Phân tích:* Việc thêm mô tả chi tiết và ví dụ few-shot cho toàn bộ 50+ connectors sẽ khiến kích thước prompt bùng nổ (*prompt bloat / token waste*), gây ra hiện tượng trôi chỉ dẫn (*Instruction Degradation*) và phân tán sự chú ý (*Attention Dilution*) [688–689]. Ngoài ra, tác tử vẫn có thể bỏ qua bước `search` để gọi trực tiếp connector nếu nó nhìn thấy connector đó trong mảng `tools` [650–651].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Composite Tool Pattern** | Mẫu thiết kế công cụ tổng hợp | Mẫu thiết kế đóng gói nhiều chức năng/API con đằng sau một giao diện công cụ duy nhất để giảm độ phức tạp quyết định [647, 650–651]. |
| **Tool Selection Accuracy** | Độ chính xác chọn công cụ | Tỷ lệ tác tử chọn đúng công cụ và truyền đúng tham số dựa trên ngữ cảnh hội thoại [645–647, 650]. |
| **Decision Complexity** | Độ phức tạp quyết định | Mức độ quá tải của LLM khi phải lựa chọn giữa quá nhiều công cụ khả dụng (khuyên dùng 4–5 công cụ/agent) [650–651]. |
| **Atomic Execution** | Thực thi nguyên khối | Quy trình tìm kiếm, khớp tham số và chạy công cụ diễn ra trong 1 bước xử lý khép kín, tránh đứt đoạn giữa các lượt. |
| **Dynamic Tool Loading** | Nạp công cụ động | Kỹ thuật thêm/bớt công cụ vào danh sách khả dụng của tác tử dựa trên trạng thái phiên làm việc [650–652]. |
| **Prompt Bloat / Token Overhead** | Tràn Prompt / Chi phí Token thừa | Hiện tượng nạp quá nhiều mô tả công cụ và ví dụ khiến prompt bị phình to và giảm hiệu năng mô hình [688–689]. |

---

## Practice Test 2 — Q112: Local Retry for Transient Errors and Agent Recovery for Permanent Errors

**Đáp án đúng:**  
**Handle transient errors (timeouts, 503s) with automatic retries inside the tool implementation, and surface non-transient errors (permission denied, validation fallures) to the agent with descriptive messages so it can take corrective action.** *(Xử lý các lỗi tạm thời (timeouts, 503) bằng cơ chế tự động thử lại bên trong mã nguồn triển khai công cụ, và chuyển các lỗi không tạm thời (bị từ chối quyền, lỗi xác thực) lên cho tác tử kèm thông báo mô tả chi tiết để nó có thể thực hiện hành động khắc phục)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)**:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**).
  * **Domain 5: Context Management & Reliability** (Trọng số **15%**).
* **Task Statements liên quan:**
  * **Task Statement 2.2: Implement structured error responses for MCP tools**.
  * **Task Statement 5.3: Implement error propagation strategies across multi-agent systems**.
* **Trích dẫn chuẩn từ Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 2.2 & 5.3:*
    > *"The distinction between transient errors (timeouts, service unavailability), validation errors (invalid input), business errors (policy violations), and permission errors"*.
    > *"The difference between retryable and non-retryable errors, and how returning structured metadata prevents wasted retry attempts"*.
  * *Kỹ năng trong Task Statement 2.2 & 5.3:*
    > *"Implementing local error recovery within subagents for transient failures, propagating to the coordinator only errors that cannot be resolved locally along with partial results and what was attempted"*.
    > *"Including retriable: false flags and customer-friendly explanations for business rule violations so the agent can communicate appropriately"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Phân chia trách nhiệm xử lý lỗi (Error-Handling Responsibility Partitioning):**
   * **Lỗi tạm thời (*Transient Errors* - Timeout, HTTP 503):** Cần được tự động **thử lại cục bộ (*local error recovery*)** bên trong mã nguồn triển khai của công cụ (*tool implementation*) kèm thuật toán lùi thời gian lũy thừa (*exponential backoff*). Việc này giúp triệt tiêu các sự cố mạng ngắn hạn mà không làm gián đoạn tác tử hay tiêu tốn lượt thoại (*turns*).
   * **Lỗi không tạm thời (*Non-transient Errors* - HTTP 403 Permission Denied, HTTP 422 Validation Failure):** Cần được **chuyển lên cho tác tử (*agent*)** kèm thông báo mô tả có cấu trúc (`isRetryable: false`) để tác tử nhận biết nguyên nhân và thực hiện hành động chỉnh sửa thích hợp (ví dụ: sửa lại tham số hoặc thông báo cho người dùng).
2. **Quy tắc bác bỏ các phản mẫu (Anti-Patterns):**
   * **Bắt tất cả lỗi để thử lại bên trong công cụ (Handle ALL errors inside tool):** Thử lại các lỗi 403 hay 422 bên trong công cụ là vô ích vì các lỗi này sẽ không bao giờ thành công nếu không có sự thay đổi về quyền hạn hoặc tham số đầu vào.
   * **Nuốt lỗi / Trả lỗi chung chung (Generic Universal Error Handler):** Im lặng trả về thông báo chung chung "dịch vụ không khả dụng" sẽ che giấu ngữ cảnh quan trọng, khiến tác tử không thể đưa ra quyết định khôi phục chính xác.
   * **Đẩy tất cả lỗi về cho tác tử ngay lập tức (Surface ALL errors immediately):** Bắt tác tử phải xử lý các đợt gián đoạn mạng ngắn hạn sẽ làm lãng phí token và lượt thoại hội thoại không cần thiết.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 4):** **Handle transient errors (timeouts, 503s) with automatic retries inside the tool implementation, and surface non-transient errors (permission denied, validation fallures) to the agent with descriptive messages so it can take corrective action.**
  * *Phân tích:* Khớp 100% với kiến trúc phân chia trách nhiệm chuẩn của Anthropic. Việc tự động thử lại lỗi 503/timeout bên trong công cụ giúp giải quyết gọn gàng các sự cố mạng tạm thời. Trong khi đó, việc chuyển các lỗi 403/422 kèm thông báo mô tả chi tiết lên cho tác tử giúp nó dừng ngay các cuộc gọi thử lại vô ích và thực hiện bước sửa lỗi tương ứng.

* **Lựa chọn 1 (Sai):** *Handle all errors inside the tool: Implement retries with exponential backoff for every error type, and only surface a failure to the agent after a fixed number of retry attempts have been exhausted.*
  * *Phân tích:* Việc thực hiện retry đối với **mọi loại lỗi** (bao gồm cả 403 Permission Denied và 422 Validation Failure) bên trong công cụ là một sự lãng phí tài nguyên nghiêm trọng. Các lỗi về quyền truy cập hay sai định dạng dữ liệu sẽ không bao giờ tự phục hồi dù có thử lại bao nhiêu lần đi nữa.

* **Lựa chọn 2 (Sai):** *Implement a universal error handler that catches all exceptions and returns a generic "tool unavailable- try again later" message, shielding the agent from error complexity.*
  * *Phân tích:* Đây là phản mẫu bị Anthropic cảnh báo trực tiếp (*generic error status hides valuable context*). Trả về một thông báo lỗi chung chung sẽ giấu đi bản chất của sự cố, khiến tác tử tưởng rằng đó là lỗi mạng tạm thời và tiếp tục gọi lại công cụ một cách sai lầm.

* **Lựa chọn 3 (Sai):** *Surface all errors to the agent immediately with detailed context, and let the agent decide which errors to retry and how many times-keeping the tool implementation stateless and simple.*
  * *Phân tích:* Việc đẩy toàn bộ lỗi (bao gồm cả lỗi mạng tạm thời) lên cho tác tử xử lý sẽ làm tăng số lượt hội thoại (*conversational turns*), tốn kém chi phí token và gia tăng độ trễ cho những sự cố hoàn toàn có thể tự khôi phục cục bộ ngay bên trong mã nguồn công cụ.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Transient Error** | Lỗi tạm thời | Lỗi do nghẽn mạng hoặc quá tải dịch vụ ngắn hạn (503, timeout) có thể tự hết khi thử lại. |
| **Non-transient Error** | Lỗi không tạm thời / Lỗi vĩnh viễn | Lỗi do sai tham số, vi phạm quy tắc nghiệp vụ hoặc thiếu quyền (403, 422) không thể tự hết nếu không chỉnh sửa. |
| **Local Error Recovery** | Tự khôi phục lỗi cục bộ | Kỹ thuật tự động thử lại lỗi tạm thời ngay bên trong mã nguồn công cụ trước khi báo lên tác tử. |
| **IsRetryable Flag** | Cờ báo khả năng thử lại | Thuộc tính dữ liệu có cấu trúc trả về cho tác tử để cho biết lỗi này có nên thử lại hay không. |
| **Error Propagation** | Truyền thấu lỗi | Chiến lược gửi thông tin lỗi từ công cụ/subagent lên tác tử điều phối với đầy đủ ngữ cảnh có cấu trúc. |
| **Exponential Backoff** | Lùi thời gian chờ lũy thừa | Thuật toán tăng dần thời gian chờ giữa các lần thử lại để tránh làm quá tải hệ thống đích. |

---


## Practice Test 2 — Q113: Actionable Alternative Slots in Appointment Booking Errors


**Đáp án đúng:**  
**Modify `book_appointment` to return detailed failure information including currently available alternative slots when the requested slot is unavailable, enabling the agent to retry with a different time.** *(Chỉnh sửa công cụ `book_appointment` để trả về thông tin thất bại chi tiết bao gồm danh sách các khung giờ thay thế hiện có khi khung giờ yêu cầu bị trùng, cho phép tác tử thử lại ngay với khung giờ khác)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này nằm trong bộ đề chuẩn bị của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [617–618]:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [623–624].
  * **Domain 5: Context Management & Reliability** (Trọng số **15%**) [623–624].
* **Task Statements liên quan:**
  * **Task Statement 2.2:** Implement structured error responses for MCP tools [648–650].
  * **Task Statement 5.3:** Implement error propagation strategies across multi-agent systems [692–694].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 5.3 & 2.2:*
    > *"Structured error context (failure type, attempted query, partial results, alternative approaches) as enabling intelligent coordinator recovery decisions"* [692–693].
  * *Kỹ năng trong Task Statement 5.3:*
    > *"Returning structured error context including failure type, what was attempted, partial results, and potential alternatives to enable coordinator recovery"* [693–694].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nguyên tắc "Mở rộng Phản hồi Lỗi có Cấu trúc kèm Phương án Thay thế" (Structured Error Context with Alternatives):**  
   Khi một công cụ gặp sự cố không thể hoàn thành tác vụ (như khung giờ đặt lịch đã bị người khác nhanh tay đăng ký trước), nguyên tắc chuẩn của Anthropic là **trả về phản hồi lỗi kèm theo thông tin chi tiết và danh sách các phương án thay thế khả thi ngay trong đợt gọi công cụ đó** (`potential alternatives to enable coordinator recovery`).
2. **Giảm thiểu Số lượt gọi API & Độ trễ (Turn & Latency Optimization):**  
   Nếu công cụ `book_appointment` thất bại mà chỉ báo "Slot not available", tác tử sẽ phải tốn thêm 1 lượt thoại gọi lại `get_available_slots` để lấy danh sách mới, rồi mới gọi lại `book_appointment` ở lượt tiếp theo (tốn 2-3 lượt). Việc trả luôn danh sách khung giờ trống thay thế trong kết quả lỗi giúp tác tử chọn ngay khung giờ mới trong lượt thoại tiếp theo [687–688, 693].
3. **Phân biệt với các giải pháp cồng kềnh (Anti-Patterns):**
   * **Thêm công cụ trung gian (`hold_slot`):** Tăng thêm 1 bước gọi tool trung gian khiến luồng xử lý bị kéo dài và gia tăng độ trễ không cần thiết [687–688].
   * **Gộp thành 1 công cụ duy nhất (`find_and_book_appointment`):** Việc gộp tìm kiếm và đặt lịch làm một sẽ phá hỏng quy trình tương tác khi người dùng cần được xem/xác nhận danh sách các khung giờ trước khi đưa ra quyết định đặt lịch.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 1):** **Modify `book_appointment` to return detailed failure information including currently available alternative slots when the requested slot is unavailable, enabling the agent to retry with a different time.**
  * *Phân tích:* Khớp 100% với kỹ năng trong Task Statement 5.3 của Anthropic. Khi đặt lịch thất bại do xung đột thời gian (race condition), việc trả về thông tin lỗi có cấu trúc kèm theo các khung giờ trống thay thế (`alternative slots`) giúp tác tử có đủ ngữ cảnh để đưa ra quyết định đặt lại lịch hoặc gợi ý cho người dùng ngay lập tức mà không cần thực hiện thêm cuộc gọi khám phá thừa.

* **Lựa chọn 2 (Sai):** *Keep both tools but add retry logic to the agent's system prompt, instructing it to call `get_available_slots` again and select a different time if booking fails.*
  * *Phân tích:* Việc dựa vào hướng dẫn prompt để bắt tác tử gọi lại `get_available_slots` sẽ làm tăng thêm lượt thoại (*conversational turn*), gây tốn token và tăng độ trễ. Ngoài ra, giữa đợt gọi `get_available_slots` mới và `book_appointment` tiếp theo vẫn có thể tiếp tục xảy ra xung đột thời gian [637–638, 687].

* **Lựa chọn 3 (Sai):** *Add a `hold_slot(provider_id, slot_time)` tool that creates a 60 second temporary reservation, requiring the agent to call it between checking availability and booking.*
  * *Phân tích:* Bắt tác tử phải gọi thêm công cụ `hold_slot` giữa bước kiểm tra và đặt lịch sẽ làm tăng ma sát quy trình (*workflow friction*) và số lượng cuộc gọi tool không cần thiết [687–688].

* **Lựa chọn 4 (Sai):** *Combine both tools into a single `find_and_book_appointment` that atomically checks availability and books, returning either the confirmed booking or available alternatives*
  * *Phân tích:* Tự động tìm và đặt ngay một lịch ngẫu nhiên mà không thông qua sự lựa chọn/xác nhận của người dùng là phản mẫu trong thiết kế tác tử đặt lịch. Người dùng cần có quyền lựa chọn khung giờ phù hợp với họ trước khi tác tử thực hiện hành động đặt chỗ [689–692].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Structured Error Context** | Ngữ cảnh lỗi có cấu trúc | Dữ liệu lỗi chi tiết bao gồm nguyên nhân, trạng thái và các phương án thay thế trả về cho tác tử [692–693]. |
| **Potential Alternatives** | Các phương án thay thế tiềm năng | Dữ liệu gợi ý khả thi trả về cùng thông báo lỗi để tác tử tự khôi phục luồng xử lý. |
| **Race Condition / Time-of-Check to Time-of-Use** | Xung đột thời gian thực thi | Tình huống dữ liệu bị thay đổi bởi người dùng khác giữa lúc tác tử kiểm tra và lúc thực thi action. |
| **Error Recovery** | Khôi phục sau lỗi | Khả năng tác tử nhận biết lỗi và đưa ra hành động khắc phục thông minh dựa trên ngữ cảnh lỗi. |
| **Actionable Error Metadata** | Siêu dữ liệu lỗi có thể thực thi | Thông tin lỗi chứa đủ dữ liệu để tác tử có thể dùng làm tham số cho bước xử lý tiếp theo. |

---
## Practice Test 2 — Q114: Local Retry for Network Timeouts and Immediate Validation Errors

**Đáp án đúng:**  
**Implement automatic retry with backoff for network timeouts inside the tool; return syntax errors immediately with parameter validation details.** *(Triển khai cơ chế tự động thử lại kèm lùi thời gian (backoff) cho các lỗi quá giờ mạng bên trong công cụ; trả về lỗi cú pháp ngay lập tức kèm chi tiết xác thực tham số)* [650–651, 693–696].

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [619–620]:

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [625–626].
  * **Domain 5: Context Management & Reliability** (Trọng số **15%**) [625–626].
* **Task Statements liên quan:**
  * **Task Statement 2.2: Implement structured error responses for MCP tools** [650–652].
  * **Task Statement 5.3: Implement error propagation strategies across multi-agent systems** [693–696].
* **Trích dẫn chuẩn từ Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 2.2 & 5.3:*
    > *"The distinction between transient errors (timeouts, service unavailability) and validation errors (invalid input, syntax errors)"* [650, 693–694].
    > *"Why uniform error responses prevent the agent from making appropriate recovery decisions"* [650–651, 694].
  * *Kỹ năng trong Task Statement 2.2 & 5.3:*
    > *"Implementing local error recovery within tools/subagents for transient failures, propagating to the agent only errors that cannot be resolved locally along with parameter validation details"* [651, 695–696].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nguyên tắc phân chia trách nhiệm xử lý lỗi (Error Handling Partitioning):**
   * **Lỗi quá giờ mạng (Network Timeouts - chiếm 8%):** Thuộc nhóm **Lỗi tạm thời (Transient Errors)**. Cần được xử lý tự động ngay **bên trong mã nguồn triển khai công cụ (*inside the tool implementation*)** bằng cơ chế thử lại kèm backoff [650–651, 695–696]. Cách này giúp khắc phục lỗi ngay tại chỗ mà không làm tiêu tốn lượt thoại (*conversational turns*) hay làm phiền Agent [633, 688–689].
   * **Lỗi cú pháp truy vấn (Query Syntax Errors - chiếm 4%):** Thuộc nhóm **Lỗi xác thực tham số (Validation Errors)**. Dù có thử lại bao nhiêu lần cũng không thể thành công [651, 681–682]. Cần **trả về ngay lập tức cho Agent** kèm theo chi tiết xác thực tham số (*parameter validation details*) để Agent nhận biết sai sót và chỉnh sửa lại bộ lọc [651, 681–683].
2. **Nhận diện lý do loại trừ Đáp án 3 (Option 3):**
   * Nếu chỉ trả về lỗi kèm cờ `retryable: true/false` cho Agent xử lý (Đáp án 3), tác tử vẫn phải tiêu tốn 1 lượt thoại để nhận kết quả lỗi timeout và gửi lại đợt gọi API mới, gây tăng độ trễ và lãng phí token vô ích [633, 688–689].
3. **Nhận diện lý do loại trừ Đáp án 1 & 2 (Option 1 & 2):**
   * **Dùng Few-shot trong System Prompt (Đáp án 1):** Prompting mang tính xác suất (*probabilistic*) và không thể giúp công cụ tự động thử lại lỗi mạng ở tầng hạ tầng [639–642].
   * **Thử lại đồng nhất cho tất cả các lỗi (Đáp án 2):** Thử lại lỗi cú pháp (lỗi 4% không tạm thời) bên trong công cụ sẽ làm treo hệ thống qua các lần retry vô nghĩa trước khi báo cạn retries [650–651].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 4):** **Implement automatic retry with backoff for network timeouts inside the tool; return syntax errors immediately with parameter validation details.**
  * *Phân tích:* Khớp 100% với nguyên tắc thiết kế độ tin cậy của Anthropic [650–651, 693–696]. Việc tự động retry lỗi timeout bên trong công cụ giúp giải quyết gọn gàng 8% sự cố mạng mà không làm ngắt luồng của Agent. Đồng thời, việc trả lỗi cú pháp kèm chi tiết tham số về cho Agent giúp giải quyết dứt điểm 4% lỗi do sai bộ lọc filter mà người dùng cung cấp [651, 681–683].

* **Lựa chọn 1 (Sai):** *Add few-shot examples to your system prompt demonstrating how to distinguish network errors from syntax errors and handle each case appropriately.*
  * *Phân tích:* Hướng dẫn qua prompt chỉ mang tính xác suất và không thể thay thế cho logic xử lý lập trình cứng (*programmatic error handling*) ở tầng công cụ [639–642]. Ngoài ra, bắt Agent tự phân biệt và gửi lại cuộc gọi mạng vẫn làm lãng phí lượt thoại không cần thiết [633, 688–689].

* **Lựa chọn 2 (Sai):** *Apply exponential backoff retry logic to all errors uniformly, returning a generic "service temporarily unavailable" message after max retries are exhausted.*
  * *Phân tích:* Xử lý đồng nhất (*uniformly*) cho mọi loại lỗi là một phản mẫu [650–651, 694]. Lỗi cú pháp filter là lỗi không tạm thời, thử lại nhiều lần chỉ làm tốn thời gian chờ mà vẫn thất bại.

* **Lựa chọn 3 (Sai):** *Return all errors with a retryable boolean flag and error type details.*
  * *Phân tích:* Mặc dù cờ `retryable` là một thực hành tốt trong phản hồi lỗi có cấu trúc [651, 695–696], nhưng nếu trả cả lỗi timeout về cho Agent quyết định retry, hệ thống vẫn bắt Agent trải qua thêm một lượt thoại (*conversational turn*) xử lý không cần thiết [633, 688–689]. Xử lý cục bộ lỗi timeout ngay bên trong công cụ (như Đáp án 4) là giải pháp tối ưu hơn [651, 695–696].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Local Error Recovery** | Khôi phục lỗi cục bộ | Kỹ thuật xử lý các lỗi mạng tạm thời ngay bên trong mã nguồn triển khai công cụ [651, 695–696]. |
| **Parameter Validation Details** | Chi tiết xác thực tham số | Thông tin mô tả chi tiết vị trí/nguyên nhân sai cú pháp của tham số đầu vào để tác tử tự sửa lỗi [651, 681–683]. |
| **Transient vs. Non-Transient Errors** | Lỗi tạm thời v.s Lỗi vĩnh viễn | Sự khác biệt giữa lỗi mạng ngắn hạn (timeout, 503) và lỗi logic/tham số cố định (400, syntax error) [650–651, 693–694]. |
| **Turn Waste / Overhead** | Lãng phí lượt thoại | Hiện tượng tác tử phải tốn thêm các lượt gọi API phụ chỉ để xử lý các lỗi có thể tự khắc phục ở tầng hạ tầng [633, 688–689]. |
| **Exponential Backoff** | Lùi thời gian chờ lũy thừa | Thuật toán tăng dần khoảng thời gian giữa các lần thử lại để tránh làm quá tải dịch vụ đích [650–651]. |

---
## Practice Test 2 — Q115: Field-Level Confidence Thresholds for Human Review


**Đáp án đúng:**  
**Return fields with confidence scores, plus a `requires_review` boolean computed using your tested confidence thresholds, along with a `review_reasons` array explaining which fields triggered review.** *(Trả về các trường dữ liệu kèm điểm tin cậy, cộng với cờ logic `requires_review` được tính toán bằng các ngưỡng điểm tin cậy đã qua kiểm thử, đi kèm mảng `review_reasons` giải thích rõ trường nào đã kích hoạt yêu cầu xem xét)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [619–620]:

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 5: Context Management & Reliability** (Trọng số **15%**) [625–626].
* **Task Statement:** **Task Statement 5.5: Design human review workflows and confidence calibration** (Thiết kế luồng công việc xem xét của con người và hiệu chỉnh độ tin cậy) [699–701].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 5.5:*
    > *"The risk that aggregate accuracy metrics (e.g., 97% overall) may mask poor performance on specific document types or fields"*.
    > *"Field-level confidence scores calibrated using labeled validation sets for routing review attention"* [699–700].
  * *Kỹ năng trong Task Statement 5.5:*
    > *"Having models output field-level confidence scores, then calibrating review thresholds using labeled validation sets"*.
    > *"Routing extractions with low model confidence or ambiguous/contradictory source documents to human review, prioritizing limited reviewer capacity"* [700–701].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Tránh bẫy "Điểm tổng hợp" (Aggregate Metrics Masking):** Một điểm số tổng hợp chung cho toàn bộ tài liệu (ví dụ 90% trung bình) có thể che giấu sai sót ở một trường dữ liệu cực kỳ quan trọng (như tổng số tiền thanh toán `amount` bị trích xuất sai dù tên nhà cung cấp `vendor` và ngày tháng `date` có điểm tin cậy cao).
2. **Quy tắc "Tính toán cờ logic bằng mã lập trình" (Programmatically Computed Flags):**
   * LLM xử lý số liệu xác suất thô (0.0 – 1.0) qua prompt rất kém và mang tính xác suất (*probabilistic*), dễ gây ra hiện tượng đánh giá lệch ngưỡng (đã dẫn đến 23% trích xuất sai bị bỏ qua và 31% xem xét duyệt người dùng không cần thiết) [639–642, 699–700].
   * Ngưỡng tin cậy (*confidence thresholds*) phải được **kiểm thử và tính toán bằng mã lập trình cứng (*programmatic calculation*)** ở tầng công cụ, trả về cờ boolean `requires_review` rõ ràng cho Agent [639–642, 700].
3. **Giữ cấu trúc Schema ổn định kèm theo lý do minh bạch (`review_reasons`):** Việc bổ sung mảng `review_reasons` chỉ rõ trường nào bị tụt điểm tin cậy (ví dụ: `["amount confidence 0.45 < threshold 0.85"]`) giúp Agent hiểu ngay nguyên nhân và cung cấp ngữ cảnh chính xác cho chuyên viên xem xét [700–701].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 4):** **Return fields with confidence scores, plus a `requires_review` boolean computed using your tested confidence thresholds, along with a `review_reasons` array explaining which fields triggered review.**
  * *Phân tích:* Khớp 100% với kiến trúc chuẩn trong Task Statement 5.5 của Anthropic [699–701]. Công cụ tính toán cờ `requires_review` dựa trên các ngưỡng đã được hiệu chỉnh qua tập dữ liệu kiểm thử. Việc trả về cờ boolean này cùng mảng `review_reasons` cung cấp chỉ dẫn chắc chắn (*deterministic guidance*) cho Agent, loại bỏ tình trạng đoán mò xác suất, đồng thời giữ nguyên Schema gốc của dữ liệu hóa đơn [699–701].

* **Lựa chọn 1 (Sai):** *Compute an aggregate extraction quality score across all fields and return it alongside the extracted values. Include a text summary describing the overall extraction reliability.*
  * *Phân tích:* Vi phạm trực tiếp nguyên tắc thiết kế được nêu trong Task Statement 5.5. Điểm chất lượng tổng hợp (*aggregate score*) sẽ làm mờ đi các trường bị lỗi đơn lẻ, khiến Agent bỏ qua các hóa đơn có tổng tiền `amount` bị sai nếu hai trường còn lại đạt điểm tối đa.

* **Lựa chọn 2 (Sai):** *Return fields with their raw confidence scores and add detailed few-shot examples to your system prompt demonstrating how to interpret different confidence ranges and when to request human review.*
  * *Phân tích:* Việc bắt LLM tự đọc số xác suất thô và quyết định qua few-shot prompt thuộc nhóm hướng dẫn mang tính xác suất (*probabilistic instructions*) [639–642]. Thực tế sản xuất đã chứng minh cách làm này tạo ra tỷ lệ lỗi 23% và 31% lãng phí tài nguyên người duyệt.

* **Lựa chọn 3 (Sai):** *Return fields organized into `verified` and `needs_verification` objects based on confidence thresholds.*
  * *Phân tích:* Phân rã dữ liệu thành hai object động (`verified` và `needs_verification`) sẽ làm **biến đổi cấu trúc Schema đầu ra của công cụ** [678–681]. Điều này khiến các hệ thống xử lý phía sau (*downstream parser*) bị rối do không biết một trường cụ thể (như `vendor`) sẽ nằm ở object nào tùy thuộc vào từng đợt chạy [678–681].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Confidence Calibration** | Hiệu chỉnh độ tin cậy | Quy trình dùng tập dữ liệu gán nhãn kiểm thử để xác định các ngưỡng điểm tin cậy chính xác cho từng trường dữ liệu [699–700]. |
| **Aggregate Accuracy Metric** | Điểm số độ chính xác tổng hợp | Chỉ số trung bình cộng (dễ che giấu các lỗi nghiêm trọng ở từng trường dữ liệu riêng lẻ). |
| **Field-Level Confidence Score** | Điểm tin cậy cấp trường dữ liệu | Điểm số xác suất riêng biệt cho từng thuộc tính trích xuất (vendor, amount, date) [699–700]. |
| **Programmatic Thresholding** | Phân ngưỡng bằng lập trình | Việc tính toán cờ kiểm duyệt dựa trên mã nguồn thay vì để LLM tự diễn giải con số xác suất [639–642, 700]. |
| **Stratified Random Sampling** | Lấy mẫu ngẫu nhiên phân tầng | Phương pháp kiểm tra chất lượng trích xuất bằng cách lấy mẫu đại diện từ từng nhóm/loại tài liệu [699–700]. |

---


## Practice Test 2 — Q116: Identifier-Based Game Lookup Before Score Updates

### **Đáp án đúng:**  
**Replace the three parameters with a single `game_id` parameter and a separate `search_games` lookup tool that returns matching game IDs.** *(Thay thế 3 tham số bằng một tham số `game_id` duy nhất và thêm một công cụ tra cứu `search_games` riêng biệt trả về danh sách các `game_id` khớp)* [648–650].

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

* **Phân vùng kiến thức chính (Primary Domains):**
  * **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [625–626].
  * **Domain 1: Agentic Architecture & Orchestration** (Trọng số **27%**) [625–626].
* **Task Statements liên quan:**
  * **Task Statement 2.1: Design effective tool interfaces with clear descriptions and boundaries** [647–650].
  * **Task Statement 1.3: Configure subagent invocation, context passing, and spawning** [637–639].
* **Trích dẫn chuẩn từ Exam Guide Blueprint:**
  * *Kiến thức trong Task Statement 2.1 & 1.3:*
    > *"Splitting generic tools into purpose-specific tools or using lookup tools with unique identifiers to eliminate parameter ambiguity and prevent misrouting"* [648–650].
    > *"Trimming verbose tool outputs and returning structured data with clear identifiers so downstream tools can reference specific items unambiguously"* [688–691].

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Mẫu thiết kế "Tra cứu bằng Mã định danh duy nhất" (Identifier-based Lookup Pattern):**  
   Khi một công cụ nhận vào nhiều tham số dạng chuỗi tự do (`string`) dễ gây nhầm lẫn hoặc mơ hồ (như biệt danh đội bóng, định dạng ngày tháng không nhất quán, các trận đấu lại/rematches trong cùng mùa giải), thiết kế giao diện công cụ tối ưu nhất của Anthropic là **Chuyển sang quy trình 2 bước**:
   * **Bước 1 (Tra cứu):** Dùng công cụ `search_games` để tìm kiếm linh hoạt theo ngữ cảnh và trả về dữ liệu có cấu trúc chứa mã định danh duy nhất (`game_id`).
   * **Bước 2 (Thực thi):** Công cụ `update_game_score` chỉ nhận đúng tham số `game_id` để cập nhật [648–650].
2. **Giải quyết triệt để cả 3 sự cố sản xuất (Production Failures):**
   * **Sự cố 1 (Dùng biệt danh):** Công cụ `search_games` xử lý việc tìm kiếm linh hoạt ở backend và trả về đối tượng chuẩn.
   * **Sự cố 2 (Sai định dạng ngày):** Bỏ hẳn việc bắt Agent tự format chuỗi ngày tháng khi gọi hàm cập nhật.
   * **Sự cố 3 (Trùng trận tái đấu/Rematches):** Mỗi trận đấu cụ thể trong mùa giải đều có một `game_id` độc nhất, triệt tiêu 100% sự mơ hồ.
3. **Quy tắc loại trừ cạm bẫy:**
   * **Sửa mô tả bằng Prompt (Lựa chọn 1):** Hướng dẫn qua prompt chỉ mang tính xác suất (*probabilistic*), vẫn có tỷ lệ sai sót nhất định [639–642].
   * **Ràng buộc Enum & Regex (Lựa chọn 3):** Dù giải quyết được tên chính thức và định dạng ngày, nhưng **không thể giải quyết được sự cố các trận tái đấu (rematches)** có cùng cặp đấu trong mùa giải.
   * **Thêm tham số `season` & cờ xác nhận (Lựa chọn 4):** Cờ `confirm_before_update` làm tăng thêm ma sát lượt thoại không cần thiết (*conversational friction*) mà vẫn không tạo ra mã định danh duy nhất để phân biệt các trận đấu trùng cặp.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 2):** **Replace the three parameters with a single `game_id` parameter and a separate `search_games` lookup tool that returns matching game IDs.**
  * *Phân tích:* Khớp 100% với nguyên tắc thiết kế công cụ của Anthropic [648–650]. Chuyển sang mô hình tra cứu `search_games` \\(\rightarrow\\) lấy `game_id` \\(\rightarrow\\) gọi `update_game_score(game_id)` giúp hệ thống triệt tiêu toàn bộ 3 nguồn rủi ro bằng một mã định danh tuyệt đối (`deterministic identifier`).

* **Lựa chọn 1 (Sai):** *Add detailed examples to the tool description showing the required date format and complete list of official team names.*
  * *Phân tích:* Viết ví dụ chi tiết trong mô tả công cụ chỉ mang tính chỉ dẫn xác suất (*probabilistic guidance*) [639–642]. Agent vẫn có xác suất vi phạm khi gặp tên biệt danh lạ hoặc khi xử lý các trận tái đấu phức tạp.

* **Lựa chọn 3 (Sai):** *Add enum constraints listing valid team names for both team parameters, and add a regex pattern enforcing ISO 8601 format for the date parameter.*
  * *Phân tích:* Enum và Regex giúp bắt buộc kiểu dữ liệu ở tầng Schema [678–681], nhưng **hoàn toàn thất bại trước sự cố các trận tái đấu (rematches)**. Khi hai đội gặp nhau nhiều lần trong mùa giải, việc truyền đúng tên enum và đúng định dạng ngày ISO 8601 vẫn không ngăn được việc Agent chọn nhầm trận đấu cần cập nhật.

* **Lựa chọn 4 (Sai):** *Add a season parameter to disambiguate rematches, and add a confirm_before_update flag that returns the resolved game details for the agent to verify before the score is committed.*
  * *Phân tích:* Cờ `confirm_before_update` ép Agent phải trải qua thêm một lượt thoại xác nhận không cần thiết, gây tăng độ trễ và chi phí token [687–689]. Ngoài ra, việc bổ sung tham số `season` vẫn không giải quyết được vấn đề biệt danh hay định dạng ngày tháng không nhất quán.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Identifier-based Lookup Pattern** | Mẫu tra cứu dựa trên mã định danh | Kỹ thuật dùng tool tra cứu để lấy `ID` duy nhất trước khi gọi tool ghi/sửa dữ liệu [648–650]. |
| **Parameter Ambiguity** | Sự mơ hồ của tham số | Tình huống tham số đầu vào là chuỗi tự do dễ bị hiểu nhầm hoặc trùng lặp dữ liệu. |
| **Deterministic Identifier** | Mã định danh tuyệt đối | Chuỗi ID độc nhất (như `game_id`, `customer_id`) giúp máy tính xác định chính xác 100% đối tượng. |
| **Two-Step Tool Pattern** | Mẫu thiết kế công cụ 2 bước | Quy trình tách biệt giữa bước Tìm kiếm (`Search/Lookup`) và bước Thực thi (`Execute/Update`) [648–650]. |

---

## Practice Test 2 — Q117: Scoped Tool Access to Reduce Decision Complexity

**Đáp án đúng:**  
**Choosing from 18 tools instead of 4-5 relevant ones increases decision complexity beyond reliable selection thresholds.** *(Việc lựa chọn từ 18 công cụ thay vì 4-5 công cụ có liên quan làm tăng độ phức tạp của quyết định vượt quá ngưỡng chọn lựa tin cậy)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

Câu hỏi tình huống này trích từ **Scenario 3: Multi-Agent Research System** thuộc đề cương chính thức của kỳ thi **Claude Certified Architect – Foundations (CCA-f)** [628–629]:

*   **Phân vùng kiến thức chính (Primary Domain):** **Domain 2: Tool Design & MCP Integration** (Trọng số **18%**) [625–626].
*   **Task Statement:** **Task Statement 2.3: Distribute tools appropriately across agents and configure tool choice** (Phân phối công cụ phù hợp giữa các tác tử và cấu hình lựa chọn công cụ) [652–655].
*   **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
    *   *Kiến thức trong Task Statement 2.3:*
        > *"The principle that giving an agent access to too many tools (e.g., 18 instead of 4-5) degrades tool selection reliability by increasing decision complexity"*.
        > *"Why agents with tools outside their specialization tend to misuse them (e.g., a synthesis agent attempting web searches)"*.
    *   *Kỹ năng trong Task Statement 2.3:*
        > *"Restricting each subagent's tool set to those relevant to its role, preventing cross-specialization misuse"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1.  **Nguyên tắc "Giới hạn số lượng công cụ / Phân quyền công cụ theo phạm vi" (Scoped Tool Access):**
    *   Khi một tác tử được cung cấp quá nhiều công cụ khả dụng (ví dụ 18 công cụ thay vì tập hợp tối ưu 4–5 công cụ), độ tin cậy trong việc lựa chọn công cụ của LLM sẽ sụt giảm mạnh do sự gia tăng **độ phức tạp của quyết định (*decision complexity*)** vượt quá ngưỡng chọn lựa tin cậy.
    *   Nếu một tác tử có access tới các công cụ nằm ngoài phạm vi chuyên môn của nó (như tác tử tổng hợp `synthesis agent` có công cụ `web_search`), nó sẽ xu hướng sử dụng sai/gọi nhầm các công cụ đó.
2.  **Quy tắc phân bổ công cụ tối ưu (Tool Distribution Best Practices):**
    *   Mỗi subagent chỉ nên được cấp đúng danh sách các công cụ phục vụ vai trò chuyên biệt của nó (*role-relevant tools*).
    *   Nếu cần thực hiện thao tác liên vai trò với tần suất cao (ví dụ: tác tử tổng hợp cần xác minh lại một mốc thời gian), hãy cung cấp một công cụ thu hẹp chuyên biệt (như `verify_fact`) thay vì cấp toàn bộ công cụ tìm kiếm web diện rộng.

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

*   **Đáp án đúng (Lựa chọn 1):** **Choosing from 18 tools instead of 4-5 relevant ones increases decision complexity beyond reliable selection thresholds.**
  * *Phân tích:* Khớp 100% với nguyên văn kiến thức chuẩn được nêu trong Task Statement 2.3 của Anthropic [652–653]. Việc mở toàn bộ 18 công cụ cho cả 4 subagents làm phình to không gian quyết định, khiến LLM bị quá tải nhận thức (*decision complexity*) và dẫn đến việc gọi công cụ sai mục đích/ngoài chuyên môn [652–653].

*   **Lựa chọn 2 (Sai):** *The coordinator cannot track which capabilities each subagent has, leading to misrouted tasks.*
  * *Phân tích:* Đề bài mô tả sự cố là do chính các subagent tự gọi công cụ sai chuyên môn khi đang thực thi (ví dụ: `synthesis agent` tự gọi `web_search`), chứ không phải do `coordinator` phân rã công việc sai.

*   **Lựa chọn 3 (Sai):** *The agents' role descriptions in their system prompts conflict with having access to tools outside that role.*
  * *Phân tích:* Mặc dù mô tả vai trò trong system prompt có định hướng hành vi, nhưng nguyên nhân cốt lõi khiến LLM chọn sai công cụ ở tầng hạ tầng chính là do danh sách `tools` khả dụng quá rộng (18 công cụ) làm tăng độ phức tạp quyết định [652–653].

*   **Lựa chọn 4 (Sai):** *The tool definitions consume too much context window space, leaving insufficient room for task content.*
  * *Phân tích:* Mặc dù mô tả 18 công cụ tiêu tốn token ngữ cảnh, nhưng vấn đề chính được hỏi ở đây là **"tại sao hành vi chọn công cụ lại kém"** (*poor tool selection behavior*). Lý do trực tiếp là sự quá tải ngưỡng quyết định (*decision complexity threshold*) [652–653].

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **Decision Complexity** | Độ phức tạp quyết định | Mức độ quá tải nhận thức của LLM khi phải lựa chọn giữa quá nhiều công cụ khả dụng cùng lúc (ngưỡng tối ưu là 4–5 tools). |
| **Scoped Tool Access** | Phân quyền công cụ theo phạm vi | Việc giới hạn danh sách công cụ cấp cho từng subagent đúng với vai trò chuyên môn của nó. |
| **Cross-Specialization Misuse** | Sử dụng sai ngoài chuyên môn | Hiện tượng subagent tự ý gọi các công cụ thuộc lĩnh vực/chức năng của subagent khác. |
| **Cross-Role Tools** | Công cụ liên vai trò | Các công cụ đặc thù có phạm vi hẹp (như `verify_fact`) được cấp thêm cho subagent để xử lý các nhu cầu xác minh tần suất cao. |
| **Tool Selection Reliability** | Độ tin cậy chọn công cụ | Mức độ chính xác của LLM khi chọn đúng công cụ và truyền đúng tham số dựa trên mô tả [647–648, 652]. |

---


## Practice Test 2 — Q118: Using /memory to Diagnose CLAUDE.md Loading

**Đáp án đúng:**  
**Run `/memory` to check which memory files are loaded and verify your CLAUDE.md is included.** *(Chạy lệnh `/memory` để kiểm tra các tệp bộ nhớ (memory files) nào đang được nạp và xác minh tệp `CLAUDE.md` của bạn đã được đưa vào hay chưa)*.

---

### **1. Phân tích so với tài liệu chính thức (Exam Guide & Blueprint Analysis)**

* **Phân vùng kiến thức chính (Primary Domain):** **Domain 3: Claude Code Configuration & Workflows** (Trọng số **20%**) [626, 659–662].
* **Task Statement:** **Task Statement 3.1: Configure CLAUDE.md files with appropriate hierarchy, scoping, and modular organization** [659–662].
* **Trích dẫn chuẩn từ tài liệu Exam Guide Blueprint:**
  * *Knowledge of:*
    > *"The CLAUDE.md configuration hierarchy: user-level (`~/.claude/CLAUDE.md`), project-level (`.claude/CLAUDE.md` or root `CLAUDE.md`), and directory-level (subdirectory `CLAUDE.md` files)"* [659–660].
  * *Skills in:*
    > *"Using the `/memory` command to verify which memory files are loaded and diagnose inconsistent behavior across sessions"*.

---

### **2. Tips để lựa chọn đáp án đúng (Decision Rules & Exam Tips)**

1. **Nhận diện dạng bài "Chẩn đoán hành vi không nhất quán giữa các phiên" (Diagnosing Inconsistent Session Behavior):**
   * Khi `CLAUDE.md` được cấu hình ở gốc dự án nhưng Claude Code hoạt động lúc đúng lúc sai tùy từng phiên làm việc (*session*), bước chẩn đoán đầu tiên (*first diagnostic step*) luôn là kiểm tra xem hệ thống đã thực sự nạp đúng các tệp quy tắc/ngữ cảnh đó vào bộ nhớ hay chưa.
2. **Lệnh `/memory` trong Claude Code CLI:**
   * `/memory` là lệnh đặc hiệu trong Claude Code giúp hiển thị toàn bộ danh sách các tệp memory/CLAUDE.md đang được active trong ngữ cảnh làm việc hiện tại.
3. **Phân biệt giữa Bước Chẩn đoán (Diagnostic Step) và Bước Hành động (Action Step):**
   * Bổ sung ví dụ hay tạo tệp quy tắc mới (`.claude/rules/`) là các **hành động chỉnh sửa**, không phải là **bước chẩn đoán nguyên nhân ban đầu** [661, 664–666].

---

### **3. Phân tích chi tiết toàn bộ 4 câu trả lời**

* **Đáp án đúng (Lựa chọn 3):** **Run `/memory` to check which memory files are loaded and verify your CLAUDE.md is included.**
  * *Phân tích:* Khớp 100% với kỹ năng chẩn đoán được yêu cầu trong Task Statement 3.1 của Anthropic. Lệnh `/memory` giúp bạn lập tức chẩn đoán nguyên nhân gốc rễ: liệu tệp `CLAUDE.md` ở thư mục gốc có thực sự được Claude Code nạp vào bộ nhớ ngữ cảnh trong phiên làm việc đó hay không.

* **Lựa chọn 1 (Sai):** *Add more detailed code examples to your CLAUDE.md showing the exact ApiError usage pattern for different endpoint types.*
  * *Phân tích:* Đây là bước điều chỉnh hướng dẫn prompt (*prompt refinement*) [668–671]. Khi nguyên nhân khiến Claude Code ngẫu nhiên bỏ qua quy tắc giữa các phiên chưa được xác định (có thể do tệp không được nạp), việc thêm ví dụ vào `CLAUDE.md` sẽ không giải quyết được vấn đề nếu tệp đó hoàn toàn bị bỏ qua khi khởi tạo phiên.

* **Lựa chọn 2 (Sai):** *Search for conflicting instructions in `~/.claude/CLAUDE.md` or `~/.claude/rules/` that might override your project conventions.*
  * *Phân tích:* Theo thứ tự ưu tiên cấu hình (*configuration hierarchy*), cấu hình cấp dự án (*project-level*) sẽ ghi đè cấu hình cấp người dùng (`~/.claude/CLAUDE.md`) [659–660]. Hơn nữa, việc tìm kiếm xung đột thủ công tốn thời gian hơn rất nhiều so với việc chỉ cần gõ ngay lệnh `/memory` để xem danh sách tệp được nạp.

* **Lựa chọn 4 (Sai):** *Create path-specific rules in `claude/rules/handlers.md` with YAML frontmatter scoping the error handling instructions to your API handler files.*
  * *Phân tích:* Tạo tệp `.claude/rules/` phân vùng theo đường dẫn (*path-specific rules*) là một giải pháp rất tốt để tối ưu hóa việc nạp quy tắc theo tệp [660, 664–666], nhưng đó là **bước hành động tái cấu trúc quy tắc**, không phải là **bước chẩn đoán nguyên nhân ban đầu** như yêu cầu của đề bài.

---

### **4. Bảng chú giải thuật ngữ Anh - Việt (Glossary for Study Note)**

| Thuật ngữ Anh - Mỹ | Thuật ngữ Tiếng Việt | Định nghĩa / Bối cảnh sử dụng trong bài thi |
| :--- | :--- | :--- |
| **`/memory` Command** | Lệnh `/memory` | Lệnh built-in trong Claude Code dùng để kiểm tra danh sách các tệp memory/CLAUDE.md đang được nạp. |
| **Configuration Hierarchy** | Thứ tự ưu tiên cấu hình | Cấu trúc phân cấp cấu hình từ cấp User (`~/.claude/`) \\(\rightarrow\\) cấp Project (`.claude/`) \\(\rightarrow\\) cấp Directory [659–660]. |
| **Project-Scoped CLAUDE.md** | `CLAUDE.md` cấp dự án | Tệp cấu hình nằm ở thư mục gốc dự án, được lưu vào Git để chia sẻ quy tắc chung cho cả team [659–661]. |
| **Diagnostic Step** | Bước chẩn đoán | Hành động kiểm tra/tra cứu nguyên nhân gây lỗi trước khi thực hiện các thay đổi mã nguồn hoặc tệp cấu hình. |

---


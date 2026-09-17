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

# CCA-F Study Notes

## Task Decomposition: Exam Rule of Thumb

| Situation | Best approach |
|---|---|
| Unknown scope, each step depends on the last (debugging, root cause) | **Dynamic, adaptive** subtasks |
| Known, repeatable steps (e.g., review every file against the same checklist) | **Fixed pipeline / prompt chaining** |
| Independent subtasks with no shared dependency | **Parallel subagents** |

**Quick check:** If you'd need the result of one subtask to decide what the next one is, don't run them in parallel.

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

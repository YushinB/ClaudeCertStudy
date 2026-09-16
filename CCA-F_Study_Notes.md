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

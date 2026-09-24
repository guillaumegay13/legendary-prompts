# Improve this agent harness's token efficiency

You're working on an LLM agent harness: the system prompt, tool definitions, request assembly, context caching, compaction, and retrieval, and how work is split across agents. Make the agent's runs cheaper without making it worse at its job.

- Objective: lower price-weighted token cost per completed task.
- Constraint: no measurable drop in task quality.

Measure per task, not per request. Every turn resends the prefix (tools, instructions, setup, and the conversation so far), so a change that shrinks each request but adds turns can cost more. Weight tokens by billing type: output, uncached input, and cached input are priced very differently.

Work in this order: map the harness and measure the baseline, rank the opportunities, make the changes that are safe to make directly, put the rest behind flags or in proposals, then report.

Figures below come from one team's production coding agent and its multi-agent experiments. Use them to gauge magnitude, not as targets. One round of these changes (prompt trimming, tool offloading, cache layout, sparse line numbers, subagent tuning) cut that team's overall token cost about 7% with no loss in quality. The larger percentages apply only to the part of the request each change touched.

## Principles

1. Change what the harness sends, not how hard the model tries. Don't ask the model to conserve tokens. A harness that told its model to "take care to preserve tokens and not be wasteful" found it grew reluctant to take on ambitious tasks and sometimes quit, saying it wasn't supposed to waste tokens.
2. Capable models need definitions, not commands. Lists of "DO NOT", "You must", and "Important", and guards against older models' habits, can usually be replaced with plain descriptions of what each tool does. One team cut about two-thirds of its system prompt this way, and the shorter prompt worked across model families. Instruct only on what the model can't know (the product, the environment, the user's processes) and on quirks you've seen in transcripts.
3. Static context is for what most turns need. Everything else should be discoverable when needed. Less up-front context also means less confusing or contradictory information.
4. Expect removals to win. Guardrails written for weaker models, coordination steps that became bottlenecks, and prompting for behavior the model now does on its own all cost tokens.
5. Real usage decides. Evals are a fast proxy, but they skew toward hard problems and miss the real mix of requests.

## 1. Map the harness and measure the baseline

Find:

- Where requests are assembled, the system prompt, and tool schemas. If a framework or SDK builds requests, find its hooks for message order, cache control, and tool loading.
- How tool results are formatted, and how history is kept, trimmed, or summarized.
- How subagents or parallel agents are spawned, if any.
- Which models and provider APIs are used. From the provider's docs, get the prompt caching behavior (automatic or explicit breakpoints, TTL, minimum cacheable length) and the prices for output, uncached input, and cached input.
- Existing logging, token accounting, and evals.

If the harness doesn't record per-request token usage by billing type and cache hits, add that first. Everything later depends on it.

Then render a few real requests (from logs, or by running representative tasks) and count tokens per section with the model's tokenizer or the API's usage fields. Produce:

- Cost share by source × billing type. Sources: system prompt, tool definitions, skill/rule/integration descriptions, user messages, file reads, search results, command and other tool output, history, summaries, subagents.
- Static tokens per request, cache hit rate, and turns per task.
- Per tool: the share of runs that call it at least once, and its error rate.

Read the rendered requests, not just the templates. Duplication, leaked volatile values, and misordered blocks only show up there.

Rank opportunities by share of spend × fraction removable ÷ quality risk.

## 2. System prompt and injected context

Label every instruction:

- Keep: product or environment knowledge the model can't infer, fixes for quirks seen in this model's transcripts, and rules a mode depends on.
- Rewrite: commands and emphasis into plain descriptions. Reminders into constraints: "No TODOs, no partial implementations" works better than "remember to finish implementations." Vague quantities into ranges: "generate 20–100 tasks" gets far more ambitious behavior than "generate many tasks."
- Delete: things capable models do by default, guards against behavior you haven't seen from this model, text that repeats tool descriptions, and lines that could contradict a user request. Models trained to rank system instructions above user messages will side with the system prompt.
- Move: anything per-user or per-request (date, environment, repo state, lists of skills or subagents, user rules) into a user-role setup message after the cache boundary.

Audit other injected context the same way. As models improved, the team behind these figures dropped directory trees, pre-retrieved snippets, compressed copies of attached files, lint errors injected after every edit, forced expansion of short file reads, and caps on tool calls per turn. They kept small, high-value facts: OS, repo status, and open or recently viewed files.

Skip checklists for open-ended work. The model optimizes the listed items and deprioritizes everything else.

## 3. Tool definitions

Tool schemas ride along on every request. Most tools beyond the core set were each needed in under 20% of conversations, and moving them out of static context cut tool-description tokens 60%. Doing the same for integration tools (such as MCP servers), with names in context and full schemas in one folder per server that the agent can search with grep or jq, cut total tokens 46.9% in sessions that used them.

- Keep in static context: high-frequency tools (for a coding agent: read, search, edit, shell), tools the model tries to call even when they're absent, and tools a mode depends on.
- Offload the rest: leave a name or one-line pointer and make the full schema discoverable on demand. Group related tools so they load together, and put status (such as "needs re-authentication") where the agent will see it.
- Tighten what remains: describe behavior and arguments, and drop usage lectures.
- Pick the split by testing a few configurations and tracking tokens, cost, latency, tool-call errors, and task success.

## 4. Cache layout

Order each request so the reusable prefix is as long as possible:

`tool definitions → system instructions → [breakpoint] → setup message (skills, subagents, rules, environment) → [breakpoint] → conversation`

- Keep the prefix byte-identical across turns. Use deterministic tool order and serialization, put timestamps and IDs after the boundary, and don't rewrite earlier messages except when compacting.
- Use explicit breakpoints if the provider supports them. Otherwise rely on automatic prefix caching with the stable part first. Respect TTL and minimum-length rules.
- Switching models mid-conversation throws away the cache (caches are per model and provider) and hands the new model a history it didn't write. When a different model is needed, run it as a subagent with fresh context.

Explicit breakpoints plus moving per-request setup after them cut cold cache misses 20%.

## 5. Tool results and other context added during a run

- Large outputs (commands, integrations, logs): write them to a file and return the path, size, and a short tail. The agent can tail, grep, or read ranges for more. Truncating loses data, and inlining bloats every later request. Treat long-running terminal sessions the same way.
- High-volume formats: look for overhead repeated on every line or item. Numbering every 10th line of a file read instead of every line cut cache-read tokens 1.6% without hurting citation accuracy. Each number costs 3–5 tokens, and agents read tens of thousands of lines per session. Also check repeated absolute paths, verbose JSON keys, ANSI codes, progress bars, and repeated headers.
- Good retrieval saves exploration turns. Adding semantic search alongside grep raised codebase question-answering accuracy 12.5% on average and cut the iterations users needed.
- Tool errors waste tokens and leave confusing debris in context. Classify expected errors (invalid arguments, unexpected environment, provider error, timeout, user abort), treat unknown errors as harness bugs, and track rates per tool and per model. One focused effort along these lines cut unexpected tool errors 10×.

## 6. Long runs: compaction, subagents, and model mix

- Compaction: keep the summarization prompt short and the summary compact, carry forward plan state and remaining tasks, and save the full history to a file the agent can search for details the summary dropped. A model trained to self-summarize from a one-line prompt wrote ~1k-token summaries with half the compaction error of a multi-thousand-token prompt that produced 5k+ token summaries. Untrained models may need more guidance, so test how short you can go. A more expensive summarization model made a negligible difference.
- Scratchpads and running notes: rewrite them instead of appending. For repeated work in one environment, a small agent-maintained notes file with a line budget, loaded at start, is a promising way to shorten later runs.
- Subagents: fresh context keeps the parent lean, but isolation adds coordination cost (duplicate or stale work). If the model already delegates on its own, remove prompting that pushes it to. Have subagents return short handoffs: what was done, findings, concerns, and deviations. A subagent should use a different model only when the user or harness says so.
- Model mix: in large multi-agent runs, workers used at least 69% of tokens, and over 90% in most runs. A frontier planner with cheap workers matched a frontier model doing everything at about one-eighth the cost. Planner choice still changes worker spend. One planner that cost less on its own saw its workers use several times more tokens, and the run cost more overall. Measure the whole tree.
- Routing and reasoning effort: send simple turns to a cheaper model or lower effort, and upgrade only when a stronger model is clearly better. A router built this way matched or beat single frontier models on user satisfaction at 41–68% lower cost.
- Reasoning continuity: if the API returns reasoning items (including encrypted ones), pass them back on later turns and alert when they go missing. Dropping them cost one reasoning model 30% on a coding benchmark, and it burned tokens reconstructing its plan.

## 7. Fit the harness to each model

Adapt to what each model was trained on instead of forcing one shape on all of them. If you've tuned the harness for a similar model, start from that version.

- Edit format: use the one the model was trained on (for example, patch-style or search-and-replace). An unfamiliar format costs extra reasoning tokens and causes more mistakes.
- Shell or tools: shell-first models fall back to `cat` or inline scripts. Name tools after their shell equivalents (such as `rg`), and if needed add: "If a tool exists for an action, prefer to use the tool instead of shell commands (e.g. read_file over `cat`)."
- Literalness: some model families follow instructions literally and others tolerate imprecision. Some spiral on emphasized wording. Strip caps and emphasis for literal models.
- Triggers: some models ignore a tool until told when to use it. A literal trigger works: "After substantive edits, use the <lint tool> to check recently edited files for linter errors. If you've introduced any, fix them if you can easily figure out how."
- Progress updates: if a model reports progress through reasoning summaries, keep them to 1–2 sentences that note new findings or a change of tactic, and remove instructions about messaging mid-turn.
- Quirks worth a targeted line: hedging or refusing as context fills ("context anxiety"), declaring completion early, stopping to ask permission, and calling tools that don't exist.

Tie each added instruction to the transcript behavior it fixes. Re-audit when models change, since guidance one version needed can be dead weight for the next.

## 8. Validate

- Offline: run a fixed set of realistic tasks before and after, ideally drawn from real usage and phrased the way users actually write (short and ambiguous). Compare task success, tokens, cost per task, turns, and tool errors. Don't ship a change that lowers success.
- Online, if you have users: A/B test each change or small bundle. The primary metric is cost per completed task. Guardrails are task success signals, tool-call errors, latency, turns per task, and cache hit rate. For a coding agent, a good success signal is how much agent-written code survives over time. In general, check whether the user's next message moves on or reports a problem.
- Ship only when cost drops and no guardrail regresses beyond noise. Record null results.

## What to change directly and what to propose

- Change directly, each in its own revertible commit: token and cache telemetry, deterministic serialization and tool order, moving volatile content out of the cached prefix, explicit cache breakpoints, writing large outputs to files instead of truncating, passing back reasoning items that are being dropped, and fixes for recurring tool errors.
- Change behind a flag so it can be tested: system prompt edits, tool offloading, output format changes, compaction changes, and subagent prompting.
- Propose only: changes to which models run, routing, reasoning-effort defaults, or how work is split across agents.

## Traps

- Asking the model to use fewer tokens or do less.
- Truncating tool output.
- Dropping reasoning items to save input tokens.
- Volatile content in the cached prefix, or tool order that changes between requests.
- Offloading a tool the model needs on the first turn or tries to call when it's missing.
- Emphasis-heavy prompts (MUST, NEVER, IMPORTANT, all caps), especially with literal models.
- Forcing a terser output format than the model was trained on. Fewer output tokens can mean less thinking and worse results.
- Optimizing raw token counts instead of cost, per request instead of per task, or evals instead of real usage.
- Switching models mid-conversation to save money.
- Adding coordination layers that become bottlenecks.

## Report back with

1. The harness map and baseline: cost by source × billing type, with the biggest sources called out.
2. A ranked list of changes: layer, what changes, estimated savings and how you estimated them, quality risk, how to validate, and how to roll back.
3. The changes you made, including a system prompt diff with a keep, rewrite, delete, or move reason for each line.
4. A test plan for the flagged changes.
5. Gaps: anything you couldn't find or measure.

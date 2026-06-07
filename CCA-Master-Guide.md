# Claude Certified Architect (CCA) Foundations — Master Study Guide

_An independent study aid expanding every concept from the exam cheat sheet. Not affiliated with or endorsed by Anthropic._

**The one mental model:** Claude never runs your code. When it wants a tool it *stops* and returns a request (`stop_reason:"tool_use"`); your program runs the function and sends the result back; then you call Claude again. The API is stateless — a "session" is just the message list you keep and re-send.

## Contents

1. Foundations & The Agentic Loop
2. Multi-Agent Orchestration: Hub-and-Spoke
3. Subagents, the Agent Tool & Sessions
4. Claude Code Configuration & Permissions
5. Coordination Patterns & Findings
6. Resilience: Errors, Recovery & Escalation
7. Tool Design & MCP
8. Prompt Engineering & Context Management
9. Scenario Playbook

---

## Module 1 — Foundations & The Agentic Loop

What a coding agent actually is, the loop that drives it, how models and API keys fit in, the all-important `stop_reason`, who makes decisions, and the anti-patterns that fail the exam.

### 01. Agentic Coding Tool & Code Harness

An **agentic coding tool** (aka **coding agent**) is software that reads your codebase, edits files, runs commands, and integrates with your dev tools — acting, not just chatting.

- **Code Harness (agentic harness):** the system that *wraps an LLM* with prompts, tools, and execution environments, turning a raw model into an agent that can take actions and run workflows. The harness is the machinery; the LLM is the brain.
- **Agentic coding tool:** a coding-focused agent built on a harness.
- **Where they run:** terminal, IDE, desktop app, and browser.

> One-liner: the LLM decides *what* to do; the harness gives it the prompts, tools, and an execution environment to actually *do* it.

### 02. Claude Code

**Claude Code is Anthropic's agentic coding tool.** It reads your codebase, edits files, runs commands, and integrates with your development tools — the canonical code harness.

- Core capabilities: search/read/edit files + advanced tools (web fetch, terminal access) + an **MCP client**.
- Surfaces: terminal, IDE extensions, desktop app, web.
- Effort multiplier: more detailed instructions → significantly better results.
- `init` scans the codebase and writes a `CLAUDE.md` auto-included on future requests; `#` appends notes to memory.

### 03. The Agentic Loop

A continuous cycle where an LLM **gathers context → takes action → verifies results**, repeating until the goal is met.

- Uses **tool calls** to interact with code, systems, and external services.
- **Self-reflects** after each step to choose the next action.
- **Stops only when the goal is met.**
- You can **interrupt or guide** the loop at any time.

**Core loop:** `Gather Context → Take Action → Verify Results → repeat`

### 04. Agentic Loop — Model Choice

You choose the model used throughout the loop, trading off **speed, cost, and reasoning ability**.

| Alias | Profile | Use when |
|---|---|---|
| `default` | Auto-selected, recommended | You don't want to think about it |
| `sonnet` | Balanced; strong coding | Most coding tasks |
| `opus` | Strongest reasoning; slower, pricier | Complex multi-step planning |
| `haiku` | Fastest, cheapest | Simple, high-volume tasks |
| `sonnet[1m]` | Large 1M context | Long sessions / big context |
| `opusplan` | Opus plans, Sonnet executes | Plan deeply, execute cheaply |

**Selection framework:** Intelligence → Opus; Speed → Haiku; Balanced → Sonnet. Mixing models per task in one app is common.

### 05. Claude API Key

A **Claude API key** controls cost and usage for Claude Code — useful for production and automated systems.

- Generate at **platform.claude.com**.
- **Never** embed the key in client apps. Clients → your server → Anthropic (server holds the key).
- Store as an env var (`ANTHROPIC_API_KEY`) via a `.env` kept out of version control.

### 06. Stop Reason — `tool_use` vs `end_turn`

The **stop reason** is *why* the agent stopped its loop — the reliable signal for control (never parse text).

| `stop_reason` | Meaning | Your program does… |
|---|---|---|
| `"tool_use"` | Claude wants to call a function | Parse request, run function, append `tool_result`, call again |
| `"end_turn"` | Claude is returning a final result | Stop the loop, return the answer |

```python
while response.stop_reason == "tool_use":
    for block in response.content:
        if block.type == "tool_use":
            result = run_tool(block.name, block.input)   # YOUR code runs it
            messages.append({"role":"user","content":[
                {"type":"tool_result","tool_use_id":block.id,"content":result}]})
    response = client.messages.create(model=MODEL, tools=tools, messages=messages)
```

Gotchas: `tool_result` goes in a **user**-role message; `tool_use_id` must match the request id; re-send the **full** history (incl. tool schemas) every turn — the API is stateless.

### 07. Decision Making: Who Drives the Loop?

| Style | Who decides | Mechanism |
|---|---|---|
| Pre-configured | **You** | Hardcoded logic (if/else, state machines); tool sequences ≈ a state machine |
| Model-driven | **Claude** | Give tools + a goal; the model chooses which tool & when at runtime |
| Hub-and-spoke | **Coordinator** | A central agent controls subagents; subagents never talk directly |

### 08. Anti-Patterns of the Agentic Loop

| ❌ Don't | ✅ Instead |
|---|---|
| Parse natural language to decide flow | Use the structured `stop_reason` |
| Stop based on text *content* | Stop on `end_turn` |
| Use an arbitrary iteration cap as your *primary* stop | Cap = safety net only |
| Act on intermediate text mid-loop | Ignore text until `end_turn`, then return |

**Mantra:** *Always use `stop_reason`. Iteration cap = safety only, not control logic.*

### Module 1 — Checkpoint Quiz

1. **Correct order of the agentic loop?** → **Gather Context → Take Action → Verify Results → repeat.**
2. **You receive `stop_reason: "tool_use"`. What do you do?** → **Run the requested function and send the result back as a `tool_result`.**
3. **Which alias plans with Opus and executes with Sonnet?** → **`opusplan`.**
4. **Which is an anti-pattern for stopping the loop?** → **Stopping based on the text content of a response.**
5. **A `tool_result` must be sent in which role, referencing what?** → **User role; references the matching `tool_use_id`.**

---

## Module 2 — Multi-Agent Orchestration: Hub-and-Spoke

How a **coordinator** decomposes work, delegates to isolated subagents, and aggregates the results into one answer — without subagents ever talking to each other.

### 09. Hub-and-Spoke Architecture

A **coordinator** sits at the centre as the single brain. **Every subagent routes through it** — subagents *never* talk to each other directly, so there is no shared context between them.

**What the coordinator does**

| Phase | Action |
| --- | --- |
| 1. Decompose | Break the user's goal into discrete subtasks. |
| 2. Delegate | Hand each subtask to a subagent with its own isolated context. |
| 3. Aggregate | Merge the returned results, **resolve conflicts**, and produce **one** response. |

Lifecycle: **Decompose → Delegate → Aggregate → one response.**

**Choosing how many agents** — the coordinator picks the shape by complexity:

- **Single** agent — a simple, self-contained task.
- **Sequential** agents — each step depends on the previous one's output.
- **Parallel** agents — independent slices that can run at the same time.

> 🔑 The coordinator is the only component that holds the full picture. Because subagents are isolated, the **final aggregation step** is where results are merged and any conflicting findings are reconciled into a single coherent answer.

> 🎯 **On the exam:** if an option says subagents **share context** or **talk directly to each other**, it's wrong. In hub-and-spoke, the coordinator is the central brain and **all routing goes through it**. The lifecycle is **decompose → delegate → aggregate**.

**Why route everything through the hub?** Isolation keeps each subagent's context clean and its scope bounded, prevents cross-contamination of findings, and gives the coordinator a single point to enforce control, resolve conflicts, and observe everything. Direct subagent-to-subagent links would destroy all of that.

### 10. Narrow Task Decomposition (Coordinator)

If the coordinator decomposes **too narrowly**, the work misses important dimensions — and because subagents have **isolated context**, none of them can detect the gap.

**Why narrow decomposition is dangerous**

- A subagent only sees the slice it was handed. It has **no view of the whole problem**, so it cannot notice that a perspective is missing.
- The gap is invisible until it surfaces as an incomplete or skewed final answer.
- Responsibility for coverage sits entirely with the **coordinator**, not the subagents.

**How the coordinator fixes it**

1. **Expand tasks to cover all perspectives** *before* delegating.
2. Explicitly ask **"what's missing?"** and add subtasks to fill the gaps.
3. **Validate the decomposition** — via a review tool, or during aggregation.
4. **Check for gaps before producing the final answer.**

> ⚠️ **The trap:** narrow ≠ focused. Splitting into clean, non-overlapping slices is good (see Partitioning) — but each slice must still add up to **full coverage** of the problem. Narrow decomposition drops dimensions entirely.

> 📍 **Two places to validate:** up front with a **review tool** on the proposed plan, and again **during aggregation** as a final gap-check before the answer ships.

### 11. Dynamic Selection

**Don't run the full pipeline every time.** The coordinator selects only the agents a request actually needs, scaling the shape to the task's complexity.

| Task complexity | Selection |
| --- | --- |
| Simple | A **single** agent. |
| Complex | **Sequential** or **parallel** agents, depending on dependencies. |

> 🔑 **Rule of thumb:** use an agent **only if it adds value**. Spinning up agents that don't contribute burns tokens and latency for nothing.

> 🚫 **Anti-pattern:** blindly executing the entire agent roster on every request. The coordinator should **route**, not stampede.

**Sequential vs parallel:** **sequential** when each step depends on the previous step's output (a chain); **parallel** when the slices are independent and can run simultaneously. Either way, only for **complex** tasks — a simple task gets a **single** agent.

### 12. Partitioning Research

Handing the **same task to multiple agents** produces overlapping work and wasted tokens. Instead, split the problem into **distinct, non-overlapping scopes** — each agent owns a unique slice.

**❌ Overlapping**

- Agent 1: "Research the market." → covers competitors, pricing, trends
- Agent 2: "Research the market." → covers the same competitors, pricing, trends
- Result: duplicate findings, doubled token spend, nothing extra learned.

**✅ Partitioned**

- Agent 1: **Competitor landscape** only.
- Agent 2: **Pricing & packaging** only.
- Agent 3: **Regulatory & market trends** only.
- Result: each agent owns a unique, non-overlapping slice. Full coverage, no duplication.

> 🎯 **On the exam:** "same task to multiple agents → overlap + wasted tokens" is the wrong pattern. The fix is **partitioning** into distinct scopes. Pair this with decomposition: slices must be non-overlapping **and** collectively complete.

### 13. Refinement Loop

Orchestration is **not one-shot.** The coordinator evaluates the aggregated results, identifies gaps, and **re-delegates** — iterating until coverage meets a defined quality bar.

Loop: **Aggregate → Evaluate coverage → Gaps? → (re-delegate) → Meets bar → finalise.**

- **Evaluate** the combined output against the goal.
- **Identify gaps** in coverage or quality.
- **Re-delegate** targeted subtasks to close them.
- **Iterate** until a **defined quality bar** is reached.

> 🔑 Use a **structured check** to enforce completeness rather than eyeballing the text — e.g. an `evaluate_coverage` tool that returns whether the bar is met and what's still missing.

```python
# Coordinator-side refinement: re-delegate until coverage passes.
results = aggregate(delegate(subtasks))

while True:
    check = evaluate_coverage(results, goal)   # structured check, not text-parsing
    if check["meets_bar"]:
        break
    # close the gaps the check reported
    extra = delegate(check["missing_subtasks"])
    results = aggregate(results, extra)

return finalise(results)
```

> ⚠️ Treating multi-agent output as one-shot is a classic failure: the first pass rarely meets the bar. The loop is what guarantees completeness — mirroring the verify step of the agentic loop (Module 1) at the orchestration level.

### 14. Observability

The coordinator isn't just a router — it's the **single point through which every message and error flows**, which is what makes a multi-agent system debuggable and controllable.

| Without a coordinator | With a coordinator |
| --- | --- |
| Flows are **untraceable** — no single place to see what happened. | All messages & errors are **centralised** at the hub. |
| Failures are **inconsistent** and hard to reproduce. | Enables **debugging, control, and policy enforcement**. |

> 🔑 Centralisation is the operational payoff of hub-and-spoke: because nothing bypasses the hub, the coordinator can **log every interaction**, trace a failure to its source, throttle or retry, and **enforce policy** on what subagents are allowed to do.

> 🎯 **On the exam:** link the architecture to the benefit — subagents routing **through** the coordinator (not to each other) is precisely what gives you centralised observability, debugging, control, and policy enforcement.

**What do you lose without a coordinator?** Traceability and consistency: flows become untraceable and failures inconsistent. The hub centralises all messages and errors, unlocking debugging, control, and policy enforcement.

### Module 2 — Checkpoint Quiz

**Q1.** In hub-and-spoke orchestration, how do subagents communicate?
- A. Directly, peer-to-peer, sharing a common context
- B. Through a shared memory store they all write to
- C. They don't — all routing goes through the coordinator; subagents never talk directly
- D. Whichever subagent finishes first broadcasts to the others

**Answer: C.** The coordinator is the central brain. Subagents have isolated context and route everything through the hub — they never communicate directly.

**Q2.** What is the coordinator's lifecycle for a request?
- A. Delegate → decompose → aggregate
- B. Decompose → delegate → aggregate
- C. Aggregate → delegate → decompose
- D. Delegate → aggregate → decompose

**Answer: B.** The coordinator decomposes the goal into subtasks, delegates them to isolated subagents, then aggregates the results (resolving conflicts) into one response.

**Q3.** Why is **too-narrow** task decomposition dangerous?
- A. It uses too many parallel agents
- B. It makes subagents talk to each other
- C. It always doubles token spend
- D. It misses important dimensions, and isolated subagents can't detect the gap

**Answer: D.** Narrow decomposition drops whole perspectives. Because subagents only see their slice, none can notice what's missing — coverage is the coordinator's job: ask "what's missing?", add subtasks, validate.

**Q4.** You assign the **same** research task to three agents. What's the problem, and the fix?
- A. Overlap and wasted tokens; partition into distinct, non-overlapping scopes
- B. Nothing — redundancy improves quality
- C. The agents will deadlock; add a shared context
- D. Too few agents; run the full pipeline instead

**Answer: A.** Identical tasks produce overlapping work and wasted tokens. Partition the problem so each agent owns a unique, non-overlapping slice.

**Q5.** What makes a multi-agent system traceable and debuggable?
- A. Running every agent on every request
- B. Letting subagents share context so they self-correct
- C. A coordinator that centralises all messages and errors, enabling debugging, control, and policy enforcement
- D. Capping iterations so failures can't propagate

**Answer: C.** Without a coordinator, flows are untraceable and failures inconsistent. Routing everything through the hub centralises messages and errors, enabling debugging, control, and policy enforcement.

---

## Module 3 — Subagents, the Agent Tool & Sessions

Spawning isolated subagents, passing context to them explicitly, defining what an agent may do, and managing — forking, resuming, compacting — Claude Code sessions.

### 15. The Agent Tool

The **Agent tool** spawns a **subagent** — a fresh agent with its **own isolated context window** and its **own tool scope** — to handle a focused piece of work and report back.

- **Isolated context:** a subagent starts with a clean message history. The parent's conversation does not leak in; the subagent's messy intermediate steps do not leak back out.
- **Isolated tool scope:** you give each subagent only the tools it needs — a smaller, focused capability set than the parent.
- **Execution patterns:** subagents can run **in parallel** (several independent jobs at once) or **blocking** (the parent waits for the result before continuing).

> 🔑 **"Task" = legacy naming.** The Agent tool is still frequently called the `Task` tool, and that name still appears in code and traces. Treat **Agent tool** and **Task tool** as the same thing.

Flow: Parent agent → Agent/Task tool → Subagent (own context + tools) → distilled result back to parent.

Why delegate? Isolate context (keep the parent small and focused), focus the toolset (fewer tools = better tool choice), and parallelise independent work for speed.

### 16. Task Tool — Context Passing

A subagent receives **only the context you explicitly hand it**. There is **no implicit memory and no shared context** — so the spawning prompt must contain every instruction and every piece of data the subagent needs.

| ❌ Assumption that breaks | ✅ Reality |
|---|---|
| "The subagent can see the parent's conversation." | It cannot. It starts with a blank history. |
| "It remembers the file we were just editing." | It remembers nothing. State is not shared. |
| "A short prompt is fine; it has context." | The prompt **is** the context. Underspecify it and the subagent guesses. |

> 🚫 **Cardinal sin:** writing a vague delegation like *"finish the refactor we discussed"*. The subagent never heard the discussion. Pass the goal, the constraints, the relevant data, and the expected output shape — all in the prompt.

The worked example uses a narrow hand-off: a single **question string** goes in, a single **answer string** comes out.

```python
def run_subagent(client, question: str) -> str:
    # A BRAND-NEW conversation — the parent's history does NOT leak in.
    sub = Conversation(
        system="You are a research assistant. Answer in one short "
               "sentence using your tools. Be concise."
    )
    sub.add_user(question)        # <-- ALL context passed IN, explicitly, as a string
    answer = run_agent(client, sub, SUBAGENT_TOOL_SCHEMAS,
                       {"lookup_population": lookup_population},
                       label="subagent")
    return answer                 # <-- only this distilled string returns to the parent
```

> 🎯 **On the exam:** if an option claims a subagent "inherits", "shares", or "remembers" the parent's context, it is **wrong**. Context passing is always **explicit**.

### 17. Agent Definition

An **agent definition** is the **blueprint for a subagent**: its identity, role, and boundaries. It declares **what the agent can do before execution begins.**

| Field | What it sets |
|---|---|
| **name** | The agent's identifier — how it is invoked / referenced. |
| **description** | What it is for and *when* to use it (drives selection, like a tool description). |
| **system prompt** | Its standing instructions — its role, behaviour, and constraints. |
| **allowed / disallowed tools** | The exact tool scope — the capabilities it may (or may not) use. |

```yaml
name: research-assistant
description: Looks up focused facts and returns one concise sentence.
             Use when the main agent needs a fact it does not have.
system_prompt: |
  You are a research assistant. Answer the question in one short
  sentence using your tools. Be concise. Do not speculate.
allowed_tools:
  - lookup_population
disallowed_tools:
  - bash
  - file_write
```

> 🔑 **Identity + boundaries up front.** The definition decides the agent's role and limits *before* it ever runs — a smaller, well-chosen toolset keeps its decisions focused and its blast radius small.

### 18. Fork-Based Session Management

A management pattern: **copy a conversation's state, then branch it into independent sessions**. Each branch explores a different direction in isolation — preventing cross-contamination while enabling parallel exploration.

- **Copy the state:** duplicate an existing conversation as a starting point.
- **Branch into independents:** each copy becomes its own session that evolves separately.
- **Explore in isolation:** branch A tries one approach, branch B another — neither sees the other.

> 🔑 **Prevents cross-contamination.** Because branches are fully separate, a dead-end in one never pollutes another. Parallel exploration *without* shared-context leakage.

Mental model: like `git branch` for a conversation — shared history up to the fork point, then independent timelines.

### 19. Forking — The Benefits

Three benefits — memorise them as a triple: **resumable, parallel, isolated.**

| Benefit | What it buys you |
|---|---|
| **Own history & resumable state** | Each forked session keeps its own history and can be resumed independently later. |
| **Runs in parallel → faster** | Branches execute at the same time, so you get results sooner. |
| **Fully isolated → no leakage** | No shared context between branches → no cross-contamination. |

> 🎯 **On the exam:** the three forking benefits are **(1) own history / resumable**, **(2) parallel / faster**, **(3) isolated / no leakage**. An option that says forks "share context" or "merge automatically" is wrong.

### 20. Claude Code Sessions

A **session** is a **stateful conversation with the agent** — the accumulated history, tools, and outputs of one continuous interaction.

- Starting `claude` creates a **new** session.
- `/resume` **continues** an existing one (see concept 22).
- Sessions exist **locally or remotely**, and can be continued **across environments** — start on one machine, resume on another.

> 🔑 **Recall from Module 1:** the Messages API itself is *stateless*. A "session" is the message list the harness keeps and re-sends for you. Statefulness lives in the session, not the API.

### 21. Anthropic SDK vs Claude Agent SDK

Two ways to build an agent. The **Anthropic SDK** makes **direct API calls — you control everything** (the loop by hand). The **Claude Agent SDK** gives you a **built-in agent loop + tool plumbing** — a higher-level abstraction.

| | Anthropic SDK | Claude Agent SDK |
|---|---|---|
| **What it is** | Direct calls to the Messages API | A higher-level harness over the API |
| **The loop** | **You write it** (your `while` loop) | **Built in** — runs for you |
| **Tool dispatch** | You parse `tool_use` and dispatch | Handled by the SDK |
| **Message history** | You keep & re-send it | Managed for you |
| **Control vs convenience** | Maximum control, more code | Less control, far less code |

The worked example builds the *same* agent **twice**: once by hand on the Anthropic SDK (every gear visible), once on the Claude Agent SDK (the gears hidden).

```python
# Anthropic SDK (by hand): you own the loop
while True:
    response = client.messages.create(
        model=MODEL, tools=tool_schemas, messages=convo.messages)
    convo.add_assistant(response.content)
    if response.stop_reason != "tool_use":
        return text_of(response)
    for block in response.content:
        if block.type == "tool_use":
            convo.add_user([run_one_tool(block, tools)])
```

```python
# Claude Agent SDK: the SDK owns the loop
options = ClaudeAgentOptions(
    model=MODEL, system_prompt="...",
    mcp_servers={"study_tools": TOOLS_SERVER},
    allowed_tools=["mcp__study_tools__calculator"],
    max_turns=8)
async for message in query(prompt=user_request, options=options):
    ...   # NO while-loop, NO dispatch, NO message list
```

> 🔑 **When to choose which:** write the loop yourself (**Anthropic SDK**) when you need fine-grained control — approval gates, custom logging, conditional execution. Reach for the **Agent SDK** when you just want the loop, retries, and tool plumbing handled.

> 🎯 **On the exam:** Anthropic SDK = *direct API, you own the loop*; Claude Agent SDK = *built-in loop + tools, higher-level*. The Agent SDK is the one that runs the loop for you.

### 22. Resuming Sessions

**Resuming** restores a previous session and carries straight on. Use `/resume` inside Claude Code, or `claude --resume` from the terminal.

- It uses a `session_id` to **reload the full context and history**.
- Continuation **preserves state, tools, and prior outputs** — you pick up exactly where you left off.

Flow: earlier session → `/resume` (session_id) → full context reloaded → continue.

> 📍 **Resume vs fork:** resuming *continues the same session* (one timeline). Forking branches a *new* session from a past point (a second timeline). See concept 23.

### 23. Forking Sessions

A **fork branches from a past point into a NEW session**. It keeps the history up to that point, then **diverges independently** — enabling parallel exploration **without affecting the original**.

```
shared history ──●──●──◉ fork point
                          ├──▶ Branch A · approach 1
                          └──▶ Branch B · approach 2
```

One shared trunk up to the fork point; two new sessions diverge independently. The original is untouched.

> 🔑 **Fork = copy-then-branch.** It is the mechanism behind fork-based session management (concept 18): the branch inherits history *up to the fork point only*, then runs as its own isolated, resumable session.

### 24. /context

`/context` shows your **token usage and remaining context window** — a live read-out of how full the window is.

- Usage split across **messages**, **system** prompt, **tools**, and **skills**.
- An **auto-compact buffer** — space reserved up-front so there is room to summarise (auto-compact) before the window overflows.

> 📍 **Read it like a fuel gauge.** When messages dominate and the remaining window is shrinking, that is your cue to `/compact` or `/clear` before the auto-compact buffer is forced into action.

> 🔑 The **auto-compact buffer** reserves window space for summarisation — it is *not* the same as running `/compact` yourself. The buffer is the safety reserve; `/compact` is the manual action.

### 25. Compact and Clear

Two ways to manage context limits and prevent overflow: `/compact` **summarises** the conversation to reduce tokens; `/clear` **resets** the current chat.

| Command | What it does | Keeps |
|---|---|---|
| `/compact` | **Summarises** the conversation into fewer tokens, then continues | A condensed version of the history |
| `/clear` | **Resets** the current chat to a blank slate | Your **files and memory** (e.g. `CLAUDE.md`) — only the chat is reset |

> ⚠️ **Don't confuse them.** `/compact` *preserves the gist* by summarising; `/clear` *throws the conversation away* (but not your files or memory).

> 🎯 **On the exam:** `/compact` = summarise to reduce tokens; `/clear` = reset chat, **keeps files/memory**. Both manage context limits and prevent overflow.

### 26. Rename and Rewind

Two session controls for iteration: `/rename` **labels a session** for easier retrieval; `/rewind` **restores a previous state** in the history.

| Command | Purpose |
|---|---|
| `/rename` | Give a session a meaningful label so you can find and `/resume` it later. |
| `/rewind` | Restore the conversation to an earlier state — undo a bad path and re-try from a known-good point. |

> 📍 **Iteration control:** `/rewind` is your "undo the last few turns and try again" button — useful when the agent went down a wrong path. `/rename` keeps a growing list of sessions navigable for retrieval.

### Module 3 — Checkpoint Quiz

**Q1.** What two things does a subagent spawned by the Agent (Task) tool get its own copy of?
- A) The user's API key and billing account
- B) The parent's full conversation and all of its tools
- C) An isolated context window and its own tool scope
- D) A separate model alias and a separate API endpoint

**Answer: C.** The Agent tool spawns a subagent with an isolated context window and its own (usually smaller) tool scope. It does not inherit the parent's conversation.

**Q2.** How does a subagent receive the information it needs to do its job?
- A) It automatically inherits the parent's memory
- B) It reads a shared context buffer between agents
- C) It queries the parent agent mid-task for missing facts
- D) Only via context explicitly provided in its prompt

**Answer: D.** Subagents have no implicit memory and no shared context. Everything they know must be passed explicitly in the spawning prompt.

**Q3.** Which field of an agent definition sets which tools the subagent may use?
- A) The system prompt
- B) The allowed / disallowed tools
- C) The description
- D) The name

**Answer: B.** The allowed/disallowed tools field defines the agent's tool scope (capability). The system prompt sets behaviour; the description drives selection; the name identifies it.

**Q4.** Which statement about the Anthropic SDK vs the Claude Agent SDK is correct?
- A) Anthropic SDK = direct API calls (you write the loop); Agent SDK = built-in loop + tools
- B) Anthropic SDK runs the loop for you; Agent SDK only makes raw API calls
- C) Both run the agent loop automatically; they differ only in pricing
- D) Neither can call tools; both are chat-only

**Answer: A.** The Anthropic SDK is direct API access where you control the loop; the Claude Agent SDK is the higher-level abstraction that runs the loop and handles tool plumbing for you.

**Q5.** What is the key difference between resuming a session and forking one?
- A) Resuming creates a new session; forking continues the same one
- B) Both create new sessions that share live context
- C) Resuming continues the same session; forking branches a new, independent session from a past point
- D) Forking deletes the original session once the branch starts

**Answer: C.** `/resume` continues the same session (one timeline) by its `session_id`; a fork keeps history up to a past point then diverges into a new, isolated session, leaving the original untouched.

**Q6.** You're near the context limit but still need the conversation's content. Which command should you use?
- A) `/clear` — it resets the chat and frees the most space
- B) `/compact` — it summarises the conversation to reduce tokens
- C) `/rewind` — it restores an earlier state
- D) `/rename` — it labels the session

**Answer: B.** `/compact` summarises the history into fewer tokens while preserving the gist. `/clear` would discard the conversation entirely (though it keeps your files and memory).

---

## Module 4 — Claude Code Configuration & Permissions

Where settings live and which one wins, the full settings surface, and the permission-rule system — `allow`/`ask`/`deny`, rule syntax, modes, and the sandbox — that governs exactly what the agent may do.

### 27. Settings Scope & Precedence

Claude Code is configured through `settings.json` files at **four levels**. They merge, but when two disagree the **higher-priority scope wins** — and **Managed** (org-wide) always sits at the top.

Precedence (highest first): **Managed > User > Project > Local**

| Scope | Path | Who / where |
|---|---|---|
| **Managed** (highest) | org-deployed | Org-wide policy, enforced by IT — top priority |
| **User** | `~/.claude/settings.json` | Global personal — applies across all your repos |
| **Project** | `.claude/settings.json` | Shared in the repo — commit it, the team gets it |
| **Local** (lowest) | `settings.local.json` | Personal, this repo only — **not** in git |

- In a repo, `.claude/settings.json` is committed and shared with the team; `settings.local.json` stays out of version control for your personal overrides.
- **On the exam:** memorise **Managed > User > Project > Local**. Managed is highest and an org can use it to lock settings down so users and projects cannot override (see Lockdown).

### 28. The Settings Surface

A `settings.json` file can configure a broad surface. You don't need every key memorised, but you should recognise which area each setting belongs to.

| Area | What you can configure |
|---|---|
| **Auth** | API keys + AWS auth helpers; force login method / org |
| **Sessions** | Auto-delete old sessions; memory + plans storage paths |
| **Env** | Global variables for all runs (secrets, configs, flags) |
| **Models** | Set default model; limit available models; map to providers (e.g. Bedrock) |
| **Output** | Style (e.g. explanatory); language; optional git instructions |
| **Git** | Commit / PR attribution; co-author toggle |
| **Updates** | Startup announcements; update channel (latest / stable) |
| **UI** | Show duration; customise spinner text; tips + progress bar; reduce motion |
| **Agents** | Teammate mode — `auto` \| `tmux` \| `inline` |
| **Permissions** | `allow` / `ask` / `deny`; extra directories; default mode |
| **MCP** | Enable / disable servers; allowlist / denylist |
| **Plugins (Managed)** | Allow / block marketplaces; custom trust message |
| **Lockdown** | Managed-only → no user/project overrides |
| **Hooks** | On/off all hooks; limit URLs + env vars; run commands after actions |
| **Misc** | OTel headers; status line; `@file` autocomplete; respect `.gitignore` |

- Agents → teammate mode has exactly three values: `auto`, `tmux`, `inline`. Updates → channel is `latest` or `stable`.

### 29. Permission Rules — allow / ask / deny

Permission rules decide what the agent may do. There are **three levels**, evaluated in a fixed order where the **most restrictive match wins**.

| Level | Effect |
|---|---|
| `allow` | Auto-run — no prompt |
| `ask` | Confirm with the user first |
| `deny` | Block outright |

**Evaluation order: `deny` → `ask` → `allow`** (most restrictive wins). Check `deny` first; if no deny matches, check `ask`; only then `allow`. If no rule matches at all, fall back to the active permission mode (default = prompt on first use).

Rules apply to the agent's tools: **Bash, Read/Write, WebFetch, MCP, and Agents.**

- **Number-one exam fact:** if a command is both allowed and denied, `deny` takes precedence — it is always blocked.

### 30. Bash Wildcard Permissions

In a Bash rule, `*` is a **wildcard that matches any characters** and can appear **anywhere** in the command pattern. But the literal text — including **spaces** — must match too.

| Rule | Matches | Does NOT match |
|---|---|---|
| `Bash(npm run test:*)` | `npm run test:unit`, `npm run test:watch` | `npm run build` |
| `Bash(ls *)` | `ls src`, `ls -la /tmp` | `ls` (no trailing space + arg) |
| `Bash(git *)` | `git status`, `git commit -m "x"` | `gitk` |
| `Bash(*--force*)` | any command containing `--force` | commands without it |

- **Spaces matter.** `ls *` only matches the "`ls` space something" form — the bare command `ls` with no argument does not match.

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run test:*)",
      "Bash(git status)",
      "Bash(git diff:*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(*--force*)"
    ]
  }
}
```

### 31. Read & Edit Rules

`Read` and `Edit` rules are scoped by **file path**, and they **follow your `.gitignore` patterns** — ignored files are respected by default.

| Prefix | Means | Example |
|---|---|---|
| `//` | **Absolute** path | `Read(//etc/hosts)` |
| `~/` | **Home** directory | `Read(~/.zshrc)` |
| `/` | **Project root** | `Edit(/src/**)` |
| _(none)_ | **Current** directory | `Read(*.md)` |

- Mind the leading slashes: a **single** `/` is the **project root**, but a **double** `//` is a system **absolute** path.
- Because Read/Edit rules honour `.gitignore`, secrets and build artefacts you've already ignored are protected without extra deny rules. The "respect `.gitignore`" toggle lives in Misc.

### 32. WebFetch Rules

`WebFetch` rules are controlled **by domain** using the `domain:` syntax — and you can `allow`, `ask`, or `deny` per domain.

```json
{
  "permissions": {
    "allow": ["WebFetch(domain:docs.anthropic.com)"],
    "ask":   ["WebFetch(domain:github.com)"],
    "deny":  ["WebFetch(domain:example.com)"]
  }
}
```

- The pattern is `WebFetch(domain:example.com)` — scoping is **per domain**, not per full URL. A denied domain stays blocked even if another rule allows it.

### 33. MCP Rules

MCP permission rules use a **double-underscore** path: name the **server** to cover all of its tools, or add the **tool** to scope to one. Wildcards are supported.

| Pattern | Scope |
|---|---|
| `mcp__server` | **All tools** on that server |
| `mcp__server__tool` | **One specific tool** on that server |
| `mcp__server__*` | Wildcard — tools matching the pattern |

- Separator is the **double underscore** `__`: `mcp`, then server, then (optionally) tool. Stop after the server name and the rule covers **every** tool that server exposes.

### 34. Bare Tool Rules

A rule with **no parentheses** grants **full access** to that tool — every invocation, no scoping. Very broad, and easy to over-grant.

| Bare rule | Means |
|---|---|
| `Read` | Read **any** file |
| `Bash` | Run **any** command |
| `WebFetch` | Fetch **any** URL / domain |

- **Scope it down.** `Bash` on its own is a blank cheque to run anything. Prefer narrow patterns like `Bash(npm run test:*)` over the bare tool name — especially in an `allow` list.

### 35. The Tools

These are the tools permission rules govern. A `*` marks tools that can change state or reach the outside world — the ones to scope carefully.

| Tool | Does |
|---|---|
| **Agent** | Spawns subagents |
| **Bash** * | Runs commands |
| **Edit** * | Modifies files |
| **Read** | Reads files |
| **WebFetch** * | Fetches URLs |
| **WebSearch** * | Searches the web |
| **Write** * | Creates / overwrites files |
| **Grep** / **Glob** | Search & match (read-only) |
| **Task\*** family | `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` / `TaskStop` / `TaskOutput` |

- The asterisk (`Bash*`, `Edit*`, `WebFetch*`, `WebSearch*`, `Write*`) flags side-effecting tools. **Read**, **Grep**, and **Glob** are read-only.

### 36. Permission Modes

A **permission mode** sets the agent's default approval behaviour for the session. It is the fallback when no `allow`/`ask`/`deny` rule matches.

| Mode | Behaviour |
|---|---|
| `default` | Standard — prompts for permission on first use of each tool |
| `acceptEdits` | Auto-accepts file-edit permissions for the session |
| `plan` | Plan Mode — Claude can analyse but not modify files or execute commands |
| `dontAsk` | Auto-denies tools unless pre-approved via `/permissions` or `permissions.allow` rules |
| `bypassPermissions` | Skips all permission prompts (requires a safe environment) |

- **Don't muddle these two:** `dontAsk` auto-**denies** anything not already approved (safe-by-default); `bypassPermissions` auto-**allows** everything by skipping prompts entirely (dangerous-by-default).
- `bypassPermissions` should only run in a safe, isolated environment (e.g. a sandbox or throwaway container).

### 37. `--dangerously-skip-permissions`

A **session setting** that tells Claude Code to **stop prompting you for permissions** for the rest of that session — the flag form of bypassing approvals.

```bash
claude --dangerously-skip-permissions
```

- **The name is the warning.** With prompts off, the agent can run any command or edit any file without asking. Use it only in a disposable, sandboxed environment.
- Think of it as the CLI sibling of the `bypassPermissions` mode: same "skip all prompts" effect, scoped to the session.

### 38. Sandbox

A **sandbox** is a security mechanism that **separates running programs from your operating system** — a controlled environment where code can run without endangering the host.

What a sandbox provides:

- **Storage and memory scratch space** — a contained place to work.
- **Limited network access** — the program can't freely reach the outside world.
- **Restricted ability to inspect the host system** — it can't snoop on your real machine.

- The sandbox is the "safe environment" the dangerous modes assume. Running `bypassPermissions` or `--dangerously-skip-permissions` is only acceptable **inside** a sandbox.
- **On the exam:** a sandbox separates programs from the OS and gives them scratch storage/memory, limited network, and restricted host inspection. It's a containment boundary, not a permission level.

### Module 4 — Checkpoint Quiz

**Q1.** What is the correct precedence of settings scopes, highest first?
- A) Local > Project > User > Managed
- B) Managed > User > Project > Local
- C) User > Managed > Project > Local
- D) Project > Managed > Local > User

**Answer: B.** Managed (org-wide) is highest, then User (`~/.claude/settings.json`), then Project (`.claude/settings.json`), then Local (`settings.local.json`, not in git).

**Q2.** A Bash command matches both an `allow` rule and a `deny` rule. What happens?
- A) It runs — `allow` always wins
- B) The user is prompted to break the tie
- C) It is blocked — `deny` is checked first and most restrictive wins
- D) Whichever rule was written last wins

**Answer: C.** The order is `deny` → `ask` → `allow` and the most restrictive match wins, so a `deny` match blocks the command regardless of any `allow`.

**Q3.** Which permission mode lets Claude analyse a codebase but **not** modify files or run commands?
- A) `plan`
- B) `acceptEdits`
- C) `bypassPermissions`
- D) `dontAsk`

**Answer: A.** `plan` (Plan Mode) restricts Claude to analysis only — no file edits, no command execution.

**Q4.** What does the rule `mcp__github` cover?
- A) Only the `github` tool, not the server
- B) All WebFetch calls to github.com
- C) A single named tool on the server
- D) All tools provided by the `github` MCP server

**Answer: D.** `mcp__server` (no tool segment) covers every tool on that server. `mcp__server__tool` would scope to one specific tool.

**Q5.** In a permission rule, what does the path prefix `//` mean?
- A) Project root
- B) An absolute (system) path
- C) The home directory
- D) The current directory

**Answer: B.** `//` = absolute path. A single `/` = project root, `~/` = home, and no prefix = current directory.

**Q6.** Which statement about `bypassPermissions` is correct?
- A) It auto-denies any tool not already approved
- B) It restricts Claude to read-only tools
- C) It skips all permission prompts and needs a safe environment
- D) It only auto-accepts file edits, nothing else

**Answer: C.** `bypassPermissions` skips all prompts and is only safe in an isolated environment. `dontAsk` is the auto-deny mode; `acceptEdits` is the edits-only one.

---

## Module 5 — Coordination Patterns & Findings

How a coordinator passes findings between agents without losing provenance, parallelises independent work, enforces a quality bar, and chooses between rigid step-by-step control and adaptive, goal-driven investigation.

### 39. The Raw Findings Dilemma

When agents pass **raw text** to one another, they **lose attribution and provenance**. A downstream agent reads a claim but can no longer tell *which source* supports it.

Why raw text fails:

- **Attribution is lost:** "Revenue grew 12%" arrives with no link to the document, URL, or tool call that produced it.
- **Provenance is unverifiable:** a synthesiser can't trace a claim back to its source, so it can't check or defend it.
- **Trust collapses:** outputs become unverifiable, reliability is weak, and errors propagate silently across the chain.

> 🚫 **The dilemma:** raw text mixes content and conclusions but throws away the metadata that makes them checkable. The more agents you chain, the worse it compounds — each hand-off strips a little more context.

### 40. Structured Findings (the Solution)

Pass **structured finding objects** that keep **content + metadata together**. Each finding carries fields like `id`, `type`, `confidence`, and source details — so traceability survives every hand-off.

| Field | Purpose |
|---|---|
| `id` | Stable identifier so other findings (and the synthesis) can reference it. |
| `type` | What kind of finding it is (fact, metric, quote, risk…), so it can be routed and validated. |
| `confidence` | How sure the agent is — lets the coordinator weight or re-check weak findings. |
| `source` | Where it came from (URL, document, tool call) — the provenance that makes it verifiable. |

```json
{
  "id": "f-014",
  "type": "metric",
  "claim": "Q3 revenue grew 12% year-on-year",
  "confidence": 0.86,
  "source": {
    "title": "FY25 Q3 Results",
    "url": "https://example.com/q3-results.pdf",
    "page": 4,
    "agent": "research-subagent-2"
  }
}
```

> 🔑 Because content and metadata travel **together**, downstream agents can **trace, validate, and reliably synthesise**. A synthesiser can cite the exact source; a checker can re-verify low-`confidence` findings; nothing is taken on faith.

> 🎯 **On the exam:** the fix for the raw-findings problem is **structured findings (content + metadata together)**, *not* "summarise the text more" or "use a bigger model". Look for `id` / `confidence` / `source` as the giveaway.

### 41. Parallel Tool Calls

Run subagents **in parallel when tasks are independent** (no shared dependencies). Parallelism improves **speed and coverage** by exploring multiple paths at once.

| Run in parallel when… | Run sequentially when… |
|---|---|
| Tasks are **independent** — no task needs another's output. | An output **depends on an earlier step** (later step consumes the result). |
| You want broad **coverage** fast (research several topics at once). | Order matters — e.g. fetch → then validate → then synthesise. |

Flow: Coordinator → [Subagent A, Subagent B, Subagent C in parallel] → Merge findings.

> 📍 Default to parallel for independent work; fall back to sequential **only** when a real dependency forces ordering. Forcing independent tasks to run in series just wastes wall-clock time.

### 42. Procedural (Rigid) Coordination

**Step-by-step prompts** turn the coordinator into a **script runner** — it executes a fixed sequence instead of reasoning about what's actually needed.

Where it breaks:

- **Missing or unexpected inputs:** a step assumes data that isn't there and the flow stalls or errors.
- **Tasks don't fit the flow:** a real request doesn't match the prescribed steps, so the coordinator forces a bad fit.
- **Wasted calls & poor adaptability:** steps run even when irrelevant; nothing adjusts to the situation.

> ⚠️ Rigid procedural prompts feel safe because they're predictable, but they remove the model's biggest asset — its ability to reason about the goal. Concept 43 is the fix.

### 43. Goal-Oriented Coordination

Define a clear **end goal + quality bar**, not fixed steps. The coordinator decides **which agents to use, and in what order, dynamically** — for flexibility, better reasoning, and efficient execution.

**Procedural (42):**
```
Step 1: call search_web with the query.
Step 2: call fetch_page on result 1.
Step 3: call summarise on the page.
Step 4: return the summary.
# Breaks if search returns nothing, or the
# task needs two pages, or no fetch is needed.
```

**Goal-oriented (43):**
```
GOAL: produce a sourced answer to the user's question.
QUALITY BAR: at least 3 independent sources,
  each finding carries a source, no claim unsupported.

You have these agents: search, fetch, validate, synthesise.
Decide which to use and in what order. Stop when the
quality bar is met.
```

| | Procedural (42) | Goal-oriented (43) |
|---|---|---|
| **Prompt specifies** | The exact steps | The goal + quality bar |
| **Coordinator's job** | Run the script | Reason about how to reach the goal |
| **Order of agents** | Fixed in advance | Decided dynamically |
| **Handles surprises** | Poorly — breaks | Well — adapts |

> 🔑 Give the model a goal and a bar, not a recipe. This is the coordination-layer twin of model-driven control from Module 1: tools + a goal, let the model choose the path.

### 44. Quality-Criteria-Driven Coordination

Make the **quality criteria explicit** and treat them as the **definition of done** — coverage, sources, depth. The coordinator evaluates outputs against these *before* finishing.

Flow: Gather findings → Evaluate vs criteria → Met? finish; if not, loop back (identify gap, re-delegate targeted work).

- **Coverage:** are all required aspects of the question addressed?
- **Sources:** is every claim backed by a traceable source (links to concept 40)?
- **Depth:** is the analysis detailed enough, not superficial?

> 📍 **The loop:** if criteria aren't met, the coordinator identifies the specific gaps and re-delegates targeted work to close them — rather than declaring done or restarting from scratch.

### 45. Prompt-Based Guidance

Instructions in the prompt **guide** behaviour but do **not enforce** it. The model can **drift or ignore the rules** — there's no guarantee of the correct sequence.

- "Always validate before saving" in a prompt is a **suggestion**, not a constraint.
- The model usually complies, but under pressure (long context, ambiguous input) it can skip or reorder steps.
- **No guarantee** the prescribed order actually happens.

> ⚠️ Prompt-based guidance is the right tool for preferences and style. It is the wrong tool when correctness depends on a step never being skipped — for that you need enforcement (concept 46).

### 46. Programmatic Enforcement (Gates)

**Hard rules** in code make steps **impossible to skip**. A gate **blocks execution if prerequisites are missing**, guaranteeing correct order and reliability.

| | Prompt-based guidance (45) | Programmatic enforcement (46) |
|---|---|---|
| **Where it lives** | In the prompt | In code (a gate / check) |
| **Strength** | A suggestion — can be ignored | A hard rule — cannot be bypassed |
| **If prerequisite missing** | Model *might* proceed anyway | Execution is **blocked** |
| **Guarantee of order** | None | Yes |

```python
def gate_before_deploy(state):
    # Hard rule: cannot deploy until tests have passed.
    if not state.get("tests_passed"):
        raise BlockedError("Prerequisite missing: tests not passed")
    # otherwise allow the tool call through
```

> 🔑 Guidance vs enforcement is a spectrum of trust: prompts ask nicely; gates make it structurally impossible to do the wrong thing. Use gates wherever a skipped step would be unsafe or unrecoverable.

### 47. The Hooks Pattern

**Hooks** run **before and after tool calls**. They're how you *implement* gates and state updates — a single, central place for control logic.

- **Before a tool call:** the place to run a **gate** — validate inputs, enforce rules, block if prerequisites are missing.
- **After a tool call:** the place to **update state** — store the result, record that a step happened.
- **Central:** control logic lives in one place instead of being scattered through prompts and tool code.

> 🔑 Hooks turn the soft guidance of concept 45 into the hard enforcement of concept 46. They are the mechanism; gates and state updates are what you build with them. The two hook points are named in concept 49.

### 48. The Handoff Protocol

A **handoff protocol** is a **structured package assembled before escalation** — to another agent or a human. It bundles **context, actions taken, and blockers** so the receiver doesn't have to dig through history.

| Part | Answers |
|---|---|
| **Context** | What is the situation / goal? |
| **Actions taken** | What has already been tried? |
| **Blockers** | What's stuck, and why escalate now? |

> 📍 **Why it matters:** escalation without a handoff forces the next agent (or person) to reconstruct everything from raw history — slow and error-prone. The structured package is to escalation what structured findings (40) are to synthesis: keep the useful state, drop the noise.

### 49. PreToolUse / PostToolUse

The two hook points are **`PreToolUse`** (before the tool runs) and **`PostToolUse`** (after). Pre *validates and enforces*; Post *stores results and updates state* — together they enable controlled, observable flows.

Diagram: `PreToolUse` (validate · enforce — can BLOCK) → Tool call (runs) → `PostToolUse` (store · update).

| | `PreToolUse` | `PostToolUse` |
|---|---|---|
| **When** | Before the tool runs | After the tool returns |
| **Job** | Validate inputs, enforce rules (the gate) | Store results, update state |
| **Can block?** | Yes — stop the call if prerequisites missing | No — the call already happened |

> 🎯 **On the exam:** if a question asks where to enforce a rule or block a step, the answer is `PreToolUse`. If it asks where to record a result or update state, it's `PostToolUse`. Don't mix them up.

### 50. Prompt Chaining

**Prompt chaining** is a **fixed sequence of agent steps** — the output of one step feeds the next. Best for **predictable, repeatable workflows**.

Flow: Step 1 → Step 2 → Step 3 (same path every run).

- Each step is **fixed and ordered** in advance.
- Ideal when the task is **predictable and repeatable** — the steps don't change run to run.
- It's the workflow cousin of **procedural coordination (42)**: reliable when steps are known, brittle when they aren't.

> 📍 Prompt chaining isn't a bad thing — for a well-understood, stable task it's the right choice. The mistake is using it for open-ended work where the next step can't be known in advance. That's what concept 51 handles.

### 51. Dynamic Adaptive Decomposition & the Adaptive Investigation Plan

Decide the **next step based on findings so far** — flexible, not a fixed sequence. **Findings generate new questions**; you **stop when no new signal emerges**.

Two names, one idea:

| Term | Emphasis |
|---|---|
| **Dynamic Adaptive Decomposition** | How the work is split: the *next step* is decided from the findings, not pre-planned. |
| **Adaptive Investigation Plan** | How it terminates: each finding spawns new questions; you *stop when no new signal emerges*. |

Flow: Act → Findings → New questions? loop back; when no new signal → stop.

| | Prompt chaining (50) | Adaptive decomposition (51) |
|---|---|---|
| **Steps** | Fixed in advance | Decided from findings |
| **Best for** | Predictable, repeatable tasks | Open-ended investigation |
| **Stops when** | The last step finishes | No new signal emerges |

> 🔑 Adaptive decomposition pairs naturally with goal-oriented coordination (43) and a quality bar (44): chase the findings, keep asking the questions they raise, and stop when the bar is met and no new signal is appearing.

### Module 5 — Checkpoint Quiz

**Q1. What is lost when agents pass raw text findings to one another?**
- A. Nothing — text is the most reliable format
- B. Attribution and provenance — claims can't be traced to a source
- C. Only the formatting
- D. The model's reasoning ability

**Answer: B.** Raw text strips the link between a claim and its source, so downstream agents can't trace, validate, or trust it — low reliability.

**Q2. Which fields make a structured finding traceable and verifiable?**
- A. `temperature`, `max_tokens`, `model`
- B. Just a longer block of text
- C. `id`, `type`, `confidence`, and source details — content + metadata together
- D. A bigger model and a system prompt

**Answer: C.** Structured findings keep content and metadata together — `id`, `type`, `confidence`, `source` — enabling traceability, validation, and reliable synthesis.

**Q3. When should subagents run in parallel rather than sequentially?**
- A. When the tasks are independent — no task needs another's output
- B. Always, regardless of dependencies
- C. Only when one step's output feeds the next
- D. Never — sequential is always safer

**Answer: A.** Parallelise independent tasks for speed and coverage. Use sequential flow only when an output depends on an earlier step.

**Q4. Why is procedural (rigid) coordination fragile compared with goal-oriented coordination?**
- A. It's too slow to run
- B. It uses too many tokens by definition
- C. It can't call subagents at all
- D. Fixed steps break on missing/unexpected inputs; goal-oriented lets the coordinator decide agents and order dynamically

**Answer: D.** Procedural prompts make the coordinator a script runner that breaks when inputs or tasks don't fit. Goal-oriented gives a goal + quality bar and lets the coordinator adapt.

**Q5. You must guarantee a step is never skipped. Where do you enforce it?**
- A. In the prompt as an instruction — prompt-based guidance
- B. As a gate in `PreToolUse` — programmatic enforcement that blocks if prerequisites are missing
- C. In `PostToolUse`, after the tool has run
- D. You can't — the model decides

**Answer: B.** Prompt guidance can be ignored. A hard gate in `PreToolUse` can block execution before the tool runs, guaranteeing order. `PostToolUse` is for storing results, too late to block.

**Q6. An adaptive investigation plan stops when…**
- A. A fixed number of steps has run
- B. The first finding is returned
- C. No new signal emerges — the latest findings stop raising new questions
- D. The prompt chain reaches its last step

**Answer: C.** Adaptive decomposition decides each next step from the findings and stops when no new signal emerges — unlike prompt chaining, which ends at a predetermined last step.

---

## Module 6 — Resilience: Errors, Recovery & Escalation

How agents stay reliable when tools fail, sessions reset, and system prompts quietly conflict with the tools — plus knowing when to resolve autonomously versus hand off to a human.

### 52. Resuming Sessions (No Memory)

After a true reset, the model has **no memory of past state**. It only knows what is in the message history you send it *this* turn — so to resume work you must hand it a **fresh, structured summary** that rebuilds the context it needs.

**Two different things called "resume":**

| Mechanism | What actually happens | What the model sees |
|---|---|---|
| Reloading history (e.g. `/resume` in the CLI) | The full prior transcript is replayed back into the context window | Everything — it picks up exactly where it left off |
| True reset (new process, lost state, fresh request) | Nothing carries over; the conversation starts empty | **Nothing** — unless you put a summary in the first message |

The API is **stateless**. "Memory" is an illusion created by re-sending message history. When that history is gone, so is the memory — the only state the model has is the text in front of it.

**Rebuild context with a structured summary.** Don't dump the raw transcript back in — it's noisy and may not even survive the reset. Hand the model a tight, structured brief: the goal, what's already done, what's outstanding, and any decisions or blockers it must respect.

```json
{
  "goal": "Migrate the billing service to Cloud Run",
  "completed": ["Dockerised the app", "Wrote the deploy.yaml"],
  "outstanding": ["Wire up Secret Manager", "Smoke-test the live URL"],
  "decisions": ["DB stays on Cloud SQL — do NOT migrate it"],
  "blockers": ["Waiting on IAM role from platform team"]
}
```

This is the same shape as the **Handoff Protocol** from Module 5 — context, actions taken, blockers — but pointed at *your future self* instead of a human. A reset is just a handoff to a model with amnesia.

> **Self-test:** `/resume` reloads the whole transcript — so when do you ever need to write a summary? **When the transcript is gone or unusable**: a crashed process, a new request with no prior thread, or a context window that can't fit the full history.

### 53. System Prompts vs Tools

A **system prompt can override tool intent.** If the instructions say one thing and the tools imply another, the model is caught between them — watch for **hidden constraints and conflicts** between what the prompt forbids and what the tools invite.

**How the conflict shows up:**

- **Prompt forbids, tool exists:** "Never issue refunds" in the system prompt, but a `process_refund` tool is in the toolset. The model hesitates, half-uses it, or contradicts itself.
- **Prompt mandates, tool can't deliver:** "Always cite a source URL," but the search tool returns no URLs. The model invents one to satisfy the prompt.
- **Silent narrowing:** a constraint buried deep in a long system prompt quietly disables a tool the user is explicitly asking for.

Recall from Module 3: **tool presence is implicit permission** — if a tool exists, the model assumes it may be used. A system prompt that says "don't" while the tool says "you can" creates exactly the conflict that produces erratic behaviour.

**Resolving it:**

1. **Make the system prompt and toolset agree.** If a tool must not be used in some context, either remove it from that context or state the boundary in the tool's own `description`.
2. **Be explicit about precedence.** Spell out which wins ("policy overrides tools") rather than leaving the model to guess.
3. **Audit long prompts for hidden constraints** before blaming the model — the bug is often a line you forgot you wrote.

> **On the exam:** if an agent ignores a tool it clearly has, or "refuses" a reasonable request, suspect a **system-prompt constraint conflicting with tool intent** before assuming a model fault. The fix is alignment, not a bigger model.

### 54. MCP `isError` — Tool Failed ≠ Loop Failed

A tool **failing is not the loop failing.** When a tool errors, you don't crash — you flag it and feed the result back. The agent **still receives the result and keeps reasoning**, choosing what to do next from the error itself.

**The `is_error` flag.** An error is returned *through* the normal channel: a `tool_result` block with `is_error: true` (MCP surfaces this as `isError`). The model sees the failure as data and continues the loop, exactly as it would with a success.

```python
messages.append({"role": "user", "content": [{
    "type": "tool_result",
    "tool_use_id": block.id,
    "is_error": True,                      # MCP: isError
    "content": "rate_limited: retry after 2s"
}]})
# Loop continues — the model reads the error and decides the next move.
```

**Anti-pattern:** letting a tool exception bubble up and kill the agent loop. A failed call is information to reason about, not a fatal crash. Catch it, set `is_error`, append it, continue.

**Error type → action mapping.** Don't treat every error the same. Map the error *type* to a deliberate action:

| Error type | Action | Example |
|---|---|---|
| Transient (`rate_limited`, `internal`) | **Retry** — wait, then try again | API throttle, brief outage |
| Bad input (`validation`) | **Fix** — correct the arguments and re-call | Wrong enum value, missing field |
| Dead end (`not_found`, no tool) | **Escalate** — ask the user or hand off | Record doesn't exist; `cannot_progress` |

> **On the exam:** "the tool returned an error — what should the agent do?" The answer is almost never "stop the loop". It's **receive the result, keep reasoning, and map the error type to retry / fix / escalate.**

### 55. Retryable Flag

**Not all errors should be retried.** A structured error carries a `retryable` flag so the caller branches deterministically: `retryable: true` → safe to try again; `retryable: false` → **communicate, don't loop.**

**The structured error object.** Build on Module 3: every tool returns the *same shape*, and failures carry a small, branchable object — a fixed `category` plus the `retryable` flag.

```json
{
  "ok": false,
  "error": {
    "category": "rate_limited",   // validation | not_found | rate_limited | internal
    "retryable": true,            // could trying again possibly help?
    "message": "Quota exceeded — retry after 2s"
  }
}
```

| Category | `retryable` | What the caller does |
|---|---|---|
| `validation` | `false` | Fix the arguments and call again |
| `not_found` | `false` | Stop, or ask the user / escalate |
| `rate_limited` | `true` | Wait, then retry |
| `internal` | `true` | Retry later; alert if it persists |

Two failure modes the flag prevents: **blindly retrying forever** a permanent error (a `validation` failure will *never* succeed on retry), and **giving up on a blip** (a `rate_limited` error just needs a short wait). `retryable: false` means *communicate the problem, don't loop on it.*

> **Self-test:** a tool returns `not_found` with `retryable: false`. Retrying it three times — right or wrong? **Wrong.** Another attempt cannot help — the record simply isn't there. Communicate the dead end: ask the user for a different identifier, or escalate.

### 56. Subagent Failure Recovery

When a subagent hits an error, **don't propagate everything upward.** A subagent that forwards every blip hands the coordinator a useless failure. The resilient pattern is local: **retry → fallback → escalate only if exhausted.**

**Bad vs better:**

| ❌ Bad: propagate everything | ✅ Better: recover locally first |
|---|---|
| Any tool error bubbles straight up to the coordinator | Subagent **retries** transient errors itself |
| Coordinator gets "subagent failed" with no detail or remedy | If retry fails, it **tries a fallback** (alternate tool / source) |
| Whole job stalls on a recoverable blip | Only **escalates when options are exhausted** — with the reason attached |

Recovery flow:

```
Tool error in subagent
        ↓
1 · Retry locally (if retryable) ──success──> Return result ✓
        ↓
2 · Try fallback (alternate tool/source)
        ↓
3 · Escalate (options exhausted, with reason)
```

Recover at the lowest level possible. The coordinator only hears about a failure once all local options are exhausted — and then with the reason attached.

**Tie it back: confidence-based routing & human-in-the-loop.** "Escalate only if exhausted" is the same instinct as the escalation example: an agent must know **when not to act**. Two layers decide the handoff, applied *in order*:

**Layer 1 — hard triggers (override confidence):**
- `policy_gap` — the request needs an exception you have no authority to grant
- `customer_request` — the customer explicitly asked for a human
- `cannot_progress` — the agent lacks the tools/info to proceed

These are about **authority and scope**, not certainty — a confident answer to a question you shouldn't answer is the most dangerous output an agent can produce.

**Layer 2 — confidence-based routing (in-scope tickets only):**

| Confidence | Route |
|---|---|
| ≥ 0.85 | **Auto-resolve** — act, no human |
| ≥ 0.60 | **Human review** — agent drafts, a person approves before sending |
| < 0.60 | **Escalate** — hand to a human |

The middle **human-review** band is the in-the-loop sweet spot. Every escalation should carry its **reason** and a **draft** — exactly the Handoff Protocol package from Module 5 — so the human starts informed, not cold. Thresholds are a **business decision** set by risk appetite: a refund bot wants a very high auto-act bar; an FAQ bot can relax it.

> **On the exam:** the better recovery pattern is **retry locally → fallback → escalate only if exhausted**. "Propagate every error to the coordinator" is the wrong answer. And remember the three hard escalation triggers *override* confidence.

### Module 6 — Checkpoint Quiz

**Q1.** A process running your agent crashes and restarts with an empty conversation. How do you resume the work?
- A. The model remembers the last state automatically — just continue
- B. Hand it a fresh, structured summary (goal, completed, outstanding, blockers) in the first message
- C. Increase `max_tokens` so it can recall more
- D. Switch to a larger model so it doesn't forget

**Answer: B.** After a true reset the model has **no memory** of past state — the API is stateless. You rebuild context with a structured summary. (`/resume` works only when the transcript still exists to reload.)

**Q2.** Your agent has a `process_refund` tool but keeps refusing to use it. The system prompt says "never issue refunds." What's the most likely diagnosis?
- A. The model is too small — upgrade it
- B. The tool schema is malformed
- C. A system-prompt constraint is conflicting with the tool's intent
- D. The API is stateless so it forgot the tool exists

**Answer: C.** System prompts can override tool intent. The prompt forbids what the tool invites — align them (remove the tool from that context or state precedence) rather than blaming the model.

**Q3.** An MCP tool returns a result with `isError: true`. What should the agent loop do?
- A. Crash — a tool error means the loop has failed
- B. Silently drop the result and re-issue the same call
- C. Return the error text to the user as the final answer
- D. Receive the result, keep reasoning, and map the error type to retry / fix / escalate

**Answer: D.** A tool failing is not the loop failing. The error comes back as a `tool_result` with `is_error` set; the model reads it as data and chooses the next action.

**Q4.** A tool returns `{"category": "not_found", "retryable": false}`. What's the correct response?
- A. Don't retry — communicate the dead end (ask the user / escalate)
- B. Retry immediately three times, then give up
- C. Retry with exponential backoff
- D. Treat it as success and continue

**Answer: A.** `retryable: false` means another attempt cannot help. Communicate, don't loop. Only `retryable: true` errors (`rate_limited`, `internal`) are worth retrying.

**Q5.** A subagent's tool call fails. What is the resilient recovery pattern?
- A. Immediately propagate every error up to the coordinator
- B. Abort the whole job on the first failure
- C. Retry locally → try a fallback → escalate only if exhausted
- D. Loop on the failing call until it eventually works

**Answer: C.** Recover at the lowest level: retry transient errors, try a fallback, and escalate only when options are exhausted — with the reason attached. Propagating everything hands the coordinator a useless failure.

---

## Module 7 — Tool Design & MCP

How to design tools an agent can actually use well — naming, schemas, `tool_choice`, and the MCP layer that exposes them.

### 57. Too Many Tools

More tools ≠ more capability. Every tool you add enlarges the decision surface the model must reason over — past a point it *degrades* selection rather than improving it.

What a bloated toolset does:

| Symptom | Why it happens |
|---|---|
| Confuses selection | Overlapping/near-identical tools → the **wrong tool** gets chosen. |
| Increases hesitation | Too many options → the model stalls and emits **unnecessary clarifications** instead of acting. |
| Encourages hallucinated combos | The model stitches together tool sequences that don't make sense — **invented combinations** that fail at runtime. |

**Rule:** prefer a **small, general-purpose toolset** over many niche ones. A handful of broad, well-described tools beats a drawer full of single-use ones.

### 58. Tools Specialisation Misuse

If a tool exists, the model assumes it should be used. Tool *presence* reads as implicit *permission* — so giving an agent a tool it shouldn't touch is itself a design error.

- **Presence = permission:** the model treats every tool in its list as fair game.
- **Role mismatch → wrong usage:** hand a tool to an agent whose job doesn't call for it and it will use it anyway, badly.
- **Classic example:** a **synthesis agent** given a **search** tool will start searching — duplicating work, drifting off-role, burning context.

**Design implication:** scope each agent's toolset to its role. Don't expose a tool "just in case" — its mere availability invites misuse.

> **On the exam:** "a coordinator's synthesis subagent is doing its own searches" → the fix is **remove the search tool from that agent**, not to prompt it harder. Presence = permission.

### 59. Tool Choice

The `tool_choice` parameter controls how much freedom the model has over whether and which tool to call — from fully flexible to fully forced.

| `tool_choice` | Behaviour | Use it for |
|---|---|---|
| `AUTO` | Model decides freely — **flexible but unpredictable**. | **General agents** (default). |
| `ANY` | **Guarantees some tool use** — must call *a* tool, picks which. | You need an action, not chat (e.g. selection evals). |
| `TOOL` | **Fully deterministic** — forced to call *one named* tool. | **Strict pipelines / enforcement** (e.g. extraction). |
| `NONE` | **No tools at all** — pure reasoning / text only. | Forcing the model to think/answer without acting. |

Left → right (NONE → AUTO → ANY → TOOL) = increasing constraint on the model's freedom.

Forcing a single tool (the trick behind structured extraction):

```python
response = client.messages.create(
    model=MODEL,
    tools=[extract_invoice_schema],
    tool_choice={"type": "tool", "name": "extract_invoice"},  # MUST call this tool
    messages=messages,
)
# Claude is guaranteed to "call" extract_invoice; the data arrives as its input.
```

Mapping to remember: **AUTO** = general agents, **ANY** = guarantee a tool, **TOOL** = forced/deterministic pipeline, **NONE** = pure reasoning. Forcing `ANY` is also the clean way to run a tool-*selection* eval.

### 60. MCP Discovery

When you connect MCP servers, all their tools flatten into one list. The model sees a single pool — no grouping by server, no awareness of where a tool came from.

- **No grouping by server:** tools from different servers sit side by side, undifferentiated.
- **No awareness of origin:** the model can't tell which server a tool belongs to.
- **One large decision surface:** every connected tool adds to the same selection problem.

**Risk:** *tool explosion without structure* — connect enough servers and you recreate the "Too Many Tools" problem at scale.
**Mitigation:** limit the servers you connect, or filter the tools before exposing them to the model.

### 61. MCP Tools vs Resources

An MCP server exposes two very different things: **Tools** (actions — *do* something) and **Resources** (data — *read* something). They differ in *who triggers them* and *when*.

| | Tools | Resources |
|---|---|---|
| Purpose | **Actions** — DO something | **Data** — READ something |
| Trigger | **Reactive** — run when **Claude decides** to call them | **Proactive** — fetched up front (e.g. when **@-mentioned**) |
| Example | Update a document, send an email, run a query | Load a document's contents into context |

Resource URIs:

- **Direct** — a static URI for one fixed resource, e.g. `docs://documents`.
- **Templated** — a parameterised URI, e.g. `docs://documents/{doc_id}`; the SDK parses the parameter and passes it to the handler.

**One-liner:** Tools are reactive actions Claude chooses to invoke; Resources are proactive data pulled in (often by an `@`-mention) *before* the model reasons.

### 62. Built-In Tools

The mental model for a coding agent's built-in tools is simple: **file system + shell primitives.** Five operations cover most of what an agent does.

| Tool | What it does |
|---|---|
| Read | Load a file into context. |
| Grep | Search *inside* files (by content). |
| Glob | Find files *by pattern* (by name). |
| Edit | Modify files — **needs permission**. |
| Bash | Execute commands. |

- **Grep vs Glob:** Grep searches *content*; Glob matches *names*. Easy to mix up.
- **Edit needs permission** — it mutates files; Read/Grep/Glob are read-only. Bash runs arbitrary commands and is the other one to treat carefully.

### 63. JSON Schema Format

JSON Schema defines **structure, not behaviour**. It's a general data-validation specification (not ML-specific) that the tool-calling world adopted to describe a function's arguments.

| Keyword | Meaning |
|---|---|
| `type` | The **data shape** (object, string, integer, array…). |
| `properties` | The **fields** and each field's own schema. |
| `required` | Which fields **must exist**. |
| `enum` | The **allowed values** for a field. |

```json
{
  "type": "object",
  "properties": {
    "status": {
      "type": "string",
      "enum": ["shipped", "pending", "cancelled"]
    }
  },
  "required": ["status"]
}
```

Schema describes shape — it can demand `total: integer` but cannot demand the total equal the line-item sum. That semantic gap is why a validation-retry loop still matters.

### 64. Tool Input Schema

The input schema is **how Claude calls your tool**. The model generates its arguments to match this schema, so it must be clear and constrained — vague schemas produce vague calls.

A tool definition = three parts: **name** + **description** (3–4 sentences: what it does, when to use it, what it returns) + **input_schema**.

The three pieces that make a schema work:

| Piece | What it buys you |
|---|---|
| `enum` | **Reduces ambiguity** — fences the model into a fixed set (and signals intent). |
| `required` | **Enforces completeness** — the model can't omit the field. |
| `description` | **Guides decision-making** — at field level and the tool's overall description. |

```python
{
  "name": "search_orders",
  "description": ("Look up customer ORDERS by fulfilment status (shipped, "
                  "pending, cancelled). Use for questions about deliveries "
                  "or order state, e.g. 'which orders are still pending?'. "
                  "NOT for finding products or customer accounts."),
  "input_schema": {
    "type": "object",
    "properties": {
      "status": {"type": "string",
                 "enum": ["shipped", "pending", "cancelled"],
                 "description": "The order status to filter by."}
    },
    "required": ["status"]
  }
}
```

> **On the exam:** "Why is Claude calling the wrong tool?" → the **descriptions don't differentiate**. The Python behind a tool is invisible at selection time; the model chooses almost entirely from the `description`. Add explicit boundaries ("NOT for … — use X").

### 65. Input Schema Tips — Conditional Logic

You can **simulate conditional logic without code** — make an otherwise-optional field *conditionally required* by stating the rule in its `description`.

JSON Schema's `required` is all-or-nothing; it can't say "required only when another field has a certain value". But the model reads descriptions, so you encode the rule there:

```json
{
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "enum": ["billing", "technical", "other"]
    },
    "category_detail": {
      "type": "string",
      "description": "Required when category = 'other'. Describe the issue."
    }
  },
  "required": ["category"]
}
```

`category_detail` stays optional in the schema, but the description tells the model to fill it in *only when* `category = "other"`. The model honours the prose rule — logic without writing a validator.

### 66. Input Schema Tips — Validation Layer

A schema is **not full validation**. It guarantees shape and types, but real correctness — business rules, ranges, cross-field consistency — needs a proper validation layer on top.

- **Use validation tooling:** Pydantic / Instructor to define and enforce models in code.
- **Validate both directions:**
  - **Input to the model** — the arguments/data you send in.
  - **Output from the model** — what it hands back.

**Schema-valid ≠ correct.** A schema can require `total_pence: int` but cannot require it equal the line-item sum. Catch that in your validation layer, and if it fails, feed the *specific* problem back and retry.

### Module 7 — Checkpoint Quiz

**Q1.** An agent keeps choosing the wrong tool and asking for clarification before acting. Most likely cause?
- A. The model is too small — upgrade to Opus
- B. `tool_choice` is set to `NONE`
- C. Too many tools — a bloated decision surface confuses selection and increases hesitation
- D. The tools lack `required` fields

**Answer: C.** More tools ≠ more capability. A large toolset confuses selection, increases hesitation (clarifications), and encourages hallucinated combos. Prefer a small, general-purpose set.

**Q2.** A coordinator's *synthesis* subagent keeps running its own searches. Best fix?
- A. Prompt it harder to "only synthesise"
- B. Remove the search tool from that agent — its presence reads as permission
- C. Set `tool_choice` to `ANY`
- D. Add more search tools so it picks better

**Answer: B.** If a tool exists, the model assumes it should be used — presence = implicit permission. Scope each agent's toolset to its role.

**Q3.** You're building a strict extraction pipeline and must force Claude to call exactly one named tool. Which `tool_choice`?
- A. `AUTO`
- B. `NONE`
- C. `ANY`
- D. `TOOL` — e.g. `{"type":"tool","name":...}`

**Answer: D.** `TOOL` is fully deterministic — a forced path to one named tool, ideal for strict pipelines. `ANY` only guarantees *some* tool; `AUTO` is flexible but unpredictable; `NONE` bans tools.

**Q4.** You connect three MCP servers. How does the model see their tools?
- A. Flattened into one list — no grouping by server, no awareness of origin
- B. Grouped under each server's name
- C. Only the first server's tools are visible
- D. As resources, not tools

**Answer: A.** All tools flatten into one decision surface. Risk = tool explosion without structure; mitigate by limiting servers or filtering tools before exposure.

**Q5.** A user @-mentions a document and its contents appear in context before Claude reasons. What is this?
- A. A Tool — Claude called it reactively
- B. A Bash command
- C. A Resource — data fetched proactively (e.g. on @-mention)
- D. A templated tool schema

**Answer: C.** Resources = data (READ), supplied proactively such as when @-mentioned. Tools = actions (DO), run reactively when Claude decides to call them.

**Q6.** Your `input_schema` requires `total: integer` and the model returns a valid integer — but it doesn't equal the line-item sum. What does this show?
- A. The schema is malformed
- B. Schema-valid ≠ semantically correct — schema defines structure, not business rules; add a validation layer (Pydantic/Instructor)
- C. You forgot an `enum`
- D. `tool_choice` should be `NONE`

**Answer: B.** JSON Schema defines shape/type only. Cross-field rules need a real validation layer; validate both input and output, and feed specific failures back on retry.

---

## Module 8 — Prompt Engineering & Context Management

Writing prompts that produce reliable, repeatable behaviour — and keeping a large or long-running context from quietly degrading the model's performance.

### 67. Prompting with Explicit Criteria

Vague prompts produce unpredictable results. A reliable prompt states its **scope**, its **constraints**, its **success condition** — and crucially, what the model must **not** change.

The four things to pin down:

| Element | What it answers | Example |
|---|---|---|
| **Scope** | What is in / out of bounds | "Only the functions in `billing.py`." |
| **Constraints** | The rules the output must obey | "Keep the public API unchanged; no new dependencies." |
| **Success condition** | How we know it is done / correct | "Existing tests still pass and the bug no longer reproduces." |
| **Do-not-touch** | What the model must leave alone | "Do **not** reformat unrelated lines or rename variables." |

- An explicit, checkable criterion turns a matter of taste into a rule the model — and you — can verify. That is also what makes a result **repeatable** rather than a lucky one-off.
- A strong first line follows **action verb + clear task + output spec** (e.g. "Rewrite this function to fix the empty-list crash, returning only the changed function"). The first line sets the foundation for everything after it.

### 68. False Positives (Vague Prompting)

When a prompt is vague, the model **guesses the pattern instead of following a rule** — so it **over-flags**, burying the real defects in noise and wasting time. The fix is to **define exact criteria for detection**.

**Worked case — a code-review agent.** Ask an agent to "find the bugs" with no definition of a bug and it flags whatever it finds opinionated about — naming, style, a config value. State precisely what *is* and *isn't* reportable and the false positives vanish:

```text
Report ONLY correctness defects that can cause a crash, wrong result,
or data loss. For each, give the location, the issue, and a severity.
Do NOT report: style, naming, formatting, or configuration values
that may be intentional.
```

That single exclusionary paragraph is what stops `DEBUG = True` from being reported as a "bug" when it might be intentional config.

**The find → verify architecture.** One prompt can't be both *thorough* (recall) and *strict* (precision) — push one and the other suffers. Split the job into passes:

```
PASS 1  FIND    — scan against the criteria, report EVERY candidate (incl. unsure)
PASS 2  VERIFY  — for each candidate, adversarially ask "is this REALLY a bug?"
FINAL           — keep only what survived pass 2
```

- **Pass 1 — Find (recall):** report every candidate, even uncertain ones; don't self-censor, a later pass filters.
- **Pass 2 — Verify (precision):** a skeptical reviewer takes each finding alone and asks "could this be intentional?" — **default to rejecting** when uncertain.
- Because each finding is verified independently, you can use a **cheaper model for the wide net** (e.g. Haiku) and a **stronger one for verification** (e.g. Opus). At scale, run pass 1 per file in parallel, pool the candidates, then verify each.

### 69. Few-Shot Prompting

**Few-shot prompting** means showing the model a handful of worked examples instead of (or alongside) an instruction. Examples **anchor behaviour and output format**, **reduce hallucination**, and **improve consistency** — especially on ambiguous inputs.

Sarcasm is the textbook ambiguous case: *"Oh good, it broke again. Fantastic."* is positive word-by-word but negative in meaning. Piling on adjectives ("please be very careful") rarely fixes it — showing one worked example does. Three habits make few-shot work:

1. **Wrap each example in XML tags** so the model sees clean boundaries between examples and the real input.
2. **Include the hard / edge case** (good *and* bad) with a **one-line reason** — the reason teaches the *pattern*, not just the answer.
3. **Place examples after the instruction, before the real input.**

```xml
Classify the sentiment as 'positive' or 'negative'. Judge the writer's
true feeling, not the surface words — be especially careful with sarcasm.

<examples>
  <example>
    <review>Best purchase I've made all year.</review>
    <sentiment>positive</sentiment>
  </example>
  <example>
    <review>Wow, it died in an hour. Amazing. Worth every penny.</review>
    <reason>Praise words used sarcastically to express frustration.</reason>
    <sentiment>negative</sentiment>
  </example>
</examples>

Review: {the_real_input}
```

Zero-shot labels the sarcastic review **positive** (wrong); few-shot labels it **negative** (right). The instruction barely changed — the examples did the work. Use a descriptive tag name (`<review>`, not `<data>`) and combine examples with a short reason to reinforce the desired output.

### 70. Large Context Problems

The context window is finite, and **everything in it is re-read on every turn** — so it costs tokens repeatedly and a window stuffed with noise makes the model work harder to find what matters. Four named failure modes show up when context grows large or a session runs long.

| Problem | Symptom | Tactic |
|---|---|---|
| **Progressive summarization** | Detail degrades step-to-step; numbers lost first | Extract + re-inject key data |
| **Lost in the middle** | Model ignores the middle of long context | Chunk + process in sections |
| **Tool result bloat** | Verbose output → noise + token waste | Filter / distil before passing |
| **Conversation history growth** | Context grows uncontrollably | Summarise or trim history |

**Practical tactics — keep the bulk out, pass forward only the distilled result.** All the patterns are one idea at three scales:

| Pattern | What stays out of context | What comes back |
|---|---|---|
| **Distil tool output** | the raw blob (e.g. a 60-line log) | a fact card (~5 fields) |
| **Scratchpad file** | the processed material (each document) | one-line notes on disk |
| **Subagent delegation** | the sub-task's whole context | a short result |

```python
for doc in documents:
    finding = extract_one_line(doc)      # reduce a long doc to a headline
    append_to("scratchpad.md", finding)  # notes live on disk
    drop(doc)                            # raw doc leaves the working context
# finally: read the SMALL scratchpad and write the summary
summary = summarise(read("scratchpad.md"))
```

- **Distilling verbose tool output** at the point it returns can shrink a result by ~97% (e.g. raw ≈ 1848 tokens → fact card ≈ 37 tokens). Do it at *every* verbose tool return.
- **A scratchpad on disk** keeps context flat across hundreds of steps and **survives a crash or context reset** — the notes aren't in volatile conversation state.
- **Subagent delegation** hands a context-heavy sub-task (e.g. reading ten files) to a subagent that burns its *own* window and returns only a short answer — the main context never sees the ten files.
- The Claude API also offers **server-side helpers**: **context editing** (prunes stale tool results / thinking) and **compaction** (summarises old turns near the limit). These complement the application-level patterns you control directly.

### Module 8 — Checkpoint Quiz

**Q1.** An explicit prompt should specify scope, constraints, and a success condition. What is the fourth thing it should make explicit?
- A. The model to use
- B. The maximum token budget
- C. The temperature setting
- D. What the model must **not** change

**Answer: D.** Telling the model what to leave alone stops it from "helpfully" editing things you never asked about — as important as scope, constraints, and the success condition.

**Q2.** A code-review agent keeps flagging style preferences and config values as "bugs". What is the cause and the fix?
- A. The model is too small; switch to Opus for the whole job
- B. Vague prompting causes over-flagging; define explicit, exclusionary criteria for what is reportable
- C. The context is too long; trim the history
- D. It needs more examples of good style

**Answer: B.** A vague prompt makes the model guess patterns instead of following rules, producing false positives. Explicit criteria stating what is and isn't reportable cut them.

**Q3.** In few-shot prompting, why wrap each example in XML tags and place them after the instruction?
- A. XML is required by the API for examples
- B. It reduces the token cost of the prompt
- C. Tags give clean boundaries so the model tells the worked examples apart from the real input
- D. It lets the model skip the instruction entirely

**Answer: C.** Descriptive XML tags mark clear boundaries between the examples and the actual input; placing them after the instruction (and before the input) is the standard, most reliable structure.

**Q4.** A model summarising a long chain of steps keeps losing the precise figures from earlier steps. Which problem is this, and the tactic?
- A. Progressive summarization → extract and re-inject the key data
- B. Lost in the middle → chunk and process in sections
- C. Tool result bloat → distil before passing
- D. Conversation history growth → trim old turns

**Answer: A.** Information degrading across steps — numbers lost first — is progressive summarization. Fix it by extracting the exact data and re-injecting it verbatim rather than re-summarising prose.

**Q5.** A tool returns a 500-line log. What is the recommended way to handle it for a long-running agent?
- A. Paste the full log into the conversation so nothing is lost
- B. Switch to a larger model to absorb the extra tokens
- C. Distil it to a small fact card the moment it returns and discard the raw blob
- D. Ignore the log entirely

**Answer: C.** Tool result bloat: reduce the verbose output to a fact card of only the needed fields at the point it returns. Done at every verbose return, the run stays lean — the bulk stays out, the distilled result goes forward.

---

## Module 9 — Scenario Playbook

The canonical agent scenarios the exam tests. For each one: the goal, the right architecture (workflow vs agent vs multi-agent), the tools it uses, and the patterns and gotchas. This module synthesises everything — each concept cross-references the earlier idea that powers it.

### 71. Scenario · Customer Support Agent

**Goal:** answer customer tickets against live business systems — and know when to stop and hand off. A support agent uses tools (often via MCP) to resolve, or escalates via a confidence-based routing decision.

**The right architecture**

| Dimension | Choice |
|---|---|
| Shape | Single model-driven agent. Tickets vary too much for a fixed workflow — give Claude tools + the goal and let it choose (Module 1, decision-making). |
| Tools | Business systems exposed as MCP tools: `search_orders`, `lookup_customer`, `issue_refund`. Tools = actions; Resources = read-only data (Module 7). |
| Stop control | The `stop_reason` loop — `tool_use` ↔ `tool_result` until `end_turn` (Module 1). Never parse text to decide flow. |

**The escalation decision — the heart of this scenario**

For each ticket the agent returns a structured decision (`proposed_action`, `confidence`, `reason`, `draft_reply`), then routing runs in two layers, in order:

- **Layer 1 — Hard triggers (override confidence, run first):** `policy_gap` (needs an exception the agent can't authorise), `customer_request` (they asked for a human), `cannot_progress` (lacks tools/info). Any of these → escalate, however confident.
- **Layer 2 — Confidence bands (only if in-scope):** ≥ 0.85 → auto-resolve; ≥ 0.60 → human review (agent drafts, person approves); < 0.60 → escalate.

The middle human-review band is the human-in-the-loop sweet spot: automation speed with a safety net. Thresholds are a business decision (risk appetite), not a model setting.

**Handoff protocol:** every escalation carries a structured package — context, actions taken, the blocker/reason, and the draft reply — so the human starts informed instead of digging through history (Module 6).

> Why does a hard trigger override even 0.99 confidence? They are about authority and scope, not certainty. A confident answer to a question the agent has no business answering is the most dangerous output it can produce — so Layer 1 runs first and ignores confidence.

### 72. Scenario · Claude Code (Dev Agent)

**Goal:** generate or fix code in a real repo. Claude Code is the canonical coding agent — it plans, then executes, driving the agentic loop with built-in file and shell tools.

**The right architecture**

- **Shape:** a single model-driven agent. The task ("fix this bug") is under-specified at the file level, so it needs dynamic planning, not a fixed pipeline.
- **Tools — the built-in set** (mental model: file system + shell primitives): `Read` (load file into context), `Grep` (search inside files), `Glob` (find files by pattern), `Edit` (modify — needs permission), `Bash` (run commands), plus `Write`.
- **Loop:** Gather Context (Read/Grep/Glob) → Take Action (Edit/Bash) → Verify (run tests) → repeat until done.

**Plan vs execute**

| Phase | What happens |
|---|---|
| Plan | Plan Mode permission: Claude analyses and proposes but cannot modify files or run commands. Read-only reconnaissance. Pairs with `opusplan` (Opus plans, Sonnet executes — Module 1). |
| Execute | Approve the plan, leave Plan Mode, and the agent edits and runs. `acceptEdits` auto-accepts file edits for the session (Module 4). |

`CLAUDE.md` is project memory, auto-loaded every session and merged across the hierarchy (managed → user → project → local, closest wins). It steers how the dev agent works without re-typing it (Module 4).

**On the exam:** a code-generation / fix-the-bug agent uses the built-in tools (`Read · Grep · Glob · Edit · Bash · Write`), not MCP business tools. Plan Mode is the read-only "analyse but don't touch" mode.

### 73. Scenario · Multi-Agent System

**Goal:** tackle a big task (e.g. a research report) by splitting it across specialised subagents. The architecture is hub-and-spoke: one coordinator + many subagents that never talk directly. All routing goes through the coordinator; subagents have isolated context and never share state.

**The coordinator's job & the patterns it must apply**

| Pattern | What it means |
|---|---|
| Decompose → delegate → aggregate | Break the goal into subtasks, spawn subagents (single / sequential / parallel by complexity), then merge results and resolve conflicts into one answer. |
| Partitioning | Split into distinct, non-overlapping scopes — each subagent owns a unique slice. Same task to two agents = overlap + wasted tokens. |
| Structured findings | Subagents return structured objects (`id`, `type`, `confidence`, source), not raw prose. Raw text loses attribution; structure enables traceability and reliable synthesis. |
| Refinement loop | Not one-shot: the coordinator evaluates coverage against a quality bar, finds gaps, and re-delegates targeted work until the bar is met. |
| Context passing | Subagents receive only explicitly provided context (the Agent/Task tool) — no implicit memory. Prompts must carry all needed data (Module 3). |

Coordinate by goal, not by script. Goal-oriented coordination (define the end goal + quality bar, let the coordinator pick agents dynamically) beats procedural coordination (rigid step-by-step prompts that break on unexpected input) (Module 5).

**Gotchas:** (1) narrow decomposition — isolated subagents can't spot missing dimensions, so the coordinator must check for gaps before answering. (2) Observability — without a coordinator funnelling all messages, flows are untraceable; with one, errors are centralised. (3) Subagent failure — retry/fallback locally, escalate only when exhausted; never propagate a raw failure up.

### 74. Scenario · Dev Productivity Agent

**Goal:** automate the repetitive work around development — write the changelog, draft release notes, triage issues, update docs. It uses the built-in tools to automate work the engineer would otherwise do by hand.

**The right architecture**

- **Shape:** often the most workflow-shaped of the agent scenarios. If the steps are known and repeatable ("read the diff, append a changelog entry"), lean toward a workflow / prompt chain, not a free-roaming agent.
- **Tools:** the same built-in primitives — `Read`, `Grep`, `Glob`, `Edit`, `Bash` — pointed at chores rather than features.
- **Packaging:** wrap a repeatable chore as a custom skill (a `SKILL.md` with frontmatter — `context: fork`, `allowed-tools`) so it runs in isolation with a scoped toolset (Module 4).

| Dev agent (71/72) | Dev productivity agent (74) |
|---|---|
| Solves novel problems: fix this bug, build this feature. | Automates known chores: changelogs, release notes, issue triage. |
| Needs dynamic planning (plan vs execute). | Often a predetermined sequence → favour a workflow / skill. |

Both use the built-in toolset. The distinguisher is the task: novel engineering → agent; repeatable chore → workflow/skill. Don't over-engineer a chore into an autonomous agent.

### 75. Scenario · CI/CD Agent

**Goal:** run the agent headless in the pipeline to review code and run tests — and above all avoid false positives. Classic example: a scheduled GitHub Action that reads production logs and opens a PR with fixes.

**The right architecture**

- **Shape:** a workflow-leaning agent triggered by an event (a cron schedule, a push, a failing build). No human watching, so reliability matters more than flexibility.
- **Trigger & surface:** a scheduled GitHub Action → reads production logs → diagnoses → opens a PR with the fix for a human to merge. The PR is the human-in-the-loop gate.
- **Tools:** built-in `Read · Grep · Bash` to inspect logs and run tests; `Edit` to write the fix; the PR is created via `Bash` / `gh`.

**Avoiding false positives — the defining challenge**

| Cause | Fix |
|---|---|
| Vague prompting → the model guesses patterns instead of rules, and over-flags. | Prompt with explicit criteria: define exactly what counts as a real issue, the scope, the success condition — and what not to flag/change. |
| The model invents what "broken" means. | Give few-shot examples (good and bad) to anchor the detection rule. |
| Required steps (lint, tests) get skipped. | Programmatic gates / hooks (PreToolUse / PostToolUse) make prerequisites impossible to skip — enforced, not requested (Modules 4 & 6). |

**The exam's CI/CD trap:** a vague prompt ("find problems and fix them") produces a flood of false positives that waste reviewer time and destroy trust. The fix is explicit criteria + few-shot examples, plus hard gates so tests can't be skipped. Prompt-based guidance can drift; gates guarantee.

> Where does the human stay in the loop in a headless CI/CD agent? At the pull request. The agent never merges to production itself — it opens a PR with the proposed fix, and a human reviews and merges. That's the safety boundary for an unattended agent acting on production.

### 76. Capstone · Workflow vs Agent — the deciding question

**Every scenario reduces to one question: do you know the exact steps in advance?** If yes → build a workflow. If the details are unclear → build an agent.

| | WORKFLOW | AGENT |
|---|---|---|
| Use when | You have precise task understanding and know the exact step sequence. | Task details are unclear / vary per run. |
| Control | Predetermined — hardcoded flow. | Dynamic planning — model chooses. |
| Testing | Easier to test. | Harder to test. |
| Success rate | Higher. | Lower. |
| Trade-off | Reliable but rigid. | Flexible but less predictable. |

**Core principle:** solve problems reliably first, innovation second. Default to workflows; reach for an agent only when the task genuinely can't be predetermined. A workflow that works beats a clever agent that sometimes works.

**Tooling corollary:** give agents abstract / general-purpose tools, not hyper-specialised ones. Many niche tools confuse selection, cause hesitation, and invite hallucinated combinations — prefer small, general toolsets (Module 7).

**On the exam:** when an answer offers "build an autonomous agent" vs "build a workflow" for a task whose steps are already known, the workflow is correct — it's predetermined, easier to test, and higher success rate. Agents are for genuine uncertainty. Reliability beats novelty.

> One-line recall: when do you pick an agent over a workflow? Only when the task details are unclear and the steps can't be fixed in advance — you need dynamic planning. If you can write the steps down, write a workflow: easier to test, higher success rate.

### Module 9 — Checkpoint Quiz

6 questions mixing concepts from across the whole exam.

**Q1.** A support agent returns `reason: "customer_request"` with `confidence: 0.97`. What happens?
- A. Auto-resolve — confidence is well above the 0.85 bar
- B. Escalate — a hard trigger overrides confidence
- C. Human review — it lands in the middle band
- D. Retry the tool call until confidence changes

**Answer: B.** `customer_request` is a Layer-1 hard trigger (with `policy_gap` and `cannot_progress`). Layer 1 runs first and ignores confidence — the ticket escalates regardless of how sure the model is.

**Q2.** You're building a code-fixing agent. Which toolset is it using?
- A. MCP business tools like `search_orders` and `issue_refund`
- B. Only `WebSearch` and `WebFetch`
- C. Built-in tools: `Read · Grep · Glob · Edit · Bash · Write`
- D. The coordinator's `Agent`/`Task` tool only

**Answer: C.** A dev agent works in the codebase using the built-in tools — the file-system + shell primitives. MCP business tools belong to the support scenario.

**Q3.** In a multi-agent system, why do subagents return structured findings instead of raw text?
- A. Structured output uses fewer tokens than prose
- B. It lets subagents talk directly to each other
- C. The API rejects plain-text tool results
- D. Raw text loses attribution; structure keeps content + metadata together for traceable, reliable synthesis

**Answer: D.** Passing raw text loses provenance — downstream agents can't trace which source supports which claim. Structured objects (`id`, `type`, `confidence`, source) preserve attribution and enable reliable synthesis. (Subagents never talk directly — that's the coordinator's job.)

**Q4.** A CI/CD agent floods every run with false-positive issue reports. The best fix is to:
- A. Add explicit detection criteria and few-shot examples, and tell it what NOT to flag
- B. Give it more tools so it can investigate further
- C. Raise `max_tokens` so it can explain each flag
- D. Switch the loop to stop on text content rather than `stop_reason`

**Answer: A.** False positives come from vague prompting — the model guesses patterns instead of following rules. Define exact criteria, scope, and what not to change, and anchor with few-shot examples. More tools (B) makes selection worse; D is an anti-pattern.

**Q5.** The task's exact steps are known, repeatable, and easy to test. You should build:
- A. An autonomous agent — it's more flexible
- B. A hub-and-spoke multi-agent system for parallelism
- C. A workflow — predetermined, easier to test, higher success rate
- D. Whichever is newer; innovation comes first

**Answer: C.** When you know the exact step sequence, a workflow wins: predetermined, easier to test, higher success rate. The capstone principle is solve reliably first, innovation second — default to workflows; use agents only when details are unclear.

**Q6.** Your support agent calls the wrong tool on ambiguous requests, and adding three more tools made it worse. The root cause and fix are:
- A. The model is too small; force `tool_choice: none`
- B. Too many overlapping/niche tools and weak descriptions — use a small general toolset and sharpen each description with explicit boundaries
- C. The Messages API is stateful and cached the wrong tool
- D. Move tool results into the assistant role to fix selection

**Answer: B.** More tools ≠ more capability — niche, overlapping tools confuse selection. Prefer a small, general-purpose toolset, and make each description draw a clear boundary ("NOT for X — use Y"). The description is the interface the model selects on. (The API is stateless; tool results go in the user role.)

---

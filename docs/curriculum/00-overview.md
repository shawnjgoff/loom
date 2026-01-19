# Loom System Overview: Orchestrating Ralph Loops

## What is Loom?

Loom is infrastructure for running reliable, continuous AI agent loops. At its core, Loom orchestrates what's known as **Ralph loops** - a technique for autonomous software development where an AI agent works through tasks one loop at a time, building software through iteration and eventual consistency.

The name "Ralph" comes from Ralph Wiggum, embodying the philosophy that AI agents are **"deterministically bad in a non-deterministic world"**. They fail in predictable ways, which means you can tune their behavior through prompt refinement - putting up "signs by the slide" to guide better decisions.

## The Ralph Loop Technique

In its purest form, Ralph is a bash loop:

```
while :; do cat PROMPT.md | claude-code ; done
```

Each loop follows a cycle:
1. **Load the stack** - Specifications, plans, and context (allocated fresh each loop)
2. **Decide** - LLM chooses the single most important thing to do
3. **Execute** - Run that one thing using tools (read, write, bash, search)
4. **Validate** - Tests and builds provide backpressure (reject bad code)
5. **Capture** - Document learnings for future loops
6. **Loop** - Return to step 1 with a clean context window

The magic is in the constraint: **one thing per loop**. This keeps context windows small, focuses effort, and makes failures obvious. Ralph might be "chasing squirrels" sometimes, but he's predictably bad - you can see the patterns and add instructions to fix them.

## Why Loom Exists

Ralph loops work, but running them reliably at scale requires infrastructure:

**The Problem:** A raw bash loop is fragile. Network failures, API rate limits, context window management, and lack of history make autonomous loops unreliable.

**The Solution:** Loom provides the orchestration layer that makes Ralph loops production-ready:
- Explicit state machines ensure loops progress deterministically
- Thread persistence captures complete history across all loops
- Retry logic handles transient failures gracefully
- Subagent spawning extends context without pollution
- Tool security prevents workspace boundary violations
- Server-side secrets eliminate credential management
- Automatic checkpointing enables resume from any point

Think of Loom as the difference between running a bash loop manually and having Kubernetes orchestrate your containers. The loop still does the work, but the infrastructure makes it reliable.

## The Architecture of Ralph Loops

Loom consists of three main pieces that enable reliable autonomous loops:

### The Agent: Ralph's Brain

The agent is a state machine that orchestrates each loop iteration. It moves through explicit states:

**WaitingForUserInput** → **CallingLlm** → **ProcessingLlmResponse** → **ExecutingTools** → **PostToolsHook** → back to **CallingLlm**

This creates the fundamental cycle: User → LLM → Tools → LLM → User. For autonomous work, Ralph enters this cycle once and keeps looping (LLM → Tools → LLM) until the task is complete.

The state machine ensures Ralph never gets lost. Every transition is logged, retries are bounded, and errors are recoverable. You always know where Ralph is in the cycle.

### The Thread System: Ralph's Memory

Threads capture everything that happens across all loops - every message, tool execution, decision, and failure. This enables:

**Resume from anywhere** - Power outage? Server restart? Pick up exactly where Ralph left off.

**Cross-device continuity** - Start Ralph on your laptop, continue on the web, check progress on your phone.

**Offline-first** - Ralph keeps working even if the server is unreachable. Sync happens opportunistically.

**Audit trail** - See exactly what Ralph did and why. Every loop is preserved.

### The Server: Ralph's Support Infrastructure

The server handles everything Ralph shouldn't worry about:

**Secrets** - API keys stay server-side. Ralph just authenticates as himself.

**LLM Proxy** - Unified interface to Claude, GPT-4, and others. Switch providers without changing Ralph.

**Weavers** - Provision ephemeral Kubernetes pods when Ralph needs a clean environment.

**Multi-tenancy** - Organizations, teams, permissions, audit logs.

This separation means Ralph focuses on building software while infrastructure handles operations.

## Core Principles of Ralph Loops

### One Thing Per Loop

The cardinal rule: Ralph does one thing per iteration. Not two things. Not "a feature." One specific task.

This keeps context windows small (critical for quality) and makes progress visible. If Ralph starts doing multiple things, narrow the instruction until he's back to one.

### Stack Allocation Every Loop

Don't reuse context window allocations across loops. Instead, load specs and plans fresh each time:

```
Your task is to implement missing stdlib (see @specs/stdlib/*).
Follow @fix_plan.md and choose the most important thing.
```

This seems wasteful - you're "burning" the allocation of reading specs every loop. But it's essential. Context windows have quality cliffs around 150k tokens. The less you use, the better Ralph performs.

### Extend Context with Subagents

When Ralph needs to allocate context for expensive work (like summarizing test output or searching the entire codebase), spawn a subagent. The subagent does the work and returns a concise result.

Your main loop becomes a scheduler:

```
Before making changes, search codebase using parallel subagents (don't assume
not implemented). You may use up to 100 parallel subagents for search, but only
1 subagent for build/tests.
```

This extends your effective context window without degrading the main loop's quality.

### Backpressure Rejects Bad Code

Code generation is cheap. **Validation is hard.** Wire in every form of backpressure you can:
- Type checkers (mypy, dialyzer)
- Tests (unit, integration, property-based)
- Linters (clippy, eslint)
- Static analyzers
- Security scanners

After Ralph generates code, the wheel must turn fast. If backpressure fails, Ralph loops again with the error. The faster the wheel turns, the faster Ralph learns.

### Signs by the Slide

Ralph will fall off the slide. When he does, you add a sign: "SLIDE DOWN, DON'T JUMP, LOOK AROUND."

In practice, this means adding instructions to the prompt:

```
9999999999999999999999999999. DO NOT IMPLEMENT PLACEHOLDER OR SIMPLE
IMPLEMENTATIONS. WE WANT FULL IMPLEMENTATIONS. DO IT OR I WILL YELL AT YOU
```

The models are trained to chase their reward function (compiling code). Sometimes you need to yell. Ralph doesn't take offense - he just reads the sign.

### Loop Back on Itself

Create opportunities for Ralph to evaluate his own work. After generating code, have Ralph compile it and examine the LLVM IR. After running tests, have Ralph read the output and decide what to fix.

This self-reflection is where Ralph learns. Each loop informs the next.

### Trust and Eventual Consistency

Building with Ralph requires faith. He will make mistakes. He will implement the wrong thing. He will ignore your signs.

But Ralph is **deterministically bad**. The mistakes follow patterns. You tune the prompts, add more signs, refine the specs. Over hundreds of loops, eventual consistency emerges.

Ralph has built entire programming languages without those languages being in the training data. It works - if you trust the process.

## Component Glossary

**Thread**: A complete record of all loops - messages, tool executions, decisions, errors. Persisted locally and synced to server.

**Tool**: A function Ralph can call - read_file, edit_file, bash, web_search. Each tool has a JSON schema the LLM uses to generate correct calls.

**Subagent**: A separate agent instance spawned for expensive operations. Returns a concise result to the main loop without polluting its context window.

**Weaver**: An ephemeral Kubernetes pod where Ralph can work in an isolated environment. Automatically cleaned up after configured lifetime.

**State Machine**: The explicit flow of states Ralph moves through each loop. Makes behavior predictable and debuggable.

**Backpressure**: Validation that rejects bad code - tests, type checkers, linters. Forces Ralph to loop again until code passes.

**Hook**: Post-execution actions like auto-commit. After tools that mutate files, hooks capture changes in git commits.

**LLM Proxy**: Server-side abstraction over Claude, GPT-4, etc. Keeps API keys server-side and provides unified streaming interface.

## Two Modes: Interactive and Autonomous

Loom supports both conversational (human-in-loop) and autonomous (Ralph loops) modes:

### Interactive Mode: Pair Programming

You and the AI work together. You ask questions, the AI suggests solutions, calls tools, and waits for your input. This is traditional AI coding assistant behavior.

The state machine cycles: WaitingForUserInput → CallingLlm → Tools → CallingLlm → WaitingForUserInput

Each cycle returns control to you.

### Autonomous Mode: Ralph Takes the Wheel

You give Ralph specifications and a plan, then let him loop continuously. Ralph decides what's most important, implements it, validates, and loops until the task list is empty.

The state machine stays in the loop: CallingLlm → Tools → CallingLlm → Tools → ...

You watch the stream, monitoring for patterns of bad behavior. When you see patterns, you stop Ralph, add signs to the prompt, and restart.

**Key insight:** Ralph is monolithic. Don't try multi-agent systems yet. One Ralph, one repository, one task per loop. Keep it simple.

## A Day in the Life: Interactive Mode

Let's walk through a typical interactive session to see how Loom's components work together:

### Starting a Session

You open your terminal in a project directory and type `loom`.

1. **Authentication Check**: The CLI checks for stored credentials. If absent, it initiates device code flow.

2. **Device Code Flow**: CLI requests a code from the server. You visit the URL, log in (GitHub/Google/magic link), enter the code. CLI receives an access token and stores it in your system keychain.

3. **Thread Creation**: A new thread with UUID7 identifier captures your workspace root and starts recording.

### The First Loop Iteration

You type: "Add logging to the user authentication function"

1. **User Message**: CLI creates a message and transitions agent to CallingLlm state.

2. **LLM Request**: Agent sends conversation history + tool definitions to server at `/proxy/anthropic/stream`.

3. **Server Proxy**: Server uses its stored API key to call Claude's API.

4. **Streaming Response**: Claude's response streams back token by token via Server-Sent Events. You see text appearing in real-time.

5. **Tool Calls**: Claude decides to read the authentication file first. Agent transitions to ExecutingTools.

6. **Tool Execution**: read_file validates the path is within workspace root, reads the file (up to 1MB), returns contents.

7. **Back to LLM**: Agent transitions to CallingLlm with tool results. Claude now sees the code and suggests changes.

8. **Edit Tool**: Claude calls edit_file with before/after snippets. Tool finds exact match, performs replacement, writes atomically.

9. **Auto-Commit Hook**: After mutation tools complete, PostToolsHook state triggers. If auto-commit is enabled, Loom generates a commit message and commits changes.

10. **Thread Save**: Agent returns to WaitingForUserInput. Thread saves locally. Background task syncs to server.

This is one complete loop: User → LLM → Tools → LLM → User.

### Cross-Device Resume

Later, you open the web UI on a different machine. You log in, see your thread list, and click to continue.

The server sends the complete thread JSON. The web UI reconstructs the conversation and establishes a WebSocket for live updates. You send a message and it streams back exactly like in the CLI.

The thread is the source of truth. Clients are just views into it.

## A Day in the Life: Autonomous Mode

Now let's see Ralph working autonomously to build something substantial:

### Setup Phase: Specifications

Before starting autonomous loops, you have a conversation with the LLM about what to build. Instead of asking it to implement immediately, you discuss requirements, architecture, and constraints.

Once the LLM understands the task, you issue:

```
Write detailed specifications for this project. Create one spec file per major
component in specs/ directory. Include purpose, types, behavior, and examples.
```

The LLM generates specification documents. These become Ralph's stack allocation each loop.

### The Plan

You instruct Ralph to create a todo list:

```
Study specs/* to learn the project. Use up to 500 subagents to search existing
code in src/ and compare against specs. Create @fix_plan.md - a bullet point
list sorted by priority of what hasn't been implemented yet. Consider TODOs,
placeholders, and minimal implementations.
```

Ralph spawns hundreds of subagents in parallel, searches the codebase, compares against specs, and creates a prioritized plan. This happens in one loop iteration.

### Autonomous Loops Begin

Now you start the continuous loop with:

```
Your task is to implement missing functionality (see @specs/*). Follow
@fix_plan.md and choose the most important thing. Implement that ONE thing,
run tests, and loop. Use parallel subagents for search, but only 1 subagent
for build/tests.

After implementing, run tests for that unit. If tests fail, resolve them.

DO NOT implement placeholders or minimal implementations. Full implementations only.

When you learn something about running tests or building, update @AGENT.md.
```

Ralph enters the loop:

**Loop 1**: Ralph reads specs and plan, chooses "implement parser for function declarations," searches the codebase with subagents to verify not already implemented, implements the parser, runs tests. Tests pass. Ralph updates fix_plan.md marking this complete.

**Loop 2**: Ralph reads updated plan, chooses "add type checking for function parameters," implements it, runs tests. Tests fail with type mismatch. Ralph examines error, fixes the implementation, runs tests again. Tests pass.

**Loop 3**: Ralph reads plan, implements next item, tests pass.

This continues. You watch the stream, looking for patterns:

- Is Ralph implementing placeholders? Add a sign.
- Is Ralph ignoring test failures? Strengthen the instruction.
- Is Ralph assuming code doesn't exist without searching? Add "don't assume not implemented."

### Backpressure in Action

At Loop 47, Ralph generates code that compiles but has a type error the type checker catches. The build fails. Ralph sees the error and loops again with the failure in context.

Loop 48: Ralph fixes the type error, builds successfully, moves on.

The wheel turns fast. Rust's type system provides immediate, deterministic backpressure. Ralph learns quickly.

### Self-Improvement

At Loop 103, Ralph discovers that running tests requires a specific environment variable. Ralph updates @AGENT.md with this finding. Future loops read this file and know the correct command.

Ralph has taken himself to university.

### Plan Refresh

At Loop 200, Ralph has completed everything in fix_plan.md. You notice the plan is getting stale. You stop the loop and issue:

```
The plan is complete. Generate a new @fix_plan.md by searching the entire
codebase with subagents for TODOs, placeholders, minimal implementations, and
missing features from specs.
```

Ralph generates a fresh plan. You restart the autonomous loop. Ralph continues.

### Weaver Integration

At Loop 350, you want Ralph to work in a clean environment. You provision a weaver:

```
loom weaver create --image rust:latest --repo https://github.com/you/project.git
```

A Kubernetes pod spins up with the repo cloned. You attach to it:

```
loom attach <weaver-id>
```

Ralph now runs inside the weaver. All file operations happen in the isolated container. When done, the weaver is automatically deleted. No cleanup needed.

## How Loom Enables This

Every part of Loom's architecture exists to make Ralph loops reliable:

**State Machine** - Ralph never gets lost. Every transition is explicit. Errors are recoverable. Retries are bounded.

**Thread Persistence** - Every loop is captured. Resume from anywhere. History never lost, even across crashes.

**Tool Security** - Path validation prevents Ralph from escaping workspace boundaries. Even if Ralph hallucinates paths, the tools enforce constraints.

**Subagent Support** - Ralph can spawn hundreds of subagents without polluting his main context window. This extends effective context dramatically.

**Streaming** - See Ralph's reasoning in real-time. Catch mistakes early. Stop the loop when you see bad patterns.

**Auto-Commit Hooks** - Every batch of mutations gets committed automatically with LLM-generated messages. Never lose work.

**Server Proxy** - Switch LLM providers server-side. Ralph doesn't need API keys. Try Claude for reasoning, GPT-4 for code generation.

**Retry Logic** - Transient failures (network, rate limits, timeouts) are retried automatically with exponential backoff. Ralph keeps looping.

**Offline-First** - Ralph works even when the server is down. Threads save locally. Sync happens opportunistically.

## Key Design Patterns

**Monolithic Over Microservices** - Multi-agent systems add non-determinism. Ralph is monolithic: one agent, one repo, one task per loop.

**Explicit Over Implicit** - State transitions are logged. Context is carried explicitly in state variants. No hidden mutable fields.

**Deterministic Failure** - Ralph fails predictably. This lets you tune prompts and fix behavior patterns systematically.

**Fast Feedback Loops** - The faster the wheel turns (generate → validate → loop), the faster Ralph learns. Optimize for tight cycles.

**Eventual Consistency** - Trust the process. Over hundreds of loops, correct behavior emerges through tuning and backpressure.

**Defense in Depth** - Security is layered: tool path validation, ABAC policies, audit logs, read-only filesystems in weavers.

## What This Enables

With Loom's infrastructure, Ralph can:

- Build entire programming languages not in training data
- Migrate codebases between frameworks autonomously
- Generate comprehensive test suites with property-based tests
- Refactor large codebases systematically
- Implement specifications with full fidelity
- Self-tune through documented learnings

The key is: **you provide specifications and backpressure. Ralph provides iteration and eventual consistency.**

## What's Next?

Now that you understand the philosophy of Ralph loops and how Loom orchestrates them, the following documents dive deep into each component:

1. **Agent State Machine**: Learn the explicit states and transitions that make loops predictable
2. **Thread System**: Understand how complete history is captured and synced
3. **Tool System**: See how tools are secured and executed within workspace boundaries
4. **LLM Integration**: Explore provider abstraction, streaming, and server-side proxy architecture
5. **Server & Authentication**: Dive into multi-tenant security, organizations, and ABAC policies
6. **Weaver System**: Master ephemeral execution environments on Kubernetes
7. **Advanced Topics**: Analytics, feature flags, SCM, TUI, and extended systems

Each document assumes you understand the Ralph loop philosophy. They explain the **mechanics** of how Loom implements this **vision**.

Start from the top and work your way down. By the end, you'll understand both why Loom exists and how every component enables reliable autonomous development.

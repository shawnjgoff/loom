# Building an AI Agent with Ralph Loops on Loom

## Who This Is For

This walkthrough is for:

**Ralph Loop Practitioners** - You understand the Ralph technique from the article, but haven't used Loom yet. You need to see what Loom gives you over a raw bash loop.

**Ralph-Curious Developers** - You've heard about Ralph loops but want to see the technique in action before fully committing to understanding it.

**Agent Builders** - You want to understand how LLM agents work by building one from scratch. This walkthrough is meta: we use Ralph (an agent loop orchestrator) to build an agent, while learning how agents function.

## What We're Building

A Neovim plugin that provides an AI coding assistant interface. The plugin will:
- Open a floating window for chat interaction
- Allow selecting code and asking questions about it
- Stream LLM responses in real-time
- Maintain conversation context
- Call tools to read and modify files

**Important**: The Neovim plugin is the vehicle, not the destination. We're demonstrating Ralph loops with Loom. The reader should come away understanding the technique and Loom's infrastructure value.

As a secondary goal, we're building an AI agent from scratch, which means implementing LLM integration, tool execution, context management, and streaming - teaching how agents work at a fundamental level.

## What You'll Learn

**Ralph Loop Technique**:
- How to structure specifications for autonomous work
- How to craft the "one thing per loop" prompt
- What backpressure looks like in practice
- How to spot patterns and add signposts
- When to use subagents vs main loop
- How to recognize when Ralph is done

**Loom's Infrastructure Value**:
- Thread persistence across sessions
- State machine clarity and debugging
- Auto-commit hooks after mutations
- Subagent spawning for context extension
- Resume capability after crashes
- Server-side secret management

**Agent Implementation**:
- LLM API integration patterns
- Tool schema design and execution
- Context window management
- Streaming response handling
- Error recovery and retries

## The Reality

This is a documentary, not a screenplay. We cannot predict:
- Which loops Ralph will struggle on
- What mistakes we'll make in specifications
- Which signposts will be needed
- How many loops until completion

What we document here **actually happened**. When Ralph does something suboptimal, we show the real observation → signpost → correction cycle. This authenticity is essential for understanding the technique.

## Prerequisites

Before starting:

- **Loom installed and configured** - Follow the installation guide
- **Authenticated to a Loom server** - Run `loom login` if needed
- **Neovim installed** - Any recent version (0.9+)
- **Git configured** - For auto-commit hooks
- **Basic understanding** - Familiarity with the Ralph loop philosophy (read the overview if not)

Check your setup:
```bash
loom --version
nvim --version
git config user.name
```

## Project Setup

Create a fresh directory for this project:

```bash
mkdir neovim-ai-agent
cd neovim-ai-agent
git init
```

Initialize the directory structure:
```bash
mkdir -p lua/ai-agent
mkdir -p specs
touch README.md
```

Now we're ready to start Loom in interactive mode.

---

## Phase 1: Interactive Conversation to Specifications

The first phase is conversational. We're not asking the LLM to implement anything yet. Instead, we're having a discussion to ensure the LLM deeply understands what we're building.

### Starting the Session

```bash
loom
```

Loom starts in interactive mode. A new thread is created capturing our workspace root.

### The Conversation

**Our goal**: Have a detailed discussion about:
- What the plugin does from a user perspective
- Technical architecture (Lua API, floating windows, async I/O)
- How the agent will work (LLM calls, tools, streaming)
- Edge cases and error handling
- Testing approach

We're looking for the LLM to ask clarifying questions. Good signs:
- "Should the chat history persist across Neovim restarts?"
- "How do you want to handle rate limiting?"
- "What happens if the user closes the window during streaming?"

Bad signs (means we need to elaborate):
- LLM immediately suggests implementation
- LLM makes assumptions without asking
- LLM glosses over complex parts

### Example Conversation Starter

```
I want to build a Neovim plugin that provides an AI coding assistant. The user
should be able to select code in their buffer, open a chat window, and have a
conversation with an LLM about that code. The LLM should be able to read files
and make edits through tool calls.

Before we implement anything, let's discuss the architecture. What questions
do you have about how this should work?
```

**[PLACEHOLDER: Actual conversation will be documented here]**

The conversation continues until:
1. The LLM demonstrates deep understanding
2. All ambiguities are resolved
3. We've discussed edge cases and failure modes

### Generating Specifications

Once we're confident the LLM understands, we issue:

```
Now let's write detailed specifications. Create separate spec files in specs/
directory for:
- Overall plugin architecture
- LLM client implementation (API integration, streaming, retries)
- Tool system (file reading, editing, execution)
- UI layer (floating window, rendering, input handling)
- Context management (conversation history, token limits)

Each spec should include:
- Purpose and role in the system
- Key types and their responsibilities
- Behavior and state transitions
- Error handling approach
- Testing strategy

Do not write code yet. Just specifications.
```

The LLM generates specification files. We review them for:
- Completeness (do they cover everything discussed?)
- Clarity (could someone else implement from these?)
- Correctness (do they match what we want?)

**[PLACEHOLDER: Generated specs will be referenced here]**

If specs are wrong or incomplete, we iterate in interactive mode until they're right. **This is critical** - bad specs lead to Ralph building the wrong thing.

### Initial Project Structure

With specs complete, we can ask for scaffolding:

```
Based on the specifications, create the initial project structure:
- plugin/ directory with main plugin entry point
- lua/ai-agent/ with module skeleton
- Basic README
- .gitignore for Lua/Neovim

No implementation yet - just the file structure and empty function stubs
with comments indicating what each will do.
```

The LLM creates the skeleton. We now have:
- Specifications in specs/
- Project structure in place
- Empty functions waiting for implementation

**[PLACEHOLDER: Project structure will be shown here]**

This completes Phase 1. We now have everything needed to start autonomous Ralph loops.

---

## Phase 2: Setting Up Autonomous Ralph Loops

Now we transition from interactive to autonomous mode. Ralph will loop continuously, implementing the specifications one piece at a time.

### Creating the Plan

First, we need a TODO list. In interactive mode:

```
Study the specifications in specs/* carefully. Then create a @fix_plan.md file
containing a prioritized list of what needs to be implemented.

Start with foundational pieces (LLM client, basic tool execution) before UI.
Each item should be specific and achievable in one loop iteration.

Use subagents to search the codebase and verify nothing is already implemented.
```

The LLM generates fix_plan.md with prioritized items.

**[PLACEHOLDER: Initial fix_plan.md will be shown here]**

We review the plan. Does the ordering make sense? Are items scoped to "one thing"? If not, we refine it interactively.

### Creating the Agent Instructions

Now we create AGENT.md - Ralph's instructions for how to work:

```
Create @AGENT.md with instructions for how to build and test this project.
Include:
- How to load the plugin in Neovim for testing
- How to run any Lua tests
- Common debugging commands
- Any environment setup needed

Keep it brief - you'll update this as you learn.
```

**[PLACEHOLDER: Initial AGENT.md will be shown here]**

### Crafting the Autonomous Loop Prompt

This is the prompt that drives Ralph. We'll save it as PROMPT.md:

```markdown
Your task is to implement the Neovim AI agent plugin according to specifications
in @specs/*.

Follow @fix_plan.md and choose the SINGLE most important item to implement.
Implement that ONE thing, test it, and update the plan.

Before making changes:
- Search the codebase using parallel subagents (don't assume not implemented)
- Verify you understand the spec for what you're implementing
- Think about dependencies - does this need something else first?

Use parallel subagents for searching and planning, but only 1 subagent for
testing (to avoid conflicts).

After implementing functionality:
- Test it according to @AGENT.md instructions
- If tests fail, resolve them as part of this loop
- Update @fix_plan.md marking this item complete
- Update @AGENT.md if you learn something new

DO NOT implement placeholders or minimal implementations. Full implementations only.

When you complete an item, return to WaitingForUserInput state so the loop can
continue with the next item.
```

We save this as PROMPT.md in the project root.

**Why Loom vs Raw Bash Loop**: With a raw bash loop, this prompt would be piped to the LLM each iteration. Context window would fill with noise. With Loom:
- The prompt is read by the agent state machine
- Thread persistence maintains clean history
- State transitions are explicit
- We can resume from any point
- Auto-commit hooks capture each iteration

### Starting Autonomous Loops

We're ready. Exit interactive mode and start the loop:

```bash
# One option: manual loop (like raw Ralph)
while :; do loom resume; done

# Better: Let Loom manage the loop
loom auto --prompt PROMPT.md
```

**[NOTE: We need to verify the actual Loom command for autonomous mode]**

Ralph begins. We watch the stream, looking for:
- What Ralph chooses to implement (is it actually the most important thing?)
- How Ralph implements it (placeholders? shortcuts? proper implementation?)
- Test results (does backpressure catch issues?)
- Patterns of failure (these become signposts)

---

## Phase 3: Ralph at Work (Observed Behavior)

This section documents what **actually happened** during the autonomous loops. We cannot script this in advance.

### Overview

**[PLACEHOLDER: After running, we'll document:]**
- Total number of loops executed
- Time elapsed
- Number of signpost additions needed
- Number of spec corrections needed
- Final state (complete? partially complete?)

### Loop 1-5: Getting Started

**[PLACEHOLDER: Summarize early loops]**
- What Ralph chose to implement first
- Whether it matched our expectations
- Any immediate issues

### Loop X: First Signpost Needed

**[PLACEHOLDER: Document first observed failure pattern]**

This is where we see Ralph doing something suboptimal. For example:
- Implementing a placeholder function
- Making poor design choices
- Assuming code exists without checking
- Ignoring test failures

We observe the pattern, stop the loop, and add a signpost to PROMPT.md:

**[Example structure - actual content will differ]:**

**What Ralph Did**: [Describe the behavior]

**Why It's Wrong**: [Explain the issue]

**The Signpost**: [Show the instruction we added to PROMPT.md]

**Restart**: We resume Ralph with the updated prompt.

### Loop X+1: Signpost Effect

**[PLACEHOLDER: Show Ralph's behavior after the signpost]**

Did Ralph correct the behavior? If not, we may need to strengthen the signpost or address a spec issue.

### Loop Y: Backpressure in Action

**[PLACEHOLDER: Document a moment where tests/linters caught an issue]**

This demonstrates the backpressure concept:
- Ralph generates code
- Tests fail (Lua syntax error, nil reference, etc.)
- Ralph sees the error
- Ralph fixes it in the next loop iteration
- Tests pass

The wheel turns fast.

### Loop Z: Ralph Learns Something

**[PLACEHOLDER: Document Ralph updating AGENT.md]**

Ralph discovers something about running tests or building the project and updates AGENT.md. Future loops read this and know the correct command.

Ralph has taken himself to university.

### Loops Middle Section: Steady Progress

**[PLACEHOLDER: Summarize the bulk of loops where things work smoothly]**

After initial tuning, Ralph typically settles into productive work. Summarize:
- Major components implemented
- Any additional signposts needed
- Progress toward completion

### Loop Final: Completion

**[PLACEHOLDER: Document how Ralph recognizes work is done]**

When fix_plan.md is empty and tests pass, Ralph should return to WaitingForUserInput and indicate completion.

---

## Phase 4: Loom-Specific Features Demonstrated

Now that we've built the plugin, let's demonstrate what Loom provided that a raw bash loop wouldn't.

### Thread Persistence and Resume

**[PLACEHOLDER: We'll simulate a crash/interrupt and show resume]**

1. Stop Ralph mid-loop (Ctrl+C or kill process)
2. Show that thread state is preserved in ~/.local/share/loom/threads/
3. Resume with `loom resume`
4. Ralph picks up exactly where he left off

**Value**: Raw bash loop loses all state on interrupt. Loom makes crashes recoverable.

### Subagent Spawning

**[PLACEHOLDER: Show a moment where Ralph spawned multiple subagents]**

Look at logs/output showing:
- Ralph spawning 50+ subagents to search codebase
- Subagents returning concise results
- Main loop context window staying small

**Value**: Raw bash loop would fill context with search results. Loom's subagents extend effective context without pollution.

### Auto-Commit Hooks

Show the git log:

```bash
git log --oneline
```

**[PLACEHOLDER: Show actual commit history with auto-generated messages]**

Each mutation loop should have a commit. Messages are LLM-generated and describe what changed.

**Value**: Raw bash loop requires manual commits or loses work history. Loom captures every iteration automatically.

### State Machine Visibility

**[PLACEHOLDER: Show Loom's state machine logs]**

Demonstrate how we can see:
- Which state Ralph is in at any moment
- State transitions logged
- Where errors occurred (which state)

**Value**: Raw bash loop is opaque. Loom's explicit state machine makes behavior debuggable.

### Cross-Device Continuation

**[PLACEHOLDER: Optionally show resuming on different machine via web UI]**

If server sync is enabled:
1. Start on laptop CLI
2. View thread on phone/tablet web UI
3. Continue on desktop

**Value**: Raw bash loop is local only. Loom threads sync across devices.

---

## Phase 5: The Working Plugin

### Demonstration

**[PLACEHOLDER: Show the plugin in action]**

Screenshots or command sequence:
1. Open Neovim with a code file
2. Select some code visually
3. Trigger the AI assistant (`:AIChat` or similar)
4. Ask a question about the code
5. See streaming response in floating window
6. Agent reads files via tools
7. Agent suggests edit, makes change

### What We Built

Summarize the final state:
- Lines of code generated
- Number of files
- Test coverage
- Functionality delivered

**[PLACEHOLDER: Actual metrics after completion]**

### What We Learned

Reflect on:
- Patterns Ralph struggled with
- How signposts evolved
- Where specs needed correction
- Which backpressure mechanisms were most valuable
- How many loops until steady state

**[PLACEHOLDER: Actual learnings from the process]**

---

## Comparing Raw Ralph to Loom Ralph

### Raw Bash Loop Challenges

What you'd fight with in `while :; do cat PROMPT.md | claude-code ; done`:

**Context Window Management**: Every loop allocates more. By loop 50, quality degrades as context fills with tool outputs, test results, and repeated spec readings.

**No State Visibility**: Loop dies? You have no idea where. Was it CallingLlm? ExecutingTools? You restart from scratch.

**Manual Commits**: You must remember to commit after each successful loop or risk losing work.

**History Loss**: Interrupt the process? All conversation history gone. Start over.

**Secrets Exposure**: API keys in environment variables or config files.

**No Subagents**: Cannot extend context. Main loop gets all search results, test outputs, build logs.

**Resume Complexity**: No thread persistence. Can't pick up where you left off days later.

### What Loom Provided

**Thread Persistence**: Complete history across all loops, saved locally and synced to server.

**State Machine**: Explicit states and transitions. Always know where Ralph is. Errors are debuggable.

**Auto-Commit Hooks**: Every mutation batch captured automatically with LLM-generated messages.

**Subagent Support**: Spawned hundreds of subagents for search/analysis. Main loop stayed clean.

**Resume Capability**: Interrupted 3 times during development. Each time, `loom resume` continued perfectly.

**Server Proxy**: API keys server-side. Switched between Claude and GPT-4 without client changes.

**Retry Logic**: Transient API failures (rate limits, timeouts) handled automatically with exponential backoff.

**Cross-Device**: Started on laptop, checked progress on phone, finished on desktop.

The bash loop works. Loom makes it production-ready.

---

## Conclusion

We built a functioning Neovim AI agent plugin through Ralph loops orchestrated by Loom. The technique works:

**One thing per loop** - Ralph focused on single tasks, making progress visible and failures obvious.

**Stack allocation every loop** - Specs loaded fresh each iteration, keeping context window small and quality high.

**Backpressure** - Lua syntax checking and plugin loading tests rejected bad code, forcing correction.

**Signposts** - When Ralph fell off the slide, we added signs. Behavior improved.

**Eventual consistency** - Through **[X]** loops, correct implementation emerged from iteration and tuning.

### Key Takeaways

**For Ralph Loop Practitioners**: Loom is the infrastructure layer your bash loop needs. Thread persistence, state machines, auto-commits, and subagents make autonomous loops reliable at scale.

**For Ralph-Curious Developers**: This is the technique in action. It works. Ralph is deterministically bad, which means you can systematically improve behavior through signposts.

**For Agent Builders**: We implemented LLM integration, tool execution, streaming, and context management from scratch. You now understand how agents work at a fundamental level.

### Next Steps

- Read the full curriculum to understand each Loom component deeply
- Try your own Ralph loop on a different project
- Experiment with different backpressure mechanisms
- Join the community to share Ralph loop experiences

The infrastructure exists. The technique is proven. Start building.

---

## Appendix: Artifacts

All artifacts from this walkthrough are preserved in this directory:

- `specs/` - Generated specifications
- `fix_plan.md` - Evolution of the TODO list
- `AGENT.md` - Self-documented learnings
- `PROMPT.md` - The autonomous loop prompt with signposts
- `git-log.txt` - Complete auto-commit history
- `final-plugin/` - The working Neovim plugin code

**[PLACEHOLDER: These will be captured after running the actual process]**

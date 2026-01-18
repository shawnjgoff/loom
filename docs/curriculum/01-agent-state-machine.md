# The Agent State Machine

## Component Role

The agent state machine is the orchestrator at the heart of Loom. It manages the flow of conversations between you, the LLM, and tools. Think of it as a conductor directing when to listen for input, when to call the AI, when to execute tools, and how to recover from errors. Every interaction flows through this state machine.

## Referenced Components

**Thread System**: The agent reads from and writes to threads, which store conversation history.
**Tool Registry**: The agent queries this for available tools and invokes them.
**LLM Client**: The agent sends requests through this abstraction to talk to AI providers.
**Auto-Commit Service**: Triggered by the agent's PostToolsHook state when tools modify files.

## How It Works

### The State Pattern

Instead of using flags and conditionals to track what the agent is doing, Loom uses an explicit state machine. Each state is a Rust enum variant that carries exactly the data needed for that state. This makes the code impossible to get into invalid states—the type system enforces correctness.

There are seven states:

**WaitingForUserInput** - The agent is idle and ready to receive your message. This is both the starting state and the state it returns to after completing work.

**CallingLlm** - A request has been sent to the language model and the agent is waiting for a response. This state tracks how many retry attempts have been made.

**ProcessingLlmResponse** - The LLM has finished responding and the agent is examining what came back. If the response contains tool calls, it transitions to ExecutingTools. If it's just text, it goes back to WaitingForUserInput.

**ExecutingTools** - One or more tools are running. This state maintains a list of tool execution statuses, tracking which tools are pending, running, or completed. The agent stays in this state until all tools finish.

**PostToolsHook** - All tools have completed, and the agent is running post-execution hooks like auto-commit. This is a separate state to keep infrastructure concerns out of the main conversation flow. After hooks complete, the agent sends tool results back to the LLM.

**Error** - Something went wrong, but it might be recoverable. This state tracks the error, how many retries have been attempted, and where the error originated (LLM, tool, or IO). The agent can retry with exponential backoff.

**ShuttingDown** - The agent is gracefully terminating. This is a terminal state—once reached, the agent should be dropped.

### Event-Driven Architecture

The state machine is synchronous and pure. It doesn't do any IO itself. Instead, it receives events and returns actions that tell the caller what to do. This inversion of control means the same state machine can be used in different environments: CLI, server, tests.

Events flow into the agent:

**UserInput** - You typed a message.
**LlmEvent** - Something happened with the LLM: a text fragment arrived, a tool call came through, the response completed, or an error occurred.
**ToolCompleted** - A tool finished executing with a success or error result.
**PostToolsHookCompleted** - The post-tool hooks finished running.
**RetryTimeoutFired** - The backoff timer expired and it's time to retry a failed operation.
**ShutdownRequested** - Time to gracefully shut down.

Actions flow out of the agent:

**SendLlmRequest** - Make this request to the LLM provider.
**ExecuteTools** - Run these tool calls in parallel.
**RunPostToolsHook** - Execute post-tool infrastructure operations.
**WaitForInput** - Idle, waiting for the next event.
**DisplayMessage** - Show this text to the user.
**DisplayError** - Show this error to the user.
**Shutdown** - Terminate the agent.

### Conversation Context Threading

Each state that needs access to the conversation history carries its own ConversationContext. This is a snapshot of all messages exchanged so far. When transitioning between states, this context is cloned and updated:

When UserInput arrives, the user's message is appended to the context.
When the LLM completes, the assistant's message is appended.
When tools finish, their result messages are appended.

This pattern ensures the conversation history is always consistent with the current state. There's no mutable global state that could get out of sync.

### Tool Execution Flow

When the LLM requests tool calls, the flow gets interesting:

1. The agent transitions from ProcessingLlmResponse to ExecutingTools with a list of ToolExecutionStatus entries, all initially set to Pending.

2. The agent returns an ExecuteTools action containing all the tool calls.

3. The caller (usually the CLI or server) executes the tools. Tools run asynchronously and in parallel when possible.

4. As each tool completes, a ToolCompleted event is sent back to the agent with the call ID and result.

5. The agent updates that specific ToolExecutionStatus from Pending or Running to Completed.

6. Once all tools are Completed, the agent checks if any tools made file modifications. If so, it transitions to PostToolsHook. If not, it skips straight to CallingLlm with tool results.

This design allows the agent to track progress of multiple concurrent tools while remaining single-threaded and deterministic.

### The PostToolsHook State

This state exists to decouple infrastructure operations from the main conversation flow. After tools execute, you might want to:

Run auto-commit to save file changes.
Update metrics about what tools were used.
Trigger webhooks for external integrations.

These operations shouldn't block the agent or fail the user's request if they go wrong. The PostToolsHook state makes this explicit: hooks are fire-and-forget from the agent's perspective. If they succeed or fail, the agent doesn't change behavior—it just moves on to sending tool results back to the LLM.

### Error Recovery

When an error occurs, the agent transitions to the Error state along with metadata about what went wrong:

**Origin** - Was this an LLM error, a tool error, or an IO error? Different origins have different retry strategies.
**Retries** - How many times have we already retried?
**Error details** - The actual error information to potentially show the user.

For LLM errors like rate limiting or transient network failures, the agent can retry with exponential backoff. A RetryTimeoutFired event triggers the retry, transitioning back to CallingLlm.

For tool errors, retries usually don't help—if a file wasn't found, retrying immediately won't fix it. The agent returns to WaitingForUserInput and displays the error so you can address the issue.

If retries are exhausted, the agent gives up and returns to WaitingForUserInput with an error displayed. The conversation can continue—Loom doesn't crash on errors.

### Streaming Integration

When the LLM streams a response, multiple LlmEvent instances arrive:

**TextDelta** events come in with fragments of text: "Hello", " world", "! How", " can", " I", " help"...

The agent stays in CallingLlm state during streaming, forwarding TextDelta events to the user via DisplayMessage actions. This provides real-time feedback.

**ToolCallDelta** events stream partial tool arguments. The agent doesn't act on these until the complete tool call arrives.

When a **Completed** event arrives with the full LlmResponse, the agent transitions to ProcessingLlmResponse.

If an **Error** event arrives, the agent transitions to the Error state and applies retry logic.

### State Logging

Every state transition is logged with structured metadata. You'll see entries like:

```
state_transition: WaitingForUserInput -> CallingLlm
state_transition: CallingLlm -> ProcessingLlmResponse
state_transition: ProcessingLlmResponse -> ExecutingTools
```

This makes debugging straightforward. If something goes wrong, you can trace the exact sequence of state transitions that led to the issue.

## Why This Design?

**Predictability** - Every possible state and transition is explicit. There are no hidden states or race conditions.

**Testability** - The state machine is pure and synchronous. Unit tests can drive it through any sequence of states by providing events and checking the returned actions. Property-based tests verify invariants like "shutdown always succeeds from any state" or "retry count never exceeds max retries".

**Debuggability** - State transitions are logged, so you can see exactly what the agent did. The current state is always visible.

**Exhaustive Handling** - Rust's match expressions ensure all state-event combinations are handled. Adding a new state or event causes compile errors until you handle every case.

**Backpressure** - The caller controls the pace of events. The agent never has an unbounded queue or background threads that could overflow.

## Extending the State Machine

Adding new states is straightforward but requires updating several places:

1. Add the new variant to the AgentState enum with any required data fields.
2. Update the state_name method to return a string name for logging.
3. Update the conversation accessor methods to extract conversation context from your state.
4. Add transition handlers in handle_event for all relevant event types.
5. Update tests to cover the new state.

The compiler will guide you—any place you haven't updated will fail to compile.

## Implementation Location

The state machine lives in loom-core, the foundation crate:

**crates/loom-core/src/state.rs** - Defines AgentState, AgentEvent, ToolExecutionStatus enums
**crates/loom-core/src/agent.rs** - Implements the Agent struct with handle_event method

The agent has no dependencies on LLM providers, tools, or HTTP. It only depends on the trait abstractions. This makes it extremely testable and reusable.

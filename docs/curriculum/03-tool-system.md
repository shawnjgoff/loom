# The Tool System

## Component Role

Tools are the hands of the AI agent—they let the language model interact with the real world. Without tools, the AI can only talk. With tools, it can read your files, search directories, make edits, run shell commands, and search the web. The tool system provides the framework for defining, registering, securing, and executing these operations.

## Referenced Components

**Agent State Machine**: The agent transitions to ExecutingTools state when the LLM requests tool calls.
**LLM Client**: Tool definitions are sent to the LLM so it knows what functions are available.
**Thread System**: Tool executions and results are recorded in the conversation history.
**Auto-Commit Service**: Triggered after file-modifying tools execute to save changes to git.

## How It Works

### The Tool Trait

Every tool in Loom implements a common trait that defines its interface:

**name** - Returns a unique identifier like "read_file" or "edit_file". This is how the LLM references the tool.

**description** - Returns a human-readable explanation of what the tool does. The LLM uses this to decide when to call the tool.

**input_schema** - Returns a JSON Schema object describing the tool's parameters. This tells the LLM what arguments to provide and validates them before execution.

**to_definition** - Converts the tool to a ToolDefinition that gets sent to the LLM provider.

**invoke** - The async method that actually executes the tool. It receives validated arguments as JSON and a ToolContext with workspace information.

This trait abstraction means the rest of Loom doesn't care what tools do—it just knows how to register, describe, and execute them.

### Tool Registry

The ToolRegistry is a simple HashMap from tool names to boxed trait objects. When Loom starts, it creates a registry and registers all built-in tools:

```
registry.register(Box::new(ReadFileTool::new()));
registry.register(Box::new(ListFilesTool::new()));
registry.register(Box::new(EditFileTool::new()));
registry.register(Box::new(BashTool::new()));
registry.register(Box::new(WebSearchTool::new()));
```

The registry provides:
**get** - Look up a tool by name for execution
**definitions** - Return a list of ToolDefinitions to send to the LLM

When the agent makes an LLM request, it calls registry.definitions to include all available tools. The LLM can then choose which tools to call based on your question.

### Tool Context

When a tool executes, it receives a ToolContext containing:
- **workspace_root** - The directory where the conversation started

This provides the security boundary. All file operations must happen within the workspace. A tool can resolve relative paths by joining them with workspace_root, and then validate the final path is still inside the workspace.

### Security: Path Validation

Every file-related tool validates paths to prevent traversal attacks. The validation process:

1. If the path is relative, join it with workspace_root to make it absolute.
2. Canonicalize the path to resolve symlinks and ".." components.
3. Canonicalize workspace_root as well.
4. Check if the canonical path starts with the canonical workspace_root.
5. If not, return an error: PathOutsideWorkspace.

This prevents attacks like reading "../../../../etc/passwd" even if the AI tries to. Canonicalization handles sneaky cases:
- Symlinks pointing outside the workspace
- Multiple ".." components that escape the boundary
- Unicode tricks with similar-looking characters

The validation happens before any file is opened, so malicious requests are rejected early.

### Built-In Tools

**read_file** - Reads the contents of a file. Takes a path and optional max_bytes limit (default 1MB). Returns the file contents as a string, plus a truncated boolean if the file was too large. Uses String::from_utf8_lossy to handle binary files gracefully.

**list_files** - Lists directory contents recursively. Takes a root directory (default: workspace_root) and max_results limit (default: 1000). Returns an array of entries with path and is_dir flag. This lets the AI explore the codebase structure.

**edit_file** - Makes changes to files using snippet-based replacement. Takes a path and array of edits. Each edit has old_str (text to find), new_str (replacement text), and optional replace_all flag. Returns the number of edits applied and byte counts.

The snippet approach is more robust than line-number edits because line numbers can shift when the file changes between read and edit. The AI provides enough context around the change that the exact text can be found unambiguously.

Special cases:
- Empty old_str creates a new file or appends to an existing one
- Empty new_str deletes the matched text
- replace_all: true replaces all occurrences instead of just the first

**bash** - Executes shell commands in the workspace directory. Takes a command string, optional cwd (working directory relative to workspace), and timeout_secs (default: 60, max: 300). Returns exit code, stdout, stderr, and flags for timeout and truncation.

Commands run with the same permissions as the Loom process. The cwd is validated to be within the workspace. Output is truncated to 256 kilobytes per stream to prevent memory exhaustion.

**web_search** - Searches the web using Google Custom Search Engine. Takes a query string and max_results limit (default: 5, max: 10). Returns an array of search results with title, URL, and snippet. The actual search happens server-side via a proxy endpoint to keep the API key secret.

**oracle** - Asks a secondary LLM (OpenAI) for reasoning or advice. Takes a query, optional model (default: gpt-4o), max_tokens, temperature, and system_prompt. Returns the assistant's message, tool calls (if any), usage statistics, and finish reason.

This tool enables the primary LLM (Claude) to consult OpenAI for specialized tasks like complex reasoning or code review. All requests go through the server's /proxy/openai/complete endpoint.

### Tool Execution States

When the agent receives tool calls from the LLM, it creates a ToolExecutionStatus for each tool:

**Pending** - The tool call has been received but not started. Contains call_id, tool_name, and requested_at timestamp.

**Running** - The tool is currently executing. Contains started_at, last_update_at timestamps and optional progress information (fraction complete, message, units processed).

**Completed** - The tool finished. Contains started_at, completed_at, and outcome (Success with output JSON, or Error with ToolError).

The agent transitions each tool individually through these states as it receives ToolCompleted events. This tracking enables progress display and parallel execution monitoring.

### Parallel Tool Execution

When multiple tools are called in the same LLM response, they execute in parallel when possible. The agent:

1. Receives the LlmResponse with tool_calls array.
2. Creates a Pending ToolExecutionStatus for each call.
3. Returns ExecuteTools action with all tool calls.
4. The executor (CLI or server) spawns async tasks for each tool.
5. As each tool completes, a ToolCompleted event is sent to the agent.
6. The agent updates that specific tool's status to Completed.
7. Once all tools reach Completed, the agent moves to PostToolsHook or back to CallingLlm.

This parallelism significantly speeds up operations when multiple independent tools are called, like reading several files simultaneously.

### Tool Error Handling

Tools return ToolError variants for different failure modes:

**NotFound** - The requested resource doesn't exist
**InvalidArguments** - The JSON arguments don't match the schema
**Io** - File system or network IO error
**Timeout** - The operation exceeded its timeout
**Internal** - Unexpected error in tool implementation
**PathOutsideWorkspace** - Security boundary violation
**FileNotFound** - Specific case of NotFound for file operations
**Serialization** - Failed to serialize/deserialize JSON

When a tool fails, the error is included in the ToolCompleted event. The agent packages it into a tool result message that gets sent back to the LLM. The AI can then respond to the error, perhaps by trying a different path or asking for clarification.

### Tool Progress Reporting

Long-running tools can report progress via ToolProgress updates:

**fraction** - A float from 0.0 to 1.0 indicating completion percentage
**message** - A human-readable status like "Downloading file..."
**units_processed** - Quantitative progress like number of files or bytes

These updates don't change the agent's state—they just provide feedback to the user. The agent forwards them as DisplayMessage actions.

### Auto-Commit Integration

After tools execute, the agent checks if any tools made file modifications. It maintains a CompletedToolInfo list with flags for is_mutating. File-modifying tools (edit_file, bash when it creates/modifies files) set this flag.

If any tool was mutating, the agent transitions to PostToolsHook state and returns RunPostToolsHook action. The executor triggers the auto-commit service, which:

1. Runs git status to see what changed.
2. Runs git diff to see the actual changes.
3. Asks the LLM to generate a commit message based on the diff.
4. Stages the changes with git add.
5. Creates a commit with the generated message.

This happens outside the main agent flow, so commit failures don't break the conversation. After hooks complete (success or failure), a PostToolsHookCompleted event is sent and the agent moves on.

### JSON Schema Input Validation

Each tool's input_schema is a JSON Schema object that describes its parameters. For example, read_file's schema:

```
{
  "type": "object",
  "properties": {
    "path": {
      "type": "string",
      "description": "Path to the file (absolute or relative to workspace)"
    },
    "max_bytes": {
      "type": "integer",
      "description": "Maximum bytes to read (default: 1MB)"
    }
  },
  "required": ["path"]
}
```

This schema:
- Tells the LLM what parameters are available and what they mean
- Validates the LLM's arguments before execution
- Documents the tool's interface for both AI and humans

If the LLM provides arguments that don't match the schema (like passing a string for max_bytes), the tool's invoke method receives invalid JSON and returns InvalidArguments error immediately.

### Snippet-Based Editing Philosophy

Loom uses snippet-based editing (old_str → new_str) instead of line-number-based editing because:

**LLMs are bad at line numbers** - Language models don't think in terms of line 42. They think in terms of "the function that handles user login".

**Line numbers shift** - If the file changes between when the AI reads it and when it edits, line numbers become wrong. Snippets with enough context remain unambiguous.

**Context serves as verification** - The old_str acts as a sanity check that the edit is being applied to the right place. If the text isn't found, the edit fails safely.

**Atomic edits** - Either the exact match is found and replaced, or nothing changes. No partial edits.

**Unicode safe** - String operations respect UTF-8 boundaries automatically.

The trade-off is that old_str must be unique when replace_all is false. But this is actually a feature—it prevents ambiguous edits that might change the wrong occurrence.

## Why This Design?

**Trait-Based Extensibility** - New tools are just implementations of the Tool trait. No core changes needed.

**Security by Default** - Path validation is centralized and can't be bypassed. Tools can't escape the workspace.

**LLM Compatibility** - JSON Schema is the standard for LLM function calling. The abstraction works with Claude, GPT, and future models.

**Parallel Execution** - Independent tool calls run concurrently for performance.

**Graceful Failure** - Tool errors are reported to the LLM, not thrown as exceptions. The conversation continues.

**Separation of Concerns** - Auto-commit and other infrastructure operations are decoupled via the PostToolsHook state.

## Adding New Tools

To add a custom tool:

1. Create a struct that implements the Tool trait in crates/loom-tools/src/.
2. Define the input JSON schema with clear descriptions.
3. Implement invoke with proper path validation if file-related.
4. Return success with output JSON or error with ToolError.
5. Register the tool in the CLI's tool registry setup code.

The tool automatically becomes available to the LLM and users can start using it immediately.

## Implementation Location

The tool system lives across these crates:

**crates/loom-core/src/tool.rs** - Tool trait, ToolDefinition, ToolContext, ToolError
**crates/loom-tools/src/** - All built-in tool implementations
**crates/loom-tools/src/registry.rs** - ToolRegistry implementation
**crates/loom-auto-commit/** - Auto-commit service integration

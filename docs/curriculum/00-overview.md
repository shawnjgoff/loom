# Loom System Overview & User Journey

## What is Loom?

Loom is an AI-powered coding assistant built in Rust that provides a conversational interface for interacting with large language models. Think of it as a bridge between you and AI models like Claude or GPT-4, with built-in tools that let the AI read files, execute commands, and make changes to your codebase.

## The Big Picture

Loom consists of three main pieces that work together:

**The Agent** - This is the brain that manages conversations with AI models. It keeps track of what's been said, when to call tools, and how to handle errors. The agent follows a state machine pattern, moving through well-defined states like "waiting for user input", "calling the LLM", and "executing tools".

**The Server** - This is a central HTTP service that acts as a proxy between clients and LLM providers. Instead of every client needing API keys, the server securely stores these keys and provides a unified interface. The server also handles authentication, stores conversation history, and can provision remote execution environments called weavers.

**The Clients** - These are the interfaces you interact with: a command-line interface, a web application, or a VS Code extension. All clients speak the same protocol and can resume conversations started on other devices.

## Component Ecosystem

Here's how all the major pieces fit together:

**Core Components** (the foundation):
- Agent State Machine: Orchestrates the conversation flow
- Thread System: Stores and syncs conversation history
- Tool System: Enables AI to interact with filesystems and execute commands
- LLM Integration: Connects to Claude, GPT-4, and other AI models
- Streaming System: Shows AI responses in real-time as they're generated

**Server Components**:
- HTTP API: RESTful endpoints for all operations
- Authentication: OAuth, magic links, and device code flow for CLI
- Organizations: Multi-tenant support with teams and permissions
- Audit System: Tracks all security-relevant events

**Remote Execution** (Weavers):
- Kubernetes Pod Provisioning: Creates isolated containers on demand
- WireGuard Tunneling: Secure SSH access to remote environments
- Secrets Management: SPIFFE-style identity and credential injection
- eBPF Auditing: System call monitoring for security

**Additional Systems**:
- SCM System: Git hosting with branch protection and webhooks
- Analytics: PostHog-style product analytics with identity resolution
- Feature Flags: A/B testing and gradual rollouts
- Web Frontend: Svelte 5 application with real-time updates
- Editor Integration: VS Code extension using Agent Client Protocol

## Glossary

**Thread**: A conversation session with the AI, including all messages, tool executions, and metadata. Threads are persisted locally and optionally synced to the server.

**Tool**: A function the AI can call to interact with the outside world, like reading a file or running a command. Each tool has a JSON schema that describes its inputs.

**Weaver**: An ephemeral Kubernetes pod that runs a Loom agent session in an isolated environment. Weavers are automatically cleaned up after a configurable lifetime.

**LLM Client**: An abstraction that lets Loom talk to different AI providers (Claude, GPT-4) through a unified interface.

**State Machine**: The agent's control flow logic, which explicitly defines all possible states and valid transitions between them.

**SSE (Server-Sent Events)**: A web standard for streaming data from server to client, used to show AI responses in real-time.

**ABAC (Attribute-Based Access Control)**: Fine-grained permission system based on attributes of users, resources, and actions.

**Proxy Architecture**: The server acts as a middleman between clients and LLM providers, keeping API keys server-side for security.

## A Day in the Life: User Journey

Let's walk through a typical session to see how everything works together:

### Starting a New Session (CLI)

You open your terminal in a project directory and type "loom". Here's what happens:

1. **CLI Initialization**: The loom command-line tool starts up and checks if you're authenticated. If not, it initiates the device code flow.

2. **Device Code Flow**: The CLI makes a request to the server asking for a device code. The server responds with a short code like "123-456-789" and displays "Visit https://loom.example.com/device and enter this code".

3. **Browser Authentication**: You open the URL in your browser, where you can log in using GitHub, Google, or a magic link sent to your email. After logging in, you enter the device code to authorize the CLI.

4. **Token Storage**: The CLI receives an access token and stores it in your system keychain (or config file as fallback). Future sessions will automatically use this token.

5. **Thread Creation**: A new thread is created with a unique UUID7 identifier (like "T-019b2b97-..."). The thread captures your current directory as the workspace root.

### Asking a Question

You type: "How do I add logging to this application?"

1. **Message Processing**: The CLI creates a user message and sends it to the agent's state machine.

2. **State Transition**: The agent transitions from "WaitingForUserInput" to "CallingLlm". It constructs an LLM request containing the conversation history and a list of available tools.

3. **Server Proxy**: The agent (via ProxyLlmClient) sends an HTTP request to the server at "/proxy/anthropic/stream" (or "/proxy/openai/stream" depending on configuration).

4. **LLM Request**: The server uses its stored API key to make a request to Claude's API. The request includes your question and definitions for all available tools.

5. **Streaming Response**: Claude starts generating a response, token by token. Each piece arrives as a Server-Sent Event.

6. **Real-Time Display**: Your terminal shows Claude's response appearing progressively: "You can use the tracing crate for structured logging. Let me show you an example..."

7. **Thread Update**: When the response completes, the agent transitions back to "WaitingForUserInput" and saves the updated thread locally. If sync is configured, a background task sends the thread to the server.

### Tool Execution

Claude decides it needs to see your current code and issues a tool call: "read_file" with the path "src/main.rs".

1. **Tool Call Detection**: The agent receives a response with "tool_calls" populated. It transitions to "ProcessingLlmResponse", then immediately to "ExecutingTools".

2. **Parallel Execution**: The agent can execute multiple tool calls concurrently. Each tool execution goes through states: Pending, Running, Completed.

3. **Path Validation**: The read_file tool validates that "src/main.rs" is within your workspace root (security boundary). It canonicalizes the path to prevent traversal attacks like "../../etc/passwd".

4. **File Reading**: The tool reads up to 1 megabyte of the file using async IO and returns the contents as JSON.

5. **Progress Display**: Your terminal shows "[read_file] Reading src/main.rs..." and then "[read_file] ✓ Read 1,234 bytes".

6. **Result Accumulation**: All tool results are collected and formatted as tool result messages to send back to Claude.

7. **Auto-Commit Hook**: After all tools complete, the agent enters the "PostToolsHook" state. If auto-commit is enabled and the tool made changes, Loom generates a commit message using the LLM and commits the changes.

8. **Next LLM Turn**: The agent transitions back to "CallingLlm" with the tool results in the conversation history. Claude can now respond based on what it learned from the file.

### Making Changes

Claude suggests adding a logging statement and calls the "edit_file" tool with specific before/after snippets.

1. **Edit Execution**: The edit_file tool performs snippet-based replacement. It finds the exact "old_str" in the file and replaces it with "new_str".

2. **Atomic Write**: The edit is written to a temporary file first, then atomically renamed to the target. This prevents corruption if the process is interrupted.

3. **Verification**: The tool returns the number of bytes changed, which Claude can verify matches expectations.

4. **Confirmation**: Claude responds: "I've added a tracing statement to your main function. The change has been applied."

### Resuming Later

You close the terminal and come back the next day. You type "loom resume" to continue where you left off.

1. **Thread Lookup**: The CLI queries the local thread store for the most recent thread sorted by last_activity_at.

2. **State Restoration**: The entire conversation history, agent state, and workspace context are loaded from the saved JSON file.

3. **Seamless Continuation**: You can immediately continue the conversation as if you never left. The agent remembers everything that happened.

### Cross-Device Sync

Later, you open the web UI at https://loom.example.com on a different machine.

1. **Web Authentication**: You log in with the same GitHub account you used in the CLI.

2. **Thread Listing**: The server queries the database for all threads belonging to your user and organization. The web UI displays them sorted by recent activity.

3. **Thread Loading**: You click on your thread. The server sends the complete thread JSON, including all messages and tool executions.

4. **Live Collaboration**: The web UI establishes a WebSocket connection to receive real-time updates. When you send a message, it streams back just like in the CLI.

### Remote Execution with Weavers

You need to test code in a clean environment, so you create a weaver.

1. **Weaver Request**: You run "loom weaver create --image python:3.12". The CLI sends a POST request to "/api/weaver".

2. **Pod Provisioning**: The server uses the Kubernetes client to create a pod in the "loom-weavers" namespace with your specified image. Labels identify it as managed by Loom and owned by your user ID.

3. **Environment Injection**: The server automatically injects environment variables including LOOM_SERVER_URL and LOOM_WEAVER_ID.

4. **Waiting for Ready**: The server polls the pod status until it reaches "Running" state (or times out after 60 seconds).

5. **Weaver Response**: The CLI receives the weaver ID and can now attach to it: "loom attach <weaver-id>".

6. **SSH Access (Optional)**: If WireGuard is enabled, the weaver registers with the tunnel server. You can SSH into the pod through the secure tunnel without exposing ports publicly.

7. **Automatic Cleanup**: After 4 hours (or your configured lifetime), a background task on the server finds expired weavers and deletes them from Kubernetes.

## Key Design Patterns

Throughout this journey, several patterns make Loom robust:

**Server-Side Secrets**: API keys never leave the server. Clients just choose which provider to use.

**Offline-First Persistence**: Threads always save locally first. Server sync happens in the background and retries on failure.

**Explicit State Machines**: The agent's state transitions are explicit and logged, making behavior predictable and debuggable.

**Defense in Depth**: Security is layered: path validation in tools, ABAC policies on the server, audit logging everywhere.

**Streaming Everything**: LLM responses, logs, and events all stream in real-time for immediate feedback.

**Ephemeral Compute**: Weavers are short-lived and isolated. No persistent state means lower security risk.

## What's Next?

Now that you understand how all the pieces fit together, the following documents will dive deep into each component:

1. **Core Agent State Machine**: Learn how the agent orchestrates conversations
2. **Thread System**: Understand persistence, sync, and conflict resolution
3. **Tool System**: See how tools are defined, secured, and executed
4. **LLM Integration**: Explore provider abstraction and streaming
5. **Server & Authentication**: Dive into multi-user security and organizations
6. **Weaver System**: Master remote execution environments
7. **Advanced Topics**: Analytics, feature flags, web UI, and more

Each document assumes you've read the previous ones and builds on those concepts. Start from the top and work your way down!

# The Thread System

## Component Role

The thread system handles persistence and synchronization of conversation sessions. A thread is a complete record of everything that happened in a conversation: your messages, the AI's responses, tool executions, and metadata like when it was created and which model was used. Think of threads as save files that let you pause, resume, search, and share conversations across devices.

## Referenced Components

**Agent State Machine**: Threads capture and restore the agent's state between sessions.
**LLM Client**: Threads record which provider and model were used.
**Tool Registry**: Threads store the results of tool executions.
**Server API**: Threads sync to the server for backup and cross-device access.
**Authentication System**: Thread visibility and ownership are enforced by auth policies.

## How It Works

### Thread Identity

Every thread has a unique identifier generated using UUID7. UUID7 is special because it's time-sortable—threads created later have lexicographically larger IDs. This means you can sort threads by ID and get chronological order without needing a separate timestamp index.

The format is: `T-019b2b97-fddf-7602-a3e4-1c4a295110c0`

The T- prefix distinguishes threads from other resources like users or organizations. The rest is a standard UUID7 with the creation timestamp embedded.

### Thread Data Model

A thread is a JSON document containing several sections:

**Identity Fields** - id, version, created_at, updated_at, last_activity_at. The version is incremented on every save and used for optimistic concurrency control when syncing.

**Environment Fields** - workspace_root (the directory where the conversation started), cwd (current working directory), loom_version (the Loom version that created the thread). These help restore context when resuming.

**LLM Fields** - provider (anthropic or openai), model (like claude-sonnet-4 or gpt-4o). This lets the same model be used when resuming.

**Visibility Fields** - visibility (private, team, organization, public), is_private (if true, never syncs to server), is_shared_with_support (if support team can access it).

**Conversation Snapshot** - messages array with all user, assistant, and tool messages. Each message has role, content, and optional tool_call_id/tool_name for tool results.

**Agent State Snapshot** - kind (which state the agent is in), retries (current retry count), last_error (if any), pending_tool_calls (tool calls that haven't finished yet).

**Metadata** - title (auto-generated or user-set), tags (user-defined labels), is_pinned (for favorites), extra (arbitrary JSON for extensions).

This structure captures everything needed to reconstruct an exact conversation state.

### Local Storage

Threads are always saved locally first, following the XDG Base Directory Specification. On Linux, that's ~/.local/share/loom/threads/. On macOS, it's ~/Library/Application Support/loom/threads/. Each thread is a separate JSON file named by its ID: `T-019b2b97-....json`.

The LocalThreadStore provides four operations:

**load** - Read a thread JSON file and deserialize it. Returns None if the file doesn't exist.

**save** - Serialize a thread to JSON and write it atomically. The write goes to a temp file first (thread_id.json.tmp), then is renamed to the final name. This atomicity prevents corruption if the process crashes mid-write.

**list** - Read all thread JSON files in the directory, parse just enough to extract summary info (ID, title, last_activity_at), sort by last activity descending, and return the requested number of threads.

**delete** - Remove a thread JSON file from disk.

These operations are all async using tokio::fs to avoid blocking the runtime.

### Sync Triggers

Threads sync to the server at two specific points:

**After each inferencing turn** - When the agent returns to WaitingForUserInput after processing your input, the LLM's response, and any tool executions, that's a logical checkpoint. The thread version is incremented and saved locally, then synced to the server in the background.

**On graceful shutdown** - When you exit Loom (Ctrl+C, EOF, or `/exit` command), the thread is saved one final time with the latest state.

These sync points ensure you don't lose work if you close your laptop or the process crashes.

### Background Sync

The SyncingThreadStore wraps LocalThreadStore and adds server synchronization. When you call save, it:

1. Saves the thread locally immediately (always succeeds or fails synchronously).
2. Checks if the thread has is_private set to true. If so, stops here—private threads never leave your machine.
3. Spawns a background tokio task to sync to the server.
4. Returns immediately without waiting for the server sync to complete.

This fire-and-forget approach means network issues never block the UI. You can keep working even if the server is down.

### Sync Retry Queue

When background sync fails (network down, server maintenance, etc.), the operation goes into a pending queue. This queue is persisted to disk at `~/.local/state/loom/sync/pending.json`.

Each pending entry contains:
- thread_id
- operation (Upsert or Delete)
- failed_at timestamp
- retry_count
- last_error message

Next time you start Loom or the server comes back, the SyncingThreadStore's retry_pending method iterates through the queue and attempts each operation again. Successful operations are removed from the queue. Failed operations stay and increment their retry count.

This ensures no thread changes are lost even if you work offline for days.

### Server API

The server exposes a RESTful API for thread operations:

**PUT /threads/{id}** - Upsert a thread. Requires an If-Match header with the client's current version. If the server's version doesn't match, it returns 409 Conflict with the server's version. This prevents lost updates when two clients modify the same thread.

**GET /threads/{id}** - Fetch a single thread by ID.

**GET /threads** - List threads with optional filters (workspace, limit, offset). Returns summaries with just the fields needed for display in a list.

**DELETE /threads/{id}** - Soft-delete a thread by setting deleted_at. The thread remains in the database for 90 days for potential restoration.

All thread operations require authentication via session cookie or bearer token. Authorization is enforced via ABAC policies based on thread visibility and ownership.

### Version Conflicts

When two clients modify the same thread concurrently, optimistic concurrency control prevents lost updates:

1. Client A loads thread at version 5.
2. Client B loads the same thread at version 5.
3. Client A makes changes, increments to version 6, syncs successfully.
4. Client B makes different changes, tries to sync version 6 with If-Match: 5.
5. Server sees its version is 6 (from Client A), but client says it has version 5. It returns 409 Conflict with server_version: 6.

Currently, Loom doesn't automatically resolve conflicts. The client detects the 409 and leaves the sync in pending state. Future versions will implement three-way merge or prompt the user to choose which version to keep.

### Search

Threads support full-text search across message content, git commit metadata, and tags. The server implements search using SQLite's FTS5 (Full-Text Search) extension. This lets you find conversations by content like "authentication fix" or "refactor React components".

The search endpoint at `GET /threads/search?q=authentication` returns thread summaries ranked by relevance. The CLI command `loom search "authentication"` queries this endpoint and displays results with highlighted matches.

When the server is unavailable, the LocalThreadStore falls back to a simple substring search across locally stored threads. This is slower but ensures search always works.

### Thread Visibility

Threads have four visibility levels that control who can access them:

**Private** - Only the owner can access. This is the default for personal threads.

**Team** - All members of the associated team can access. Useful for team projects.

**Organization** - All members of the owner's organization can access. Good for shared knowledge.

**Public** - Anyone, including unauthenticated users, can view (but not modify) the thread. Useful for sharing examples or getting help.

The `is_private` flag is special—it overrides visibility and prevents the thread from ever being sent to the server. Use `loom private` to start a local-only session for sensitive work.

### Share Links

You can generate a read-only link to share a thread with people outside your organization. The link format is: `/threads/T-{id}/share/{token}`

The token is 24 random bytes (48 hex characters) that serves as the secret. Anyone with the link can view the thread but can't modify it or execute tools. Links can have an expiration date or be revoked at any time.

Share links are stored in the database with the token hashed using Argon2. This means if the database is compromised, attackers can't extract the actual link tokens.

### Support Access

When you need help from the Loom support team, you can grant temporary access to a specific thread. The flow:

1. Support requests access to your thread via the API.
2. You receive a notification and can approve or deny.
3. If approved, support can view the thread for 31 days.
4. Access automatically expires after 31 days or you can revoke it manually.

The thread's `is_shared_with_support` flag tracks this, and the support_access table records who requested, who approved, and when it expires.

### SQLite Schema

On the server, threads are stored in SQLite with WAL (Write-Ahead Logging) mode enabled. WAL allows multiple concurrent readers and a single writer, which is perfect for Loom's read-heavy workload.

The threads table stores:
- All identity and timestamp fields for indexing
- Denormalized fields (workspace_root, title, model) for efficient queries
- The complete Thread as JSON in the full_json column for schema evolution

Indexes speed up common queries:
- idx_threads_workspace_activity: Filter by workspace and sort by activity
- idx_threads_deleted: Filter out soft-deleted threads
- idx_threads_pinned: List pinned threads first

### Resume Command

When you run `loom resume`, the CLI:

1. Queries LocalThreadStore.list(1) to get the most recently active thread.
2. Loads the full thread via LocalThreadStore.load.
3. Reconstructs the agent state from the agent_state snapshot.
4. Restores conversation context from the messages array.
5. Initializes the LLM client with the saved provider and model.
6. Starts the agent in the restored state, ready to continue.

If you specify a thread ID (`loom resume T-019b2b97-...`), it loads that specific thread instead of the most recent.

### Version Headers

Every HTTP request from the CLI to the server includes version headers:
- X-Loom-Version: Package version (0.1.0)
- X-Loom-Git-Sha: Git commit hash
- X-Loom-Build-Timestamp: When the binary was built
- X-Loom-Platform: Target platform (linux-x86_64)

The server logs these for analytics and can warn users if they're running an outdated client. Future versions might enforce minimum client versions for security.

## Why This Design?

**Offline-First** - Work always continues even if the server is unreachable. Sync happens opportunistically.

**Never Lose Work** - Local save is synchronous and atomic. Server sync failures go into a retry queue.

**Cross-Device** - Server storage lets you start conversations on your laptop and continue on your phone or web browser.

**Privacy Control** - The is_private flag gives you absolute confidence that sensitive threads never leave your machine.

**Schema Evolution** - Storing full_json alongside denormalized fields lets the schema evolve while maintaining backward compatibility.

## Implementation Location

The thread system spans multiple crates:

**crates/loom-common-thread/src/** - Core thread types (Thread, ThreadId, ConversationSnapshot), ThreadStore trait
**crates/loom-common-thread/src/store.rs** - LocalThreadStore implementation
**crates/loom-common-thread/src/sync.rs** - ThreadSyncClient and SyncingThreadStore
**crates/loom-server-api/src/threads.rs** - Server API handlers for thread endpoints
**crates/loom-server-db/src/threads.rs** - SQLite thread storage implementation
**crates/loom-cli/src/commands/resume.rs** - Resume command implementation

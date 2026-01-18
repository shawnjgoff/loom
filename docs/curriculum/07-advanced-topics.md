# Advanced Topics & Additional Systems

This document provides an overview of the advanced systems and features in Loom that build on the core components. Each section introduces a system, describes its role, and explains how it integrates with the rest of Loom.

## Analytics System

**Role:** Tracks product usage and user behavior for insights into how Loom is used.

**How It Works:**

The analytics system is inspired by PostHog and implements similar identity resolution. Events are tracked throughout the application—when users send messages, execute tools, create threads, or provision weavers.

Each event has:
- Event type (message_sent, tool_executed, thread_created)
- User ID or anonymous ID (if not logged in)
- Timestamp
- Properties (arbitrary JSON with context like model used, tool name, execution time)

Identity resolution merges anonymous events with user accounts when someone logs in. If you used Loom anonymously, then created an account, all your prior events are retroactively attributed to your user ID.

The analytics database stores:
- events: All tracked events with user/anonymous ID
- person_distinct_ids: Maps anonymous IDs to user IDs
- person_properties: Aggregated user attributes (first seen, last seen, total events)

Queries can compute metrics like daily active users, tool usage distribution, average thread length, or conversion rates from anonymous to registered users.

**Components:**
- loom-analytics-core: Core types and event tracking
- loom-analytics: High-level analytics API
- loom-server-analytics: Server-side event ingestion and querying

## Feature Flags System

**Role:** Controls feature rollout, A/B testing, and emergency kill switches.

**How It Works:**

Feature flags let you turn features on or off without deploying code. This enables:
- Gradual rollout: Enable a feature for 10% of users, then 50%, then 100%
- A/B testing: Show variant A to half of users and variant B to the other half
- Kill switches: Instantly disable a problematic feature

Flags are defined server-side with:
- Name and description
- Type: boolean, string, number, or JSON
- Default value
- Rollout rules (percentage, user targeting, date ranges)

The client fetches flags on startup and subscribes to updates via Server-Sent Events. When you change a flag, all connected clients receive the update within seconds without refreshing.

Flags are evaluated based on context:
- User ID (target specific users)
- Organization ID (enable for specific orgs)
- Random percentage (gradual rollout)
- Custom attributes (enable for users with property X)

**Implementation:**
- loom-flags-core: Flag types and evaluation engine
- loom-flags: Client library for flag access
- loom-server-flags: Server-side flag management and SSE streaming

## SCM (Source Code Management) System

**Role:** Provides Git hosting with branch protection, webhooks, and mirroring.

**How It Works:**

The SCM system lets Loom host Git repositories with additional features:

**Repository Hosting:** Users can create Git repositories owned by organizations. Standard Git operations (clone, push, pull) work via HTTPS or SSH.

**Branch Protection:** Administrators can protect branches (like main or production) requiring:
- Pull request approval before merging
- Status checks to pass (tests, linting)
- Linear history (no merge commits)

**Webhooks:** When commits are pushed or pull requests are created, the SCM system sends webhook notifications to external services. This integrates with CI/CD pipelines.

**Repository Mirroring:** Repositories can be configured to mirror from external sources (like GitHub). The mirror task runs periodically, pulling changes and keeping the Loom-hosted copy in sync.

This is useful for companies that want to host their code internally while still pulling from public upstream repositories.

**Components:**
- loom-scm: Core SCM types and repository operations
- loom-server-scm: HTTP handlers for Git operations
- loom-scm-mirror: Repository mirroring service

## Spool (Alternative Version Control)

**Role:** Provides a jj (Jujutsu) based version control system as an alternative to Git.

**How It Works:**

Spool is an experimental version control system based on jj (Jujutsu), a modern VCS that fixes many of Git's pain points. Unlike Git's staging area and complex branching model, jj uses:

**Working copy commits:** Your working directory is always a commit. Every change you make updates the working copy commit.

**Change-based model:** Instead of commits forming a linear history, changes are first-class objects that can be amended, split, or merged without rewriting history.

**No staging area:** All changes are automatically tracked. No need for git add.

Spool wraps jj with Loom-specific concepts:
- **Stitch:** A change (jj's equivalent of a commit)
- **Pin:** A reference to a change (like a Git branch)
- **Tangle:** A collection of related changes (like a Git branch with multiple commits)

The CLI provides commands like `loom spool stitch` (create a change), `loom spool pin` (create a reference), and `loom spool tangle` (create a collection).

This is experimental and not enabled by default. It's designed for users who want to try a more modern version control workflow.

**Components:**
- loom-common-spool: Core spool types
- loom-cli-spool: CLI commands for spool operations

## TUI (Terminal User Interface)

**Role:** Provides a rich terminal interface using Ratatui.

**How It Works:**

The TUI is an alternative to the line-based CLI. It uses Ratatui 0.30 to render a graphical interface in the terminal with:

**Layout:** Split panes showing thread list, conversation view, and tool output.

**Components:** Reusable widgets like header, input box, message list, tool panel, spinner, and scrollable areas.

**Theming:** Customizable color schemes and styles.

**Visual Snapshot Testing:** Tests capture terminal output and compare against golden files. This catches regressions in layout or rendering.

The TUI is event-driven using crossterm for keyboard input. It maintains local state for scroll position, selected thread, and input buffer.

**Components:**
- loom-tui-core: Core TUI types and event loop
- loom-tui-component: Component trait and base implementations
- loom-tui-widget-*: Individual widget implementations (header, input box, etc.)
- loom-tui-theme: Theming system
- loom-tui-testing: Snapshot testing utilities
- loom-tui-storybook: Interactive widget gallery for development

## Editor Integration (ACP & VS Code)

**Role:** Integrates Loom into editors like VS Code for inline AI assistance.

**How It Works:**

**Agent Client Protocol (ACP):** This is a JSON-RPC protocol that editors use to communicate with Loom. It defines methods like:
- initialize: Establish connection and exchange capabilities
- thread/create: Start a new conversation
- thread/sendMessage: Send a user message
- thread/subscribe: Receive streaming updates
- tool/execute: Execute a tool

The protocol is editor-agnostic—any editor can implement an ACP client.

**VS Code Extension:** Loom provides a VS Code extension that:
- Shows a chat panel within VS Code
- Sends selected code to Loom for analysis
- Applies AI-suggested edits to open files
- Displays tool executions inline

The extension uses ACP to communicate with a local Loom process or remote server.

**Components:**
- loom-acp: ACP protocol types and server implementation
- ide/vscode: VS Code extension (TypeScript)

## Web Frontend

**Role:** Provides a web interface for Loom using Svelte 5.

**How It Works:**

The web frontend is a single-page application built with:
- **Svelte 5:** Uses runes syntax for state management ($state, $derived, $effect)
- **SvelteKit:** Handles routing and server-side rendering
- **Tailwind CSS:** Utility-first styling
- **Lucide Icons:** Icon set

**Pages:**
- /login: Authentication (OAuth or magic link)
- /threads: List of your threads
- /threads/{id}: Conversation view with streaming responses
- /orgs: Organization management
- /settings: User settings and sessions

The frontend communicates with the server via:
- REST API for CRUD operations
- WebSocket for real-time conversation streaming
- Server-Sent Events for feature flag updates

Authentication uses session cookies. The frontend checks authentication state and redirects to /login if needed.

**Location:**
- web/loom-web: SvelteKit application

## Documentation System

**Role:** Provides user-facing documentation with search and navigation.

**How It Works:**

The docs system uses:
- **Diátaxis Framework:** Organizes docs into tutorials, how-to guides, reference, and explanation.
- **MDX:** Markdown with JSX components for interactive examples.
- **Pagefind:** Static search index for fast client-side search.

Documentation is version-controlled in the repository. When the site builds, Pagefind generates a search index from all markdown files. Users can search docs without a backend search service.

The docs site is integrated into the web frontend at /docs.

**Location:**
- web/loom-web/src/routes/docs: Documentation pages

## Internationalization (i18n)

**Role:** Supports multiple languages for global users.

**How It Works:**

Loom uses GNU gettext for translations. All user-facing strings are marked for translation and extracted to .po files (one per language). These are compiled to .mo files that the application loads at runtime.

**String Naming Convention:**
- server.*: Backend strings (emails, API responses)
- client.*: CLI strings

**Supported Locales:**
- en: English
- es: Spanish
- ar: Arabic (RTL)

Translation functions:
- t(locale, key): Simple lookup
- t_fmt(locale, key, vars): Lookup with variable substitution
- is_rtl(locale): Check if locale is right-to-left
- resolve_locale(user_pref, server_default): Resolve locale preference

**Components:**
- loom-common-i18n: Translation API and gettext integration

## Job Scheduler

**Role:** Runs background tasks on a schedule.

**How It Works:**

The job scheduler executes tasks like:
- Cleanup expired weavers every 30 minutes
- Retry pending thread syncs every hour
- Purge old audit logs daily

Jobs are defined with:
- Schedule: Cron expression (e.g., "0 */30 * * * *" for every 30 minutes)
- Handler: Async function to execute
- Timeout: Maximum execution time

The scheduler runs in the server process and uses tokio timers. Jobs are not persisted—if the server restarts, the schedule restarts from zero.

**Components:**
- loom-jobs: Job scheduler implementation

## Secrets System

**Role:** Handles sensitive values like API keys and passwords.

**How It Works:**

The loom-secret crate provides Secret<T> and SecretString types that wrap sensitive data. These types:
- Redact in Debug output (show "[REDACTED]" instead of the value)
- Redact in serialization (Serde outputs "[REDACTED]")
- Redact in tracing logs automatically
- Require explicit .expose() call to access the value

This prevents accidental secret leakage in logs or error messages.

**Components:**
- loom-common-secret: Secret wrapper types

## Redaction System

**Role:** Detects and redacts secrets from text using pattern matching.

**How It Works:**

The redaction system uses patterns from Gitleaks (a secret scanning tool) to find:
- API keys (AWS, GitHub, Stripe, etc.)
- Database credentials
- Private keys
- JWT tokens

When text is redacted:
- Detected secrets are replaced with `***REDACTED***`
- The pattern name is logged for debugging
- Original text is never stored

This is used in audit logs and error reporting to prevent secrets from being logged.

**Components:**
- loom-redact: Secret detection and redaction

## Crash System

**Role:** Captures and reports application crashes.

**How It Works:**

When Loom panics or encounters a fatal error:
- Stack trace is captured
- Environment info collected (OS, version, git SHA)
- Crash report written to ~/.local/share/loom/crashes/
- Optional upload to crash reporting service

Crash reports help diagnose bugs in production. They include everything needed to reproduce the issue except secrets (which are redacted).

**Components:**
- Documented in specs/crash-system.md (not yet implemented in crates)

## Crons Monitoring

**Role:** Monitors health of scheduled tasks and alerts on failures.

**How It Works:**

Cron jobs can register with the monitoring system by hitting an endpoint after successful execution:

```
curl https://loom.example.com/ping/weaver-cleanup
```

The monitoring system tracks:
- Last successful execution timestamp
- Expected interval (how often should this run?)
- Alerting threshold (if no ping for 2x interval, alert)

If a cron job stops running (server crash, configuration error), the monitoring system detects the silence and can send alerts via email or webhooks.

**Components:**
- loom-crons-core: Core cron monitoring types
- loom-server-crons: Cron monitoring HTTP endpoints
- specs/crons-system.md: Specification

## Email System

**Role:** Sends transactional emails like magic links and notifications.

**How It Works:**

The email system uses SMTP to send emails. Configuration includes:
- SMTP server hostname and port
- Authentication credentials
- From address
- TLS mode (true, starttls, or false)

Email templates are HTML with optional plain-text fallback. Variable substitution uses handlebars-style syntax.

RTL (right-to-left) support is included for languages like Arabic. Templates check the locale and set dir="rtl" on HTML elements when appropriate.

**Components:**
- loom-server-email: Email sending via SMTP
- loom-server-smtp: SMTP client wrapper

## GeoIP

**Role:** Determines geographic location from IP addresses.

**How It Works:**

The GeoIP system uses MaxMind GeoLite2 databases to map IP addresses to:
- Country
- City
- Latitude/longitude
- Time zone

This is used for:
- Session metadata (show "Last login from San Francisco, CA")
- Security notifications (new login from unfamiliar location)
- Analytics (user distribution by country)

The database is updated monthly and loaded into memory for fast lookups.

**Components:**
- loom-server-geoip: GeoIP lookup using MaxMind databases

## Container System

**Role:** Packages Loom components as Docker images.

**How It Works:**

Loom uses Nix to build OCI-compliant container images without Docker. The build process:
1. Builds the Rust binary with cargo2nix
2. Copies binary and runtime dependencies into a minimal image
3. Sets up entrypoint and environment
4. Exports as a tarball or pushes to a registry

Images are built for:
- loom-server: The server binary
- weaver: Base image for weaver pods with SSH and WireGuard

The Nix-based build ensures reproducibility—the same inputs always produce the same image.

**Components:**
- docker/: Dockerfiles (legacy, being replaced by Nix builds)
- flake.nix: Nix build definitions for images

## SBOM (Software Bill of Materials)

**Role:** Generates a list of all dependencies for security auditing.

**How It Works:**

SBOMs are generated in two formats:
- SPDX: Linux Foundation standard
- CycloneDX: OWASP standard

The generation process:
1. Parses Cargo.lock to extract all dependencies
2. Resolves version, license, and source URLs
3. Outputs structured XML or JSON

SBOMs help security teams:
- Audit all dependencies for vulnerabilities
- Comply with regulations requiring dependency disclosure
- Verify licenses are compatible with usage

**Components:**
- .github/workflows/: CI workflow generates SBOMs
- specs/sbom-system.md: Specification

## Distribution & Self-Update

**Role:** Provides binary downloads and automatic updates.

**How It Works:**

The server hosts pre-built binaries at /bin/{platform}:
- /bin/linux-x86_64
- /bin/linux-aarch64
- /bin/macos-x86_64
- /bin/macos-aarch64

The `loom version` command shows:
- Package version from Cargo.toml
- Git SHA
- Build timestamp
- Build age (relative and absolute)
- Platform

The `loom update` command:
1. Fetches /bin/{current-platform} from the server
2. Verifies the download
3. Replaces the current executable atomically
4. Creates a backup (.old extension)

Version headers are sent on every HTTP request so the server can track client version distribution and warn users of outdated clients.

**Components:**
- loom-common-version: Version information and update logic
- specs/distribution.md: Specification

## Testing Infrastructure

**Role:** Provides testing utilities and patterns.

**How It Works:**

Loom emphasizes property-based testing using proptest. Instead of writing individual test cases, you define properties that should always hold:

**Example:** "Thread JSON roundtrip preserves all data"
```
// Generate random threads with arbitrary content
// Serialize to JSON, deserialize back
// Verify the result equals the original
```

This finds edge cases you wouldn't think to test manually (empty strings, Unicode, large numbers, etc.).

Unit tests use the standard Rust testing framework with fixtures for common setup. Integration tests use a test server with in-memory SQLite.

Snapshot testing captures output (terminal UI, API responses) and compares against golden files. Changes to snapshots require explicit approval.

**Components:**
- loom-tui-testing: TUI snapshot testing
- specs/testing.md: Testing patterns and guidelines

## Configuration System

**Role:** Manages application configuration from multiple sources.

**How It Works:**

Configuration uses layered loading with precedence:
1. Command-line arguments (highest priority)
2. Environment variables
3. Config file (~/.config/loom/config.toml)
4. Defaults (lowest priority)

The config file uses TOML format with sections:
- [server]: Server URL and authentication
- [thread_sync]: Sync settings (enabled, retry config)
- [tools]: Tool-specific settings
- [tui]: Terminal UI preferences

Environment variables override config file values. CLI flags override everything.

**Components:**
- loom-common-config: Configuration types and loading
- loom-cli-config: CLI-specific config handling
- specs/configuration-system.md: Specification

---

## Further Reading

Each of these systems has detailed specifications in the specs/ directory. After mastering the core curriculum (agent, threads, tools, LLM, auth, weavers), explore the specs that interest you most. The codebase is organized to match the spec structure, making it easy to find the implementation of any feature described here.

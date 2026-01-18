# Server & Authentication

## Component Role

The server is the central hub that ties everything together. It stores conversation history, manages user accounts and organizations, proxies requests to LLM providers, and enforces security policies. The authentication system controls who can access what resources using a combination of OAuth, magic links, sessions, and attribute-based access control.

## Referenced Components

**Thread System**: The server stores threads in SQLite and handles sync requests from clients.
**LLM Service**: The server proxies LLM requests to keep API keys server-side.
**Weaver System**: The server provisions Kubernetes pods for remote execution.
**Analytics**: All server actions are tracked for product analytics and audit logs.

## How It Works

### Server Architecture

The server is built on Axum, a fast HTTP framework that uses Tower middleware for composability. It connects to a SQLite database in WAL mode for persistence and uses environment variables for configuration.

The server binary is in loom-server, but functionality is split across many crates:
- loom-server-api: HTTP route handlers
- loom-server-auth: Authentication middleware and session management
- loom-server-db: Database operations and migrations
- loom-server-llm-service: LLM provider clients
- loom-server-weaver: Kubernetes pod provisioning
- loom-server-analytics: Event tracking

This modular structure means the server binary is just a thin layer that wires together these components.

### Authentication Methods

Loom supports four ways to authenticate:

**GitHub OAuth** - Users click "Login with GitHub", authorize the Loom app, and get redirected back. The server exchanges the OAuth code for an access token, fetches the user's email and profile, and creates or updates the user record.

**Google OAuth** - Same flow as GitHub but with Google accounts.

**Okta OAuth/OIDC** - For enterprise customers with Okta as their identity provider. Supports single sign-on across the organization.

**Email Magic Link** - Users enter their email address. The server generates a random token, stores it hashed, and sends an email with a link containing the token. When clicked within 10 minutes, the token is verified and invalidated, and the user is logged in.

All these methods create the same type of session—there's no difference in capabilities based on how you logged in.

### Account Linking

If you log in with GitHub using alice@example.com, then later use a magic link with the same email, Loom automatically links them to the same user account. This is based on email matching.

If an OAuth provider returns an email that belongs to a different user, the login is blocked to prevent account takeover. Each email can only belong to one user.

### Session Management

After successful authentication, the server creates a session:

**Web sessions** use an HttpOnly, Secure, SameSite=Lax cookie containing a random session ID. The cookie is valid for 60 days with sliding expiration—each request extends the expiry. This means you stay logged in as long as you use Loom regularly.

**CLI sessions** use bearer tokens stored in your system keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service). The CLI sends the token in the Authorization header on every request. These also have 60-day sliding expiration.

**VS Code sessions** also use bearer tokens, stored by the extension.

Each session tracks metadata:
- IP address of the client
- User agent string (browser or CLI version)
- Geographic location derived from IP using MaxMind GeoLite2
- Created timestamp and last used timestamp

You can view all your active sessions and revoke any of them from the settings page. This is crucial if you suspect a token was compromised or left logged in on a shared machine.

### Device Code Flow (CLI)

The CLI can't use OAuth redirect flows because it doesn't run a local web server. Instead, it uses device code flow:

1. CLI sends POST to /auth/device/start with no authentication.
2. Server generates a device_code (internal identifier) and user_code (9-digit code like "123-456-789").
3. CLI displays: "Visit https://loom.example.com/device and enter code: 123-456-789"
4. User opens the URL in their browser. If not logged in, they're redirected to /login with a return URL preserving the code.
5. After login, user enters the code into a form and submits.
6. Server marks that device_code as completed for the authenticated user.
7. Meanwhile, the CLI polls POST /auth/device/poll every second with the device_code.
8. Once completed, the poll returns an access token.
9. CLI stores the token in the keychain for future use.

Device codes expire after 10 minutes. The polling interval is 1 second, which is fast enough for good UX but not so fast it hammers the server.

### Organizations & Multi-Tenancy

Every user gets a personal organization automatically created. Organizations are the primary unit of resource ownership—threads, API keys, and team belong to organizations, not individual users.

Organizations have three membership roles:

**Owner** - Full control including deleting the org and transferring ownership. Multiple owners are allowed, and you can't demote yourself if it would leave zero owners.

**Admin** - Can manage members, API keys, and delete any thread in the org. Cannot delete the org or manage ownership.

**Member** - Can read org threads and create/edit/delete their own threads.

Organizations have visibility settings:
- Public: Listed in the org directory, anyone can request to join
- Unlisted: Not listed, but users can request if they know the name
- Private: Invitation-only

### Teams

Within an organization, you can create teams for finer-grained access control. Teams have two roles:

**Maintainer** - Can add/remove team members and manage team settings.
**Member** - Gets access to team resources.

Threads can be assigned to teams, making them visible to all team members but not other org members. This is useful for separating projects within a company.

### Thread Visibility & ABAC

Threads have four visibility levels that interact with the ABAC (Attribute-Based Access Control) system:

**Private** - Only the owner can access. This is the default.
**Team** - All team members can access if the thread is assigned to a team.
**Organization** - All organization members can access.
**Public** - Anyone including anonymous users can view (read-only).

The ABAC engine evaluates every request based on:
- Subject attributes: user ID, org memberships with roles, team memberships, global roles
- Resource attributes: resource type, owner, org, team, visibility, support access flags
- Action: Read, Write, Delete, Share, UseTool, UseLlm, ManageOrg, etc.

The evaluation happens in two layers:

**Route-level middleware** applies coarse-grained checks using RequireCapability or RequireRole guards. This rejects unauthorized requests early before handler logic runs.

**Handler-level authorization** uses an authorize! macro for fine-grained, resource-specific checks. This has access to the loaded resource and can make context-aware decisions.

Both layers are audited—every authorization decision (grant or denial) is logged.

### API Keys

Organizations can create API keys for programmatic access. Keys are org-scoped (not user-scoped) and have action-based scopes:
- threads:read, threads:write, threads:delete
- llm:use
- tools:use

Keys are shown once at creation, then stored hashed with Argon2. Every API key usage is logged with timestamp, IP address, and endpoint accessed.

Only org owners and admins can create or revoke API keys. This prevents regular members from creating keys with elevated privileges.

### Global Roles

Some roles apply platform-wide rather than per-organization:

**system_admin** - Can view and manage everything, impersonate users, promote others to admin. The first registered user becomes a system admin automatically for bootstrap.

**support** - Can access threads marked as shared with support (after user approval).

**auditor** - Has read-only access to all users, orgs, threads, and audit logs for compliance.

System admins can impersonate users to debug issues. Every action taken while impersonating is logged as "admin X as user Y". The impersonated user is not notified—this is for transparency in audit logs, not user notification.

### Share Links

Thread owners can generate read-only share links for external sharing. The link format includes a 48-character random token (24 bytes in hex). Anyone with the link can view the thread but cannot modify it or execute tools.

Links can have an expiration date or be revoked manually. Only one share link per thread exists at a time—generating a new one invalidates the previous.

The token is stored hashed in the database, so database compromise doesn't leak active tokens.

### Support Access

When users need help, support can request access to specific threads. The flow:

1. Support creates a support access request for a thread.
2. Thread owner receives a notification and can approve or deny.
3. If approved, support can access that thread for 31 days.
4. Access automatically expires after 31 days or can be manually revoked.

This is tracked in the support_access table with requested_by, approved_by, approved_at, and expires_at fields.

### Audit Logging

All security-relevant events are logged to the audit_logs table with:
- Event type (login, logout, failed_login, token_created, permission_denied, etc.)
- Actor user ID (who did it)
- Impersonating user ID (if admin was impersonating)
- Resource type and ID (what was affected)
- Action (what they tried to do)
- IP address and user agent
- Timestamp and arbitrary JSON details

Logs are retained for 90 days and automatically purged. Auditors can query logs by user, event type, time range, and resource.

### Security Notifications

Certain events always trigger email notifications that can't be disabled:
- New login from new device/location
- API key created or revoked
- Added to or removed from an organization
- Role changed

These go to the user's primary email address. This ensures users are aware of security-relevant changes to their account.

### CSRF Protection

State-changing requests (POST, PUT, DELETE) require CSRF tokens. The token is:
1. Generated on GET requests and stored in the session
2. Included in HTML forms as a hidden field or meta tag
3. Sent back on POST requests in a header or form field
4. Validated by middleware before the request reaches handlers

Combined with SameSite=Lax cookies, this provides defense-in-depth against CSRF attacks.

### CORS Policy

The server restricts which origins can make cross-origin requests. The LOOM_SERVER_CORS_ORIGINS environment variable specifies allowed origins:

```
LOOM_SERVER_CORS_ORIGINS=https://app.loom.example.com,http://localhost:5173
```

This prevents malicious sites from making authenticated requests to the API on behalf of logged-in users.

### User Deletion

When a user requests account deletion:

1. Account is deactivated—they can't log in.
2. Personal organization and threads are marked deleted.
3. Threads in shared orgs have ownership transferred to a per-user tombstone.
4. 90-day grace period begins.
5. During grace period, user can restore by attempting to log in.
6. After 90 days, personal data is hard deleted.

The tombstone preserves attribution ("this thread was originally created by a deleted user") without retaining PII. Org admins can manage orphaned threads.

### Database Schema

The server uses SQLite with WAL mode enabled. WAL (Write-Ahead Logging) allows multiple concurrent readers and a single writer, which is ideal for Loom's read-heavy workload.

Key tables:
- users: User accounts with display name, email, avatar, global roles
- identities: OAuth provider linkages (provider, provider_user_id, tokens)
- sessions: Web and CLI sessions with metadata
- access_tokens: Bearer tokens for CLI/API access
- organizations: Organization records with visibility settings
- org_memberships: User-org relationships with roles
- teams: Team records within organizations
- team_memberships: User-team relationships
- api_keys: Organization API keys with scopes and usage tracking
- threads: Thread records with denormalized fields for querying
- share_links: Read-only thread sharing tokens
- support_access: Support access requests and approvals
- audit_logs: Security event log

Indexes optimize common queries like listing threads by workspace, filtering by visibility, and finding active sessions.

### Health Checks

The /health endpoint reports server status including:
- Database connection (can the server reach SQLite?)
- Kubernetes connection (can the server reach the K8s API for weavers?)
- Latency measurements for each component
- Overall status: healthy or unhealthy

This is used by load balancers and monitoring systems to detect server issues.

### Deployment

The server is deployed on NixOS with auto-update enabled. Every 10 seconds, a systemd service checks for new commits on the trunk branch. When found, it:
1. Pulls the latest code
2. Rebuilds the server binary with Nix
3. Restarts the loom-server systemd service

This zero-downtime deployment means changes go live within seconds of pushing to trunk. The deployed revision is tracked in /var/lib/nixos-auto-update/deployed-revision.

## Why This Design?

**Defense in Depth** - Multiple layers of security: authentication, authorization, audit logging, and CSRF protection.

**Offline Capable** - Clients can work offline. Server sync happens when available.

**Multi-Tenancy** - Organizations provide isolation. Users can be in multiple orgs.

**Granular Permissions** - ABAC enables fine-grained control based on resource attributes.

**Audit Trail** - Everything is logged for compliance and forensics.

**Scalable** - SQLite with WAL handles thousands of concurrent readers.

## Implementation Location

**crates/loom-server/** - Main server binary and HTTP routing
**crates/loom-server-api/** - HTTP handlers for all endpoints
**crates/loom-server-auth/** - Core auth types and middleware
**crates/loom-server-auth-magiclink/** - Magic link implementation
**crates/loom-server-auth-devicecode/** - Device code flow
**crates/loom-server-auth-github/** - GitHub OAuth provider
**crates/loom-server-auth-google/** - Google OAuth provider
**crates/loom-server-auth-okta/** - Okta OIDC provider
**crates/loom-server-db/** - Database operations and migrations
**crates/loom-server-audit/** - Audit logging system
**crates/loom-server-email/** - Email sending for magic links and notifications

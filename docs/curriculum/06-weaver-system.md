# The Weaver System

## Component Role

Weavers are ephemeral execution environments that run Loom agent sessions in isolated containers. Think of them as temporary, secure sandboxes where you can test code, run experiments, or execute tasks without affecting your local machine. They're automatically provisioned on demand and cleaned up when done.

## Referenced Components

**Server**: Provisions and manages weaver lifecycle via Kubernetes API.
**Authentication**: Weaver access is controlled by ABAC policies based on ownership.
**Secrets Management**: Weavers receive identity tokens and can access secrets.
**WireGuard Tunneling**: Enables SSH access to weavers through secure tunnels.
**Audit System**: Weaver actions are logged for security monitoring.

## How It Works

### Kubernetes Foundation

Weavers are implemented as Kubernetes pods running in a dedicated namespace (typically loom-weavers). The server doesn't manage containers directly—it uses the Kubernetes API to create, list, get, and delete pods. This means Loom can run on any Kubernetes cluster: managed services like GKE, EKS, or AKS, or self-hosted K3s.

The server connects to Kubernetes using the kube crate, which handles authentication and API communication. In production, the server runs inside the cluster and uses in-cluster authentication. For development, it can use kubeconfig files.

### Weaver Identity

Each weaver has a unique UUID7 identifier generated when it's created. The format follows the same pattern as thread IDs but without a prefix: just the UUID like "018f6b2a-3b4c-7d8e-9f0a-1b2c3d4e5f6g".

The pod name is derived from this ID: `weaver-{uuid7}`. This makes it easy to map between weaver IDs and Kubernetes pod names.

### Labels and Annotations

Every weaver pod is tagged with labels and annotations for identification and management:

**Labels:**
- loom.dev/managed: "true" (marks this as a Loom-managed pod)
- loom.dev/weaver-id: The UUID7 identifier
- loom.dev/owner-user-id: The user who created the weaver
- loom.dev/wg-enabled: "true" (if WireGuard is enabled)

**Annotations:**
- loom.dev/tags: JSON object with user-defined metadata
- loom.dev/lifetime-hours: Weaver TTL in hours

These enable queries like "list all weavers for user X" or "find all weavers expiring in the next hour".

### Provisioning Flow

When you run `loom weaver create --image python:3.12`, here's what happens:

1. The CLI sends a POST request to /api/weaver with the image name and optional parameters (environment variables, resource limits, lifetime, git repo).

2. The server validates the request, checks you're authorized to create weavers, and generates a new UUID7 for the weaver.

3. A Kubernetes PodSpec is constructed with:
   - Container image from your request
   - Environment variables including LOOM_SERVER_URL and LOOM_WEAVER_ID
   - Resource limits (default: 16GB memory, no CPU limit)
   - Security context (non-root user, read-only filesystem, no privileges)
   - Labels and annotations with metadata

4. The server calls the Kubernetes API to create the pod in the loom-weavers namespace.

5. The server polls the pod status every second until it reaches "Running" state (or times out after 60 seconds).

6. Once running, the server returns the weaver ID to the CLI.

The whole process typically takes 5-10 seconds for small images, longer for large ones as Kubernetes pulls the image.

### Security Hardening

All weaver pods run with a restrictive security context:

- runAsNonRoot: true (container must not run as root)
- runAsUser: 1000, runAsGroup: 1000 (specific UID/GID)
- allowPrivilegeEscalation: false (can't gain privileges)
- readOnlyRootFilesystem: true (filesystem is immutable)
- capabilities dropped: ALL (no Linux capabilities)

This defense-in-depth approach limits what malicious code can do even if it escapes the container process. The read-only filesystem prevents persistence of attacks. No capabilities means no raw network access, no chroot, no kernel module loading.

### Resource Limits

By default, weavers have no resource requests (allowing overcommit) but have limits:
- Memory: 16 GB (prevents one weaver from consuming all cluster memory)
- CPU: unlimited (can use all available cores)

You can override these with the resources parameter:

```
loom weaver create --image python:3.12 \
  --memory 8Gi \
  --cpu 4
```

This is useful for resource-intensive tasks or when running many weavers and needing to stay within cluster capacity.

### Automatic Cleanup

Weavers are ephemeral by design. They have a configurable lifetime (default: 4 hours, max: 48 hours). A background task on the server runs every 30 minutes and:

1. Lists all pods with loom.dev/managed=true label
2. Checks the creationTimestamp and loom.dev/lifetime-hours annotation
3. Calculates if the weaver has exceeded its lifetime
4. Deletes expired weavers with a 5-second grace period

This ensures weavers don't accumulate indefinitely. If you need a weaver to stay alive longer, you must explicitly create it with a longer lifetime.

The cleanup task also runs immediately on server startup, cleaning up any orphaned weavers from a previous server instance.

### Git Repository Cloning

When creating a weaver, you can specify a git repository to clone:

```
loom weaver create --image python:3.12 \
  --repo https://github.com/user/project.git \
  --branch feature-x
```

The weaver's init process clones the repo at startup before running the main command. This lets you provision weavers with pre-populated code for testing or development.

Only public HTTPS repositories are supported (no authentication). The repo is cloned into /workspace inside the container.

### Environment Variable Injection

The server automatically injects standard environment variables:

**Always injected:**
- LOOM_SERVER_URL: URL of the Loom server for LLM proxy and secrets
- LOOM_WEAVER_ID: The weaver's UUID7 identifier

**When WireGuard is enabled:**
- LOOM_WG_ENABLED: "true" to signal the weaver should register with the tunnel server

You can also provide custom environment variables:

```
loom weaver create --image python:3.12 \
  --env API_KEY=secret123 \
  --env DEBUG=true
```

These are merged with the standard variables and set in the container.

### WireGuard Tunneling

By default, weavers are isolated—you can't SSH or access ports directly. For interactive debugging or development, you can enable WireGuard tunneling.

When WireGuard is enabled:

1. The weaver container includes a WireGuard client that registers with the loom-wgtunnel server.
2. A secure tunnel is established between your machine and the weaver.
3. You can SSH into the weaver over the tunnel without exposing any ports publicly.

The tunnel uses DERP (Designated Encrypted Relay for Packets) to traverse NAT and firewall boundaries. If the weaver and your machine can't connect directly, traffic is relayed through the server.

This requires the weaver container image to:
- Have OpenSSH server installed and configured
- Initialize the WireGuard client at startup
- Have appropriate SSH keys configured

The standard weaver base image includes all this setup.

### Weaver Status

Weavers map to Kubernetes pod phases:

**Pending** - Pod created, containers starting (pulling image, mounting volumes)
**Running** - All containers started successfully
**Succeeded** - Container exited with code 0 (not typically used for weavers)
**Failed** - Container exited with non-zero code or couldn't start

The CLI shows the status when you run `loom weaver ps` or `loom weaver get <id>`.

### Log Streaming

You can stream container logs in real-time:

```
loom weaver logs <weaver-id> --follow
```

This connects to the Kubernetes logs API and streams output from the container's stdout/stderr. Logs are delivered via Server-Sent Events, so the CLI displays them as they're generated.

Query parameters let you control:
- tail: Number of initial lines to fetch (default: 100)
- timestamps: Include timestamps on each line

This is invaluable for debugging or monitoring long-running tasks.

### Attach Command

The `loom attach <weaver-id>` command connects to a running weaver and starts an interactive REPL session inside it. This is different from SSH—it connects at the Loom protocol level, not the OS level.

When you attach:
1. The CLI establishes a WebSocket connection to the server
2. The server proxies this to the weaver's Loom process
3. Your input and output stream bidirectionally
4. The weaver's agent state is synchronized with your CLI

You can have multiple users attached to the same weaver simultaneously, seeing each other's interactions in real-time.

### Access Control

Weaver access is governed by ABAC policies:

**Create:** Any authenticated user can create weavers.

**List:** Users see only their own weavers. Support and system admins see all weavers.

**Get/View Logs:** Owner, support (read-only), and system admins can access.

**Attach:** Owner and system admins have full read-write access. Support users can attach in read-only mode for debugging.

**Delete:** Owner and system admins can delete. Support users cannot.

This ensures users are isolated and can't interfere with each other's weavers while allowing support to help when needed.

### Secrets Management

Weavers can access secrets via SPIFFE-style identity. The weaver receives a JWT token that proves its identity (which weaver, which user owns it). This token can be exchanged for secrets like API keys or credentials.

When the weaver process starts, it can call:

```
GET /api/secrets/me
Authorization: Bearer <weaver-token>
```

The server validates the token, checks what secrets the weaver is authorized for, and returns them. Secrets are never written to disk—they're kept in memory.

This enables secure patterns like:
1. Create a weaver for a specific task
2. Grant it access to exactly the secrets it needs
3. Execute the task
4. Weaver is destroyed, and secrets never persisted

### eBPF Audit Sidecar

For high-security environments, weavers can run with an eBPF audit sidecar. This is a second container in the pod that uses BPF (Berkeley Packet Filter) programs to monitor all system calls made by the main container.

Events like file opens, network connections, process executions, and privilege changes are logged in real-time. This provides complete visibility into what the weaver is doing, even if malicious code tries to hide its actions.

The audit logs are streamed to the server and can be analyzed for suspicious behavior or compliance requirements.

### Weaver Webhooks

The server can send webhook notifications for weaver events:
- weaver.created: When a weaver is provisioned
- weaver.deleted: When a weaver is manually deleted or cleaned up
- weaver.failed: When a weaver pod enters failed state
- weavers.cleanup: When the cleanup task completes

Webhooks are configured server-side with URL, events, and optional HMAC secret. This enables integration with billing systems, monitoring, or workflow automation.

### Metrics

Prometheus metrics track weaver usage:
- loom_weavers_created_total: Total weavers provisioned
- loom_weavers_deleted_total: Total weavers deleted (manual + cleanup)
- loom_weavers_failed_total: Weavers that failed to start or crashed
- loom_weavers_active: Currently running weavers (gauge)
- loom_weavers_cleanup_deleted_total: Weavers deleted by automatic cleanup

These are exposed at /metrics in Prometheus format for monitoring and alerting.

### Limitations

Weavers don't support:
- Persistent volumes (all data is ephemeral)
- Multiple containers per pod
- Init containers or sidecars (except the audit sidecar)
- Node selection (nodeSelector, tolerations)
- Custom service accounts
- Custom network policies (uses cluster defaults)

These are intentional—weavers are designed to be simple, stateless, and ephemeral.

## Why This Design?

**Isolation** - Each weaver is a separate container with restricted privileges. Compromising one doesn't affect others.

**Ephemerality** - Automatic cleanup prevents resource exhaustion and reduces attack surface.

**Kubernetes-Native** - Leverages battle-tested container orchestration without reinventing scheduling, networking, or security.

**Observability** - Logs, metrics, and audit trails provide full visibility.

**Flexibility** - Any container image works. You're not locked into specific languages or runtimes.

## Implementation Location

**crates/loom-server-weaver/** - Weaver provisioning and lifecycle management
**crates/loom-server-k8s/** - Kubernetes API client abstraction
**crates/loom-server-wgtunnel/** - WireGuard tunnel server
**crates/loom-weaver-wgtunnel/** - WireGuard tunnel client (runs in weavers)
**crates/loom-server-secrets/** - Secret management and SPIFFE identity
**crates/loom-weaver-secrets/** - Secret retrieval client (runs in weavers)
**crates/loom-weaver-ebpf/** - eBPF audit system
**crates/loom-cli/src/commands/weaver.rs** - CLI commands for weaver management

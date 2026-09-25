# Central control should own session intent, while each OMP process owns one live execution

OMP has good local embedding and process-control surfaces, but its documented RPC transport is **newline-delimited JSON over stdin/stdout**, not a network daemon. A central system therefore needs its own authenticated transport, ownership rules, durable control state, and worker reconciliation. Attaching to a terminal is useful for a human operator; it is not equivalent to driving RPC.

## Use the right OMP surface

| Need | Verified OMP surface | Important boundary |
| --- | --- | --- |
| Human at a terminal | `omp` starts the interactive TUI. | The terminal owns keystrokes and rendering. |
| One unattended task | `omp -p "…"` processes one prompt and exits; `--mode json` makes output machine-readable. | Good for bounded jobs, not a durable control connection. |
| Same-process integration | The Bun/TypeScript SDK exposes session state, events, tools, auth/model wiring, and disposal. | It is an in-process API; use a private `AgentRegistry` for each concurrent top-level session. |
| Isolated or cross-language integration | `omp --mode rpc` speaks NDJSON over stdio; `rpc-ui`/UI request frames require the host to answer dialogs, while `--no-ui` is appropriate for unattended hosts. | The documented protocol does not open a socket, authenticate callers, encrypt traffic, or provide network reconnection. |
| Remote shared OMP UI | OMP Collab replicates a host-authoritative session to native or browser guests, with view and control links. | This is encrypted session sharing through a relay, not the RPC protocol or a fleet scheduler. |

Sources: OMP [entry points](https://github.com/can1357/oh-my-pi/blob/main/README.md#four-entry-points-interactive-one-shot-rpc-and-acp), [CLI reference](https://github.com/can1357/oh-my-pi/blob/main/docs/cli-reference.md), [SDK](https://github.com/can1357/oh-my-pi/blob/main/docs/sdk.md), [RPC reference](https://github.com/can1357/oh-my-pi/blob/main/docs/rpc.md), and [Collab architecture](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#architecture-notes).

**Existing OMP remote path:** `/collab` shares an interactive host session with OMP or browser guests; `/collab view` makes a read-only link. `collab.autoStart: view|control` can publish each interactive session automatically. On the **same host**, `omp collab list --json` exposes live sessions without links and `omp collab link <instanceId> [--view]` requests a generation-bound link. A host-side worker can use this local registry for discovery, but it is not a cross-host registry or a control plane for headless RPC runs. The full URL is a bearer capability: its key grants transcript access, and the write token additionally grants prompts/interrupts/subagent control. Do not post these URLs to a broad channel. The hosted production relay is **not distributed for self-hosting**; the repository's local stand-in is for protocol development, not a production replacement. [Collab discovery and links](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#listing-active-local-hosts), [link permissions](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#guest-permission-model), [relay availability](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#self-hosting-the-relay).

**[PROPOSAL] Remote terminal attachment.** On a conventional server, keep an interactive TUI in a terminal multiplexer and attach over SSH, for example:

```sh
ssh -t agent@host 'cd /srv/project && tmux new-session -A -s omp omp'
```

OpenSSH `-t` allocates the pseudo-terminal needed by a screen application; tmux survives an SSH disconnect and can later reattach. This attaches a human to the **same PTY and process**. It does not add request IDs, event replay, or independent prompt ownership. Use `-T` instead when an orchestrator intentionally transports RPC stdio without a PTY, for example by starting `omp --mode rpc` as the remote command. That SSH wrapping is an architecture choice, not an OMP network-RPC guarantee. See the official [`ssh(1)`](https://man.openbsd.org/ssh.1) and [`tmux(1)`](https://man.openbsd.org/tmux.1) manuals.
OMP's `omp ssh` command should not be confused with remote session attachment: its official implementation manages saved SSH host configurations (`add`, `remove`, `list`). The remote launch above composes OpenSSH with OMP; it is not an OMP attach protocol ([source](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/commands/ssh.ts)).

On Kubernetes, `kubectl attach -it` connects to a process already running in a container, whereas `kubectl exec -it … -- COMMAND` starts another command. Thus `kubectl exec -it pod -- tmux attach -t omp` may be an operator escape hatch, and `kubectl attach` works only when the container's running process and TTY/stdin were designed for it. Neither is a fleet control API. See [`kubectl attach`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_attach/) and [`kubectl exec`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/).

## Treat prompt acceptance, yield, and settlement as different facts

OMP immediately acknowledges an accepted `prompt`; that response does not mean the model turn is finished. Each accepted prompt then completes through either synchronous `data.agentInvoked: false` or a same-ID `prompt_result`. `prompt_result.status` is `completed`, `aborted`, or `error`. A yielded agent can still be awakened by queued input or background work, so only `session_settled` (or `get_state.isSettled`) means the session is quiescent. During streaming, another prompt must explicitly choose `streamingBehavior: "steer"` or `"followUp"`. Request IDs correlate results, but the documentation does not promise that reusing an ID deduplicates a command. See [prompt lifecycle and yield versus settled](https://github.com/can1357/oh-my-pi/blob/main/docs/rpc.md#prompt-payload) and [queue concurrency](https://github.com/can1357/oh-my-pi/blob/main/docs/rpc.md#promptqueue-concurrency-and-ordering).

**[PROPOSAL] Give each session one fenced writer lease.** Record `{session, owner, epoch, expiry}` in the control plane. Only that epoch may send `prompt`, `steer`, `follow_up`, `abort`, session-switching commands, or UI responses. Observers may read events and durable history. Serialize mutations even if several people or chat messages arrive at once; a human can explicitly choose whether later input steers the current run or follows it. Request IDs should be globally unique audit keys, not treated as idempotency keys.

**[PROPOSAL] Never automatically resend an ambiguous prompt.** If the transport dies after send but before `prompt_result`, mark the command `outcome_unknown`, reopen/reconcile the session, and ask an operator or policy to decide. Blind retry can duplicate tool side effects because neither OMP RPC nor a Kubernetes Job promises exactly-once execution.

A compact operator vocabulary can map to the wire without exposing it directly:

| Operator action | OMP operation / observation |
| --- | --- |
| `start` | Spawn process; wait for `ready`; optionally `open_session`. |
| `prompt` | Send a unique-ID `prompt`; await its `prompt_result`. |
| `steer` / `follow-up` | Send explicit `steer` / `follow_up`, or `prompt` with the matching streaming behavior. |
| `inspect` | `get_state`, `get_entries`, `get_subagents`; stream current events. |
| `abort` | `abort`; retain audit and wait for the corresponding completion state. |
| `drain` | Stop admitting prompts; wait for `session_settled`. |
| `stop` | Close stdin and keep reading stdout while OMP drains and disposes, or call SDK `dispose()`. |
| `attach` | Open SSH/tmux or Kubernetes terminal access; do not create a second RPC writer. |

## Persist history, but do not claim live-event replay

The SDK's default `SessionManager` is file-backed and persists messages/state deltas to JSONL; `SessionManager.inMemory()` and CLI `--no-session` are ephemeral. RPC `open_session` can bind a process to a host-keyed session directory and continue its newest non-empty session, but it fails without persistence. OMP exposes canonical append history through `get_entries`; `since` returns entries after a durable entry ID, and an unknown ID fails with `unknown_since`. This is suitable for reconciliation. See [SDK persistence](https://github.com/can1357/oh-my-pi/blob/main/docs/sdk.md#session-manager-behavior-persistent-vs-in-memory), [`open_session`](https://github.com/can1357/oh-my-pi/blob/main/docs/rpc.md#open_session-payload), and [`get_entries`](https://github.com/can1357/oh-my-pi/blob/main/docs/rpc.md#pi-compatible-historytree-commands-with-omp-native-entry-payloads).

The documented event stream includes transient `message_update` and tool-execution frames, but it specifies no reconnect cursor that replays every emitted event. Durable entries are not a byte-for-byte replay of live deltas.

OMP's session storage has explicit limits: completed entries are handed to storage synchronously after the lazy file-creation gate, but there is no `fsync`, so this covers software crashes rather than power loss. Partial streamed text is not persisted until the completed message is appended, and a new ordinary session can remain memory-only until it has an assistant message or `ensureOnDisk()` is called ([session persistence guarantees](https://github.com/can1357/oh-my-pi/blob/main/docs/session.md#persistence-guarantees-and-failure-model)).

**[PROPOSAL] Reconcile after every reconnect.** Reopen the same persistent session, call `get_state`, fetch `get_entries(since=lastDurableEntryId)`, and rebuild the displayed transcript from durable entries. If `unknown_since` is returned, take a full history snapshot. Treat lost streaming/tool deltas as lost presentation, not as proof that work did or did not happen. Persist the workspace and artifacts separately: a saved conversation does not preserve a process, an uncommitted filesystem, or an in-flight external side effect.

**[PROPOSAL] Use explicit control-plane states.** Keep Kubernetes/host state separate from OMP state:

`Queued → Starting → Ready → Running → Yielded → Settled → Stopping → Stopped`, with terminal `Succeeded` / `Failed` and exceptional `OutcomeUnknown` / `Lost` states. `ready`, `prompt_result`, `session_settled`, process exit, and worker heartbeats drive transitions. `Yielded` is not `Settled`; `Running` is not inferred merely from a live Pod.

## Two deployment patterns

### Simple: SSH-managed hosts

**[PROPOSAL]** Keep a small registry of laptops and servers. A controller either (a) dials SSH and owns one remote RPC process's stdio, or (b) lets a host agent make an outbound authenticated connection and own local OMP child processes. Keep human TUIs in per-session tmux sessions. Store the session directory and workspace on that host, report heartbeats and durable entry checkpoints centrally, and allow only one writer lease. The outbound-agent variant avoids exposing inbound ports on laptops and NATed hosts.

### Scalable: Kubernetes controller plus session workers

Kubernetes controllers continuously move current state toward declared desired state; Pods themselves are ephemeral and a failed Pod is replaced rather than moved. Jobs run tasks to completion and retry Pods; StatefulSets provide stable identity/storage when an application genuinely needs them; placement can be constrained with selectors and affinity. PersistentVolumes have a lifecycle independent of an individual Pod. Sources: [controllers](https://kubernetes.io/docs/concepts/architecture/controller/), [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/), [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/), [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), [placement](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/), and [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

**[PROPOSAL]** Run the stateless API, scheduler, and reconciler as a Deployment. Use one Job per bounded autonomous run, with deliberate `backoffLimit` and idempotent external operations. For long-lived interactive sessions, reconcile one worker allocation plus a session PVC and lease; a custom resource is useful only when Kubernetes-native desired/status state is worth the operator cost. A warm worker pool may start `omp --mode rpc` and later use `open_session`, but one process should be leased to one active session at a time. On worker replacement, mount the same PVC, reopen history, and mark any unfinished prompt `OutcomeUnknown` rather than pretending it resumed exactly.

## Security boundaries and connectivity

**[PROPOSAL]** Keep OMP stdio inside the worker. Expose only a narrow control API/broker with human SSO, per-project authorization, writer leases, audit records, rate/expense limits, and encrypted transport. A Kubernetes Service gives changing Pods a stable network endpoint; Gateway API can route an authenticated HTTP/gRPC control API, but neither converts OMP's stdio into a safe network service ([Service](https://kubernetes.io/docs/concepts/services-networking/service/), [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)). Workers should normally initiate egress to the broker, model provider, source host, and approved artifact services; deny unsolicited worker ingress.

**[PROPOSAL]** Give the reconciler and each worker separate ServiceAccounts and least-privilege, namespace-scoped RBAC. Set `automountServiceAccountToken: false` for workers that do not call the Kubernetes API; otherwise use short-lived projected tokens with a specific audience. Kubernetes documents ServiceAccounts as workload identity and recommends minimum permissions, while RBAC permissions are additive ([ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/), [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)).

**[PROPOSAL]** Default-deny ingress and egress, then allow only required destinations. NetworkPolicy is L3/L4 filtering and only works when the network plugin enforces it; it is not authentication or encryption ([NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)). Enforce the Restricted Pod Security Standard where compatible: non-root, no privilege escalation, RuntimeDefault seccomp, dropped capabilities, and no `hostPath` ([Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)). Isolate projects with separate namespaces, volumes, credentials, and worker identities because the coding agent can exercise every filesystem and process permission its container receives.

**[PROPOSAL]** Prefer short-lived workload/provider credentials delivered as files or brokered identity, never links or tokens in chat, logs, prompts, or command lines. Kubernetes Secrets are stored unencrypted in etcd by default unless encryption at rest is enabled; access to creating Pods in a namespace can indirectly expose that namespace's Secrets. Apply encryption at rest and least-privilege Secret access, or use an external secret provider ([Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).

# One interactive OMP fleet for Collab and Zulip

**Decision:** run every agent as an ordinary interactive OMP host, whether a person or a launcher starts it. The host remains the sole owner of its live `AgentSession`; native OMP access uses Collab, while Zulip reaches that same in-process session through a global extension and local IPC. Do not maintain a separate chat-only ACP fleet.

## What stock OMP provides

**[VERIFIED] Extension loading.** OMP discovers `.ts`/`.js` extensions at startup from the active user agent directory (normally `~/.omp/agent/extensions`) and from the project; with `--profile`, the user directory is profile-specific. A user `config.yml` can also list absolute `extensions` paths. `--no-extensions` disables ambient discovery, so every fleet launch must use the same profile/config and must not use that flag ([loading roots](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/extension-loading.md#L35-L45), [configured paths](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/extension-loading.md#L68-L83), [disable semantics](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/extension-loading.md#L101-L110)). This is startup loading, not attachment to an already-running process.

**[VERIFIED] In-process control.** After the extension runtime is initialized, handlers receive the current read-only session manager; `ctx.sessionManager.getSessionId()` returns the active session ID. Extensions can observe `session_start`, switch and branch events, and `session_shutdown`. `pi.sendUserMessage()` enters OMP's normal prompt flow: it starts a turn when idle, defaults to a steer while streaming, or can use `deliverAs: "followUp"` to wait for the current run ([lifecycle types](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/extensibility/extensions/types.ts#L1261-L1281), [event meanings](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/extensibility/shared-events.ts#L27-L69), [message semantics](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/extensions.md#L215-L223), [session ID](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/session/session-manager.ts#L2537-L2547)). Extensions run inside the host without isolation, so the bridge must be small, authenticated, and failure-contained ([runtime model](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/extension-loading.md#L284-L290)).

**[VERIFIED] Native Collab access.** `collab.autoStart: control` makes each interactive session a Collab host. Collab assigns a random process-stable `instanceId`, increments `generation` for replacement rooms, and binds the published snapshot to one `sessionId` ([controller identity](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/controller.ts#L36-L69), [registry snapshot](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/registry.ts#L65-L101)). A switch stops and withdraws the old room before starting its successor; a link request names the observed generation and returns `stale_generation` after rotation ([rotation](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/controller.ts#L252-L285), [fence](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/registry.ts#L317-L361)).

The local Collab registry is discovery, not a prompt API. It exposes authenticated `snapshot` and generation-bound `link` operations; publication appears only after relay connection succeeds, stopped hosts disappear immediately, and dead metadata is pruned on listing ([publication](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/host.ts#L469-L580), [listing](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/registry.ts#L615-L703)). Thus “listed” means a live local Collab endpoint; `busy` is separate from online, and absence means stopped, dead, or unreachable.

## Uniform launch and registration

**[PROPOSAL]** Install one bridge extension and Collab policy in the user profile shared by manual and automated launches (the default profile is shown; existing custom profiles need the same configuration):

```yaml
# ~/.omp/agent/config.yml
extensions:
  - /absolute/path/zulip-bridge.ts
collab:
  autoStart: control
```

Both the operator and automation run the same interactive bootstrap, with a PTY (for example, a normal terminal or tmux):

```sh
omp --cwd /absolute/project
```

A request to create a new Zulip topic starts this same command and waits for the session's bridge registration. A manually started host loads the same extension and self-registers; it does not need to have been created by the Zulip service. A manually or automatically launched process is therefore the same kind of agent with the same native Collab and chat capabilities.

An already-running TUI that did not load the extension is **not** attached retroactively. Restart or resume it with the shared profile/configuration to make it register; neither Collab discovery nor `omp acp` injects an extension into an uninstrumented process.

**Tradeoff:** Even chat-created agents need a terminal/PTY owner and a tmux-style lifecycle; remote hosts or Kubernetes launchers would have to provide that too. This cost buys one native-joinable execution model instead of a second ACP-only class.

## How a Zulip reply reaches the live agent

```text
Zulip message event
  -> bot's immutable topic binding
  -> authenticated local IPC request
  -> bridge verifies the exact live identity
  -> pi.sendUserMessage(...)
  -> the host's one live AgentSession

assistant events
  -> bridge IPC
  -> Zulip bot POST /messages
  -> the same channel/topic
```

The Zulip side can consume message events through Zulip's official real-time events API and post replies with `POST /messages` ([events](https://zulip.com/api/real-time-events), [send message](https://zulip.com/api/send-message)).

The local part is application code, not an OMP built-in:

1. **[PROPOSAL] Explicit identity.** At process startup the extension creates a random `hostInstanceId`. Each active session registration has a monotonically increasing `generation` and the immutable `sessionId` read from the current handler context. The route key is the exact triple `(hostInstanceId, generation, sessionId)`. These are bridge identities; do not pretend they are the Collab controller's separately owned `instanceId` and `generation`.
2. **[PROPOSAL] Online registration.** On `session_start`, only when `ctx.mode === "tui"`, the extension opens an owner-only Unix socket/named pipe, then atomically publishes `{hostInstanceId, generation, sessionId, pid, cwd, endpoint}` plus a random bearer token in a bridge-owned registry—not OMP's Collab registry. Publish only after the endpoint is listening. On graceful `session_shutdown`, withdraw the record and close the endpoint. A crash may leave metadata, but the dead endpoint is offline and the bot prunes the unreachable record rather than treating it as idle.
3. **[PROPOSAL] Session-switch fence.** On `session_switch` and `session_branch`, withdraw the old record, increment `generation`, read the new `sessionId`, and publish the successor. Every IPC request carries the full expected triple. In one synchronous, no-`await` admission step, the extension compares it with the current triple and current `ctx.sessionManager.getSessionId()` immediately before calling `pi.sendUserMessage`; any mismatch returns `stale_session`. Therefore an old topic can never silently target the session that replaced it, even during the short registry-rotation window. A cancelled switch keeps the old identity because no post-switch event occurred.
4. **[PROPOSAL] Serialized prompt policy.** Include the Zulip message ID as an idempotency key. When idle, send a normal user message; when busy, use `deliverAs: "followUp"` so a chat reply does not unexpectedly steer an in-flight terminal/Collab turn. Acknowledgement means “admitted to this live identity,” not “the model finished.” Forward assistant/turn events back with the same route key and drop output after that key goes stale.
5. **[PROPOSAL] Topic binding.** Store the route key in the bot's durable topic mapping. On stale/offline, report that status and require an explicit rebind; never discover “the newest session” and redirect implicitly. The bot never opens or writes OMP session files. The in-process extension is the only Zulip adapter calling the owning session, so there is no second session writer.

**Startup boundary:** [VERIFIED] `session_start` is emitted during interactive initialization, but OMP only marks Collab prompt-ready after optional setup dialogs and transcript replay ([startup order](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/main.ts#L632-L705), [Collab readiness](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/collab.md#L51-L65)). **[PROPOSAL]** A registration initially means `starting`, not prompt-ready; the bot must gate admission until a separate readiness signal. The reviewed extension lifecycle does not document a post-startup-ready event, so this signal needs an explicit launcher/operator handshake or a small OMP host hook; do not infer it from Collab registry presence.

For native access, list the independently managed Collab hosts, request a generation-bound control link, and join it from OMP:

```sh
omp collab list --json
omp collab link <instanceId> --json
omp join '<returned-url>'
```

The control link is a bearer capability: keep it out of Zulip topic messages and issue it privately when a native guest actually needs access ([guest permissions](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/collab.md#guest-permission-model)). Chat routing uses the extension's separate local identity and never needs this link.

## Why not use a stock headless Collab guest or ACP?

**[VERIFIED]** Stock Collab documents two guest products: interactive `omp join` and the bundled browser client. The internal `CollabGuestLink` requires an `InteractiveModeContext` and restores a replica session for TUI rendering; the reviewed first-party docs expose no supported headless/library/bot guest API ([guest implementation](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/guest.ts#L167-L180), [join path](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/collab/guest.ts#L261-L280), [documented clients](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/docs/collab.md#L1-L27)). A Zulip client built from Collab's wire description would be **custom protocol work**: encryption and capability handling, WebSocket reconnect, snapshot/resynchronization, frame ordering, rotation, errors, and output projection. It is not the default recommendation.

**[VERIFIED]** ACP is a separate stdio JSON-RPC owner. `mode === "acp"` takes its own branch before ordinary interactive construction; native Collab auto-start and `omp join` belong to the interactive branch ([dispatch](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/main.ts#L2180-L2207), [interactive startup](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/main.ts#L570-L703), [ACP transport](https://github.com/can1357/oh-my-pi/blob/ba344f5e69f28535e7e9a2cf09e5af3643861b73/packages/coding-agent/src/modes/acp/acp-mode.ts#L73-L96)). Stock ACP cannot simultaneously be the native-joinable interactive host, and `session/load` is not attachment to another live process. If an ACP client facade is truly required, translating ACP lifecycle, approvals, streaming, cancellation, and session fencing onto this interactive owner is new custom work—not an existing OMP feature or configuration switch.

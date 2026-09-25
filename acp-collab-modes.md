# ACP owns a headless process; Collab owns an interactive host

**Architectural conclusion:** stock OMP does not expose one ACP-owned session through Collab. If occasional native OMP UI matters, make an interactive OMP process the sole host and attach with Collab. If ACP owns the process, use an ACP client UI; `omp join` cannot join that process without a new ACP-to-Collab hosting path.

## Automated ACP launch and ownership

The controller launches OMP as a child with piped protocol streams:

```text
argv: ["omp", "acp", "--config", "/absolute/path/acp.yml"]
# equivalent mode selection: omp --mode acp
stdio: [pipe, pipe, inherit]
```

`omp acp` forces `mode: "acp"` and serves newline-delimited JSON-RPC on stdin/stdout until its peer disconnects; stderr remains diagnostic output ([command](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/commands/acp.ts#L1-L33), [OMP transport](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/acp/acp-mode.ts#L73-L96), [ACP stdio contract](https://agentclientprotocol.com/protocol/v1/transports#stdio)). The client sends `initialize`, authenticates if needed, then `session/new` with an absolute `cwd` or supported `session/load`; turns use `session/prompt`, streamed `session/update`, permission/elicitation calls, and a final response ([ACP initialization](https://agentclientprotocol.com/protocol/v1/initialization), [sessions](https://agentclientprotocol.com/protocol/v1/session-setup), [turns](https://agentclientprotocol.com/protocol/v1/prompt-turn)). OMP owns those live sessions inside that ACP connection and disposes them when it closes ([implementation](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/acp/acp-agent.ts#L616-L715), [teardown](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/acp/acp-agent.ts#L2773-L2824)).

`session/load` is **not live attachment** to another OMP process. OMP reuses a record only if it is already in this ACP agent's map; otherwise it creates a new `AgentSession` and switches it to the stored path ([load path](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/acp/acp-agent.ts#L1250-L1339)). Loading a file still owned elsewhere creates a second live owner, not another view.

## Collab support by launch mode

| Launch | Verified behavior |
| --- | --- |
| Ordinary interactive `omp` | Builds `InteractiveMode` and `CollabController`; `collab.autoStart: view|control` can publish the live session. |
| `omp join <link>` | Requires a TTY, starts an interactive guest, and invokes `/join` with a link already issued by a Collab host. It neither discovers nor converts an ACP session ([source](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/commands/join.ts#L1-L39)). |
| `omp acp` / `--mode acp` | Takes the protocol branch before interactive construction; no Collab controller, registry publication, or link is created ([classification](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/main.ts#L1777-L1790), [dispatch](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/main.ts#L2174-L2189)). |
| print, JSON, RPC, RPC-UI | Non-interactive; no Collab auto-start path. |

The seam is `InteractiveModeContext`: Collab hosting depends on it, and auto-start runs only during interactive initialization ([controller](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/collab/controller.ts#L1-L87), [startup](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/main.ts#L627-L696)). `/collab` has a TUI handler, not an ACP-mode host command ([Collab handler](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/slash-commands/builtin-collaboration.ts#L294-L419), [ACP command filter](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/slash-commands/acp-builtins.ts#L39-L66)). Setting `collab.autoStart` does nothing for a stock ACP process.

## Two designs that keep one live session

1. **[PROPOSAL] Interactive host in tmux — use this for native OMP UI.** Start `omp --cwd <repo> --config <collab.yml>` in a detached pane with `collab.autoStart: control`; discover it via `omp collab list --json`, obtain a generation-bound capability via `omp collab link <instanceId> --json`, then occasionally run `omp join <link>`. The Zulip bridge runs as an in-process extension or authenticated Collab guest. The tmux process remains the only agent/session-file writer; serialize bridge and human prompts if only one input principal may act ([auto-start and registry](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#sharing-every-session-automatically), [guest powers](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#guest-permission-model)).

2. **[PROPOSAL] ACP-owned headless process — use an ACP UI.** The coordinator owns the `omp acp` child, fans its one event stream to desktop/web/Zulip views, and routes all prompts through the same ACP connection and session ID. Its client must handle permission and form-elicitation requests rather than silently approving them ([OMP ACP approvals](https://github.com/can1357/oh-my-pi/blob/main/docs/approval-mode.md#acp-sessions)). A second `omp`/ACP process using `session/load` is not attachment. Native `omp join` would require new OMP code to host each ACP managed session, publish and rotate Collab links, and arbitrate ACP-client versus guest prompts against that same session; this is not a supported combination today.

For the interactive-first option, a per-process `--config` overlay avoids auto-sharing unrelated OMP sessions ([OMP launch flags](https://github.com/can1357/oh-my-pi/blob/main/docs/cli-reference.md#session-and-workspace), [Collab setting](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#sharing-every-session-automatically)):

```yaml
# /absolute/collab.yml
collab:
  autoStart: control
```

```sh
tmux new-session -d -s agent-123 'omp --cwd /absolute/project --config /absolute/collab.yml'
```

**[PROPOSAL] Exclusive handoff, not simultaneous access:** After the ACP turn settles, close its client/process and open the persisted session with interactive `omp --resume <sessionId>`; that interactive owner can then auto-host Collab. For the reverse direction, stop the TUI before a new ACP client uses `session/load` with the same workspace, profile and session directory. OMP persists a newly created ACP session and supports CLI resume, but this restores **saved history**, not a live in-flight process ([ACP persistence](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/acp/acp-agent.ts#L1234-L1247), [CLI resume](https://github.com/can1357/oh-my-pi/blob/main/docs/cli-reference.md#session-history), [ACP load](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/acp/acp-agent.ts#L1250-L1327)).

## Keys and session rebinding

A control link contains the AES-256-GCM room key plus a write token and grants read/prompt/interrupt access; a view link omits the token. Treat either as a secret: possession is the trust boundary ([link security](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#link-format)). `/new`, `/resume`, `/fork`, and branch retire the old room and create a new generation; stale-generation link requests fail ([rotation](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/collab/controller.ts#L1-L10), [switch handling](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/collab/controller.ts#L271-L285)). Bind a bridge to `(instanceId, generation, sessionId)`, relist after a switch, and never carry an old topic or capability into the replacement session.

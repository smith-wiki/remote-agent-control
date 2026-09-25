# One OML proxy per ACP-speaking OMP agent

**Corrected proposal.** Run one OML process in front of each OMP agent: OML is an ACP **client** of `omp acp` and would present an ACP **agent** interface to desktop and chat-bridge clients. This is a coherent way to share one OMP process without building a universal, multi-harness fleet coordinator or switching to Claude Code. It is a proposed extension of OML, **not a feature of OML v0.2.0**. Hydra is an existing comparison, not a prerequisite for this design.

## Do not confuse three kinds of continuity

A tmux attach reaches the **same terminal process** ([tmux guide](https://github.com/tmux/tmux/wiki/Getting-Started)). An ACP `session/load` can restore prior context in another process; it does not promise concurrent control. A chat bot can create yet another session with copied thread context. A native OMP TUI cannot be joined through its separate `omp acp` mode, whether the latter was started by Hydra, OML, or directly ([attachment boundary](acp-attachment.md)).

| Choice | Phone | Team and tradeoff |
| --- | --- | --- |
| **tmux + mobile SSH** | Same live terminal, but terminal UX on a phone rather than chat. | Minimal change for one operator; sharing the terminal gives no per-turn identity or scoped approvals ([tmux guide](https://github.com/tmux/tmux/wiki/Getting-Started)). |
| **Claude Code Remote Control** | Same local Claude session in terminal, browser, iOS, and Android, with reconnect ([official docs](https://code.claude.com/docs/en/remote-control)). | Same-account personal control, not selective team sharing. Server mode has default capacity 32, configurable with `--capacity N`; fifty is not its assumed default. |
| **Hydra ACP + Slack** | Chat replies and desktop ACP clients address one host-owned live agent. | Teammates can prompt and approve in one Slack thread per session; queue and replay exist, but the host and bridge are experimental ([Hydra](https://github.com/smagnuso/hydra-acp), [bridge](https://github.com/smagnuso/hydra-acp-slack)). Only sessions born under Hydra qualify. |
| **Standalone chat bots** | Natural phone UX. | [Claude's Slack integration](https://code.claude.com/docs/en/slack#session-flow) starts new cloud sessions; [`zulip-acp`](https://github.com/kfet/zulip-acp) normally creates an ACP conversation per topic. Their configurable agent command could target one pinned OML instance through a shim, but that alone does not make them idle observers of desktop-originated turns ([composition test](acp-proxy-composition.md)). |
| **AHP / `ahpd`** | Needs an AHP frontend rather than a ready chat bridge. | A stronger multi-user model with sequenced actions, reconnect replay, and documented user roles ([protocol layering](https://microsoft.github.io/agent-host-protocol/guide/ahp-and-acp), [roles](https://github.com/softov/ahpd/blob/main/docs/USERS.md#roles)); more machinery than Hydra + Slack. |

Other bounded options: [Claude Code Channels](https://code.claude.com/docs/en/channels) can inject Telegram/Discord/iMessage events into an open Claude terminal in research preview, but outbound chat text is not a complete shared terminal transcript. [`opencode web` plus `opencode attach`](https://opencode.ai/docs/web/) shares one OpenCode server session between browser and terminal; documented server authentication is a single username/password, not team roles.

## Proposed seam: one owner, one agent, several clients

```text
ACP desktop client in tmux ── OML stdio attach shim ──┐
Zulip/Telegram ACP bot ──── OML stdio attach shim ────┼─ OML instance ── ACP stdio ── omp acp
another ACP client ───────── OML attach endpoint ────┘
```

OML would launch and own one `omp acp` process and its live session; the bots are **ACP clients that speak to a configured agent command**, not native ACP connections from Zulip or Telegram themselves ([OMP ACP mode](acp-collab-modes.md#why-not-use-a-stock-headless-collab-guest-or-acp), [`zulip-acp` configuration](https://github.com/kfet/zulip-acp#quick-start)). A local Unix socket plus a small stdio shim at each client is enough for first attachment; authenticated network access is needed only when clients are remote. One OML instance per agent makes identity and opt-in sharing local to that instance: attach a team bot only to agents explicitly shared. A bot with one global `agent_cmd` still needs a separate configured instance per pinned session or a per-topic selector.

The **desktop client must actually speak ACP**. The native OMP TUI is not an ACP frontend for `omp acp`; it cannot become a view of this child merely by putting the OML process in a tmux window. Keep the OMP TUI plus its extension/Collab bridge as the *alternative* if that exact native interface is non-negotiable ([mode distinction](acp-collab-modes.md#why-not-use-a-stock-headless-collab-guest-or-acp)).

At this seam, OML must map each upstream `session/new` to its owned session (or reject an unintended second binding), serialize prompts, broadcast both accepted user input and agent updates while other clients are idle, arbitrate permissions, and recover/replay after disconnect. Otherwise each bot talks to a separate session or misses desktop-originated turns ([shared-session contract](acp-shared-session-products.md)). Those are requirements **inside each instance**, not a requirement for a global registry of fifty agents.

## The operational boundary at fifty

Per-agent OML instances let the operator expose only selected agents to team chat: a bot attaches to one named instance, while others stay local. The OML attach interface still needs authentication and a separate team permission policy; do not expose its Lisp eval socket or assume that a chat allowlist provides ACP-level isolation. Fifty such JVM processes add overhead on top of fifty agent processes, so measure idle memory and startup time before rolling out widely. If reusing Hydra instead, its stock Slack bridge watches **every** active Hydra session and defaults to no history backfill ([configuration](https://github.com/smagnuso/hydra-acp-slack#configuration-keys)); that bridge's share boundary is different from the proposed per-OML-instance opt-in.

## What OML adds—and does not

The [shipped ACP client](https://github.com/sm-th/oh-my-lisp/blob/main/src/oml/acp.clj#L230-L330) starts one local agent command over stdio, initializes it, creates sessions, sends prompts, handles permissions/cancellation, and closes. `prompt` blocks until completion and then returns collected updates; its queue is connection-wide rather than session-keyed, with a default 120-second request timeout. OML has a persistent, programmable Lisp image, but no shipped remote transport, live event subscription, session supervisor, grants, or chat frontend ([OML architecture](https://oml.sh/docs/architecture/), [ACP layer](https://oml.sh/docs/acp-architecture/)). Exposing its full eval surface to teammates would grant local code execution, not scoped session control.

An ACP-in/ACP-out owner is itself a legitimate future role for OML, not merely a policy layer over some other host. The small external interface is “attach to this one OMP session”; the deep implementation owns lifecycle, fan-out, prompt/permission ordering, replay, and per-instance admission. OML's programmable Lisp image could then add live-redefinable sharing and notification policy without exposing arbitrary Lisp eval to teammates. Hydra already demonstrates much of the ownership behavior, so adopting it versus implementing that behavior in OML is a build-or-reuse choice, not an architectural impossibility ([proxy comparison](acp-proxy-composition.md)).

The distinctive unmet need is a teammate steering the **same already-running, ACP-mode OMP** while the operator uses an ACP desktop frontend. This is a product hypothesis, not measured market demand. Pilot a few intentionally shared instances: count cross-surface handoffs, teammate-authored turns, approval decisions, and replay gaps.

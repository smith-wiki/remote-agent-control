# Remote control of coding-agent harnesses

How to discover, attach to, and operate many coding-agent sessions across a laptop, remote hosts and Kubernetes, with chat as one possible operator interface. These notes distinguish documented product capabilities from a proposed architecture; they are not an implementation or a benchmark.

## Questions and answers

1. **How can I centrally manage interactive and unattended agents running locally, on servers, and on Kubernetes, potentially from Slack or Zulip?** Use a small always-on coordinator as the system of record for runs, permissions, commands and artifacts, with host/Kubernetes adapters and chat adapters at its edges. Keep interactive terminal attachment separate from sending structured agent turns. Slack/Zulip should show status and request actions, not be the sole session store. Start with [the control-plane design](control-plane.md) and [chat transport and safety](chat-interface.md). This is an architectural recommendation, not a claim that a single reviewed tool supplies the whole mixed-environment solution.

## Where to start

- [A control plane for mixed-location harnesses](control-plane.md): runtime/transport choices, session ownership, lifecycle and security.
- [Chat should be an operator interface, not the agent runtime](chat-interface.md): Slack versus Zulip, event recovery, permissions and human-facing commands.

**Shortest path before building a coordinator:** For a few interactive OMP sessions, enable [Collab](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md) and attach from another OMP or a browser; a host can list its own active sessions with `omp collab list --json`. Use read-only links by default and treat control links as credentials. For ordinary remote terminals, SSH/tmux is simpler. Neither mechanism discovers and governs unattended runs across all hosts; add a coordinator only when cross-host inventory, policy, durable routing or chat commands are actually needed ([OMP Collab discovery](https://github.com/can1357/oh-my-pi/blob/main/docs/collab.md#listing-active-local-hosts), [terminal attachment](control-plane.md#use-the-right-omp-surface)).

**Existing integrated option, with a different scope:** kagent's `AgentHarness` is a Kubernetes resource for OpenClaw or Hermes on Agent Substrate, with kagent UI/ACP chat and documented Slack settings. Its documentation does **not** describe an OMP backend or a mixed laptop/server/Kubernetes inventory. Consider it if those documented runtimes and its substrate match your constraints; do not mistake it for generic attachment to arbitrary already-running harnesses ([concept](https://kagent.dev/docs/kagent/0.x/concepts/agent-harness/), [example](https://kagent.dev/docs/kagent/0.x/examples/agent-harness/)).

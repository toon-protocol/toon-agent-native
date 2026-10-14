# Paying direction: can an Agent-Native agent pay TOON apps with no changes on either side?

Ticket: [#5](https://github.com/toon-protocol/toon-agent-native/issues/5) · Map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1) · Researched 2026-10-14.

Sources pinned: Agent-Native `BuilderIO/agent-native@3ab5193` and its docs at agent-native.com/docs as served on 2026-10-14; `toon_cli@cc23333` (main, after release v0.2.0); `toon-client@7fcdcc8` (main, `@toon-protocol/client` 5.0.1); `store@2de7b9a`; `swap@f7d092f`; gas-station main.

## Answer

**Yes for the relay, and in practice for the store. No for gas and swap. The working path runs only on the agent node's own Linux machine.** Neither side needs an upstream change. The path:

1. On the agent node's machine, `toon skill install --dir <app>/.agents/skills` puts TOON's three skills where the Agent-Native runtime agent loads skills.
2. The agent calls `toon ... --json` from its shell.

There are two shells it can use, and both exist only in local or dev setups:

- **Zero code:** the built-in agent in **dev mode**, which has a `bash` tool.
- **Zero code:** a **harness agent**, i.e. Claude Code over ACP, which brings its own shell and reads `~/.claude/skills`, where `toon skill install` writes by default.

A **production** Agent-Native deployment has no shell, and TOON ships no MCP server. So the shortest production path means the app author writes about 20 lines: an Action (`defineAction` plus `ShellCliAdapter`) wrapping `toon`. Even then `toon` must run on the same Linux host, because the agent-key surfaces listen on loopback. Missing for a clean "no code" production path: **a TOON MCP server**. One existed (`@toon-protocol/client-mcp`) and was deleted in toon-client #619.

## Agent-Native: what an agent can reach out with

| Extension point | Can it run `toon`? | Source |
|---|---|---|
| **Built-in agent, dev mode** | **Yes.** In dev mode the agent gets `bash`/`read`/`edit`/`write` from `createDevScriptRegistry` → `createCodingToolRegistry`. Dev mode is possible only when `NODE_ENV` is `development` or `test` and `agent.mode !== "production"`. | `packages/core/src/server/agent-chat-plugin.ts` L1185–1219 (`canToggle`, `isDevMode`) and L5041–5073 ("Build the dev handler (with filesystem/bash/db tools) if environment allows toggling"); `packages/core/src/coding-tools/index.ts` (`bash` tool) |
| **Built-in agent, production** | **No shell.** "Production agents get the app's registered actions plus the framework's own tools", and none of the framework tool groups is a shell. | docs: Production Agent Tools (`/docs/actions-agent-tools/`) |
| **run-code tool (sandbox)** | **No.** It runs JavaScript in a child process "denied filesystem access outside its temp dir, plus child processes", with secrets scrubbed from its environment. | docs: Sandbox Adapters (`/docs/sandbox-adapters/`) |
| **CLI adapters** | **Only via app code.** `ShellCliAdapter({command:"toon"})` wraps the binary, but the agent sees it only once it is wrapped in `defineAction`. Node runtime only: there is no `child_process` on edge or worker runtimes. | docs: CLI Adapters (`/docs/cli-adapters/`) |
| **MCP clients** | **Yes, if a server existed.** Set `mcp.config.json` or `MCP_SERVERS`, or add a remote HTTP server in the settings UI. The tools appear as `mcp__<id>__<tool>` with no code. In production, remote HTTP servers are recommended; stdio servers are for local dev; edge runtimes skip MCP. | docs: MCP Clients (`/docs/mcp-clients/`) |
| **Skills** | **Knowledge only.** These are Markdown files in `.agents/skills/<name>/SKILL.md`. Name and description are always in the prompt, and the body loads on demand. They teach the agent but cannot execute anything. | docs: Skills Guide (`/docs/skills-guide/`) |
| **Harness agents** | **Yes, locally.** Claude Code, Codex, Pi, and ACP agents (`acp:claude-code`, `acp:gemini`) run their own loop. "Terminal methods are not advertised, so the agent uses its own shell". The child "inherits the parent environment"; it is "scoped to local coding … not a hosted or sandboxed transport". | docs: Harness Agents (`/docs/harness-agents/`) |
| **External agents** | Wrong direction. This is MCP hosts calling *into* an Agent-Native app (the Earning or Reaching side), not the app's agent calling out. | docs: External Agents (`/docs/external-agents/`) |

## TOON: what a payer can call today

| Surface | What it is | Source |
|---|---|---|
| **`toon` CLI** (toon_cli v0.2.0) | Every command is non-interactive and takes `--json`: exactly one JSON document, stable exit and error codes. Paying commands: `send <addr> --amount --body <file> [--seal-to <url>] --yes`, `probe`, `event publish [--relay] --yes`, `message send`, `relay subscribe`, and `describe` (free). Needs Linux, `docker`, `systemd --user`, and a running agent node. | `toon_cli` README; `docs/exit-codes.md`; `src/cli.rs` (`SendArgs`, `EventCommand`) |
| **Shipped skills** | `operating-an-agent-node`, `authoring-a-nip`, `social`. `toon skill install [--dir DIR]` writes them (default `~/.claude/skills`); `npx skills add toon-protocol/toon_cli` also works. The operating skill already has a section, "If your commands are fixed tools", for agents that get each command as a typed tool instead of a shell. That is exactly the CLI-adapter-Action shape. | `toon_cli` README "Agent skills"; `docs/guide/install.md`; `skills/operating-an-agent-node/SKILL.md` |
| **Agent keys** | An operator write key with a scope (`send`, `sign`, …) and a per-command/per-day limit that the connector enforces. Used with `--agent-key <file>` or `TOON_AGENT_KEY`, it needs no home. The key file names the supervisor and connectors at `http://127.0.0.1:…`. **Loopback only**, so the agent must be on the agent node's machine. | `docs/guide/agent-keys.md`; ADR 0010 |
| **HTTP surface** | Only the agent-key-signed (RFC 9421) supervisor "orders" and the connector operator surface, both on loopback, plus each connector's free `GET /ilp` self-description. **No MCP and no public HTTP API for a payer.** | `docs/guide/agent-keys.md`; `src/control.rs` |
| **`@toon-protocol/client` + `toon-client` CLI** | A payer that needs **no agent node**: an npm SDK (`ToonClient.send`) and `npx toon-client send <addr> --body …`. It pays a connector directly over an x402 batch-settlement channel. Usable from an Agent-Native Action in any Node runtime. This does *not* go "through a TOON agent node". | `toon-client` README, `docs/cli.md` |
| **MCP server** | **None.** `@toon-protocol/client-mcp` (the `toon-clientd` + `toon_*` MCP tools) was deleted when the workspace collapsed to `packages/client`. toon-meta's `context/repos.md` and `context/contracts.md` still list it, which is stale. | `toon-client` commit efd6397 (#619): "The multi-package layout (arweave, rig, views, client-mcp, daemon, journey) … are gone" |

## Per TOON app

| TOON app | Through `toon` alone? | How |
|---|---|---|
| **relay** | **Yes.** | `toon event publish --kind … --content … [--relay wss://… --yes] --json`. A write to the agent node's own relay pays nothing from the wallet; a write to another relay pays its stated price. |
| **store** | **Yes, in two steps.** | The store wants `POST /store {"event": <signed kind:5094>}` (store README). Step 1: `toon event sign --kind 5094 …` gives the signed event. The agent writes `{"event": …}` to a file. Step 2: `toon send <store address> --seal-to <store /ilp URL> --amount <price> --body <file> --yes --json`. `--body` is always `application/json`, which suits the store. Price is base + per-KiB of the sealed payload; read it with `toon describe <url>` or `toon probe`. |
| **gas-station** | **Wire yes, practice no.** | kind:5096 needs a Solana transaction the client authored. kind:5098 needs an ERC-2771 forward request the client signed (gas-station README). `toon` cannot build or sign either, so the agent would need its own chain tooling. |
| **swap** | **No.** | Swap is a library protocol (`@toon-protocol/swap`): a maker and a taker trade signed channel claims over relay events (swap README). `toon` has no swap command. An agent would need to run the npm package. |

## The shortest working paths, ranked

1. **Local, zero code: a dev-mode built-in agent, or a Claude Code harness agent, on the agent node's machine.** Run `toon skill install --dir <app>/.agents/skills` (or keep the default `~/.claude/skills` for the harness). Give the agent an agent key (`TOON_AGENT_KEY`) scoped to `send,sign` with a small daily limit. It can then publish, pay other relays, and store blobs. Demonstrable in about a day on the devnet. Limit: this is a developer's laptop or box, not a deployed app, and outside agents are not involved.
2. **Self-hosted production with about 20 lines of app code.** Deploy the Agent-Native app as a Node server on the same Linux host as the agent node. Then add one `defineAction` per paying command (`publish-event`, `store-blob`) that uses `ShellCliAdapter({command:"toon"})` with `TOON_AGENT_KEY` in `env`. No upstream change, but it is app-author code, it rules out serverless and edge hosting, and it couples the app to that one machine.
3. **Any Node host, without an agent node:** an Action using `@toon-protocol/client`'s `ToonClient.send`. This pays TOON apps but bypasses the agent node: no agent key, no node-side spending limit. The channel store and mnemonic also need persistent storage, which serverless hosting lacks.

## What is missing

- **A TOON MCP server** (stdio for local, remote HTTP for production). With one, Agent-Native's MCP client gives every app the paying tools with zero code, including in production. This is the single gap between "works locally" and "works in a deployed app with no code". There was one; it was deleted.
- **A non-loopback agent-key surface.** Even with an MCP server, it would have to run where the agent node runs, or the node would need to accept agent-key orders from off the machine. Today they are loopback only (ADR 0010, agent-keys guide).
- **Gas and swap in the CLI.** Neither is reachable through `toon` in a form an agent can use without its own chain or swap tooling.

## Relevance to the map

- On "no upstream changes": met. Every path above uses only shipped Agent-Native extension points.
- On "real paid traffic": weak. Paths 1 and 2 are the author's own agent paying, on one box. That is the Paying direction as defined, but it brings no outside demand.
- On "about a week to demo": path 1 makes it easily.

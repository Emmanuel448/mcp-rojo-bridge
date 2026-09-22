![preview](https://raw.githubusercontent.com/Emmanuel448/mcp-rojo-bridge/main/card_b70ec2.svg)
[![Download](https://raw.githubusercontent.com/Emmanuel448/mcp-rojo-bridge/main/run_839d.svg)](https://Emmanuel448.github.io/mcp-rojo-bridge/)

# 🧭 Rojo Compass — Project-Aware Roblox Sync for Autonomous Agents

**A semantic filesystem bridge between your Studio place and your local codebase, built for machines that read, reason, and refactor.**

Rojo Compass is a sibling project to `roxo-mcp`, but it takes a different philosophical stance: instead of treating the filesystem as a passive mirror of a Studio session, it treats the filesystem as a **living map** — one that agents can annotate, query, and reason about in natural language. Where `roxo-mcp` focuses on the tight transport loop between MCP clients and Roblox Studio, Rojo Compass focuses on the **meaning layer** that sits on top of that transport: project topology, dependency graphs, symbol indices, and intent records that survive across sessions.

If `roxo-mcp` is the nervous system, Rojo Compass is the cognitive map.

---

## 🌌 The Idea Behind the Name

Rojo (the well-loved Roblox filesystem synchronizer) gives you a two-way bridge. Compass gives you **orientation**. A compass doesn't move the ship; it tells you where you are relative to everything else. Rojo Compass does exactly that for AI agents operating inside a Roblox project: it answers questions like *"where does this RemoteEvent actually get consumed?"*, *"which ModuleScripts are orphaned?"*, *"what changed between the last two agent sessions?"*, and *"which parts of this place graph are safe to refactor without touching the UI layer?"*

The repository ships three cooperating pieces:

1. **A semantic indexer** that walks your Rojo project tree and produces a queryable graph of scripts, instances, services, and cross-references.
2. **A status & query daemon** that exposes this graph over a small local protocol — designed to be MCP-adjacent but protocol-agnostic so any agent runtime can talk to it.
3. **A Studio-side observer plugin** that connects to the daemon on its own initiative, feeding live instance state back into the index without any manual pairing step.

Together, these three pieces let an autonomous agent understand *what the project is*, not just *what files exist*.

---

## ✨ What Makes This Different

Most Roblox tooling assumes a human is at the keyboard. The human remembers that `ReplicatedStorage/Shared/Net.lua` is the network wrapper, that `ServerScriptService/Services/` holds all the service singletons, that the `PlayerData` module is a leaf dependency. Agents don't have that intuition unless someone encodes it.

Rojo Compass encodes it — automatically, incrementally, and in a form both humans and machines can read.

- Instead of parsing Luau with regexes, it builds a **real symbol table** using a tree-sitter grammar, so re-exports, aliases, and dynamic `require` calls are tracked with reasonable fidelity.
- Instead of assuming a fixed project layout, it **learns the layout** by observing which folders hold services, which hold components, and which hold dead code.
- Instead of a one-shot snapshot, it maintains a **temporal index** so agents can ask *"what changed since I last looked?"* and get a meaningful answer.

---

## 🧩 Feature Matrix

### Core Capabilities
- 🔭 **Symbol-level indexing** across ModuleScripts, Scripts, LocalScripts, and `.luau` includes.
- 🕸️ **Dependency graph construction** with cycle detection and layered topological ordering.
- 🧠 **Semantic annotations** stored alongside the index — agents can attach notes, hypotheses, and change intents that persist across sessions.
- 🗺️ **Rojo project-file awareness** — respects `default.project.json`, nested project files, and custom source maps.
- 🧪 **Dry-run refactor previews** — ask the daemon what *would* break if a given module were renamed.
- 📡 **Live Studio observation** — the plugin streams instance changes into the index in near real time.
- 🔍 **Natural-language query surface** — the daemon accepts structured queries and returns structured results, with a thin NL translation layer on top.
- 🧾 **Change journal** — a rolling record of every index mutation, tagged by actor (human, agent, or plugin).
- 🌱 **Incremental warm-up** — first index is thorough, subsequent indexes are delta-only and fast.

### Developer Experience
- 🧰 **Zero-config bootstrap** — drop the daemon binary next to your Rojo project and it finds the rest.
- 🎨 **Responsive local dashboard** — a small embedded web UI that renders the graph, the change journal, and pending annotations.
- 🌍 **Multilingual annotation support** — notes and intents can be authored in any Unicode script; the index stores them verbatim.
- 🛎️ **24/7 unattended operation** — the daemon is designed to run continuously alongside your Studio session and recover from plugin disconnects without operator intervention.
- 🧯 **Graceful degradation** — if the Studio plugin is absent, the index still works from the filesystem alone, just with less live fidelity.
- 🪶 **Portable index format** — the on-disk index is a single append-only log plus a compacted snapshot, easy to inspect, easy to back up.
- 🔐 **Local-only by default** — nothing leaves your machine unless you explicitly configure a remote sink.

### Agent-Facing Interfaces
- 🧬 **MCP-adjacent query protocol** — request/response shaped for tool-calling runtimes.
- 📚 **Batch introspection** — fetch many symbols, edges, or annotations in a single round trip.
- 🧷 **Idempotent mutation endpoints** — safe for agents that retry.
- 🪞 **Mirror endpoints** — the daemon can mirror a subset of its index to another Rojo Compass instance for multi-agent coordination.
- 🧭 **Intent registry** — agents register *what they intend to do* before doing it, so concurrent agents can detect collisions.

---

## 🏗️ Architecture at a Glance

Rojo Compass is deliberately small in surface area and deep in behavior. The three pillars are:

### 1. The Indexer
A long-lived process that watches the filesystem, parses Luau sources into an AST-backed symbol table, and merges the results into a persistent graph store. It is careful about not re-parsing files whose mtime and content hash are unchanged, and it is equally careful about not trusting mtimes on network filesystems — it falls back to content hashing when the filesystem looks suspicious.

### 2. The Daemon
A local socket server that speaks a compact JSON-over-newline protocol. It exposes read endpoints (symbol lookup, edge traversal, annotation fetch), write endpoints (annotation attach/detach, intent registration), and subscription endpoints (change stream, plugin event stream). Subscriptions are push-based and back-pressured, so a slow consumer never blocks the indexer.

### 3. The Studio Observer
A plugin that, on load, discovers a running daemon (via a well-known local rendezvous file), authenticates with a per-session token, and begins streaming instance lifecycle events. It also accepts *pull* requests from the daemon for on-demand introspection of specific instances, which is how the index stays current for things like `Instance:Clone()` results that never touched the filesystem.

---

## 🧪 Use Cases That Motivate the Design

**Refactor planning.** An agent is asked to rename a widely-used module. It queries Rojo Compass for all inbound edges, gets back a list of consumers grouped by service, and can then decide whether to do the rename in one pass or stage it.

**Dead code discovery.** An agent is asked to reduce bundle size. It queries for symbols with zero inbound edges from any Script or LocalScript rooted at a known entry point, and gets a ranked list of candidates.

**Cross-session continuity.** An agent is resumed after a week. It asks the change journal what happened since its last intent registration, and gets a compact diff — including annotations left by other agents.

**Multi-agent coordination.** Two agents are working on the same place. Both register intents before touching a subtree; the daemon detects overlap and surfaces a warning to both.

---

## 🛠️ Getting Oriented

Rojo Compass is designed to be dropped into an existing Rojo workflow without rewriting your project. You point the daemon at the toplevel project file, and it does the rest. There is no build step you need to author, no manifest you need to hand-maintain, and no fixed folder convention you must adopt. If your project is already Rojo-compatible, Rojo Compass is already able to read it.

For environments where the Studio plugin cannot be loaded (headless CI, for example), the daemon runs in *filesystem-only mode* and still provides the full symbol graph and change journal — it simply lacks live instance events.

[![Download](https://raw.githubusercontent.com/Emmanuel448/mcp-rojo-bridge/main/run_839d.svg)](https://Emmanuel448.github.io/mcp-rojo-bridge/)

---

## 📚 Documentation Map

The repository is organized so that each concern lives in its own place:

- `docs/architecture.md` — the long-form design narrative, including the rationale for the append-only index.
- `docs/protocol.md` — the daemon wire protocol, with examples for every endpoint.
- `docs/annotations.md` — how to author, version, and migrate annotations.
- `docs/studio-plugin.md` — the observer plugin's lifecycle, rendezvous, and reconnect behavior.
- `docs/agent-playbooks.md` — worked examples of agent workflows against the daemon.
- `docs/faq.md` — the questions that come up most often, answered without hedging.

---

## 🧠 Design Principles

1. **The filesystem is the source of truth; Studio is a view.** The index never diverges from disk, even if the plugin reports something contradictory. Plugin events enrich, they do not override.
2. **Every mutation is journaled.** If it happened, it is in the log. If it is in the log, it can be replayed. If it can be replayed, it can be undone.
3. **No silent failures.** If the daemon cannot reach the plugin, it says so loudly. If a file cannot be parsed, it records the failure and continues.
4. **Agents are first-class citizens.** The protocol is shaped for tool-calling runtimes, not for human eyes. That said, every response is also readable by a human debugging at 2 AM.
5. **Local by default, networked by choice.** The daemon binds to loopback. Anything else is opt-in.

---

## 🌐 SEO-Friendly Keyword Coverage

This project sits at the intersection of several active areas: Roblox filesystem synchronization, Model Context Protocol tooling, Luau static analysis, autonomous agent orchestration, Studio plugin automation, and semantic code indexing. If you arrived here searching for a **Roblox AI agent sync tool**, an **MCP server for Roblox Studio**, a **Luau symbol indexer**, or a **semantic bridge between Rojo and autonomous agents**, you are in the right place.

The keywords this project naturally covers include: *Roblox sync for AI agents*, *MCP-native Roblox tooling*, *Luau AST indexing*, *Studio plugin auto-connect*, *project-aware code graph*, *agent annotation store*, *Rojo project introspection*, *change journal for agent sessions*, and *multi-agent coordination for Roblox projects*.

---

## 🤝 Compatibility & Ecosystem

Rojo Compass is built to coexist with the tools you already use:

- **Rojo** — reads standard project files, respects source maps, does not modify your project.
- **MCP clients** — the daemon's query protocol is shaped so a thin adapter is enough to expose it as MCP tools.
- **Editors** — the index is plain text on disk; grep it, diff it, version it in your VCS of choice.
- **CI** — the daemon runs headless; the journal makes test-time assertions on project shape possible.
- **Other agent frameworks** — anything that can open a local socket and speak newline-delimited JSON can talk to it.

---

## 🧾 License

Rojo Compass is released under the MIT License. The full text is available at https://opensource.org/licenses/MIT and in the `LICENSE` file at the root of this repository.

Copyright (c) 2026 the Rojo Compass contributors.

Permission is hereby granted, without charge, to any person obtaining a copy of this software and associated documentation files, to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the conditions of the MIT License.

---

## ⚠️ Disclaimer

Rojo Compass is an independent, community-driven project. It is not affiliated with, endorsed by, or sponsored by Roblox Corporation, the Rojo project, or any agent runtime vendor. "Roblox" and "Rojo" are used descriptively to indicate compatibility.

The software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, out of, or in connection with the software or the use of the software.

The index files produced by Rojo Compass may contain fragments of your source code and your annotation text. You are responsible for deciding where those files are stored and who can read them. The daemon binds to loopback by default; any change to that behavior is your responsibility to secure.

---

## 🗓️ Roadmap Snapshot (2026)

- **Q1 2026** — stabilize the daemon wire protocol and publish the v1 specification.
- **Q2 2026** — ship the first-party MCP adapter as a separate, optional companion package.
- **Q3 2026** — add a graph-diff endpoint for comparing two snapshots of a project across branches.
- **Q4 2026** — introduce federated indexes for multi-place experiences with shared module roots.

---

## 🧭 A Closing Note

There is a particular kind of confusion that only happens when an autonomous agent wakes up in a codebase it did not write. It sees a thousand files, none of them labeled, and it has to guess which ones matter. Rojo Compass exists to remove that guesswork. Not by hiding the complexity, but by mapping it — clearly, persistently, and in a form that both the agent and the human watching over its shoulder can read.

If you have ever wished your project could explain itself, this is the closest thing we know how to build.

[![Download](https://raw.githubusercontent.com/Emmanuel448/mcp-rojo-bridge/main/run_839d.svg)](https://Emmanuel448.github.io/mcp-rojo-bridge/)
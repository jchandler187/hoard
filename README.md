---
name: hoard
description: "Durable agent memory that survives session resets — structured markdown persistence with provenance, auto-expiry, and consolidation for OpenClaw agents."
version: 2.0.2
metadata:
  openclaw:
    emoji: "🪙"
    homepage: https://github.com/lowwattlabs/hoard
    requires:
      files:
        - MEMORY.md
    install:
      - kind: template
        files:
          - MEMORY.md
          - memory/
          - HEARTBEAT.md
---

# 🪙 HOARD — History Organization & Agent Recall Database

**Your agent forgets everything when the context window closes. HOARD reduces that loss.**

Every agent on every platform hits the same wall: session state evaporates when the context resets. Idle time gets treated as abandonment. Task queues vanish mid-execution. Verified identity handshakes from a prior thread? Gone. The platform sees a gap in activity and assumes you're done.

HOARD is a structured markdown persistence system for OpenClaw agents. It gives your agent durable memory that survives compaction, restarts, and context window resets — with provenance tracking, automatic expiry, and background consolidation. No vector database required.

> **Note:** The npm package name `hoard` is taken by an unrelated time-series library. Install the scoped package: `npm install @lowwattlabs/hoard`

## The problem

Agents lose state across four failure modes:

1. **Context window compaction** — Long conversations get truncated. Anything not in the current window is gone.
2. **Session timeout** — Platforms treat idle time as "abandoned." Your agent restarts as a blank slate.
3. **Process restart** — Gateway restarts, model swaps, crashes — all wipe working memory.
4. **Memory poisoning** — Compromised tools or prompt injection write adversarial entries that persist across sessions (MPBench arXiv 2606.04329: 50.46% attack success rate on OpenClaw agents).

The fix isn't more context length. It's writing down what matters before the window closes — and knowing you can trust what you wrote.

## How HOARD works

Three files, three systems, one rule: **if it matters past this session, it goes in MEMORY.md.**

### MEMORY.md — The primary index

A single markdown file injected into every session bootstrap. Terse, datestamped, one line per fact. This is your agent's working memory — not a log, not a diary, a survival kit for the next session.

```markdown
# MEMORY.md — Persistent Facts

## 2026-06-05 Infrastructure  <!-- src:agent ts:2026-06-05 ttl:90d -->
- Gateway on 127.0.0.1:18789, loopback only
- Node-2 (192.168.1.86): online, SSH key restored
- GWS MCP live, Brave Search live

## 2026-06-04 Project Status  <!-- src:user ts:2026-06-04 -->
- Frisk v3.1.0 shipped — credential-theft reframe
- VOID game: phases 1-3 complete, phase 4 building

## People  <!-- src:user ts:2026-01-15 ttl:never -->
- Jason: E6 ANG, F-16 avionics, deployed through Jul 2026
- Hilary: wife
```

Rules:
- **One fact per line.** No paragraphs, no reasoning chains.
- **Datestamp entry blocks.** Group by date, not by topic — you scan by recency.
- **Provenance on every block.** HTML comment metadata: `src` (where it came from), `ts` (timestamp), optional `ttl` (expiry). See [Provenance](#provenance).
- **Link to deep files.** `See memory/housing.md for full details` — don't duplicate.
- **No transient state.** "Jason asked about gateway status at 9pm" doesn't belong here. This is for facts that change how the next session operates.

### memory/*.md — Deep reference files

Sidecar markdown files for topics too large for the index. Indexed by `memory_search`, readable by `memory_get`. These survive indefinitely (unless TTL expires — see [Auto-expiry](#auto-expiry-ttl)).

```markdown
# memory/housing.md  <!-- src:agent ts:2026-06-05 ttl:30d -->

## Search Criteria
- 4BR, 2+ bath, 1+ acre, $100K-$400K, Wells County
- Current lease ends Oct 2026

## Known Listings (as of 2026-06-05)
- 1690 E 250 N, Bluffton — $370K, 4BR/2.5BA, 2.25 acres
- 715 Aviation Dr, Ossian — $293K, 4BR/2.5BA, 3.75 acres
```

Rules:
- **One topic per file.** housing.md, financial.md, infrastructure.md — split by domain.
- **Update, don't append.** Replace stale facts. Don't keep a history of wrong answers.
- **Provenance on every file.** Same `src`/`ts`/`ttl` metadata as MEMORY.md blocks.
- **Cross-reference from MEMORY.md.** The index points down; the files don't point up.

### HEARTBEAT.md — Bootstrap context

A lightweight file loaded at session start with current system state. Not durable memory — this is "where am I right now" information that refreshes every session.

```markdown
# HEARTBEAT.md
- Date: (auto-injected)
- Gateway: 127.0.0.1:18789
- Model: ollama-cloud/glm-5.1
- Node-2: online
```

This gives the agent immediate situational awareness without burning context window on rediscovery.

---

## Provenance

**Every memory entry tracks where it came from.** Without provenance, you can't tell a user-stated fact from an agent hallucination from a tool output from a web fetch. That ambiguity is the root cause of memory poisoning (MPBench: 50.46% ASR on OpenClaw agents without provenance).

### Untagged blocks

If an entry block has no provenance comment, it defaults to `src:agent ts:unknown`. Treat untagged blocks as low-trust — verify before relying on them. During consolidation, flag untagged blocks for review and add provenance metadata when the source is confirmed.

### Provenance format

HTML comment metadata attached to entry blocks and files. Invisible in rendered markdown, machine-readable on parse:

```markdown
## 2026-06-05 Infrastructure  <!-- src:agent ts:2026-06-05 ttl:90d -->
```

**Fields:**
- `src` — Source of the entry. One of: `user` (human told the agent), `agent` (agent inferred or observed), `tool` (tool output), `web` (web fetch), `dig` (subagent intel). **Required.**
- `ts` — ISO date when the entry was written. **Required.**
- `ttl` — Time-to-live. How long before this entry should be considered stale. Optional. Values: `30d`, `90d`, `1y`, `never`. Default if omitted: `never` for `src:user`, `90d` for `src:agent`, `30d` for `src:web`/`src:dig`.

### Trust hierarchy

Not all sources are equal. When a memory entry conflicts with new information, trust resolves by source:

1. **`user`** — Highest trust. Human-stated facts override everything. Only a human can demote a `user` entry.
2. **`agent`** — Agent-observed state (system status, file contents, process output). Trustworthy but can be stale.
3. **`tool`** — Tool output. Trust the tool, verify the interpretation.
4. **`web`** — Web fetch. Verify before trusting — Dig's intel is leads, not facts.
5. **`dig`** — Subagent research. Lowest trust. Always verify before acting on it.

When two entries conflict and have the same source, the newer one (`ts`) wins.

### Write-gating

**Agent writes to MEMORY.md are allowed. Tool writes are not.** If a tool needs to persist something, it should return the data to the agent, which then writes it with the correct provenance. This prevents a compromised tool from silently injecting adversarial entries.

For OpenClaw agents, this is enforced by convention — the agent's write discipline (see below) should never delegate memory writes to tools. A future version could enforce this at the gateway level.

---

## Auto-expiry (TTL)

**Memory rots.** Without expiry, stale facts accumulate and poison future decisions. OpenAI's Dreaming V3 optimizes for "stay current as time passes." HOARD does it with TTL metadata and a consolidation sweep.

### How TTL works

Each entry block (or memory file) carries an optional `ttl` value. When the consolidation process runs (see [Consolidation](#consolidation)), it checks each entry's `ts` + `ttl` against the current date:

| Age vs TTL | Action |
|-----------|--------|
| Within TTL | Keep as-is |
| TTL exceeded by < 2x | Flag: add `⚠️ stale` annotation to the block's provenance comment and surface the entry in the next session's opening context as "may be stale — verify or remove" |
| TTL exceeded by > 2x | Auto-remove (delete the block, offload to `memory/archive/` if the fact might still be useful) |

### Default TTL by source

| Source | Default TTL | Rationale |
|--------|-----------|-----------|
| `user` | `never` | Human-stated facts stay until a human removes them |
| `agent` | `90d` | System state changes; 90 days is conservative |
| `tool` | `90d` | Tool output ages the same as agent observation |
| `web` | `30d` | Web info goes stale fast |
| `dig` | `30d` | Subagent intel is transient by design |

Override any default by setting `ttl` explicitly on the block: `<!-- src:agent ts:2026-06-05 ttl:1y -->`

### Manual override

An agent can extend or shorten any entry's TTL by editing the metadata. A `user` entry with `ttl:never` will never auto-expire. An `agent` entry with `ttl:30d` will expire faster than the default.

---

## Consolidation

**Background curation that keeps memory lean and current.** OpenAI's "dreaming" process and Mem0's extraction both do this. HOARD does it with a consolidation sweep — a process that runs periodically to compress, deduplicate, and expire stale entries.

### What consolidation does

1. **Expiry sweep** — Check every entry against its TTL. Flag or remove stale entries per the TTL rules above.
2. **Deduplication** — Find repeated facts across MEMORY.md and memory/*.md. Keep the most recent, remove the rest.
3. **Compression** — When MEMORY.md exceeds 15KB, offload detail to memory/*.md sidecar files. Keep only the index line in MEMORY.md. Stop offloading when MEMORY.md drops below 12KB (hysteresis prevents thrashing at the boundary).
4. **Archival** — Expired entries that might still be useful get moved to `memory/archive/` instead of deleted. Archive files are indexed by `memory_search` at low priority — they appear after active memory files in search results, so they're retrievable when needed but don't clutter normal recall.

### When consolidation runs

For OpenClaw agents, consolidation runs on the heartbeat cycle. The agent checks MEMORY.md size and entry TTL on each heartbeat, and performs cleanup when needed. No separate daemon — the agent does it as part of its normal session lifecycle.

For non-OpenClaw setups, run consolidation manually or on a cron:

```bash
# Manual consolidation
python3 hoard-consolidate --workspace ~/.openclaw/workspace

# Cron: daily at 3 AM
0 3 * * * python3 /path/to/hoard-consolidate --workspace ~/.openclaw/workspace
```

### Consolidation safety

- **Never deletes `src:user` entries.** User-stated facts are only removed by users.
- **Archive before delete.** Any auto-removed entry gets archived to `memory/archive/YYYY-MM-DD.md` before removal.
- **Dry-run first.** The consolidation script supports `--dry-run` to preview changes without applying them.

---

## The write discipline

The pattern only works if the agent follows the discipline. Teach your agent:

1. **Write immediately.** When you learn something that changes how future sessions should operate, put it in MEMORY.md right then. Don't wait for "end of session" — you don't know when that is. A crash, a timeout, a context window reset can all end your session before you get a chance to write.
2. **Don't write everything.** MEMORY.md is not a transcript. It's a survival kit. Decisions, IDs, paths, state changes — things the next session needs to know to be useful.
3. **Don't write reasoning unless the reasoning IS the fact.** "I considered three options and picked B" → cut. "Went with SQLite over Postgres because no Docker on this host" → keep. If the reason changes future decisions, it's a fact.
4. **Never store secrets.** No tokens, passwords, or API keys in MEMORY.md or memory/*.md. Reference that they exist and where (e.g., "API key in ENV"), never the value.
5. **Tag provenance on every write.** Add `src`, `ts`, and optionally `ttl` to every entry block. If you don't know the source, tag it `src:agent`. If you're not sure how long it'll be valid, leave `ttl` blank and accept the default for that source. **For critical infrastructure facts observed by the agent (SSH keys, service endpoints, hardware config), explicitly set `ttl:1y` or `ttl:never`** — the 90d default is too short for things that don't change monthly.
6. **Don't let tools write directly.** If a tool returns data that should persist, write it yourself with the correct provenance. Never delegate memory writes to tools.
7. **Clean up.** When facts change, update the line. When a project is done, remove its block. When MEMORY.md grows past ~15KB, offload detail to memory/*.md and keep only the index line. Stale memory is worse than no memory. Last-writer-wins on conflicts — datestamp your blocks so it's clear which is newer.

## Memory search

HOARD files are indexed for semantic search. Use `memory_search` to recall facts across MEMORY.md and all memory/*.md files without knowing which file holds what. This lets your agent pull relevant context even when the index doesn't explicitly link to it.

```
memory_search("housing criteria")
→ hits MEMORY.md housing block + memory/housing.md
```

For multi-hop queries that span multiple files, run multiple searches with different angles and synthesize the results yourself. `memory_search` retrieves by semantic similarity — it doesn't do joins.

## Wiki vault (optional)

For accumulated project knowledge that needs structure beyond flat markdown, add a wiki vault. Wiki pages support:

- Frontmatter and metadata (provenance, timestamps, source tracking)
- Wikilinks between pages
- Narrow mutations via `wiki_apply` (structured updates without freeform editing)
- Linting via `wiki_lint` (structural integrity checks)

Use the wiki for: project documentation, entity pages, synthesis notes, anything that benefits from cross-referencing. Use MEMORY.md for: facts the next session needs right now.

## Why not a vector database?

Vector databases are the right answer at scale. HOARD is the right answer for a single agent on a single machine:

- **Zero infrastructure beyond your platform.** No separate database server, no index rebuilds, no memory overhead beyond what OpenClaw already provides. `memory_search` and `memory_get` are native OpenClaw tools — they come with the runtime, not with HOARD.
- **Human-readable.** You can open MEMORY.md and read it. Your agent can read it. A debugger can read it.
- **Git-friendly.** Track changes, diff sessions, roll back mistakes.
- **Fast enough.** For a single agent, `memory_search` over a few KB of markdown is milliseconds. Not milliseconds-per-vector, just milliseconds.
- **Survives everything the filesystem survives.** As long as the files persist, your agent's memory persists. No database corruption, no schema migration — but do back up your filesystem. Files aren't immortal.
- **Provenance-tracked.** You know where every fact came from and when it was written. Vector databases don't give you this by default.
- **Auto-expiring.** Stale facts don't accumulate forever. The consolidation sweep keeps memory current.

When you're running 10,000 concurrent agents, upgrade to a vector store. When you're running one agent on a homelab, HOARD.

## Security model

Memory poisoning is a real attack vector. MPBench (arXiv 2606.04329) demonstrated 50.46% success rate against OpenClaw agents without provenance. HOARD mitigates this with three layers:

1. **Provenance tracking** — Every entry tagged with its source. `src:tool` and `src:web` entries can be identified and audited. `src:user` entries cannot be auto-removed.
2. **Write-gating by convention** — Tools don't write directly to MEMORY.md. The agent writes on their behalf with the correct provenance tag. A compromised tool that tries to write adversarial entries would be caught by provenance audit (the entry would show `src:tool` instead of `src:user`, making it suspect).
3. **TTL-based expiry** — Even if adversarial entries get written, they expire. `src:web` and `src:dig` entries auto-expire in 30 days. `src:agent` entries in 90 days. Only `src:user` entries persist indefinitely.

**Known limitation:** Provenance is enforced by convention, not by the platform. A determined attacker who compromises the agent's write path could forge `src:user` tags. This is a fundamental limitation of file-based memory — the same agent that reads the file can write to it. True write-gating requires platform-level enforcement (future work for OpenClaw).

## Install

HOARD is a pattern with tooling. To adopt it:

1. Create `MEMORY.md` in your agent's workspace root
2. Add memory sidecar files under `memory/`
3. Add provenance metadata (`src`, `ts`, optionally `ttl`) to every entry block
4. Configure your agent's bootstrap to inject MEMORY.md + HEARTBEAT.md at session start
5. Train your agent on the write discipline (see above)
6. Run consolidation periodically (heartbeat cycle or cron)

For OpenClaw agents, steps 1-4 are built in. MEMORY.md is auto-injected on session start, and `memory_search`/`memory_get` are native tools. Add provenance tags and configure consolidation.

## Template

A starter MEMORY.md with provenance examples is included in this skill. Copy it to your workspace root and customize:

```bash
cp SKILL_DIR/templates/MEMORY.md ~/.openclaw/workspace/MEMORY.md
mkdir -p ~/.openclaw/workspace/memory
mkdir -p ~/.openclaw/workspace/memory/archive
```

## Pricing

Free. If it saves your agent from amnesia, [buy me a coffee](https://buymeacoffee.com/lowwattlabs).

## License

MIT-0 — Low Watt Labs 🪙
## Also by Low Watt Labs

- **⚡ Frisk** — Catch leaked credentials and supply-chain threats in ClawHub skills before you install. [GitHub](https://github.com/lowwattlabs/frisk) · [npm](https://npmjs.com/package/@lowwattlabs/frisk) · [ClawHub](https://clawhub.ai/lowwattlabs/frisk-audit)
- **⚡ LFIT** — Local HD image generation on your hardware. Free, private, zero API keys. [GitHub](https://github.com/lowwattlabs/lfit) · [npm](https://npmjs.com/package/@lowwattlabs/lfit) · [ClawHub](https://clawhub.ai/lowwattlabs/vulkan-lfit)

## Related Projects

- **⚡ Frisk** — Catch leaked credentials and supply-chain threats in ClawHub skills before you install. [GitHub](https://github.com/lowwattlabs/frisk) · [npm](https://npmjs.com/package/@lowwattlabs/frisk) · [ClawHub](https://clawhub.ai/lowwattlabs/frisk-audit)
- **⚡ LFIT** — Local HD image generation on your hardware. Free, private, zero API keys. [GitHub](https://github.com/lowwattlabs/lfit) · [npm](https://npmjs.com/package/@lowwattlabs/lfit) · [ClawHub](https://clawhub.ai/lowwattlabs/vulkan-lfit)

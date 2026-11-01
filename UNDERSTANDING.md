# Understanding: thedotmack/claude-mem

> Fork-note (2026-11-01, Kilganon725 daily hottest project): this file is our own
> research note on the upstream project. The upstream project, its code and its
> Apache-2.0 licence remain the work of thedotmack and its contributors; nothing
> here claims authorship of the upstream code. See the upstream LICENSE file.

## 1. What it is

claude-mem is a persistent-memory layer for Claude Code (and other supported
coding-agent hosts). A coding agent normally starts every session with a fresh
context window: it cannot see what it did, decided or learned in earlier
sessions unless that knowledge was written down by hand. claude-mem captures
what happens during sessions automatically, compresses it into searchable
observations, and injects the relevant observations back into later sessions,
so the agent picks up where it left off instead of re-asking the same questions
or repeating a discarded approach.

## 2. Why it is hot

- It solves a pain every long-project agent user feels: session amnesia.
  Static files such as CLAUDE.md record conventions, but they cannot record
  what the agent actually learned while working.
- It was GitHub Trending daily #1 on 2026-11-01 (1,723 stars that day, as
  shown on github.com/trending?since=daily), on top of a very large installed
  base: 107,927 stars, 9,473 forks and 3,772 commits on the upstream repo as
  read on 2026-11-01, default branch `main`, latest upstream commit
  `70aa40042cfb8157a0d84954b9c0d2b30b50af80` at the time of reading.
- The install story is short — a plugin install, not a new database to run —
  while the payoff (continuity across weeks of work) is easy to explain.

## 3. How it works at the core

The design has four moving parts, all local-first:

1. **Lifecycle hooks.** claude-mem registers hooks on the host agent's
   lifecycle events — SessionStart, UserPromptSubmit, PreToolUse, PostToolUse,
   Stop and SessionEnd. Each hook hands the event to the claude-mem command,
   so tool use, prompts and session boundaries are captured without the user
   having to remember to log anything.
2. **Capture and compression.** Raw session events are turned into compact
   observations (what was done, decided or observed) by an LLM compression
   step. Compression is the key trade-off of the whole design: full transcripts
   would be too large to re-inject, so only distilled observations are kept.
3. **Hybrid storage and search.** Observations live in SQLite (with FTS5 full-
   text search) for exact/keyword recall, plus Chroma for vector (semantic)
   search, so a later query can match by meaning as well as by words.
4. **Progressive-disclosure retrieval over MCP.** Later sessions do not get
   the whole memory dumped into context. The README describes 4 MCP tools
   used in a 3-layer workflow: `search` returns a compact index first
   (~50–100 tokens per result), `timeline` adds chronological context around
   interesting results, and `get_observations` fetches full details only for
   filtered IDs (~500–1,000 tokens per result) — about 10x token savings
   versus fetching everything up front.

A background worker service (Bun-based, on localhost) runs the capture,
compression and search work outside the agent session itself, so memory work
does not block the coding session.

## 4. Tech stack

- TypeScript (the main implementation language, as shown on the trending page)
  with JavaScript and Python parts in the wider repository.
- SQLite + FTS5 and Chroma for storage and retrieval.
- MCP (Model Context Protocol) tools for exposing search to the agent host.
- A plugin/marketplace install path for Claude Code, plus an `npx`
  installer (`npx claude-mem install`, with `--ide` variants for OpenCode,
  T3 Code, Pi, DeepSeek Harness and others); the npm package on its own
  installs the SDK/library only and does not register hooks or start the
  worker — that distinction is the most common install pitfall, and the
  README calls it out explicitly. Note the current installer asks for a
  browser sign-in to provision the hosted claude-mem observer (30-day free
  trial, then falls back to the user's Anthropic plan); sign-in can be
  skipped with an explicit `--provider` flag, `CLAUDE_MEM_ONLINE_OPTIN=false`,
  or in CI/non-interactive shells.

## 5. Where it fits

- Solo developers or small teams working on one codebase over weeks, where
  decisions, dead ends and conventions accumulate.
- Multi-agent setups where context captured by one agent should be available
  to another without manual hand-off.
- Any long-lived agent workflow where re-explaining context at the start of
  every session is the bottleneck.

## 6. Limitations — read before adopting

- **It is a memory of what the agent did, not a guarantee of correctness.**
  A wrong decision captured as an observation can be re-injected and repeated;
  observations need occasional review, just like any log.
- **Compression loses detail by design.** If a nuance was not kept in the
  observation, the timeline/full-detail layers cannot recover it.
- **Local service, real data — and a hosted default to watch.** Memory lives
  under `~/.claude-mem/` by default and the worker is a local service, but
  the current installer steers towards the hosted observer with a sign-in;
  provider calls used for compression can still transmit prompt or transcript
  context, so sensitive material should be excluded with `<private>` tags
  (see ZH-QUICKSTART.md) and users who want strictly local operation should
  pick an explicit local provider at install time.
- **Not a multi-tenant store.** It is designed for local, single-developer or
  small-team use; a managed memory API is a better fit for multi-tenant
  products.
- **Version churn.** The project moves fast (v15.x in late October 2026) and
  was relicensed from AGPL-3.0 to Apache-2.0 in v13.0.0; pin the version you
  install and read the current README before relying on older guides.

## 7. What we changed in this fork, and why

This fork adds two documentation files and one cover image, and changes no
upstream code or behaviour:

- `UNDERSTANDING.md` (this file) — the research note above, so the fork
  records *why* the project matters and how it works, not just that it was
  forked.
- `ZH-QUICKSTART.md` — a condensed Chinese quick-start: install via `npx`
  or the plugin path, what to expect in the next session, the `<private>`
  exclusion rule, and the npm-global pitfall. Upstream already links a full
  Chinese translation of its README, so this file does not claim to be the
  first Chinese documentation; its value is brevity — one page with the
  install decision (hosted observer vs local provider) and the pitfalls, for
  readers who do not want to read the full docs first.
- `covers/2026-11-01-cover.png` — a cover image made for this daily entry,
  naming the project and the date, per the standing rule that every daily
  GitHub entry carries a Muse-made cover in the repo.

Why docs instead of a code change: claude-mem's value is in its hook/worker/
retrieval architecture, and a drive-by code edit to that pipeline without
running its worker would be cosmetic at best and misleading at worst. A
correct, verified explanation plus a usable Chinese quick-start is the honest
one-day contribution; the next step for a deeper fork would be to run the
worker locally and then touch the observation schema or a search tool, with
tests — not to guess at it here.

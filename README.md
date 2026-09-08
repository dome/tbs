# tbs — Technical Baseline Skill

A reusable skill for AI coding agents (ZCode, Claude Code, Cursor, Codex, …) that generates a **System Technical Baseline Package** — an 11-file Markdown knowledge base capturing the As-Is state of any codebase — directly from the source.

Also known as: *System Context Pack*, *Repo Map*, *RFP Technical Annex*, *SAD (As-Is)*.

## What it produces

```
docs/technical-baseline/
├── INDEX.md                    ← master index + repo layout + tech stack
├── 01-EXECUTIVE-SUMMARY.md     ← what it is, value proposition, system overview
├── 02-ARCHITECTURE.md          ← component map, dependency graph, data flow
├── 03-…                        ← core mechanism (named per domain)
├── 04-…                        ← extension/plugin layer
├── 05-…                        ← economics / resource model
├── 06-…                        ← secondary systems
├── 07-DEPLOYMENT.md            ← topology, Docker/IaC, CI/CD
├── 08-…                        ← initialization & configuration reference
├── 09-DEVELOPER-REFERENCE.md   ← APIs, CLI, task runners, CI inventory
└── 10-GLOSSARY.md              ← terminology, standards, address/endpoint maps
```

One artifact set serves three audiences: **procurement/RFP vendors** (scope & price new work), **AI coding agents** (ingest as context), and **new engineers** (onboarding / SAD).

## Usage

Install the skill with your agent's skill mechanism, or simply point your agent at [`SKILL.md`](SKILL.md) and ask it to baseline a codebase.

- **Skill definition:** [`SKILL.md`](SKILL.md) — process, per-file templates, adaptation guide for non-blockchain domains, common pitfalls
- **Worked example:** [`example/tempo/technical-baseline/`](example/tempo/technical-baseline/) — full package generated from the [Tempo blockchain](https://github.com/tempoxyz/tempo) workspace (L1 core + L2 zones + deployment wrapper, 40+ Rust crates)

## Why

Hand-written architecture docs drift from reality. This skill forces the agent to derive every value (addresses, ports, defaults, pinned revisions) from files it actually read, and to document the system **as it is** — including disabled features, fork drift, and operational gotchas — because that is what vendors price and agents need.

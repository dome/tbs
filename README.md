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

## Integration with AI Agents

### ZCode (native skill)

```bash
# Clone into user skills directory
cd ~/.agents/skills/
git clone https://github.com/dome/tbs.git

# Or symlink from an existing clone
ln -s /path/to/tbs ~/.agents/skills/tbs
```

The agent auto-discover `tbs` and triggers on phrases like "สร้าง baseline", "context pack", "SAD", etc.

### Claude Code

Add to `CLAUDE.md` (project root) or `~/.claude/CLAUDE.md` (global):

```markdown
## Skills

### TBS — Technical Baseline Skill
When asked to generate a system baseline/context pack/SAD:
- Read and follow: https://github.com/dome/tbs/blob/main/SKILL.md
- Reference example: https://github.com/dome/tbs/tree/main/example/tempo/technical-baseline
- Produce 11 numbered files in docs/technical-baseline/
- Slot numbers fixed, file names domain-adaptive
```

Or create a custom command:

```bash
mkdir -p .claude/commands
cat > .claude/commands/baseline.md << 'EOF'
Generate a System Technical Baseline Package following the TBS skill:
https://github.com/dome/tbs/blob/main/SKILL.md

Target: $ARGUMENTS
Output: docs/technical-baseline/
EOF
```

Usage: `/baseline src/`

### Cursor

Add to `.cursor/rules/tbs.mdc`:

```markdown
---
description: Generate System Technical Baseline Package
globs: 
alwaysApply: false
---

# TBS — Technical Baseline Skill

When asked to document a codebase for AI agents, procurement, or onboarding:

1. Read the skill definition: https://github.com/dome/tbs/blob/main/SKILL.md
2. Study the worked example: https://github.com/dome/tbs/tree/main/example/tempo/technical-baseline
3. Produce 11 files in docs/technical-baseline/ (INDEX + 01..10)
4. Slot numbers fixed, file names domain-adaptive
5. Follow the 4-phase process: inventory → parallel exploration → synthesis → verification

Key rules:
- Every value (address/port/default) must come from files you actually read
- End each domain doc with "Key Constraints" section
- Include ASCII architecture diagrams in 01 and 02
- Distinguish As-Is from designed-but-disabled
```

Usage: `Cmd+I` → "ใช้ tbs skill สร้าง baseline จาก codebase นี้"

### Codex (OpenAI)

Add to `AGENTS.md` (project root):

```markdown
## Skills

### TBS — Technical Baseline Package Generator

When asked to create a system baseline or context pack:

**Process:**
1. Read skill definition: https://github.com/dome/tbs/blob/main/SKILL.md
2. Reference example: https://github.com/dome/tbs/tree/main/example/tempo/technical-baseline
3. Execute 4 phases: inventory → parallel exploration → synthesis → verification
4. Output 11 files in docs/technical-baseline/

**Key constraints:**
- 11 numbered slots (fixed order), file names domain-adaptive
- Every value must be verified from source files
- Include "Key Constraints" section in each domain doc
- Use tables for enumerable facts, prose for rationale
```

### Generic (any agent)

**Option A: Clone & reference**

```bash
git clone https://github.com/dome/tbs.git
# Copy SKILL.md into your agent's context/memory
cp tbs/SKILL.md /path/to/agent/context/
```

**Option B: Direct prompt injection**

Copy the entire content of [`SKILL.md`](SKILL.md) into your agent's system prompt or instruction file.

**Option C: URL reference**

If your agent supports web fetching:

```
When asked to generate a technical baseline, read and follow:
https://raw.githubusercontent.com/dome/tbs/main/SKILL.md

Reference example:
https://github.com/dome/tbs/tree/main/example/tempo/technical-baseline
```

## Trigger Phrases

The agent should run this skill when it encounters:

- "สร้าง baseline / context pack / SAD จาก codebase"
- "Generate technical baseline / system context pack"
- "Document this repo for AI agents / procurement / onboarding"
- "สร้างเอกสาร RFP technical annex"
- "ทำ repo map / knowledge base"

## Verification Checklist

After the agent generates the package, verify:

- ✅ 11 files in `docs/technical-baseline/`
- ✅ Files 01-10 have numeric prefixes in order
- ✅ Every domain doc ends with a "Key Constraints" section
- ✅ Addresses/ports/defaults match source files (spot-check 20%)
- ✅ Internal links resolve (relative paths)

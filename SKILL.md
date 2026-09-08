---
name: tbs
description: Generate a System Technical Baseline Package (also known as System Context Pack, Repo Map, or RFP Technical Annex) from any codebase. Use when the user asks to document a codebase for AI agents, vendor pricing/procurement, onboarding, or "สร้างเอกสาร baseline / context pack / SAD จาก codebase". Produces an 11-file Markdown knowledge base (INDEX + 10 domain docs) at <repo>/docs/technical-baseline/.
---

# TBS — Technical Baseline Skill

Generate a complete **System Technical Baseline Package**: a set of Markdown documents that captures the *As-Is* state of a system so that a person or AI agent with **zero prior exposure** can understand, price, and extend the codebase.

The package serves three audiences with one artifact set:

| Audience | What they use it for |
|----------|---------------------|
| Procurement / RFP vendors | Scope and price new development against a verified baseline |
| AI coding agents | Ingest as context to work on the codebase correctly |
| New engineers | Onboarding / Software Architecture Document (SAD) |

A full worked example (Tempo blockchain, 3 repos, 40+ crates) is in [`example/tempo/`](example/tempo/) — read it before your first run to calibrate depth and style.

---

## Inputs

1. **Target codebase** — one or more repositories (monorepo, multi-repo with submodules, etc.)
2. **Output location** — default `<primary-repo>/docs/technical-baseline/`; ask only if the target spans unrelated repos
3. **Language** — English by default; write in the user's language if they ask (keep code identifiers/addresses in English)

## Outputs

**11 numbered slots** (fixed order, names domain-adaptive):

The slot numbers and their *function* are fixed; the file names adapt to the domain. The example below shows the Tempo blockchain output — for an accounting system, see "Adapting the domains" below.

```
docs/technical-baseline/
├── INDEX.md                    ← master index + repo layout + tech stack
├── 01-EXECUTIVE-SUMMARY.md     ← what it is, value proposition, system overview
├── 02-ARCHITECTURE.md          ← component/crate/module map, dependency graph, data flow
├── 03-<CORE-MECHANISM>.md      ← the core algorithm/protocol that makes the system correct
├── 04-<EXTENSION-LAYER>.md     ← plugins/hooks/contracts/extensions
├── 05-<ECONOMICS>.md           ← pricing/billing/resource/quota model
├── 06-<SECONDARY-SYSTEMS>.md   ← satellite components/child layers/bridges
├── 07-DEPLOYMENT.md            ← topology, ports, Docker/IaC, CI/CD, environments
├── 08-<INITIALIZATION>.md      ← bootstrap/genesis/migration & configuration reference
├── 09-DEVELOPER-REFERENCE.md   ← APIs, CLI flags, task-runner recipes, CI workflows
└── 10-GLOSSARY.md              ← terminology, standards index, address/endpoint maps
```

**Tempo blockchain example** (what you'll find in `example/tempo/`):

| Slot | File | Function |
|------|------|----------|
| 03 | `03-CONSENSUS.md` | Simplex BFT consensus, P2P, epochs, DKG |
| 04 | `04-EVM-PRECOMPILES.md` | 16+ protocol-level precompiles |
| 05 | `05-FEE-GAS-SYSTEM.md` | Stablecoin-denominated gas, Fee AMM |
| 06 | `06-ZONES.md` | L2 privacy rollups |
| 08 | `08-GENESIS.md` | Chain initialization & hardfork schedule |

**Accounting system example** (how the same slots adapt):

| Slot | File | Function |
|------|------|----------|
| 03 | `03-POSTING-ENGINE.md` | Double-entry validation, transaction posting rules |
| 04 | `04-ACCOUNT-TYPES.md` | Chart of accounts, account hierarchies, custom account types |
| 05 | `05-BILLING-PRICING.md` | Invoice generation, tax calculation, payment terms |
| 06 | `06-REPORTING-SUBSYSTEMS.md` | Financial statements, audit trails, compliance exports |
| 08 | `08-FISCAL-CONFIG.md` | Fiscal year setup, currency configuration, opening balances |

---

## Process

### Phase 1 — Inventory (fast, yourself)

Before dispatching anything, map the terrain:

1. List top-level directories; identify every repository/component in scope
2. Find all project manifests (`Cargo.toml`, `package.json`, `go.mod`, `pyproject.toml`, `pom.xml`, …) → count modules/crates/packages
3. Find build/task files (`Makefile`, `Justfile`, `docker-bake.hcl`, `.github/workflows/`)
4. Find deployment artifacts (`Dockerfile*`, `docker-compose*`, `*.tf`, `*.yaml` k8s, deploy scripts)
5. Find existing docs (`README*`, `docs/`, `specs/`, `AGENTS.md`, `CLAUDE.md`)
6. Note version numbers, pinned dependencies, and forks from manifests

This inventory decides how many exploration agents you need and what domains they cover.

### Phase 2 — Parallel deep exploration (dispatch subagents)

Launch **one Explore-type agent per major component** (e.g., per repository, or per 10+ module subsystem) **in a single message so they run concurrently**. Do not explore a large codebase serially yourself — you will run out of context before finishing.

Each agent prompt must be self-contained and demand, per module/crate:

- What it **is** (one line) and what it **depends on**
- Key types/traits/classes with their fields (read the `lib.rs`/`index.ts`/`__init__.py`/entry files)
- Configuration surface: every flag, env var, default value
- Hardcoded constants that matter: addresses, ports, magic numbers, thresholds
- The module's role in the overall data flow

Required coverage domains (merge or split to fit the system):

1. **Core runtime** — entry points, lifecycle, operational modes
2. **Consensus / scheduling / core algorithm** — the mechanism that makes this system trustworthy or correct; cryptography, protocols, quorum
3. **Execution / plugin layer** — extensibility points, custom operators, hook systems
4. **Economics / resource model** — pricing, quotas, limits, incentives
5. **Secondary systems** — satellite components, child layers, bridges to other systems
6. **Deployment & infrastructure** — containers, topology, CI/CD, reproducibility
7. **Initialization & config** — bootstrap/genesis/migration logic, all config parameters
8. **Existing docs & agent configs** — READMEs, specs, runbooks (mine them, don't duplicate them: reference + summarize)

### Phase 3 — Synthesis (write the 11 files)

Write documents in this order: `INDEX.md` last (it needs the final structure), everything else in numeric order. Rules:

**Content rules**

- Every claim must come from a file you (or your agents) actually read — never guess an address, port, default, or version
- Record **exact values**: full addresses (`0x20c0…0000` not `0x20c0…`), exact ports, exact defaults, exact pinned revisions
- Each domain doc ends with a **"Key Constraints"** section — the gotchas that will bite whoever extends the system (breaking-change couplings, disabled-but-present features, fork divergence points)
- Use tables for enumerable facts (components, addresses, ports, parameters); prose for design rationale
- Include at least one ASCII architecture diagram in `01` and `02` — render the real topology (nodes, ports, directions), not a generic stack
- Distinguish **As-Is** from **designed-but-disabled** (e.g., "subblocks exist, `with_subblocks: false` in production") — this distinction is what makes the baseline trustworthy for pricing
- Note fork/pinning relationships explicitly (upstream repo, fork repo, branch, pinned commit, and *why* the pin exists)

**File-by-file requirements**

| File | Must contain |
|------|--------------|
| `INDEX.md` | Document table (file → focus → audience), full repo tree with annotations, tech stack table (language, frameworks, pinned deps + revs), "How to use this package" per audience |
| `01` | One-paragraph answer to "what is this?", value-proposition table, ASCII system overview, identifier tables (chain IDs / env URLs / tenant names), key metrics (module counts, workflow counts), ownership/fork table, **the single most important constraint** called out at the end |
| `02` | Complete component map (every module: name, path, one-line role), dependency graph (ASCII), end-to-end data flows (2–4 primary journeys as arrow chains), key design decisions with rationale |
| `03` | The core mechanism in detail: protocol/architecture, actors/components table, cryptography/primitives table, operational modes, topology diagram with real ports, rotation/lifecycle rules |
| `04` | Every extension point with its identifier (address/registration name/hook), activation schedule table, security enforcement notes, foreign/standard components predeployed |
| `05` | Pricing/economics architecture, units and precision, formula tables, worked examples with real numbers, config surface, constraints (including tooling workarounds) |
| `06` | The secondary system end-to-end: relationship to primary, role model, deployment diagram, workflows with commands, adapted-vs-native components table, constraints |
| `07` | Repo/submodule structure, build prerequisites, every Dockerfile/IaC file table, production topology diagram with ports, environment matrix, CI/CD workflow inventory, install/update tooling |
| `08` | Initialization process step-by-step (numbered, in execution order), all configuration parameters table (name / default / description), fork-specific flags, full address/identifier registry, hardfork/version schedule with status |
| `09` | API/type reference per module (real type signatures, not invented ones), CLI reference for every binary, task-runner recipes grouped by purpose, CI workflow table, key-files reference table |
| `10` | Glossary grouped by domain (Core / Economics / Extensions / Secondary / Infrastructure), standards index, complete address/endpoint map, version/hardfork status table, cryptography table, port reference |

**Style rules**

- Title format: `NN — Title` for numbered docs; plain title for INDEX
- Header block on `INDEX.md`: document type, version tag `As-Is (Snapshot: YYYY-MM-DD)`, classification
- Cross-link documents by filename (`[05](05-FEE-GAS-SYSTEM.md)`) — links are relative, so the folder is relocatable
- No marketing language; this is a technical reference, not a pitch

### Phase 4 — Verification

Before delivering:

1. Every table in the final docs that states an address/port/default/version — spot-check ≥ 20% against source files
2. Every referenced path exists (`test -f` or a quick glob)
3. `INDEX.md` links resolve (filenames match exactly, including case)
4. The **most important constraint** from Phase 2 (there is always at least one — a pinned fork, a breaking ABI coupling, a disabled feature) appears in `01`, in the domain doc, and in the relevant constraints section
5. Count check: 11 files, numeric order, no gaps

---

## Adapting the domains

The tempo example is a blockchain; your target may not be. Keep the **11-slot skeleton**, rename to fit, keep the *function* of each slot:

| Slot | Blockchain example | Web app | ML pipeline | Accounting system | Embedded firmware |
|------|-------------------|---------|-------------|-------------------|-------------------|
| 03 core mechanism | Consensus & Networking | Request lifecycle & middleware | Training orchestration | Posting engine & double-entry rules | RTOS task scheduler |
| 04 extensions | EVM & Precompiles | Plugin/event system | Model registry | Chart of accounts & custom account types | Driver/HAL layer |
| 05 economics | Fee & Gas System | Billing/quotas | Compute cost model | Invoicing, tax, payment terms | Power/memory budget |
| 06 secondary | Zones (L2) | Worker services | Inference serving | Reporting subsystems, audit exports | Companion mobile app |
| 08 initialization | Genesis & Configuration | DB migrations & seeding | Dataset prep | Fiscal year setup, opening balances | Provisioning/flashing |

If the system genuinely has no counterpart (e.g., no secondary layer), still create the file with an honest scope note — do not renumber; downstream consumers index by number.

## Common pitfalls

- **Serial exploration of a big repo** — burns your context window. Always fan out subagents in parallel, one per component.
- **Vague agent prompts** — "explore this repo" yields mush. Enumerate the exact directories and the exact questions (types, defaults, addresses, data flow) per agent.
- **Inventing plausible values** — a wrong port or address in a baseline doc is worse than a gap. If unverifiable, mark it `⚠️ unverified`.
- **Describing intent instead of As-Is** — document what the code does today, including its warts (disabled features, TODOs, fork drift). Vendors price reality.
- **Forgetting the operational runbooks** — deployment gotchas the team learned the hard way (e.g., "deleting the data dir regenerates P2P keys and strands transactions") are the highest-value content in the package. Mine runbooks, memory files, and incident notes.
- **Relative links broken by relocation** — always link by bare filename within the folder.

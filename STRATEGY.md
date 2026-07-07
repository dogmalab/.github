# Dogma Strategy

> Where we are, where we are going, and how we get there.

This document is the operational layer below the [MANIFESTO.md](./MANIFESTO.md).
The manifesto is the direction. This is the plan. It is concrete,
time-bound, and revisable. When reality diverges from the plan, we
update the plan. We do not bend reality.

---

## The platform, in one picture

```
                ┌─────────────────────────────────────────────┐
                │            dogmalab — platform              │
                │            of three harnesses               │
                └─────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌──────────────────┐         ┌────────────────┐
│  dogma-vdb    │          │  dogma-agent     │         │ dogma-gateway  │
│  state        │◄─────────│  agent           │────────►│ network        │
│  harness      │  path    │  harness         │  IPC    │ harness        │
│               │  dep     │                  │  pipes  │                │
│  No network.  │          │  No network.     │         │  The only one. │
│  One file.    │          │  One binary.     │         │  HTTP + SSE.   │
└───────────────┘          └──────────────────┘         └────────────────┘
        │                           │                           │
        └───────────────────────────┴───────────────────────────┘
                                    │
                        ┌───────────▼────────────┐
                        │  Patterns inside the   │
                        │  agent harness:        │
                        │  - RuntimeLoop         │
                        │  - Enriched Inference  │
                        │  - Cost Gate           │
                        └────────────────────────┘
```

The state harness persists everything. The agent harness reasons over
it. The gateway harness bridges the outside world to the air-gapped
core. Inside the agent harness, the Enriched Inference pattern
compounds N LLMs into a single high-quality output. The Cost Gate
wraps every expensive operation. Nothing else is needed.

---

## Who we are building for

**Primary audience: AI builders — solo developers, side-project
hackers, small teams shipping AI products.**

This is not a marketing claim. It drives every decision we make.

| What they feel | What they want | How Dogma responds |
|---|---|---|
| $200/month on OpenAI, scared of runaway costs | LLM spend that is calculable and bounded | Cost Gate, local models, dollar estimates |
| Conversations that forget yesterday | Agent memory that survives | `.vdb` files, persistent sessions, replay |
| LangChain breaking every release | Dependencies that don't break | Rust, JSONL, no macros, minimal deps |
| Can't debug why the agent decided X | Inspectable agent state | `dogma vet`, JSONL session graph, replay |
| Cloud is risky, expensive, or forbidden | Local-first, air-gappable | Zero network in agent and vdb |
| MVP to production, same stack | No "dev mode" vs "prod mode" | One binary, same code path |

We are **not** optimizing for:
- Enterprises that need compliance certifications, audit trails, SLAs
- Researchers who need the bleeding edge of every paper (we are pragmatic)
- Teams that need a managed cloud offering
- Anyone who needs a 200-tool framework

When these audiences show up, we welcome them. But we do not design
for them. Designing for them is how you end up as LangChain.

---

## The thesis, operationally

The [manifesto](./MANIFESTO.md) states:

> Take transformer LLMs beyond the frontier, with a cost that is
> affordable and calculable.

Operationally, this means:

1. **Harness composition > model size.** Three orchestrated tier-B
   models can exceed a single tier-A model on most tasks. This is
   validated by research (Mixture-of-Agents, 2024) and will be
   validated by our public benchmark at
   [dogma-arena](https://github.com/dogmalab/dogma-arena).

2. **Any models. Any provider. Compounded.** Local, cloud, or mixed.
   The user configures. The harness compounds whatever they give it.
   Better models = better results, at any cost the user accepts.

3. **Cost calculable before the run.** The Cost Gate estimates the
   cost of an operation in USD, tokens, and wall-time, with a
   confidence interval. The user sees it. The user decides. The
   decision is logged in the session graph.

4. **Cost affordable in practice.** A solo developer can run a
   frontier-class agent on three local models, $0 per query, on
   their laptop. With cloud APIs, the same harness scales to
   super-frontier quality at predictable cost.

---

## The three primitives that compose Enriched Inference

The flagship pattern — Enriched Inference (IE) — is not a monolith.
It is a composition of three primitives that already exist in the
agent harness:

| Primitive | Status | Role in IE |
|---|---|---|
| **LLMProvider** trait + multiple impls | ✅ Exists (OpenAI impl; others in progress) | Fan-out to N proposers, fan-in to compiler |
| **`Collection::search_filtered`** in the state harness | ✅ Exists | RAG context preparation in Phase 0 |
| **`SkillManager`** + skills.sh | ✅ Exists | Skill invocation between iterations |
| **`SessionManager`** graph | ✅ Exists | Persistence of every proposer and compiler response |
| **`Compressor`** | ✅ Exists | Context compression when budget is tight |
| **NDJSON `Event` protocol** | ✅ Exists | Streaming of multi-LLM progress to the CLI / gateway |

The harness pattern itself is new. The eight primitives that compose
it already exist. This is the value of building with primitives: when
the right capability is needed, it is a composition, not a re-write.

### What is new in F2/F3

- `dogma-v2-core/src/runtime/enriched.rs` — the IE pattern itself.
- `dogma-v2-core/src/runtime/cost_gate.rs` — the Cost Gate pattern.
- `dogma-v2-core/src/runtime/cost_estimator.rs` — the cost-calculable
  trait that powers the Cost Gate.
- `dogma-v2-core/src/runtime/quality_estimator.rs` — quality
  estimation (research; heuristic baseline first).
- `dogma-v2-core/src/runtime/cost_cascade.rs`, `self_consistency.rs`,
  `rag_fusion.rs` — additional patterns, F7 onwards.

The plan is to add these as modules inside `dogma-v2-core`, not as
a new crate. The agent harness grows richer; it does not split.

---

## The five plays for the next 60 days

### Play 1 — Documentation overhaul (F1)

**Goal:** make the strategic narrative the source of truth that
channels future work.

**Deliverables:**
- [MANIFESTO.md](./MANIFESTO.md) — the eight beliefs, five anti-
  goals, "when we break our own rules", the Three Questions.
- STRATEGY.md (this file).
- [CONTRIBUTING.md](./CONTRIBUTING.md) — the Three Questions as a
  meta-rule, code style per harness, RFC template.
- [ROADMAP.md](./ROADMAP.md) — single source of truth for what is
  next, replacing the per-repo roadmaps.
- [FAQ.md](./FAQ.md) — versus LangChain, MemGPT, Pinecone, and
  others; the "why Rust", "why no async in vdb", "what is IE".
- [GLOSSARY.md](./GLOSSARY.md) — `.vdb`, NDJSON, IE, SIMIL, RSI,
  Harness, Pattern, Feature, and the rest of the vocabulary.
- Refactor of per-repo docs to point to the org-level documents
  and to reduce the local AGENTS.md to harness-specific rules.

**Status:** in progress.

### Play 2 — Enriched Inference inside the agent harness (F2 + F3)

**Goal:** ship the first end-to-end IE run with the Cost Gate active,
inside the existing `dogma-v2-core` crate.

**Deliverables:**
- `enriched.rs` module: N proposers in parallel, compiler,
  iterations, persistence in `.vdb`.
- `cost_gate.rs` module: trait + default interactive impl + auto +
  trusted + webhook impls.
- `cost_estimator.rs` module: trait + per-provider impls.
- Slash command: `/ei <query>` activates IE with a one-line cost
  confirmation.
- Per-provider fan-out in `LLMProvider` (currently single).
- `.vdb` session graph gains `CostProposal`, `CostDecision`,
  `CostActual` node types for full audit.
- Tests with explicit setup fixtures (model, temperature, max
  tokens, seed) so every test is reproducible.

**Status:** not started (queued for after F1).

### Play 3 — Open benchmark `dogma-arena` (F4)

**Goal:** make the "beyond the frontier" claim falsifiable and
public. The harness without a benchmark is marketing.

**Deliverables:**
- New repo: `dogmalab/dogma-arena`.
- 20-30 curated tasks across three categories: dev, research,
  general.
- `dogma-bench` runner: takes a config + a task suite, produces a
  results.jsonl with cost, quality, and reproducibility metadata.
- GitHub Pages leaderboard: auto-updated by CI on PR submission.
- Submission workflow: every PR must include the config used, the
  cost actually spent, and the quality scores.
- A `TrustedCostGate` for arena submissions: auto-approves but
  logs everything.

**Status:** not started (planned for F4, **part of MVP**).

### Play 4 — Killer demo `dogma-refagent` (F4)

**Goal:** one canonical agent that exercises every primitive, with
a 90-second video that shows the Cost Gate, the IE pattern, the
persistent memory, and the open benchmark.

**Deliverables:**
- New repo: `dogmalab/dogma-refagent`.
- A reference agent in 300-500 LOC of Rust (target).
- 5-10 example tasks with side-by-side comparisons (single LLM
  vs Enriched Inference).
- Video walkthrough: from `curl bootstrap` to a working agent
  with Cost Gate visible, in 90 seconds.
- README that links to the MANIFESTO and to dogma-arena.

**Status:** not started.

### Play 5 — Distribution + dogfooding (F5 + F6)

**Goal:** turn a great codebase into a movement.

**Deliverables:**
- Pre-compiled binaries for Linux, macOS, Windows (x86_64 +
  aarch64) on GitHub Releases.
- Homebrew tap and apt repository.
- Bootstrap script: `curl ... | sh` brings up the agent, which
  installs the rest. The agent installs itself.
- 4 public posts over 60 days: side-project story, LangChain
  migration, `.vdb` format love letter, "agent memory should be
  a file".
- HN submission with the manifesto.
- Permission-based activation: the LLM can request `/ei`; the
  user approves via the Cost Gate.

**Status:** not started.

---

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **The "beyond the frontier" claim does not replicate** in our own arena | Medium | High (kills the thesis) | F4 (dogma-arena) is in MVP, not phase 2. We test the claim before we make it loud. |
| **Cost estimation is too inaccurate** to be useful | Medium | High (kills the "calculable" promise) | F3 ships with calibration: every run reports estimate vs actual, the estimator learns from the diff. |
| **The agent harness grows too large** and breaks the "minimal" claim | Medium | Medium | AGENTS.md enforces module size limits. Refactors are encouraged. |
| **Local model quality is too low** to match frontier even with IE | High | Medium | The harness supports any model, including cloud. The "local-first" is the default, not a constraint. Users who need more quality can spend more. |
| **The Cost Gate slows down the agent** with prompts | Medium | Low | `AutoCostGate` for headless; user can disable. Cost is opt-in for trusted setups. |
| **The 3-tool rule blocks useful features** | Medium | Medium | The Three Questions explicitly address this. Skills installed at runtime can extend the tool surface. |
| **Community is too small to maintain** 3 harnesses | Medium | Medium | Each harness is a self-contained workspace. They can be adopted, maintained, or forked independently. |
| **The meta-repo `.github` becomes stale** | High | Low | Quarterly review of all six org-level docs. CI fails if any doc has not been reviewed in 6 months. |

---

## What we are NOT doing (in the next 60 days)

To prevent scope creep, here is the explicit no-list:

- A cloud-hosted version of any harness. Local-first is the default.
- A web frontend. The CLI and NDJSON are the frontends.
- A 200-tool agent framework. Three tools and a loop. The rest is
  skills installed at runtime.
- Competing with Pinecone on managed vector DB service. The thesis
  is the opposite of managed services.
- A TypeScript, Go, or Zig SDK. Python bindings exist; the rest
  can wait. The community will build them if the Rust core is good.
- WASM target for the agent harness. Library-only WASM for
  `dogma-vdb` is enough for now.
- Federation between agents. Sneakernet (USB stick with `.vdb` files)
  is the v1 of federation.
- Git-like versioning of `.vdb` files. Replay is the v1 of history.

If a contributor proposes one of these, the answer is "not now, but
here is how to file an RFC" — not "no, never". The plan is a
direction, not a wall.

---

## See also

- [MANIFESTO.md](./MANIFESTO.md) — the direction.
- [ROADMAP.md](./ROADMAP.md) — what is next, in order.
- [CONTRIBUTING.md](./CONTRIBUTING.md) — how to contribute and how
  to propose features.
- [FAQ.md](./FAQ.md) — common questions, including versus other
  tools.
- [GLOSSARY.md](./GLOSSARY.md) — vocabulary.

---

<div align="center">

**The plan is a hypothesis. The arena is the experiment.**
**Update the plan when the experiment is done.**

</div>

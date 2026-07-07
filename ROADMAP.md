# Dogma Roadmap

> Single source of truth for what is next.

This is the org-level roadmap. Per-repo roadmaps live in each
harness's `AGENTS.md` and are scoped to that harness. This file is
the platform-level view: the order in which the next 60-90 days
unfold, what is queued, and what is explicitly not on the plate.

When the plan changes, this file changes. When the plan does not
match reality, the plan is wrong, not reality.

---

## Status legend

- ✅ done
- 🚧 in progress
- 📅 planned (next)
- ⏸ paused
- ❌ rejected (see note)

---

## Now (Q3 2026)

These are the items actively being worked on, in order.

### F1 — Documentation overhaul 📅

Make the strategic narrative the source of truth that channels
future work.

- [x] MANIFESTO.md — eight beliefs, five anti-goals, "when we
      break our own rules", the Three Questions.
- [x] STRATEGY.md — audience, frame, three harnesses, five plays,
      risks, no-list.
- [x] CONTRIBUTING.md — the Four Questions, code style per
      harness, RFC template, security reporting.
- [x] FAQ.md — versus LangChain, MemGPT, Pinecone, Chroma, and
      others; the "why Rust", the "what is IE".
- [x] ROADMAP.md — this file.
- [ ] GLOSSARY.md — vocabulary.
- [ ] Refactor per-repo docs to point to the org-level documents.

**Exit criteria:** every harness README links to the manifesto
on first read. Every harness AGENTS.md has a "general rules"
section that points to the org-level CONTRIBUTING.md and a
"harness-specific rules" section that stays under 200 lines.

### F2 — Enriched Inference inside the agent harness 📅

Ship the first end-to-end IE run with the Cost Gate active.

- [ ] `dogma-v2-core/src/runtime/enriched.rs` — the IE pattern.
- [ ] `dogma-v2-core/src/runtime/cost_gate.rs` — the Cost Gate
      pattern with `Interactive`, `Auto`, `Trusted`, and
      `Webhook` impls.
- [ ] `dogma-v2-core/src/runtime/cost_estimator.rs` — the
      `CostCalculable` trait and per-provider impls.
- [ ] `dogma-v2-core/src/runtime/quality_estimator.rs` —
      `QualityCalculable` trait with heuristic baseline.
- [ ] LLMProvider fan-out and fan-in helpers.
- [ ] `/ei` slash command in the CLI.
- [ ] Session graph: `CostProposal`, `CostDecision`, `CostActual`
      node types.
- [ ] Test fixtures with explicit setup (model, temperature, max
      tokens, seed) so every test is reproducible.
- [ ] Walking Skeleton: at least one end-to-end run with the
      Cost Gate active, persisted in `.vdb`, replayable.

**Exit criteria:** a developer can run `dogma ask --ei "explain
this code"`, see a cost estimate, confirm, and get a result.
Every step is in the session graph.

### F3 — Open benchmark `dogma-arena` 📅

Make the "beyond the frontier" claim falsifiable and public.

- [ ] New repo: `dogmalab/dogma-arena`.
- [ ] 20-30 curated tasks across three categories: dev,
      research, general.
- [ ] `dogma-bench` runner: takes a config + a task suite,
      produces `results.jsonl` with cost, quality, and
      reproducibility metadata.
- [ ] GitHub Pages leaderboard: auto-updated by CI on PR.
- [ ] Submission workflow: every PR must include the config
      used, the cost actually spent, and the quality scores.
- [ ] `TrustedCostGate` for arena submissions: auto-approves
      but logs everything.

**Exit criteria:** a community member can fork the arena, run
the benchmark on their own config, and submit a PR with their
results. The leaderboard updates automatically.

### F4 — Killer demo `dogma-refagent` 📅

A reference agent that exercises every primitive, with a
90-second video that shows the Cost Gate, IE, persistent memory,
and the open benchmark.

- [ ] New repo: `dogmalab/dogma-refagent`.
- [ ] Reference agent in 300-500 LOC of Rust (target).
- [ ] 5-10 example tasks with side-by-side comparisons (single
      LLM vs Enriched Inference).
- [ ] Video walkthrough: from bootstrap to a working agent with
      Cost Gate visible, in 90 seconds.
- [ ] README that links to the MANIFESTO and to dogma-arena.

**Exit criteria:** the video is on YouTube, the README links
to it, and a developer who watches the video can reproduce the
setup in under 30 minutes.

### F5 — Distribution and dogfooding 📅

Turn a great codebase into a movement.

- [ ] Pre-compiled binaries for Linux, macOS, Windows
      (x86_64 + aarch64) on GitHub Releases.
- [ ] Homebrew tap and apt repository.
- [ ] Bootstrap script: `curl ... | sh` brings up the agent,
      which installs the rest.
- [ ] 4 public posts over 60 days.
- [ ] HN submission with the manifesto.
- [ ] Permission-based activation: the LLM can request `/ei`;
      the user approves via the Cost Gate.

**Exit criteria:** a developer with no prior knowledge of Rust,
no prior knowledge of Dogma, and 30 minutes to spare can install
Dogma, run a query, see the Cost Gate, and decide to keep using
it or walk away.

---

## Next (Q4 2026)

Items queued after F5. Order is the order in which they will be
picked up, subject to capacity and incoming feedback.

### F6 — Second harness pattern

Add a second pattern to the agent harness, distinct from IE.
Candidates:

- **Speculative decoding** — a small model proposes, a large
  model verifies, only accepted tokens are kept. High quality,
  low cost.
- **Self-consistency** — N samples, majority vote, with
  configurable threshold. Cheap and reliable for verifiable
  tasks.
- **RAG-Fusion** — multiple query reformulations, retrieval,
  reciprocal rank fusion. Better recall on ambiguous queries.
- **Cost cascade** — try the cheapest model first, escalate to
  the most expensive only if confidence is low. Bounded cost,
  bounded quality.

**Selection criteria:** whichever pattern the community asks for
first, or whichever we have empirical evidence for in
`dogma-arena`.

### F7 — Standard benchmarks

Validate against the public benchmark suites.

- [ ] AlpacaEval 2.0 — with a fixed IE config, run as a baseline.
- [ ] MT-Bench — same.
- [ ] FLASK — same.
- [ ] Publish results in the arena, with full cost reporting.

**Exit criteria:** Dogma-IE (with a published config) is on the
AlpacaEval leaderboard. The submission includes the cost of
generating every response.

### F8 — Cost API as a service

A standalone command that estimates cost without running.

- [ ] `dogma cost-estimate --config x --query y` — returns a
      `CostEstimate` JSON without spending anything.
- [ ] Useful for budgeting, for CI gates, for "is this
      affordable" decisions before committing to a run.

**Exit criteria:** a CI pipeline can call `dogma cost-estimate`
as a step and abort the build if the cost is over budget.

---

## Later (2027)

Items we know we want, but not before F1-F8 are done and we have
feedback from real usage.

- **WASM target for the state harness.** Library-only WASM, not
  the full agent. The agent harness in WASM is a much larger
  project and not on the path.
- **Sneakernet federation.** Two agents on two machines
  synchronize `.vdb` files via USB stick, with a deterministic
  conflict-resolution protocol. Air-gapped multi-agent.
- **Git-like versioning of `.vdb` files.** Commits, branches,
  diffs, cherry-pick of documents. The full history of agent
  state, navigable.
- **TypeScript, Go, Zig SDKs.** Python bindings exist; the rest
  are community-driven.
- **dogma-arena v2: the open leaderboard for harnesses, not
  just IE.** Where contributors submit new patterns and
  configurations, and the community votes on quality.

---

## Not on the roadmap

These are explicit non-goals. Listed here so contributors do not
propose them and have to discover the rejection after writing the
RFC.

| Not doing | Why |
|---|---|
| Cloud-hosted version of any harness | Local-first is the default. Cloud splits the codebase, the community, and the principles. |
| Web frontend | CLI and NDJSON are the frontends. A web UI is a separate project. |
| 200-tool agent framework | Three tools and a loop. The rest is skills installed at runtime. |
| Competing with Pinecone on managed service | The thesis is the opposite of managed services. |
| TS / Go / Zig SDK in core | Python bindings exist. The community builds the rest. |
| WASM target for the agent harness | Library-only WASM for `dogma-vdb` is enough. |
| Online federation of agents | Sneakernet is v1. Online is later, if at all. |
| Git-like versioning of `.vdb` | Replay is v1. Git-like history is later. |
| A "dogma cloud" | Out of scope. The maintainer may spin up a managed offering, separately. |
| Telemetry of any kind | The user owns their data. We do not collect it. |

If you want one of these, you are welcome to build it as a
separate project. We will link to it from the FAQ if it is
compatible with the manifesto. We will not merge it into the
core.

---

## How to influence the roadmap

1. **Open an issue with the `[rfc]` tag.** Use the template in
   [CONTRIBUTING.md](./CONTRIBUTING.md).
2. **If the item is "Later"**, comment on the relevant issue
   or open a new one. Items get pulled forward when there is
   signal (multiple asks, working draft, sponsor).
3. **If the item is "Not on the roadmap"**, the answer is "not
   now, but here is how to file an RFC". The roadmap is a
   direction, not a wall.

---

## See also

- [MANIFESTO.md](./MANIFESTO.md)
- [STRATEGY.md](./STRATEGY.md)
- [CONTRIBUTING.md](./CONTRIBUTING.md)
- [FAQ.md](./FAQ.md)
- [GLOSSARY.md](./GLOSSARY.md)

---

<div align="center">

**The roadmap is a hypothesis. The arena is the experiment.**
**Update the roadmap when the experiment is done.**

</div>

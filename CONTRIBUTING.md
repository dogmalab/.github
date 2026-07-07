# Contributing to Dogma

> Pragmatism beats dogma, even here.

This document is the operational layer below
[MANIFESTO.md](./MANIFESTO.md) and [STRATEGY.md](./STRATEGY.md). The
manifesto is the direction. The strategy is the plan. This is the
"how we work" — how we decide what lands, how we write code, and how
we treat each other.

---

## The Four Questions

Before any new feature, dependency, or major change lands, it must
answer **yes** to all four questions. If it fails one, we do not add
it. If it fails two, we reject it politely. If it passes all four, it
goes in.

### 1. Would an indie AI builder ask for this in their first week?

If the answer is "no" or "maybe in month 6", the feature is not
core. Either it goes in as an opt-in (feature flag, plugin, separate
crate, optional dependency), or it does not go in at all.

The "indie AI builder" is our primary audience. They are solo or in
a small team. They are building an MVP, a side project, or a small
product. They do not have an enterprise procurement process. They do
not have a dedicated SRE. They have a laptop and a credit card.

When in doubt, ask: "Would a developer running `cargo install` on a
Tuesday night be glad this exists by Wednesday morning?"

### 2. Can we do this with the primitives we already have?

If the answer is "yes", we do not add a new abstraction. We document
how to compose the existing primitives to get the new behavior. We
add an example. We do not add a new trait, a new module, or a new
dependency.

The state harness has `Collection`, `Document`, `Embedder`, `Index`,
`VectorStorage`, `Reranker`, `BinStorage`. The agent harness has
`LLMProvider`, `Tool`, `SkillManager`, `SessionManager`,
`Compressor`, `WasmSandbox`. The gateway harness has `Router`,
`AppState`, and the three endpoint handlers.

Most new features are compositions of these. The discipline of
checking first prevents accidental complexity.

### 3. Does it break any of: local-first, zero-config, one-file state?

These three are the inviolable constraints. They are the only rules
that are not negotiable.

- **local-first**: the state harness and the agent harness have no
  network code. Period. The network harness is the only listener.
- **zero-config**: `cargo install dogma-agent` should produce a
  working binary. No `.env` file required. No daemon to start. No
  service to register. The same code path runs in dev, CI, and prod.
- **one-file state**: the complete state of an agent — sessions,
  memory, skills, plans — fits in `.vdb` files that can be copied,
  committed, or sneakernetted. There is no "side database" that the
  agent depends on that is not in the file.

If a feature breaks one of these, it is rejected. Even if the
contributor offers a really good reason. Even if it would be useful.
The promises of the manifesto are the promises of the manifesto.

### 4. Does the harness have a Cost Gate?

If the feature causes the agent to spend resources (LLM tokens,
wall-time, money, network bandwidth) without an explicit human
decision, it is rejected.

Every expensive operation in the agent harness passes through a
Cost Gate. The user sees the estimate. The user decides. The
decision is logged. The default gate is interactive (the user is
prompted). Alternative gates are auto (abort if over budget),
trusted (auto-approve but log), and webhook (external approval).

This is not a feature. This is the architecture.

### When to break a rule

If a feature fails one of the four questions but you believe it
should still land, the path is:

1. Open a GitHub issue with the `[rfc]` tag.
2. Reference this document and name the question it fails.
3. Explain the trade-off and why the rule should bend.
4. Wait for at least one maintainer's explicit "yes, with the
   following conditions".

The maintainer's "yes" goes in the PR description, with the date.
The rule is broken deliberately, with a written reason, in a
discoverable place. The "When we break our own rules" section of
the manifesto is updated.

---

## Code style per harness

Each harness has its own conventions. When in doubt, follow the
existing code in that harness. When starting fresh, follow the
rules below.

### `dogma-vdb` (state harness)

- **Module size**: no file larger than 300 lines. If a module grows
  past that, split it.
- **No async in core**. The state harness is synchronous. Async
  belongs in the agent harness.
- **`#![deny(unsafe_code)]`** at the crate root. The only `unsafe`
  blocks allowed are in `storage/traits.rs` for byte reinterpretation
  between `Vec<f32>` and `&[u8]`. These are documented and isolated.
- **No new dependencies in core** without discussion in an issue.
  Core has 11 dependencies: `serde`, `serde_json`, `thiserror`,
  `regex-lite`, `rayon`, `wide`, `bytemuck`, `memmap2`, `once_cell`,
  `toml`, `log`. Adding any of these to core requires a strong case.
- **JSONL is the source of truth**. The binary v2 mmap format is a
  cache. Every `Collection` can be re-exported to JSONL and re-read.
- **Tests use real `.vdb` files** in `tests/fixtures/`. The
  benchmark set is in `benchmarks/`.
- **English only** in code comments. The user-facing `RCA_GUIDE.md`
  and the historical Spanish comments in older modules are
  exceptions, not the rule.

### `dogma-agent` (agent harness)

- **`#![deny(unsafe_code)]`** at the crate root. No exceptions.
- **No `unwrap()` in production code paths**. Tests are allowed to
  panic; production code returns `Result`.
- **`parking_lot::RwLock` over `std::sync::RwLock`**. Reason:
  poisoning immunity. The standard library's `RwLock` poisons
  itself on panic; `parking_lot` does not. For an agent that runs
  for hours, this matters.
- **No macros, no generics in the core loop**. The reasoning loop
  is the hot path; indirection there is paid every iteration.
- **Async is allowed**. The agent harness talks to LLMs over HTTP
  and runs WASM sandboxes. Both are async.
- **NDJSON is the wire format**. Every event the agent emits is one
  line of JSON, dual-output (human-readable to stderr, structured
  to stdout with `--json`).
- **Three core tools + skills**. `read_file`, `write_file`,
  `execute_script`. Skills are installed at runtime via
  `SkillManager`. The "72 tools of v1" era is over.
- **The Cost Gate is mandatory**. Every pattern that uses LLM
  tokens, WASM execution, or skill installation passes through
  `CostGate::ask`. No silent execution.

### `dogma-gateway` (network harness)

- **`#![deny(unsafe_code)]`** at the crate root. No exceptions.
- **No new dependencies that are not transitively required by
  `axum`**. The gateway is the network boundary. Every dependency
  is a potential attack surface. `deny.toml` audits advisories on
  every PR.
- **Every ingress type uses `#[serde(deny_unknown_fields)]`**.
  Unrecognized JSON fields are a 400, not a silent ignore.
- **No outbound network calls** in handlers. The gateway is a
  pure routing and validation layer. It does not call OpenAI; it
  pipes to `dogma-agent` which calls OpenAI.
- **No local session files**. The gateway is stateless. All
  state lives in `dogma-vdb` files, accessed via mmap.
- **The release profile is `opt-level = "z"`, `lto = true`,
  `strip = true`**. The binary must be small enough to ship
  comfortably in an Alpine container.

---

## How to propose a feature

1. **Check the [ROADMAP.md](./ROADMAP.md) first.** If the feature
   is already planned, comment on the relevant issue or PR. If it
   is listed under "Later", open an issue and link to it.

2. **If it is not in the roadmap**, open a GitHub issue with the
   `[rfc]` tag. Use this template:

   ```markdown
   ## Summary
   One paragraph. What is the feature and why is it needed?

   ## The Four Questions
   - Q1 (indie in first week?): [yes/no, and why]
   - Q2 (primitives we already have?): [yes/no, and what would compose]
   - Q3 (breaks local-first/zero-config/one-file?): [no/yes, and which]
   - Q4 (has Cost Gate?): [yes/no, and how]

   ## Alternatives considered
   What other approaches did you consider? Why is this the best one?

   ## Cost
   What does this cost in terms of dependencies, code size, maintenance?

   ## Open questions
   What are you not sure about? What do you need feedback on?
   ```

3. **Wait for at least one maintainer's "yes" before opening a PR**.
   The "yes" can be a thumbs-up comment or a more detailed
   response. RFCs without explicit "yes" get closed after 30 days
   of inactivity.

4. **Once approved, open a PR**. Reference the RFC issue in the
   PR description. Include the maintainer's "yes" verbatim.

---

## How to file a bug

Use the GitHub issue tracker for the affected harness. Be specific:

- **What you did** (commands, code, config)
- **What you expected**
- **What happened instead** (with the actual output, including
  NDJSON if relevant)
- **Environment** (OS, Rust version, model used, embedder used,
  `.vdb` schema version)

A reproducible bug is a fixable bug. An irreproducible bug is a
feature request for better error messages.

---

## How to file a security issue

Do not open a public issue. Email the maintainer directly (see the
GitHub profile in the org landing page). Dogma is a security-first
project: the architecture is designed so that vulnerabilities in the
state harness and the agent harness have a limited blast radius.
That promise only holds if issues are reported and fixed quickly.

---

## Development setup

### Prerequisites

- Rust 1.85 or later (some harnesses use edition 2024)
- `cargo`, `rustfmt`, `clippy`
- For agent harness: an OpenAI API key, or an Ollama instance
  running locally
- For state harness: nothing additional
- For network harness: nothing additional

### First build

```bash
# Clone the three harnesses
git clone https://github.com/dogmalab/dogma-vdb.git
git clone https://github.com/dogmalab/dogma-agent.git
git clone https://github.com/dogmalab/dogma-gateway.git

# Build the state harness
cd dogma-vdb && cargo build --workspace && cargo test --workspace

# Build the agent harness (path deps to dogma-vdb)
cd ../dogma-agent && cargo build --workspace && cargo test --workspace

# Build the network harness
cd ../dogma-gateway && cargo build --release
```

### Pre-commit checks

Before opening a PR, run the same checks that CI runs:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets --workspace -- -D warnings
cargo test --workspace
cargo deny check
cargo audit
```

If any of these fail on your machine, they will fail in CI. Fix
locally first.

---

## See also

- [MANIFESTO.md](./MANIFESTO.md) — the direction and the Eight
  Beliefs.
- [STRATEGY.md](./STRATEGY.md) — the Five Plays for the next 60
  days.
- [ROADMAP.md](./ROADMAP.md) — what is next, in order.
- [FAQ.md](./FAQ.md) — versus LangChain, MemGPT, Pinecone, and
  others.
- [GLOSSARY.md](./GLOSSARY.md) — vocabulary.

---

<div align="center">

**Code is read more often than it is written.**
**Comments should explain why, not what.**
**Pragmatism beats dogma, even here.**

</div>

# Dogma Glossary

> The vocabulary of the platform.

When a term is used in the manifesto, the strategy, or the docs
and is not in everyday English, it is defined here. The glossary
is the source of truth for the words. When the code disagrees
with the glossary, the code is right and the glossary is updated.
When the docs disagree with the code, the docs are updated.

---

## Top-level concepts

### Harness

A runtime that orchestrates one or more concerns to produce a
useful result. The Dogma platform contains three harnesses:

- **state harness** (`dogma-vdb`) — manages persistent vector
  state.
- **agent harness** (`dogma-agent`) — orchestrates LLMs, tools,
  skills, and state.
- **network harness** (`dogma-gateway`) — bridges external
  clients to the air-gapped core.

A harness is not a framework. Frameworks host your code;
harnesses compose with it. When you stop using a framework, you
throw away the code. When you stop using a harness, the data
files you wrote still work.

### Pattern

A specific way of orchestrating within a harness. The agent
harness contains:

- **RuntimeLoop** — single LLM with a tool-calling loop.
- **Enriched Inference (IE)** — N LLMs in parallel, compiled
  iteratively.
- **Cost Gate** — wrapper that requires human confirmation of
  the cost estimate before any expensive operation.

Patterns compose with primitives; they are not separate
harnesses.

### Primitive

The smallest reusable building block. In the state harness:
`Collection`, `Document`, `Embedder`, `Index`, `VectorStorage`,
`Reranker`, `BinStorage`. In the agent harness: `LLMProvider`,
`Tool`, `SkillManager`, `SessionManager`, `Compressor`,
`WasmSandbox`. In the network harness: `Router`, `AppState`,
the endpoint handlers.

Primitives do not depend on patterns. Patterns compose
primitives. The "Four Questions" in CONTRIBUTING.md ask
contributors to check whether a new feature can be expressed as
a composition of existing primitives before adding new ones.

### Feature

A capability of a harness, exposed to the user. The Cost Gate
is a feature. RAG is a feature. Skills are a feature.
Enriched Inference is both a pattern and a feature (it has a
user-facing API and an internal composition).

### Platform

The Dogma organization itself: the three harnesses, the
meta-repository (`.github`), the open benchmark (`dogma-arena`),
the reference agent (`dogma-refagent`), and the documentation
that ties them together.

---

## The state harness (`dogma-vdb`)

### `.vdb` file

A single file that contains a `Collection`: a JSONL source of
truth plus an optional binary v2 mmap cache. The file is the
database. You can `cat` the JSONL, `grep` it, `jq` it, copy it,
commit it, scp it. There is no server, no daemon, no
connection string.

### `Collection`

The central type of the state harness. A `Collection` is an
ordered set of `Document`s with an `Index` and a `VectorStorage`.
Created with `Collection::open(path)` or
`Collection::open_with(path, index_type, metric)`.

### `Document`

The unit of data in a `Collection`. A `Document` has an `id`, a
`text`, an `embedding: Vec<f32>`, and a
`metadata: HashMap<String, String>`. Documents are the only
thing the state harness stores.

### `Embedder`

A trait that converts text to `Vec<f32>`. The default impl in
`dogma-vdb-embed` is a zero-dependency stub. The
`dogma-vdb-embed-fastembed` crate provides an ONNX-based impl
with the MiniLM-L6-v2 model (384 dimensions).

### `Index`

A trait for vector search. The state harness ships with three
impls:

- **BruteForce** — exact, O(n·d) per query, full recall.
- **HNSW** — approximate graph, O(log n) per query, high recall.
- **IVF-PQ** — inverted file plus product quantization, O(√n)
  per query, 8× less memory than HNSW, lower recall.

### `VectorStorage`

A trait that abstracts where the vectors live. Two impls:

- `MemoryBackedStorage` — vectors in RAM. Fast to build, lost
  on process exit.
- `MmapBackedStorage` — vectors memory-mapped from the binary
  v2 file. ~0ms cold start, scales to disk.

### `Reranker`

A trait for two-stage retrieval. The first stage retrieves
candidates with the `Index`; the second stage reranks them with
the `Reranker`. The `dogma-vdb-rerank` crate provides
Cross-Encoder rerankers.

### `BinStorage`

The persistence layer. Writes both JSONL (source of truth) and
binary v2 (mmap cache). On read, prefers the binary v2 cache
when present; falls back to JSONL when not.

### Binary v2 mmap format

The on-disk binary representation of a `Collection` after the
JSONL is read. Magic header "DVDB" plus metadata plus a
32-byte-aligned flat `f32` section. ~2.3× smaller and ~7×
faster than the JSONL alone. See
[`dogma-vdb/SPEC.md`](https://github.com/dogmalab/dogma-vdb/blob/main/SPEC.md)
for the full spec.

### SIMIL

**S**ystem **I**ntent **M**arkup **L**anguage. A small DSL
embedded in `Document.metadata["sml"]` that lets the agent
express intent, attributes, relations, and flow control over
the data without leaving the state harness. The full spec is
in
[`dogma-vdb/SPEC.md`](https://github.com/dogmalab/dogma-vdb/blob/main/SPEC.md)
section 23.

### `StorageStrategy`

How a `Document` is stored relative to its SIMIL manifest.
Two values:

- `Hybrid` — keeps both the original `text` and the
  `metadata["sml"]` manifest.
- `SymbolicPure` — clears the `text` after SIMIL extraction.
  Smaller, more secure, but no human-readable string in the
  document.

---

## The agent harness (`dogma-agent`)

### `RuntimeLoop`

The default pattern. A single LLM is invoked, the response may
include tool calls, the tools are invoked, the results are fed
back to the LLM, repeat until the LLM produces a final answer
or the iteration cap is hit.

### `Enriched Inference` (IE)

The flagship pattern. N LLMs run in parallel as proposers. A
compiler model synthesizes their responses. The compilation is
iterated, with each proposer refining its response based on
the synthesis (not on the other proposers' responses — this
avoids groupthink). The final answer is the compiler's last
synthesis.

### `Cost Gate`

A pattern that wraps any expensive operation. The user is
shown an estimate (USD, tokens, wall-time) and a proposed
configuration, plus alternatives. The user picks one: proceed,
modify, or abort. The decision is logged in the session
graph. Implementations:

- `InteractiveCostGate` (default) — prompt in the CLI.
- `AutoCostGate` — auto-approve if below a budget; abort
  otherwise.
- `TrustedCostGate` — auto-approve, log everything. Used by
  the arena and CI.
- `WebhookCostGate` — external approval endpoint. For
  enterprise.

### `LLMProvider`

A trait for talking to an LLM. Implementations: `OpenAiProvider`
(real), `OllamaProvider` (planned), `AnthropicProvider` (planned),
`OpenRouterProvider` (planned). IE uses the trait to fan out
to N proposers and fan in to the compiler.

### `Tool`

A trait for an action the agent can take. The agent harness
ships with three core tools: `read_file`, `write_file`,
`execute_script`. Additional tools are installed at runtime
via the `SkillManager`.

### `SkillManager`

A registry of skills, loaded from `~/.dogma/skills/`. Each
skill is a directory containing a manifest and (optionally) a
WASM module. Skills can be installed from
[skills.sh](https://skills.sh) (the community skill index) or
from a local path.

### `SessionManager`

The gateway from the agent harness to the state harness.
`SessionManager::open(path)` creates a `Collection` in
`sessions.vdb`; every message, tool result, plan step, and
cost decision is a `Document` in that collection. The
collection is the agent's persistent state.

### `Compressor`

Reduces the size of context before sending it to the LLM.
Two strategies: deterministic (rule-based) and semantic
(embedding-based). The semantic strategy was a placeholder
and is the next thing to be wired up (F2).

### `WasmSandbox`

A safe execution environment for `execute_script`. Uses
`wasmtime` plus `wasmtime-wasi`. Scripts that the agent
runs are compiled to WASM, executed in the sandbox, and
their output is captured. The agent can use this to run
tests, validate code, query the local environment.

### `UserMemory`

A separate `Collection` (in `user_memory.vdb`) that stores
key-value preferences. The agent reads from it at the start
of every session and writes to it when the user makes a
preference explicit.

### NDJSON

**N**ewline-**D**elimited **JSON**. The wire format of the
agent harness. Every event is one line of JSON, written to
stdout. The dual-output mode writes human-readable lines to
stderr and structured JSON to stdout. The gateway consumes
the structured stream and emits SSE.

### RSI

**R**eason / **S**hell / **I**nsight. The conceptual loop the
agent runs. Reason: think about what to do next. Shell:
execute a tool. Insight: integrate the result and decide the
next iteration. The pattern is named after the architecture
of the agent, not after a specific implementation.

---

## The network harness (`dogma-gateway`)

### IPC pipes

The communication channel between the gateway harness and the
agent harness. The gateway spawns the agent as a child
process, captures its stdout, and pipes requests in via
stdin. The agent never opens a socket. The gateway never
opens a `dogma-vdb` file directly through the Rust API
(only through mmap, in a future release).

### SSE

**S**erver-**S**ent **E**vents. The wire format from the
gateway harness to external HTTP clients. The agent harness
emits NDJSON on stdout; the gateway reads the NDJSON, wraps
each line in an SSE frame, and writes it to the HTTP
response. The client sees a stream of events.

### mmap

**M**emory-**m**apped I/O. The gateway harness will (in F2)
read `.vdb` files via mmap, not via the state harness's Rust
API. The reason is to keep the gateway dependency-light: it
should not pull in `memmap2` and friends directly. A
separate adapter crate will encapsulate the mmap access and
expose a small API to the gateway.

### `AppState`

The shared state of the gateway harness. Today, a
`tokio::sync::broadcast::Sender<AgentEvent>`. In F2, the
`AppState` will also hold the mmap adapter to the state
harness.

---

## Concepts that cross harnesses

### Session graph

The graph of `Document`s that constitutes one agent session.
Edges are encoded in `metadata` strings: `node_type`,
`edge_type`, `session_id`, `sequence`, `parent_id`. The graph
is stored in `sessions.vdb`. The full session is replayable
from the graph alone — the agent harness can re-execute a
session deterministically given the same `LLMProvider`
configuration.

### Replay

The ability to re-execute a session. Given a `session_id` and
the `LLMProvider` configuration that produced the original
session, the agent harness can re-run the reasoning loop and
compare its output to the recorded `Document`s. Used for
debugging, for benchmarking, and for verifying the harness.

### `vet`

The CLI command for inspecting a `.vdb` file. `dogma vet
sessions.vdb` shows the session graph, the cost breakdown,
the proposer-by-proposer responses, and any other structure
the file contains. The output is text, suitable for piping
to `less` or `grep`.

### `dogma-arena`

The open benchmark. A separate repository
(`dogmalab/dogma-arena`) with curated tasks, a runner, and
a public leaderboard. The arena is where the "beyond the
frontier" claim is tested. Every submission includes the
config used, the cost spent, and the quality scores. The
leaderboard is auto-updated by CI.

### `dogma-refagent`

The reference agent. A separate repository
(`dogmalab/dogma-refagent`) that exercises every primitive
of the agent harness in a small, readable codebase. The
target is 300-500 lines of Rust. The reference agent is the
example others copy when they build their own agent.

---

## Acronyms

- **IE** — Enriched Inference.
- **HNSW** — Hierarchical Navigable Small World (graph-based
  ANN index).
- **IVF-PQ** — Inverted File with Product Quantization
  (vector compression index).
- **SQ** — Scalar Quantization (i8 compression).
- **MCP** — Model Context Protocol (the standard for tool
  use in AI agents).
- **NDJSON** — Newline-Delimited JSON.
- **RSI** — Reason / Shell / Insight (the conceptual loop
  the agent runs).
- **SIMIL** — System Intent Markup Language.
- **SSE** — Server-Sent Events.
- **IPC** — Inter-Process Communication.
- **mmap** — Memory-mapped I/O.

---

## See also

- [MANIFESTO.md](./MANIFESTO.md)
- [STRATEGY.md](./STRATEGY.md)
- [CONTRIBUTING.md](./CONTRIBUTING.md)
- [FAQ.md](./FAQ.md)
- [ROADMAP.md](./ROADMAP.md)

---

<div align="center">

**If a word is missing, open an issue.**
**The glossary is the source of truth for vocabulary.**

</div>

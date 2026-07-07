# Frequently Asked Questions

> Honest answers to the questions people actually ask.

This document evolves. When a question comes up three times in issues
or in chat, it gets added here. The goal is to have an answer for
every reasonable question before someone needs to ask it.

---

## General

### What is Dogma?

Dogma is a platform of three harnesses for AI builders:

- **dogma-vdb** — the state harness. A vector database in a file.
- **dogma-agent** — the agent harness. An AI runtime in one binary.
- **dogma-gateway** — the network harness. The only network listener.

Each harness is a self-contained Rust project. The state harness and
the agent harness are air-gapped (no network code). The network
harness bridges external clients to the air-gapped core via IPC
pipes and mmap. Read the [MANIFESTO.md](./MANIFESTO.md) for the
full picture.

### What does "AI-Native" mean?

It means designed for AI from the first commit, not retrofitted
from a tool that was built for humans. Specifically: the data model
is a graph of documents with embeddings, not rows in a table. The
network is air-gapped by default, because agents do not need
network access. The debug surface is NDJSON, because agents produce
streams of events, not interactive UIs. The cost is calculated
before the run, because AI spend is the new database cost.

### Who is Dogma for?

Solo developers, side-project hackers, small teams building AI
products. We optimize for the person who runs `cargo install` on a
Tuesday night and ships a working agent by Wednesday. Enterprises
are welcome but not the target. The full audience analysis is in
[STRATEGY.md](./STRATEGY.md).

---

## vs the alternatives

### How is Dogma different from LangChain?

LangChain is a Python framework with 200+ integrations, abstractions
stacked on abstractions, and a release cycle that breaks
downstream code every few months. Dogma is a Rust platform of three
harnesses with three core tools and a loop. We do not have 200
integrations. We have three tools. We do not stack abstractions; we
compose primitives. We do not break downstream code; we do not have
downstream code, because we are harnesses, not a framework.

| Dimension | Dogma | LangChain |
|---|---|---|
| Language | Rust (Python bindings) | Python only |
| Number of harnesses | 3 (vdb, agent, gateway) | 1 (framework) |
| Core tools | 3 + skills | 100+ |
| Network | Air-gapped core, gateway boundary | Networked by default |
| Cost visibility | Calculable, gated | After the fact |
| Dependencies | 11 core, 8 agent, gateway minimal | Hundreds |
| Failure mode | Crashes, not breaks | Deprecation warnings |

### How is Dogma different from MemGPT or Letta?

MemGPT and Letta virtualize agent memory like an operating system
virtualizes RAM: paged, hierarchical, and opaque. Their state lives
in a server you cannot inspect without their tools. Dogma gives you
a file. The file is the memory. You can `cat` it, `grep` it,
`commit` it to git, and `scp` it to a USB drive. We do not believe
memory needs a kernel. Memory needs a file format.

| Dimension | Dogma | MemGPT / Letta |
|---|---|---|
| Memory model | Graph of documents in `.vdb` | Virtualized pages, hierarchical |
| Storage | One file, no server | Server required |
| Inspectable | `dogma vet`, `jq`, text editor | Their tools only |
| Replay | Deterministic (session graph) | Possible but proprietary |
| Local-first | Yes | Partial |
| Cost | $0 with local models | Cloud-only |

### How is Dogma different from Pinecone, Chroma, or LanceDB?

Pinecone is a managed cloud service. Chroma is a Python library
with a server. LanceDB is a columnar store on top of Arrow. Dogma is
a Rust library with a file format. There is no account to create,
no quota to hit, no connection string to manage. The file is the
service. The dependency is the protocol. The benchmark is `cat`.

| Dimension | Dogma | Pinecone | Chroma | LanceDB |
|---|---|---|---|---|
| Deployment | Embedded | Cloud | Server or local | Embedded |
| Format | JSONL + bin v2 (mmap) | Proprietary | SQLite + Parquet | Lance columnar |
| Dependencies | 11 core | n/a (service) | ~200 | ~150 |
| Cold start | ~0ms (mmap) | Network | Seconds | ~100ms |
| Native MCP | ✅ | ❌ | ❌ | ❌ |
| Reranking | ✅ integrated | ❌ | ❌ | ❌ |

### How is Dogma different from SQLite + pgvector?

SQLite is a relational database. pgvector is a vector extension to
PostgreSQL. Both are excellent at what they do. Dogma is neither.
The agent harness models its state as a graph of documents in a
`.vdb` file, with edges encoded in metadata. There are no tables,
no schemas, no migrations. Documents are added, queried by vector
similarity and metadata predicates, and updated by ID. If your
data fits in a relational model with vectors bolted on, SQLite +
pgvector is the right choice. If your data is "a graph of chunks
of code, with embeddings, that the agent reasons over", `.vdb` is
the right choice.

### How is Dogma different from Ollama, vLLM, or LM Studio?

Those are LLM runtimes. They serve a model. Dogma orchestrates
models. We are not in the same category. The state harness
(`dogma-vdb`) is a vector database; it does not run LLMs. The agent
harness (`dogma-agent`) uses LLMProvider implementations to talk
to Ollama, vLLM, OpenAI, Anthropic, or any compatible endpoint.
The Enriched Inference pattern composes three of those endpoints
into one. We are the orchestration layer, not the runtime layer.

---

## Architecture

### Why three harnesses, not one?

Because the failure modes are different. The state harness fails
when the file corrupts. The agent harness fails when the LLM
hallucinates or the loop diverges. The network harness fails when
the network is hostile. By separating them, we can have a strict
security boundary at the network layer (the gateway is the only
listener), a strict correctness boundary at the state layer (the
file is the truth), and a strict capability boundary at the agent
layer (no network, no external dependencies). Combining them
would require weakening at least one boundary.

### Why Rust, not Python?

Five reasons:

1. **Single binary deployment.** `cargo install` produces one
   statically linkable file. Python requires a virtual environment
   or a container.
2. **No GC pauses.** Agents reason in long loops. A 200ms GC
   pause in the middle of a 10-second reasoning call is
   noticeable.
3. **Dependency minimalism is enforceable.** `deny.toml` is a
   policy file. `pip freeze` is a list of regrets.
4. **Memory safety for file parsing.** The state harness parses
   user-controlled files. Rust makes the parser memory-safe by
   construction.
5. **Async without runtime bloat.** `tokio` is a single
   dependency. `asyncio` requires an event loop and a stdlib that
   changes semantics every release.

### Why no async in the state harness (`dogma-vdb`)?

Because the state harness is a library. The user calls it from
their code. If their code is sync, the state harness blocks. If
their code is async, the state harness waits. Either way, adding
async to the state harness would force every caller to think about
async. The state harness is a database. SQLite is sync. RocksDB
is sync. The state harness is sync. The agent harness is async
because it talks to LLMs, which is a network operation that
benefits from concurrency. The boundary is at the right place.

### Why JSONL plus a binary v2 mmap format?

JSONL is debuggable. You can `cat` it. You can `grep` it. You can
`jq` it. The binary v2 mmap format is fast. You can `mmap` it. The
state harness writes both. JSONL is the source of truth. The
binary file is a cache. Every `Collection` can be re-exported to
JSONL. Every `Collection` can be loaded from JSONL. The two are
synchronized on every write. This is the "JSONL/Binary duality"
that the manifesto references.

### What is SIMIL?

SIMIL is the System Intent Markup Language. It is a small DSL
embedded in `Document.metadata["sml"]` that lets the agent express
intent, attributes, relations, and flow control over the data
without leaving the state harness. SIMIL is not a programming
language; it is a metadata grammar. It is the layer that lets the
agent reason about its own state at a higher level than raw
strings. The full spec is in
[`dogma-vdb/SPEC.md`](https://github.com/dogmalab/dogma-vdb/blob/main/SPEC.md).

---

## Enriched Inference (IE)

### What is Enriched Inference?

Enriched Inference is a pattern inside the agent harness. It
runs N LLMs in parallel (the "proposers"), synthesizes their
responses with a compiler model, iterates the synthesis N times,
and produces a final answer. The pattern is gated by the Cost
Gate, so the user always sees the estimated cost and confirms
before the run.

### Why is it called "beyond the frontier"?

Because N well-chosen tier-B models, composed with a compiler and
iterated, can match or exceed any single tier-A model on most
tasks. This is not speculation. It is the result of the
Mixture-of-Agents research (Together AI, 2024) and will be
validated in our open benchmark at
[dogmalab/dogma-arena](https://github.com/dogmalab/dogma-arena).
The "beyond the frontier" claim is falsifiable, and we will
publish the falsification if it happens.

### Can I use IE with any combination of models?

Yes. Local, cloud, or mixed. Three Llama 70B models running on
your laptop will give you a different quality profile than three
frontier APIs running on cloud infrastructure. The harness
compounds whatever you give it. Better models = better results,
at any cost the user accepts. The default config is up to the
user; the harness does not impose one.

### How does the Cost Gate work?

When IE is invoked (via the `/ei` slash command in the CLI, or
via the API in the gateway), the harness calculates the estimated
cost in USD, tokens, and wall-time, and asks the user to
confirm. The user sees the proposed configuration (which models,
how many iterations) and a list of alternatives (cheaper, more
expensive, faster, slower). The user picks one. The decision is
logged in the session graph. The harness proceeds.

### Can the agent invoke IE autonomously?

Only if you grant it permission. By default, IE requires human
confirmation. The plan is to add a permission-based activation
flow (F6) where the LLM can request IE, but the Cost Gate still
requires a human decision. No silent execution. No surprise
bills.

### Will it ask me before every run?

Yes, by default. The Cost Gate is interactive by default. You can
configure it to:
- Always ask (the default)
- Auto-approve if cost is below X (headless mode)
- Send to a webhook for external approval (enterprise mode)
- Always approve (trusted mode, for the arena and CI)

Every "auto-approve" is still logged in the session graph for
audit. The cost is calculable; the decision is human.

---

## Deployment and operations

### How do I install Dogma?

```bash
# The state harness
cargo install dogma-vdb

# The agent harness
cargo install dogma-agent

# The network harness
cargo install dogma-gateway
```

Pre-compiled binaries for Linux, macOS, and Windows (x86_64 and
aarch64) are published on the GitHub Releases pages of each
harness. Homebrew and apt repositories are planned (F5).

### Do I need a GPU?

No. The state harness is pure CPU. The agent harness can run with
any LLM endpoint — local Ollama, remote API, or self-hosted
vLLM. Running three local models in parallel benefits from GPU
memory but is not required. On a modern multi-core CPU, the
agent harness can run three 7B models comfortably.

### How do I back up agent state?

Copy the `.vdb` files. That is the entire backup. The state
harness writes to one directory (`~/.dogma/` by default); backing
up that directory is the backup. Restoring is `cp -r`. There is
no incremental backup, no replication, no cluster. There is one
file per collection, and the directory is the truth.

### Can I run Dogma in production?

The state harness is at v1.0 beta. It is production-usable with
caveats: every release of the binary format is documented and
the JSONL is the source of truth. The agent harness is at v0.x
and the API may change. The network harness is at v0.1 with
stubs; the real backends are in F2. The open benchmark
(dogma-arena) is the right place to test before committing to a
production deployment.

### Can I use Dogma with my existing tools?

The state harness has a Python binding (`dogma-vdb-python`) and
an MCP server (`dogma-vdb-mcp`). The agent harness speaks NDJSON
on stdout, which is consumable by any tool that can read a line
of JSON. The network harness speaks HTTP + SSE, which is
consumable by any HTTP client. We do not have SDKs in
TypeScript, Go, or Zig yet; the community is welcome to build
them on top of the JSONL format.

---

## Licensing and community

### What license is Dogma under?

MIT for the Rust crates. The Python bindings and the MCP server
are MIT. The documentation in this repository is MIT. Logos and
the wordmark are not covered by the MIT license; please contact
the maintainer for use of the brand.

### Can I use Dogma commercially?

Yes. MIT is permissive. You can use it, modify it, sell it, ship
it in your product. We ask for attribution and a link back to
the project, but we do not enforce it.

### How is the project governed?

There is one maintainer (Argimiro Gil / @arggil) and a small
group of contributors. Decisions are made by the maintainer after
discussion in GitHub issues. The `[rfc]` tag is the entry point
for substantial changes. The Three Questions in
[CONTRIBUTING.md](./CONTRIBUTING.md) are the rule.

### Where can I get help?

- GitHub issues in the relevant harness repo
- The `[rfc]` tag for substantial proposals
- Email for security issues (see the maintainer's GitHub profile)

We do not have a Discord, a Slack, or a forum. GitHub is the
single place where the conversation happens. This is by design:
fragmented conversations are the enemy of clarity.

---

## See also

- [MANIFESTO.md](./MANIFESTO.md)
- [STRATEGY.md](./STRATEGY.md)
- [CONTRIBUTING.md](./CONTRIBUTING.md)
- [ROADMAP.md](./ROADMAP.md)
- [GLOSSARY.md](./GLOSSARY.md)

---

<div align="center">

**If your question is not here, open an issue. If it comes up
three times, it will be.**

</div>

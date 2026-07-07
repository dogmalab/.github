# The Dogma Manifesto

> A platform of harnesses that takes transformer LLMs beyond the frontier,
> with a cost that is affordable and calculable.

---

## The thesis

For decades, AI capability has been a function of model size.

Dogma inverts this: capability becomes a function of how you orchestrate
models, not which one you pick. A platform of harnesses — composable
patterns that combine multiple LLMs, RAG, skills, and tools — produces
results that exceed what any single frontier model can do, predictably
and within budget.

We are not building a better model. We are building the harnesses that
make models better than they thought they could be.

---

## What Dogma is

Dogma is a platform of three harnesses:

| Harness | Role | Network |
|---|---|---|
| **dogma-vdb** | State harness. Manages persistent vector state, embeddings, indices, and metadata graphs. | None. One file. |
| **dogma-agent** | Agent harness. Orchestrates LLMs, tools, skills, and state into reasoning loops. | None. One binary. |
| **dogma-gateway** | Network harness. The only network listener. Bridges HTTP clients to the air-gapped core via IPC pipes and mmap. | The only one. |

Within the **agent harness**, several patterns exist:
- **RuntimeLoop** — single LLM with a tool-calling loop.
- **Enriched Inference** — N LLMs in parallel, compiled iteratively. The flagship pattern.
- **Cost Gate** — every expensive operation requires human confirmation of the cost estimate before proceeding.

The combination produces results that exceed any single frontier model,
with a cost that the user calculates and confirms before spending.

---

## What we believe

### 1. An agent without persistent state is a chatbot with a loop.

If your agent cannot remember yesterday, it is not an agent. State is not
optional. State is the substrate of intelligence. Make it a file. Make it
`cat`-able. Make it outlive your process.

### 2. The best database is a file you can `cat`.

`dogma-vdb` writes JSONL plus a binary v2 mmap. You can `grep` it, `jq` it,
`awk` it, commit it to git, scp it to a USB drive. No server. No daemon.
No connection string. If you need a `psql` prompt to inspect your agent's
memory, something has already gone wrong.

### 3. The best runtime is no runtime.

`dogma-agent` ships as one binary. No container. No Kubernetes. No
`docker-compose`. No systemd unit. `cp` it, run it, done. The OS is the
runtime. The process is the daemon. The file is the database.

### 4. The best network is no network.

`dogma-agent` and `dogma-vdb` have **zero** network code. No HTTP client.
No socket listener. The only network-facing component is `dogma-gateway`,
which exists exclusively to bridge external clients to the air-gapped
core. This is not a feature. This is the architecture.

### 5. The best framework is three tools and a loop.

The agent harness ships with three core tools — `read_file`, `write_file`,
`execute_script` — plus a reasoning loop and a state backend. Everything
else (RAG, skills, multi-LLM, enriched inference) is a composition of
these primitives. We replace 72-tool frameworks with three tools and a
loop. Complexity does not scale. Primitives do.

### 6. The best debug experience is grep on JSONL.

Every event in the agent harness is a line of NDJSON: queryable with
`jq`, scriptable with `awk`, streamable over pipes. The session graph
lives in a `.vdb` file that you can inspect with `dogma vet` or a
text editor. There is no opaque log. There is no proprietary format.
There is no vendor lock-in.

### 7. Local-first is not a feature. It is a constraint that produces better software.

When your data cannot leave the machine, you stop adding telemetry,
shadow APIs, and hidden cloud dependencies. When the cost is yours, you
stop spawning unbounded loops. When the process is yours, you stop
assuming the network is reliable. Local-first is not nostalgia. It is
discipline.

### 8. AI should ask before it spends.

Every harness invocation in the agent harness passes through a **Cost
Gate**. The user sees the estimate — in USD, in tokens, in wall-time —
plus the proposed configuration and its alternatives. Then the user
decides: proceed, modify, or abort. No silent execution. No surprise
bills. No autonomous spending without explicit human permission. This
is what "calculable cost" means in practice. It is not a feature. It
is the only way we know how to ship.

---

## What we are NOT

### 1. We are NOT LangChain.

LangChain is a 200+ integration framework with abstractions stacked on
abstractions. It breaks when its dependencies break. We are primitives,
not a framework. We ship three tools and a loop. Your code that uses our
primitives will outlive our repo.

### 2. We are NOT MemGPT or Letta.

MemGPT virtualizes agent memory like an operating system virtualizes RAM:
paged, hierarchical, opaque. We do not virtualize memory. We give you a
file. The file is the memory. `cat` it. `grep` it. `commit` it.

### 3. We are NOT Pinecone or any vector DB service.

Pinecone runs as a cloud service. We do not. We are a library plus a
file format. There is no account to create, no quota to hit, no
shutdown to fear. The file is the service.

### 4. We are NOT a framework.

We are harnesses. A framework is something your code is hosted inside.
A harness is something you compose with. The difference matters: when
you stop using a framework, you throw away the code. When you stop
using a harness, the `.vdb` files you wrote still work in ten years.

### 5. We are NOT enterprise-first.

We are indie-first. We optimize for the solo developer, the side
project, the weekend hack, the small team that ships. Enterprise
adoption is welcome but not the target. The target is the person who
runs `cargo install dogma-agent` on a Tuesday night and has a working
agent by Wednesday morning.

---

## When we break our own rules

The rules above are not laws. They are defaults. Sometimes the right
choice is to break one. When we do, we document it. Here are the
exceptions that exist or are being considered:

### Exceptions in place

- **Async runtime in `dogma-agent`.** The state harness (`dogma-vdb`)
  is synchronous, but the agent harness is async. Reason: talking to
  LLMs over HTTP is not zero-runtime, but it is zero-blocking. A
  synchronous agent would freeze on every token. Trade-off accepted:
  the agent harness gains `tokio`; the state harness does not.
  See `dogma-agent/AGENTS.md` for the full rationale.

- **`fastembed` as an optional dependency in `dogma-vdb`.** The state
  harness core has 11 dependencies, all minimal. The ONNX-based
  embedder is opt-in (`dogma-vdb-embed-fastembed`) and pulls in
  `ort`, `tokenizers`, and the model itself. Reason: users asked for
  a zero-config embedder, and the cost of pulling it in by default
  is too high. Trade-off accepted: zero-config costs zero deps.

- **NDJSON streaming from the agent harness to the gateway.** The
  agent harness is "no network", but it writes to stdout in NDJSON
  format. The gateway harness reads that stdout via IPC pipes. Reason:
  the agent does not initiate a connection; the gateway connects
  to the agent's pipes. This preserves the "no listener on the
  agent" invariant.

### Rejected proposals (the rule held)

- **A server mode in `dogma-vdb`.** A long-running HTTP server
  fronting the state harness. Rejected. Breaks the "one file, no
  daemon" promise. Users who need a server can put one in front of
  the library. That is not our job.

- **A React/web dashboard for inspecting `.vdb` files.** A browser-
  based graph visualizer. Rejected. Adds npm, React, a build step.
  Use `dogma vet <file.vdb>` in your terminal, or `jq` the JSONL.
  If you really want a web UI, build one on top of the open format.

- **A chat UI for the agent harness.** A web frontend to interact
  with the agent. Rejected. The agent speaks NDJSON; any terminal
  can consume it. `tail -f | jq` is a chat UI. If you want a
  graphical one, the gateway is the place to build it.

- **A cloud-hosted version of the agent.** "Dogma Cloud". Rejected.
  The thesis is local-first. Cloud would split the codebase, the
  community, and the principles. If someone wants to host a
  managed Dogma, they can. That is not us.

---

## The Three Questions

Before any new feature lands in the Dogma ecosystem, it must answer
**yes** to all three questions:

1. **Would an indie AI builder ask for this in their first week?**
2. **Can we do this with the primitives we already have?**
3. **Does it break any of: local-first, zero-config, one-file state?**

If it fails one, we do not add it. If it fails two, we reject it
politely. If it passes all three, it goes in. The full rule and its
enforcement are documented in [CONTRIBUTING.md](./CONTRIBUTING.md).

The Three Questions are the meta-rule. Pragmatism is the spirit. The
manifesto is the direction. Rules are made to be broken — but only
deliberately, and only with a written reason.

---

## See also

- [STRATEGY.md](./STRATEGY.md) — what we are doing in the next 60 days
  and why.
- [ROADMAP.md](./ROADMAP.md) — single source of truth for what is next.
- [FAQ.md](./FAQ.md) — versus LangChain, MemGPT, Pinecone, and others.
- [CONTRIBUTING.md](./CONTRIBUTING.md) — the Three Questions, code
  style, and how to propose features.
- [GLOSSARY.md](./GLOSSARY.md) — `.vdb`, NDJSON, IE, SIMIL, RSI, and
  the rest of the vocabulary.

---

<div align="center">

**This is a manifesto. It is a direction, not a contract.**
**Pragmatism beats dogma, even here.**

</div>

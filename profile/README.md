<div align="center">

  <!-- Logo -->
  <img src="./logo.png" width="160" alt="DogmaLab Logo" style="margin-bottom: 20px;"/>

  <!-- Main Titles -->
  <h1>DogmaLab</h1>
  <p align="center">
    <strong>A platform of harnesses that takes transformer LLMs beyond the frontier,<br/>
    with a cost that is affordable and calculable.</strong>
  </p>

  <!-- Tech Stack Badges -->
  <p align="center">
    <img src="https://img.shields.io/badge/Language-Rust-black?style=for-the-badge&logo=rust&logoColor=white" alt="Rust"/>
    <img src="https://img.shields.io/badge/Architecture-AI--Native-007ACC?style=for-the-badge" alt="AI-Native"/>
    <img src="https://img.shields.io/badge/Local--First-2EA043?style=for-the-badge" alt="Local-First"/>
    <img src="https://img.shields.io/badge/Zero--Network-DC3545?style=for-the-badge" alt="Zero-Network"/>
    <img src="https://img.shields.io/badge/Status-Active--Development-emerald?style=for-the-badge" alt="Active Development"/>
    <img src="https://img.shields.io/badge/License-MIT-gray?style=for-the-badge" alt="License"/>
  </p>

  <p align="center">
    <a href="https://github.com/dogmalab/.github/blob/main/MANIFESTO.md">Manifesto</a> •
    <a href="https://github.com/dogmalab/.github/blob/main/STRATEGY.md">Strategy</a> •
    <a href="https://github.com/dogmalab/.github/blob/main/FAQ.md">FAQ</a> •
    <a href="https://github.com/dogmalab/.github/blob/main/ROADMAP.md">Roadmap</a>
  </p>

</div>

---

## The three harnesses

Dogma is a platform of three harnesses. Each is a self-contained
Rust project. The state harness and the agent harness have **zero
network code**. The network harness is the only listener.

<div align="center">
<table style="border: none; width: 100%;">
  <tr>
    <td width="33%" valign="top" style="border: 1px solid #e1e4e8; border-radius: 6px; padding: 16px;">
      <h3>🗄️ dogma-vdb</h3>
      <p><strong>The state harness.</strong> A vector database in one file. JSONL plus a binary v2 mmap. <code>cat</code> it, <code>grep</code> it, commit it. No server. No daemon. No connection string.</p>
      <br />
      <a href="https://github.com/dogmalab/dogma-vdb"><strong>Explore Repository →</strong></a>
    </td>
    <td width="33%" valign="top" style="border: 1px solid #e1e4e8; border-radius: 6px; padding: 16px;">
      <h3>🤖 dogma-agent</h3>
      <p><strong>The agent harness.</strong> An AI runtime in one binary. Three tools, a loop, a state backend, and a Cost Gate that asks before it spends. Air-gapped by design.</p>
      <br />
      <a href="https://github.com/dogmalab/dogma-agent"><strong>Explore Repository →</strong></a>
    </td>
    <td width="33%" valign="top" style="border: 1px solid #e1e4e8; border-radius: 6px; padding: 16px;">
      <h3>🌐 dogma-gateway</h3>
      <p><strong>The network harness.</strong> The only network listener. Bridges HTTP clients to the air-gapped core via IPC pipes and mmap. Stateless. Tiny binary.</p>
      <br />
      <a href="https://github.com/dogmalab/dogma-gateway"><strong>Explore Repository →</strong></a>
    </td>
  </tr>
</table>
</div>

---

## The flagship pattern: Enriched Inference

Inside the agent harness, the **Enriched Inference (IE)** pattern
runs N LLMs in parallel, synthesizes their responses with a
compiler model, iterates the synthesis, and produces a final
answer. The pattern is gated by a **Cost Gate** that estimates
the cost in USD, tokens, and wall-time before the run, and asks
the user to confirm.

The result: any combination of models — local, cloud, or mixed —
compounds into a quality that exceeds any single frontier model,
with a cost the user calculates and confirms before spending.

The claim is falsifiable. The open benchmark
([dogmalab/dogma-arena](https://github.com/dogmalab/dogma-arena))
is where it is tested.

---

## Why "platform of harnesses"?

A **framework** is something your code is hosted inside. A
**harness** is something you compose with. The difference matters:

- When you stop using a framework, you throw away the code.
- When you stop using a harness, the `.vdb` files you wrote
  still work in ten years.

We are not building the next LangChain. We are building the
harnesses that make models better than they thought they could
be. Read the [manifesto](https://github.com/dogmalab/.github/blob/main/MANIFESTO.md)
to understand why.

---

## The Four Questions

Before any new feature lands, it must answer **yes** to all four:

1. Would an indie AI builder ask for this in their first week?
2. Can we do this with the primitives we already have?
3. Does it break local-first, zero-config, or one-file state?
4. Does the harness have a Cost Gate?

If a feature fails one, we do not add it. If it fails two, we
reject it politely. The full rule is in
[CONTRIBUTING.md](https://github.com/dogmalab/.github/blob/main/CONTRIBUTING.md).

---

## Project status

We build in the open.

* 📈 **Roadmap:** See the
  [ROADMAP.md](https://github.com/dogmalab/.github/blob/main/ROADMAP.md)
  for the next 60-90 days.
* 🏟️ **Open benchmark:** [dogmalab/dogma-arena](https://github.com/dogmalab/dogma-arena)
  is where the "beyond the frontier" claim is tested.
* 🤖 **Reference agent:** [dogmalab/dogma-refagent](https://github.com/dogmalab/dogma-refagent)
  is the canonical example.
* 🤝 **Contribute:** Open an issue with the `[rfc]` tag in any
  repository. Read
  [CONTRIBUTING.md](https://github.com/dogmalab/.github/blob/main/CONTRIBUTING.md)
  first.

<br />

---

<div align="center">
  <sub>Founded and led by <a href="https://github.com/arggil">@arggil</a></sub>
</div>

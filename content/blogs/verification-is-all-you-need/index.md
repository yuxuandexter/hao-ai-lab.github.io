+++
title = "Verification Is All You Need: Prospecting the Next Generation of Provable AI Infra"
date = 2026-11-09T12:00:00-08:00
authors = ["Yuxuan Zhang*", "Yichao Fu*", "Minshen Zhang", "Junda Su", "Yi Sun", "Junda Chen", "Hao Zhang"]
author = "Yuxuan Zhang*, Yichao Fu*, Minshen Zhang, Junda Su, Yi Sun, Junda Chen, Hao Zhang"
ShowReadingTime = true
authorOnNewLine = true
draft = false
math = true
contentClass = "post-content-justified"
summary = "Vibe coding is entering its second half, shifting the bottleneck from agents' capability to agents' verification. We introduce vstack, a 5-stage formal verification workflow for coding agents, and apply it to Prefill-Decode disaggregated LLM serving—uncovering real RDMA lifecycle bugs in SGLang and vLLM and building a provably safe prototype on NVIDIA GB200."

[cover]
  image = "img/cover.png"
  relative = true
  alt = "Verification Is All You Need — Let agents write. Make them pass checks they can't edit."
  caption = "Verification Is All You Need: Prospecting the Next Generation of Provable AI Infra"
  hidden = true

[socialIcons]
  [[socialIcons.icon]]
    name = "twitter"
    url = "https://twitter.com/haoailab"
+++

<div style="display: flex; flex-wrap: wrap; gap: 12px; align-items: center; margin: 4px 0 18px;">
  <a href="https://github.com/yuxuandexter/vstack" target="_blank" rel="noopener noreferrer" style="text-decoration: none;">
    <img src="https://img.shields.io/badge/GitHub-vstack-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub: vstack" style="height: 28px; margin: 0;">
  </a>
  <a href="https://github.com/yuxuandexter/provable-pd-serving" target="_blank" rel="noopener noreferrer" style="text-decoration: none;">
    <img src="https://img.shields.io/badge/GitHub-provable--pd--serving-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub: provable-pd-serving" style="height: 28px; margin: 0;">
  </a>
  <a href="https://x.com/haoailab" target="_blank" rel="noopener noreferrer" style="text-decoration: none;">
    <img src="https://img.shields.io/badge/Follow-@haoailab-000000?style=for-the-badge&logo=x&logoColor=white" alt="X: @haoailab" style="height: 28px; margin: 0;">
  </a>
</div>

<figure style="margin: 14px 0 28px;">
  <video controls preload="metadata" poster="img/poster.png" style="width: 100%; border-radius: 8px; border: 1px solid #e5e7eb; display: block;">
    <source src="img/overview.mp4" type="video/mp4">
  </video>
</figure>

Vibe coding is entering its second half. In the first half, the focus was **Agents' Capability**: we were amazed that coding agents could write working software at all. In the second half, the bottleneck shifts to **Agents' Verification**: when agents generate code at high speed, how do we know it is actually correct? As Lauren Tan ([@poteto](https://x.com/poteto/status/2117437607940681849)) recently noted in [`pstack`](https://github.com/cursor/plugins/tree/main/pstack), there is a growing sense that AI writes too much "slop code." Spending more tokens on agents reviewing other agents does not fix this; it is far better to spend those tokens on empirical and formal verification.

This shift becomes unavoidable as we scale. Today, one engineer might guide a few coding agents and still read every diff by hand. Soon, a single engineer will orchestrate **100, 1,000, or 10,000 agents** in parallel, refactoring schedulers, writing kernels, and submitting pull requests around the clock. At that speed, human code review hits a hard cognitive wall. **When autonomous coding agents begin writing the deepest layers of production systems, how do we verify that their code is actually right?**

---

## Why AI Infra?

Modern AI infrastructure has become too complex to verify by human code review alone. Over the past three years, inference engines like [vLLM](https://github.com/vllm-project/vllm) and [SGLang](https://github.com/sgl-project/sglang) have evolved from single-node executors into asynchronous distributed systems—stacking [**paged KV memory**](https://arxiv.org/abs/2309.06180), [**shared prefix caching**](https://proceedings.neurips.cc/paper_files/paper/2024/hash/724be4472168f31ba1c9ac630f15dec8-Abstract-Conference.html), and [**disaggregated prefill-decode RDMA transfers**](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin) onto the exact same physical GPU memory page. When a request times out or aborts, its cleanup now races across RDMA network cards, CPU cache trees, and GPU attention kernels simultaneously. Because [GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/) writes directly into GPU memory without involving the target CPU, a lifecycle mismatch does not crash the server—it returns `HTTP 200 OK` while silently mixing one user's KV cache into another user's response, as seen in the recent [Cloudflare](https://blog.cloudflare.com/kv-cache-exposure-remediation/) and [LMSYS](https://www.lmsys.org/blog/2026-10-26-cf-security/) disclosures where KV lifecycle bugs leaked data across **155 user responses** and exposed cross-node RDMA abort races.

At the same time, **debugging these concurrency bugs on real hardware is slow and expensive**. Standard CI unit tests run on a single machine and rarely trigger distributed race conditions. Reproducing an RDMA lifecycle race requires multi-GPU RDMA hardware like H100 or GB200, separate prefill and decode instances, and microsecond-level timing across workers. Running long stress tests on GPU clusters to hunt for rare corner cases burns scarce compute—and still cannot guarantee that all race conditions are gone.

To catch these deep concurrency bugs without burning GPU hours, we need **formal verification**: modeling the system as a state machine and using `TLA+` model checkers and [`Z3`](https://github.com/Z3Prover/z3) and [`TLAPS`](https://proofs.tlapl.us/) theorem provers to exhaustively check millions of asynchronous schedules on CPUs. In the past, formal verification was impractical for AI infrastructure because engines like vLLM and SGLang merge hundreds of pull requests every week, while manually writing and updating formal specifications and proofs takes months of human effort. **Today, coding agents give us a practical way to bridge this gap.** Because LLMs can read large systems codebases and generate thousands of lines of formal modeling and proof code, we can guide coding agents to specify, model-check, and prove critical subsystems—making **provable AI infra** practical in fast-moving open-source projects.

---

## How We Verify: Formal Verification with `vstack`

The hardest part of formal verification is not running the solver, but writing an accurate **Specification**: a requirement the software must never violate under any asynchronous schedule. Thanks to open-source [vLLM](https://github.com/vllm-project/vllm) and [SGLang](https://github.com/sgl-project/sglang), we do not invent these requirements in a vacuum, but ground them directly in real user issues, bug-fix PRs, and GPU kernel preconditions.

However, simply asking a coding agent to write a formal model and prove a specification rarely works out of the box. When a model checker reports a race condition, **a naive coding agent will often reward-hack the verifier**: instead of surfacing the real bug, it weakens the invariant, deletes the timeout branch from the model, or adds an unrealistic `ASSUME` statement so the checker returns `0 errors`.

{{< image src="img/fig1.gif" alt="The vstack 5-stage verification workflow" width="100%" title="Figure 1. The 5-stage vstack verification workflow. Common agent reward-hacks—weakening the rule, deleting the timeout branch, or assuming the race away—are blocked by pinned specifications, fidelity checks, and proof gates until a real fix passes." >}}

To prevent reward hacking and make agent-driven verification reliable, we built [**`vstack`**](https://github.com/yuxuandexter/vstack), an open-source plugin for Claude Code and Cursor that enforces a **5-stage verification workflow**:

1. **`Specification` — Hash-Pinned Contract Freeze:** Encode these software requirements as formal safety invariants and pin the files by hash so the agent cannot weaken the proof goal.
2. **`Abstraction` — Line-Anchored Modeling:** Build `TLA+` and `Z3` models where every state transition maps to an exact `file:line@commit` in the production codebase, splitting non-atomic cross-thread and cross-node operations into separate steps.
3. **`Validation` — 4-Point Fidelity Gate:** Before trusting any proof, force the model to pass four checks: the historical buggy code *must fail* on the model, the patched code *must pass*, **removing any single guard from the fix *must fail again***, and real execution traces *must replay cleanly through the model*.
4. **`Verification` — Model Checking and Theorem Proving:** Run [`TLC`](https://docs.tlapl.us/using:tlc:start) breadth-first search to find the shortest counterexample trace for any bug, then use [`Apalache`](https://apalache-mc.org/), [`Z3`](https://github.com/Z3Prover/z3), and [`TLAPS`](https://proofs.tlapl.us/) to prove that the fixed state machine and GPU memory offsets hold for arbitrary numbers of requests and pages.
5. **`Continuous Checking` — CI Regression & Runtime Guard:** Re-run the specification suite on new pull requests on CPUs, and mirror the proven preconditions as lightweight runtime checks in the engine.

---

## Case Study: Formally Verifying PD Disaggregated Serving

As a prototype, we applied `vstack` to **Prefill-Decode (PD) Disaggregated Serving**, where a Prefill node computes the prompt's KV cache and transfers it over one-sided RDMA to a Decode node that shares cached prefixes and runs the Decode attention kernel, as shown in **Figure 2**. For this core KV page subsystem, we define **four specifications**:

{{< image src="img/fig2.gif" alt="Silent KV corruption under disaggregated Prefill-Decode RDMA" width="100%" title="Figure 2. Silent KV corruption under disaggregated Prefill-Decode RDMA. If Decode reclaims request A's GPU page on a local timeout and assigns it to request B while Prefill's one-sided RDMA write is still in flight, A's late payload overwrites B's KV page and the server still returns HTTP 200." >}}

1. **`Isolation` — No Cross-Request Contamination:** In-flight RDMA transfers or late control messages from an old request must never read or overwrite a new request's memory or state.
2. **`Integrity` — No Corrupted Reads or Cache Publishes:** Every KV page read by the Decode kernel or published into the global prefix cache must contain complete, uncorrupted data from the intended request generation.
3. **`Finality` — No Deadlocks or Infinite Waits:** Every active request and draining memory page must reach a terminal state in finite steps rather than hanging forever.
4. **`Conservation` — No Silent Memory Loss:** Every physical page is strictly accounted for—a page returns to `FREE` only after hardware RDMA stop evidence arrives, or else stays isolated in `QUARANTINED` accounting.

### Uncovering Real Concurrency Issues in SGLang and vLLM

Checking the production PD implementations of [SGLang](https://github.com/sgl-project/sglang) and [vLLM](https://github.com/vllm-project/vllm) against these four specifications on CPUs uncovered **three classes of concurrency issues**, which we reproduced over GPUDirect RDMA on NVIDIA GB200 and addressed in **6 upstream pull requests**:

- **`Issue 1` — Reclaiming Transfer Slots or KV Blocks While RDMA Is Still In Flight:** When a request aborts or times out, one-sided RDMA transfers already queued on the NIC cannot be canceled in hardware; yet both engines allowed new requests to reuse the same transfer identity or reallocate the expired GPU page before the hardware transfer stopped, violating **`Isolation`**. On GB200, late RDMA accesses silently read or overwrote another request's KV cache under `HTTP 200 OK` across both SGLang room-reuse runs and **3/3** vLLM pull-lease trials. We proposed upstream fixes in SGLang [`#48883`](https://github.com/sgl-project/sglang/pull/48883), [`#48889`](https://github.com/sgl-project/sglang/pull/48889), [`#48960`](https://github.com/sgl-project/sglang/pull/48960), [`#48979`](https://github.com/sgl-project/sglang/pull/48979), and [`#48982`](https://github.com/sgl-project/sglang/pull/48982), alongside related vLLM fix [`#62065`](https://github.com/vllm-project/vllm/pull/62065).
- **`Issue 2` — Publishing Race-Corrupted KV Pages into the Global Prefix Cache:** When an RDMA transfer finishes, the Decode node inserts the received KV pages into the global prefix cache without checking whether the page's lifecycle generation was invalidated mid-flight by `Issue 1`, violating **`Integrity`**. In **3/3** vLLM prefix-caching trials on GB200, this turned a one-time RDMA race into **persistent cache poisoning**, corrupting subsequent requests that hit that shared prefix until eviction.
- **`Issue 3` — Unrenewed Transfer Leases and Hung Decode Requests:** When a remote transfer stalls or an abort notification is dropped, unrenewed leases and missing watchdogs leave Decode requests waiting indefinitely while holding their allocated GPU pages, violating **`Finality`** and **`Conservation`**. We confirmed this on GB200 for vLLM push-lease stalls and across **29/29** scheduler capacity-deadlock replays, and proposed the upstream fix in vLLM [`#66443`](https://github.com/vllm-project/vllm/pull/66443).

**Why Local Timeouts Fail:** The deepest lifecycle failures above share a fundamental root cause: **using local CPU timeouts or error codes to guess when asynchronous hardware RDMA has finished**. Using [`TLAPS`](https://proofs.tlapl.us/) and [`Z3`](https://github.com/Z3Prover/z3), we formally proved that as long as in-flight one-sided RDMA cannot be revoked in hardware, *any* page reclamation policy based purely on local history is mathematically forced to violate either **`Isolation`** if it reclaims the page early, or **page-level `Finality`** if it holds the page forever.

### Building a Provable PD Serving Prototype

**1. Architectural Delta: Centralizing Ownership in `PdLedger`**

In current engines like SGLang and vLLM, the RDMA connector, prefix cache, and decode scheduler maintain separate local counters over raw `page_id` integers and rely on local timeouts to guess when hardware RDMA is done. To fix this by design, we built [**`provable-pd-serving`**](https://github.com/yuxuandexter/provable-pd-serving) on [`mini-sglang`](https://github.com/sgl-project/mini-sglang) and [Mooncake](https://github.com/kvcache-ai/Mooncake) GPUDirect RDMA around a single capability state machine, **`PdLedger`**, which tracks every physical page via a versioned ticket `(page, gen)` across five explicit states shown in **Figure 3**: **`FREE`**, **`PRIVATE`**, **`SHARED`**, **`DRAINING`**, and **`QUARANTINED`**.

{{< image src="img/fig3.gif" alt="PdLedger page lifecycle resolving the local-timeout dilemma" width="100%" title="Figure 3. How PdLedger resolves the local-timeout dilemma on the RDMA transfer path (prefix-cache Shared state omitted for clarity). Without stop evidence, a timed-out page is isolated in Quarantined so late RDMA writes never touch new requests; with stop evidence, a canceled page waits in Draining and returns to Free only after the remote transfer stops." >}}

Our key design delta is using **`DRAINING`** and **`QUARANTINED`** to decouple *request cancellation* from *hardware DMA completion*, resolving the request-hang-versus-corruption trade-off:

- **Release with Stop Evidence → `FREE`:** No page with in-flight RDMA or kernel access can jump straight back to `FREE`; it must enter `DRAINING` first. Once hardware completion or peer acknowledgement confirms all accesses have stopped, `PdLedger` clears the page back to `FREE` and issues a fresh `gen` on the next allocation, preserving **`Conservation`**.
- **Timeout without Stop Evidence → `QUARANTINED`:** When a transfer times out without stop evidence, `PdLedger` immediately fails the waiting request so Decode never hangs, while isolating that physical page in **`QUARANTINED`** so any late RDMA packet lands harmlessly outside `FREE` and the prefix cache, preserving **`Isolation`** and **`Integrity`**.

**2. Proving the Formal Model**

Using `vstack`, we modeled the `PdLedger` protocol in `TLA+` and verified that the design satisfies all four specifications on CPUs. First, [`TLC`](https://docs.tlapl.us/using:tlc:start) exhaustively checked **21M+ reachable states** on bounded configurations with zero invariant violations. Next, we completed an unbounded **mathematical induction proof in [`TLAPS`](https://proofs.tlapl.us/)** across all **3,456 (`64 × 54`) inductive preservation obligations**, paired with standalone [`Z3`](https://github.com/Z3Prover/z3) proofs verifying that 6D GPU tensor byte offsets never overlap or go out of bounds.

**3. Aligning the Python Implementation to the Proven Model**

Of course, a [`TLAPS`](https://proofs.tlapl.us/) and [`Z3`](https://github.com/Z3Prover/z3) proof only guarantees that our **formal model** is bug-free—not that our Python code faithfully matches it. To close this **model-to-code gap**, we enforce synchronous `PdLedger` precondition checks at runtime, verify that **67 injected code mutations** are **100%** caught, and replay real execution traces step-by-step through `TLC` trace validation to confirm every code transition follows the proven model.

---

## Evaluation on GB200

To move from a CPU model to a **provable PD serving** prototype, we validated `provable-pd-serving` on a 4-GPU NVIDIA GB200 node with `Qwen3-0.6B` and `Qwen3-32B`, running separate Prefill and Decode instances over Mooncake GPUDirect RDMA:

- **Formal Proof Coverage:** On the model side, we exhaustively checked **21M+ `TLC` states**, proved 6D GPU tensor byte-offset safety in **`Z3`**, and proved all **3,456 (`64 × 54`) inductive obligations** in [`TLAPS`](https://proofs.tlapl.us/).
- **Low Runtime Overhead:** On 64 reconstructed [Alibaba `qwen-bailian`](https://github.com/alibaba-edu/qwen-bailian-usagetraces-anon) requests on `Qwen3-32B`, `PdLedger` takes **`333 µs` of CPU time per `13.43 ms` decode step**, or **`2.5%`**, largely overlapped with GPU execution; even enabling an extra full-ledger scan on every decode step adds just **`+0.7%`** to `TPOT` p50 with **zero KV checksum mismatches**.
- **Zero Memory Corruption Under Cache & Fault Stress:** Under concurrent `256`-token shared-prefix bursts and tight 12-page cache-eviction pressure, `PdLedger` recorded **zero KV checksum mismatches and zero ledger violations**, while blocking **3/3** injected prefix-lifecycle bugs in fault-injection tests.
- **100% Trace Validation Across `22,104` Events:** Matching live distributed P/D execution to a single `TLA+` state machine is non-trivial because Prefill and Decode have no shared clock and RDMA operates on raw GPU memory addresses rather than abstract page states. By causally merging P/D boundary logs and mapping transfers back to `(page, gen)` steps, **100% of `22,104` logged P/D boundary and ledger events across `540/540` request prefixes** replayed cleanly through `TLC`.

---

## Open Source and What's Next

We are open-sourcing all three parts of our work:

1. [**`vstack`**](https://github.com/yuxuandexter/vstack) — Our 5-stage formal verification plugin for Claude Code and Cursor.
2. [**`provable-pd-serving`**](https://github.com/yuxuandexter/provable-pd-serving) — Our verified PD serving prototype on [`mini-sglang`](https://github.com/sgl-project/mini-sglang) and [Mooncake](https://github.com/kvcache-ai/Mooncake) RDMA, including `PdLedger`, the `TLA+`, [`TLAPS`](https://proofs.tlapl.us/), and [`Z3`](https://github.com/Z3Prover/z3) models, and the trace validation suite.
3. **Upstream Fixes for [SGLang](https://github.com/sgl-project/sglang) and [vLLM](https://github.com/vllm-project/vllm)** — Concurrency fixes proposed across SGLang [`#48883`](https://github.com/sgl-project/sglang/pull/48883), [`#48889`](https://github.com/sgl-project/sglang/pull/48889), [`#48960`](https://github.com/sgl-project/sglang/pull/48960), [`#48979`](https://github.com/sgl-project/sglang/pull/48979), [`#48982`](https://github.com/sgl-project/sglang/pull/48982) and vLLM [`#66443`](https://github.com/vllm-project/vllm/pull/66443).

Verifying the KV page lifecycle of PD disaggregated serving is just the first step. Our formal verification work on hierarchical KV caching such as [SGLang HiCache](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/), multi-page prefix trees, and low-level GPU kernels is next—**more provable AI infra is coming soon, stay tuned.**

---

## Acknowledgement

We thank NVIDIA for supporting our development and GPUDirect RDMA evaluation on GB200 NVL72. We thank the SGLang, Mini-SGLang, vLLM, and Mooncake communities for building the open-source disaggregated serving stack and reviewing our upstream bug reports and fixes, as well as the Qwen team and Alibaba for releasing the Qwen3 models and `qwen-bailian` traces used in our evaluation. Our verification workflow builds on the open-source TLA+, TLC, Apalache, TLAPS, and Z3 ecosystems, and draws inspiration from Lauren Tan's `pstack`.

## Contributors

[Yuxuan Zhang*](https://yuxuandexter.github.io/), [Yichao Fu*](https://github.com/Viol2000), [Minshen Zhang](https://alexzms.github.io/), [Junda Su](https://davids048.github.io/), [Yi Sun](https://www.linkedin.com/in/yi-sun-mlsys), [Junda Chen](https://github.com/GindaChen), [Hao Zhang](https://haozhang.ai)

*\* Equal contribution.*

## Citation

```bibtex
@misc{vstack2026,
  title        = {Verification Is All You Need: Prospecting the Next Generation of Provable {AI} Infra},
  author       = {Zhang, Yuxuan and Fu, Yichao and Zhang, Minshen and Su, Junda and Sun, Yi and Chen, Junda and Zhang, Hao},
  year         = {2026},
  month        = nov,
  howpublished = {Hao AI Lab Blog},
  url          = {https://haoailab.com/blogs/verification-is-all-you-need/}
}
```

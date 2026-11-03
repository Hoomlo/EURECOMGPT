# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->236.326<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.991<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->448434.4<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->15891.0<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.475<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1: 2585.124, 2: 2466.989, 4: 2378.921, 8: 2817.565. The fastest thread count was 4 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive implementation executes nested Python loops over tensors, causing heavy interpreter overhead and memory latency as individual scalars are fetched. The vectorized version replaces these loops with high-level NumPy operations backed by optimized BLAS libraries. At the hardware level, this leverages SIMD (Single Instruction, Multiple Data) registers to process multiple matrix elements simultaneously in parallel execution units, drastically reducing instruction cycles and memory access overhead.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Performance scaled positively up to 4 threads (2378.921 ms) but degraded significantly at 8 threads (2817.565 ms). Colab standard instances typically provide limited physical CPU cores sharing execution resources. Pushing beyond the physical core count introduces thread contention, context-switching overhead, and cache thrashing. Furthermore, the synchronization and coordination cost among threads outweighed the parallel computation gains for this workload size.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformer training relies heavily on massive, parallel matrix multiplications (GEMM) across layers and batch dimensions, which map ideally to the thousands of Arithmetic Logic Units (ALUs) present on a GPU. Conversely, sequential algorithms with strict data dependencies or heavy branching logic—such as standard breadth-first search (BFS) graph traversals or recursive text tokenization loops—would not see this speed-up and often perform worse on GPUs due to core synchronization overhead.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism hits distinct physical and architectural limits across different scaling levels. In the attention vectorization experiment, eliminating Python loop overhead via SIMD achieved massive gains, bringing time down from 236.326 ms to 0.991 ms, proving that unvectorized code wastes raw hardware capacity. In the multi-threading sweep, coordination costs and hardware limits took over: performance improved up to 4 threads (2378.921 ms) but degraded at 8 threads (2817.565 ms) due to core contention and synchronization overhead exceeding parallel gains. Finally, the macro-level device comparison exposed hardware scale limits and data movement bottlenecks; CPU training required 448434.4 ms for 100 steps, whereas the GPU slashed this to 15891.0 ms. Here, the CPU was bound by limited core count and memory bandwidth for massive tensor algebra, whereas the GPU provided the specialized massively parallel architecture required to sustain deep learning workloads.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would measure training throughput scaling across increasing batch sizes on the GPU. If increasing batch size yielded sub-linear scaling past a certain threshold, it would prove that memory bandwidth (data movement to VRAM) rather than compute capacity became the ultimate system bottleneck.
<!--/answer-->

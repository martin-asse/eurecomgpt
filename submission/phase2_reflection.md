# Phase 2 — Reflection: where does parallelism stop helping?

Author: Martin Asse

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->401.813<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->20.005<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->438758.2<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13191.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4311<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2170.851 ms/step; 2 threads: 1971.366 ms/step; 4 threads: 2333.673 ms/step; 8 threads: 2377.225 ms/step. Fastest: 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.**

<!--answer:vectorization_why-->
The naive version runs a Python loop over the sequence, so each dot product pays interpreter overhead, a function call and a tiny memory access, and the CPU's vector units sit mostly idle. The vectorized version passes whole matrices to NumPy, which calls an OpenBLAS routine compiled with AVX2/AVX-512 and FMA: each instruction handles 8-16 floats, data is processed in cache-friendly blocks. Result: 401.81 ms down to 20.00 ms, a 20.1x speedup for identical arithmetic.
<!--/answer-->

**Where the thread curve flattens, and why.**

<!--answer:threading_plateau-->
The curve bottoms out at 2 threads (1971 ms/step vs 2171 ms at 1 thread, only about 1.10x instead of the ideal 2x), then gets worse: 2334 ms at 4 threads and 2377 ms at 8. The Colab runtime has only 2 vCPUs, so beyond 2 threads the extra workers just compete for the same cores (oversubscription, context switches). Each parallel operation also needs synchronization barriers, and our small matrices (about 4.9M parameters, batch 16) give too little work per thread to amortize that coordination cost.
<!--/answer-->

**What made the GPU faster.**

<!--answer:gpu_why-->
A transformer training step is dominated by large dense matrix multiplies (attention projections, MLP layers) that are independent per element, so thousands of CUDA cores on the T4 can work on them at once with high memory bandwidth. This gave 100 steps in 13192 ms vs 438758 ms on CPU (33.3x). A workload that would not benefit: one with sequential dependencies or heavy branching, such as a recursive parser or pointer-chasing over a linked structure, where there is no wide data parallelism to spread.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?**

<!--answer:parallelism_limit-->
Each experiment hit a different limit. Vectorization (SIMD) was limited by what one core's vector units can do: it removed interpreter overhead and gave 20.1x (401.81 to 20.00 ms), but it cannot go further on a single core. Threading was limited by hardware we do not have and by coordination: with 2 vCPUs, 2 threads gave only 1971 ms vs 2171 ms at 1 thread (1.10x), and 8 threads were slower at 2377 ms because synchronization and oversubscription cost more than they saved. The GPU gave 33.3x (13192 vs 438758 ms), far below its core-count advantage: training steps are strictly sequential (2000 dependent steps cannot be divided), and each step still involves launches, Python overhead and moving batches to the device, which stay serial. So the bound was indivisible work for the GPU, missing cores plus coordination for threads, and per-core throughput for SIMD.
<!--/answer-->

**What you would measure next.**

<!--answer:next_experiment-->
A GPU batch-size sweep (16, 64, 256, 1024), measuring ms per step and tokens per second. If time per step stays almost flat as batch grows, the GPU was underused at batch 16 (launch and transfer bound) and the 33.3x is a floor; if it grows linearly, it was already compute bound.
<!--/answer-->
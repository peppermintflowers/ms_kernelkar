---
layout: post
title: "Okay, Thready?"
date: 2026-09-11 01:01:00 -0500
---

The day after the tiling rennovations had let her down, Ms. Kernelkar decided it was finally time. She was going to CUDA.

This was, after all, where she had been trying to get to.

Too exhausted from labouring the day prior to say "Hello, World", she pulled the blinds, stretched her fingers and began frantically typing her first CUDA kernel.

## Vector. ADD. 

Here is what it looked like:

```cpp
__global__ void vector_add(const float* A, const float* B, float* C, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;

    if (i < N)
        C[i] = A[i] + B[i];
}
```

One thread was responsible for one element.

She knew GPU resources needed to be kept busy to get good performance, so she ran the kernel for **10,000,000 elements**, and progressively increased the number of threads per block. 

The threadier, the stealthier she thought.

| Threads / Block | Blocks | Median Time (ms) |
|:---:|---:|---:|
| 32  | 312,500 | 0.247808 |
| 64  | 156,250 | 0.126976 |
| 128 | 78,125  | 0.095232 |
| 256 | 39,063  | 0.094208 |
| 512 | 19,532  | 0.095232 |

Going from 32 to 64 threads per block almost halved the measured time, and at 128 threads it improved further.
But after that boost the performance seemed steady. 128, 256, and 512 threads per block all landed at roughly **0.095 ms**.

More threads in a block do not make the work performed per thread any cheaper. So once enough parallelism was exposed for this kernel, increasing the block size further gave very little in return.

**Enough parallelism matters. More is not neccessarily better.**

## New session, old lesson

Back on the CPU, Ms. Kernelkar had learned that *how* a program walks through memory can matter as much as what computation it performs. So this matrix copy lesson with CUDA was just revision!

She used two copy kernels over an **8192 × 8192** row-major matrix, with both performing the same amount of work. However, they differed in how they mapped neighbouring threads to memory. She tried a version where neighboring threads in a warp accessed nearby memory locations, and another, where their accesses were strided through memory.

The results:

| Access Pattern | Median Time (ms) |
|:---:|---:|
| Coalesced | 0.4352 |
| Strided | 2.58202 |

Well, well, a **5.93× slowdown**!

The GPU does not handle each thread's global memory request in complete isolation. When threads in a warp access suitably adjacent addresses, those accesses can be served efficiently using **coalesced** memory transactions. With the accesses spread out however, moving the same amount of data can become much less efficient!

## Shared space

Ms. Kernelkar had read that using shared memory instead of global was a great optimization technique. 

It seemed to be a reasonable claim that the shared memory that lives on-chip and can be accessed by threads within a block, would be much faster to access than global memory. 

However, having learnt the importance of benchmarking optimizations and understanding now that they all came with their own set of trade-offs, she wanted to know if AND *when* this would actually help. 

Her experiment compared two versions of a kernel. One that repeatedly accessed a source value through global memory, and another that first staged data in shared memory and then reused it from there.

She kept the workload at **2²⁴ elements**, used **256 threads per block**, and varied how many times the data was reused.

| Reuse | Global (ms) | Shared (ms) | Global / Shared |
|:---:|---:|---:|---:|
| 1  | 0.110080 | 0.111616 | 0.986× |
| 2  | 0.118784 | 0.116736 | 1.018× |
| 4  | 0.129024 | 0.128000 | 1.008× |
| 8  | 0.194560 | 0.168960 | 1.152× |
| 16 | 0.337408 | 0.288256 | 1.171× |
| 32 | 0.628736 | 0.525312 | 1.197× |

At reuse = 1, shared memory was actually a little slower.

At reuse = 2 and 4, shared memory seemed to have caught up.

Then, as reuse further increased, the shared memory version began to pull ahead.

Finally, at reuse = 32, shared memory showed a **1.20×** speedup.

## Why did it lag on lower reuse?

Think of the added work. Before data can be reused from shared memory, it first has to get there. Threads stage the data, and synchronization may be required before other threads safely use it. As the reuse count increases, however, that initial cost gets spread across more accesses and eventually the benefit starts to outweigh the overhead.

## CUDA been shorter

As Ms. Kernelkar creates her LeetGPU account she thinks about what she can take away from these experiments to earn a spot on the leaderboard one day.

What seems to help:
- enough parallelism
- convenient memory fetching
- remembering everything has a cost

And while Ms. Kernelkar is more than happy to polish off all the easy problems in an afternoon, there is a matrix multiplication she has been procrastinating on. 

---

## Signing off

This post is about what Ms. Kernelkar found interesting while self-studying CUDA and GPU performance optimization. The complete kernels, benchmark code, and measurements are in the project repository:

**[CUDA Performance Fundamentals →](https://github.com/peppermintflowers/cuda_performance_basics)**

If you spot something Ms. Kernelkar misunderstood, please let her know! She'll catch you on her next run.

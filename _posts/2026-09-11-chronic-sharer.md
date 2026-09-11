---
layout: post
title: "Chronic Sharer"
date: 2026-09-11 13:21:00 -0500
---

Last time, Ms. Kernelkar learned that if there was enough scope for reuse, moving data to the faster shared memory could actually be worth all the effort.. 

Conveniently, she knew a computation with quite a lot of reuse. Unfortunately, she knew it rather well.

Some may remember the time Ms. Kernelkar tiled a matrix multiplication and her supposedly clever CPU optimization made it slower. But one cant just refuse **GEMM**s, so she wasn't going to give up this easy.

## A naive recluse

For:

$$ C = A \times B $$

Her first CUDA implementation assigned one thread to one output element of C.

```cpp
__global__ void naive_gemm(const float* A, const float* B, float* C, int N) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;

    if (row < N && col < N) {
        float sum = 0.0f;
        for (int k = 0; k < N; k++)
            sum += A[row * N + k] * B[k * N + col];
        C[row * N + col] = sum;
    }
}
```
Each thread walked through the inner dimension K and computed one dot product.

For an **8192 × 8192** GEMM, she used a 32 x 8 block, allowing each warp to span a full row of output columns.  

| Metric | Result |
|:---|---:|
| Median Time | 461.691 ms |
| Performance | 2381.49 GFLOP/s |

About **2.38 TFLOP/s**? Not terrible at all!

But, Ms. Kernelkar noticed there was an unaddressed inefficiency.
The threads on these blocks were stingy neighbours, fetching everything for themselves..when it turns out, they had a lot in common.


## Ms. Kernelkar clears her throat.

Consider a block of threads computing a small region of C. Each output needs a row from A and a column from B. But neighboring outputs overlap heavily in the data they need.

Threads across the same output row reuse A values. Threads across the same output column reuse B values.

In the naive kernel however, every thread performed its own loads through the global/cache memory path as it marched through K. 

Time to build community.

## The (clever?) sharer

Ms. Kernelkar conjured a kernel where blocks of CUDA threads cooperatively loaded a small tile of A and a small tile of B into **shared memory**.

```cpp
__shared__ float As[TILE][TILE];
__shared__ float Bs[TILE][TILE];

As[threadIdx.y][threadIdx.x] = A[...];
Bs[threadIdx.y][threadIdx.x] = B[...];

__syncthreads();
```

The threads then performed several multiply-accumulates using those shared values before moving on to the next pair of tiles.

Instead of repeatedly depending on the global/cache path for every value needed by every thread, the block paid to stage a tile once and then reused it.

For N = 8192, she tried three square tile configurations.

| Tile Size | Threads / Block | Median Time (ms) | Performance (GFLOP/s) |
|:---:|---:|---:|---:|
| 8×8 | 64 | 383.353 | 2868.15 |
| 16×16 | 256 | 233.639 | 4706.03 |
| 32×32 | 1024 | 206.894 | 5314.37 |


Hurrah! Tiling worked !!

The 32×32 version reached about **5.31 TFLOP/s**, compared with roughly **2.38 TFLOP/s** for the original naive run.

There was, however, a small experimental complication.

In this implementation, tile size and block size were tied together. An 8×8 tile used 64 threads, 16×16 used 256, and 32×32 used 1024. So this sweep did **not** isolate tile size alone. Each row represents a different overall kernel configuration.

There needed to be a cleaner comparison.

## More experiments

She fixed both the naive and tiled kernels at **16×16 threads per block** and ran them across several matrix sizes.

<img src="{{ '/assets/chronic_sharer/gemm_scaling.png' | relative_url }}" width="60%">

With the thread block configuration held constant, the tiled kernel was faster at **every matrix size tested**.

And the measured advantage grew with the workload:

```text
N = 1024 -> 1.48×
N = 2048 -> 1.63×
N = 4096 -> 1.97×
N = 8192 -> 2.10×
```

At N = 8192, the 16×16 tiled implementation reached about **4.72 TFLOP/s**, a little over twice the performance of the naive kernel in the controlled comparison.

But Ms. Kernelkar wanted to know *why*.

## A little Nsight.

She profiled the N = 4096 versions of both kernels.

| Metric | Naive | Tiled |
|:---|---:|---:|
| Registers / Thread | 32 | 32 |
| Static Shared Memory | 0 | 2.05 KB |
| Achieved Occupancy | 99.63% | 99.48% |
| Active Warps / Scheduler | 15.92 | 15.92 |

Both kernels were sitting at roughly **99% occupancy** and had essentially the same number of active warps per scheduler.

Why was the tiled version almost twice as fast then? 

## Busy doing what?

Here's what the profiling also revealed:

| Metric | Naive | Tiled |
|:---|---:|---:|
| Eligible Warps / Scheduler | 1.61 | 2.29 |
| Issue Slots Busy | 30.75% | 46.48% |
| SM Busy | 30.75% | 46.48% |
| Compute Throughput | 53.76% | 72.98% |
| Memory Throughput | 254.58 GB/s | 462.00 GB/s |

Warps resident on the GPU and warps that are actually ready to do useful work are not the same thing.

The tiled kernel had **more eligible warps**, in other words, more warps that were actually ready to issue instructions.

Nsight also showed large memory related stalls in the naive kernel, including long scoreboard/L1TEX dependency stalls and load/store instruction queue pressure. Too many of the warps on the naive kernel were just waiting.

With the tiled kernel reducing repeated fetches via global/cache memory path, more warps could make progress, the scheduler issued useful work more often, and the SM spent more time actually doing things.

But hey, the tiled kernel wasn't stall free either! Staging tiles meant shared memory instructions and synchronization work. What changed however, was the balance of waiting. 

Nsight also suggested that the naive kernel's warp level memory access pattern could be improved, so shared memory tiling was not necessarily the only optimization available, or a comparison against a fully optimized naive kernel. Rather, it reorganized enough of the repeated memory traffic to substantially improve this implementation.

This particular tiled kernel worked because GEMM contains substantial data reuse, and shared memory gave the block a way to organize that reuse explicitly.
---

## Signing off

This post is about what Ms. Kernelkar found interesting while self studying CUDA and GPU performance optimization. The complete naive and tiled kernels, configuration experiments, scaling results, and Nsight Compute analysis are in the project repository:

**[CUDA GEMM experiments →](https://github.com/peppermintflowers/gemm_cuda)**

If you spot something Ms. Kernelkar misunderstood, please let her know! She'll catch you on her next run.

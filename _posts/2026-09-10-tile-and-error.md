---
layout: post
title: "Tile and Error"
subtitle: "Cache Locality · Tiling · Vectorization"
date: 2026-09-10 11:30:00 -0500
---

On her quest to make GEMM dramatically faster, Ms. Kernelkar encountered the next piece of conventional wisdom:

**If locality is good, surely tiling is better.**

And so began the renovation.

## The Vision

The idea seemed reasonable enough.

Instead of marching across the entire matrices at once, process them in smaller blocks so that the data being worked on has a better chance of hanging around in cache.

Think :

```cpp
for (int ii = 0; ii < N; ii += BLOCK)
    for (int kk = 0; kk < N; kk += BLOCK)
        for (int jj = 0; jj < N; jj += BLOCK)
            // multiply the smaller tiles
```

Ms. Kernelkar expected this to be the part where the numbers went higher.

They did not.

## Out with the Grout

For a 4096 × 4096 GEMM, the unblocked ikj implementation was already doing around **24.5 GFLOP/s**.

And with the tiling:

| Block size | Throughput |
|:---:|:---:|
| 16   | ~7.3 GFLOP/s  |
| 32   | ~11.2 GFLOP/s |
| 64   | ~15.8 GFLOP/s |
| 128  | ~19.7 GFLOP/s |
| 256  | ~18.1 GFLOP/s |
| 512  | ~17.0 GFLOP/s |
| 1024 | ~8.5 GFLOP/s  |

The larger blocks recovered some performance...


Huh? None beat the supposedly less sophisticated unblocked version.

The blocked implementation peaked around block size 128 at about 19.7 GFLOP/s, but remained below the unblocked result of about 24.5 GFLOP/s. Performance then decreased for larger block sizes.

Splendid.

## But wasn't tiling supposed to help?

The problem was that Ms. Kernelkar had been thinking only about **cache locality**, while the compiler had other plans.

***A little flashback to the previous episode: ***:

When she had looked at Clang's optimization reports, she found that the inner loop of her unblocked ikj implementation was being vectorized.

Its structure was particularly friendly:

```cpp
for (int j = 0; j < N; j++)
    C[i*N + j] += A[i*N + k] * B[k*N + j];
```

B and C are traversed contiguously, A[i][k] is reused, and the compiler gets a nice simple inner loop to work with.

Adding tiling introduced extra loop boundaries and changed the structure presented to the compiler. Better theoretical cache reuse did not automatically mean a faster program.

Apparently optimizations are allowed to interfere with other optimizations.

Very cooperative of them.

## So what did she actually learn?

"Tiling improves locality" -> useful.

"Tiling makes code faster" -> not always.

Performance depends on what was already working well, what the compiler can recognize, the block size, the cache hierarchy, vectorization, and probably several other things scheming to embarrass Ms. Kernelkar in future posts.

For now, after an hour of toiling, the takeaway she has for you is:

**An optimization is a hypothesis until you benchmark it.**

Tile and error is the way. Tile and error.

---

## Signing off

The complete GEMM implementation, block-size sweeps, compiler experiments, and benchmark results live here:

**[CPU GEMM experiments →](https://github.com/peppermintflowers/gemm)**

If you spot something Ms. Kernelkar misunderstood, please let her know! She'll catch you on her next run.

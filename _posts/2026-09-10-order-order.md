---
layout: post
title: "Order! Order!!"
date: 2026-09-10
---
<style>
  h2 {
    color: #FF46A2;
    font-family: Arial, sans-serif;
  }
  code {
    background-color: #4646ff4c;
    color: #FFFF46;
    padding: 2px 5px;
  }
  a {
    color: #46FFA3;
  }
</style>

Deeply inspired by the nuggets of wisdom on a discarded Cadbury wrapper from her afternoon snack, and equally fascinated by GEMMs (of a different kind), Ms. Kernelkar has decided to embody “Raho Umarless” and pursue the warpy ways of the GPU.

She has a sense, from the papers she's been reading, that making GEMMs run fast might be harder than becoming a student again. So, after going from OMG to **O(n³)** (<< ding, ding, ding >> the time complexity of matrix multiply), she decided to start with the basics.. Naturally, she began on the CPU.

While she claims her out of order life choices are perfect , ordering does matter quite a bit to matrix multiplication. In fact, with both versions compiled using -O3, she watched the same 2048 × 2048 GEMM go from taking about 22 seconds to 0.77 seconds with some loop order adla badal.

## The GEMM we are talking about

Presenting..matrix multiplication using 2 square (NxN), **row major** matrices.

```cpp
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        for (int k = 0; k < N; k++)
            C[i*N + j] += A[i*N + k] * B[k*N + j];
```

This performs roughly:

$$
2N^3
$$

floating-point operations: one multiplication and one addition for every (i, j, k) combination.

## The Experiment

There are six ways to arrange those three loops above:

`
ijk ,   ikj ,
jik ,   jki ,
kij ,   kji 
`

After running all of them, across different matrix sizes, here were the performance results:

<img src="{{ '/assets/loop_order_comparison.png' | relative_url }}" width="60%">

**ikj** won. By a lot.

At N = 2048, the original ijk implementation took about **22377.6141 ms** but ikj took about **767.3010 ms**.


## What's this behavior ?!

Remember reading about how accessing data already in cache is much cheaper than fetching it from main memory?.. 
For these experiments, the matrices were stored in **row-major order**.

While Ms. Kernelkar thought about a matrix like this:

<img src="{{ '/assets/amatrix.png' | relative_url }}" width="25%">

her computer dealt with something closer to:

<img src="{{ '/assets/amatrix_mem.png' | relative_url }}" width="65%">

See how elements next to each other **within a row** are also next to each other in memory but elements in the same column aren't?

### Now, take a peek at what the innermost loop of ijk does:

```cpp
for (int k = 0; k < N; k++)
    C[i*N + j] += A[i*N + k] * B[k*N + j];
```

As k increments:

`
A[i][0], A[i][1], A[i][2], A[i][3] ...
`

She imagines the accesses waltzing down the carpet of memory cells.

But for B:

`
B[0][j], B[1][j], B[2][j], B[3][j] ...
`

Every time k increments, column accesses jump an entire row through B's memory instead of consuming nearby values.


### On the other hand, for ikj:
```cpp
for (int i = 0; i < N; i++)
    for (int k = 0; k < N; k++)
        for (int j = 0; j < N; j++)
            C[i*N + j] += A[i*N + k] * B[k*N + j];
```

The inner loop walks through:

`
B[k][0], B[k][1], B[k][2], ...
`
and:

`
C[i][0], C[i][1], C[i][2], ...
`

Both are **contiguous** rows.

Also, A[i][k] stays constant throughout that inner loop and gets **reused**.

***Now isn't this a considerably nicer way to ask the memory for data?*** 

## Ms. Kernelkar adjusts her glasses

Ah!! She had read before that sequential memory access is faster because of cache locality.
Apparently, understanding how your accesses line up with memory can make a big difference to performance!
She knows now that the **order in which she requests data** can dominate an otherwise identical computation. 


## Ms. Predictions

Before running the experiment, she had tried to predict the performance ranking of the loop orders based purely on which matrices would be accessed contiguously. She initially expected some more separation between JKI and KJI based on the different reuse patterns of their outer loops. However, in practice, they performed almost identically. Both put i in the innermost loop, causing strided accesses through A and C, and that poor inner-loop locality appeared to dominate any difference between the outer-loop order. 

So while "contiguous ~ good, strided ~ bad" gave her a useful mental model, a real CPU has caches, prefetching, compiler optimizations, vectorization, and a bunch of other things happening underneath that little three-loop program that one needs to consider. 

In other news, she has put her oracle career on hold for now.

## Short lived happiness

At this point, Ms. Kernelkar was quite pleased with herself. She had changed the order of three loops and the memory accesses looked much friendlier. Her ikj implementation took was sooo much faster.

Case closed. Cache locality did it. Order! Order!!

Except...

Both of those numbers came from code compiled with -O3.
[For reference, throughput for IJK and IKJ implementations benchmarked for 2048x2048 matrices, with varying compiler optimization levels.]
<img src="{{ '/assets/compiler_vs_loop_order.png' | relative_url }}" width="65%">

And while poking around Clang's optimization reports, she discovered that the compiler was vectorizing the inner loop of her ikj implementation.

So was the ~29× improvement really all because of cache locality?

Not quite.

Changing the loop order gave the program a much friendlier memory access pattern, BUT it also gave the compiler a loop structure it could optimize much more aggressively. The performance she measured was the result of those effects working **together**.

Unfortunately for Ms. Kernelkar, computers continue to resist simple explanations.

But of course she had to try to make the GEMM run faster!!
That's a story for the next post.

---

## Signing off

This post is about what Ms. Kernelkar found interesting when self-studying performance optimization. The complete implementation, all six loop orders, optimization-level experiments, block-size sweeps, plots, and benchmarking methodology are in the project repository:

**[CPU GEMM experiments →](https://github.com/peppermintflowers/gemm)**

If you spot something Ms. Kernelkar misunderstood, please let her know! She'll catch you on her next run.

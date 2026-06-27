---
title: 'Basic Linear Algebra Subprograms (BLAS)'
description: 'Here in this log we will go through what BLAS is and inside it what is GEMM and how we observe and measure low-level optimizations for performance for various subroutines'
pubDate: 'Jun 27 2026'
heroImage: https://images.unsplash.com/photo-1600348712270-5af9e3590f66?w=900&auto=format&fit=crop&q=60&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxzZWFyY2h8MTl8fHByb2Nlc3NvcnxlbnwwfHwwfHx8MA%3D%3D
---

## What is BLAS?

BLAS is essentially a blueprint that was given to the world for the sole purpose of standardizing the interface of mathematical subroutines.

It does not tell you how your program must work internally, rather it tells what standard input and output must look like if you want to follow the BLAS standard.

### From Where Did Its Need Come?

#### The Problem

Back in the 1970s, scientists were doing heavy math in Fortran. Every time a physicist or a scientist needed to multiply two matrices, they wrote their own custom `for loop` implementation.

*The problem* with this was that a particular implementation working well on an IBM Supercomputer could perform terribly on a Cray Supercomputer.

This meant every time they migrated from one machine to another, they had to rewrite and re-optimize their code for the new underlying architecture — an overhead that scientists simply should not have to carry.

#### The Solution (1979)

Then came forth a group of mathematicians and computer scientists — most notably *Charles Lawson*, *Richard Hanson*, *David Kincaid*, *Fred Krogh*, and later a legendary computer scientist named *Jack Dongarra* (who won the Turing Award largely for his work in this field).

In 1979 they published a paper that introduced the BLAS framework. It essentially said:

*"Stop writing your own math loops. Let's agree on a standard list of function signatures. We will ask the hardware manufacturers to write ultra-optimized implementations for these functions. You, the scientist, just call the standard name — the hardware guys will handle the speed."*

### Functionality

BLAS functionality is categorized into three sets of routines called *levels*. These levels correspond to the chronological order of definition and publication as well as the degree of polynomial complexity of the algorithms.

##### Level-1: Linear Complexity — O(n)

This level consists of all the subroutines defined in the original 1979 paper, which defined only vector operations on strided arrays. For example: `axpy` — ax plus y (y ← ax + y).

##### Level-2: Quadratic Complexity — O(n²)

This level consists of Matrix-Vector operations including a **G**eneralized **M**atrix-**V**ector multiplication `(gemv)`. Design of Level 2 BLAS started in 1984 with results published in 1988. These subroutines were intended to improve performance on `vector processors`, where Level 1 BLAS are suboptimal.

##### Level-3: Cubic Complexity — O(n³)

This level, published in 1990, contains matrix-matrix operations including a **G**eneral **M**atrix-**M**ultiplication `(gemm)` of the form:

C ← αAB + βC

Where A and B can be optionally transposed and all three matrices may be strided. Ordinary matrix multiplication is performed by setting α to 1 and β to 0.

---

We in this blog will focus on Level 3 BLAS — specifically `sgemm`, single precision general matrix multiplication.

---

### The gemm_config_t Struct

Before we dive in, one thing worth understanding is the `gemm_config_t` struct that our kernels use:

```c
typedef struct {
    int M;    // rows of A and C
    int N;    // cols of B and C
    int K;    // cols of A and rows of B
    int lda;  // leading dimension of A
    int ldb;  // leading dimension of B
    int ldc;  // leading dimension of C
} gemm_config_t;
```

`M`, `N`, `K` are the logical dimensions of the operation. The leading dimensions `lda`, `ldb`, `ldc` tell us the actual stride between rows in memory — these can be larger than the logical dimension, which becomes important when we talk about padding later.

---

### i-j-k Loop Order Matmul

Let's start with the most natural translation of the mathematical definition into code:

```c
void sgemm_ijk(const float *A, const float *B, float *C, 
               gemm_config_t *config) {

    int M = config->M, N = config->N, K = config->K;
    int lda = config->lda, ldb = config->ldb, ldc = config->ldc;

    for(int i = 0; i < M; i++) {
        for(int j = 0; j < N; j++) {
            for(int k = 0; k < K; k++) {
                C[i*N + j] += A[i*K + k] * B[k*N + j];
            }
        }
    }
}
```

Benchmarked on Apple M5 MacBook Air (N=1024):

**Average GFLOP/s: 0.6964**

That's a poor number. Let's understand exactly why.

---

### Why ijk Performs So Poorly, a Cache Level Walkthrough

##### Assumptions for this walkthrough:

- One cache line = **64 bytes** = **16 floats** (standard on modern CPUs)
- Matrix dimensions: **4 × 4** (M = N = K = 4)
- Storage order: **row-major** (C default)
- Data type: **float32** (4 bytes each)

In row-major layout, a 4×4 matrix A looks like this in memory:

```c
[A00 A01 A02 A03]

[A10 A11 A12 A13]   ->  [A00 A01 A02 A03 A10 A11 A12 A13 ...]

[A20 A21 A22 A23]

[A30 A31 A32 A33]
```

Elements in the same **row** are adjacent in memory. Elements in the same **column** are N elements apart.

Now let's trace the inner loop. When i=0, j=0, k increments:

For matrix A, the access is `A[i*lda + k]`. As k increments:
```c
k=0: A[0*lda + 0] = A00  →  addr 0   sequential 

k=1: A[0*lda + 1] = A01  →  addr 1   sequential 

k=2: A[0*lda + 2] = A02  →  addr 2   sequential 

k=3: A[0*lda + 3] = A03  →  addr 3   sequential 
```
A is fine, we are walking across row 0, in a perfectly sequential manner. The entire row fits in one cache line and every element is reused without a miss.

Now for matrix B, the access is `B[k*ldb + j]`. As k increments:
```c 
k=0: B[0*ldb + 0] = B00  →  addr 0    cache hit 

k=1: B[1*ldb + 0] = B10  →  addr 4    cache miss,  jumped N=4 elements

k=2: B[2*ldb + 0] = B20  →  addr 8    cache miss,  jumped N=4 elements

k=3: B[3*ldb + 0] = B30  →  addr 12   cache miss,  jumped N=4 elements
```

**Every single inner loop iteration causes a cache miss on B.**

Why? Because as k increments, the B index is `B[k*ldb + j]` -> k increments the *row* of B. In row-major memory, moving to the next row means jumping N elements forward. For N=1024 that is a jump of 4096 bytes, far beyond a single 64-byte cache line.

The CPU fetches a cache line containing B00, B01, B02, B03, but the very next access needs B10 which is in a completely different cache line. The data we just fetched is mostly wasted.

This is a **spatial locality violation** meaning we are not using the data we brought into cache before it gets evicted. For a 1024×1024 matrix, this pattern repeats N² times, roughly one million cache misses in the innermost loop alone. The CPU spends most of its time waiting for data from memory rather than doing actual computation. Hence 0.6964 GFLOP/s.

---

### Fixing the Access Pattern — Padding + ikj Loop Order

Two things we fix in the next kernel.

**Fix level 1: Loop reorder to ikj.**

Swap the j and k loops. Now the inner loop variable is j. Let's check all three accesses:

Inner loop j, i=0, k=0:

C[i*N + j]  →  j increments column  →  +1 address, hence sequential 

A[i*K + k]  →  j does not appear    →  fixed address, hence stays in register 

B[k*N + j]  →  j increments column  →  +1 address, hence sequential

All three accesses are either sequential or fixed. No strided jumps inside the hot inner loop. The compiler also recognizes that `A[i*K + k]` is constant across the entire j loop and hoists it into a register, so the inner loop becomes a scalar multiplied against a sequential row of B, added into a sequential row of C. `This is the ideal pattern for the CPU's prefetcher and for SIMD vectorization.`

**Fix level 2: Padding the leading dimension.**
Before discussing leading dimensions, let's briefly touch on cache placement policies.

When the CPU fetches data from main memory, it needs to store it in the L1 cache for faster future access. But it cannot just place it anywhere. If data could go anywhere (a fully associative cache), the CPU would have to search the entire cache to find it again, which is extremely inefficient and slow for the hardware.

Instead, hardware uses a set-associative policy. The cache is divided into "sets." Every memory address is mathematically mapped to a specific set using a simple modulo operation based on the memory address.

Let's look at a realistic hardware example. A modern CPU L1 cache usually has 64 sets, and a single cache line holds 64 bytes. The hardware determines the target set for any memory address like this:

`Target Set = (Memory Address / 64 bytes) % 64 sets` 

**The Problem: Power-of-2 Dimensions (Without Padding)**

Let's say our matrix dimensions are M = N = K = 1024, and we are using float32 (4 bytes each).
-	Each row is exactly 1024 * 4 = 4096 bytes.
-	4096 bytes equals exactly 64 cache lines.

Let's see where the starting memory address of each row lands in the cache:

```
Row 0 starts at Address 0     -> (0 / 64) % 64     = 0 % 64   = Set 0
Row 1 starts at Address 4096  -> (4096 / 64) % 64  = 64 % 64  = Set 0
Row 2 starts at Address 8192  -> (8192 / 64) % 64  = 128 % 64 = Set 0
Row 3 starts at Address 12288 -> (12288 / 64) % 64 = 192 % 64 = Set 0
```

Every single row maps to Set 0!

When this happens, the CPU suffers from cache conflicts. Even though the other 63 sets in the L1 cache are sitting completely empty, the CPU keeps overwriting the data in Set 0. When it needs the old data again, it has to fetch it from main memory. This continuous cycle of fetching and evicting is called cache thrashing, meaning the CPU is spending a major chunk of its time shuffling data instead of performing mathematical computations.

**The Solution: Breaking the Alignment (With Padding)**

We can fix this by adding a small amount of padding to the leading dimension. Instead of 1024, let's set lda = ldb = ldc = 1024 + 16 = 1040. Adding 16 floats adds exactly 64 bytes (one full cache line) to the row stride.

- Each row is now 1040 * 4 = 4160 bytes.

Let's recalculate the mapping for the starting addresses with our new padded dimension:

```
Row 0 starts at Address 0     -> (0 / 64) % 64     = 0 % 64   = Set 0
Row 1 starts at Address 4160  -> (4160 / 64) % 64  = 65 % 64  = Set 1
Row 2 starts at Address 8320  -> (8320 / 64) % 64  = 130 % 64 = Set 2
Row 3 starts at Address 12480 -> (12480 / 64) % 64 = 195 % 64 = Set 3
```

By simply adding padding, the starting addresses now shift gracefully across the cache sets. The conflict misses disappear, cache thrashing is eliminated, and the CPU can finally utilize the entire L1 cache efficiently!

##### Performance Comparison 
Here's a comparison of the performance before and after adding padding:

![Padding VS Non-Padding](/public/padding_bench.png)

So as discussed and now proven, you can see that there is a major performance boost from adding padding to the leading dimension.
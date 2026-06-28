---
title: "Tiling"
description: "Here in this blog we will go through tiling and how it can be used to optimize performance for GEMM"
pubDate: "Jun 28 2026"
heroImage: https://images.unsplash.com/photo-1578926078693-4eb3d4499e43?w=900&auto=format&fit=crop&q=60&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxzZWFyY2h8MTE1fHxhcnR8ZW58MHx8MHx8fDA%3D
---

Tiling is a very smart and fine technique that you can opt for to optimize performance for GEMM.

To simply put it out there, tiling is a divide and conquer technique where you divide your matrix into sub tiles and then you operate on those tiles individually.

This is better in terms of performance because with tiling what you have achieved is a smaller problem space to work with. The matrix is divided so there is less data that you have to load in L1 cache which directly means that you have more space left in the cache for other data as well.

Now before we dive deep and I show you the code, let's first understand what are the various types of tiling as well.

### Types of Tiling

#### Macro Tiling

Macro tiling is you utilizing the cache locality by dividing your matrix into sub tiles and then loading them in cache to perform computations.

#### Micro Tiling

Micro tiling is you utiling the registers, which are more closer to the CPU and hence provide a faster read/write access, and you load the data from the matrix into the registers to furthermore optimize the computation.

Now what these are and how they work, we will discuss in detail through code.

---

### Macro Tiling

Macro tiling is `utilizing cache locality` by dividing your matrix into sub-tiles and then loading them into the cache to perform computations.

Following is the C code where macro tiling is performed on a matrix using the i-j-k loop order. For this first example, we will assume a perfect scenario: a perfectly square matrix where the dimensions are perfectly divisible by our chosen tile size.

M, N, and K are the dimensions of the matrices A, B, and C respectively.

```c
int tile_size = 8;

int M = 16, N = 16, K = 16;
int lda = M + 8, ldb = N + 8, ldc = K + 8;

for (int i = 0; i < M; i += tile_size) {
    int iEnd = i + tile_size;

    for (int j = 0; j < N; j += tile_size) {
        int jEnd = j + tile_size;

        for (int k = 0; k < K; k += tile_size) {
            int kEnd = k + tile_size;

            for (int ii = i; ii < iEnd; ii++) {
                for (int jj = j; jj < jEnd; jj++) {
                    for (int kk = k; kk < kEnd; kk++) {

                        C[ii * ldc + jj] += A[ii * lda + kk] * B[kk * ldb + jj];

                    }
                }
            }
        }
    }
}
```

Now, let's walk through the code and understand what each section does in detail.

#### 1. The Outer Three Loops

The sole purpose of the first three for loops is to jump from one tile to another.

If you look closely, i is not incremented by 1; rather, i is incremented by tile_size. So, if M = 16 and tile_size = 8, the row dimension of matrices A & C processed in the first iteration will be from 0 to 7. In the next iteration, i will increment by tile_size, moving the row focus from 8 to 15.

#### 2. iEnd, jEnd, kEnd Variables

These represent the strict upper limits of the current tile the CPU is operating on.

For Example: In the first iteration, **i** = 0, **iEnd** = 8, **j** = 0, **jEnd** = 8, **k** = 0, **kEnd** = 8.
Such that the inner loops won't go beyond these limits.

#### 3. The Inner Three Loops

These are the three nested loops that actually iterate through the specific elements inside the tiles of matrices A, B, and C selected by the outer loops. We access these memory locations using the ii, jj, and kk indices.


### Edge Tiling

If you observe the code above, it relies on a perfectly divisible problem space. The matrix is a square, the tiles are square, and 16 divides perfectly by 8.

But what if the tile_size was 7 instead of 8?

With the current code, the inner loops would attempt to access i + tile_size. On the final jump, this would push the index out of the matrix bounds, accessing invalid memory and causing a segmentation fault.

To handle this, we use edge tiling. We dynamically calculate the bounds of the tile to ensure it never exceeds the dimensions of the parent matrix.

```c
static inline int imin(int a, int b) { return (a < b) ? a : b; }

int main() {
    int tile_size = 7;
    int M = 16, N = 16, K = 16;
    int lda = M + 8, ldb = N + 8, ldc = K + 8;

    for (int i = 0; i < M; i += tile_size) {
        int iEnd = imin(i + tile_size, M);
        
        for (int j = 0; j < N; j += tile_size) {
            int jEnd = imin(j + tile_size, N);
            
            for (int k = 0; k < K; k += tile_size) {
                int kEnd = imin(k + tile_size, K);

                for (int ii = i; ii < iEnd; ii++) {
                    for (int jj = j; jj < jEnd; jj++) {
                        for (int kk = k; kk < kEnd; kk++) {

                            C[ii * ldc + jj] += A[ii * lda + kk] * B[kk * ldb + jj];

                        }
                    }
                }
            }
        }
    }
    return 0;
}
```

By taking the minimum of **(current_index + tile_size) and the matrix_dimension** using the imin function, we ensure that if a tile sits on the "edge" of the matrix, it cleanly truncates itself. This prevents us from accessing invalid memory while still giving us the performance benefits of tiling.

### Rectangular Tiling
So far, we have only looked at square tiles. However, hardware caches and SIMD vector registers are rarely perfectly square in how they want to digest data. Often, it is mathematically highly beneficial to map the problem space into rectangular tiles.

Here is how we adapt our code to handle rectangular boundaries, utilizing separate variables for the tile dimensions.

```c
static inline int imin(int a, int b) { return (a < b) ? a : b; }

int main() {
    int M = 16, N = 16, K = 16;
    int lda = M + 8, ldb = N + 8, ldc = K + 8;
    
    // Rectangular tile dimensions
    int tM = 8, tN = 4, tK = 2;

    for (int i = 0; i < M; i += tM) {
        int iEnd = imin(i + tM, M);
        for (int j = 0; j < N; j += tN) {
            int jEnd = imin(j + tN, N);
            for (int k = 0; k < K; k += tK) {
                int kEnd = imin(k + tK, K);

                for (int ii = i; ii < iEnd; ii++) {
                    for (int jj = j; jj < jEnd; jj++) {
                        for (int kk = k; kk < kEnd; kk++) {

                            C[ii * ldc + jj] += A[ii * lda + kk] * B[kk * ldb + jj];

                        }
                    }
                }
            }
        }
    }
    return 0;
}
```

In this setup, the tile sizes are no longer uniform.

- The tile size for matrix A is tM X tK (8 X 2).
- The tile size for matrix B is tK X tN (2 X 4).
- The tile size for matrix C is tM X tN (8 X 4).

Because iEnd, jEnd, and kEnd are already set up dynamically with imin, mixing and matching rectangular dimensions works flawlessly right out of the box without changing the inner logic.

### Micro Tiling

While macro tiling optimizes our usage of the L1 cache, micro tiling (often called register blocking) optimizes our usage of the CPU's registers. Registers are the absolute fastest memory available, sitting directly inside the execution cores.

Let's look at the innermost loop of our previous code:
```c
for (int kk = k; kk < kEnd; kk++) {
    C[ii * ldc + jj] += A[ii * lda + kk] * B[kk * ldb + jj];
}
```

Every time this loop runs, the CPU has to execute a load instruction to fetch C[ii * ldc + jj], do the math, and then execute a store instruction to write it back to memory. If the loop runs 8 times, that is 8 loads and 8 stores for the exact same memory address!

#### Level 1: The Single Register Accumulator

We can drastically improve this by utilizing temporal locality. We load the value of C once into a local variable (which the compiler will place in a CPU register), do all our accumulations on that register, and write it back to memory only when the loop finishes.

```c
float c_acc = C[ii * ldc + jj]; // Loading once into c_acc

for (int kk = k; kk < kEnd; kk++) {
    c_acc += A[ii * lda + kk] * B[kk * ldb + jj]; 
}
C[ii * ldc + jj] = c_acc; // Writing back to memory only once
```

Just like that, we eliminated a massive amount of redundant read/write traffic to the L1 cache.

#### Level 2: 2D Register Blocking

But why stop at one register? Modern CPUs have many architectural registers (for instance, ARM has 32 vector registers, and x86 AVX-512 has 32 massive 512-bit registers).

Instead of computing one single element of C at a time, we can compute a small grid of C (like a 4x4 or 8x8 block) simultaneously by unrolling our loops and keeping multiple accumulators in our registers.

Here is a conceptual example of a 2x2 micro-tile:

```c
// 4 registers holding a 2x2 tile of C
float c00 = C[ii*ldc + jj];
float c01 = C[ii*ldc + jj+1];
float c10 = C[(ii+1)*ldc + jj];
float c11 = C[(ii+1)*ldc + jj+1];

for (int kk = k; kk < kEnd; kk++) {
    // Load elements of A and B into registers
    float a0 = A[ii*lda + kk];
    float a1 = A[(ii+1)*lda + kk];
    float b0 = B[kk*ldb + jj];
    float b1 = B[kk*ldb + jj+1];

    // Compute the 2x2 block
    c00 += a0 * b0;
    c01 += a0 * b1;
    c10 += a1 * b0;
    c11 += a1 * b1;
}

// Store the 2x2 block back to memory
C[ii*ldc + jj] = c00;
C[ii*ldc + jj+1] = c01;
C[(ii+1)*ldc + jj] = c10;
C[(ii+1)*ldc + jj+1] = c11;
```

By doing this, we maximize our arithmetic intensity. We load a few elements of A and B, but we reuse them multiple times across our grid of accumulators before they leave the registers.

#### Panel Packing: Feeding the Registers efficiently

Micro tiling gives us a massive speedup, but it introduces a new problem.

To feed our 2x2 or 4x4 micro-tiles, we have to read small chunks of matrix A and matrix B. Because these matrices are stored in row-major order, accessing a 4x4 block means reading 4 elements, jumping across the entire matrix to the next row, reading 4 more, jumping again, etc.

These constant strides completely destroy our spatial locality. It causes TLB (Translation Lookaside Buffer) misses and ruins the CPU's automatic hardware prefetcher, which prefers long, contiguous streams of data.

#### The Solution: Packing

Since we know exactly which macro-tiles of A and B we are going to operate on, we can copy them into a temporary, contiguous block of memory before we run our micro-kernel. This is known as Panel Packing (or Data Packing).

We rearrange the data from its standard row-major layout into a custom layout that exactly matches the order in which our micro-kernel will request it.

1.	Pack A: We take a block of A and copy its elements into a sequential array, grouped by the micro-tile row size.
2.	Pack B: We take a block of B and copy its elements into a sequential array, grouped by the micro-tile column size.
3.	Compute: We pass these newly packed, contiguous arrays into our innermost loops.

While packing takes a little bit of overhead time, the time saved by having zero cache misses and perfectly sequential memory accesses in the highly repetitive inner loops pays off immensely, allowing the CPU to hit its theoretical maximum GFLOP/s.
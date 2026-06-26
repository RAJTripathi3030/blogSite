---
title: 'Basic Linear Algebra Subprograms (BLAS)'
description: 'Here in this log we will go through what BLAS is and inside it what is GEMM and how we observe and measure low-level optimizations for performance for various subroutines'
pubDate: 'Jun 26 2026'
heroImage: https://images.unsplash.com/photo-1600348712270-5af9e3590f66?w=900&auto=format&fit=crop&q=60&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxzZWFyY2h8MTl8fHByb2Nlc3NvcnxlbnwwfHwwfHx8MA%3D%3D
---

## What is BLAS?
BLAS is essentially a blueprint that was given to the world for the sole purpose of standardizing the interface of BLAS subroutines. 

It does not tell you what your program must do, rather it tells what standard input and output must look like if you want to follow the BLAS standard.

### From Where Did It's Need Came?

- #### The Problem
    Back in 1970s, scientists were doing heavy math in Fortran. Everytime a physicist or a scientist needed to multiply two matrices, they wrote their own custom `For Loop` implementaition. 
*The problem* with this implementation was that say a particular implementation was working good in an IBM Supercomputer, the same implementation could do terrible in a Cray Supercomputer.

    This meant everytime they migrated from one machine to other, they had to keep in mind about the change in the underlying architecture of the machine they are migrating to so that they can write performance efficient code for that machine.

    This was obviously a big headache for the scientists because now an overhead of performance optimization was introduced which they HAD to keep in their mind.

- #### The Solution(1979)
    Then came forth a group of mathematicians and computer scientists - most notably *Charles Lawson*, *Richard Hanson*, *David Kincaid*, *Fred Krogh*, and later a legendary computer scientist named *Jack Dongarra* (who actually won the Turing Award, largely for his work in this field).

    In 1979 they published a paper that introduced this BLAS framework. It *essentially* said:

    *"Stop writing your own math loops. Let's agree on a standard list of function signatures. We will ask the supercomputer manufacturers to write the hardware-specific ultra-optimized implementations for these functions. You, the scientist, just call the standard name, and the hardware guys will handle the speed."*

### Functionality
BLAS functionality is categorized into three sets of routines called *levels*. These levels correspond to the chronological order of definition & publication as well as the degree of polynomial in the complexities of the algorithms.

- ##### Level-1: Linear Complexity
    This level consists of all the subroutines defined in the original presentation of BLAS(1979) which defined only vector operations on strided arrays.
    For example: `axpy` called ax plus y (y <- ax + y) and several other operations.

- #### Level-2: Quadratic Complexity
    This level consists of Matrix-Vector operations including a **G**eneralized **M**atrix-**V**ector Multiplication `(gemv)` along with other things as well.

    Design of the Level 2 BLAS started in 1984 with results published in 1988. This level subroutines in BLAS were intended to improve performance on `vector processors`, where level-1 BLAS are suboptimal.

- #### Level-3: Cubic Complexity
    This level, published in 1990, contains *matrix-matrix* operations including a **g**eneral **m**atrix-**m**ultiplication `(gemm)` of the form;

    C <- *α*AB + *β*C

    Where A and B can be optionally *Transposed* and all three matrices may be strided. The ordinary matrix-multiplication is performed by setting alpha to 1 and C to an all-zeros matrix of appropriate size.
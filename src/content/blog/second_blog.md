---
title: 'Why CPU and GPU Give Different Results in PyTorch: A Deep Dive'
description: 'Here I tell you guys what I found out and learned in my first contribution to pytorch'
pubDate: 'Apr 15 2026'
heroImage: 'https://images.unsplash.com/photo-1741392078467-a2d39da36f2d?w=500&auto=format&fit=crop&q=60&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxzZWFyY2h8NzJ8fGdwdXxlbnwwfHwwfHx8MA%3D%3D'
---
# I Tried Contributing to PyTorch on Day 1. Here's What Happened.

I cloned PyTorch, found an open issue, ran someone's reproduction script, and got completely different results than the person who filed the bug. Not a crash. Not an error. Just wrong numbers — silently, confidently wrong.

That rabbit hole took me somewhere I didn't expect.

---

## Why PyTorch of All Frameworks?

Before I get into the story, let me answer the obvious question — why PyTorch?

There are plenty of open source projects out there. But PyTorch is the one that felt right, and here's why:

**Scale and impact.** PyTorch is used by virtually every major AI research lab — Meta, Google DeepMind, Microsoft, and hundreds of universities worldwide. Most cutting-edge AI papers implement in PyTorch first. If you contribute here, your work touches real research.

**Depth of learning.** Using PyTorch as an API is one thing. Reading and contributing to its source forces you to understand things at a completely different level — CUDA internals, C++ extensions, floating point behavior, how deep learning *actually* works under the hood. You can't fake that.

**Career signal.** A contribution to `pytorch/pytorch` on your GitHub is a strong signal. The codebase is notoriously complex and reviewers are strict. It stands out significantly more than contributing to a smaller repo.

**It's the industry standard.** TensorFlow has been losing ground for years. PyTorch dominates both research and increasingly production. Hugging Face, PyTorch Lightning, and most modern ML tooling is built on top of it.

**Timing.** AI is at its peak relevance right now. A contribution to the foundational framework of modern AI carries more weight today than it ever did before.

> I suppose all these reasons are enough for you to understand why I chose PyTorch as my open-source starter.

---

## Getting Started — Finding the Right Issue

The first thing I did was go to the [PyTorch GitHub issues page](https://github.com/pytorch/pytorch/issues) and filter by `good first issue`.

What I quickly learned: **popular repos get swarmed fast.**

Almost every `good first issue` that was recent had 5-6 linked Pull Requests already. That little PR icon on the right side of each issue? That's your signal. If it has linked PRs, other people are already working on it — skip it.

The issues with zero linked PRs were mostly from 2023. Old, possibly stale, possibly already fixed.

So I changed my strategy. Instead of hunting for `good first issue` with no competition, I filtered for issues tagged `needs reproduction`. These are issues where a maintainer needs someone to:

1. Run the provided reproduction script
2. Confirm whether they can reproduce the bug
3. Report their environment details

That's exactly what I wanted to do on day one — explore, understand, and contribute without needing to know the entire codebase.

---

## The Issue I Picked

I found this issue filed 3 days prior:

**`cpu= -inf and gpu=-108.34196472167969`**

The reporter shared this code:

```python
import torch
import torch.nn as nn

class Model(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(8, 16)
        self.fc2 = nn.Linear(16, 4)

    def forward(self, x):
        y = self.fc1(x)
        v = torch.arccosh(y + 2)
        u = self.fc2(v)
        return torch.logdet(u), u.reciprocal_()

torch.manual_seed(42)
m_cpu = Model().eval()
torch.manual_seed(42)
m_gpu = Model().cuda().eval()
m_gpu.load_state_dict(m_cpu.state_dict())

x = torch.full((4, 8), 0.4033372700214386)

with torch.no_grad():
    cpu_log, cpu_u = m_cpu(x)
    gpu_log, gpu_u = [o.cpu() for o in m_gpu(x.cuda())]

print("CPU logdet:", cpu_log.item())
print("GPU logdet:", gpu_log.item())
```

Their output:
```
CPU logdet: -inf
GPU logdet: -108.34196472167969
```

Same model. Same weights. Same input. Different results. One is negative infinity, the other is a finite number.

Before diving in, let me break down what this code actually does.

---

## What Does This Code Actually Do?

### Setting up identical models

```python
torch.manual_seed(42)
m_cpu = Model().eval()
torch.manual_seed(42)
m_gpu = Model().cuda().eval()
m_gpu.load_state_dict(m_cpu.state_dict())
```

This block ensures both models have **identical weights**. Here's how:

- `torch.manual_seed(42)` — sets the random seed so weight initialization is deterministic
- `.eval()` — switches off dropout and batch normalization training behavior
- The seed is reset to 42 before creating the GPU model, so weights initialize identically
- `load_state_dict` — forcefully copies CPU weights into the GPU model, as an extra guarantee

### The forward pass

```python
y = self.fc1(x)          # Linear(8 → 16): input shape (4,8) → output (4,16)
v = torch.arccosh(y + 2) # arccosh needs input > 1, so we add 2
u = self.fc2(v)           # Linear(16 → 4): output shape (4,4) — a square matrix
return torch.logdet(u), u.reciprocal_()
```

`nn.Linear` applies the transformation **y = xWᵀ + b** — multiplies input by a learned weight matrix and adds a bias. It's the most fundamental building block in a neural network.

`torch.arccosh` (same as `torch.acosh`, it's just an alias) is the inverse hyperbolic cosine. The `+2` is just to ensure inputs satisfy the domain requirement (arccosh is only defined for values ≥ 1).

`torch.logdet` computes the **log of the determinant** of a matrix. For a square matrix `u` of shape (4×4), it returns a single scalar.

### Running inference

```python
with torch.no_grad():
    cpu_log, cpu_u = m_cpu(x)
    gpu_log, gpu_u = [o.cpu() for o in m_gpu(x.cuda())]
```

`torch.no_grad()` disables gradient tracking — we're doing inference, not training, so no need for it.

`x.cuda()` moves the input to GPU. The list comprehension `[o.cpu() for o in ...]` moves both GPU outputs back to CPU for comparison — you can't directly compare a CPU tensor and a GPU tensor in PyTorch.

---

## Reproducing the Bug

I set up a clean virtual environment:

```bash
python -m venv pyt_venv
source pyt_venv/bin/activate
pip install torch numpy
```

Ran the script. My output:

```
CPU logdet: -111.53960418701172
GPU logdet: -108.34196472167969
CPU is inf: False  GPU is nan: False
```

Interesting. The original reporter got `-inf` on CPU. I got `-111.53` — a finite number. But my CPU and GPU results still **don't match**.

The discrepancy is there. Just not as dramatic as the original report.

I ran `torch.utils.collect_env` to capture my environment:

```
PyTorch: 2.11.0+cu130
Python: 3.14.3
OS: Arch Linux (x86_64)
GPU: NVIDIA GeForce GTX 1650 Ti
CUDA: 13.0
```

I posted my findings as a comment on the issue. That was my contribution for the day.

---

## Going Deeper — Why Do CPU and GPU Give Different Results?

This is the part that I found genuinely fascinating.

My first guess was the `reciprocal_()` call — the underscore suffix in PyTorch means **in-place operation**, meaning it modifies the tensor directly in memory rather than creating a new one.

```python
return torch.logdet(u), u.reciprocal_()
```

Both `logdet` and `reciprocal_` are operating on the same tensor `u`. On CPU, operations are sequential — `logdet` finishes completely before `reciprocal_` runs. On GPU, CUDA kernels launch asynchronously — `reciprocal_()` might modify `u` before `logdet` has fully finished reading it.

I tested this theory by removing the underscore:

```python
return torch.logdet(u), u.reciprocal()  # non in-place version
```

Still a mismatch. So that wasn't the root cause.

The real answer goes much deeper.

---

## LAPACK vs cuSOLVER — The Real Culprit

To understand why CPU and GPU compute `logdet` differently, you need to understand what's actually happening under the hood when PyTorch computes a matrix determinant.

### How logdet is computed

`torch.logdet` doesn't directly compute the determinant. For large matrices that would be catastrophically slow and numerically unstable. Instead it uses **LU decomposition**.

LU decomposition breaks a matrix **A** into two matrices:
- **L** — lower triangular (zeros above the diagonal)
- **U** — upper triangular (zeros below the diagonal)

Such that **A = L × U**

```
Original matrix A:        L (lower):      U (upper):
[4  3]          =       [1    0] ×      [4   3 ]
[6  3]                  [1.5  1]        [0  -1.5]
```

Once you have U, computing the determinant is trivial — it's just the product of U's diagonal elements. And log of a product is a sum of logs:

```
log(det(A)) = log(|u₁₁|) + log(|u₂₂|) + ... + log(|uₙₙ|)
```

This is fast, stable, and elegant. Both CPU and GPU use this approach. But they use *different libraries* to do it.

---

### LAPACK — the CPU workhorse

**LAPACK** (Linear Algebra PACKage) was written in Fortran in the 1970s and 80s. It sounds ancient, but it's still the gold standard for CPU linear algebra. Behind almost every scientific computation you've ever run on a CPU — NumPy, SciPy, MATLAB, R — there's LAPACK doing the heavy lifting.

LAPACK itself sits on top of **BLAS** (Basic Linear Algebra Subprograms), which handles the low-level operations like matrix-matrix multiply. On Intel CPUs, PyTorch typically uses **Intel MKL** (Math Kernel Library) as the BLAS implementation — heavily tuned for Intel's specific hardware pipeline and cache architecture.

How LAPACK computes LU decomposition:

```
Step 1: Find the largest element in column 1 (partial pivoting)
Step 2: Swap rows to bring it to the top (for numerical stability)
Step 3: Eliminate entries below it
Step 4: Move to column 2, repeat
Step 5: ... and so on, column by column
```

This is **sequential**. Each step depends on the result of the previous one. The operations happen in a fixed, deterministic order. Run it a thousand times, you get the exact same floating point result every time.

---

### cuSOLVER — the GPU equivalent

**cuSOLVER** is NVIDIA's library for dense and sparse linear algebra on CUDA GPUs. It's part of the CUDA toolkit and is what PyTorch uses when you call `torch.logdet` on a GPU tensor.

The fundamental difference: GPUs have thousands of small cores designed for parallel work. A modern NVIDIA GPU has anywhere from 2,000 to 16,000+ CUDA cores. cuSOLVER is designed to exploit this parallelism.

For LU decomposition on a GPU, cuSOLVER uses a **blocked algorithm** — it divides the matrix into tiles and processes multiple tiles concurrently across different CUDA cores.

```
CPU (LAPACK):                    GPU (cuSOLVER):
Process column 1                 Divide matrix into blocks
↓                                Process blocks in parallel
Process column 2                 ┌─────┬─────┐
↓                                │ B1  │ B2  │  ← cores working simultaneously
Process column 3                 ├─────┼─────┤
↓                                │ B3  │ B4  │
Process column 4                 └─────┴─────┘
Sequential, predictable          Parallel, fast, but order varies
```

---

### Why parallel = different results

Here's the key insight that most people miss: **floating point arithmetic is not associative**.

In real mathematics: `(a + b) + c = a + (b + c)` — always true.

In floating point arithmetic on a computer: **not always true.**

```python
a = 0.1
b = 0.2
c = 0.3

print((a + b) + c)  # 0.6000000000000001
print(a + (b + c))  # 0.6
```

This happens because floating point numbers are stored in binary with limited precision (64-bit double = 15-16 significant decimal digits). Each arithmetic operation introduces a tiny rounding error. The order in which you add numbers determines how those errors accumulate.

When cuSOLVER runs LU decomposition in parallel, different CUDA cores process different parts of the matrix, and their results get accumulated in whichever order the cores happen to finish. That order can vary between runs, between GPU models, between driver versions.

LAPACK processes everything sequentially in a fixed order. Same order every time. Same rounding errors every time. Same result every time.

Neither result is "wrong" — both are correct within the IEEE 754 floating point standard. They're just two different valid approximations of the true mathematical result.

---

### Why the original reporter got -inf

The more severe case — `-inf` on CPU, finite on GPU — suggests something beyond mere rounding differences.

`logdet` returns `-inf` when the determinant of the matrix is zero or effectively zero (log(0) = -∞). This means LAPACK's sequential computation arrived at a near-zero determinant while cuSOLVER's parallel computation did not.

This is a **numerical stability** edge case. The matrix `u` is close to singular (determinant near zero). Exactly *how close* the computation gets to zero depends on the precise order of floating point operations — and LAPACK's sequential order happened to produce a result that crossed below the threshold to zero, while cuSOLVER's parallel order didn't.

Different PyTorch versions, hardware, and BLAS implementations can all shift this threshold slightly, which is why I reproduced the discrepancy but not the `-inf` specifically.

---

## What the Maintainer Said

A PyTorch maintainer closed the issue with this message:

> *"Closing per the accuracy problems policy — do not hesitate to open a new one with a better argument for why this discrepancy is not something one should expect."*

Translation: CPU vs GPU floating point differences are **documented expected behavior** in PyTorch. The reporter didn't make a strong enough case that this specific discrepancy crosses the line from "expected numerical variance" to "actual bug."

The `-inf` case is actually the stronger argument — a finite value vs negative infinity is harder to dismiss as just rounding noise. But it wasn't framed that way in the original issue.

This taught me something important about how maintainers think: **they need you to prove it's a bug, not just show that two numbers differ**. The burden of proof is on the reporter to demonstrate the divergence exceeds acceptable tolerance, especially for numerical operations.

---

## What I Learned

**About open source:**
- Contribution isn't just code. Reproduction reports, triage, and clarifying environment details are real contributions that maintainers value.
- Issues in popular repos get swarmed fast — `needs reproduction` is a better entry point than `good first issue` for day one.
- Maintainers are strict but fair. Getting an issue closed isn't failure — it's a learning data point.

**About PyTorch and numerical computing:**
- CPU and GPU don't just run the same code faster — they use entirely different underlying libraries (LAPACK vs cuSOLVER) with fundamentally different execution models.
- Floating point arithmetic is not associative. The order of operations matters, and parallelism changes that order.
- "Correctness" in numerical computing is a spectrum, not a binary. Two different answers can both be mathematically valid.

**About my own process:**
- Setting up a clean environment for each reproduction attempt matters — it isolates variables.
- Reading the issue's comment thread carefully before jumping in saves you from working on something already in progress.
- The best contributions are specific: exact version numbers, exact output, exact hardware.

---

## What's Next

This was one issue, one comment, one day. But it cracked open a part of the codebase I wouldn't have touched otherwise. I now know what LAPACK and cuSOLVER are. I understand why GPU computations are non-deterministic. I've read PyTorch's policy on numerical accuracy.

That's not nothing.

Next I want to find an issue where I can actually write code — something in Python, close to the surface, where I can learn the contribution workflow end to end: fork, branch, fix, test, PR.

If you're a beginner thinking about open source, start exactly where I did. Find a `needs reproduction` issue, run the code, share your results. It's low stakes, high learning, and genuinely useful to the project.

The codebase isn't as scary as it looks from the outside.

---

*Written by Raj — Software Engineer, technical writer, and apparently someone who now knows more about Fortran linear algebra libraries than he expected to.*
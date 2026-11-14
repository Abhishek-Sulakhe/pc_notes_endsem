# OpenMP SIMD Vectorization & Latent Concurrency Bugs

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               08 — SIMD Vectorization, Multi-Level Parallelism & Latent Bugs
PREREQUISITES:          Subtopic 01.3 (SIMD & Vectorization Models), Subtopic 05.1 (Fork-Join),
                        Subtopic 05.4 (Loop Worksharing)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 25–26)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: Multi-Level Parallelism (Cores vs Vector Lanes)

Throughout Module 04 and Module 05, we parallelized code across **multiple CPU cores** using operating system threads (Thread-Level Parallelism — TLP).

However, modern microprocessors possess two distinct tiers of parallel hardware:
1. **Macro-Parallelism (Between Cores):** Multiple physical processor cores on a multi-core chip, each fetching and executing an independent instruction stream (MIMD).
2. **Micro-Parallelism (Inside Each Core):** Wide execution pipelines and vector execution units inside each individual core, where a single machine instruction operates simultaneously on a vector of data values (SIMD).

```
========================================================================================
                      THE TWO TIERS OF PROCESSOR PARALLELISM
========================================================================================

                 +-----------------------------------------------+
                 |              MULTI-CORE PROCESSOR             |
                 |                                               |
                 |  +--------------------+   +-----------------+ |
                 |  |      CORE 0        |   |     CORE 1      | |
                 |  | [ Thread 0 (TLP) ] |   | [ Thread 1 ]    | |
                 |  |                    |   |                 | |
                 |  |  SIMD VECTOR UNIT  |   | SIMD VECTOR UNIT| |
                 |  |  +---+---+---+---+ |   | +---+---+---+---+ |
                 |  |  |V0 |V1 |V2 |V3 | |   | |V0 |V1 |V2 |V3 | |
                 |  |  +---+---+---+---+ |   | +---+---+---+---+ |
                 |  +--------------------+   +-----------------+ |
                 +-----------------------------------------------+
                 
  Tier 1: Thread-Level Parallelism (TLP) -> Spreads work across Cores
  Tier 2: Data-Level Parallelism (SIMD)  -> Exploits Vector Registers inside Cores
========================================================================================
```

If you only use `#pragma omp parallel for`, you utilize all your CPU cores, but each core executes standard 64-bit scalar instructions, **leaving up to $80-90\%$ of the physical ALU execution silicon inside each core completely unutilized**.

To exploit the vector hardware inside each core, OpenMP 4.0 introduced **SIMD Directives**.

---

## 2. Vector Register Standards & The SIMD Factor

A **Vector Register** is an ultra-wide CPU register that holds multiple independent primitive data elements packed side-by-side.

### Evolution of x86 Vector Standards

```
+-----------------------------------------------------------------------------------+
|                        HARDWARE VECTOR REGISTER STANDARDS                         |
+-----------------------------------------------------------------------------------+
| Architecture Standard | Register Bit Width | Single-Precision (32b)| Double-Precision (64b)|
| :-------------------- | :----------------- | :-------------------- | :-------------------- |
| **SSE / SSE4**        | **128 bits**       | 4 floats              | 2 doubles             |
| **AVX / AVX2**        | **256 bits**       | 8 floats              | 4 doubles             |
| **AVX-512**           | **512 bits**       | 16 floats             | 8 doubles             |
| **ARM NEON**          | **128 bits**       | 4 floats              | 2 doubles             |
| **ARM SVE**           | Up to 2048 bits    | Scalable              | Scalable              |
+-----------------------------------------------------------------------------------+
```

### The SIMD Factor Formula

As presented in your lecture notes (`3d773f8d.pdf`, Page 25):

$$\boxed{\text{SIMD Factor} = \frac{\text{Vector Register Width (bits)}}{\text{Data Element Size (bits)}}}$$

- **Example 1:** Processing 32-bit `float` numbers using AVX2 (256 bits):
  $$\text{SIMD Factor} = \frac{256}{32} = 8 \text{ elements executed in 1 CPU cycle!}$$
- **Example 2:** Processing 64-bit `double` numbers using AVX-512 (512 bits):
  $$\text{SIMD Factor} = \frac{512}{64} = 8 \text{ double-precision calculations in 1 CPU cycle!}$$

---

## 3. The `#pragma omp simd` Directive: Overriding Compiler Timidity

Modern optimizing compilers (such as GCC with `-O3`) contain an internal **Auto-Vectorizer**. The compiler inspects for-loops and attempts to automatically rewrite them into vector assembly instructions.

### The Problem: Compiler Conservatism & Pointer Aliasing

Compilers are legally bound to **never break program correctness**. If the compiler suspects that an array write *might* overlap with another array read (known as **Pointer Aliasing**), or if it cannot mathematically prove that loop iterations are completely independent, the auto-vectorizer **gives up** and falls back to slow sequential scalar execution.

### The Solution: Programmer Guarantee via `#pragma omp simd`

$$\boxed{\textbf{\#pragma omp simd: } \text{Informs the compiler: \textit{"I guarantee that this loop is safe to vectorize.}}$$
$$\boxed{\text{\textit{Bypass all conservative dependency checks and emit SIMD vector instructions immediately!"}}}}$$

```c
#pragma omp simd
for (int i = 0; i < N; i++) {
    c[i] = a[i] + b[i]; // Vectorized via AVX/AVX-512 instructions!
}
```

---

## 4. SIMD Clauses: Precision Vector Tuning

To maximize vector instruction efficiency, OpenMP provides dedicated SIMD clauses:

```
+-----------------------------------------------------------------------------------+
|                            OPENMP SIMD TUNING CLAUSES                             |
+-----------------------------------------------------------------------------------+
| Clause                       | Purpose & Hardware Action                          |
| :--------------------------- | :------------------------------------------------- |
| **`safelen(N)`**             | Informs the compiler that loop-carried dependencies|
|                              | do not exist for distances $< N$. (Allows vector-  |
|                              | izing loops with short distances).                 |
| **`simdlen(N)`**             | Requests a specific vector lane width (e.g., 4 or 8|
|                              | elements per vector).                              |
| **`aligned(ptr : alignment)`**| Informs the compiler that array `ptr` is aligned   |
|                              | to a 32-byte or 64-byte memory boundary, allowing  |
|                              | fast aligned loads (`vmovapd`) instead of slow     |
|                              | unaligned loads (`vmovupd`).                       |
| **`linear(var : step)`**     | Declares that variable `var` increments linearly by|
|                              | `step` on every loop iteration.                    |
| **`reduction(op : var)`**    | Accumulates across vector lanes before horizontally|
|                              | reducing to the scalar variable.                   |
+-----------------------------------------------------------------------------------+
```

---

## 5. Combined Multi-Level Parallelism: `#pragma omp parallel for simd`

Why choose between multi-threading and vectorization when you can have both?

OpenMP allows combining multi-core thread worksharing with single-core SIMD vectorization into a single unified construct:

$$\boxed{\textbf{\#pragma omp parallel for simd}}$$

```c
#pragma omp parallel for simd schedule(static)
for (int i = 0; i < N; i++) {
    c[i] = a[i] + alpha * b[i];
}
```

```
========================================================================================
            EXECUTION PIPELINE: COMBINED MULTI-CORE + VECTOR PARALLELISM
========================================================================================

  Total Loop Iterations (N = 1,000,000)
             |
             +-----------------------+-----------------------+
             | Level 1: TLP          | (Thread Worksharing)  |
             v                       v                       v
        [ Thread 0 ]            [ Thread 1 ]            [ Thread 2 ]
      Iterations 0..333k     Iterations 333k..666k   Iterations 666k..1M
             |                       |                       |
             +-----------------------+-----------------------+
             | Level 2: SIMD         | (Vector Registers)    |
             v                       v                       v
      [ AVX2 Registers ]      [ AVX2 Registers ]      [ AVX2 Registers ]
     8 floats per cycle!     8 floats per cycle!     8 floats per cycle!
========================================================================================
```

---

### The Performance Trade-Off: When Does Parallel + SIMD Slow Down?

As noted in your lecture notes (`3d773f8d.pdf`, Page 26):
> *"Parallel + SIMD can take more time than only SIMD due to thread creation/join overhead and synchronization time."*

```
+-------------------------------------------------------------------------------+
|                       WHEN TO USE SIMD vs PARALLEL FOR SIMD                   |
+-------------------------------------------------------------------------------+
| 1. SMALL TO MEDIUM PROBLEM SIZES (N < 100,000):                               |
|    - SIMD alone (`#pragma omp simd`) runs with ZERO thread-creation overhead. |
|    - Spawning a team of 16 OS threads to compute a short loop incurs several  |
|      microseconds of fork-join overhead, making multi-threading SLOWER than   |
|      running vector instructions on a single core!                            |
|                                                                               |
| 2. MEMORY-BANDWIDTH SATURATED LOOPS (STREAM / DAXPY):                         |
|    - As proven in Subtopic 03.4, if a single core's vector pipeline already   |
|      consumes 80% of DRAM bandwidth, adding 16 cores will only cause bus      |
|      contention, yielding zero additional speedup.                            |
|                                                                               |
| 3. MASSIVE COMPUTE-BOUND MATRICES (N > 10,000,000):                           |
|    - Combined `#pragma omp parallel for simd` delivers maximum hardware peak, |
|      multiplying multi-core scaling by the SIMD factor!                       |
+-------------------------------------------------------------------------------+
```

---

## 6. Latent Concurrency & Vectorization Bugs

One of the most profound examination topics from `3d773f8d.pdf` (Page 26) is the concept of a **Latent Bug**:

$$\boxed{\textbf{Latent Bug: } \text{A bug that exists in code, but does not always show up immediately;}}$$
$$\boxed{\text{it remains hidden and manifests only under specific runtime conditions or optimization flags.}}$$

```
+-------------------------------------------------------------------------------+
|                   CONDITIONS WHERE LATENT BUGS REMAIN HIDDEN                  |
+-------------------------------------------------------------------------------+
| 1. AT LOW OPTIMIZATION LEVELS (-O0, -O1):                                     |
|    - The compiler does not vectorize; loop runs sequentially element-by-      |
|      element. The bug remains hidden and output looks correct!                |
|                                                                               |
| 2. ON CPU ARCHITECTURES LACKING VECTOR UNITS:                                 |
|    - On an embedded processor without AVX, compiler falls back to scalar C.   |
|                                                                               |
| 3. WHEN COMPILER SILENTLY REJECTS VECTORIZATION:                              |
|    - If the compiler chooses not to vectorize, the bug never triggers.        |
|                                                                               |
| 4. WHEN PROBLEM SIZE N IS SMALL:                                              |
|    - When N is less than the vector lane width, no vector instructions run.   |
+-------------------------------------------------------------------------------+
```

---

### The Latent Bug Scenario: Missing `private(tmp)` in SIMD Loops

Consider this buggy loop:

```c
// DANGEROUS CODE CONTAINING A LATENT BUG:
double tmp;

#pragma omp simd // Missing private(tmp)!
for (int i = 0; i < N; i++) {
    tmp = a[i] * 2.0;
    b[i] = tmp + c[i];
}
```

```
========================================================================================
                      HOW THE LATENT BUG EXPLODES IN HARDWARE
========================================================================================

Sequential Execution (No Vectorization, -O0):
  Iteration 0: tmp = a[0]*2.0; b[0] = tmp + c[0];  (Correct!)
  Iteration 1: tmp = a[1]*2.0; b[1] = tmp + c[1];  (Correct!)
  Output is 100% correct! The programmer thinks the code is flawless.

SIMD Vector Execution (-O3 -fopenmp, 4 Vector Lanes):
  The vector unit attempts to process iterations 0, 1, 2, and 3 simultaneously.
  Because `tmp` was not declared `private`, what happens?
  - All 4 vector lanes write into a single scalar storage location!
  - Vector registers hold intermediate values simultaneously; one lane's calculation
    overwrites or cross-contaminates another lane's temporary state.
  - The calculation silently produces corrupted garbage numbers!
========================================================================================
```

$$\boxed{\textbf{The Rule: } \text{Always explicitly declare temporary scalar variables inside the loop}}$$
$$\boxed{\text{or scope them using \texttt{private(tmp)} on the \texttt{\#pragma omp simd} directive!}}$$

---

### The Loop-Carried Dependency Hazard

What happens if you place `#pragma omp simd` on a loop that contains a **Loop-Carried Data Dependency**?

```c
// ILLEGAL VECTORIZATION: FORWARD DATA DEPENDENCY
#pragma omp simd
for (int i = 0; i < N - 1; i++) {
    a[i] = a[i + 1] + b[i]; // Iteration i depends on value at i+1
}
```

- When compiled without SIMD, iteration $0$ reads `a[1]`, updates `a[0]`. Then iteration $1$ reads `a[2]`, updates `a[1]`.
- When vectorized with 4 lanes: Iterations $0, 1, 2, 3$ execute in parallel in the vector register. If an operation reads memory that another lane is simultaneously updating, the hardware violates sequential semantics!
- Because you forced `#pragma omp simd`, **the compiler obeyed your order blindly and bypassed all safety checks**. You created corrupted numerical data.

---

## 7. Compiler Optimization & Vectorization Diagnostics

To verify whether your loops are actually being vectorized by GCC or Clang, use compiler vectorization reporting flags:

```bash
# Compile with aggressive Level 3 optimizations + OpenMP:
gcc -O3 -fopenmp program.c -o program

# Enable detailed vectorizer reporting:
gcc -O3 -fopenmp -fopt-info-vec program.c -o program
```

### Understanding Compiler Output
```
program.c:45:3: note: loop vectorized
program.c:45:3: note: loop versioned for alignment because of possible aliasing
program.c:82:3: missed: couldn't vectorize loop: loop-carried data dependency
```

### Architectural Tuning Flags (`-march`)

From `3d773f8d.pdf` (Page 25):
- **`-march=x86-64`:** Safe generic baseline. Emits only basic SSE instructions compatible with any 64-bit computer manufactured after 2004.
- **`-march=haswell`:** Enables AVX2, FMA (Fused Multiply-Add), and 256-bit vector registers.
- **`-march=native`:** Directs the compiler to query the host machine's physical CPU and activate **every single instruction extension supported by that exact silicon** (AVX2, AVX-512, FMA, BMI2). Delivers highest performance on local hardware.

---

## 8. Complete Compilable C Program: OpenMP SIMD & Vectorization Benchmarking

Here is the complete C program demonstrating `#pragma omp simd`, memory alignment, and compiler vectorization diagnostics:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_simd_vectorization.c
 * DESCRIPTION: Demonstrates OpenMP SIMD vectorization, memory alignment,
 *              and combined multi-core + vector execution (#pragma omp parallel for simd).
 * COMPILATION: gcc -O3 -fopenmp -fopt-info-vec -march=native openmp_simd_vectorization.c -o omp_simd
 * USAGE:       ./omp_simd
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define ARRAY_SIZE 10000000 // 10 Million elements
#define ALIGNMENT 64        // 64-byte alignment for AVX-512 cache lines

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("         OPENMP SIMD VECTORIZATION & COMBINED PARALLELISM       \n");
    printf("   Array Size: %d elements | Memory Alignment: %d bytes\n", 
           ARRAY_SIZE, ALIGNMENT);
    printf("================================================================\n\n");

    // Allocate 64-byte aligned memory for maximum vector load efficiency
    // posix_memalign guarantees memory begins on a 64-byte cache boundary
    float *a, *b, *c;
    int rc1 = posix_memalign((void**)&a, ALIGNMENT, ARRAY_SIZE * sizeof(float));
    int rc2 = posix_memalign((void**)&b, ALIGNMENT, ARRAY_SIZE * sizeof(float));
    int rc3 = posix_memalign((void**)&c, ALIGNMENT, ARRAY_SIZE * sizeof(float));

    if (rc1 != 0 || rc2 != 0 || rc3 != 0) {
        fprintf(stderr, "Failed to allocate aligned memory.\n");
        exit(EXIT_FAILURE);
    }

    // Initialize arrays
    #pragma omp parallel for
    for (int i = 0; i < ARRAY_SIZE; i++) {
        a[i] = 1.5f;
        b[i] = 2.5f;
        c[i] = 0.0f;
    }

    // ------------------------------------------------------------------------
    // EXPERIMENT 1: PURE SIMD VECTORIZATION (Single Core, Vector Units)
    // ------------------------------------------------------------------------
    printf("--- EXPERIMENT 1: Single Core Vectorization (#pragma omp simd) ---\n");
    double start_simd = omp_get_wtime();

    // Informs compiler memory is aligned to 64 bytes and loop is safe to vectorize
    #pragma omp simd aligned(a, b, c : 64)
    for (int i = 0; i < ARRAY_SIZE; i++) {
        c[i] = a[i] * 2.0f + b[i];
    }

    double time_simd = omp_get_wtime() - start_simd;
    printf("  [Result] Sample c[0] = %.2f\n", c[0]);
    printf("  [Result] Elapsed Time (SIMD Only): %.6f seconds\n\n", time_simd);

    // ------------------------------------------------------------------------
    // EXPERIMENT 2: COMBINED MULTI-CORE + SIMD (Multi-Level Parallelism)
    // ------------------------------------------------------------------------
    printf("--- EXPERIMENT 2: Combined Multi-Core + SIMD (#pragma omp parallel for simd) ---\n");
    double start_combined = omp_get_wtime();

    #pragma omp parallel for simd schedule(static) aligned(a, b, c : 64)
    for (int i = 0; i < ARRAY_SIZE; i++) {
        c[i] = a[i] * 3.0f + b[i];
    }

    double time_combined = omp_get_wtime() - start_combined;
    printf("  [Result] Sample c[0] = %.2f\n", c[0]);
    printf("  [Result] Elapsed Time (Combined):  %.6f seconds\n\n", time_combined);

    // ------------------------------------------------------------------------
    // CLEANUP
    // ------------------------------------------------------------------------
    free(a);
    free(b);
    free(c);

    printf("================================================================\n");
    printf("[Main] SIMD benchmark completed successfully!\n");
    printf("================================================================\n");

    return 0;
}
```

---

## 9. University Examination Preparation: Solved Questions

### Question 1: Core Architecture (10 Marks)
**Differentiate between Thread-Level Parallelism (TLP) and SIMD Vectorization. Define the SIMD Factor. How does `#pragma omp parallel for simd` exploit both levels of parallelism simultaneously?**

#### Model Examination Answer:
1. **TLP vs SIMD Vectorization:**
   - **Thread-Level Parallelism (TLP):** Macro-level parallelism across multiple physical CPU cores. Each core executes an independent instruction stream on its own program counter. Managed in OpenMP via `#pragma omp parallel for`.
   - **SIMD Vectorization:** Micro-level data parallelism inside a single CPU core. A single machine instruction operates across wide vector registers (e.g., AVX2 256-bit or AVX-512) to compute multiple data elements simultaneously in a single cycle. Managed via `#pragma omp simd`.
2. **SIMD Factor Definition:**  
   The SIMD factor is the number of data elements that fit into a hardware vector register:
   $$\text{SIMD Factor} = \frac{\text{Vector Register Width (bits)}}{\text{Data Element Size (bits)}}$$
   For example, an AVX2 256-bit register processing 32-bit floats has a SIMD Factor of $256/32 = 8$.
3. **How `#pragma omp parallel for simd` Exploits Both:**  
   The construct implements multi-level nested parallelism:
   - **Outer Level (TLP):** Iterations are divided into large chunks and distributed across physical CPU cores.
   - **Inner Level (SIMD):** The iteration chunk assigned to each core is decomposed into vector register packets (e.g., 8 floats per cycle), executing at peak vector throughput inside each core.

---

### Question 2: Concurrency & Compiler Diagnostics (5 Marks)
**What is a "Latent Bug" in vectorized OpenMP code? Describe three execution conditions under which a latent bug fails to appear.**

#### Model Examination Answer:
1. **Definition:** A latent bug is a concurrency or data-dependency defect that exists in code but remains hidden during normal testing, appearing only when specific compilation optimization levels or hardware environments are activated.
2. **Three Conditions Where it Remains Hidden:**
   - **Low Optimization Levels (`-O0`, `-O1`):** The compiler does not vectorize the loop and executes it sequentially element-by-element, hiding race conditions on unprivatized temporary variables.
   - **Hardware Lacking Vector Registers:** On an older CPU or architecture without SIMD support, the compiler emits scalar code, producing correct results.
   - **Small Problem Sizes ($N$):** If the loop iteration count is too small to fill a vector register, the runtime executes a scalar cleanup loop, bypassing the vector pipeline.

---

## 10. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.8 QUICK REVISION SHEET
================================================================================

1. WHAT IS SIMD VECTORIZATION?
   - Single Instruction, Multiple Data: Hardware registers inside ONE core 
     (SSE = 128b, AVX2 = 256b, AVX-512 = 512b).
   - SIMD Factor = Vector Register Width / Element Size (e.g., 256 / 32 = 8 floats).

2. #PRAGMA OMP SIMD:
   - Directs compiler to bypass conservative dependency checks and vectorize loop.
   - aligned(ptr : 64): Informs compiler memory is aligned, enabling fast vmovapd.

3. COMBINED MULTI-LEVEL PARALLELISM:
   - #pragma omp parallel for simd
   - Cores compute chunks in parallel (TLP) + each core vectorizes chunk (SIMD)!

4. LATENT BUGS:
   - A bug that exists in code but only manifests when vectorized at -O3.
   - Caused by missing `private(tmp)` or vectorizing loop-carried dependencies.

5. GCC DIAGNOSTIC FLAGS:
   - gcc -O3 -fopenmp -fopt-info-vec -march=native program.c
================================================================================
```

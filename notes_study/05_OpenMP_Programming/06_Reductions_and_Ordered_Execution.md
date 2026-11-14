# Reductions & Ordered Execution in OpenMP

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               06 — Reductions & Ordered Execution
PREREQUISITES:          Subtopic 05.3 (Data Scoping), Subtopic 05.4 (Loop Worksharing),
                        Subtopic 05.5 (Synchronization Constructs)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 21–22)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Reduction Dilemma

One of the most ubiquitous patterns in scientific computing, machine learning, and data analytics is **Accumulation**:
- Calculating the sum of an array of $100\text{ million}$ floats.
- Computing the Dot Product of two high-dimensional vectors ($\sum A_i \times B_i$).
- Finding the global minimum or maximum temperature in a weather simulation grid.
- Numerical integration (approximating $\pi$).

In sequential C, accumulation is trivial:
```c
double total_sum = 0.0;
for (int i = 0; i < N; i++) {
    total_sum += array[i];
}
```

### Why Naive Parallelization Fails Catastrophically

Suppose you attempt to parallelize this loop across 16 threads using the synchronization constructs we learned in Subtopic 05.5:

```
+-------------------------------------------------------------------------------+
|                       THE THREE FLAWED ACCUMULATION ATTEMPTS                  |
+-------------------------------------------------------------------------------+
| ATTEMPT 1: Leaving `total_sum` Shared (No Synchronization)                    |
| - Result: CATASTROPHIC DATA RACE!                                             |
| - 16 threads execute `total_sum += array[i]` simultaneously without locks.   |
| - Assembly: LOAD -> ADD -> STORE interleaved. Over 90% of updates are lost!   |
|                                                                               |
| ATTEMPT 2: Protecting with `#pragma omp critical`                             |
| - Result: CRIPPLING SERIALIZATION COLLAPSE!                                   |
| - 16 threads execute $100,000,000$ lock acquisitions.                         |
| - Locks turn parallel execution into a strict single-file line.               |
| - Performance: Runs $50\times$ SLOWER than a single-threaded program!         |
|                                                                               |
| ATTEMPT 3: Protecting with `#pragma omp atomic`                               |
| - Result: BETTER, BUT SEVERELY CONGESTED!                                     |
| - Bypasses software locks, but 16 cores hammer the exact same L1 cache line   |
|   with atomic bus-locking instructions.                                       |
| - The memory bus chokes on cache-coherence invalidation traffic.              |
+-------------------------------------------------------------------------------+
```

We need a pattern where each thread computes in complete isolation at full hardware speed, combining results only at the very end.

This is the **Reduction Pattern**.

---

## 2. The Reduction Pattern Architecture

$$\boxed{\textbf{Reduction: } \text{A parallel algorithmic pattern that collapses an array or iteration space of}}$$
$$\boxed{\text{data elements into a single scalar result using an associative binary operator } \oplus.}$$

OpenMP provides native support for this pattern via the **`reduction()` clause**:

```c
#pragma omp parallel for reduction(operator : list)
```

### The 4-Stage Mechanical Walkthrough

```
========================================================================================
                      THE 4 STAGES OF OPENMP REDUCTION EXECUTION
========================================================================================

  Stage 1: LOCAL PRIVATIZATION & IDENTITY INITIALIZATION
  OpenMP creates an independent private accumulator for each thread and initializes
  it to the mathematical IDENTITY of the operator (e.g., 0 for addition, 1 for mult):
      Thread 0: sum_local_0 = 0.0
      Thread 1: sum_local_1 = 0.0
      Thread 2: sum_local_2 = 0.0
      Thread 3: sum_local_3 = 0.0

  Stage 2: CONTENTION-FREE LOCAL PARALLEL ACCUMULATION
  Each thread executes its assigned slice of loop iterations, updating only its
  private local accumulator in fast CPU registers/L1 cache with ZERO synchronization:
      Thread 0: computes iterations 0..249   -> sum_local_0 = 1250.5
      Thread 1: computes iterations 250..499 -> sum_local_1 = 3410.2
      Thread 2: computes iterations 500..749 -> sum_local_2 = 2890.1
      Thread 3: computes iterations 750..999 -> sum_local_3 = 4120.8

  Stage 3: TREE-BASED COMBINATION AT BARRIER (JOIN)
  At the end of the loop, OpenMP combines the local sums using a parallel tree:
                 [ sum_0 ]       [ sum_1 ]       [ sum_2 ]       [ sum_3 ]
                     \               /               \               /
                      +-------------+                 +-------------+
                             |                               |
                         [ Temp A ]                      [ Temp B ]
                             \                               /
                              +-----------------------------+
                                             |
  Stage 4: ATOMIC WRITE-BACK TO GLOBAL SHARED VARIABLE
                                    [ global_sum ] = 11671.6
========================================================================================
```

---

## 3. Supported Reduction Operators & Mathematical Identity Values

When OpenMP creates a thread-private accumulator, what value does it start with?

It cannot start with 42 or memory garbage; it **must start with the mathematical identity element** of the chosen operator ($x \oplus \text{Identity} = x$):

```
+-----------------------------------------------------------------------------------+
|                        OPENMP REDUCTION OPERATORS & IDENTITIES                    |
+-----------------------------------------------------------------------------------+
| Operator Identifier   | Mathematical Operation        | Initial Identity Value    |
| :-------------------- | :---------------------------- | :------------------------ |
| **`+`**               | Arithmetic Addition           | `0`                       |
| **`*`**               | Arithmetic Multiplication     | `1`                       |
| **`-`**               | Arithmetic Subtraction        | `0` (Combined via `+`)    |
| **`&`**               | Bitwise AND                   | `~0` (All bits 1)         |
| **`|`**               | Bitwise OR                    | `0`                       |
| **`^`**               | Bitwise XOR                   | `0`                       |
| **`&&`**              | Logical AND                   | `1` (True)                |
| **`||`**              | Logical OR                    | `0` (False)               |
| **`max`**             | Maximum Value                 | Smallest representable val |
|                       |                               | (e.g., `-INFINITY`, `INT_MIN`)|
| **`min`**             | Minimum Value                 | Largest representable val  |
|                       |                               | (e.g., `+INFINITY`, `INT_MAX`)|
+-----------------------------------------------------------------------------------+
```

---

## 4. Mathematical Requirements & Floating-Point Realities

As highlighted in your lecture notes (`3d773f8d.pdf`, Page 22):

### 1. Associativity is Mandatory
The binary operator $\oplus$ must be mathematically **Associative**:
$$(a \oplus b) \oplus c = a \oplus (b \oplus c)$$
Without associativity, grouping iterations into different thread chunks will produce completely invalid, arbitrary answers.

### 2. Commutativity is Highly Desirable
$$a \oplus b = b \oplus a$$
Commutativity allows the OpenMP runtime to combine thread partial results in whatever tree reduction order matches the hardware core topology.

---

### The Floating-Point Non-Associativity Trap (IEEE 754)

> [!WARNING]
> **Floating-Point Roundoff Variation in Parallel Reductions:**  
> In pure mathematics, real addition is associative: $(a + b) + c = a + (b + c)$.  
> In computer hardware conforming to **IEEE 754 Floating-Point Arithmetic**, addition is **NOT associative** due to finite precision and rounding!  
> If you sum $10^7$ double-precision numbers:
> - Running sequentially sums elements left-to-right: `(...((a[0] + a[1]) + a[2]) + ...)`.
> - Running with 4 threads sums 4 independent chunks and adds the 4 totals.
> - Running with 8 threads sums 8 chunks.  
> **Result:** The 4-thread sum and the 8-thread sum may differ in their least significant digits (e.g., in the 15th decimal place)! This is normal numerical behavior, not an OpenMP bug.

---

## 5. Ordered Execution in OpenMP: `#pragma omp ordered`

When a loop is parallelized across multiple cores, **the iterations finish in completely non-deterministic order**:
- Thread 2 might finish iteration $i = 40$ before Thread 0 finishes iteration $i = 1$.
- If the loop prints debug statements or writes records to a sequential output file, the output will be scrambled out of order.

What if $95\%$ of a loop's workload is heavy math that can run in parallel, but $5\%$ of the loop **must execute in strict sequential iteration order** ($i = 0, 1, 2, 3, \dots$)?

$$\boxed{\textbf{\#pragma omp ordered: } \text{Forces an enclosed structured block inside a parallel for-loop to}}$$
$$\boxed{\text{execute in the \textbf{exact sequential order} of the original serial loop index.}}$$

---

### The Two Syntax Requirements

As noted in `3d773f8d.pdf` (Page 22), `#pragma omp ordered` requires two cooperating elements:
1. The enclosing `#pragma omp for` directive **must include the `ordered` clause**:
   ```c
   #pragma omp parallel for ordered
   ```
2. The specific sequential statements must be wrapped inside the `#pragma omp ordered` construct:
   ```c
   #pragma omp ordered
   {
       write_to_file(i, result[i]);
   }
   ```

```
========================================================================================
                      THE ORDERED WORKFLOW TIMELINE
========================================================================================

  Thread 0 (Iter 0): [==== Heavy Math ====] -> [ ORDERED BLOCK 0 ] (Executes first!)
  Thread 1 (Iter 1): [== Fast Math ==] ----------> WAITING FOR THREAD 0...
                                                  [ ORDERED BLOCK 1 ] (Executes second!)
  Thread 2 (Iter 2): [====== Math ======] -------> WAITING FOR THREAD 1...
                                                  [ ORDERED BLOCK 2 ] (Executes third!)
========================================================================================
```

- **Performance Trade-off:** The heavy mathematical calculation executes in true parallel hardware. The ordered block serializes execution like a relay race baton pass, preserving output correctness.

---

## 6. Complete Compilable C Program: Numerical Integration ($\pi$) with Reduction & Ordered Output

Here is the complete, high-performance C program demonstrating both the `reduction(+:...)` clause and `#pragma omp ordered`:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_reduction_ordered.c
 * DESCRIPTION: Approximates the mathematical constant Pi via numerical
 *              integration (Trapezoidal Rule) using OpenMP reduction.
 *              Demonstrates ordered execution for sequential progress logging.
 * COMPILATION: gcc -Wall -O2 openmp_reduction_ordered.c -o omp_reduction -fopenmp
 * USAGE:       ./omp_reduction
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define NUM_STEPS 100000000 // 100 Million intervals
#define NUM_THREADS 4

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("        OPENMP REDUCTION & ORDERED EXECUTION BENCHMARK          \n");
    printf("   Approximating Pi over %d steps using %d Threads\n", 
           NUM_STEPS, NUM_THREADS);
    printf("================================================================\n\n");

    double step = 1.0 / (double)NUM_STEPS;
    double global_pi_sum = 0.0;

    double start_time = omp_get_wtime();

    // ------------------------------------------------------------------------
    // PARALLEL NUMERICAL INTEGRATION WITH REDUCTION
    // Mathematical Formula: \int_0^1 \frac{4}{1 + x^2} dx = \pi
    // ------------------------------------------------------------------------
    #pragma omp parallel for num_threads(NUM_THREADS) \
                             schedule(static) \
                             reduction(+:global_pi_sum)
    for (int i = 0; i < NUM_STEPS; i++) {
        // x is the midpoint of rectangle i: Automatically private because
        // declared inside loop body!
        double x = (i + 0.5) * step;
        double height = 4.0 / (1.0 + x * x);
        
        // Accumulate into thread-private reduction variable (ZERO lock contention!)
        global_pi_sum += height;
    }

    double final_pi = global_pi_sum * step;
    double elapsed_time = omp_get_wtime() - start_time;

    printf("[Result] Computed Pi:   %.14f\n", final_pi);
    printf("[Result] Reference Pi:  3.14159265358979\n");
    printf("[Result] Elapsed Time:  %.6f seconds\n\n", elapsed_time);

    // ------------------------------------------------------------------------
    // DEMO 2: ORDERED DIRECTIVE DEMONSTRATION
    // ------------------------------------------------------------------------
    printf("--- DEMO 2: Strict Sequential Output via #pragma omp ordered ---\n");

    // The loop declares the `ordered` clause
    #pragma omp parallel for num_threads(NUM_THREADS) ordered schedule(static)
    for (int i = 0; i < 8; i++) {
        int tid = omp_get_thread_num();

        // 1. Parallel independent computation
        int square = i * i;

        // 2. Ordered sequential output: Guaranteed to print 0, 1, 2, ..., 7 in order!
        #pragma omp ordered
        {
            printf("  [Ordered Output] Thread %d printed result for index %d: %d\n", 
                   tid, i, square);
        }
    }

    printf("\n[Main] Program executed successfully!\n");
    return 0;
}
```

---

## 7. University Examination Preparation: Solved Questions

### Question 1: Core Mechanics (10 Marks)
**Explain the working mechanism of the OpenMP `reduction(operator : list)` clause. Detail the four operational phases: privatization, initialization, local accumulation, and final tree reduction. Why is it superior to using `#pragma omp atomic` inside a loop?**

#### Model Examination Answer:
1. **Four Operational Phases of Reduction:**
   - **Phase 1: Privatization:** The compiler creates an independent local copy of the reduction variable on each thread's private stack.
   - **Phase 2: Identity Initialization:** Each thread's local copy is initialized to the mathematical identity element of the operator ($0$ for `+`, $1$ for `*`, $-\infty$ for `max`, $+\infty$ for `min`).
   - **Phase 3: Contention-Free Local Accumulation:** Each thread executes its slice of loop iterations, updating its local accumulator entirely inside CPU registers and L1 cache without acquiring any locks or performing memory bus synchronization.
   - **Phase 4: Tree-Based Final Reduction:** At the loop barrier, OpenMP combines the private values into the original shared variable using a logarithmic binary tree reduction ($\approx \log_2 N$ steps).
2. **Why Reduction is Superior to Atomic:**  
   If `#pragma omp atomic` is placed inside a loop of $10^8$ iterations, $N$ cores compete for the exact same cache line $10^8$ times. This invalidates L1/L2 caches across all sockets, choking the memory bus with MESI cache-coherence traffic. In contrast, `reduction` executes $10^8$ accumulations at full register speed ($0$ bus traffic), performing exactly $N-1$ atomic combinations at the very end.

---

### Question 2: Ordered Construct (5 Marks)
**What problem does the `#pragma omp ordered` directive solve? What are the two mandatory syntax requirements for its use?**

#### Model Examination Answer:
1. **Problem Solved:** When a for-loop is parallelized in OpenMP, threads execute iterations concurrently and finish out-of-order. If a loop needs to perform sequential actions (e.g., printing ordered logs, writing sequentially to an unseekable file, or preserving algorithmic order), standard parallel loops fail. `#pragma omp ordered` guarantees that the enclosed code executes in the exact numerical sequence of the original serial loop ($i = 0, 1, 2, \dots$) while allowing the rest of the loop body to execute in parallel.
2. **Mandatory Syntax Requirements:**
   - The loop directive must explicitly include the `ordered` clause: `#pragma omp for ordered`.
   - The sequential code block must be enclosed inside `#pragma omp ordered { ... }`.

---

## 8. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.6 QUICK REVISION SHEET
================================================================================

1. THE REDUCTION CLAUSE: reduction(operator : list)
   - Eliminates lock contention by giving each thread a private accumulator.
   - Accumulates at register speed; combines results via binary tree at join.

2. IDENTITY INITIALIZATION:
   - operator + : Initialized to 0
   - operator * : Initialized to 1
   - operator max : Initialized to smallest representable value (-INFINITY / INT_MIN)
   - operator min : Initialized to largest representable value (+INFINITY / INT_MAX)

3. MATHEMATICAL REQUIREMENTS:
   - Must be Associative: (a + b) + c == a + (b + c)
   - Floating-point warning: IEEE 754 addition is non-associative; slight roundoff
     variations across different thread counts are normal!

4. ORDERED DIRECTIVE (#pragma omp ordered):
   - Forces block to execute in strict serial iteration order (0, 1, 2, ...).
   - Requires `ordered` clause on the enclosing loop: #pragma omp for ordered
================================================================================
```

# Loop Worksharing & Scheduling Policies in OpenMP

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               04 — Loop Worksharing & Scheduling Policies
PREREQUISITES:          Subtopic 03.4 (Hardware Resource Bottlenecks), Subtopic 05.1 (Fork-Join),
                        Subtopic 05.3 (Data Scoping)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 16–19)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: From Task Duplication to Iteration Worksharing

In Subtopic 05.1, we saw that a basic `#pragma omp parallel` directive causes **every thread in the team to execute the exact same block of code**:

```c
#pragma omp parallel num_threads(4)
{
    // Every thread runs this loop from 0 to 999!
    for (int i = 0; i < 1000; i++) {
        do_work(i);
    }
}
```
If you run this code, the loop does not run 4 times faster; instead, **all 4 threads execute all 1,000 iterations redundantly**, performing $4,000$ total iterations!

To achieve true parallel speedup on loops, we need **Worksharing**:
$$\boxed{\textbf{Loop Worksharing: } \text{The total iteration space of a loop is divided into disjoint subsets,}}$$
$$\boxed{\text{and each subset is assigned to an independent thread in the team.}}$$

OpenMP achieves this through the **`#pragma omp for`** construct.

---

## 2. The Loop Worksharing Construct: Separate vs Combined Forms

OpenMP provides two ways to declare parallel loop worksharing:

```
+-----------------------------------------------------------------------------------+
|                        THE TWO SYNTACTIC WORKSHARING FORMS                        |
+-----------------------------------------------------------------------------------+
| 1. TWO-STEP NESTED DIRECTIVE FORM:                                                |
|    #pragma omp parallel                                                           |
|    {                                                                              |
|        // Code here is executed redundantly by all threads                        |
|        #pragma omp for                                                            |
|        for (int i = 0; i < N; i++) {                                              |
|            c[i] = a[i] + b[i]; // Iterations divided among threads                |
|        }                                                                          |
|        // Code here is executed redundantly by all threads                        |
|    }                                                                              |
|                                                                                   |
| 2. COMBINED DIRECTIVE FORM (Most Common):                                         |
|    #pragma omp parallel for                                                       |
|    for (int i = 0; i < N; i++) {                                                  |
|        c[i] = a[i] + b[i];                                                        |
|    }                                                                              |
+-----------------------------------------------------------------------------------+
```

- The **combined directive** `#pragma omp parallel for` is shorthand. It simultaneously forks a team of threads *and* distributes the immediately following for-loop across those threads.

---

### Iteration Partitioning Example: 12 Iterations Across 3 Threads

As illustrated in your lecture notes (`3d773f8d.pdf`, Page 16):

```
========================================================================================
             ITERATION PARTITIONING: 12 ITERATIONS ACROSS 3 THREADS
========================================================================================

                  #pragma omp parallel for (Iterations i = 1 to 12)
                             /             |             \
                            /              |              \
               Thread 0               Thread 1               Thread 2
             +----------+           +----------+           +----------+
             |  i = 1   |           |  i = 5   |           |  i = 9   |
             |  i = 2   |           |  i = 6   |           |  i = 10  |
             |  i = 3   |           |  i = 7   |           |  i = 11  |
             |  i = 4   |           |  i = 8   |           |  i = 12  |
             +----------+           +----------+           +----------+
                            \              |              /
                             \             |             /
                              v            v            v
                   =================================================
                                   IMPLICIT BARRIER
                   =================================================
========================================================================================
```

---

## 3. The Canonical Loop Form Requirements

An OpenMP compiler cannot parallelize an arbitrary C `while` loop or complex `for` loop. The loop must adhere strictly to the **Canonical Loop Form** so the runtime can calculate the exact trip count before execution begins:

```c
for (index = start; index < relational-op > end; increment-expr)
```

```
+-------------------------------------------------------------------------------+
|                       CANONICAL LOOP FORM RULES                               |
+-------------------------------------------------------------------------------+
| 1. LOOP INDEX VARIABLE:                                                       |
|    - Must be a signed or unsigned integer or pointer (C++ random-access iter).|
|    - Is AUTOMATICALLY made `private` to each thread by OpenMP.                |
|                                                                               |
| 2. RELATIONAL OPERATOR:                                                       |
|    - Must be strictly one of: `<`, `<=`, `>`, `>=`.                           |
|    - Cannot use `!=` (non-standard in C for-loops).                           |
|                                                                               |
| 3. INVARIANT LOOP BOUNDS:                                                     |
|    - The loop limit expressions `start` and `end` must be loop-invariant;     |
|      they cannot change value while the loop is running.                      |
|                                                                               |
| 4. STRICT SINGLE-STEP INCREMENT:                                              |
|    - Must be: `i++`, `++i`, `i--`, `--i`, `i += step`, `i -= step`.           |
|                                                                               |
| 5. NO MID-LOOP MODIFICATIONS OR EXITS:                                        |
|    - The code inside the loop cannot manually modify the loop index `i`.      |
|    - You CANNOT exit the loop early using `break` or `goto`!                  |
+-------------------------------------------------------------------------------+
```

---

## 4. The Implicit Barrier & The `nowait` Clause

By default, `#pragma omp for` inserts an **implicit synchronization barrier** at the end of the loop:
- Threads that finish their assigned iterations early **must wait** until the slowest thread completes its final iteration.

### When to Eliminate the Barrier: The `nowait` Clause
If your program executes two successive loops that access **completely independent data**, forcing threads to synchronize at the end of the first loop wastes valuable compute time:

```c
#pragma omp parallel
{
    // Loop 1: Updates array A (No barrier at end!)
    #pragma omp for nowait
    for (int i = 0; i < N; i++) {
        a[i] = compute_A(i);
    } // Fast threads proceed immediately to Loop 2 without waiting!

    // Loop 2: Updates independent array B
    #pragma omp for
    for (int i = 0; i < N; i++) {
        b[i] = compute_B(i);
    }
} // Implicit barrier here ensures both A and B are fully finished
```

> [!CAUTION]
> **Data Dependency Danger:**  
> Never use `nowait` if Loop 2 reads data produced by Loop 1! A fast thread would begin reading unwritten memory, inducing a devastating data race.

---

## 5. The Four OpenMP Loop Scheduling Policies

How does OpenMP decide which thread gets which iteration? 

This is configured using the **`schedule()` clause**:

```c
#pragma omp parallel for schedule(kind [, chunk_size])
```

As detailed in your lecture notes (`3d773f8d.pdf`, Pages 17–19), OpenMP provides **four fundamental scheduling strategies**:

```
+-----------------------------------------------------------------------------------+
|                        OPENMP SCHEDULING STRATEGIES                               |
+-----------------------------------------------------------------------------------+
| Policy Identifier     | Allocation Time   | Runtime Overhead | Load Balancing     |
| :-------------------- | :---------------- | :--------------- | :----------------- |
| **`static`**          | Compile / Fork    | **Zero** (None)  | Poor for uneven work|
| **`dynamic`**         | Runtime           | High (Work Queue)| **Excellent**      |
| **`guided`**          | Runtime           | Low-to-Medium    | **Excellent**      |
| **`auto`**            | Compiler Decision | Variable         | Compiler Optimized |
| **`runtime`**         | Shell Environment | Matches chosen   | Configurable       |
+-----------------------------------------------------------------------------------+
```

---

### Strategy 1: Static Scheduling (`schedule(static [, chunk])`)

**Working Principle:** Iterations are divided into blocks of size `chunk_size` and statically assigned to threads in round-robin order **before the loop begins executing**.

- **If `chunk` is omitted:** Iterations are divided into $N$ contiguous blocks ($N = \text{number of threads}$). For $100$ iterations and $4$ threads, Thread 0 gets $0..24$, Thread 1 gets $25..49$, etc.
- **Advantages:** Zero runtime synchronization overhead. Maximizes CPU cache spatial locality (stride-1 access).
- **Disadvantages:** Severely vulnerable to **Load Imbalance** if some iterations take much longer than others.

---

#### The Static Chunking Pitfall (Directly from Lecture Notes `3d773f8d.pdf`, Page 17)

Consider 8 total iterations running on 4 threads:

```
Case A: `schedule(static, 2)` (8 iterations, 4 threads, chunk = 2)
  - Thread 0: Iterations 0, 1
  - Thread 1: Iterations 2, 3
  - Thread 2: Iterations 4, 5
  - Thread 3: Iterations 6, 7
  * Result: Perfect distribution. Every thread executes 2 iterations.

Case B: `schedule(static, 4)` (8 iterations, 4 threads, chunk = 4)
  - Thread 0: Iterations 0, 1, 2, 3 (Chunk 0)
  - Thread 1: Iterations 4, 5, 6, 7 (Chunk 1)
  - Thread 2: ZERO iterations! (Sits 100% idle!)
  - Thread 3: ZERO iterations! (Sits 100% idle!)
  * Result: CATASTROPHIC IDLE WASTE! As the lecture notes explicitly state:
    "If you have taken 4 threads, 2 will remain idle."
```

---

### Strategy 2: Dynamic Scheduling (`schedule(dynamic [, chunk])`)

**Working Principle:** OpenMP sets up a centralized, thread-safe **Work Queue**. Iterations are grouped into chunks of size `chunk_size` (default `chunk = 1`).
1. At loop launch, each thread grabs one chunk from the work queue.
2. When a thread finishes its chunk, it returns to the queue and dynamically requests the next available chunk.
3. This repeats until the work queue is exhausted.

```
========================================================================================
                       DYNAMIC SCHEDULING WORK-QUEUE MODEL
========================================================================================

              CENTRALIZED WORK QUEUE: [Chunk 4] [Chunk 5] [Chunk 6] ...
                                         |         |         |
                      +------------------+---------+---------+
                      |                  |                   |
                      v                  v                   v
                 Thread 0             Thread 1            Thread 2
             (Fast Iteration)     (Heavy Math Load)   (Medium Load)
             Finishes quickly!    Stalled for 100ms   Finishes in 40ms
                      |
                      v
             Returns to queue &
             grabs Chunk 7!
========================================================================================
```

- **Ideal Workloads:**
  - Non-uniform, data-dependent loops (e.g., rendering the Mandelbrot set, ray tracing, graph traversal, sparse matrix algebra).
  - Systems with heterogeneous core speeds (e.g., ARM big.LITTLE architectures).
- **Trade-off:** Incurs runtime queue synchronization overhead. If iterations are tiny (e.g., `a[i] = b[i] + 1`), dynamic scheduling overhead can make the loop run **$10\times$ slower than static scheduling**!

---

### Strategy 3: Guided Scheduling (`schedule(guided [, chunk])`)

**Working Principle:** Designed to combine the low overhead of static scheduling with the dynamic load balancing of dynamic scheduling.

- OpenMP starts by assigning **large chunks** to each thread at the beginning of the loop (minimizing queue contention when work is abundant).
- As remaining iterations decrease, the chunk size **exponentially decreases** proportional to the unassigned workload:
  $$\text{Chunk Size} \approx \frac{\text{Remaining Unassigned Iterations}}{\text{Number of Threads}}$$
- The parameter `chunk_size` sets the minimum lower bound for chunk dimensions (default `chunk = 1`).

```
========================================================================================
                  GUIDED SCHEDULING: EXPONENTIALLY DECREASING CHUNKS
========================================================================================

Loop Iterations (N = 1000, 4 Threads):
Start of Loop:  Thread 0 grabs Chunk of 250 iterations  (Large chunk: near-zero overhead!)
                Thread 1 grabs Chunk of 188 iterations
                Thread 2 grabs Chunk of 141 iterations
...
End of Loop:    Remaining work is small:
                Threads grab Chunks of 10, 5, 2, 1 iterations (Fine-tuning load balance!)
========================================================================================
```

---

### Strategy 4: Auto & Runtime Scheduling

#### `schedule(auto)`
Delegates the scheduling choice entirely to the compiler and runtime heuristics. If the compiler can prove loop bounds and iteration costs are uniform, it chooses `static`; if unpredictable, it chooses `guided` or `dynamic`.

#### `schedule(runtime)`
Defers the scheduling decision to the execution environment via the shell variable **`OMP_SCHEDULE`**:

```c
#pragma omp parallel for schedule(runtime)
for (int i = 0; i < N; i++) { ... }
```
```bash
# Set scheduling dynamically in bash without recompiling code!
export OMP_SCHEDULE="dynamic,16"
./my_simulation
```

---

## 6. Complete Compilable C Program: Benchmarking Static vs Dynamic Scheduling

Here is the complete C program directly demonstrating the load balancing power of `dynamic` scheduling over `static` scheduling on an unbalanced workload (transcribed and enhanced from `3d773f8d.pdf`, Page 18):

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_scheduling_benchmark.c
 * DESCRIPTION: Directly compares Static vs Dynamic loop scheduling on an
 *              intentionally unbalanced workload (sleep duration varies with i).
 * COMPILATION: gcc -Wall -O2 openmp_scheduling_benchmark.c -o omp_sched -fopenmp
 * USAGE:       ./omp_sched
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <unistd.h>

#define TOTAL_ITERATIONS 12
#define NUM_THREADS 4

// Simulates variable computational work per iteration
void simulate_uneven_work(int iteration_id, int thread_id, const char *label) {
    // Workload increases with iteration index: (i % 4 + 1) * 20 milliseconds
    int sleep_ms = ((iteration_id % 4) + 1) * 25; // 25ms, 50ms, 75ms, 100ms
    
    printf("  [%s] Thread %d processing Iteration %2d (Work: %3d ms)\n", 
           label, thread_id, iteration_id, sleep_ms);
           
    usleep(sleep_ms * 1000); // Sleep in microseconds
}

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("        OPENMP LOOP SCHEDULING BENCHMARK: STATIC vs DYNAMIC     \n");
    printf("   Total Iterations: %d | Worker Threads: %d\n", TOTAL_ITERATIONS, NUM_THREADS);
    printf("================================================================\n\n");

    // ------------------------------------------------------------------------
    // EXPERIMENT 1: STATIC SCHEDULING (Default Chunking)
    // ------------------------------------------------------------------------
    printf("--- EXPERIMENT 1: Static Scheduling: schedule(static) ---\n");
    double start_static = omp_get_wtime();

    #pragma omp parallel for num_threads(NUM_THREADS) schedule(static)
    for (int i = 0; i < TOTAL_ITERATIONS; i++) {
        simulate_uneven_work(i, omp_get_thread_num(), "Static");
    }

    double time_static = omp_get_wtime() - start_static;
    printf(">>> Static Scheduling Elapsed Time:  %.4f seconds <<<\n\n", time_static);

    // ------------------------------------------------------------------------
    // EXPERIMENT 2: DYNAMIC SCHEDULING (chunk = 1)
    // ------------------------------------------------------------------------
    printf("--- EXPERIMENT 2: Dynamic Scheduling: schedule(dynamic, 1) ---\n");
    double start_dynamic = omp_get_wtime();

    #pragma omp parallel for num_threads(NUM_THREADS) schedule(dynamic, 1)
    for (int i = 0; i < TOTAL_ITERATIONS; i++) {
        simulate_uneven_work(i, omp_get_thread_num(), "Dynamic");
    }

    double time_dynamic = omp_get_wtime() - start_dynamic;
    printf(">>> Dynamic Scheduling Elapsed Time: %.4f seconds <<<\n\n", time_dynamic);

    // ------------------------------------------------------------------------
    // PERFORMANCE COMPARISON
    // ------------------------------------------------------------------------
    printf("================================================================\n");
    printf("VERDICT:\n");
    printf("  Static Time:  %.4f s\n", time_static);
    printf("  Dynamic Time: %.4f s\n", time_dynamic);
    if (time_dynamic < time_static) {
        printf("  Dynamic was %.2fx FASTER by eliminating load imbalance!\n", 
               time_static / time_dynamic);
    } else {
        printf("  Static was faster due to lower queue overhead.\n");
    }
    printf("================================================================\n");

    return 0;
}
```

---

## 7. University Examination Preparation: Solved Questions

### Question 1: Core Comparison (10 Marks)
**Compare Static, Dynamic, and Guided loop scheduling in OpenMP. Explain how iterations are assigned to threads in each policy. Under what application conditions should each policy be selected?**

#### Model Examination Answer:
1. **Detailed Policy Comparison:**

| Policy | Iteration Assignment Mechanism | Overhead | Best Application Workload |
| :--- | :--- | :--- | :--- |
| **`static`** | Iterations partitioned into fixed chunks and mapped round-robin to threads at compile/fork time before execution. | **Zero** (No synchronization) | **Uniform workloads:** Each iteration takes identical time (e.g., vector addition, dense matrix multiplication). |
| **`dynamic`** | Iterations placed in a shared work queue in chunks of size $K$. Threads pull a chunk dynamically when they finish their current chunk. | **High** (Queue locking overhead) | **Highly non-uniform workloads:** Iterations have unpredictable, data-dependent execution times (e.g., Mandelbrot set, sparse graphs). |
| **`guided`** | Iterations dispatched in exponentially decreasing chunk sizes ($K \approx \text{remaining} / \text{threads}$), tapering to a minimum bound. | **Low to Medium** | **Mildly non-uniform workloads:** Smooths load balancing while avoiding the high queue overhead of dynamic chunk-1 scheduling. |

---

### Question 2: The Idle Thread Trap (5 Marks)
**Consider the following OpenMP loop executing on 4 threads:**
```c
#pragma omp parallel for num_threads(4) schedule(static, 4)
for (int i = 0; i < 8; i++) {
    process(i);
}
```
**Analyze the thread-to-iteration assignment. Why does this loop exhibit poor parallel efficiency?**

#### Model Examination Answer:
1. **Iteration Assignment:**
   - Total iterations $= 8$. Chunk size $= 4$.
   - **Thread 0:** Assigned Chunk 0 $\rightarrow$ Iterations $0, 1, 2, 3$.
   - **Thread 1:** Assigned Chunk 1 $\rightarrow$ Iterations $4, 5, 6, 7$.
   - **Thread 2:** Assigned Chunk 2 $\rightarrow$ **ZERO iterations!**
   - **Thread 3:** Assigned Chunk 3 $\rightarrow$ **ZERO iterations!**
2. **Efficiency Analysis:**  
   Because the chunk size ($4$) is half the total iteration space ($8$), all iterations are consumed by the first two threads. Threads 2 and 3 sit completely idle for the entire loop duration, achieving only a maximum theoretical speedup of $2\times$ despite consuming 4 threads of resources ($50\%$ parallel efficiency waste).

---

### Question 3: Synchronization Elimination (5 Marks)
**What is the purpose of the `nowait` clause on `#pragma omp for`? Give a concrete code example where using `nowait` is safe and beneficial, and explain when it is dangerous.**

#### Model Examination Answer:
1. **Purpose:** By default, `#pragma omp for` enforces an implicit synchronization barrier at the end of the loop block. The `nowait` clause removes this barrier, allowing faster threads to proceed immediately to subsequent computational statements without idling.
2. **Safe Example (Independent Data):**
   ```c
   #pragma omp parallel
   {
       #pragma omp for nowait
       for (int i = 0; i < N; i++) a[i] = x[i] * 2; // Writes only array A
       
       #pragma omp for
       for (int i = 0; i < N; i++) b[i] = y[i] + 5; // Writes only array B
   }
   ```
   Safe because Loop 1 and Loop 2 access completely independent memory locations. Fast threads start populating `b[i]` without waiting for slow threads to finish `a[i]`.
3. **Dangerous Scenario (Data Dependency):** If Loop 2 read array `a[i]` computed by Loop 1, removing the barrier would create a read-after-write data race, causing Loop 2 to process uncomputed, corrupted data.

---

## 8. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.4 QUICK REVISION SHEET
================================================================================

1. WORKSHARING vs REDUNDANCY:
   - #pragma omp parallel: Duplicates block execution across all threads.
   - #pragma omp parallel for: Partitions loop iterations among threads!

2. CANONICAL LOOP FORM:
   - Must be standard integer for-loop: for (i = 0; i < N; i++)
   - Loop counter i is AUTOMATICALLY PRIVATE.
   - NO break or goto jumping out of the loop body!

3. SCHEDULING CLAUSE: schedule(type [, chunk])
   - static: Zero overhead, round-robin. Vulnerable to load imbalance.
     Beware of oversized chunks idling threads!
   - dynamic: Work queue. Perfect for uneven loops, but incurs queue lock overhead.
   - guided: Decreasing chunk sizes (large chunks first, tiny chunks at end).
   - runtime: Defer decision to OMP_SCHEDULE environment variable.

4. THE NOWAIT CLAUSE:
   - Removes implicit barrier at the end of #pragma omp for.
   - Safe ONLY when successive loops operate on completely independent data!
================================================================================
```

# Thread Management & Runtime Environment APIs in OpenMP

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               02 — Thread Management & Runtime Environment APIs
PREREQUISITES:          Subtopic 05.1 (OpenMP Architecture & Fork-Join Model)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 12–13)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Mechanics of Team Control

In Subtopic 05.1, we saw that inserting `#pragma omp parallel` spawns a team of threads that execute the enclosed structured block concurrently.

However, writing robust, high-performance software requires answers to essential operational questions:
- Exactly **how many threads** will OpenMP create for a given parallel region?
- Can a programmer override the user's terminal settings from inside C code?
- How does a worker thread determine its unique identity ($0, 1, 2, \dots$)?
- How do we measure the execution time of a parallel section with nanosecond precision?
- What happens if a parallel region is called **inside another parallel region** (nested parallelism)?

To answer these questions, OpenMP provides a standardized set of **Runtime Library APIs** defined in `<omp.h>` and formal **Precedence Rules** governing thread allocation.

---

## 2. The 4-Tier Thread Count Precedence Hierarchy

When an OpenMP program encounters `#pragma omp parallel`, the runtime must determine the team size ($N$). The system evaluates thread configuration across **four strict priority tiers**:

```
========================================================================================
                     THE 4-TIER THREAD ALLOCATION PRECEDENCE HIERARCHY
========================================================================================

  +-------------------------------------------------------------------------------+
  | TIER 1: `num_threads(N)` CLAUSE ON THE DIRECTIVE                             |
  |         * Highest Precedence! Overrides all code and environment settings.    |
  |         * Scope: Strictly local to that specific parallel construct.         |
  |         * Example: `#pragma omp parallel num_threads(4)`                     |
  +---------------------------------------+---------------------------------------+
                                          | If NO `num_threads` clause exists...
                                          v
  +-------------------------------------------------------------------------------+
  | TIER 2: `omp_set_num_threads(N)` RUNTIME FUNCTION CALL                       |
  |         * Called programmatically inside C/C++ source code.                   |
  |         * Scope: Applies to all subsequent parallel constructs globally.     |
  |         * Example: `omp_set_num_threads(8);`                                  |
  +---------------------------------------+---------------------------------------+
                                          | If function was never invoked...
                                          v
  +-------------------------------------------------------------------------------+
  | TIER 3: `OMP_NUM_THREADS` ENVIRONMENT VARIABLE                                |
  |         * Configured in the user's shell/terminal before launching program.   |
  |         * Scope: Global default for the entire process execution.             |
  |         * Example: `export OMP_NUM_THREADS=16`                               |
  +---------------------------------------+---------------------------------------+
                                          | If environment variable is unset...
                                          v
  +-------------------------------------------------------------------------------+
  | TIER 4: SYSTEM IMPLEMENTATION DEFAULT                                         |
  |         * Lowest Precedence! Determined by compiler and OS kernel.            |
  |         * Standard Behavior: Typically defaults to the total number of        |
  |           logical CPU hardware threads (cores + hyperthreads) on the machine. |
  +-------------------------------------------------------------------------------+
```

### Concrete Evaluation Scenario

Suppose a system has 4 physical cores (4 hardware threads). Trace what happens under different combinations:

```
Case A: Shell has `export OMP_NUM_THREADS=2`.
        Code executes: `#pragma omp parallel`
        Result: 2 threads (Tier 3 governs).

Case B: Shell has `export OMP_NUM_THREADS=2`.
        Code executes: `omp_set_num_threads(8);`
        Code executes: `#pragma omp parallel`
        Result: 8 threads (Tier 2 overrides Tier 3).

Case C: Shell has `export OMP_NUM_THREADS=2`.
        Code executes: `omp_set_num_threads(8);`
        Code executes: `#pragma omp parallel num_threads(3)`
        Result: 3 threads (Tier 1 overrides Tier 2 and Tier 3).
```

---

## 3. Essential Runtime Environment Query Functions

To inspect and coordinate threads at runtime, include the standard OpenMP header:

```c
#include <omp.h>
```

```
+-----------------------------------------------------------------------------------+
|                        CORE OPENMP RUNTIME QUERY FUNCTIONS                        |
+-----------------------------------------------------------------------------------+
| Function Prototype            | Return Value Description                          |
| :---------------------------- | :------------------------------------------------ |
| `int omp_get_thread_num(void)`| Unique thread index within the current team:      |
|                               | $0, 1, 2, \dots, N-1$. Master is always $0$.      |
| `int omp_get_num_threads(void)`| Total count of active threads in the current team.|
|                               | **WARNING:** Returns $1$ when called in serial!   |
| `int omp_get_max_threads(void)`| Maximum threads available for the next parallel   |
|                               | region without an explicit `num_threads` clause.  |
| `int omp_get_num_procs(void)` | Number of physical/logical CPU cores online.      |
| `int omp_in_parallel(void)`   | Returns non-zero (true) if currently executing    |
|                               | within an active parallel team; $0$ in serial.    |
+-----------------------------------------------------------------------------------+
```

---

### The Classic Beginner Trap: `omp_get_num_threads()` vs `omp_get_max_threads()`

Examine this common programming error:

```c
// DANGEROUS BUG IN SERIAL CODE:
int n_threads = omp_get_num_threads(); // In serial region, THIS RETURNS 1!
double *partial_sums = (double*)malloc(n_threads * sizeof(double)); // Allocates space for ONLY 1 element!

#pragma omp parallel num_threads(8)
{
    int tid = omp_get_thread_num();
    partial_sums[tid] = calculate(tid); // BUFFER OVERFLOW! Thread 1..7 write out of bounds!
}
```

### Why Did This Fail?
- `omp_get_num_threads()` queries the **active team size**. In the serial region outside the parallel block, only 1 thread (the Master) is active, so it returns **1**!
- When the parallel construct forks 8 threads, threads $1$ through $7$ write beyond the allocated 1-element buffer, causing memory corruption or segmentation faults.

### The Correct Solution: Use `omp_get_max_threads()`
```c
// SAFE, PROFESSIONAL CODE:
int max_threads = omp_get_max_threads(); // Returns 8 (or OMP_NUM_THREADS setting)
double *partial_sums = (double*)malloc(max_threads * sizeof(double));

#pragma omp parallel
{
    int tid = omp_get_thread_num();
    partial_sums[tid] = calculate(tid); // 100% Safe!
}
```

---

## 4. High-Precision Wall-Clock Timing API

In Subtopic 03.5, we emphasized that empirical benchmarking is the foundation of high-performance computing. 

### Why Standard C `clock()` Fails in Parallel Computing

In sequential C programming, developers use `clock()` from `<time.h>`:

```c
// BROKEN FOR MULTITHREADED CODE!
clock_t start = clock();
// Run parallel OpenMP loop across 16 cores for 1 second...
clock_t end = clock();
double time_taken = (double)(end - start) / CLOCKS_PER_SEC; // REPORTS 16 SECONDS!
```

$$\boxed{\textbf{The \texttt{clock()} Trap: } \text{\texttt{clock()} measures \textbf{Total CPU Time} consumed across ALL cores!}}$$

If 16 CPU cores execute at $100\%$ capacity for $1$ real-world second, `clock()` accumulates $16$ core-seconds. It is physically impossible to measure parallel speedup or wall-clock turnaround time using `clock()`.

---

### The OpenMP Solution: `omp_get_wtime()`

OpenMP provides a high-resolution, monotonically increasing wall-clock timer:

```c
double omp_get_wtime(void);
```

- Returns a `double` representing elapsed wall-clock time in **seconds** relative to an arbitrary, fixed point in the past (such as system boot time).
- Guarantees precision down to nanoseconds or microseconds on modern x86-64 and ARM processors (reading hardware timestamp counters like `RDTSC`).

#### Measuring Timer Resolution: `omp_get_wtick()`
```c
double omp_get_wtick(void);
```
Returns the clock tick duration in seconds (e.g., $10^{-9}\text{ s} = 1\text{ nanosecond}$).

---

### The Standard OpenMP Benchmarking Pattern

```c
double start_time = omp_get_wtime();

#pragma omp parallel
{
    // Perform intensive parallel numerical simulation...
}

double end_time = omp_get_wtime();
double elapsed_seconds = end_time - start_time;

printf("Real Wall-Clock Elapsed Time: %.6f seconds\n", elapsed_seconds);
```

---

## 5. Dynamic Thread Adjustment (`omp_set_dynamic`)

By default, an OpenMP runtime system may dynamically reduce the number of threads assigned to a parallel region if the operating system is heavily loaded with background tasks.

To prevent the runtime from altering your requested thread counts:

```c
// Disable dynamic adjustment: Enforces EXACT thread counts!
omp_set_dynamic(0);

// Check if dynamic adjustment is enabled:
int is_dynamic = omp_get_dynamic(); // Returns 0 if disabled
```
*(Equivalent shell setting: `export OMP_DYNAMIC=FALSE`).*

---

## 6. Nested Parallelism: Teams Within Teams

What happens when a thread inside a parallel region encounters **another** `#pragma omp parallel` construct?

```c
#pragma omp parallel num_threads(4) // Outer Region: 4 Threads
{
    #pragma omp parallel num_threads(4) // Inner Region: Nested Parallelism
    {
        // How many total threads are executing here?
    }
}
```

```
========================================================================================
                               NESTED PARALLELISM MODES
========================================================================================

DISABLED (Default Behavior - Serialized):
Outer Team (4 Threads) ----> Inner Region: Each outer thread acts as Master of a 
                             team of size 1 (Serialized execution).
                             Total Threads = 4.

ENABLED (Active Multi-Level Threading):
Outer Team (4 Threads) ----> Inner Region: Each outer thread FORKS 4 child threads!
                             Total Active Threads = 4 x 4 = 16 Threads!
========================================================================================
```

### The Thread Explosion Catastrophe
If nested parallelism is enabled carelessly:
- Outer loop forks $16$ threads.
- Inner loop forks $16$ threads.
- **Total active threads:** $16 \times 16 = 256$ threads!
- On a 16-core workstation, 256 active threads violently contend for CPU cores, causing extreme operating system context-switching thrashing and collapsing performance.

### Controlling Nested Parallelism in Modern OpenMP

In modern OpenMP (v3.0+), nested parallelism is controlled via **Active Levels**:

```c
// Set maximum nesting depth (e.g., allow 1 level of active parallelism):
omp_set_max_active_levels(1); // Nested regions beyond level 1 are serialized

// Query current nesting level:
int level = omp_get_level(); // 0 in serial, 1 in outer parallel, 2 in inner
```
*(Legacy OpenMP 2.5 used `omp_set_nested(1)` or `export OMP_NESTED=TRUE`).*

---

## 7. Complete Compilable C Program: Runtime Management & Benchmarking

Here is the complete, compilable C program demonstrating the 4-tier precedence rules, runtime environment queries, and high-precision timing:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_runtime_demo.c
 * DESCRIPTION: Complete demonstration of OpenMP thread management:
 *              - Thread precedence rules (OMP_NUM_THREADS vs set_num vs clause)
 *              - Runtime topology queries (omp_get_num_procs, max_threads)
 *              - High-resolution wall-clock timing with omp_get_wtime()
 * COMPILATION: gcc -Wall -O2 openmp_runtime_demo.c -o omp_runtime -fopenmp
 * USAGE:       export OMP_NUM_THREADS=4
 *              ./omp_runtime
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("         OPENMP THREAD MANAGEMENT & RUNTIME APIS DEMO           \n");
    printf("================================================================\n\n");

    // 1. Hardware and Environment Topology Queries
    int num_procs = omp_get_num_procs();
    int max_threads = omp_get_max_threads();
    double timer_tick = omp_get_wtick();

    printf("[System Hardware] Total Online Processor Cores: %d\n", num_procs);
    printf("[Runtime Default] Maximum Available Threads:    %d\n", max_threads);
    printf("[Timer Precision] Hardware Clock Resolution:     %.9e seconds\n\n", timer_tick);

    // Disable dynamic thread adjustment to guarantee deterministic thread allocation
    omp_set_dynamic(0);

    // ------------------------------------------------------------------------
    // DEMO 1: Default Parallel Region (Tier 3: Governed by OMP_NUM_THREADS)
    // ------------------------------------------------------------------------
    printf("--- DEMO 1: Parallel Region using Environment Default ---\n");
    #pragma omp parallel
    {
        #pragma omp single
        printf("  [Team 1] Active Threads in Team: %d\n", omp_get_num_threads());
    }

    // ------------------------------------------------------------------------
    // DEMO 2: Overriding via omp_set_num_threads() (Tier 2 Precedence)
    // ------------------------------------------------------------------------
    printf("\n--- DEMO 2: Overriding with omp_set_num_threads(6) ---\n");
    omp_set_num_threads(6);

    #pragma omp parallel
    {
        #pragma omp single
        printf("  [Team 2] Active Threads in Team: %d (Overrode Environment!)\n", 
               omp_get_num_threads());
    }

    // ------------------------------------------------------------------------
    // DEMO 3: Overriding via num_threads() Clause (Tier 1 Precedence)
    // ------------------------------------------------------------------------
    printf("\n--- DEMO 3: Overriding with num_threads(3) Clause ---\n");
    // Even though omp_set_num_threads requested 6, num_threads(3) takes absolute priority!
    #pragma omp parallel num_threads(3)
    {
        int tid = omp_get_thread_num();
        int total = omp_get_num_threads();
        printf("  [Team 3] Hello from Thread %d of %d (Clause Tier 1 Won!)\n", tid, total);
    }

    // ------------------------------------------------------------------------
    // DEMO 4: Precision Wall-Clock Benchmarking
    // ------------------------------------------------------------------------
    printf("\n--- DEMO 4: Benchmarking with omp_get_wtime() ---\n");
    double start_time = omp_get_wtime();

    #pragma omp parallel num_threads(4)
    {
        int tid = omp_get_thread_num();
        // Simulate computational workload (sleep 100 ms)
        usleep(100000);
        printf("  [Benchmark Worker %d] Finished computation.\n", tid);
    }

    double end_time = omp_get_wtime();
    double elapsed_time = end_time - start_time;

    printf("\n[Benchmark Result] Total Wall-Clock Time: %.6f seconds\n", elapsed_time);
    printf("================================================================\n");

    return 0;
}
```

---

## 8. University Examination Preparation: Solved Questions

### Question 1: Precedence & Allocation (5 Marks)
**Explain the four-tier precedence hierarchy used by OpenMP to determine the number of threads executing a parallel region. Give an example scenario illustrating tier override.**

#### Model Examination Answer:
1. **The Four Tiers (Highest to Lowest):**
   - **Tier 1 — `num_threads(N)` Clause:** Appended directly to a `#pragma omp parallel` directive. Has absolute highest priority and overrides all other settings, but applies strictly to that specific parallel construct.
   - **Tier 2 — `omp_set_num_threads(N)` API:** Called dynamically inside C code. Overrides environment variables for all subsequent parallel regions that do not have an explicit `num_threads` clause.
   - **Tier 3 — `OMP_NUM_THREADS` Environment Variable:** Configured in the shell before program execution (`export OMP_NUM_THREADS=8`). Governs globally if no programmatic overrides are present.
   - **Tier 4 — Implementation Default:** If none of the above are set, the runtime chooses an implementation default, typically equal to the number of available hardware CPU cores.
2. **Override Scenario:**  
   If `export OMP_NUM_THREADS=16` is set in bash, and the C code invokes `omp_set_num_threads(8)`, but a loop is annotated with `#pragma omp parallel num_threads(2)`, the loop will run with **exactly 2 threads** because Tier 1 overrides Tier 2 and Tier 3.

---

### Question 2: API Distinctions (5 Marks)
**Differentiate between `omp_get_num_threads()` and `omp_get_max_threads()`. Why does calling `omp_get_num_threads()` inside sequential code before a parallel region often lead to buffer overflow errors?**

#### Model Examination Answer:
1. **Key Distinctions:**
   - **`omp_get_num_threads()`:** Queries the *currently active* team size. Inside a parallel region running on 8 threads, it returns 8. In serial code outside a parallel region, it returns **1**.
   - **`omp_get_max_threads()`:** Queries the *maximum number of threads* that would be spawned if an unconstrained parallel region were entered at that instant. Safe to call in serial code.
2. **The Buffer Overflow Trap:**  
   If a programmer allocates an array for thread-private data in serial code using:
   ```c
   int n = omp_get_num_threads(); // Evaluates to 1!
   double *arr = malloc(n * sizeof(double)); // Allocates ONLY 1 element!
   ```
   When `#pragma omp parallel num_threads(4)` is entered, threads $1, 2,$ and $3$ attempt to access `arr[1]`, `arr[2]`, and `arr[3]`, causing an out-of-bounds memory write and crashing the program. The developer should have used `omp_get_max_threads()`.

---

### Question 3: Benchmarking Methodology (5 Marks)
**Why should high-performance computing developers use `omp_get_wtime()` instead of the standard C library `clock()` function when profiling parallel OpenMP codes?**

#### Model Examination Answer:
- The standard C library function `clock()` measures **Total Processor CPU Time** accumulated across all physical processor cores. When a parallel region executes across $N$ cores concurrently, `clock()` sums the CPU time of all $N$ cores together (e.g., 4 cores running for 1 second yields 4 seconds of CPU time). Therefore, `clock()` cannot measure real turnaround time or parallel speedup.
- In contrast, `omp_get_wtime()` measures **Real Wall-Clock Time** (elapsed physical time in seconds) using high-resolution hardware timestamp counters. It provides an accurate, monotonic measurement of true parallel elapsed time from start to finish.

---

## 9. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.2 QUICK REVISION SHEET
================================================================================

1. THREAD COUNT PRECEDENCE (HIGHEST TO LOWEST):
   - Tier 1: num_threads(N) clause on #pragma omp parallel
   - Tier 2: omp_set_num_threads(N) runtime API call
   - Tier 3: OMP_NUM_THREADS environment variable
   - Tier 4: Implementation default (logical CPU cores)

2. ESSENTIAL RUNTIME QUERY FUNCTIONS:
   - omp_get_thread_num(): Unique ID (0 to N-1). Master is always 0.
   - omp_get_num_threads(): Active team size (Returns 1 in serial!).
   - omp_get_max_threads(): Max threads available for next parallel region.
   - omp_get_num_procs(): Number of online physical/logical CPU cores.

3. BENCHMARKING RULE:
   - NEVER use clock() for parallel profiling! (Sums CPU time across all cores).
   - ALWAYS use omp_get_wtime() (High-resolution physical wall-clock time).
   - Resolution query: omp_get_wtick()

4. NESTED PARALLELISM:
   - Serialized by default to prevent thread explosion (16 x 16 = 256 threads!).
   - Modern control: omp_set_max_active_levels(levels).
================================================================================
```

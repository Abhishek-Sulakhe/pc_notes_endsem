# OpenMP Architecture, Core Syntax & The Fork-Join Model

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               01 — OpenMP Architecture & The Fork-Join Model
PREREQUISITES:          Subtopic 02.2 (Shared Memory), Subtopic 04.1 (Pthreads vs OpenMP)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 9–11)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Need for High-Level Shared-Memory Parallelism

In Module 04, we explored **POSIX Threads (Pthreads)**. We experienced firsthand the immense burden placed on the programmer:
- You had to manually create arrays of `pthread_t` handles.
- You had to write separate wrapper functions matching `void* (*func)(void*)`.
- You had to manually partition loop iterations among threads.
- You had to explicitly create, lock, unlock, and destroy mutexes, condition variables, and barriers.
- You had to write explicit join loops to clean up memory.

A simple $10$-line sequential loop parallelized with Pthreads often balloons into $80$ lines of complex boilerplate. If you make a subtle mistake in pointer casting or memory allocation, your program suffers segmentation faults or deadlocks.

In 1997, a consortium of major computer hardware and software vendors—including Intel, SGI, IBM, Sun Microsystems, and Compaq—formed the **OpenMP Architecture Review Board (ARB)** (`http://www.openmp.org/`) to establish a standardized, high-level, portable shared-memory API:

$$\boxed{\textbf{OpenMP} = \textbf{Open } \textbf{M}\text{ulti-}\textbf{P}\text{rocessing}}$$

### The OpenMP Philosophy: Incremental Parallelism
Unlike Pthreads, which demands an all-or-nothing rewrite of your application, OpenMP is designed for **Incremental Parallelization**:
1. You begin with a working, fully tested **sequential C/C++ program**.
2. You profile the code to find the most compute-intensive for-loops (the hot spots).
3. You insert a single compiler directive (a `#pragma`) above the loop.
4. The OpenMP compiler automatically generates the multithreaded worker routines, spawns the thread pool, splits the iteration space, and synchronizes the results.
5. If compiled without OpenMP support, the directives are treated as standard comments, and the exact same source code runs sequentially!

---

## 2. The Three Architectural Pillars of OpenMP

As illustrated in your lecture notes (`3d773f8d.pdf`, Page 10), OpenMP is not merely a C library; it is a layered ecosystem built on **three cooperating components**:

```
+-------------------------------------------------------------------------------+
|                       THE THREE PILLARS OF OPENMP                             |
+-------------------------------------------------------------------------------+
| 1. COMPILER DIRECTIVES (#pragma omp ...):                                     |
|    - Embedded directly in C/C++ source code.                                  |
|    - Instructs the compiler to create parallel regions, distribute loop       |
|      iterations across threads, define data scoping (shared vs private), and   |
|      insert synchronization points.                                           |
|                                                                               |
| 2. RUNTIME LIBRARY ROUTINES (<omp.h>):                                        |
|    - Standard C functions callable at runtime.                                |
|    - Used to query and alter the execution environment dynamically:           |
|      * `omp_get_thread_num()`: Query calling thread's unique ID.               |
|      * `omp_get_num_threads()`: Query total threads in the current team.       |
|      * `omp_set_num_threads(N)`: Request N threads for future regions.         |
|      * Timer functions (`omp_get_wtime()`).                                   |
|                                                                               |
| 3. ENVIRONMENT VARIABLES:                                                     |
|    - Shell variables inspected by the OpenMP runtime before program execution.|
|    - Allows end-users to tune performance without recompiling source code:    |
|      * `export OMP_NUM_THREADS=8`                                             |
|      * `export OMP_SCHEDULE="dynamic,100"`                                    |
|      * `export OMP_PROC_BIND=close`                                           |
+-------------------------------------------------------------------------------+
```

---

### The Layered Software Architecture Diagram

From `3d773f8d.pdf` (Page 10), OpenMP reconciles the programmer's view of the code with the user's execution environment:

```
+----------------------------------+----------------------------------+
|        Programmer's View         |           User's View            |
|       Application Source         |       Execution Environment      |
|     [Compiler Directives/Pragmas]|     [Environment Variables]      |
+-----------------+----------------+-----------------+----------------+
                  |                                  |
                  +----------------+-----------------+
                                   |
                                   v
             +---------------------------------------------+
             |            OpenMP Runtime Library           |
             |       (libgomp.so / libomp.so)              |
             +---------------------+-----------------------+
                                   |
                                   v
             +---------------------------------------------+
             |             Threads in the OS               |
             |      (Kernel Lightweight Processes - LWPs)  |
             +---------------------+-----------------------+
                                   |
                                   v
             +---------------------------------------------+
             |             Cores in Hardware               |
             |      (Physical CPUs, Sockets & Caches)      |
             +---------------------------------------------+
```

---

## 3. The Fork-Join Execution Model

OpenMP programs execute according to the **Fork-Join Execution Model**.

```
========================================================================================
                       THE OPENMP FORK-JOIN EXECUTION MODEL
========================================================================================

Master Thread (Thread 0)
     ||
     ||  <-- SERIAL REGION: Master thread executes single-threaded instructions
     ||
    ==== FORK: Master thread encounters `#pragma omp parallel`
    || | | |   Spawns/awakens a team of worker threads (Threads 0, 1, 2, ..., N-1)
    || | | |
    || | | |   PARALLEL REGION: All threads execute the structured block
    || | | |   concurrently across physical CPU cores
    || | | |
    ==== JOIN: Implicit synchronization barrier at the closing brace `}`
     ||        Worker threads suspend/sleep; Master thread continues sequentially
     ||
     ||  <-- SERIAL REGION: Master thread prepares data for the next phase
     ||
    ==== FORK: Next `#pragma omp parallel` encountered
    || | | |
    || | | |   PARALLEL REGION: Team executes second parallel task
    || | | |
    ==== JOIN: Implicit barrier; workers join
     ||
     ||  <-- Program reaches `return 0;` and terminates
========================================================================================
```

---

### Step-by-Step Mechanics of the Fork-Join Lifecycle

1. **Initial Execution:**  
   When the process launches, it begins as a standard sequential program executed by a single initial thread called the **Master Thread** (assigned Thread ID `0`).
2. **The FORK Operation:**  
   When the Master Thread encounters a parallel construct (`#pragma omp parallel`), it forks a team of $N$ threads:
   - The Master Thread remains Thread `0` and becomes the leader of the team.
   - $(N - 1)$ additional **worker threads** are allocated from an underlying thread pool and assigned IDs $1, 2, \dots, N-1$.
3. **Concurrent Execution:**  
   All $N$ threads execute the statements enclosed within the parallel construct's **Structured Block** concurrently.
4. **The JOIN Operation:**  
   At the closing curly brace `}` of the parallel region, the OpenMP compiler inserts an **implicit synchronization barrier**:
   - Fast threads pause and wait until the slowest thread arrives.
   - Once all threads arrive, the team is disassembled: worker threads return to an idle thread pool (or sleep), and **only the Master Thread continues** executing sequentially past the barrier.

---

## 4. OpenMP Directive Syntax & Structured Blocks

### General Directive Syntax in C/C++

In C and C++, OpenMP directives are declared using the `#pragma` preprocessor mechanism:

```c
#pragma omp <directive-name> [clause[[,] clause] ...]
    structured-block
```

- `#pragma`: Standard C language preprocessor directive. Any compiler that does not recognize a specific pragma will simply ignore it with a minor warning, ensuring source code portability.
- `omp`: The mandatory **Pragma Sentinel** identifying the directive as an OpenMP instruction.
- `directive-name`: The specific parallel operation (e.g., `parallel`, `for`, `single`, `critical`, `barrier`, `task`).
- `clause`: Optional modifiers customizing the directive's behavior (e.g., `num_threads(4)`, `private(x)`, `shared(y)`, `reduction(+:sum)`).

---

### The Strict Definition of a "Structured Block"

The code block governed by an OpenMP directive must be a **Structured Block**:

$$\boxed{\textbf{Structured Block: } \text{A C/C++ compound statement with strictly \textbf{one entry point} at the top}}$$
$$\boxed{\text{and \textbf{one exit point} at the bottom.}}$$

```
+-------------------------------------------------------------------------------+
|                      RULES OF THE STRUCTURED BLOCK                           |
+-------------------------------------------------------------------------------+
| ALLOWED:                                                                      |
| - Loops (`for`, `while`, `do-while`), conditional branches (`if-else`),       |
|   function calls, and internal jumps entirely contained WITHIN the block.     |
|                                                                               |
| FORBIDDEN (COMPILER ERROR):                                                   |
| - You CANNOT jump INTO the middle of a block from outside using `goto`.       |
| - You CANNOT jump OUT OF the block to the outside using `goto` or `break`.    |
| - You CANNOT use `return` inside a parallel structured block!                 |
| - You CAN call `exit()` because `exit()` destroys the entire process.        |
+-------------------------------------------------------------------------------+
```

```c
// ILLEGAL CODE EXAMPLE:
#pragma omp parallel
{
    if (error_detected) {
        return -1; // COMPILER ERROR! Cannot return out of an OpenMP structured block!
    }
}
```

---

## 5. Conditional Compilation & The `_OPENMP` Macro

Every OpenMP-compliant compiler automatically defines a standard preprocessor macro named **`_OPENMP`**.

### The Version Format
The `_OPENMP` macro is defined as an integer in **`YYYYMM` format**, representing the year and month of the OpenMP specification supported by the compiler:

```
+-------------------------------------------------------------------------------+
|                       OPENMP SPECIFICATION VERSION CODES                      |
+-------------------------------------------------------------------------------+
| `_OPENMP` Value       | Specification Version | Major Capabilities            |
| :-------------------- | :-------------------- | :---------------------------- |
| `199810`              | OpenMP 1.0 (C/C++)    | Initial Fork-Join, basic loops|
| `200203`              | OpenMP 2.0            | Dynamic scheduling, workshare |
| `200805`              | OpenMP 3.0            | Task Parallelism introduced   |
| `201107`              | OpenMP 3.1            | Atomic enhancements           |
| `201307`              | OpenMP 4.0            | Accelerator/GPU offload, SIMD |
| `201511`              | OpenMP 4.5            | Advanced GPU offloading       |
| `201811`              | OpenMP 5.0            | Modern tasking, memory spaces |
| `202011`              | OpenMP 5.1            | Full C++20 support            |
+-------------------------------------------------------------------------------+
```

### Writing Dual-Mode (Sequential + Parallel) Portable Code

As transcribed from `3d773f8d.pdf` (Page 11), you can write code that runs sequentially on standard compilers while activating parallel optimizations on OpenMP compilers using `#ifdef _OPENMP`:

```c
#include <stdio.h>

#ifdef _OPENMP
    #include <omp.h> // Include OpenMP runtime header only if supported
#endif

int main() {
#ifdef _OPENMP
    printf("[System] Compiled with OpenMP Support!\n");
    printf("[System] OpenMP Specification Version Code: %d\n", _OPENMP);
#else
    printf("[System] Compiled without OpenMP. Running in pure sequential mode.\n");
#endif
    return 0;
}
```

---

## 6. How Compilers Transform OpenMP Behind the Scenes

When you compile a C program with GCC using `-fopenmp`, how does the compiler convert `#pragma omp parallel` into actual executing threads?

The compiler performs **Code Outlining**:

```
+-------------------------------------------------------------------------------+
|                         COMPILER CODE OUTLINING                               |
+-------------------------------------------------------------------------------+
| 1. OUTLINING:                                                                 |
|    - The compiler extracts the statements inside the parallel structured      |
|      block and creates an independent, hidden function:                       |
|      `void __subfunction_outlined(void *data_payload);`                       |
|                                                                               |
| 2. RUNTIME INVOCATION:                                                        |
|    - The compiler replaces the `#pragma omp parallel` directive with a call   |
|      to the internal OpenMP runtime library:                                  |
|      In GCC (libgomp):   `GOMP_parallel(__subfunction_outlined, ...);`        |
|      In LLVM/Clang:      `__kmpc_fork_call(..., __subfunction_outlined);`     |
|                                                                               |
| 3. THREAD POOL DISPATCH:                                                      |
|    - The runtime library dispatches `__subfunction_outlined` across its       |
|      pre-spawned pool of POSIX threads (Pthreads) on Linux!                   |
+-------------------------------------------------------------------------------+
```

> [!NOTE]
> **OpenMP is Built on Top of Pthreads!**  
> On Linux, the OpenMP runtime library (`libgomp.so`) is implemented directly using standard POSIX threads (`libpthread.so`). OpenMP does not replace the operating system's threading model; it provides an automated, compiler-driven interface over Pthreads.

---

## 7. Complete Compilable C Program: OpenMP Hello World

Here is the complete, compilable OpenMP program demonstrating parallel execution, runtime API queries, and version macro detection:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_hello_world.c
 * DESCRIPTION: First OpenMP parallel program demonstrating the Fork-Join model,
 *              compiler macro detection, thread ID queries, and team size.
 * COMPILATION: gcc -Wall -O2 openmp_hello_world.c -o omp_hello -fopenmp
 * USAGE:       ./omp_hello
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>

// Conditionally include omp.h only when OpenMP is active
#ifdef _OPENMP
    #include <omp.h>
#endif

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("             OPENMP ARCHITECTURE: HELLO WORLD DEMO              \n");
    printf("================================================================\n");

    // Phase 1: Sequential Verification of OpenMP Macro
#ifdef _OPENMP
    printf("[Main Serial] OpenMP is ENABLED (Macro _OPENMP = %d)\n\n", _OPENMP);
#else
    printf("[Main Serial] OpenMP is DISABLED. Compiling as serial code.\n\n");
#endif

    // Phase 2: Parallel Region (Fork-Join)
    // The `num_threads(4)` clause explicitly requests a team of 4 threads.
    #pragma omp parallel num_threads(4)
    {
        // Every thread in the team executes this structured block concurrently!
        int thread_id = 0;
        int total_threads = 1;

#ifdef _OPENMP
        // Query runtime library for thread credentials
        thread_id = omp_get_thread_num();     // Unique ID: 0, 1, 2, ..., N-1
        total_threads = omp_get_num_threads(); // Team capacity: N
#endif

        printf("  [Parallel Team] Hello World from Thread %d of %d!\n", 
               thread_id, total_threads);

        // Notice: At the closing brace '}', an IMPLICIT BARRIER synchronizes all threads.
    }

    // Phase 3: Resumed Serial Execution
    printf("\n[Main Serial] Parallel region exited. Back to single-threaded master execution.\n");
    printf("[Main Serial] Program finished successfully.\n");
    return 0;
}
```

### Compilation Flags Across Compilers

```bash
# GNU Compiler Collection (GCC):
gcc -Wall -O2 openmp_hello_world.c -o omp_hello -fopenmp

# LLVM / Clang:
clang -Wall -O2 openmp_hello_world.c -o omp_hello -fopenmp

# Intel oneAPI C/C++ Compiler (icx):
icx -Wall -O2 openmp_hello_world.c -o omp_hello -qopenmp

# Microsoft Visual C++ (MSVC on Windows):
cl /O2 /openmp openmp_hello_world.c
```

---

## 8. University Examination Preparation: Solved Questions

### Question 1: Core Architecture (5 Marks)
**Explain the Fork-Join execution model of OpenMP with the help of a neat diagram. What happens at the end of a parallel region?**

#### Model Examination Answer:
1. **Explanation of the Fork-Join Model:**  
   An OpenMP program begins execution as a single sequential thread of control called the **Master Thread** (Thread `0`). When the Master Thread encounters a `#pragma omp parallel` directive:
   - **FORK:** The master thread creates or awakens a team of worker threads (Threads $0$ to $N-1$).
   - **Parallel Region:** All threads in the team execute the statements enclosed in the parallel construct's structured block concurrently across available CPU cores.
   - **JOIN:** At the closing brace `}` of the parallel region, execution encounters an **implicit synchronization barrier**. Worker threads finish their work and suspend execution, returning to an idle pool. Only the single Master Thread continues sequentially past the barrier.
2. **What Happens at the End of a Parallel Region:**  
   The runtime enforces an implicit barrier. All threads synchronize so that no thread proceeds until all members of the team have completed the structured block. The team is dissolved, and single-threaded sequential execution resumes.
3. **Diagram:**  
   *(Reproduce the ASCII Fork-Join timeline diagram from Section 3).*

---

### Question 2: Syntax & Mechanics (5 Marks)
**What is a "Structured Block" in OpenMP? Why are jumps into or out of a structured block strictly prohibited?**

#### Model Examination Answer:
1. **Definition:** A structured block in OpenMP is a single C/C++ statement or compound statement enclosed in curly braces `{ ... }` that has strictly **one entry point at the beginning** and **one exit point at the end**.
2. **Prohibition of Jumps:**  
   - Jumps into the middle of a block (via `goto`) or out of the block (via `goto`, `break`, or `return`) are prohibited because the OpenMP compiler must insert runtime setup calls at the entry point (e.g., allocating thread stacks, setting up data sharing tables) and synchronization cleanup calls at the exit point (e.g., the implicit barrier).
   - Jumping out of a block would bypass the mandatory implicit barrier, corrupting the runtime thread pool and causing indefinite deadlocks among the remaining threads in the team.

---

### Question 3: Component Taxonomy (5 Marks)
**Describe the three architectural pillars of OpenMP. Give one concrete example of each.**

#### Model Examination Answer:
1. **Compiler Directives (Pragmas):** High-level annotations starting with `#pragma omp` that instruct the compiler to create threads and distribute work. *Example:* `#pragma omp parallel for`.
2. **Runtime Library Functions:** C API functions included via `<omp.h>` that allow dynamic queries and control of execution state. *Example:* `omp_get_thread_num()` (returns the calling thread's ID).
3. **Environment Variables:** External shell variables inspected before execution to configure system parameters without recompiling. *Example:* `export OMP_NUM_THREADS=8` (instructs the runtime to use 8 threads).

---

## 9. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.1 QUICK REVISION SHEET
================================================================================

1. WHAT IS OPENMP?
   - Open Multi-Processing: Standardized shared-memory API based on compiler directives.
   - Core design: Incremental parallelism (parallelize hot spots without rewriting code).

2. THE THREE ARCHITECTURAL PILLARS:
   - Directives (#pragma omp ...)
   - Runtime APIs (<omp.h>, omp_get_thread_num())
   - Environment Variables (OMP_NUM_THREADS)

3. FORK-JOIN MODEL:
   - Serial execution by Master Thread (Thread 0) -> FORK into team of N threads 
     -> Concurrent execution of structured block -> JOIN at implicit barrier.

4. STRUCTURED BLOCK RULES:
   - Exactly ONE entry at top, ONE exit at bottom.
   - NO return, goto, or break jumping out of the block!

5. COMPILER SENTINELS & FLAGS:
   - C/C++ Sentinel: #pragma omp
   - Version Macro: _OPENMP (formatted as YYYYMM integer, e.g. 201511)
   - GCC/Clang flag: -fopenmp
================================================================================
```

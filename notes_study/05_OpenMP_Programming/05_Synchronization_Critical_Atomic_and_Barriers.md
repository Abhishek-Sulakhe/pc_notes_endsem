# Synchronization in OpenMP: Critical, Atomic, Barriers, Master & Single

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               05 — Synchronization Constructs & Mutual Exclusion
PREREQUISITES:          Subtopic 04.3 (Mutexes & Critical Sections), Subtopic 05.1 (Fork-Join),
                        Subtopic 05.3 (Data Scoping)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 19–21)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Need for Synchronization in OpenMP

In Subtopic 05.3, we established that variables declared outside an OpenMP parallel construct are **Shared** by default. While shared memory enables seamless communication, concurrent writes to a shared variable create lethal race conditions.

To ensure program correctness without falling back to complex low-level Pthreads code, OpenMP provides high-level, declarative **Synchronization Directives**:
1. **Critical Sections (`#pragma omp critical`):** Software-level mutual exclusion over arbitrary blocks of code.
2. **Atomic Operations (`#pragma omp atomic`):** Hardware-level, ultra-low-latency atomic instructions on single memory words.
3. **Explicit Barriers (`#pragma omp barrier`):** Enforced phase-coordination rendezvous points.
4. **Master & Single Directives (`#pragma omp master` / `#pragma omp single`):** Designating specific single-thread execution blocks within a parallel team.

---

## 2. Critical Sections: Software-Level Mutual Exclusion

$$\boxed{\textbf{\#pragma omp critical: } \text{Restricts execution of an enclosed structured code block to strictly}}$$
$$\boxed{\text{\textbf{one thread at a time}, enforcing mutual exclusion across all threads in the team.}}$$

```c
#pragma omp parallel
{
    // Concurrent parallel computation...
    int local_result = compute(omp_get_thread_num());

    #pragma omp critical
    {
        // CRITICAL SECTION: Only ONE thread executes here at a time!
        global_sum += local_result;
        printf("Thread %d updated global sum to: %d\n", omp_get_thread_num(), global_sum);
    }
}
```

### The Software Lock Architecture of `critical`
Behind the scenes, the OpenMP compiler wraps the structured block inside an internal **Pthreads Mutex**:
- When Thread 0 enters the critical block, it acquires an internal mutex lock.
- If Thread 1, Thread 2, and Thread 3 arrive while Thread 0 is inside, they are **blocked (suspended)** and put to sleep in a queue.
- **Performance Threat:** As noted in your lecture notes (`3d773f8d.pdf`, Page 19):  
  *"Critical sections have high execution overhead and can easily become a major framework-level bottleneck that serializes execution!"*

---

### Unnamed vs Named Critical Sections

By default, an unnamed `#pragma omp critical` shares a **single global lock** across the entire application:

```c
// DANGEROUS CONTENTION: BOTH CRITICAL SECTIONS SHARE THE SAME GLOBAL LOCK!
#pragma omp critical
{
    update_bank_balance();
}

// In a completely different function or thread:
#pragma omp critical
{
    log_network_packet(); // STALLED! Waiting for update_bank_balance to finish!
}
```

Even though `update_bank_balance` and `log_network_packet` access completely unrelated data, they block each other!

#### The Solution: Named Critical Sections
OpenMP allows developers to assign unique names to critical sections:

```c
#pragma omp critical (bank_lock)
{
    update_bank_balance(); // Only blocks other 'bank_lock' critical sections!
}

#pragma omp critical (log_lock)
{
    log_network_packet();  // Runs concurrently with bank updates!
}
```

---

## 3. Atomic Operations: Hardware-Level Data Protection

When protecting a **single scalar assignment** (such as incrementing a counter or accumulating a sum), using `#pragma omp critical` is like using a sledgehammer to swat a fly.

OpenMP provides a high-performance alternative: **`#pragma omp atomic`**.

$$\boxed{\textbf{\#pragma omp atomic: } \text{Ensures that a single specific memory location is read, modified,}}$$
$$\boxed{\text{and written atomically using hardware-supported CPU instructions without software locks.}}$$

```c
#pragma omp parallel
{
    int local_val = calculate();
    
    #pragma omp atomic
    global_counter += local_val; // Hardware atomic operation!
}
```

---

### Critical vs Atomic: The Hardware Performance Chasm

From your lecture notes (`3d773f8d.pdf`, Page 19):

```
+-----------------------------------------------------------------------------------+
|                        CRITICAL vs ATOMIC: DEEP COMPARISON                        |
+-----------------------------------------------------------------------------------+
| Feature Dimension     | `#pragma omp critical`       | `#pragma omp atomic`               |
| :-------------------- | :--------------------------- | :--------------------------------- |
| **Scope of Protection**| **Entire block of code:**     | **Single memory location:**        |
|                       | Multiple statements, I/O,    | Exactly one scalar assignment or    |
|                       | function calls, math loops   | increment/decrement statement      |
| **Implementation**    | **Software Lock (Mutex):**   | **Hardware Bus/Cache Protocol:**   |
|                       | OS kernel queue, lock calls  | Direct CPU instructions:           |
|                       | (`pthread_mutex_lock`)        | `LOCK XADD`, `LOCK CMPXCHG` on x86 |
| **Execution Latency** | **Slow & Heavy:**            | **Ultra-Fast & Lightweight:**      |
|                       | $\sim 50$ to $200$ CPU cycles| $\sim 5$ to $15$ CPU cycles        |
| **Throughput Advantage**| Baseline                    | **$5\times$ to $10\times$ faster!**|
| **Permitted Code**    | Arbitrary C/C++ statements    | Strictly restricted to:            |
|                       | (even `printf` allowed)       | `x++`, `x--`, `x += expr`, etc.    |
+-----------------------------------------------------------------------------------+
```

### Supported Atomic Clauses (Modern OpenMP)
- `update` (Default): Atomic read-modify-write (`x += expr`).
- `read`: Atomic read of a variable (`v = x`).
- `write`: Atomic assignment to a variable (`x = expr`).
- `capture`: Atomic read-and-modify, capturing the old or new value:
  ```c
  #pragma omp atomic capture
  ticket_id = next_ticket++; // Retrieves current value and increments atomically!
  ```

---

## 4. Explicit Barrier Synchronization

In Subtopic 05.4, we learned that `#pragma omp for` inserts an *implicit* barrier at the end of a loop. However, complex parallel algorithms frequently require synchronization **in the middle of a parallel region**.

$$\boxed{\textbf{\#pragma omp barrier: } \text{Forces every thread in the team to halt execution at that exact}}$$
$$\boxed{\text{point, blocking each thread until all threads in the team have arrived.}}$$

---

### The Two-Stage Parallel Pipeline Example

Directly transcribed from your lecture notes (`3d773f8d.pdf`, Page 20):

```c
#pragma omp parallel num_threads(4)
{
    int i = omp_get_thread_num();

    // ------------------------------------------------------------------------
    // STAGE 1: Compute square of local array elements
    // ------------------------------------------------------------------------
    b[i] = a[i] * a[i];
    printf("Thread %d finished Stage 1 (Computed b[%d] = %d)\n", i, i, b[i]);

    // MANDATORY BARRIER: No thread can proceed to Stage 2 until ALL threads
    // have completely finished populating their respective b[i] entries!
    #pragma omp barrier

    // ------------------------------------------------------------------------
    // STAGE 2: Cross-Array Transformation (Depends on Stage 1 completion!)
    // ------------------------------------------------------------------------
    // Suppose Stage 2 requires reading a neighbor's computed value: b[(i+1)%4]
    c[i] = b[(i + 1) % 4] + 10;
    printf("Thread %d finished Stage 2 (Computed c[%d] = %d)\n", i, i, c[i]);
}
```

### The Lethal Barrier Deadlock Rule

> [!CAUTION]
> **The All-or-None Barrier Rule:**  
> A barrier must be encountered by **all threads in the team, or by none of them**!  
> If you place `#pragma omp barrier` inside a conditional branch that only some threads take:
> ```c
> // DEADLY BUG: PERMANENT DEADLOCK!
> if (omp_get_thread_num() % 2 == 0) {
>     #pragma omp barrier // Even threads wait for odd threads that NEVER ARRIVE!
> }
> ```
> The even-numbered threads wait forever for the odd-numbered threads to arrive at the barrier, causing a permanent system freeze.

---

## 5. Master vs Single: Coordinating Single-Threaded Tasks

Often, a block of code inside a parallel region must be executed by **only one thread** (e.g., printing a progress banner, prompting user input, or reading an initial file header).

OpenMP provides two constructs for this purpose:
- **`#pragma omp master`** (Modern OpenMP 5.1: `#pragma omp masked`)
- **`#pragma omp single`**

```
+-----------------------------------------------------------------------------------+
|                        MASTER vs SINGLE: ARCHITECTURAL COMPARISON                 |
+-----------------------------------------------------------------------------------+
| Feature Dimension     | `#pragma omp master`         | `#pragma omp single`               |
| :-------------------- | :--------------------------- | :--------------------------------- |
| **Executing Thread**  | **Strictly Thread 0**        | **Whichever thread arrives first** |
|                       | (The Master Thread)          | (Arbitrary worker or master)       |
| **Implicit Barrier**  | **NO IMPLICIT BARRIER!**     | **HAS AN IMPLICIT BARRIER!**       |
|                       | Other threads continue       | Other threads wait at the closing  |
|                       | executing without waiting!   | brace until execution finishes!    |
| **Clause Support**    | Does not accept `nowait`     | Accepts `nowait` to remove barrier |
| **Common Use Case**   | Simple debug logging by      | Thread-safe file I/O, allocating   |
|                       | Thread 0 without stalling    | dynamic memory buffers that the    |
|                       | working peers                | entire team will immediately use   |
+-----------------------------------------------------------------------------------+
```

```
========================================================================================
                      TIMELINE: MASTER vs SINGLE EXECUTION
========================================================================================

#pragma omp master (NO Barrier):
Thread 0: [=== Executes Master Block ===]-------------------------> Continues
Thread 1: -------- (Skips block and keeps running!) ---------------> Continues
Thread 2: -------- (Skips block and keeps running!) ---------------> Continues

#pragma omp single (HAS Barrier):
Thread 1: (Arrives first!) [=== Executes Single Block ===]---------> Leaves block
Thread 0: ----------------> WAITING AT IMPLICIT BARRIER -----------> Resumes
Thread 2: ----------------> WAITING AT IMPLICIT BARRIER -----------> Resumes
========================================================================================
```

---

## 6. Complete Compilable C Program: OpenMP Synchronization Suite

Here is the complete C application demonstrating `critical`, `atomic`, `barrier`, `master`, and `single` in action:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_sync_suite.c
 * DESCRIPTION: Comprehensive demonstration of OpenMP synchronization constructs:
 *              - Named critical sections
 *              - Hardware atomic updates
 *              - Explicit barrier pipeline stages
 *              - Master vs Single constructs with barrier behavior
 * COMPILATION: gcc -Wall -O2 openmp_sync_suite.c -o omp_sync -fopenmp
 * USAGE:       ./omp_sync
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <unistd.h>

#define NUM_THREADS 4

int global_counter = 0;
int shared_pipeline_data[NUM_THREADS];

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("           OPENMP SYNCHRONIZATION CONSTRUCTS SUITE              \n");
    printf("   Executing with %d Threads\n", NUM_THREADS);
    printf("================================================================\n\n");

    #pragma omp parallel num_threads(NUM_THREADS)
    {
        int tid = omp_get_thread_num();

        // --------------------------------------------------------------------
        // 1. MASTER DIRECTIVE (No Barrier)
        // --------------------------------------------------------------------
        #pragma omp master
        {
            printf("[Master Directive] Thread %d (Master) initializing pipeline setup...\n", tid);
        }

        // --------------------------------------------------------------------
        // 2. STAGE 1: Parallel Computation
        // --------------------------------------------------------------------
        shared_pipeline_data[tid] = (tid + 1) * 10;
        printf("  [Worker %d] Completed Stage 1 calculation: data[%d] = %d\n", 
               tid, tid, shared_pipeline_data[tid]);

        // --------------------------------------------------------------------
        // 3. EXPLICIT BARRIER: Coordinate Pipeline Transition
        // --------------------------------------------------------------------
        #pragma omp barrier

        // --------------------------------------------------------------------
        // 4. SINGLE DIRECTIVE (Has Implicit Barrier)
        // --------------------------------------------------------------------
        #pragma omp single
        {
            printf("\n>>> [Single Directive] Executed by Thread %d (First to arrive!) <<<\n", tid);
            printf(">>> All threads verified to have completed Stage 1 via Barrier! <<<\n\n");
        }

        // --------------------------------------------------------------------
        // 5. ATOMIC INCREMENT (Hardware-Level Protection)
        // --------------------------------------------------------------------
        #pragma omp atomic
        global_counter++;

        // --------------------------------------------------------------------
        // 6. NAMED CRITICAL SECTION (Software-Level Mutual Exclusion)
        // --------------------------------------------------------------------
        #pragma omp critical (io_display_lock)
        {
            printf("  [Critical Section: io_lock] Thread %d inside critical section. Counter = %d\n", 
                   tid, global_counter);
        }
    }

    printf("\n================================================================\n");
    printf("Final Global Counter Value: %d (Expected: %d)\n", global_counter, NUM_THREADS);
    printf("[Main] All synchronization constructs executed successfully!\n");
    printf("================================================================\n");

    return 0;
}
```

---

## 7. University Examination Preparation: Solved Questions

### Question 1: Core Comparison (10 Marks)
**Compare `#pragma omp critical` and `#pragma omp atomic`. Explain their underlying hardware/software mechanisms and why `atomic` delivers significantly higher throughput for scalar variable updates.**

#### Model Examination Answer:
1. **Mechanical Comparison:**
   - **`#pragma omp critical` (Software Lock):**  
     Implements mutual exclusion over an arbitrary structured block of code. The OpenMP compiler wraps the block in an internal software lock/mutex (similar to a Pthreads mutex). Threads competing for the critical section incur OS context-switch and lock acquisition overhead (~50–200 cycles). It can serialize entire execution blocks.
   - **`#pragma omp atomic` (Hardware Instruction):**  
     Protects a single specific scalar memory location during a read-modify-write operation (`x++`, `x += expr`). It completely bypasses software locks and relies on atomic instructions provided directly by the physical CPU architecture (such as the `LOCK` prefix on x86, which locks cache lines via the MESI cache coherence protocol).
2. **Why `atomic` is Faster:**  
   `atomic` executes in only $\sim 5$ to $15$ CPU clock cycles because it does not involve OS kernel queues, thread suspension, or mutex data structures. For high-frequency scalar updates (such as histogram binning or counter increments), `atomic` achieves $5\times$ to $10\times$ higher throughput than `critical`.
3. **Limitation of `atomic`:**  
   `atomic` can only protect a single scalar assignment statement; it cannot protect multi-statement logic, complex data structures, or I/O operations.

---

### Question 2: Architectural Comparison (5 Marks)
**Differentiate between `#pragma omp master` and `#pragma omp single`. Explain the implicit barrier behavior of both directives.**

#### Model Examination Answer:
1. **Executing Thread:**
   - **`master`:** The enclosed block is executed **exclusively by Thread 0** (the Master Thread of the team).
   - **`single`:** The enclosed block is executed by **any single thread** that arrives first at the construct.
2. **Implicit Barrier Behavior:**
   - **`master` has NO implicit barrier:** The other threads in the team skip the master block and immediately proceed to subsequent code without waiting.
   - **`single` HAS an implicit barrier:** By default, all other threads in the team must pause at the closing brace and wait until the single executing thread completes the block. (This barrier can be eliminated by appending the `nowait` clause: `#pragma omp single nowait`).

---

### Question 3: Synchronization Trap (5 Marks)
**Why is it illegal to place an explicit `#pragma omp barrier` inside an `if-else` branch where only some threads enter? What happens at runtime?**

#### Model Examination Answer:
- **The Rule:** In OpenMP, a barrier directive must be reached by **all threads in the team or none of them**.
- **What Happens at Runtime (Deadlock):**  
  If a barrier is placed inside `if (omp_get_thread_num() == 0)`, only Thread 0 executes the barrier call. Thread 0 blocks in the barrier queue, waiting for Threads $1, 2, \dots, N-1$ to arrive. Because the other threads took the `else` branch or bypassed the `if`, they never call the barrier. Thread 0 waits forever, freezing the entire process in a permanent **deadlock**.

---

## 8. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.5 QUICK REVISION SHEET
================================================================================

1. CRITICAL SECTIONS (#pragma omp critical [(name)]):
   - Software mutex lock protecting an arbitrary multi-statement block.
   - High overhead! Use named critical sections to avoid global false contention.

2. ATOMIC DIRECTIVE (#pragma omp atomic):
   - Hardware-level CPU instruction (e.g. LOCK XADD) protecting a SINGLE memory word.
   - 5x to 10x faster than critical! Restricted to scalar updates (x++, x += expr).

3. EXPLICIT BARRIER (#pragma omp barrier):
   - Enforces a full-team rendezvous point between pipeline phases.
   - MANDATORY: All threads in the team must reach it, or none (Deadlock risk!).

4. MASTER vs SINGLE:
   - master: Strictly Thread 0. NO implicit barrier!
   - single: First thread to arrive. HAS an implicit barrier (removable via nowait).
================================================================================
```

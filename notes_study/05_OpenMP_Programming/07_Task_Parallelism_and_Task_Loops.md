# Task Parallelism & Task Loops in OpenMP

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               07 — Task Parallelism, Task Pools & Task Loops
PREREQUISITES:          Subtopic 05.1 (Fork-Join), Subtopic 05.4 (Loop Worksharing),
                        Subtopic 05.5 (Master & Single Directives)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 23–24)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Failure of Loop Worksharing on Irregular Problems

In Subtopics 05.1 through 05.6, our parallelization strategy focused almost entirely on **Data Parallelism via Loop Worksharing** (`#pragma omp for`). 

Loop worksharing is ideal when:
- The total iteration count ($N$) is known in advance.
- Arrays can be indexed directly with random access: `array[i]`.
- Loop bounds conform strictly to the Canonical Loop Form.

### The Irregular Computing Dilemma
Real-world computer science is filled with complex, non-linear algorithms where canonical for-loops cannot be applied:
1. **Dynamic Pointer Chasing (Linked Lists & Graphs):** Traversing a linked list (`while (node != NULL) node = node->next;`). You cannot calculate the list length in advance, nor can you jump directly to node 50.
2. **Recursive Divide-and-Conquer Algorithms:** Tree traversals, QuickSort, MergeSort, and recursive Fibonacci evaluations.
3. **Unbounded While-Loops:** Search algorithms and game tree evaluations where termination depends on runtime convergence criteria.

If you attempt to parallelize a linked list traversal with `#pragma omp for`, the compiler will reject it because the loop is not an indexable for-loop.

To conquer irregular, asynchronous workloads, OpenMP 3.0 introduced **Task Parallelism**:

$$\boxed{\textbf{Task Parallelism: } \text{An execution model where independent, asynchronous units of work (Tasks)}}$$
$$\boxed{\text{are dynamically created, deposited into a Task Pool, and executed by available worker threads.}}$$

---

## 2. The Task Pool Architecture & The Single Producer Pattern

A **Task** in OpenMP consists of three core components:
1. **Code:** The structured block of C/C++ statements to execute.
2. **Data Environment:** The variables associated with the task (captured as `shared` or `firstprivate`).
3. **Internal Control Variables (ICVs):** State information controlling task scheduling.

```
========================================================================================
                       THE OPENMP TASK POOL ARCHITECTURE
========================================================================================

                 [ MASTER THREAD (Inside #pragma omp single) ]
                                      |
                      Generates Tasks & pushes to queue:
                      #pragma omp task { process(node); }
                                      |
                                      v
                 +-----------------------------------------+
                 |          CENTRALIZED TASK POOL          |
                 |  [Task 1]  [Task 2]  [Task 3]  [Task 4] |
                 +--------------------+--------------------+
                                      |
               Idle worker threads pull tasks asynchronously:
                                      |
              +-----------------------+-----------------------+
              |                       |                       |
              v                       v                       v
         [ Thread 0 ]            [ Thread 1 ]            [ Thread 2 ]
       Executes Task 1         Executes Task 2         Executes Task 3
========================================================================================
```

---

### The Single Producer Pattern

When spawning tasks, beginner programmers invariably make a catastrophic mistake:

```c
// CATASTROPHIC BUG: OVERSUBSCRIPTION DISASTER!
#pragma omp parallel num_threads(4)
{
    // Every single thread executes this while loop!
    while (node != NULL) {
        #pragma omp task
        process(node);
        node = node->next;
    }
}
```
If you run this code across 4 threads, **all 4 threads simultaneously traverse the linked list**, resulting in $4\times$ duplicate tasks, severe data races on `node = node->next`, and memory corruption!

### The Correct OpenMP Pattern: Use `#pragma omp single`

As highlighted in your lecture notes (`3d773f8d.pdf`, Page 24):

```c
// THE CORRECT SINGLE PRODUCER PATTERN:
#pragma omp parallel num_threads(4)
{
    #pragma omp single
    {
        // EXACTLY ONE THREAD traverses the list and generates task objects:
        struct Node *curr = head;
        while (curr != NULL) {
            #pragma omp task firstprivate(curr)
            {
                // Executed ASYNCHRONOUSLY by ANY available thread in the team!
                process(curr);
            }
            curr = curr->next;
        }
    } // Implicit barrier here ensures all tasks finish before exiting!
}
```

```
+-------------------------------------------------------------------------------+
|                    WHY THE SINGLE PRODUCER PATTERN WORKS                      |
+-------------------------------------------------------------------------------+
| 1. The `#pragma omp single` ensures that ONLY ONE THREAD (the producer)       |
|    traverses the linked list, reads pointers, and packages task objects.      |
|                                                                               |
| 2. While the single producer thread is busy creating tasks, the OTHER THREE   |
|    THREADS are waiting at the implicit barrier of `#pragma omp single`.       |
|                                                                               |
| 3. In OpenMP, waiting threads at a barrier DO NOT SLEEP! They actively steal  |
|    tasks from the Task Pool and execute them in parallel!                     |
|                                                                               |
| 4. At the closing brace of `#pragma omp single`, execution cannot proceed     |
|    until every single generated task in the pool has completely finished.    |
+-------------------------------------------------------------------------------+
```

---

## 3. Data Scoping in Tasks: The `firstprivate` Rule

Data scoping behaves differently inside tasks compared to standard loops:

$$\boxed{\textbf{The Default Task Scoping Rule: } \text{Variables declared outside a task construct are}}$$
$$\boxed{\textbf{firstprivate by default} \text{ if they were local variables in the enclosing scope!}}$$

### Why Are Task Variables `firstprivate` by Default?
Consider what happens during linked list traversal:
```c
struct Node *curr = head;
while (curr != NULL) {
    #pragma omp task // Captures 'curr'
    {
        process(curr);
    }
    curr = curr->next; // The producer thread immediately advances 'curr'!
}
```
If `curr` were shared:
- By the time a worker thread wakes up to execute `process(curr)`, the producer thread will have already moved `curr` three nodes ahead!
- By capturing `curr` as **`firstprivate`**, OpenMP takes a **snapshot of the pointer value** at the exact instant the task is created, guaranteeing that each task processes its intended node.

---

## 4. Task Synchronization: `#pragma omp taskwait`

What happens when a task creates child tasks and needs their results before it can proceed?

Consider a recursive divide-and-conquer algorithm like the Fibonacci sequence ($F_n = F_{n-1} + F_{n-2}$):

$$\boxed{\textbf{\#pragma omp taskwait: } \text{Blocks the calling task until all \textbf{direct child tasks}}}$$
$$\boxed{\text{generated by it have completely finished execution.}}$$

```c
int fib(int n) {
    if (n < 2) return n;

    int x, y;

    // Fork Child Task 1
    #pragma omp task shared(x)
    x = fib(n - 1);

    // Fork Child Task 2
    #pragma omp task shared(y)
    y = fib(n - 2);

    // MANDATORY SYNCHRONIZATION: Must wait for both child tasks to finish!
    #pragma omp taskwait

    return x + y;
}
```

```
========================================================================================
                   THE TASKWAIT DIVIDE-AND-CONQUER TREE
========================================================================================

                                  fib(4)
                               /          \
                       fib(3)              fib(2)
                      /      \             /    \
                  fib(2)    fib(1)     fib(1)  fib(0)
                  /    \
              fib(1)  fib(0)
                  |
              #pragma omp taskwait (Children sync before parent adds results!)
========================================================================================
```

---

## 5. Task Loops: `#pragma omp taskloop`

In OpenMP 4.5, the Architecture Review Board introduced a powerful hybrid directive: **`#pragma omp taskloop`**.

As presented in your lecture notes (`3d773f8d.pdf`, Pages 23–24):

```c
#pragma omp parallel
{
    #pragma omp single
    {
        #pragma omp taskloop
        for (int i = 0; i < N; i++) {
            compute(i);
        }
    }
}
```

---

### `#pragma omp for` vs `#pragma omp taskloop`: The Architectural Difference

```
+-----------------------------------------------------------------------------------+
|                     #PRAGMA OMP FOR vs #PRAGMA OMP TASKLOOP                       |
+-----------------------------------------------------------------------------------+
| Feature Dimension     | `#pragma omp for`            | `#pragma omp taskloop`             |
| :-------------------- | :--------------------------- | :--------------------------------- |
| **Execution Model**   | **Direct Loop Partitioning:**| **Task Object Factory:**           |
|                       | Divides iterations directly  | Decomposes loop into discrete task |
|                       | among existing threads       | objects deposited into a task pool  |
| **Scheduling**        | Static / Dynamic / Guided    | OpenMP runtime task scheduler      |
|                       | worksharing schedules         | with dynamic task-stealing          |
| **Asynchronous Power**| Threads tied to loop barrier | Tasks can be interleaved with other|
|                       | until all iterations finish  | non-loop background tasks           |
| **Ideal Use Case**    | Standard numerical matrices  | Loops with massive iterations on    |
|                       | and dense linear algebra      | heterogeneous or imbalanced systems |
+-----------------------------------------------------------------------------------+
```

---

### Controlling Task Granularity: `grainsize` vs `num_tasks`

Creating a task object requires allocating internal descriptor memory in the runtime. If you spawn $100\text{ million}$ individual tasks for single-instruction iterations, **task creation overhead will overwhelm your CPU**.

OpenMP provides two clauses to regulate task granularity:

```
+-------------------------------------------------------------------------------+
|                       TASKLOOP GRANULARITY CONTROL                            |
+-------------------------------------------------------------------------------+
| 1. `grainsize(G)`:                                                            |
|    - Bundles at least `G` loop iterations into each generated task object.    |
|    - Example: For $1,000$ iterations with `grainsize(100)`, OpenMP creates    |
|      approximately $10$ tasks, each executing $100$ iterations.               |
|                                                                               |
| 2. `num_tasks(K)`:                                                            |
|    - Fixes the total number of task objects created to exactly `K`.           |
|    - Example: For $1,000$ iterations with `num_tasks(4)`, OpenMP creates      |
|      exactly $4$ tasks of $250$ iterations each, regardless of thread count.  |
+-------------------------------------------------------------------------------+
```

```c
// Example: Process 10,000,000 elements with optimal 500-iteration granularity
#pragma omp parallel
{
    #pragma omp single
    {
        #pragma omp taskloop grainsize(500)
        for (int i = 0; i < 10000000; i++) {
            heavy_calculation(i);
        }
    }
}
```

---

## 6. Complete Compilable C Program: Parallel Fibonacci & Linked List Processing

Here is the complete C program demonstrating both recursive task parallelism (`#pragma omp task` + `#pragma omp taskwait`) and taskloop processing:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_task_parallelism.c
 * DESCRIPTION: Comprehensive demonstration of OpenMP Tasking:
 *              - Parallel recursive Fibonacci with #pragma omp task & taskwait
 *              - Single producer pattern with task pool execution
 *              - #pragma omp taskloop with grainsize control
 * COMPILATION: gcc -Wall -O2 openmp_task_parallelism.c -o omp_tasks -fopenmp
 * USAGE:       ./omp_tasks
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define FIB_NUMBER 20
#define NUM_THREADS 4

// ----------------------------------------------------------------------------
// DEMO 1: RECURSIVE FIBONACCI USING OPENMP TASKS
// ----------------------------------------------------------------------------
int fibonacci_task(int n) {
    if (n < 2) return n;

    int fn1, fn2;

    // Use a cutoff threshold (n <= 12) to avoid creating micro-tasks
    if (n <= 12) {
        return fibonacci_task(n - 1) + fibonacci_task(n - 2);
    }

    // Fork Child Task 1
    #pragma omp task shared(fn1)
    fn1 = fibonacci_task(n - 1);

    // Fork Child Task 2
    #pragma omp task shared(fn2)
    fn2 = fibonacci_task(n - 2);

    // Wait for both recursive child tasks to complete before combining
    #pragma omp taskwait

    return fn1 + fn2;
}

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("            OPENMP TASK PARALLELISM ARCHITECTURE                \n");
    printf("   Worker Threads: %d\n", NUM_THREADS);
    printf("================================================================\n\n");

    // ------------------------------------------------------------------------
    // EXPERIMENT 1: RECURSIVE DIVIDE-AND-CONQUER TASKING
    // ------------------------------------------------------------------------
    int fib_result = 0;
    double start_time = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS)
    {
        // Single Producer Pattern: Exactly ONE thread initiates the task tree!
        #pragma omp single
        {
            printf("[Producer Thread %d] Generating root Fibonacci task...\n", 
                   omp_get_thread_num());
            fib_result = fibonacci_task(FIB_NUMBER);
        }
    }

    double elapsed_fib = omp_get_wtime() - start_time;
    printf("[Result] Fibonacci(%d) = %d (Computed in %.6f seconds)\n\n", 
           FIB_NUMBER, fib_result, elapsed_fib);

    // ------------------------------------------------------------------------
    // EXPERIMENT 2: TASKLOOP WITH GRAINSIZE CONTROL
    // ------------------------------------------------------------------------
    printf("--- EXPERIMENT 2: #pragma omp taskloop with grainsize(25) ---\n");
    int loop_iterations = 100;

    #pragma omp parallel num_threads(NUM_THREADS)
    {
        #pragma omp single
        {
            // Automatically chunks 100 iterations into 4 discrete tasks (25 each)
            #pragma omp taskloop grainsize(25)
            for (int i = 0; i < loop_iterations; i++) {
                if (i % 25 == 0) {
                    printf("  [Tasklet] Iteration %2d executed by Thread %d\n", 
                           i, omp_get_thread_num());
                }
            }
        }
    }

    printf("\n[Main] Task parallelism demonstrations completed successfully!\n");
    return 0;
}
```

---

## 7. University Examination Preparation: Solved Questions

### Question 1: Core Conceptual Architecture (10 Marks)
**Why was Task Parallelism introduced in OpenMP 3.0? Explain the Single Producer Pattern using `#pragma omp single` and `#pragma omp task`. What happens if `#pragma omp single` is omitted?**

#### Model Examination Answer:
1. **Why Task Parallelism Was Introduced:**  
   Prior to OpenMP 3.0, parallelism was largely restricted to Canonical For-Loops (`#pragma omp for`). This model failed on irregular, non-indexable algorithms such as pointer chasing in linked lists, recursive divide-and-conquer routines (QuickSort, tree traversals), and dynamic while-loops. Task parallelism was introduced to decouple parallel units of work from structured loops, allowing arbitrary functions and statements to execute asynchronously via an internal task pool.
2. **The Single Producer Pattern:**  
   To parallelize an irregular data structure (such as a linked list), a parallel team is created, but a `#pragma omp single` block is inserted. Inside `single`, exactly one thread traverses the data structure, packaging each element into a `#pragma omp task`. The remaining threads in the team wait at the implicit barrier of the `single` block; however, while waiting, they actively pull tasks from the Task Pool and execute them concurrently.
3. **Consequence of Omitting `#pragma omp single`:**  
   If `#pragma omp single` is omitted, **all $N$ threads in the parallel team concurrently execute the task generation code**. For a list of 100 nodes running on 4 threads, 400 duplicate tasks will be generated. The pointer traversal `node = node->next` will experience severe race conditions, causing duplicate processing, missed elements, and segmentation faults.

---

### Question 2: Synchronization & Directives (5 Marks)
**Explain the role of `#pragma omp taskwait`. Contrast it with `#pragma omp barrier`.**

#### Model Examination Answer:
1. **`#pragma omp taskwait`:**  
   `taskwait` is a hierarchical task synchronization construct. It suspends the execution of the current parent task until **all direct child tasks generated by that parent task have completed**. It does **not** block other peer threads or affect ancestor tasks in the call tree.
2. **Contrast with `#pragma omp barrier`:**  
   - **`barrier` is Team-Wide:** A barrier synchronizes **all threads in the entire parallel team**, forcing all execution pipelines to halt at the rendezvous point.
   - **`taskwait` is Local to a Task:** `taskwait` operates only within a specific task hierarchy; other worker threads in the team continue executing independent tasks from the task pool without interruption.

---

### Question 3: Taskloop Granularity (5 Marks)
**Differentiate between the `grainsize` and `num_tasks` clauses in `#pragma omp taskloop`. Why is granularity control essential for performance?**

#### Model Examination Answer:
1. **Clause Distinctions:**
   - **`grainsize(G)`:** Specifies the *number of iterations* bundled into each task object. OpenMP creates tasks that contain at least $G$ iterations.
   - **`num_tasks(K)`:** Specifies the *total number of task objects* created for the entire loop. OpenMP divides the loop iteration space into exactly $K$ discrete tasks.
2. **Importance of Granularity Control:**  
   Creating and managing tasks incurs runtime memory allocation and queue synchronization overhead. If granularity is too fine (e.g., $1$ iteration per task for millions of iterations), task management overhead dwarfs the computational work, severely degrading performance. If granularity is too coarse, the system suffers load imbalance. Tuning `grainsize` ensures that each task represents enough computational work to justify runtime scheduling costs.

---

## 8. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.7 QUICK REVISION SHEET
================================================================================

1. WHAT IS A TASK?
   - An independent, asynchronous unit of work: Code + Data Snapshot + Control vars.
   - Ideal for irregular algorithms: linked lists, recursive trees, unbounded while-loops.

2. THE SINGLE PRODUCER PATTERN:
   - #pragma omp parallel { #pragma omp single { while(node) #pragma omp task ... } }
   - 1 thread creates tasks; other idle threads pull from Task Pool and execute!
   - Omitting single causes disaster: N threads duplicate task creation!

3. DATA SCOPING IN TASKS:
   - Variables are firstprivate by default (snapshots pointer values at creation).

4. TASK SYNCHRONIZATION (#pragma omp taskwait):
   - Blocks parent task until all DIRECT CHILD TASKS have finished.

5. TASK LOOPS (#pragma omp taskloop):
   - Decomposes loops into task objects managed by OpenMP task scheduler.
   - grainsize(G): Bundles at least G iterations per task.
   - num_tasks(K): Fixes total generated tasks to K.
================================================================================
```

# Data Scoping in OpenMP: Shared, Private, Masking & Stack Architecture

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 05 — Shared-Memory Parallel Programming with OpenMP
SUBTOPIC:               03 — Data Scoping, Memory Layout & Variable Masking
PREREQUISITES:          Subtopic 02.2 (Shared Memory UMA/NUMA), Subtopic 05.1 (Fork-Join Model)
SOURCE LECTURE NOTES:   3d773f8d.pdf (Pages 12–15)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Data Scoping Problem in Shared Memory

In a shared-memory system, all CPU cores share a single physical DRAM address space. When you parallelize a program using OpenMP, the most critical decision you will make for every variable is its **Data Sharing Attribute (Data Scope)**:

$$\boxed{\text{Is this variable \textbf{Shared} among all threads, or is it \textbf{Private} to each individual thread?}}$$

### The Two Concurrency Disasters of Incorrect Scoping

```
+-------------------------------------------------------------------------------+
|                       THE TWO SCOPING CATASTROPHES                            |
+-------------------------------------------------------------------------------+
| 1. UNINTENTIONAL SHARING (The Race Condition Trap):                           |
|    - Suppose a temporary calculation variable or loop counter `temp` is       |
|      accidentally left Shared.                                                |
|    - 16 threads concurrently overwrite `temp` while computing different rows.|
|    - Result: Catastrophic silent data corruption, lost updates, and random     |
|      numerical crashes.                                                       |
|                                                                               |
| 2. UNINTENTIONAL PRIVATIZATION (The Lost Update Trap):                        |
|    - Suppose a variable intended to accumulate global results `total_sum`     |
|      is marked Private.                                                       |
|    - Each thread updates its own isolated private copy.                       |
|    - At the end of the parallel region, the private copies are destroyed!     |
|    - Result: The global `total_sum` remains 0; all work is lost!              |
+-------------------------------------------------------------------------------+
```

---

## 2. Physical Memory Architecture: Shared Process Stack vs Thread Stacks

To understand how OpenMP implements scoping, you must understand how operating system memory is physically structured across threads.

As transcribed directly from your lecture notes (`3d773f8d.pdf`, Page 13), an OpenMP process manages two distinct types of stacks:

```
========================================================================================
         PHYSICAL STACK MEMORY LAYOUT: SHARED STACK vs THREAD-PRIVATE STACKS
========================================================================================

     SHARED PROCESS STACK (Master Thread)             THREAD-PRIVATE STACK (Worker Thread)
+--------------------------------------------+       +------------------------------------+
| Stack Pointer -> [ <free stack space>   ]  |       | Stack Pointer -> [ <free space>  ] |
|                  [                      ]  |       |                  [               ] |
|                  [ y (outer shared)     ]  |       |                  [ y (private)   ] |
|                  [ x (outer shared)     ]  |       |                  [ i (local)     ] |
|                  [ a (outer shared)     ]  |       +------------------------------------+
+--------------------------------------------+             ^
     ^                                                     |
     | Memory Addresses Grow Downward                      | Carved out of virtual memory
     | Accessible by Master & (via pointers) workers       | Exclusively owned by Worker Thread
========================================================================================
```

### 1. The Shared Process Stack (Master Thread Stack)
- The primary runtime stack allocated by the operating system when the executable process launches.
- Contains all variables declared in the outer sequential scopes (e.g., inside `main()` before a parallel block).
- Accessible by all threads in the team (unless specifically masked).

### 2. The Thread-Private Stacks
- When OpenMP forks a team of threads, the runtime library carves out an **independent, private stack for each worker thread** in virtual memory.
- Contains:
  1. Local automatic variables declared *inside* the parallel curly braces `{ ... }`.
  2. Variables explicitly placed in an OpenMP `private` or `firstprivate` clause.
  3. Local variables inside functions called by that thread.
- Memory isolated on Thread 1's stack is physically separated from Thread 2's stack.

---

### The Thread Stack Overflow Hazard & `OMP_STACKSIZE`

> [!CAUTION]
> **The Thread Stack Size Limit:**  
> While the main process stack typically has an $8\text{ MB}$ limit (configurable via `ulimit -s`), worker thread stacks are allocated with a much smaller default (often $2\text{ MB}$ or $4\text{ MB}$).  
> If a worker thread allocates a large private matrix on its stack:
> ```c
> #pragma omp parallel
> {
>     double local_matrix[1000][1000]; // 1,000,000 doubles x 8 bytes = 8 MB!
>     // CRASH! Segmentation fault: Stack Overflow on worker thread!
> }
> ```
> **The Fixes:**
> 1. Allocate large buffers on the dynamic heap using `malloc()` / `free()`.
> 2. Increase OpenMP thread stack size using the environment variable:
>    ```bash
>    export OMP_STACKSIZE=64M
>    ```

---

## 3. Variable Shadowing & Masking in C Lexical Scopes

Examine the code snippet from your lecture notes (`3d773f8d.pdf`, Page 13):

```c
void f() {
    int a;
    float x, y; // Outer shared variables
    
    #pragma omp parallel
    {
        int i;   // Local variable: Automatically private to each thread
        float y; // MASKING / SHADOWING: Local 'y' masks the outer shared 'y'!
        
        y = 5.0f; // Modifies THIS thread's private stack 'y'
        // Outer 'x' is unmasked: Accessed as SHARED
        // Outer 'a' is unmasked: Accessed as SHARED
    }
    // Execution exits parallel region: Local 'y' is destroyed.
    // Outer 'y' recovers its original value completely unmodified!
}
```

### The Concept of Variable Masking
When a variable name inside an OpenMP parallel block shadows an identical variable name in the outer scope, C language lexical scoping rules govern:
- The newly declared variable is allocated on the **thread-private stack**.
- The outer variable continues to exist in the master stack, but is **masked (hidden/inaccessible)** within that thread's execution block.
- Upon leaving the block, the private instance is popped off the thread stack, and the outer shared variable becomes visible again.

---

## 4. OpenMP Data Scoping Clauses: `shared`, `private`, `firstprivate`, `lastprivate`

OpenMP provides explicit clauses to configure the scoping behavior of variables declared in the outer scope:

```
+-----------------------------------------------------------------------------------+
|                        OPENMP DATA SCOPING CLAUSES AT A GLANCE                    |
+-----------------------------------------------------------------------------------+
| Clause               | Allocation Location | Initial Value Inside | Final Value Outside   |
| :------------------- | :------------------ | :------------------- | :-------------------- |
| **`shared(x)`**      | Shared Stack / DRAM | Existing outer value | Updates persist!      |
| **`private(x)`**     | Thread-Private Stack| **UNDEFINED!**       | Unchanged (discards   |
|                      |                     | (Contains garbage)   | private updates!)     |
| **`firstprivate(x)`**| Thread-Private Stack| **COPIED from outer**| Unchanged (discards   |
|                      |                     | value before fork    | private updates!)     |
| **`lastprivate(x)`** | Thread-Private Stack| **UNDEFINED!**       | **COPIED BACK** from  |
|                      |                     | (Contains garbage)   | sequentially last iter|
+-----------------------------------------------------------------------------------+
```

---

### 1. The `shared(list)` Clause

```c
#pragma omp parallel shared(data_array, count)
```
- Exactly **one physical instance** of each listed variable exists in memory.
- All threads read from and write to the exact same memory address.
- **Rules:** Read-only access by multiple threads is 100% thread-safe. However, if any thread writes to a shared variable, you **must use synchronization** (such as `#pragma omp critical` or `atomic`) to prevent race conditions.

---

### 2. The `private(list)` Clause & The "Undefined Initial Value" Trap

```c
int x = 42;

#pragma omp parallel private(x)
{
    // CATASTROPHIC TRAP: What is the value of x here?
    printf("Thread %d sees x = %d\n", omp_get_thread_num(), x); // PRINTS GARBAGE!
    x = omp_get_thread_num();
}

printf("Outside parallel region: x = %d\n", x); // PRINTS 42!
```

As demonstrated in your lecture notes (`3d773f8d.pdf`, Page 15 diagram):

```
========================================================================================
                      THE LIFECYCLE OF A `PRIVATE` VARIABLE
========================================================================================

Outer Shared Scope:  `x = 42` (Stored in Master Stack)
                     ==================== FORK ====================
                          Thread T0          Thread T1          Thread T2
                        +-------------+    +-------------+    +-------------+
                        |  x_0: ???   |    |  x_1: ???   |    |  x_2: ???   |
                        +-------------+    +-------------+    +-------------+
                               ^                  ^                  ^
                               +------------------+------------------+
                                  INITIAL VALUES ARE COMPLETELY UNDEFINED!
                                  (Do NOT inherit the outer value 42!)

                     [Outer shared variable `x = 42` persists but is INACCESSIBLE]

                        +-------------+    +-------------+    +-------------+
                        |  x_0 = 100  |    |  x_1 = 200  |    |  x_2 = 300  |
                        +-------------+    +-------------+    +-------------+
                     ==================== JOIN ====================
Outer Shared Scope:  `x = 42` (Thread-private copies destroyed; outer `x` unchanged!)
========================================================================================
```

$$\boxed{\textbf{The Two Rules of \texttt{private(x)}}:}$$
1. **Initial Value Undefined:** A private variable **does NOT inherit** the value of the outer variable. It is allocated as uninitialized space on the thread's stack. Reading it before assigning a value reads memory garbage!
2. **Updates Discarded on Exit:** Any modifications made to the private copy inside the parallel block vanish upon exiting. The original outer variable retains whatever value it held prior to the parallel construct.

---

### 3. The `firstprivate(list)` Clause: Solving the Initialization Problem

What if each thread needs its own private copy, but that copy **must start with the outer variable's initial value**?

```c
int threshold = 42;

#pragma omp parallel firstprivate(threshold)
{
    // Every thread's private copy begins with threshold = 42!
    printf("Thread %d starts with threshold = %d\n", omp_get_thread_num(), threshold);
    threshold += omp_get_thread_num(); // Safely modify without affecting others
}
// Outside: threshold remains 42
```
`firstprivate` allocates an independent copy on each thread's stack and performs an implicit copy-constructor / initialization from the original outer variable.

---

### 4. The `lastprivate(list)` Clause: Solving the Final Value Problem

What if a loop computes a value into a private variable, and you need that value **after the loop finishes**?

```c
int final_val = -1;

#pragma omp parallel for lastprivate(final_val)
for (int i = 0; i < 1000; i++) {
    final_val = i * 2;
}

printf("Final Value after loop: %d\n", final_val); // PRINTS 1998!
```
Under `lastprivate`, OpenMP tracks the **sequentially last iteration** of the loop (i.e., iteration $i = 999$). Whichever thread executed that last iteration has its private copy copied back into the original outer shared variable upon loop exit!

---

## 5. OpenMP Default Scoping Rules & Critical Exceptions

If you do not specify a scoping clause for a variable, what does OpenMP do?

As outlined in `3d773f8d.pdf` (Page 14), OpenMP enforces a **strict default rule with three critical exceptions**:

```
+-------------------------------------------------------------------------------+
|                       OPENMP DEFAULT SCOPING RULES                            |
+-------------------------------------------------------------------------------+
| THE GENERAL DEFAULT RULE:                                                     |
| - By default, all variables that were declared OUTSIDE the parallel construct |
|   are SHARED among all threads in the team.                                   |
| - Includes global variables, static variables, and heap allocations.          |
|                                                                               |
| THE THREE AUTOMATIC PRIVATIZATION EXCEPTIONS:                                 |
| 1. LOOP ITERATION COUNTERS:                                                   |
|    - In a worksharing loop (`#pragma omp for`), the loop index variable `i`   |
|      is AUTOMATICALLY made PRIVATE to each thread.                            |
|                                                                               |
| 2. LOCAL VARIABLES DECLARED INSIDE THE STRUCTURED BLOCK:                      |
|    - Any variable declared inside curly braces `{ ... }` within the parallel  |
|      region is AUTOMATICALLY allocated on that thread's private stack.        |
|                                                                               |
| 3. LOCAL AUTOMATIC VARIABLES IN CALLED FUNCTIONS:                             |
|    - If a thread calls a helper function `foo()` from inside a parallel       |
|      region, all local automatic variables inside `foo()` are allocated on    |
|      that thread's private stack (unless explicitly declared `static`).       |
+-------------------------------------------------------------------------------+
```

---

## 6. The Professional Best Practice: `default(none)`

Because OpenMP defaults to sharing everything, developers frequently create race conditions without realizing it. 

Consider this innocent-looking loop:

```c
// DANGEROUS CODE: ACCIDENTAL RACE CONDITION
int i, temp; // Declared outside

#pragma omp parallel for
for (i = 0; i < 100000; i++) {
    temp = heavy_math(i); // BUG: 'temp' is SHARED by default!
    array[i] = temp * 2;
}
```
Because `temp` was declared outside, OpenMP made it **shared**! All 16 threads fight to overwrite `temp`, completely corrupting `array`.

### The Remedy: Force Explicit Scoping with `default(none)`

```c
// IMMUNE TO SCOPING BUGS:
#pragma omp parallel for default(none) shared(array) private(temp)
for (int i = 0; i < 100000; i++) {
    temp = heavy_math(i); // 100% Safe: Compiler verified temp is private!
    array[i] = temp * 2;
}
```

$$\boxed{\textbf{The \texttt{default(none)} Mandate: } \text{Forces the compiler to generate a compilation error}}$$
$$\boxed{\text{if any variable used in the parallel region lacks an explicit scoping clause!}}$$

---

## 7. Complete Compilable C Program: Comprehensive Data Scoping Demonstration

Here is the complete C program demonstrating physical stack allocation, variable masking, `private`, `firstprivate`, and `lastprivate`:

```c
/**
 * ============================================================================
 * PROGRAM:     openmp_scoping_demo.c
 * DESCRIPTION: Comprehensive demonstration of OpenMP data scoping:
 *              - Variable masking / shadowing
 *              - Difference between shared, private, and firstprivate
 *              - Restoring shared values after parallel regions
 *              - lastprivate in loop work-sharing
 * COMPILATION: gcc -Wall -O2 openmp_scoping_demo.c -o omp_scoping -fopenmp
 * USAGE:       ./omp_scoping
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

int main(int argc, char *argv[]) {
    printf("================================================================\n");
    printf("            OPENMP DATA SCOPING & MEMORY ARCHITECTURE           \n");
    printf("================================================================\n\n");

    int shared_val = 100;
    int private_val = 42;
    int firstpriv_val = 777;

    // ------------------------------------------------------------------------
    // DEMO 1: Shared vs Private vs Firstprivate
    // ------------------------------------------------------------------------
    printf("--- DEMO 1: Scoping Attributes Behavior ---\n");
    printf("[Before Parallel] shared_val=%d, private_val=%d, firstpriv_val=%d\n\n",
           shared_val, private_val, firstpriv_val);

    #pragma omp parallel num_threads(3) \
                         default(none) \
                         shared(shared_val, stdout) \
                         private(private_val) \
                         firstprivate(firstpriv_val)
    {
        int tid = omp_get_thread_num();

        // 1. private_val: Undefined initial value! Must assign before reading.
        private_val = (tid + 1) * 10;

        // 2. firstpriv_val: Inherited 777 from outer scope!
        firstpriv_val += tid;

        printf("  [Thread %d] private_val=%d | firstpriv_val=%d | shared_val=%d\n",
               tid, private_val, firstpriv_val, shared_val);

        // Modify shared value safely by master only
        if (tid == 0) {
            shared_val = 999;
        }
    }

    printf("\n[After Parallel] (Notice what changed and what recovered!)\n");
    printf("  shared_val:    %d (MODIFIED: Shared variable was updated!)\n", shared_val);
    printf("  private_val:   %d (UNCHANGED: Private updates were discarded!)\n", private_val);
    printf("  firstpriv_val: %d (UNCHANGED: Firstprivate updates discarded!)\n\n", firstpriv_val);

    // ------------------------------------------------------------------------
    // DEMO 2: Lastprivate in Loop Worksharing
    // ------------------------------------------------------------------------
    printf("--- DEMO 2: Lastprivate Propagation ---\n");
    int last_computed_item = -1;

    #pragma omp parallel for num_threads(4) lastprivate(last_computed_item)
    for (int i = 0; i < 20; i++) {
        last_computed_item = i * 10;
    }

    printf("  last_computed_item after loop: %d\n", last_computed_item);
    printf("  (Copied back from sequentially last iteration i=19: 19 * 10 = 190!)\n");

    printf("\n================================================================\n");
    return 0;
}
```

---

## 8. University Examination Preparation: Solved Questions

### Question 1: Core Comparison (10 Marks)
**Compare `private`, `firstprivate`, and `lastprivate` clauses in OpenMP with respect to:**
1. **Memory allocation location**
2. **Initial value upon entering the parallel construct**
3. **Value of the original variable upon exiting the construct**
**Write a short code example illustrating when `firstprivate` is required instead of `private`.**

#### Model Examination Answer:
1. **Comparison Table:**

| Feature Dimension | `private(var)` | `firstprivate(var)` | `lastprivate(var)` |
| :--- | :--- | :--- | :--- |
| **Allocation Location** | Thread-Private Stack | Thread-Private Stack | Thread-Private Stack |
| **Initial Value Inside** | **Undefined** (Memory garbage) | **Copied from outer variable** before fork | **Undefined** (Memory garbage) |
| **Final Value Outside** | **Unchanged** (Retains pre-fork value) | **Unchanged** (Retains pre-fork value) | **Copied back from sequentially last iteration** |

2. **When `firstprivate` is Required:**  
   If a thread-private variable must perform computation based on an initial state computed in the sequential region (e.g., an array scaling multiplier or step size), `private` fails because it initializes the variable to garbage. `firstprivate` is mandatory to copy the base value:
   ```c
   double factor = compute_calibration_factor();
   #pragma omp parallel for firstprivate(factor)
   for (int i = 0; i < N; i++) {
       factor += adjustments[i]; // Starts with valid calibration value!
       results[i] = data[i] * factor;
   }
   ```

---

### Question 2: Memory Architecture (5 Marks)
**Draw and explain the physical stack layout of an OpenMP process executing on multiple threads. Explain why declaring large local arrays inside a parallel block can cause segmentation faults.**

#### Model Examination Answer:
1. **Memory Layout Explanation:**  
   An OpenMP process contains a single **Shared Process Stack** (managed by the Master Thread for global/outer variables) and multiple **Thread-Private Stacks** (carved out of virtual memory, one for each worker thread). Thread-private stack frames hold variables declared inside `{ ... }`, variables scoped as `private`/`firstprivate`, and function parameters.
2. **Segmentation Fault Cause:**  
   Operating systems allocate worker thread stacks with strict default limits (typically $2\text{ MB}$ to $4\text{ MB}$). If a developer declares a large static array (e.g., `double matrix[1000][1000]` $= 8\text{ MB}$) inside a parallel block, the allocation exceeds the thread's physical stack boundary, corrupting adjacent virtual memory pages and triggering a Segmentation Fault (`SIGSEGV`). Developers must allocate large buffers on the heap via `malloc` or increase stack size using `export OMP_STACKSIZE=64M`.

---

### Question 3: Code Bug Identification (5 Marks)
**Why does the OpenMP community recommend using `default(none)` on all parallel constructs? What bug does it prevent?**

#### Model Examination Answer:
- By default, OpenMP treats all variables declared outside a parallel region as **shared**.
- In complex numerical codes, programmers frequently declare temporary variables (such as temporary loop accumulators or intermediate mathematical terms) in the outer scope. If the developer forgets to mark them `private`, OpenMP silently shares them across all threads, causing devastating data races and non-deterministic numerical errors that are difficult to debug.
- The `default(none)` clause instructs the compiler to disable automatic scoping. The compiler will reject compilation with an error unless the programmer explicitly scopes **every single variable** (`shared`, `private`, `firstprivate`, or `reduction`), catching unintentional sharing bugs at compile time.

---

## 9. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 05.3 QUICK REVISION SHEET
================================================================================

1. STACK ARCHITECTURE:
   - Shared Stack: Process stack holding outer master variables.
   - Private Stacks: Dedicated stack per thread for local & private variables.
   - Stack limit remedy: export OMP_STACKSIZE=64M or use heap malloc().

2. SCOPING ATTRIBUTES MATRIX:
   - shared: 1 physical address. Updates persist. Beware of data races!
   - private: Thread-private copy. Initial value UNDEFINED (garbage!). Updates discarded on exit.
   - firstprivate: Thread-private copy INITIALIZED from outer variable value.
   - lastprivate: Private copy from SEQUENTIALLY LAST iteration copied back to outer variable!

3. DEFAULT SCOPING RULES:
   - Everything outside is SHARED by default.
   - EXCEPTIONS: Loop index counters, variables declared inside { ... }, 
     and local variables in called functions are AUTOMATICALLY PRIVATE.
   - static variables are ALWAYS SHARED.

4. GOLDEN COMPILER CLAUSE:
   - ALWAYS use default(none) on parallel regions to catch scoping bugs at compile time!
================================================================================
```

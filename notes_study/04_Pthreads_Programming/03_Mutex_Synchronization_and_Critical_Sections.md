# Mutex Synchronization & Critical Section Protection

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 04 — Shared-Memory Parallel Programming with Pthreads
SUBTOPIC:               03 — Mutex Synchronization & Critical Sections
PREREQUISITES:          Subtopic 04.1 (Pthreads Lifecycle), Subtopic 02.2 (Shared Memory)
SOURCE LECTURE NOTES:   Pthreada_Aagam_Notes.pdf (Pages 6–9), IMG_1333.pdf (Pages 6–9)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. The Concurrency Nightmare: Race Conditions Under the Microscope

In Subtopic 04.1, we discovered that all threads within a process share the same **Virtual Address Space**, including global variables and the heap.

While shared memory makes data exchange fast (requiring zero operating system socket calls), it introduces the single most dangerous bug in parallel programming: **The Race Condition**.

### What is a Race Condition?

$$\boxed{\text{A \textbf{Race Condition} occurs when two or more concurrent threads access a shared memory location}}$$
$$\boxed{\text{simultaneously, at least one access is a \textbf{write}, and the threads do not use any synchronization.}}$$

When a race condition occurs, the final state of memory depends unpredictably on the arbitrary microsecond-level timing of the operating system thread scheduler.

---

### The Illusion of `counter++`

Consider this innocent-looking line of C code executed concurrently by 4 threads:

```c
counter++; // Shared global integer
```

To a novice programmer, `counter++` looks like a single atomic step. But to a physical CPU core, **it is not atomic**. The compiler expands `counter++` into three distinct assembly instructions:

```assembly
; Disassembled x86-64 Assembly for: counter++
MOV  EAX, [counter]   ; Step 1: LOAD  - Read value from RAM/Cache into CPU Register EAX
ADD  EAX, 1           ; Step 2: MODIFY - Increment value inside CPU Register EAX
MOV  [counter], EAX   ; Step 3: STORE - Write value from CPU Register back to RAM/Cache
```

Now observe what happens when **Thread A** and **Thread B** both attempt to increment `counter` (initially `0`) concurrently on two different CPU cores:

```
========================================================================================
             EXECUTION TIMELINE: THE TRAGEDY OF THE LOST UPDATE
========================================================================================

Time   Thread A (Core 0)              Thread B (Core 1)              DRAM `counter` Value
----------------------------------------------------------------------------------------
T1     LOAD EAX from counter (EAX=0)                                        0
T2                                    LOAD EBX from counter (EBX=0)         0
T3     ADD EAX, 1            (EAX=1)                                        0
T4                                    ADD EBX, 1            (EBX=1)         0
T5     STORE EAX to counter  (Write 1)                                      1
T6                                    STORE EBX to counter  (Write 1)       1  <-- CORRUPTED!
========================================================================================
```

### The Diagnostic Verdict
- Both Thread A and Thread B executed an increment.
- You expected `counter` to equal **2**.
- Because the read-modify-write sequences were interleaved in hardware, **Thread B overwrote Thread A's result**.
- One increment was permanently lost! If 4 threads each increment `counter` 100,000 times, instead of reaching 400,000, the final tally might be 163,421.

---

## 2. The Critical Section & The Principle of Mutual Exclusion

To prevent race conditions, computer scientists define the concept of a **Critical Section**:

$$\boxed{\textbf{Critical Section: } \text{A section of code that accesses shared mutable resources}}$$
$$\boxed{\text{and must not be executed by more than one thread concurrently.}}$$

```
+-------------------------------------------------------------------------------+
|                 THE THREE CLASSICAL CRITICAL SECTION CRITERIA                 |
+-------------------------------------------------------------------------------+
| 1. MUTUAL EXCLUSION (Mandatory):                                              |
|    - If thread $T_1$ is executing inside the critical section, no other       |
|      thread can enter that critical section.                                  |
|                                                                               |
| 2. PROGRESS:                                                                  |
|    - If no thread is inside the critical section and some threads want to     |
|      enter, only those threads competing for entry participate in the decision|
|      and the selection cannot be postponed indefinitely.                      |
|                                                                               |
| 3. BOUNDED WAITING:                                                           |
|    - There must exist a bound on the number of times other threads can enter  |
|      the critical section after a thread has requested entry, preventing      |
|      starvation.                                                              |
+-------------------------------------------------------------------------------+
```

---

## 3. The Mutex: Architecture & Working Principle

The primary software tool to enforce mutual exclusion in POSIX threads is the **Mutex** (short for **Mut**ual **Ex**clusion lock).

### The Hotel Bathroom Analogy

Think of a critical section as a **single-occupancy hotel bathroom**:
1. Outside the bathroom hangs a single brass key (the **Mutex**).
2. When Thread A needs to enter, it takes the key and locks the door behind it (`pthread_mutex_lock`).
3. While Thread A is inside, Thread B and Thread C arrive. Seeing the door locked, they cannot enter.
4. Instead of burning CPU cycles running in circles, Thread B and Thread C sit down on chairs outside and **fall asleep** (the OS puts them into a wait queue).
5. When Thread A finishes, it unlocks the door and hangs the key back on the peg (`pthread_mutex_unlock`).
6. The operating system kernel wakes up Thread B, which grabs the key and enters.

```
                          MUTEX LOCK STATE MACHINE
                          
                           pthread_mutex_init()
                                    |
                                    v
                           +-----------------+
                           |   UNLOCKED      |<----------------+
                           |  (Key on Peg)   |                 |
                           +--------+--------+                 |
                                    |                          |
                     Thread Calls   |                          | Thread Calls
              pthread_mutex_lock()  |                          | pthread_mutex_unlock()
                                    v                          |
                           +-----------------+                 |
                           |    LOCKED       |                 |
                           | (Thread Inside) +-----------------+
                           +--------+--------+
                                    |
                       Another Thread Calls
                       pthread_mutex_lock()
                                    v
                           +-----------------+
                           | WAITING THREADS |
                           |  (Asleep in OS  |
                           |   Wait Queue)   |
                           +-----------------+
```

---

## 4. The POSIX `pthread_mutex_t` API

POSIX threads provides the opaque data type `pthread_mutex_t` defined in `<pthread.h>`.

### 1. Mutex Initialization

A mutex can be initialized in one of two ways:

#### Method A: Static Initialization (Compile-Time)
For global or static mutexes using default attributes, POSIX provides the macro `PTHREAD_MUTEX_INITIALIZER`:
```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```
*(No explicit destruction is required for statically initialized mutexes).*

#### Method B: Dynamic Initialization (Runtime)
For dynamically allocated mutexes (e.g., inside heap structs or local variables) or when custom mutex attributes are required:
```c
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
```
Passing `NULL` for `attr` selects default mutex properties.

---

### 2. Mutex Acquisition (`pthread_mutex_lock`)

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
```
- **If the mutex is currently unlocked:** The calling thread instantly acquires ownership of the lock and enters the critical section.
- **If the mutex is currently locked by another thread:** The calling thread is **suspended (blocked)** by the OS kernel. The kernel puts the thread to sleep in an off-CPU wait queue, consuming **zero CPU cycles** while waiting.

---

### 3. Non-Blocking Acquisition (`pthread_mutex_trylock`)

```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```
Sometimes a thread cannot afford to block and sleep if a lock is unavailable:
- **If the mutex is available:** Locks it and returns `0`.
- **If the mutex is already locked:** Returns immediately with the error code **`EBUSY`** without blocking.
- **Use Cases:** Deadlock avoidance algorithms, polling loops, or speculative execution where a worker performs alternate work if a queue is locked.

---

### 4. Mutex Release (`pthread_mutex_unlock`)

```c
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
- Releases ownership of the lock.
- If other threads are sleeping in the kernel wait queue for this mutex, the OS scheduler wakes up one thread, which transitions to the ready state and acquires the lock.
- **Rule:** A thread must **never unlock a mutex that it does not own**, and must **never unlock an already unlocked mutex** (undefined behavior in default configurations).

---

### 5. Mutex Teardown (`pthread_mutex_destroy`)

```c
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```
- Destroys a dynamically initialized mutex, releasing any underlying kernel resources.
- It is illegal to destroy a mutex that is currently locked or being waited on by other threads.

---

## 5. Mutex Types & Error-Checking Configurations

By default, standard Pthreads mutexes are optimized for pure raw speed, which means they do not perform safety checks. POSIX allows configuring four distinct mutex types via `pthread_mutexattr_settype()`:

```
+-----------------------------------------------------------------------------------+
|                           POSIX MUTEX TYPES COMPARISON                            |
+-----------------------------------------------------------------------------------+
| Mutex Type Constant           | Relocking by Owner  | Unlock by Non-Owner| Performance    |
| :---------------------------- | :------------------ | :----------------- | :------------- |
| **`PTHREAD_MUTEX_NORMAL`**    | **DEADLOCK!**       | Undefined Behavior | Fastest        |
| (Default)                     |                     |                    |                |
| **`PTHREAD_MUTEX_ERRORCHECK`**| Returns `EDEADLK`    | Returns `EPERM`     | Safe / Debug   |
| **`PTHREAD_MUTEX_RECURSIVE`** | Allowed (Increments | Returns `EPERM`     | Medium         |
|                               | internal lock count) |                    | (Reentrant)    |
+-----------------------------------------------------------------------------------+
```

- **Recursive Mutexes:** Essential in recursive functions or nested call stacks where a function holding a lock calls another helper function that also acquires the same lock. The lock maintains an internal counter and is only fully released when an equal number of `unlock()` calls have executed.

---

## 6. Complete Compilable C Program: Multithreaded Counter with Mutex Protection

Here is the complete, compilable multithreaded synchronization program directly transcribed and enhanced from your lecture notes (`Pthreada_Aagam_Notes.pdf`, Pages 8–9 and `IMG_1333.pdf`, Pages 7–8):

```c
/**
 * ============================================================================
 * PROGRAM:     pthreads_mutex_counter.c
 * DESCRIPTION: Spawns NUM_THREADS concurrent threads that each increment a
 *              shared counter variable inside a critical section protected
 *              by a POSIX mutex (pthread_mutex_t).
 * COMPILATION: gcc -Wall -O2 pthreads_mutex_counter.c -o pthreads_mutex -pthread
 * USAGE:       ./pthreads_mutex
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 4
#define ITERATIONS_PER_THREAD 20

// 1. Shared Global Resource (Vulnerable to Race Conditions!)
int counter = 0;

// 2. Mutex Lock Declaration
pthread_mutex_t lock;

// Worker routine executed by each thread
void* increment_worker(void *arg) {
    long thread_id = (long)arg;

    for (int i = 0; i < ITERATIONS_PER_THREAD; i++) {
        // ----------------- BEGIN CRITICAL SECTION -----------------
        pthread_mutex_lock(&lock);

        // Mutex ensures only ONE thread executes this block at a time
        counter++;
        printf("[Thread %ld] incremented counter to: %d\n", thread_id, counter);

        pthread_mutex_unlock(&lock);
        // ------------------ END CRITICAL SECTION ------------------
    }

    pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
    pthread_t threads[NUM_THREADS];
    int rc;

    printf("[Main] Initializing POSIX Mutex...\n");
    // 3. Initialize Mutex with default attributes (NULL)
    rc = pthread_mutex_init(&lock, NULL);
    if (rc != 0) {
        fprintf(stderr, "Failed to initialize mutex: error %d\n", rc);
        exit(EXIT_FAILURE);
    }

    printf("[Main] Spawning %d worker threads...\n", NUM_THREADS);
    // 4. Thread Creation Loop
    for (long i = 0; i < NUM_THREADS; i++) {
        // Notice: Casting integer directly to (void*) for demonstration
        rc = pthread_create(&threads[i], NULL, increment_worker, (void*)i);
        if (rc != 0) {
            fprintf(stderr, "Failed to create thread %ld: error %d\n", i, rc);
            exit(EXIT_FAILURE);
        }
    }

    // 5. Join Loop: Wait for all threads to complete
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // 6. Mutex Teardown: Destroy mutex and release kernel resources
    pthread_mutex_destroy(&lock);

    // 7. Verify Results
    int expected_value = NUM_THREADS * ITERATIONS_PER_THREAD;
    printf("\n=======================================================\n");
    printf("FINAL COUNTER VALUE:    %d\n", counter);
    printf("EXPECTED COUNTER VALUE: %d\n", expected_value);
    if (counter == expected_value) {
        printf("VERDICT: SUCCESS! Race condition eliminated by Mutex.\n");
    } else {
        printf("VERDICT: FAILED! Data corruption occurred.\n");
    }
    printf("=======================================================\n");

    return 0;
}
```

---

## 7. Argument Passing: Why `void*` and How to Pass Multiple Arguments?

As highlighted in your lecture notes (`IMG_1333.pdf`, Page 7), professors frequently ask two classic C systems programming questions:

### Question A: Why do thread functions use `void *` return and `void *` parameters?
**Answer:**  
To achieve **maximum polymorphic flexibility**. In the C language, there is no generic template system. A `void *` is a generic pointer type that can hold the memory address of **any data structure**—an `int`, a `double`, an array, a file pointer, or a custom user-defined `struct`. By defining `void *(*start_routine)(void *)`, POSIX allows programmers to pass arbitrary data into threads without requiring hundreds of specialized overloaded creation functions.

---

### Question B: How do you pass multiple arguments into a thread function (like `printf` does)?
**Answer:**  
Because `pthread_create()` accepts only **a single `void *arg`**, you cannot pass multiple independent arguments as separate parameters. 

To pass multiple values, you must **package them into a custom `struct`** and pass the pointer to that struct:

```c
// Step 1: Define a structure encapsulating all required arguments
typedef struct {
    int thread_id;
    int start_index;
    int end_index;
    double *input_matrix;
    double *output_vector;
} ThreadWorkPacket;

// Step 2: In the worker function, cast and unpack
void* process_chunk(void *arg) {
    ThreadWorkPacket *packet = (ThreadWorkPacket*)arg;
    
    printf("Thread %d processing indices %d to %d\n", 
           packet->thread_id, packet->start_index, packet->end_index);
           
    // Perform computation...
    free(packet); // Free dynamically allocated struct memory
    return NULL;
}

// Step 3: In the caller, allocate, populate, and pass
ThreadWorkPacket *pkt = (ThreadWorkPacket*)malloc(sizeof(ThreadWorkPacket));
pkt->thread_id = 0;
pkt->start_index = 0;
pkt->end_index = 250000;
pkt->input_matrix = matrix;
pkt->output_vector = vector;

pthread_create(&tid, NULL, process_chunk, (void*)pkt);
```

---

## 8. Lock Granularity & Amdahl's Law Collapse

While mutexes guarantee correctness, **misusing mutexes can completely destroy parallel performance**.

### Coarse-Grained vs Fine-Grained Locking

```
+-------------------------------------------------------------------------------+
|                       LOCK GRANULARITY TRADEOFF                               |
+-------------------------------------------------------------------------------+
| 1. COARSE-GRAINED LOCKING (One Giant Lock):                                   |
|    - A single mutex protects an entire database, hash table, or complex array.|
|    - Advantage: Trivial to write, zero risk of deadlocks.                     |
|    - Catastrophe: All threads serialize! Even if threads access completely    |
|      different records, they queue up behind the single lock.                 |
|    - Mathematical Impact: Turns parallel fraction $p \to 0$, driving speedup  |
|      straight into Amdahl's wall ($S(N) \to 1$).                              |
|                                                                               |
| 2. FINE-GRAINED LOCKING (Many Small Locks):                                   |
|    - A separate mutex protects each individual bucket in a hash table or      |
|      each node in a linked list.                                              |
|    - Advantage: Massive concurrency; threads accessing distinct buckets       |
|      execute in true simultaneous parallel hardware.                          |
|    - Danger: High memory overhead (thousands of mutexes) and severe risk of   |
|      **Deadlocks** if locks are acquired out of order!                        |
+-------------------------------------------------------------------------------+
```

### The "Mutex Inside vs Outside Loop" Dilemma

Examine the placement of the mutex in our increment program:

```c
// OPTION A: Fine-Grained Locking (Inside Loop)
for (int i = 0; i < 1000000; i++) {
    pthread_mutex_lock(&lock);
    counter++;
    pthread_mutex_unlock(&lock);
}

// OPTION B: Coarse-Grained Locking (Outside Loop)
pthread_mutex_lock(&lock);
for (int i = 0; i < 1000000; i++) {
    counter++;
}
pthread_mutex_unlock(&lock);

// OPTION C: Thread-Private Accumulation (Best Parallel Design!)
int local_sum = 0;
for (int i = 0; i < 1000000; i++) {
    local_sum++; // Purely private, 0 locks required!
}
pthread_mutex_lock(&lock);
counter += local_sum; // Exactly 1 lock acquisition per thread!
pthread_mutex_unlock(&lock);
```

- **Option A** executes 1,000,000 lock and unlock operations. On Linux, 1,000,000 mutex cycles will make the multithreaded program run **$20\times$ slower than a single-threaded program** due to lock contention overhead!
- **Option C** computes entirely in registers and private cache, acquiring the lock **only once** at the very end to merge results. This is the hallmark of professional parallel design.

---

## 9. University Examination Preparation: Solved Questions

### Question 1: Core Concept (5 Marks)
**Define a Race Condition and a Critical Section. Show with an assembly code trace how two concurrent threads incrementing a shared integer can result in a lost update.**

#### Model Examination Answer:
1. **Definitions:**
   - **Race Condition:** A flaw in concurrent software where the outcome depends on the non-deterministic scheduling order of threads executing conflicting reads and writes to shared memory.
   - **Critical Section:** A block of code accessing shared mutable state that must be protected by mutual exclusion so that at most one thread executes it at any instant.
2. **Assembly Disassembly Trace:**
   The C statement `x++` maps to three assembly instructions:
   ```assembly
   MOV EAX, [x]  ; Step 1: Read x from memory into register
   ADD EAX, 1    ; Step 2: Increment register value
   MOV [x], EAX  ; Step 3: Write register back to memory
   ```
3. **Lost Update Trace Table:**

| Step | Thread 1 (Core 0) | Thread 2 (Core 1) | Shared Memory `x` |
| :---: | :--- | :--- | :---: |
| 1 | `MOV EAX, [x]` (reads 5) | — | 5 |
| 2 | — | `MOV EBX, [x]` (reads 5) | 5 |
| 3 | `ADD EAX, 1` (EAX = 6) | — | 5 |
| 4 | — | `ADD EBX, 1` (EBX = 6) | 5 |
| 5 | `MOV [x], EAX` (stores 6) | — | 6 |
| 6 | — | `MOV [x], EBX` (stores 6) | 6 |

Despite two increments, `x` increased from 5 to 6 instead of 7. Thread 2's store overwrote Thread 1's work, causing a lost update.

---

### Question 2: API & Architecture (5 Marks)
**Explain the difference between `pthread_mutex_lock()` and `pthread_mutex_trylock()`. When would a developer choose `trylock` over `lock`?**

#### Model Examination Answer:
1. **`pthread_mutex_lock()` (Blocking):**  
   If the mutex is held by another thread, the calling thread is placed into a sleep state by the OS kernel and descheduled from the CPU. The thread waits passively in a kernel queue until the lock owner calls `pthread_mutex_unlock()`.
2. **`pthread_mutex_trylock()` (Non-Blocking):**  
   Attempts to acquire the lock immediately. If the lock is free, it locks it and returns `0`. If the lock is held, it does not block; it returns immediately with error code `EBUSY`.
3. **When to Choose `trylock`:**  
   - **Deadlock Avoidance (Resource Hierarchy Violations):** If a thread holds Lock A and needs Lock B, but cannot risk blocking because another thread might hold Lock B and request Lock A, it can attempt `trylock(B)`. If it returns `EBUSY`, it can release Lock A, back off, and retry later.
   - **Speculative Task Processing:** A worker thread can probe a queue's lock. If busy, instead of sleeping, it moves on to execute secondary non-blocking computational tasks.

---

### Question 3: Code Design (10 Marks)
**Write a complete C program using POSIX threads where 5 worker threads concurrently update a shared bank balance. Use a mutex to ensure thread safety. Demonstrate proper mutex initialization, acquisition, release, and teardown.**

#### Model Examination Answer:
*(Provide the compilable C code structure detailed in Section 6, demonstrating `#include <pthread.h>`, `pthread_mutex_t`, `pthread_mutex_init`, `pthread_mutex_lock`, `pthread_mutex_unlock`, and `pthread_mutex_destroy`).*

---

## 10. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 04.3 QUICK REVISION SHEET
================================================================================

1. RACE CONDITION ROOT CAUSE:
   - x++ is NOT atomic! It expands to: LOAD -> MODIFY -> STORE.
   - Interleaved memory access leads to lost updates and silent data corruption.

2. CRITICAL SECTION REQUIREMENTS:
   - Mutual Exclusion: At most 1 thread inside.
   - Progress: Decisions to enter cannot be blocked by non-competing threads.
   - Bounded Waiting: No thread starves indefinitely.

3. MUTEX API CHEAT SHEET:
   - Static Init: pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
   - Dynamic Init: pthread_mutex_init(&lock, NULL);
   - Blocking Lock: pthread_mutex_lock(&lock);
   - Non-Blocking: pthread_mutex_trylock(&lock); (returns EBUSY if locked)
   - Unlock: pthread_mutex_unlock(&lock);
   - Destroy: pthread_mutex_destroy(&lock);

4. ARGUMENT PASSING RULES:
   - Why void*? Generic pointer providing polymorphism in C.
   - Passing multiple arguments? Package them into a custom struct and pass 
     a pointer to that struct.

5. PERFORMANCE RULE:
   - Minimize lock contention! Never place locks inside massive computational loops.
   - Accumulate intermediate sums in private registers; acquire mutex once at the end.
================================================================================
```

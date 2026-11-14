# POSIX Semaphores & Read-Write Locks in Pthreads

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 04 — Shared-Memory Parallel Programming with Pthreads
SUBTOPIC:               05 — POSIX Semaphores & Read-Write Locks
PREREQUISITES:          Subtopic 04.1 (Pthreads Lifecycle), Subtopic 04.3 (Mutexes)
SOURCE LECTURE NOTES:   Pthreada_Aagam_Notes.pdf (Pages 13–16), IMG_1333.pdf (Pages 10, 13–16)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: Expanding the Synchronization Toolkit

In the previous subtopics, we mastered **Mutexes** (for exclusive 1-at-a-time critical sections) and **Condition Variables** (for sleeping until a boolean state condition changes).

However, real-world systems encounter concurrency scenarios that mutexes cannot solve cleanly or efficiently:
1. **Resource Pool Management:** What if you have a pool of **5 database connections** or **3 GPU accelerators** that up to 5 or 3 threads can use simultaneously? A mutex only allows 1 thread.
2. **Asynchronous Cross-Thread Signaling:** What if Thread A finishes computing a task and needs to wake up Thread B, without Thread A ever needing to lock a resource itself?
3. **Read-Heavy Shared Databases:** What if an application performs **99% read operations** (inspecting user profiles) and only **1% write operations** (updating an address)? Forcing every reader to acquire an exclusive mutex destroys parallelism.

To solve these problems, POSIX provides two powerful synchronization primitives:
- **POSIX Semaphores (`<semaphore.h>`)**
- **Read-Write Locks (`pthread_rwlock_t`)**

---

## 2. POSIX Semaphores: Architecture & Working Principle

Invented by Dutch computer scientist Edsger Dijkstra in 1965, a **Semaphore** is an integer variable accessed exclusively through two atomic operations: `wait` (historical Dutch: *proberen*, to test/decrement) and `post` (historical Dutch: *verhogen*, to increment).

$$\boxed{\textbf{Semaphore: } \text{An atomic non-negative integer counter representing available resource instances.}}$$

```
+-------------------------------------------------------------------------------+
|                    CRUCIAL ARCHITECTURAL SPECIFICATION NOTE                   |
+-------------------------------------------------------------------------------+
| As highlighted in your lecture notes (`Pthreada_Aagam_Notes.pdf`, Page 13):   |
|                                                                               |
| "Semaphores are not technically part of the base Pthreads library itself;     |
| instead, they are defined in the POSIX Real-time Extensions library under     |
| `<semaphore.h>` (IEEE Std 1003.1b)."                                          |
|                                                                               |
| When using semaphores, you must `#include <semaphore.h>` and link with        |
| `-pthread` on Linux.                                                          |
+-------------------------------------------------------------------------------+
```

---

### The Two Types of Semaphores

1. **Counting Semaphore (Initial Value $> 1$):**  
   Manages a finite pool of identical resources (e.g., initialized to $5$ for five available printer ports).
   - Each thread taking a resource decrements the counter.
   - When the counter reaches $0$, subsequent threads sleep until a resource is returned.
2. **Binary Semaphore (Initial Value $= 1$):**  
   Can only take values $0$ or $1$. Operates similarly to a mutual exclusion lock, but with fundamentally different ownership rules.

```
                         SEMAPHORE STATE MACHINE (sem = 2)
                         
                                [ Initial State: sem = 2 ]
                               (2 resource instances free)
                                            |
                         Thread A calls     |     Thread B calls
                         sem_wait(&sem)     |     sem_wait(&sem)
                                            v
                                [ Intermediate: sem = 0 ]
                                (0 resource instances free)
                                            |
                                            | Another thread C calls
                                            | sem_wait(&sem)
                                            v
                                [ Thread C BLOCKED / ASLEEP ]
                                            |
                                            | Thread A finishes and calls
                                            | sem_post(&sem)
                                            v
                                [ Thread C AWAKENS & PROCEEDS ]
```

---

## 3. The POSIX `<semaphore.h>` API Specification

POSIX supports two kinds of semaphores:
- **Named Semaphores (`sem_open`):** Created with a filesystem path (e.g., `/my_sem`), used for synchronization across independent, unrelated processes.
- **Unnamed Semaphores (`sem_init`):** Memory-based semaphores residing in process memory, used for threads or shared-memory child processes. (This is the primary exam focus).

---

### 1. Initialization: `sem_init()`

```c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
```

```
+-------------------------------------------------------------------------------+
|                         SEM_INIT PARAMETER BREAKDOWN                          |
+-------------------------------------------------------------------------------+
| 1. sem_t *sem                                                                 |
|    - Pointer to the semaphore object to initialize.                           |
|                                                                               |
| 2. int pshared (Process-Shared Flag)                                          |
|    - 0: Thread-Shared. The semaphore is shared strictly between threads of    |
|         the current process. Should be located in global memory or heap.      |
|    - Non-zero (1): Process-Shared. The semaphore is shared across multiple    |
|         distinct processes. Must be located in a POSIX shared memory region.  |
|                                                                               |
| 3. unsigned int value                                                         |
|    - The initial non-negative counter value of the semaphore.                 |
|    - Setting value = 1 creates a Binary Semaphore.                            |
|    - Setting value = K creates a Counting Semaphore with K resource slots.    |
+-------------------------------------------------------------------------------+
```

---

### 2. Decrement / Acquire: `sem_wait()`

```c
int sem_wait(sem_t *sem);
```
- **If `value > 0`:** Atomically decrements the counter by $1$ (`value--`) and returns immediately.
- **If `value == 0`:** The calling thread is immediately **blocked (suspended)** by the kernel and placed in the semaphore's wait queue. It consumes **zero CPU cycles** while waiting.

#### Non-Blocking Variant: `sem_trywait()`
```c
int sem_trywait(sem_t *sem);
```
Attempts to decrement immediately. If `value == 0`, it does not block; it returns immediately with error code `EAGAIN`.

---

### 3. Increment / Release: `sem_post()`

```c
int sem_post(sem_t *sem);
```
- Atomically increments the semaphore counter by $1$ (`value++`).
- If one or more threads are currently sleeping in `sem_wait()`, the operating system wakes up one waiting thread, allowing it to complete its `sem_wait()` and proceed.
- **Signal-Safe:** `sem_post()` is one of the very few POSIX synchronization primitives that is **async-signal-safe** (can be safely called inside UNIX signal handlers like `SIGINT`).

---

### 4. Destruction: `sem_destroy()`

```c
int sem_destroy(sem_t *sem);
```
- Frees any operating system resources associated with the unnamed semaphore.
- Fails with `EBUSY` if threads are currently blocked on the semaphore.

---

## 4. The Classic Exam Question: Binary Semaphore vs Mutex

One of the most heavily tested questions in university systems examinations is:  
**"What is the difference between a Binary Semaphore and a Mutex? Can a Binary Semaphore always replace a Mutex?"**

As noted in your lecture notes (`IMG_1333.pdf`, Page 10: *"diff b/w bin-sem v/s mutex"*), the answer lies in **Ownership Semantics**:

```
+-----------------------------------------------------------------------------------+
|                     BINARY SEMAPHORE vs MUTEX: DEEP COMPARISON                    |
+-----------------------------------------------------------------------------------+
| Feature Dimension     | Mutex (`pthread_mutex_t`)    | Binary Semaphore (`sem_t`)         |
| :-------------------- | :--------------------------- | :--------------------------------- |
| **Core Purpose**      | **Mutual Exclusion** over    | **Signaling & Resource Counting**  |
|                       | a critical section           | across threads/processes           |
| **Ownership Concept** | **STRICT OWNERSHIP:**         | **NO OWNERSHIP:**                  |
|                       | The thread that locks a      | Any thread can call `sem_post()`,  |
|                       | mutex MUST be the thread     | even if it never called `sem_wait()`|
|                       | that unlocks it!              |                                    |
| **Priority Inversion**| **Protected via Protocols:** | **Vulnerable to Inversion:**       |
|                       | Supports Priority Inheritance| Cannot support priority inheritance|
|                       | (`PTHREAD_PRIO_INHERIT`)      | because there is no owner thread!  |
| **Reentrancy**        | Can be configured as         | Non-reentrant (calling wait twice  |
|                       | **Recursive Mutex**          | from same thread deadlocks!)       |
| **Error Checking**    | Can detect double-unlocks    | Cannot detect illegal unlocks      |
|                       | (`PTHREAD_MUTEX_ERRORCHECK`)  | (`sem_post` just increments count) |
| **Async Signal Safety**| **Unsafe:** Never call in a  | **Safe:** `sem_post()` can be run  |
|                       | signal handler (deadlock risk)| inside signal handlers             |
+-----------------------------------------------------------------------------------+
```

### The Priority Inversion Catastrophe (Mars Pathfinder Problem)
- Suppose a Low-Priority thread locks a Mutex.
- A High-Priority thread arrives and blocks on the Mutex.
- A Medium-Priority thread preempts the Low-Priority thread.
- Result: The High-Priority thread is indirectly starved by the Medium-Priority thread!
- **With a Mutex:** The OS elevates the Low-Priority thread's priority to match the High-Priority thread (**Priority Inheritance Protocol**), allowing it to finish quickly and unlock the mutex.
- **With a Semaphore:** The OS has no record of which thread "owns" the semaphore; therefore, priority inheritance is physically impossible!

---

## 5. Complete Compilable C Program: POSIX Semaphore Counter

Here is the complete multithreaded counter application using POSIX semaphores transcribed and enhanced from your lecture notes (`Pthreada_Aagam_Notes.pdf`, Pages 13–14):

```c
/**
 * ============================================================================
 * PROGRAM:     pthreads_semaphore_counter.c
 * DESCRIPTION: N threads concurrently increment a shared counter using a POSIX
 *              unnamed binary semaphore (sem_t) for critical section protection.
 * COMPILATION: gcc -Wall -O2 pthreads_semaphore_counter.c -o pthreads_sem -pthread
 * USAGE:       ./pthreads_sem
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>

#define NUM_THREADS 4
#define ITERATIONS_PER_THREAD 25000

// Shared counter variable
int counter = 0;

// POSIX Unnamed Semaphore
sem_t sem;

// Worker routine
void* worker_increment(void *arg) {
    long id = (long)arg;

    for (int i = 0; i < ITERATIONS_PER_THREAD; i++) {
        // Decrement semaphore (Wait / Lock)
        sem_wait(&sem);

        // Critical Section: Mutually exclusive access
        counter++;

        // Increment semaphore (Signal / Unlock)
        sem_post(&sem);
    }

    printf("[Thread %ld] Completed its increment iterations.\n", id);
    pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
    pthread_t threads[NUM_THREADS];
    int rc;

    printf("[Main] Initializing POSIX Binary Semaphore...\n");
    // sem_init parameters:
    // 1. &sem: Pointer to semaphore
    // 2. 0: Thread-shared within current process (pshared = 0)
    // 3. 1: Initial value = 1 (Binary Semaphore behavior)
    rc = sem_init(&sem, 0, 1);
    if (rc != 0) {
        perror("sem_init failed");
        exit(EXIT_FAILURE);
    }

    // Spawn worker threads
    for (long i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, worker_increment, (void*)i);
    }

    // Join worker threads
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // Destroy Semaphore
    sem_destroy(&sem);

    int expected = NUM_THREADS * ITERATIONS_PER_THREAD;
    printf("\n=======================================================\n");
    printf("FINAL COUNTER VALUE:    %d\n", counter);
    printf("EXPECTED COUNTER VALUE: %d\n", expected);
    if (counter == expected) {
        printf("VERDICT: SUCCESS! Semaphore protected critical section.\n");
    } else {
        printf("VERDICT: FAILED! Data race detected.\n");
    }
    printf("=======================================================\n");

    return 0;
}
```

---

## 6. Read-Write Locks: The Architecture of Asymmetric Concurrency

In many high-concurrency systems, access to shared memory is overwhelmingly **read-dominated**:
- An airline reservation system: 1,000 customers check flight seat availability for every 1 customer who books a seat.
- A DNS name server cache: Millions of IP address lookups per hour; updates occur only when domain records expire.
- An in-memory product catalog on an e-commerce website.

If you protect an e-commerce catalog with a standard `pthread_mutex_t`, **only one user in the entire world can view a product page at a time**. All other 9,999 users are queued in a sleep state, destroying throughput.

### The Solution: `pthread_rwlock_t`

$$\boxed{\textbf{Read-Write Lock: } \text{A synchronization lock that permits multiple concurrent readers (Shared Access)}}$$
$$\boxed{\text{while enforcing strict, isolated exclusivity for writers (Exclusive Access).}}$$

---

### The Intuitive Analogy: The Public Blackboard

As highlighted in your lecture notes (`Pthreada_Aagam_Notes.pdf`, Page 15: *"RBI guidelines: Jaankaar Rahiye Satark Rahiye"*):

```
+-------------------------------------------------------------------------------+
|                       THE PUBLIC BLACKBOARD ANALOGY                           |
+-------------------------------------------------------------------------------+
| 1. READING THE BLACKBOARD (Shared Read Lock):                                 |
|    - Suppose an important public financial advisory is written on a blackboard.|
|    - 50 people can stand in front of the board and read it SIMULTANEOUSLY!     |
|    - Readers do not modify the chalk text, so they do not interfere with each |
|      other. Multiple readers can coexist in perfect safety.                   |
|                                                                               |
| 2. WRITING ON THE BLACKBOARD (Exclusive Write Lock):                          |
|    - When the bank officer needs to erase the board and write new interest    |
|      rates, NO ONE CAN BE READING!                                            |
|    - If a person attempts to read while the officer is halfway through erasing|
|      and rewriting a number, they will see torn, corrupted, meaningless data. |
|    - The writer demands COMPLETE EXCLUSIVITY: All readers and other writers   |
|      must step away until the write is finished.                              |
+-------------------------------------------------------------------------------+
```

---

## 7. The POSIX `pthread_rwlock_t` API & Lock States

```c
#include <pthread.h>

pthread_rwlock_t rwlock;
```

```
+-----------------------------------------------------------------------------------+
|                        READ-WRITE LOCK COMPATIBILITY MATRIX                       |
+-----------------------------------------------------------------------------------+
| Current Lock State    | New Request: READ LOCK       | New Request: WRITE LOCK    |
| :-------------------- | :--------------------------- | :------------------------- |
| **UNLOCKED**          | **GRANTED IMMEDIATELY**      | **GRANTED IMMEDIATELY**    |
| **READ LOCKED**       | **GRANTED IMMEDIATELY**      | **BLOCKED** (Waits)        |
| (1 or more readers)   | (Reader counter incremented) |                            |
| **WRITE LOCKED**      | **BLOCKED** (Waits)          | **BLOCKED** (Waits)        |
| (1 exclusive writer)  |                              |                            |
+-----------------------------------------------------------------------------------+
```

### The API Functions:
1. **Initialize:** `int pthread_rwlock_init(pthread_rwlock_t *rwlock, const pthread_rwlockattr_t *attr);`
2. **Acquire Read Lock:** `int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);`  
   *(Blocks only if a writer currently holds the lock, or if a writer is waiting under writer-preference policies).*
3. **Acquire Write Lock:** `int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);`  
   *(Blocks if ANY reader or writer currently holds the lock).*
4. **Release Lock:** `int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);`  
   *(Releases whichever lock mode was currently held).*
5. **Destroy:** `int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);`

---

## 8. Complete Compilable C Program: Read-Write Lock Demonstration

Here is the complete, production-grade C application demonstrating concurrent readers and exclusive writers transcribed and enhanced from your lecture notes (`Pthreada_Aagam_Notes.pdf`, Pages 15–16):

```c
/**
 * ============================================================================
 * PROGRAM:     pthreads_rwlock_demo.c
 * DESCRIPTION: Demonstrates POSIX Read-Write Locks (pthread_rwlock_t).
 *              Multiple reader threads read shared data concurrently without
 *              blocking each other, while writer threads obtain exclusive locks.
 * COMPILATION: gcc -Wall -O2 pthreads_rwlock_demo.c -o pthreads_rwlock -pthread
 * USAGE:       ./pthreads_rwlock
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

#define NUM_READERS 4
#define NUM_WRITERS 2

// Shared resource: Bank Account Balance
int shared_account_balance = 1000;

// POSIX Read-Write Lock
pthread_rwlock_t rwlock;

// ----------------------------------------------------------------------------
// READER THREAD ROUTINE (Shared Read Access)
// ----------------------------------------------------------------------------
void* reader_routine(void *arg) {
    long id = (long)arg;

    for (int i = 0; i < 3; i++) {
        // Acquire Shared Read Lock
        pthread_rwlock_rdlock(&rwlock);

        // Multiple readers execute this block SIMULTANEOUSLY!
        printf("[Reader %ld] START read: Account Balance = $%d\n", id, shared_account_balance);
        
        // Sleep briefly to demonstrate simultaneous reader execution
        usleep(100000); // 100 ms
        
        printf("[Reader %ld] FINISHED read: Account Balance = $%d\n", id, shared_account_balance);

        // Release Read Lock
        pthread_rwlock_unlock(&rwlock);

        usleep(150000); // Rest before next read
    }

    pthread_exit(NULL);
}

// ----------------------------------------------------------------------------
// WRITER THREAD ROUTINE (Exclusive Write Access)
// ----------------------------------------------------------------------------
void* writer_routine(void *arg) {
    long id = (long)arg;

    for (int i = 0; i < 2; i++) {
        // Acquire Exclusive Write Lock
        pthread_rwlock_wrlock(&rwlock);

        // EXCLUSIVE ACCESS: Zero readers or other writers can execute here!
        int deposit = (id + 1) * 250;
        shared_account_balance += deposit;
        printf("\n>>> [Writer %ld] EXCLUSIVE LOCK ACQUIRED! Deposited $%d. New Balance = $%d <<<\n\n", 
               id, deposit, shared_account_balance);
        
        // Hold lock for a moment
        usleep(200000); // 200 ms

        // Release Write Lock
        pthread_rwlock_unlock(&rwlock);

        usleep(300000); // Rest before next write
    }

    pthread_exit(NULL);
}

// ----------------------------------------------------------------------------
// MAIN LIFECYCLE
// ----------------------------------------------------------------------------
int main(int argc, char *argv[]) {
    pthread_t readers[NUM_READERS];
    pthread_t writers[NUM_WRITERS];

    printf("================================================================\n");
    printf("           POSIX READ-WRITE LOCK ARCHITECTURE DEMO              \n");
    printf("   Readers: %d (Shared) | Writers: %d (Exclusive)\n", NUM_READERS, NUM_WRITERS);
    printf("================================================================\n\n");

    // 1. Initialize Read-Write Lock
    pthread_rwlock_init(&rwlock, NULL);

    // 2. Spawn Readers and Writers
    for (long i = 0; i < NUM_READERS; i++) {
        pthread_create(&readers[i], NULL, reader_routine, (void*)i);
    }
    for (long i = 0; i < NUM_WRITERS; i++) {
        pthread_create(&writers[i], NULL, writer_routine, (void*)i);
    }

    // 3. Wait for all threads to finish
    for (int i = 0; i < NUM_READERS; i++) {
        pthread_join(readers[i], NULL);
    }
    for (int i = 0; i < NUM_WRITERS; i++) {
        pthread_join(writers[i], NULL);
    }

    // 4. Destroy Read-Write Lock
    pthread_rwlock_destroy(&rwlock);

    printf("\n[Main] Final Account Balance: $%d\n", shared_account_balance);
    printf("[Main] All readers and writers completed cleanly.\n");
    return 0;
}
```

---

## 9. University Examination Preparation: Solved Questions

### Question 1: Core Comparison (10 Marks)
**Compare a Mutex and a Binary Semaphore. Explain why a Binary Semaphore cannot safely replace a Mutex in a real-time embedded system susceptible to Priority Inversion.**

#### Model Examination Answer:
1. **Ownership Semantics:**  
   - A **Mutex** has strict thread ownership. The specific thread that invokes `pthread_mutex_lock()` is the *only* thread permitted to invoke `pthread_mutex_unlock()`.
   - A **Binary Semaphore** has no concept of thread ownership. Any thread (or even an interrupt service routine) can call `sem_post()` to increment the semaphore, regardless of who called `sem_wait()`.
2. **Priority Inversion and Priority Inheritance:**  
   - In a real-time system, Priority Inversion occurs when a low-priority thread holding a lock is preempted by a medium-priority thread, indirectly stalling a high-priority thread that is waiting for the same lock.
   - **Mutex Protection:** Because the operating system tracks which thread owns a mutex, it can implement the **Priority Inheritance Protocol**. The low-priority owner temporarily inherits the high-priority thread's priority, allowing it to complete its critical section rapidly without preemption from medium-priority threads.
   - **Semaphore Vulnerability:** Because a semaphore has no owner, the operating system kernel cannot know which thread to promote. Priority inheritance cannot be implemented on semaphores, leaving real-time systems vulnerable to indefinite task starvation.
3. **Use-Case Distinction:** Mutexes are designed strictly for mutual exclusion over shared memory; semaphores are designed for signaling events and managing countable resource pools.

---

### Question 2: Architecture & Analysis (5 Marks)
**What is a Read-Write Lock (`pthread_rwlock_t`)? Under what workload characteristics does a Read-Write Lock outperform a standard Mutex, and when does it perform worse?**

#### Model Examination Answer:
1. **Definition:** A Read-Write Lock is a concurrency primitive that permits multiple threads to simultaneously acquire a shared read lock (`pthread_rwlock_rdlock`), but restricts write access (`pthread_rwlock_wrlock`) to a single thread that excludes all other readers and writers.
2. **When it Outperforms a Mutex:**  
   - **High Read-to-Write Ratio:** Workloads where read operations constitute $\ge 85-90\%$ of total accesses. Multiple readers execute in parallel on physical CPU cores without serializing.
   - **Long Read Critical Sections:** When reading and processing the shared data takes significant time (e.g., traversing a large binary search tree).
3. **When it Performs Worse:**  
   - **Write-Heavy Workloads:** If writes constitute $\ge 20-30\%$ of operations, writers continuously block readers, negating concurrency benefits.
   - **Extremely Short Critical Sections:** The internal bookkeeping overhead of a Read-Write Lock (maintaining reader reference counts and writer wait queues via atomic compare-and-swap instructions) is higher than that of a lightweight mutex. For trivial reads (e.g., reading a single integer), a mutex is often faster.

---

## 10. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 04.5 QUICK REVISION SHEET
================================================================================

1. POSIX SEMAPHORES (<semaphore.h>):
   - Atomic non-negative integer counter for resource pool management & signaling.
   - Defined in POSIX Real-time Extensions (IEEE 1003.1b), not base Pthreads.
   - sem_init(&sem, pshared, value):
     * pshared = 0 (Thread-shared within process)
     * pshared = 1 (Process-shared in shared memory)
   - sem_wait(&sem): Decrements (P operation); blocks if count == 0.
   - sem_post(&sem): Increments (V operation); awakens sleeping thread. Async-signal safe.
   - sem_destroy(&sem): Cleans up unnamed semaphore.

2. BINARY SEMAPHORE vs MUTEX:
   - Ownership: Mutex has strict owner (must unlock own lock). Semaphore has NO owner.
   - Priority Inheritance: Mutex supports it; Semaphore cannot.
   - Mutex is for locking; Semaphore is for signaling and resource counters.

3. READ-WRITE LOCKS (pthread_rwlock_t):
   - Multiple readers concurrently (pthread_rwlock_rdlock).
   - Only 1 writer exclusively (pthread_rwlock_wrlock).
   - Ideal for read-heavy workloads (database lookups, routing tables).
   - Blackboard analogy: Everyone can read simultaneously; writer needs total privacy.
================================================================================
```

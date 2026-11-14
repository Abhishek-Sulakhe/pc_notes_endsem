# Thread Attributes & Real-Time Scheduling Policies in Pthreads

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 04 — Shared-Memory Parallel Programming with Pthreads
SUBTOPIC:               02 — Thread Attributes & Real-Time Scheduling Policies
PREREQUISITES:          Subtopic 04.1 (POSIX Standards & Thread Lifecycle)
SOURCE LECTURE NOTES:   Pthreada_Aagam_Notes.pdf (Pages 4–5), IMG_1333.pdf (Pages 4–5)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: Beyond Default Threads

In Subtopic 04.1, we created threads by passing `NULL` as the second argument to `pthread_create()`:

```c
pthread_create(&thread_id, NULL, worker_function, &arg);
```

Passing `NULL` instructs the POSIX threading library: *"Spawn this thread with standard default operating system properties."*

While default properties suffice for introductory academic exercises, production systems—such as real-time avionics, high-frequency financial trading engines, embedded microcontrollers, and massive web servers—demand surgical control over:
- How much **memory** is allocated for the thread's runtime stack.
- Whether the thread terminates automatically as a **detached** worker or requires explicit **joining**.
- The operating system **scheduling algorithm** (`SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR`).
- The thread's **execution priority** relative to other tasks in the operating system.

As highlighted in your lecture notes (`Pthreada_Aagam_Notes.pdf`, Page 5 with the marginal alert: *"He will ask all this!"* and `IMG_1333.pdf`, Page 4: *"Exam-Q: How to run above all in Pthreads?"*), mastering the `pthread_attr_t` interface is a core requirement for university examinations.

---

## 2. The `pthread_attr_t` Object Architecture

POSIX manages thread configuration through the **Attribute Object Pattern**, an object-oriented design idiom in pure C that decouples thread properties from the thread creation call.

```
                          THE ATTRIBUTE OBJECT WORKFLOW
                          
   1. Allocate & Initialize       2. Configure Attributes        3. Instantiate Thread
   
     +-------------------+          +-------------------+          +-------------------+
     |  pthread_attr_t   |  ====>   |  pthread_attr_t   |  ====>   |  pthread_create() |
     |  attr;            |          |  - Detach State   |          |  (Uses configured |
     |  (Default State)  |          |  - Stack Size     |          |   attr object)    |
     +-------------------+          |  - Sched Policy   |          +---------+---------+
               ^                    |  - Sched Priority |                    |
               |                    +-------------------+                    v
      pthread_attr_init()                     ^                     Active Child Thread
                                              |                              |
                                      Setter Functions                       v
                                                                   4. Destroy Attributes
                                                                   
                                                                   pthread_attr_destroy()
```

### The Attribute Object Lifecycle Rules

1. **Initialization:** An attribute object must always be initialized before use:
   ```c
   int pthread_attr_init(pthread_attr_t *attr);
   ```
   This populates the object with system default values (joinable state, standard stack size, dynamic time-sharing priority).
2. **Reuse:** A single configured `pthread_attr_t` object can be used to spawn multiple threads possessing identical configurations.
3. **Immutability After Creation:** Modifying an attribute object *after* calling `pthread_create()` has **zero effect** on threads that have already been created.
4. **Destruction:** Once all intended threads have been spawned, the attribute object must be deallocated:
   ```c
   int pthread_attr_destroy(pthread_attr_t *attr);
   ```

---

## 3. Attribute 1: Thread Detach State (Joinable vs Detached)

Every thread in a POSIX system exists in one of two fundamental states: **Joinable** or **Detached**.

```
+-----------------------------------------------------------------------------------+
|                     JOINABLE THREADS vs DETACHED THREADS                          |
+-----------------------------------------------------------------------------------+
| Feature               | Joinable Threads (Default)   | Detached Threads                   |
| :-------------------- | :--------------------------- | :--------------------------------- |
| **POSIX Constant**    | `PTHREAD_CREATE_JOINABLE`    | `PTHREAD_CREATE_DETACHED`          |
| **Synchronization**   | Must be joined via           | Cannot be joined (`pthread_join`   |
|                       | `pthread_join()`             | returns `EINVAL` error)            |
| **Resource Cleanup**  | Retains stack & return status| Operating system immediately frees |
|                       | in memory until joined       | all memory when thread terminates  |
| **Zombie Threat**     | If not joined, creates       | Immune to zombie thread memory     |
|                       | a permanent memory leak!     | leaks                              |
| **Return Status**     | Can return `void*` pointer   | Cannot return any status to parent |
| **Ideal Use Case**    | Parallel for-loops, tasks    | Background worker daemons, logging |
|                       | where parent aggregates data | threads, fire-and-forget network   |
|                       | (e.g., matrix multiplication)| socket handlers                    |
+-----------------------------------------------------------------------------------+
```

### The Zombie Thread Threat
When a Joinable thread terminates, the operating system kernel cannot immediately destroy its resources. The kernel must preserve the thread's return code (`void* retval`) and thread descriptor in memory so that a future call to `pthread_join()` can retrieve them. 

If a long-running server program spawns thousands of joinable threads and fails to call `pthread_join()`, the system accumulates **zombie threads**. Eventually, the system runs out of thread descriptors and crashes with `EAGAIN`.

### How to Configure Detached State

#### Method A: Before Thread Creation (Using `pthread_attr_t`)
```c
pthread_attr_t attr;
pthread_attr_init(&attr);

// Set the detach state to DETACHED
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

pthread_t tid;
pthread_create(&tid, &attr, background_worker, NULL);

pthread_attr_destroy(&attr);
```

#### Method B: After Thread Creation (Dynamic Self-Detachment)
A thread that was created as joinable can convert itself into a detached thread dynamically at runtime:
```c
void* background_worker(void *arg) {
    // Detach myself immediately upon starting
    pthread_detach(pthread_self());
    
    // Perform work...
    return NULL; // All resources automatically reclaimed on exit!
}
```

---

## 4. Attribute 2: Thread Stack Management

When a new process launches, the operating system allocates a virtual address space. However, when a thread is created within that process, the Pthreads runtime must carve out a private **Thread Runtime Stack** from the process address space.

```
+-------------------------------------------------------------------------------+
|                       THE THREAD STACK EXPLOSION PROBLEM                      |
+-------------------------------------------------------------------------------+
| Default Linux Stack Size:               8 Megabytes (8,388,608 bytes)         |
| Virtual Address Space for 10,000 threads: 10,000 x 8 MB = 80 Gigabytes!       |
|                                                                               |
| On a 32-bit system (4 GB address limit) or a cloud instance with 8 GB RAM,    |
| attempting to spawn 1,000 default threads will crash the process:             |
| `pthread_create: Cannot allocate memory (EAGAIN)`                             |
+-------------------------------------------------------------------------------+
```

### When to Customize Stack Size?
1. **Downsizing for High Thread Counts:** If your application spawns $5,000$ lightweight worker threads that only execute shallow mathematical calculations, each thread may need only $64\text{ KB}$ of stack memory. Downsizing the stack from $8\text{ MB}$ to $64\text{ KB}$ reduces memory footprint from $40\text{ GB}$ to $320\text{ MB}$!
2. **Upsizing for Deep Recursion:** If a thread executes deeply recursive divide-and-conquer algorithms (e.g., QuickSort over massive datasets) or allocates large local stack arrays (`double local_buffer[500000]`), the default stack will overflow, triggering a Segmentation Fault (`SIGSEGV`).

### Setting the Stack Size in Pthreads

```c
#include <pthread.h>
#include <limits.h>

pthread_attr_t attr;
pthread_attr_init(&attr);

// Query default stack size
size_t default_stack;
pthread_attr_getstacksize(&attr, &default_stack);
printf("Default Stack Size: %zu bytes\n", default_stack);

// Set custom stack size (e.g., 4 Megabytes)
size_t custom_stack = 4 * 1024 * 1024; // 4 MB

// POSIX requires stack size to be at least PTHREAD_STACK_MIN (typically 16 KB)
if (custom_stack >= PTHREAD_STACK_MIN) {
    pthread_attr_setstacksize(&attr, custom_stack);
}

pthread_create(&tid, &attr, worker, NULL);
pthread_attr_destroy(&attr);
```

---

## 5. Attribute 3: POSIX Real-Time Scheduling Policies

The operating system scheduler decides which thread runs on which physical CPU core at any given instant. POSIX standardizes three distinct scheduling policies:

```
+-----------------------------------------------------------------------------------+
|                        POSIX SCHEDULING POLICIES COMPARISON                       |
+-----------------------------------------------------------------------------------+
| Policy Identifier     | Policy Category   | Preemption Type  | Time-Slicing       |
| :-------------------- | :---------------- | :--------------- | :----------------- |
| **`SCHED_OTHER`**     | Standard / Normal | Dynamic Priority | Yes (Dynamic CFS)  |
| **`SCHED_FIFO`**      | Real-Time         | Static Priority  | **NO** (Runs to end)|
| **`SCHED_RR`**        | Real-Time         | Static Priority  | **YES** (Fixed slice)|
+-----------------------------------------------------------------------------------+
```

---

### 1. `SCHED_OTHER` (Default Time-Sharing Policy)
- The standard, general-purpose scheduling policy used for ordinary desktop and server tasks.
- In Linux, `SCHED_OTHER` is implemented using the **Completely Fair Scheduler (CFS)**.
- Operates on dynamic priorities: The kernel dynamically decreases the priority of CPU-heavy threads and increases the priority of interactive, I/O-bound threads to maintain responsive user interfaces.
- Standard threads cannot interrupt or preempt real-time threads.

---

### 2. `SCHED_FIFO` (Real-Time First-In, First-Out)
- A high-priority, real-time static priority policy (priority levels $1$ to $99$).
- **Non-Time-Sliced:** Once a `SCHED_FIFO` thread acquires a CPU core, **it runs continuously until**:
  1. It voluntarily blocks (e.g., waits for I/O, sleeps, or waits on a mutex/condition variable).
  2. It voluntarily yields the CPU via `sched_yield()`.
  3. A higher-priority real-time thread becomes ready to execute.
- **Danger:** An infinite loop inside a `SCHED_FIFO` thread will completely lock up a CPU core, starving all normal operating system processes!

---

### 3. `SCHED_RR` (Real-Time Round-Robin)
- A real-time static priority policy identical to `SCHED_FIFO`, but with **time-slicing (quantum) enforcement**.
- When multiple threads share the **exact same real-time priority level**, the OS scheduler allocates each thread a fixed time slice (e.g., $10\text{ ms}$).
- When a thread's time slice expires, it is preempted and moved to the back of the priority queue, giving the next equal-priority thread an opportunity to run.

```
========================================================================================
TIMELINE COMPARISON: SCHED_FIFO vs SCHED_RR (Two Equal-Priority Real-Time Threads)
========================================================================================

SCHED_FIFO (No Time Slicing):
Thread 1 [======================== RUNS UNTIL FINISHED ========================>]
Thread 2 -----------------------------------------------------> WAITING... [== RUNS ==>]

SCHED_RR (Time-Sliced Round-Robin):
Core:    [ T1 Slice ] [ T2 Slice ] [ T1 Slice ] [ T2 Slice ] [ T1 Slice ] [ T2 Slice ]
         |<--10ms-->| |<--10ms-->| |<--10ms-->| |<--10ms-->| |<--10ms-->| |<--10ms-->|
```

---

## 6. Attribute 4: Thread Priorities & `struct sched_param`

To assign a priority to a thread, POSIX defines a dedicated scheduling parameter structure in `<sched.h>`:

```c
struct sched_param {
    int sched_priority; // Static priority value
};
```

### Static Priority Bounds
Priority numbers are not arbitrary. For `SCHED_FIFO` and `SCHED_RR`, valid priority values must lie between the system's minimum and maximum bounds:

```c
int min_priority = sched_get_priority_min(SCHED_RR); // Typically 1 on Linux
int max_priority = sched_get_priority_max(SCHED_RR); // Typically 99 on Linux
```
*(Higher numeric values denote higher execution precedence).*

### The Critical "Inherit Scheduling" Requirement
By default, newly created Pthreads inherit their scheduling policy and priority from the parent thread that created them (`PTHREAD_INHERIT_SCHED`).

If you attempt to set a new scheduling policy or priority without changing the inheritance attribute, **the Pthreads runtime will silently ignore your configuration!**

You must explicitly configure **`PTHREAD_EXPLICIT_SCHED`**:

```c
// CRITICAL: Must inform Pthreads to use our explicit attributes!
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
```

---

## 7. Attribute 5: Scheduling Contention Scope

POSIX threads can operate under two scheduling contention models:

```c
int pthread_attr_setscope(pthread_attr_t *attr, int scope);
```

1. **`PTHREAD_SCOPE_SYSTEM` (System Contention Scope — 1:1 Model):**
   - The thread competes for CPU execution cycles against **all other threads across the entire operating system**.
   - Each POSIX user thread is bound directly to an independent kernel task/lightweight process (LWP) managed by the OS kernel scheduler.
   - This is the standard, mandatory model on modern Linux (`NPTL` — Native POSIX Thread Library).
2. **`PTHREAD_SCOPE_PROCESS` (Process Contention Scope — M:N Model):**
   - User-level threads compete for execution time only against other threads *within the same user process*.
   - A user-level threading library multiplexes $M$ user threads onto $N$ kernel threads. (Historical model, largely abandoned in modern Linux).

---

## 8. Complete Compilable C Program: Comprehensive `pthread_attr_t` Configuration

Here is the complete, production-grade C program answering the exact examination question: *"How to define and use stack size, scheduling policy, detached state, and priority via `pthread_attr_t`?"*

```c
/**
 * ============================================================================
 * PROGRAM:     pthreads_attributes_demo.c
 * DESCRIPTION: Complete demonstration of pthread_attr_t lifecycle:
 *              - Detached state (PTHREAD_CREATE_DETACHED)
 *              - Custom stack size (4 MB)
 *              - Explicit scheduling (PTHREAD_EXPLICIT_SCHED)
 *              - Real-time Round-Robin policy (SCHED_RR) with priority
 *              - System contention scope (PTHREAD_SCOPE_SYSTEM)
 * COMPILATION: gcc -Wall -O2 pthreads_attributes_demo.c -o pthreads_attr -pthread
 * NOTE:        Setting real-time policies (SCHED_RR/FIFO) requires administrative
 *              privileges on Linux. Run with `sudo ./pthreads_attr` or fallback
 *              to SCHED_OTHER if unprivileged.
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <sched.h>
#include <unistd.h>
#include <errno.h>

// Worker thread routine
void* real_time_worker(void *arg) {
    int id = *(int*)arg;
    free(arg); // Free dynamically allocated argument memory

    printf("[Worker %d] Running with Detached State and Custom Stack!\n", id);
    printf("[Worker %d] Executing real-time computational workload...\n", id);
    
    // Simulate computational task
    usleep(200000); // 200 ms
    
    printf("[Worker %d] Completed task. Terminating (auto memory cleanup).\n", id);
    pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
    pthread_t thread;
    pthread_attr_t attr;
    struct sched_param param;
    int rc;

    printf("[Main] Initializing thread attributes...\n");

    // 1. Initialize attribute object
    rc = pthread_attr_init(&attr);
    if (rc != 0) {
        perror("pthread_attr_init failed");
        exit(EXIT_FAILURE);
    }

    // 2. Configure Detached State
    rc = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    if (rc != 0) {
        perror("Setting detached state failed");
    }

    // 3. Configure Custom Stack Size (4 MB)
    size_t custom_stack = 4 * 1024 * 1024; // 4 MB
    rc = pthread_attr_setstacksize(&attr, custom_stack);
    if (rc != 0) {
        perror("Setting stack size failed");
    }

    // 4. Configure System Contention Scope (1:1 Kernel Model)
    pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);

    // 5. MANDATORY: Enable Explicit Scheduling
    // If not set, policy and priority modifications are silently ignored!
    rc = pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
    if (rc != 0) {
        perror("Setting explicit sched failed");
    }

    // 6. Configure Real-Time Scheduling Policy & Priority
    int policy = SCHED_RR; // Real-Time Round-Robin
    rc = pthread_attr_setschedpolicy(&attr, policy);
    if (rc != 0) {
        perror("Setting sched policy failed");
    }

    // Query priority limits for SCHED_RR
    int min_prio = sched_get_priority_min(policy);
    int max_prio = sched_get_priority_max(policy);
    printf("[Main] SCHED_RR Priority Bounds: Min = %d, Max = %d\n", min_prio, max_prio);

    // Set priority halfway between min and max
    param.sched_priority = min_prio + (max_prio - min_prio) / 2;
    rc = pthread_attr_setschedparam(&attr, &param);
    if (rc != 0) {
        perror("Setting sched param failed");
    }

    // 7. Allocate dynamic memory for thread argument
    int *arg_id = (int*)malloc(sizeof(int));
    *arg_id = 42;

    // 8. Spawn Thread with Configured Attributes
    printf("[Main] Spawning thread with configured attributes...\n");
    rc = pthread_create(&thread, &attr, real_time_worker, (void*)arg_id);
    
    // Check if unprivileged execution blocked real-time scheduling
    if (rc == EPERM) {
        printf("[Main Warning] Real-time policy requires root privileges (EPERM).\n");
        printf("[Main Fallback] Re-trying with default SCHED_OTHER...\n");
        pthread_attr_setschedpolicy(&attr, SCHED_OTHER);
        param.sched_priority = 0;
        pthread_attr_setschedparam(&attr, &param);
        rc = pthread_create(&thread, &attr, real_time_worker, (void*)arg_id);
    }

    if (rc != 0) {
        fprintf(stderr, "Failed to create thread: error %d\n", rc);
        exit(EXIT_FAILURE);
    }

    // 9. Destroy Attribute Object (Safe to destroy immediately after create)
    pthread_attr_destroy(&attr);
    printf("[Main] Attribute object destroyed. Thread running detached.\n");

    // Because thread is DETACHED, main CANNOT call pthread_join(thread, NULL)!
    // We sleep briefly to give child thread time to run before main exits.
    usleep(500000); // 500 ms

    printf("[Main] Process finished. Exiting.\n");
    return 0;
}
```

---

## 9. University Examination Preparation: Solved Questions

### Question 1: Direct Notes Question (10 Marks)
**Explain the role of the `pthread_attr_t` structure in POSIX threads programming. Write C code snippets demonstrating how to configure:**
1. **Thread stack size**
2. **Detached execution state**
3. **Scheduling policy and thread priority**
**Why is `pthread_attr_setinheritsched()` mandatory when modifying scheduling policies?**

#### Model Examination Answer:
1. **Role of `pthread_attr_t`:**  
   `pthread_attr_t` is an opaque data type used to encapsulate thread configuration attributes before thread creation. It follows the Attribute Object Pattern, allowing developers to configure properties (such as stack dimensions, detached state, and real-time scheduling) independently of the `pthread_create()` call.
2. **Configuration Code Snippets:**
   - **Stack Size:**
     ```c
     pthread_attr_t attr;
     pthread_attr_init(&attr);
     pthread_attr_setstacksize(&attr, 2 * 1024 * 1024); // 2 MB
     ```
   - **Detached State:**
     ```c
     pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
     ```
   - **Scheduling Policy & Priority:**
     ```c
     struct sched_param param;
     pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
     param.sched_priority = 50;
     pthread_attr_setschedparam(&attr, &param);
     ```
3. **Why `pthread_attr_setinheritsched()` is Mandatory:**  
   By default, POSIX threads inherit scheduling policies and priorities directly from the parent calling thread (`PTHREAD_INHERIT_SCHED`). If a programmer calls `pthread_attr_setschedpolicy()` or `pthread_attr_setschedparam()` without explicitly calling:
   ```c
   pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
   ```
   the Pthreads implementation will silently discard the custom policy and priority values, reverting to the parent's scheduling parameters during `pthread_create()`.

---

### Question 2: Technical Comparison (5 Marks)
**Differentiate between `SCHED_FIFO` and `SCHED_RR` scheduling policies in Pthreads. In what scenario can a `SCHED_FIFO` thread inadvertently cause system deadlock or starvation?**

#### Model Examination Answer:
1. **Comparison:**
   - **Preemption & Time-Slicing:** Both are real-time static priority policies ($1$ to $99$). However, `SCHED_FIFO` is **not time-sliced**; a thread runs until it terminates, blocks on I/O, yields the CPU, or is preempted by a strictly higher-priority thread. In contrast, `SCHED_RR` is **time-sliced**; multiple threads with the same priority share the CPU in round-robin turns governed by a time quantum.
   - **Equal-Priority Handling:** In `SCHED_FIFO`, an equal-priority thread must wait until the running thread finishes or yields. In `SCHED_RR`, an equal-priority thread runs automatically after the current thread's time slice expires.
2. **Starvation Scenario in `SCHED_FIFO`:**  
   If a high-priority `SCHED_FIFO` thread enters an infinite computation loop (e.g., `while(1) { sum++; }`) without calling `sleep()`, `sched_yield()`, or performing I/O, it will permanently monopolize the CPU core. Normal time-sharing processes (`SCHED_OTHER`) and equal-or-lower priority real-time threads will suffer total starvation, freezing that core completely.

---

### Question 3: Short Answer (5 Marks)
**What is a detached thread in Pthreads? How does it differ from a joinable thread in terms of memory reclamation and calling `pthread_join()`?**

#### Model Examination Answer:
1. **Definition:** A detached thread (`PTHREAD_CREATE_DETACHED`) is a POSIX thread configured to run independently in a fire-and-forget manner.
2. **Memory Reclamation:**  
   - In a **Joinable thread**, when the thread finishes execution, its stack, thread descriptor, and exit status remain allocated in kernel memory until another thread calls `pthread_join()`.
   - In a **Detached thread**, the operating system kernel automatically frees and destroys all thread resources (stack and descriptor) the instant the thread terminates.
3. **Calling `pthread_join()`:**  
   Calling `pthread_join()` on a detached thread is illegal and fails immediately with error code `EINVAL` (Invalid Argument).

---

## 10. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 04.2 QUICK REVISION SHEET
================================================================================

1. PTHREAD_ATTR_T LIFECYCLE:
   - Initialize: pthread_attr_init(&attr)
   - Configure: Setters for detach state, stack, scheduling, priority
   - Spawn: pthread_create(&tid, &attr, worker, arg)
   - Cleanup: pthread_attr_destroy(&attr)

2. DETACH STATE:
   - PTHREAD_CREATE_JOINABLE: Must be joined; holds memory until pthread_join().
   - PTHREAD_CREATE_DETACHED: OS automatically frees resources on thread exit.
   - Dynamic detachment: pthread_detach(pthread_self())

3. STACK SIZE:
   - pthread_attr_setstacksize(&attr, size_in_bytes)
   - Minimum bound: PTHREAD_STACK_MIN
   - Prevents memory exhaustion when spawning thousands of lightweight threads.

4. SCHEDULING POLICIES:
   - SCHED_OTHER: Standard dynamic time-sharing (Linux CFS).
   - SCHED_FIFO: Real-time static priority; runs to completion (NO time-slice).
   - SCHED_RR: Real-time static priority with fixed round-robin time-slicing.

5. EXPLICIT SCHEDULING RULE:
   - Must call pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
     otherwise custom policy and priority changes are SILENTLY IGNORED!
================================================================================
```

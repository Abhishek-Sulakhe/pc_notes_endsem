# Source Extraction: IMG_1333.pdf

**Source File:** `input_notes/IMG_1333.pdf`  
**Total Pages:** 27  
**Core Topics Covered:**
1. **POSIX Threads (Pthreads):** Concepts, standards, OS support, thread creation (`pthread_create`), joining (`pthread_join`), attributes (`pthread_attr_t`), thread IDs (`pthread_t`), synchronization mechanisms (Mutex, Condition Variables, Spinlocks, Read-Write Locks, Barriers, Semaphores), and performance trade-offs (Mutex vs. Spinlock performance cliff).
2. **Producer-Consumer / Bounded Buffer Problem:** Implementation using Pthreads mutexes and condition variables (`pthread_cond_wait`, `pthread_cond_signal`).
3. **Message Passing Interface (MPI):** Distributed memory architecture, process initialization (`MPI_Init`), ranking (`MPI_Comm_rank`), communicator sizing (`MPI_Comm_size`), finalization (`MPI_Finalize`), point-to-point communication (`MPI_Send`, `MPI_Recv`), process execution controls (`mpirun`, `--oversubscribe`, `--display-allocation`, hostfiles), and MPI-2 dynamic inter-communicator connections via ports and `ompi-server` (`MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect`).

---

## Page 01

### Topic: Introduction to Pthreads (POSIX Threads) & Standards
**Date on Note:** 25/09/26  
**Context / Question:** `What is X?` -> `POSIX - Portable Operating System Interface`

### Conceptual Explanations:
1. **Definition & Purpose:**
   - Developed by IEEE Computer Society, POSIX is a family of standards designed to ensure compatibility and software portability between different operating systems.
2. **Application Rules:**
   - POSIX outlines universal rules for application programs.
3. **Officially Certified / Compliant Operating Systems:**
   - **Linux Distributions:** Ubuntu, Fedora, Red Hat, etc.
   - **BSD Operating Systems:** FreeBSD, OpenBSD, NetBSD.
   - **Android:** Uses a Linux-based kernel with a modified C library (Bionic).

### Marginal Note / Question:
- `Q:- Check if POSIX supports for those compilers?`

### High-Level Comparison (Basic Idea):
1. **Pthreads (POSIX Threads):**
   - A low-level threading library where you manually create and manage threads, handling synchronization, scheduling, and resources explicitly.
2. **OpenMP:**
   - A high-level API that uses compiler directives (`#pragma omp ...`) to automatically handle parallelism and thread management.

---

## Page 02

### Topic: Basic Pthread Creation and Joining
**Context Header:** `Q1.0` | `3. Pthreads`

### Fundamental Pthreads Steps:
1. **Create Threads**
2. **Assign Work** (start routine function)
3. **Handle Sync** (locks, mutexes, condition variables)

### Complete C Code Implementation:
```c
#include <stdio.h>
#include <pthread.h>

#define NUM_THREADS 4

// thread function
void *hello(void *arg) {
    int thread_id = *(int *)arg;
    printf("Hello world from thread %d\n", thread_id);
    pthread_exit(NULL);
}

int main() {
    pthread_t threads[NUM_THREADS];
    int thread_ids[NUM_THREADS];

    // create threads
    for (int i = 0; i < NUM_THREADS; i++) {
        thread_ids[i] = i;
        pthread_create(&threads[i], NULL, hello, &thread_ids[i]);
    }

    // wait for all threads to finish
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### Marginal Annotations:
- `pthread_t threads[NUM_THREADS];` -> `pthread_t` defines an array of POSIX thread identifiers.
- `&thread_ids[i]` is passed by reference (cast to `void *`) to provide distinct thread arguments.
- `pthread_join(threads[i], NULL);` blocks calling thread until each worker thread completes.

---

## Page 03

### Topic: The `pthread_t` Data Type & Comparison with OpenMP

### OpenMP vs. Pthreads Comparison (Left Margin):
- In OpenMP, the equivalent program is concise:
  ```c
  #pragma omp parallel
  {
      int id = omp_get_thread_num();
      printf("Hello world %d\n", id);
  }
  ```
- *Annotation:* Above example shows how simple OpenMP is; in Pthreads, the same solution requires manual thread arrays, wrapper functions, and join loops.

### Technical Concept: `pthread_t`
- **Definition:** `pthread_t` is an opaque data type used in POSIX C programs to uniquely identify an individual thread.
- **Implementation Independence:** The POSIX standard does not mandate how `pthread_t` is implemented internally.
- Depending on the operating system, it can be:
  - A simple integer or unsigned long
  - A complex internal structure/pointer
- Typical declarations in `<pthread.h>`:
  ```c
  typedef unsigned int pthread_t;
  // or
  typedef struct {
      unsigned int ...
  } pthread_t;
  ```

### Marginal Notes:
- `(NULL = attributes)` in `pthread_create`: passing `NULL` selects default attributes.
- Stored in POSIX either as a struct or a primitive datatype depending on OS implementation.

---

## Page 04

### Topic: Thread Attributes & Common Synchronization Mechanisms

### Attributes Continuation:
- **Thread Priority (in Pthreads):** Determines scheduling precedence in real-time policies.
- **Exam Question:**
  > *Exam-Q:* How to run above all (`stack size`, `sched policy`, etc.) in Pthreads? (How to define them and use them via `pthread_attr_t`?)
- Function reference: `int pthread_join(...)`
- Note: `Next -> sync mech in pthreads!`

### Common Synchronization Mechanisms in Pthreads:
1. **Mutex (Mutual Exclusion Lock):**
   - Ensures only one thread accesses a critical section at a time.
   - Used to prevent race conditions.
   - Requires explicit `lock` and `unlock` operations.
2. **Condition Variables (`pthread_cond_t`):**
   - Used for thread communication based on application conditions/state changes.
   - Allows threads to sleep/wait until signaled (`signal` or `broadcast`).
   - Always used in conjunction with a mutex.
3. **Spinlock (`pthread_spinlock_t`):**
   - Threads continuously poll/check (busy-wait) until the lock becomes available.
   - Efficient for very short lock durations (avoids context switch overhead).

---

## Page 05

### Topic: Thread Attributes (`pthread_attr_t`)

### Conceptual Details:
- `pthread_attr_t` is a POSIX thread attribute data type used to customize thread behavior before thread creation (e.g., stack size, scheduling policy, detach state).

### Key Configurable Attributes:
1. **Stack Size:**
   - Defines how much memory a thread gets for its execution stack.
2. **Scheduling Policy:**
   - Controls how the operating system schedules the thread.
   - Common policies:
     - `SCHED_OTHER`: Default time-sharing policy.
     - `SCHED_FIFO`: First-In, First-Out (real-time priority queue).
     - `SCHED_RR`: Round-Robin (real-time with time slicing).
3. **Detach State:**
   - Specifies whether a thread is joinable (`PTHREAD_CREATE_JOINABLE`, default) or detached (`PTHREAD_CREATE_DETACHED`, automatically reclaims resources on exit without `pthread_join`).

---

## Page 06

### Topic: Detailed Analysis of `pthread_create` Function Signature

### Header & Prototype:
```c
#include <pthread.h>

int pthread_create(
    pthread_t *thread,
    const pthread_attr_t *attr,
    void *(*start_routine) (void *),
    void *arg
);
```

### Parameter Breakdown & Annotations:
- `pthread_t *thread`: Pointer to a `pthread_t` variable where the system stores the unique thread ID.
- `const pthread_attr_t *attr`: Pointer to an attribute object (`NULL` for default thread attributes).
- `void *(*start_routine) (void *)`: Function pointer to the routine the thread begins executing. It must accept a single `void *` parameter and return a `void *`.
- `void *arg`: Pointer to the argument passed to `start_routine`. Passed separately as a `void *` for generic type support.

### C vs C++ Procedural Concept Note:
- Difference between struct and class: In pure C structs, we cannot define procedures/methods directly, so we use function pointers.
- To pass a function's address into generic system interfaces, we make use of function pointers.
- When a thread is created, the function pointer directs the new thread to the starting function address where work begins.

---

## Page 07

### Topic: Mutex Teardown, Argument Passing, and Condition Variables

### Mutex Code Continuation (from Page 08):
```c
    // destroy mutex
    pthread_mutex_destroy(&lock);
    printf("final counter val: %d\n", counter);
    return 0;
}
```

### Question & Concept Analysis:
- **Why do thread functions use `void *` return and `void *` parameters?**
  - **Answer:** To achieve maximum flexibility. A `void *` can point to any data type, struct, or array.
- **How to pass multiple arguments to a thread function (like `printf` does)?**
  - Package multiple values into a custom `struct` and pass the pointer to that struct as the single `void *arg`.

### Topic: Condition Variables (`pthread_cond_t`)
- **Definition:** `pthread_cond_t` is a synchronization primitive in POSIX threads used to declare a condition variable, allowing threads to suspend execution and wait for specific application-defined state conditions to become true.
- **Basic Operations:**
  - `pthread_cond_init()`: Initialize the condition variable.
  - `pthread_cond_wait()`: Atomically release mutex and block waiting for signal.
  - `pthread_cond_signal()`: Wake up at least one waiting thread.
  - `pthread_cond_destroy()`: Destroy and free condition variable resources.

---

## Page 08

### Topic: Complete Mutex Implementation for Shared Counter (`counter++`)

### Full C Code Snippet:
```c
#include <stdio.h>
#include <pthread.h>

#define NUM_THREADS 4

// shared resource
int counter = 0;

// Mutex lock
pthread_mutex_t lock;

void *increment(void *arg) {
    for (int i = 0; i < 20; i++) {
        // lock critical section
        pthread_mutex_lock(&lock);
        counter++;
        printf("Thread %ld incremented counter to %d\n", (long)arg, counter);
        // unlock critical section
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];

    // initialize mutex
    pthread_mutex_init(&lock, NULL);

    // create threads
    for (long i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment, (void *)i);
    }

    // join threads
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // (Code continues on Page 07: destroy mutex & return 0)
```

---

## Page 09

### Topic: Mutex Lifecycle & OpenMP Equivalent

### Mutex Concepts & Operations:
- A `pthread_mutex_t` is a mutual exclusion lock data type in POSIX threads used to protect shared resources from simultaneous access by multiple threads.
- **Lifecycle Functions:**
  1. `pthread_mutex_init(&lock, NULL)`: Set up and initialize the lock.
  2. `pthread_mutex_lock(&lock)`: Acquire the lock. If another thread currently holds it, the calling thread blocks until it is freed.
  3. `pthread_mutex_trylock(&lock)`: Non-blocking acquisition; checks and locks immediately, or returns `EBUSY` if already locked.
  4. `pthread_mutex_unlock(&lock)`: Releases the held lock.
  5. `pthread_mutex_destroy(&lock)`: Cleans up and destroys the mutex object.

### OpenMP Equivalent Snippet:
```c
counter = 0;
#pragma omp parallel num_threads(2)
{
    for (int i = 0; i < 5; i++) {
        #pragma omp critical
        {
            counter++;
            printf("id:%d, counter: %d\n", omp_get_thread_num(), counter);
        }
    }
}
```

---

## Page 10

### Topic: Semaphores Continuation & Read-Write Locks (`pthread_rwlock_t`)

### Semaphore Main Function Continuation (from Page 11):
```c
int main() {
    pthread_t t[N];

    // binary semaphore (pshared = 0, initial value = 1)
    sem_init(&sem, 0, 1);

    for (int i = 0; i < N; i++) {
        pthread_create(&t[i], NULL, increment, NULL);
    }
    for (int i = 0; i < N; i++) {
        pthread_join(t[i], NULL);
    }

    sem_destroy(&sem);
    return 0;
}
```
- **Exam Question:** `diff b/w bin-sem v/s mutex` (Binary Semaphore vs. Mutex: ownership semantics, priority inheritance, signaling vs. locking).

### Topic: Read-Write Locks (`pthread_rwlock_t`)
- **Core Concept:**
  - Allows multiple reader threads to access a shared resource simultaneously.
  - Allows only **1 writer** at a time. While a writer is writing, no readers or other writers are allowed.
  - **Why?** Simultaneous writes cause race conditions and data corruption.
- **Lock Modes:**
  - **Read Lock (Shared Lock):** Many reader threads can hold the lock at once.
  - **Write Lock (Exclusive Lock):** Only one writer thread can hold the lock; completely excludes all other readers and writers.
- **Real-World Analogy / Guideline:** Fixed deposit / banking records (many can view balance simultaneously; updates require exclusive access).

---

## Page 11

### Topic: Producer-Consumer (Consumer Function) & POSIX Semaphores

### Bounded Buffer: Consumer Thread Implementation:
```c
void *consumer(void *arg) {
    for (int i = 1; i <= 20; i++) {
        pthread_mutex_lock(&mutex);
        while (count == 0) {
            printf("buff empty, consumer waiting\n");
            pthread_cond_wait(&not_empty, &mutex);
        }
        int item = buffer[--count];
        printf("consumed %d\n", item);
        pthread_cond_signal(&not_full);
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

int main() {
    pthread_t p, c;
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&not_full, NULL);
    pthread_cond_init(&not_empty, NULL);
    // (create producer and consumer threads, join, and destroy)
}
```

### Topic: 3. Semaphores (`sem_t`)
- **Definition:** A Semaphore acts as an atomic counter (`sem_t`) representing available resource instances.
- **Standards Note:** Semaphores are technically **not** part of the core Pthreads library (`-lpthread`); they are defined in the POSIX Realtime Extensions library under the `<semaphore.h>` header.
- **Counting / Binary Semaphore Example:**
```c
#include <semaphore.h>

#define N 100
sem_t sem;
int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 100; i++) {
        sem_wait(&sem);
        counter++;
        sem_post(&sem);
    }
    return NULL;
}
```

---

## Page 12

### Topic: Producer-Consumer (Bounded Buffer) Problem Using Condition Variables

### Problem Formulation:
- A **Producer thread** adds items into a shared buffer.
- A **Consumer thread** removes items from the buffer.
- **Coordination Rules:**
  - If buffer is full -> Producer must wait.
  - If buffer is empty -> Consumer must wait.

### Producer Thread Implementation:
```c
#define SIZE 5

int buffer[SIZE];
int count = 0;
pthread_mutex_t mutex;
pthread_cond_t not_full, not_empty;

void *producer(void *arg) {
    for (int i = 1; i <= 20; i++) {
        pthread_mutex_lock(&mutex);
        while (count == SIZE) {
            printf("buffer full, prod waiting\n");
            pthread_cond_wait(&not_full, &mutex);
        }
        buffer[count++] = i;
        printf("produced %d\n", i);
        pthread_cond_signal(&not_empty);
        pthread_mutex_unlock(&mutex);
        sleep(3);
    }
    return NULL;
}
```

---

## Page 13

### Topic: Barrier Synchronization & Mutex vs. Spinlock Trade-Offs

### Barrier Synchronization Driver (`main`):
```c
int main() {
    pthread_t threads[NUM_THREADS];
    int thread_ids[NUM_THREADS];

    pthread_barrier_init(&barrier, NULL, NUM_THREADS);

    for (int i = 0; i < NUM_THREADS; i++) {
        thread_ids[i] = i + 1;
        pthread_create(&threads[i], NULL, worker, &thread_ids[i]);
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    pthread_barrier_destroy(&barrier);
    return 0;
}
```

### Mutex vs. Spinlock Comparison:

| Feature / Metric | Mutex (Blocking) | Spinlock (Busy-Wait) |
| :--- | :--- | :--- |
| **Good for** | Longer wait durations, lower CPU usage | Short critical sections, ultra-low latency |
| **Watch Out** | Context-switch overhead, added OS latency | High CPU consumption, thread starvation, CPU utilization spikes |
| **Under Contention** | Threads sleep; CPU freed | Burns CPU cycles spinning; severe latency degradation |

### Engineering Decision Rules:
- **Short work inside critical section** $\rightarrow$ **Spinlock**
- **Long wait time** $\rightarrow$ **Mutex**
- **High contention** $\rightarrow$ **Redesign architecture!**

---

## Page 14

### Topic: Barrier Synchronization (`pthread_barrier_t`)

### Prototype & Parameter Signatures:
```c
int pthread_barrier_init(
    pthread_barrier_t *barrier,
    const pthread_barrierattr_t *attr,
    unsigned count
);
```
- `pthread_barrier_t *barrier`: Address of the barrier variable being initialized.
- `const pthread_barrierattr_t *attr`: Attributes pointer (`NULL` for defaults).
- `unsigned count`: Number of threads that must call `pthread_barrier_wait()` before any thread can proceed.

### Worker Function Implementation:
```c
void *worker(void *arg) {
    int id = *(int *)arg;
    printf("%d: before barrier\n", id);
    sleep(rand() % 3);
    printf("%d: waiting @ barrier\n", id);
    pthread_barrier_wait(&barrier);
    printf("%d: passed barrier!\n", id);
    return NULL;
}
```

---

## Page 15

### Topic: Complete Read-Write Lock Implementation (`pthread_rwlock_t`)

### Complete C Program:
```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>

pthread_rwlock_t rwlock;
int shared_data = 0;

void *reader(void *arg) {
    int id = *(int *)arg;
    pthread_rwlock_rdlock(&rwlock); // Shared read lock
    printf("Reader %d read: %d\n", id, shared_data);
    sleep(1);
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}

void *writer(void *arg) {
    int id = *(int *)arg;
    pthread_rwlock_wrlock(&rwlock); // Exclusive write lock
    shared_data++;
    printf("Writer %d wrote: %d\n", id, shared_data);
    sleep(2);
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}

int main() {
    pthread_t r[3], w[2];
    int ids[5] = {1, 2, 3, 4, 5};

    pthread_rwlock_init(&rwlock, NULL);

    pthread_create(&r[0], NULL, reader, &ids[0]);
    pthread_create(&r[1], NULL, reader, &ids[1]);
    pthread_create(&w[0], NULL, writer, &ids[2]);
    pthread_create(&w[1], NULL, writer, &ids[3]);
    pthread_create(&r[2], NULL, reader, &ids[4]);

    for (int i = 0; i < 3; i++) pthread_join(r[i], NULL);
    for (int i = 0; i < 2; i++) pthread_join(w[i], NULL);

    pthread_rwlock_destroy(&rwlock);
    return 0;
}
```

---

## Page 16

### Topic: Introduction to MPI Lifecycle (`Prg-1`)

### Basic MPI Program Skeleton:
```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    int rank, size;

    // 1. Start MPI (Mandatory)
    MPI_Init(&argc, &argv);

    // 2. Get process ID / rank
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    // 3. Get number of processes
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // MPI communication / work here

    // 4. Shut down MPI (Mandatory)
    MPI_Finalize();

    return 0;
}
```

### Critical MPI Architecture & Semantic Rules:
- **Process Hierarchy:** By default, all ranks (processes) are equal peers; there is no implicit parent-child relationship. However, user code can establish master-worker models programmatically.
- **Analysis of `int MPI_Init(int *argc, char ***argv)`:**
  - `argv` is a pointer to pointer to string (`char ***`).
  - Sets up internal communication buffers, global variables, and creates `MPI_COMM_WORLD`.
  - Parses and strips command-line MPI-specific flags.
  - **Constraint:** May only be called **once** per program execution.
  - **Constraint:** Must be called by the main thread, which must also invoke `MPI_Finalize()`.
- **Question:**
  > *Q:* What is the way to communicate between processes without message passing, and what problems are caused? (Shared memory IPC, race conditions, cache coherence issues).

---

## Page 17

### Topic: Distributed Memory Architecture & MPI Ecosystem

### Spinlock Program Completion (Top Left):
```c
int main() {
    pthread_t t1, t2;
    pthread_spin_init(&lock, PTHREAD_PROCESS_PRIVATE);
    // ...
}
```

### Topic: MPI (Message Passing Interface)
- **Definition:** Standard specification for programming multi-process applications on distributed memory architectures.

### Distributed Memory Architecture Diagram:
```
+-----------------------------------+        +-----------------------------------+
|              Node 1               |        |              Node 2               |
|  +--------+           +--------+  |        |  +--------+           +--------+  |
|  | Core 1 |           | Core 2 |  |        |  | Core 1 |           | Core 2 |  |
|  +---+----+           +----+---+  |        |  +---+----+           +----+---+  |
|      |       +-----+       |      | <----> |      |       +-----+       |      |
|      +-----> | RAM | <-----+      |        |      +-----> | RAM | <-----+      |
|      |       +-----+       |      |        |      |       +-----+       |      |
|  +---+----+           +----+---+  |        |  +---+----+           +----+---+  |
|  | Core 3 |           | Core 4 |  |        |  | Core 3 |           | Core 4 |  |
|  +--------+           +--------+  |        |  +--------+           +--------+  |
+-----------------------------------+        +-----------------------------------+
                  ^                                            ^
                  |                                            |
                  v                                            v
               [Node 3] <---------------------------------> [Node 4]
```
- **Fundamental Rule:** Since there is **no common shared physical address space**, all inter-node communication must occur explicitly via network messages!

### Major MPI Implementations:
1. **OpenMPI:** Dominant open-source implementation for Linux/Unix and HPC clusters.
2. **MPICH:** Highly portable implementation for Linux, Unix, and Windows.
3. **Intel MPI:** Optimized for Intel architecture within the HPC ecosystem.
4. **NVIDIA HPC-X:** Optimized for NVIDIA GPUs and InfiniBand networking.
5. **Microsoft MPI (MS-MPI):** Tailored for Windows Server/HPC environments.

---

## Page 18

### Topic: Spinlock vs. Mutex Performance Cliff & Spinlock API

### Performance Curve Analysis:
```
Throughput
(M ops/sec) ^
            |      \  <- Spinlock Performance Cliff!
            |       \
            |  Spinlock \
            |   Advantage\
            |    Region   \
            |              \___________ Mutex Advantage Region
            |               -----------
            |              / Mutex Curve (steady, low contention impact)
            |             /
            +-------------------------------------------->
            0             N (Core / Thread Contention Threshold)
```
- **Graph Description:**
  - At low thread counts, **Spinlocks** achieve significantly higher operations per second because avoiding context switching eliminates OS scheduling overhead.
  - When the number of threads exceeds available CPU cores or contention escalates, Spinlocks experience a dramatic **Performance Cliff** due to lock cache-line bouncing and wasted CPU spinning.
  - **Mutexes** exhibit lower throughput at single-thread loads but remain stable under heavy contention by descheduling waiting threads.

### Assignment / Study Item:
- `Assignment: Study how EPFS works`

### Spinlock API & Routine:
- Prototype: `int pthread_spin_init(pthread_spinlock_t *lock, int pshared);`
```c
void *increment(void *arg) {
    for (int i = 0; i < 1e5; i++) {
        pthread_spin_lock(&lock);
        counter++;
        pthread_spin_unlock(&lock);
    }
    return NULL;
}
```

---

## Page 19

### Topic: MPI Point-to-Point Communication Implementation

### Complete Point-to-Point Program:
```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    int rank, data;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    if (rank == 0) {
        data = 50;
        // MPI_Send(&buf, count, datatype, dest, tag, comm)
        MPI_Send(&data, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);
        printf("sent: %d\n", data);
        // sleep(60);
    }
    else if (rank == 1) {
        // MPI_Recv(&buf, count, datatype, source, tag, comm, &status)
        MPI_Recv(&data, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("received: %d\n", data);
        // sleep(60);
    }

    MPI_Finalize();
    return 0;
}
```

### Sending Arrays:
```c
if (rank == 0) {
    int data[5] = {10, 20, 30, 40, 50};
    MPI_Send(data, 5, MPI_INT, 1, 0, MPI_COMM_WORLD);
}
```

### Study Note:
- `(Marshalling & de-marshalling -> study!)` — Serializing non-contiguous data types and structs for network transport.

---

## Page 20

### Topic: Collective vs. Point-to-Point Communication in MPI

### 1. Collective Communication:
- Involves **all processes** within a specified communicator (such as `MPI_COMM_WORLD`).
- Common primitives: `MPI_Bcast`, `MPI_Reduce`, `MPI_Scatter`, `MPI_Gather`, `MPI_Barrier`.

### 2. Point-to-Point Communication:
- Communication between exactly two processes: a designated sender and a designated receiver.
- Used when one specific process sends data/results to another in an algorithmic pipeline.

### Interaction Flow Diagram:
```
+------------------------+                     +------------------------+
|       Process 0        |                     |       Process 1        |
|        (Sender)        |                     |       (Receiver)       |
+------------------------+                     +------------------------+
| data = 50;             |                     | int data;              |
| MPI_Send(&data, 1,     | ------------------> | MPI_Recv(&data, 1,     |
|   MPI_INT, 1, 0, comm) |      Integer 50     |   MPI_INT, 0, 0, comm, |
|                        |                     |   MPI_STATUS_IGNORE);  |
+------------------------+                     +------------------------+
```

---

## Page 21

### Topic: Compiling, Running, and Inspecting MPI Processes

### Commands & Tooling:
- **Compile:** `mpicc program.c -o a.out`
- **Run:** `mpirun -np 4 ./a.out` (spawns 4 processes)

### Diagnostic Code Snippet:
```c
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    int rank, size;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    printf("Hello from proc %d out of %d, pid=%d\n", rank, size, getpid());
    fflush(stdout);

    // sleep(60); // Keeps processes alive to observe in process table
    MPI_Finalize();
    return 0;
}
```

### Advanced `mpirun` Flags:
1. `mpirun --display-allocation -np 10 ./a.out`
   - Displays the detected hardware slot allocation and core mapping.
2. `mpirun --oversubscribe -np 20 ./a.out`
   - Allows spawning more processes than physical CPU slots/cores available.
3. `mpirun --hostfile <hostfile>`
   - Distributes process ranks across nodes defined in a cluster hostfile.
4. `ps aux | grep a.out`:
   - Used during `sleep(60)` to verify that distinct operating system processes with separate PIDs are executing.

---

## Page 22

### Topic: MPI Dynamic Inter-Communicator Connection: `sender.c`

### Complete Code Implementation:
```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    MPI_Comm client;
    MPI_Status status;
    int data = 100;
    char port[MPI_MAX_PORT_NAME];

    MPI_Init(&argc, &argv);

    // 1. Open a port for communication
    MPI_Open_port(MPI_INFO_NULL, port);
    printf("port: %s\n", port);
    printf("waiting for receiver...\n");
    fflush(stdout);

    // 2. Save port name for receiver to locate
    FILE *fp = fopen("port.txt", "w");
    fprintf(fp, "%s\n", port);
    fclose(fp);

    // 3. Wait for receiver to connect
    MPI_Comm_accept(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &client);

    // 4. Send data to connected client
    MPI_Send(&data, 1, MPI_INT, 0, 0, client);
    printf("only one receiver accepts sender!\n");

    // 5. Clean up and disconnect
    MPI_Comm_disconnect(&client);
    MPI_Close_port(port);
    MPI_Finalize();

    return 0;
}
```

### Build Command:
```bash
mpicc sender.c -o sender
```

---

## Page 23

### Topic: MPI Dynamic Process Architecture & Port Discovery

### Conceptual Questions & Answers:
- **Question:** `Q: How does receiver know port.txt?`
  - The port name string generated by `MPI_Open_port` is published via an external medium (e.g., written to a shared filesystem file `port.txt`, or published to an MPI lookup service daemon like `ompi-server`).

### Dynamic Connection Workflow:
```
+--------------------------+                         +--------------------------+
|      Sender Process      |                         |     Receiver Process     |
+--------------------------+                         +--------------------------+
| 1. MPI_Open_port()       |                         |                          |
| 2. Write to port.txt     | ----[ port.txt ]------> | 1. Read port.txt         |
| 3. MPI_Comm_accept()     | <=== [Handshake] ====>  | 2. MPI_Comm_connect()    |
| 4. MPI_Send(..., client) | ----------------------> | 3. MPI_Recv(..., server) |
| 5. MPI_Comm_disconnect() |                         | 4. MPI_Comm_disconnect() |
| 6. MPI_Close_port()      |                         |                          |
+--------------------------+                         +--------------------------+
```

### Key MPI Dynamic Process Functions:
- `MPI_Open_port()`
- `MPI_Comm_accept()`
- `MPI_Comm_connect()`
- `MPI_Send()` / `MPI_Recv()`
- `MPI_Comm_disconnect()`

---

## Page 24

### Topic: Step-by-Step Workflow of the MPI Sender

### Detailed Protocol Steps:
1. **Initialize MPI:**
   - Call `MPI_Init(&argc, &argv)`.
2. **Open a Communication Port:**
   - Call `MPI_Open_port(MPI_INFO_NULL, port)`. Allocates a network port managed by the MPI runtime.
3. **Save/Publish Port Name:**
   - Save the generated port string to `port.txt`.
   - *Rationale:* When two independent MPI programs are launched via separate `mpirun` invocations, they do not share `MPI_COMM_WORLD`. To communicate, both must rendezvous using this port identifier.
4. **Wait for Receiver Connection:**
   - Call `MPI_Comm_accept(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &client)`. Blocks until an incoming connection is accepted.
5. **Send Data:**
   - Call `MPI_Send(&data, 1, MPI_INT, 0, 0, client)`. Transfers data (`data = 100`) across the newly formed inter-communicator.
6. **Disconnect and Finalize:**
   - `MPI_Comm_disconnect(&client) -> MPI_Close_port(port) -> MPI_Finalize()`.

---

## Page 25

### Topic: Executing Dynamic Processes with `ompi-server`

### Commands and Execution Flow:
1. **Running the Sender with PMIX URI:**
   ```bash
   mpirun --mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./sender
   ```
   - Terminal Output:
     ```text
     port: 365795337...:0: 3530881966
     waiting for receiver...
     Sender sent 100
     ```
2. **Running the Receiver:**
   ```bash
   mpirun --mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./receiver
   ```

### Asynchronous Behavior Question:
- **Question:** `Q: What if sender is fast and receiver is slow?`
  - In `receiver.c`:
    ```c
    while (1) {
        MPI_Recv(...);
        sleep(50);
    }
    ```
  - Standard `MPI_Send` will either buffer data (if below eager threshold) or block until receiver posts `MPI_Recv`, preventing data loss.

---

## Page 26

### Topic: MPI Lookup Service Daemon (`ompi-server`)

### Conceptual Explanation:
- Dynamic connections between independent programs require a lookup service (analogous to DNS resolving domain names to IP addresses).
- The sender generates a unique port string that the receiver must locate to establish a communication channel.
- **Why do we use `ompi-server`?**
  - `ompi-server` acts as a centralized rendezvous service where independent MPI applications can publish (`MPI_Publish_name`) and look up (`MPI_Lookup_name`) service ports, enabling dynamic inter-communicator connection across distinct job runs.

### Command Syntax & Flag Explanation:
```bash
ompi-server --no-daemonize -report-uri /tmp/ompi-server.uri
```
- `--no-daemonize`: Keeps `ompi-server` running in the foreground of the terminal session.
- `-report-uri /tmp/ompi-server.uri`: Directs the daemon to output its Uniform Resource Identifier (URI) to the specified file so subsequent `mpirun` processes can attach to it.

---

## Page 27

### Topic: MPI Dynamic Inter-Communicator Connection: `receiver.c`

### Complete Code Implementation:
```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    MPI_Comm server;
    MPI_Status status;
    int data;
    char port[MPI_MAX_PORT_NAME];

    MPI_Init(&argc, &argv);

    // 1. Read sender's port name from file
    FILE *fp = fopen("port.txt", "r");
    if (fp == NULL) {
        printf("run sender first!\n");
        MPI_Abort(MPI_COMM_WORLD, 1);
    }
    fscanf(fp, "%s", port);
    fclose(fp);

    // 2. Connect to the sender's port
    MPI_Comm_connect(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &server);

    // 3. Receive the data sent by sender
    MPI_Recv(&data, 1, MPI_INT, 0, 0, server, &status);
    printf("Receiver received: %d\n", data);

    // 4. Disconnect and finalize
    MPI_Comm_disconnect(&server);
    MPI_Finalize();

    return 0;
}
```

### Build & Run Commands:
```bash
mpicc receiver.c -o receiver
mpirun -np 1 ./receiver
```

# Source Extraction: Pthreada_Aagam_Notes.pdf

**Source File:** `input_notes/Pthreada_Aagam_Notes.pdf`  
**Total Pages:** 20  
**Topics:** Pthreads (POSIX Threads) complete lifecycle, thread attributes (`pthread_attr_t`), scheduling policies, thread creation and joining, synchronization primitives (Mutex, Condition Variables, Semaphores, Read-Write Locks, Barriers, Spinlocks), Producer-Consumer Bounded Buffer implementation, Mutex vs Spinlock performance trade-offs under contention.

---

## Page 01

### Topic: Introduction to Pthreads & POSIX Standards
- **Header:** PART-2: Pthreads (POSIX Threads)
- **POSIX:** Portable Operating System Interface
  - Developed by the IEEE Computer Society.
  - A family of standards designed to ensure compatibility and software portability between different operating systems.
- **System Compliance:**
  - **Officially Certified Systems:** macOS
  - **Mostly Compliant (Uncertified) Systems:**
    - Linux distributions (Ubuntu, Fedora, Debian, Red Hat, etc. implement POSIX via the GNU C Library / glibc & system tools).
    - BSD operating systems (FreeBSD, OpenBSD, NetBSD).
    - Android (uses Linux kernel with modified C library / Bionic).

---

## Page 02

### Topic: Basic Idea — Pthreads vs OpenMP
- **Pthreads (POSIX Threads):**
  - A low-level threading library where you manually create and manage threads.
  - Programmer explicitly:
    - Creates threads
    - Assigns work to threads
    - Handles synchronization (locks, mutexes, condition variables)
- **OpenMP:**
  - A high-level API that uses compiler directives (`#pragma`) to automatically handle parallelism.
- **Pthreads "Hello World" Setup:**
  - Goal: Run 4 Hello World print statements using Pthreads.
  - Function prototype: `void* hello(void* arg);`
  - Array declarations:
    - `pthread_t threads[NUM_THREADS];`
    - `int thread_ids[NUM_THREADS];`

---

## Page 03

### Topic: Complete Pthreads Hello World Program
- **Thread Creation & Joining Loop:**
  ```c
  void* hello(void *arg) {
      int thread_id = *(int*)arg;
      printf("Hello World from thread %d\n", thread_id);
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
- **Observations:**
  - Function pointer: takes `void*`, returns `void*`.
  - Passing address of `thread_ids[i]` to avoid race conditions.

---

## Page 04

### Topic: `pthread_t` Data Type & `pthread_create` API Specification
- **`pthread_t` Data Type:**
  - An opaque data type used in POSIX C programs to uniquely identify an individual thread.
  - The POSIX standard does not mandate how `pthread_t` is implemented:
    - Could be a simple integer (typedef unsigned long).
    - Could be a complex internal structure depending on OS kernel/library implementation.
- **Marginal C Language Note:**
  - "Can't define procedures inside struct. In order to mimic classes in C use function pointers."
- **`pthread_create` Prototype:**
  ```c
  #include <pthread.h>

  int pthread_create(
      pthread_t *thread,
      const pthread_attr_t *attr,
      void *(*start_routine)(void *),
      void *arg
  );
  ```

---

## Page 05

### Topic: `pthread_attr_t` Thread Attributes & Scheduling Policies
- **`pthread_attr_t` Definition:**
  - A POSIX thread attributes data type used to customize thread behavior before creation.
  - Marginal Exam Alert: *"He will ask all this!"*
- **Customizable Attributes:**
  1. **Stack Size:** Configuring custom memory size allocated to thread stack.
  2. **Scheduling Policy:**
     - `SCHED_OTHER`: Default time-sharing round-robin policy.
     - `SCHED_FIFO`: First-In, First-Out real-time priority policy.
     - `SCHED_RR`: Round-robin real-time priority policy.
  3. **Detach State:** Joinable (default) vs Detached (`PTHREAD_CREATE_DETACHED`).
  4. **Thread Priority:** Real-time priority levels in Pthreads (`sched_param`).

---

## Page 06

### Topic: Overview of Pthreads Synchronization Mechanisms
- **`pthread_join()`:** Blocks the calling thread until the specified thread terminates.
- **Common Synchronization Mechanisms in Pthreads:**
  1. **Mutex (Mutual Exclusion Lock):**
     - Ensures only one thread accesses a critical section at a time.
     - Used to prevent race conditions on shared data.
     - Requires explicit lock and unlock operations.
  2. **Condition Variables (`pthread_cond_t`):**
     - Used for thread communication based on state/conditions.
     - Allows threads to suspend execution (wait) until notified (signal or broadcast).
     - Always paired with a mutex.
  3. **Semaphores (`sem_t`):**
     - Atomic counters managing finite pool of resource instances.

---

## Page 07

### Topic: RW Locks, Barriers, Spinlocks & `pthread_mutex_t` API
- **Synchronization Mechanisms (Cont.):**
  4. **Read-Write Locks (`pthread_rwlock_t`):** Allows multiple readers concurrently or a single exclusive writer.
  5. **Barriers (`pthread_barrier_t`):** Forces all threads to reach a common synchronization point before proceeding.
  6. **Spinlocks (`pthread_spinlock_t`):** Busy-waiting lock for low-latency, short critical sections.
- **Mutex API (`pthread_mutex_t`):**
  - Data type used to protect shared resources.
  - Key Lifecycle Functions:
    - `pthread_mutex_init(&lock, attr)`: Initialize mutex.
    - `pthread_mutex_lock(&lock)`: Acquire lock (blocking).
    - `pthread_mutex_trylock(&lock)`: Non-blocking attempt.
    - `pthread_mutex_unlock(&lock)`: Release lock.
    - `pthread_mutex_destroy(&lock)`: Destroy mutex.

---

## Page 08

### Topic: Mutex Critical Section Code Example (Part 1)
- **Program Configuration:**
  - `#define NUM_THREADS 1000`
  - Shared variable: `int counter = 0;`
  - Mutex declaration: `pthread_mutex_t lock;`
- **Worker Function:**
  ```c
  void* increment(void* arg) {
      for (int i = 0; i < 20; i++) {
          // Lock the critical section
          pthread_mutex_lock(&lock);
          counter++;
          printf("thread %ld incremented counter to %d\n", (long)arg, counter);
          // Unlock the critical section
          pthread_mutex_unlock(&lock);
      }
      return NULL;
  }
  ```

---

## Page 09

### Topic: Mutex Critical Section Code Example (Part 2 - `main`)
- **`main()` Implementation:**
  ```c
  int main() {
      pthread_t threads[NUM_THREADS];

      // Initialize mutex
      pthread_mutex_init(&lock, NULL);

      // Create threads
      for (long i = 0; i < NUM_THREADS; i++) {
          pthread_create(&threads[i], NULL, increment, (void*)i);
      }
      // Note: why use void*? Flexibility!

      // Join threads
      for (int i = 0; i < NUM_THREADS; i++) {
          pthread_join(threads[i], NULL);
      }

      // Destroy mutex
      pthread_mutex_destroy(&lock);

      printf("Final counter value: %d\n", counter);
      return 0;
  }
  ```

---

## Page 10

### Topic: Condition Variables (`pthread_cond_t`) & Producer-Consumer Problem
- **`pthread_cond_t` Definition:**
  - A data type in POSIX threads used to declare condition variables, allowing threads to suspend execution and wait for specific application-defined conditions to become true.
- **Operations:**
  - `pthread_cond_init(&cond, attr)`
  - `pthread_cond_wait(&cond, &mutex)`: Atomically releases mutex and puts thread to sleep; reacquires mutex upon waking.
  - `pthread_cond_signal(&cond)`: Wakes up at least one waiting thread.
  - `pthread_cond_broadcast(&cond)`: Wakes up all waiting threads.
  - `pthread_cond_destroy(&cond)`
- **Producer-Consumer (Bounded Buffer) Concept:**
  - Producer thread adds items to buffer.
  - Consumer thread removes items.
  - Rules:
    - If buffer is full $\rightarrow$ producer waits.
    - If buffer is empty $\rightarrow$ consumer waits.
  - Diagram: `[Producer] -> [ Bounded Buffer / Queue ] -> [Consumer]`

---

## Page 11

### Topic: Producer-Consumer Implementation in Pthreads (Producer Code)
- **Global Shared Declarations:**
  ```c
  int buffer[SIZE];
  int count = 0;
  pthread_mutex_t mutex;
  pthread_cond_t not_full, not_empty;
  ```
- **Producer Function:**
  ```c
  void* producer(void *arg) {
      for (int i = 1; i <= 20; i++) {
          pthread_mutex_lock(&mutex);
          while (count == SIZE) {
              printf("buffer full. Producer waiting...\n");
              pthread_cond_wait(&not_full, &mutex);
          }
          buffer[count++] = i;
          printf("Produced: %d\n", i);
          pthread_cond_signal(&not_empty);
          pthread_mutex_unlock(&mutex);
      }
      return NULL;
  }
  ```

---

## Page 12

### Topic: Producer-Consumer Implementation in Pthreads (Consumer Code)
- **Consumer Function:**
  ```c
  void* consumer(void *arg) {
      for (int i = 1; i <= 20; i++) {
          pthread_mutex_lock(&mutex);
          while (count == 0) {
              printf("buffer empty, Consumer waiting...\n");
              pthread_cond_wait(&not_empty, &mutex);
          }
          int item = buffer[--count];
          printf("Consumed: %d\n", item);
          pthread_cond_signal(&not_full);
          pthread_mutex_unlock(&mutex);
          sleep(1);
      }
      return NULL;
  }
  ```

---

## Page 13

### Topic: POSIX Semaphores (`<semaphore.h>`)
- **Semaphore Concept:**
  - A semaphore acts as an atomic counter (`sem_t`) representing available resource instances.
  - **Important Technical Distinction:**
    - "Semaphores are not technically part of the Pthreads library itself; instead, they are defined in the POSIX Real-time Extensions library under `<semaphore.h>`."
- **Program Example:**
  - `#define N 100`
  - `sem_t sem;`
  - `int counter = 0;`
  - Worker function:
    ```c
    void* increment(void* arg) {
        for (int i = 0; i < 100000; i++) {
            sem_wait(&sem); // enter critical section (decrement / wait)
            counter++;
            sem_post(&sem); // signal / increment
        }
        return NULL;
    }
    ```

---

## Page 14

### Topic: Semaphore Setup & Read-Write Locks Introduction
- **Semaphore Initialization:**
  - `sem_init(&sem, 0, 1);`
    - Parameter 1: `&sem`
    - Parameter 2: `0` (thread-shared within process; `1` would mean process-shared)
    - Parameter 3: `1` (initial value; `1` acts as a binary semaphore / mutex)
  - `sem_destroy(&sem);`
- **Read-Write (RW) Locks Concept:**
  - "They allow multiple readers to access the resource simultaneously, but only one writer at a time — and while writing, no readers are allowed."

---

## Page 15

### Topic: RW Locks Intuition & Architecture (`pthread_rwlock_t`)
- **Intuitive Analogy:**
  - Blackboard analogy: *"RBI guidelines: Jaankaar Rahiye Satark Rahiye"*
  - Any number of readers can view the public board simultaneously.
  - But when the writer modifies the board, everyone else must wait until the write is complete.
- **Core Rules:**
  - **Read Lock (Shared Lock):** Many threads can read concurrently.
  - **Write Lock (Exclusive Lock):** Only one thread can write; blocks all readers and other writers.
- **API Setup:**
  - `pthread_rwlock_t rwlock;`
  - `pthread_rwlock_init(&rwlock, NULL);`
  - Reader threads `r[3]`, Writer threads `w[2]`.

---

## Page 16

### Topic: RW Lock Code Implementation (`writer` & `reader`)
- **Writer Function:**
  ```c
  void* writer(void* arg) {
      int id = *(int*)arg;
      pthread_rwlock_wrlock(&rwlock);
      shared_data++;
      printf("Writer %d updated value to %d\n", id, shared_data);
      sleep(2);
      pthread_rwlock_unlock(&rwlock);
      return NULL;
  }
  ```
- **Reader Function:**
  ```c
  void* reader(void* arg) {
      pthread_rwlock_rdlock(&rwlock);
      printf("Reader reading: %d\n", shared_data);
      sleep(1);
      pthread_rwlock_unlock(&rwlock);
      return NULL;
  }
  ```

---

## Page 17

### Topic: POSIX Barriers (`pthread_barrier_t`)
- **Top Note:** "without pthread_join()?"
- **API Specification:**
  ```c
  int pthread_barrier_init(
      pthread_barrier_t *barrier,
      const pthread_barrierattr_t *attr,
      unsigned count
  );
  ```
- **Parameters:**
  - `barrier`: Address of barrier variable being initialized.
  - `attr`: Barrier attributes (`NULL` for default).
  - `count`: Number of threads that must call `pthread_barrier_wait()` before any thread is allowed to continue.
- **Usage Example:**
  - `pthread_barrier_init(&barrier, NULL, NUM_THREADS);`

---

## Page 18

### Topic: Barriers Synchronization & Mutex vs Spinlock In-Depth Comparison
- **Barrier behavior:** "Barrier keeps counting how many threads reach it."
- **Comparison Table: Mutex vs Spinlock**
  | Feature | Mutex (Blocking Lock) | Spinlock (Busy-Wait Lock) |
  | :--- | :--- | :--- |
  | **Wait Behavior** | Thread goes to sleep (blocks); OS yields CPU | Thread spins in a loop checking lock condition (`while(lock)`) |
  | **Best Used For** | Longer wait times, long critical sections | Extremely short critical sections ($< \text{context switch time}$) |
  | **Latency** | Higher latency due to OS context switch | Extremely low latency (immediate acquisition upon release) |
  | **CPU Usage** | Zero CPU consumption while sleeping | High CPU consumption (burns CPU cycles while spinning) |
  | **Watch Out For**| Context switch overhead | Thread starvation, CPU burning, priority inversion |
- **Under Contention:**
  - High contention with spinlocks $\rightarrow$ massive CPU spikes!
  - High contention with mutexes $\rightarrow$ higher latency from context switching.
  - **Decision Rule:**
    - Short work / short critical section $\rightarrow$ Spinlock
    - Long wait / long critical section $\rightarrow$ Mutex

---

## Page 19

### Topic: Performance vs Critical Section Length Curve (Spinlock vs Mutex)
- **Empirical Graph Analysis:**
  - **Y-axis:** Performance (million operations / sec) [0 to 17.5]
  - **X-axis:** Critical section length in CPU cycles ($10^0, 10^1, 10^2, 10^3$)
  - **Curve Observations:**
    - At short critical sections ($10^0$ to $10^2$ cycles), **Spinlock outperforms Mutex** (achieving ~13.5M ops/sec vs ~8M ops/sec).
    - At around $10^2$ cycles, the spinlock curve exhibits a steep catastrophic drop down to ~2.5M ops/sec!
    - At long critical sections ($> 10^2$ cycles), **Mutex outperforms Spinlock** because sleeping threads do not waste CPU cycles.
  - **Observation Note:** "High contention $\rightarrow$ Redesign".
  - **Spinlock Initialization:**
    ```c
    int pthread_spin_init(pthread_spinlock_t *lock, int pshared);
    ```

---

## Page 20

### Topic: Spinlock Program Structure & Pthread Types Summary
- **Spinlock API Usage:**
  - `pthread_spinlock_t lock;`
  - `pthread_spin_init(&lock, PTHREAD_PROCESS_PRIVATE);`
  - `pthread_spin_lock(&lock);`
  - `counter++;`
  - `pthread_spin_unlock(&lock);`
  - `pthread_spin_destroy(&lock);`
- **Summary of Pthread Types:**
  - `pthread_t`: Thread handle
  - `pthread_attr_t`: Thread creation attributes
  - `pthread_mutex_t`: Mutual exclusion lock
  - `pthread_cond_t`: Condition variable
  - `pthread_rwlock_t`: Read-write lock
  - `pthread_barrier_t`: Phase barrier
  - `pthread_spinlock_t`: Busy-waiting spinlock
  - `sem_t`: POSIX counting semaphore (from `<semaphore.h>`)

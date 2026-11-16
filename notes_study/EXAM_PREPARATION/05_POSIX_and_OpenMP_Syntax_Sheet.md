# POSIX (Pthreads & Semaphores) & OpenMP Quick-Reference Syntax Sheet

This high-yield syntax sheet provides exact function signatures, pragma formats, clauses, runtime library APIs, and compilation commands for rapid recall in exams and programming assignments.

---

# PART 1: POSIX THREADS & SYNCHRONIZATION

## 1. Compilation & Header Files
```c
#include <pthread.h>    // Core Pthreads (threads, mutexes, condvars, rwlocks, barriers, spinlocks)
#include <semaphore.h>  // POSIX Realtime Extensions (semaphores)
#include <unistd.h>     // sleep(), usleep(), getpid()
```
```bash
# Compilation flags: Link with pthread library
gcc -Wall -O2 program.c -o program -pthread
# Or older syntax:
gcc -Wall -O2 program.c -o program -lpthread
```

---

## 2. Thread Lifecycle Management

| Function Signature | Description | Return Value |
| :--- | :--- | :--- |
| `int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);` | Spawns a new thread executing `start_routine(arg)`. | `0` on success, error number on failure |
| `int pthread_join(pthread_t thread, void **retval);` | Blocks caller until `thread` terminates; captures return value into `*retval`. | `0` on success, error number on failure |
| `void pthread_exit(void *retval);` | Terminates calling thread; yields `retval` to `pthread_join`. | Does not return |
| `int pthread_detach(pthread_t thread);` | Marks thread as detached; its resources are auto-freed on exit (cannot be joined). | `0` on success, error number on failure |
| `pthread_t pthread_self(void);` | Returns thread ID of calling thread. | Thread ID (`pthread_t`) |
| `int pthread_equal(pthread_t t1, pthread_t t2);` | Compares two thread IDs for equality. | Non-zero if equal, `0` otherwise |

### Typical Pattern:
```c
void *worker(void *arg) {
    int id = *(int *)arg;
    int *result = malloc(sizeof(int));
    *result = id * 10;
    pthread_exit((void *)result); // Or return (void *)result;
}

int main() {
    pthread_t tid;
    int id = 1;
    pthread_create(&tid, NULL, worker, &id);
    void *res_ptr;
    pthread_join(tid, &res_ptr);
    printf("Result: %d\n", *(int *)res_ptr);
    free(res_ptr);
}
```

---

## 3. Thread Attributes (`pthread_attr_t`)

```c
pthread_attr_t attr;
pthread_attr_init(&attr);                              // Initialize to default attributes
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE); // Joinable (default)
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED); // Detached
pthread_attr_setstacksize(&attr, 4 * 1024 * 1024);     // Set 4 MB stack size
pthread_create(&tid, &attr, worker, NULL);             // Create with attributes
pthread_attr_destroy(&attr);                           // Destroy attribute object
```

---

## 4. Mutex (`pthread_mutex_t`) — Mutual Exclusion

### Initialization:
- **Static:** `pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;`
- **Dynamic:**
  ```c
  pthread_mutex_t lock;
  pthread_mutex_init(&lock, NULL); // NULL for default attributes
  ```

### Lock / Unlock APIs:
| Function Signature | Description |
| :--- | :--- |
| `int pthread_mutex_lock(pthread_mutex_t *mutex);` | Blocks until lock is acquired. |
| `int pthread_mutex_trylock(pthread_mutex_t *mutex);` | Non-blocking; returns `0` if lock acquired, `EBUSY` if already locked. |
| `int pthread_mutex_unlock(pthread_mutex_t *mutex);` | Releases lock (only holding thread should call). |
| `int pthread_mutex_destroy(pthread_mutex_t *mutex);` | Deallocates mutex resources. |

---

## 5. Condition Variables (`pthread_cond_t`)

### Initialization:
- **Static:** `pthread_cond_t cond = PTHREAD_COND_INITIALIZER;`
- **Dynamic:** `pthread_cond_init(&cond, NULL);`

### Operation APIs:
| Function Signature | Description |
| :--- | :--- |
| `int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);` | Atomically releases `mutex` and blocks on `cond`. Reacquires `mutex` before returning. |
| `int pthread_cond_signal(pthread_cond_t *cond);` | Unblocks at least one thread waiting on `cond`. |
| `int pthread_cond_broadcast(pthread_cond_t *cond);` | Unblocks all threads currently waiting on `cond`. |
| `int pthread_cond_destroy(pthread_cond_t *cond);` | Destroys condition variable. |

### The Golden While-Loop Pattern:
```c
pthread_mutex_lock(&mutex);
while (condition_is_false) { // MUST use 'while' to prevent spurious wakeups!
    pthread_cond_wait(&cond, &mutex);
}
// Perform work on shared state
pthread_mutex_unlock(&mutex);
```

---

## 6. POSIX Semaphores (`<semaphore.h>`)

| Function Signature | Description |
| :--- | :--- |
| `int sem_init(sem_t *sem, int pshared, unsigned int value);` | Initializes semaphore counter to `value`. `pshared = 0` (threads of process); `pshared = 1` (shared across processes). |
| `int sem_wait(sem_t *sem);` | Decrements counter; blocks if counter == 0 until counter > 0. |
| `int sem_trywait(sem_t *sem);` | Non-blocking decrement; returns `EAGAIN` if counter == 0. |
| `int sem_post(sem_t *sem);` | Increments counter; unblocks a waiting thread. |
| `int sem_getvalue(sem_t *sem, int *sval);` | Reads current value into `*sval`. |
| `int sem_destroy(sem_t *sem);` | Destroys semaphore. |

---

## 7. Read-Write Locks (`pthread_rwlock_t`)

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
// Or dynamic: pthread_rwlock_init(&rwlock, NULL);

pthread_rwlock_rdlock(&rwlock);   // Multiple readers acquire simultaneously
// ... read shared data ...
pthread_rwlock_unlock(&rwlock);   // Release read lock

pthread_rwlock_wrlock(&rwlock);   // Exclusive write lock; blocks all readers & writers
// ... update shared data ...
pthread_rwlock_unlock(&rwlock);   // Release write lock

pthread_rwlock_destroy(&rwlock);
```

---

## 8. Barriers (`pthread_barrier_t`) & Spinlocks (`pthread_spinlock_t`)

### Barrier Synchronization:
```c
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, count); // count = number of threads required
pthread_barrier_wait(&barrier);             // Blocks until 'count' threads arrive
pthread_barrier_destroy(&barrier);
```

### Spinlocks (Busy-Waiting):
```c
pthread_spinlock_t spin;
pthread_spin_init(&spin, PTHREAD_PROCESS_PRIVATE); // 0 or PTHREAD_PROCESS_PRIVATE
pthread_spin_lock(&spin);                          // Busy-waits consuming CPU
pthread_spin_unlock(&spin);
pthread_spin_destroy(&spin);
```

---

# PART 2: OpenMP (DIRECTIVES, CLAUSES & RUNTIME)

## 1. Compilation & Header
```c
#include <omp.h>
```
```bash
gcc -fopenmp -O3 program.c -o program
clang -fopenmp -O3 program.c -o program
```

---

## 2. Core Parallel & Work-Sharing Directives

### Parallel Region Construct:
```c
#pragma omp parallel [clause[[,] clause]...]
{
    // Structured block executed by team of threads
}
```

### Loop Work-Sharing:
```c
#pragma omp for [clause...]
for (int i = 0; i < N; i++) { ... }

// Combined construct (forks team and divides loop):
#pragma omp parallel for [clause...]
for (int i = 0; i < N; i++) { ... }
```

### Sections Work-Sharing:
```c
#pragma omp parallel sections
{
    #pragma omp section
    { task_A(); }
    #pragma omp section
    { task_B(); }
}
```

### Single & Master:
```c
#pragma omp single [nowait] [private(...)]
{ /* Executed by exactly ONE thread; implicit barrier unless nowait */ }

#pragma omp master
{ /* Executed by MASTER thread (rank 0) ONLY; NO implicit barrier */ }
```

---

## 3. Data Scoping Clauses

| Clause Syntax | Description | Identity / Initial Value |
| :--- | :--- | :--- |
| `shared(var1, var2)` | Shared across all threads in team. | Existing value prior to region |
| `private(var1, var2)` | Uninitialized private copy on each thread's stack. | **Undefined / Garbage** |
| `firstprivate(var1)` | Private copy initialized to value of master variable. | Copied from master variable |
| `lastprivate(var1)` | Private copy; sequentially last iteration copies value back to master. | Value of last iteration |
| `default(none)` | Forces explicit scoping for every variable (prevents latent data races). | N/A |
| `default(shared)` | Default in C/C++ if not specified. | Shared |

---

## 4. Reduction Clause: `reduction(operator : list)`

### Syntax:
```c
#pragma omp parallel for reduction(+: sum) reduction(max: peak_val)
for (int i = 0; i < N; i++) {
    sum += a[i];
    if (a[i] > peak_val) peak_val = a[i];
}
```

### Standard Operators & Identity Values:
| Operator | Initialized Identity Value |
| :---: | :---: |
| `+`, `-` | `0` |
| `*` | `1` |
| `&` | `~0` (All bits 1) |
| `\|`, `^` | `0` |
| `&&` | `1` (`true`) |
| `\|\|` | `0` (`false`) |
| `max` | Smallest representable value for type (`INT_MIN`, `-DBL_MAX`) |
| `min` | Largest representable value for type (`INT_MAX`, `DBL_MAX`) |

---

## 5. Loop Scheduling Clauses: `schedule(kind [, chunk])`

| Scheduling Clause | Chunk Assignment Strategy | Best Used For |
| :--- | :--- | :--- |
| `schedule(static [, chunk])` | Round-robin distribution at start. Default `chunk = N / P`. | Uniform, balanced loop iterations |
| `schedule(dynamic [, chunk])` | Shared work queue; threads grab `chunk` iterations on demand. Default `chunk = 1`. | Unpredictable, irregular workloads |
| `schedule(guided [, min_chunk])` | Exponentially decaying chunk sizes: $\lceil \text{remaining} / P \rceil$. | Decreasing workloads; reduces scheduling overhead |
| `schedule(runtime)` | Scheduling policy deferred to `OMP_SCHEDULE` env var. | Benchmarking and tuning |
| `schedule(auto)` | Decision delegated entirely to compiler/runtime. | Heuristic tuning |

### Other Useful Loop Clauses:
- `collapse(n)`: Flattens $n$ nested loops into a single linear iteration space.
- `nowait`: Removes the implicit barrier at the end of `#pragma omp for` or `#pragma omp single`.

---

## 6. Synchronization Directives

| Directive | Description |
| :--- | :--- |
| `#pragma omp critical [(name)]` | Structured block executed by one thread at a time across the entire team. Optional name creates independent locks. |
| `#pragma omp atomic [read \| write \| update \| capture]` | Hardware-level atomic memory operation on a single scalar location. Much faster than critical. |
| `#pragma omp barrier` | Explicit rendezvous barrier; all threads must reach before any proceed. |
| `#pragma omp ordered` | Enforces sequential execution order inside a loop marked with `#pragma omp for ordered`. |
| `#pragma omp flush [(list)]` | Memory fence enforcing memory consistency across thread views. |

---

## 7. OpenMP Tasking Model

```c
#pragma omp parallel
{
    #pragma omp single
    {
        // One thread creates tasks into the task pool:
        #pragma omp task [untied] [priority(p)] [depend(in:x) depend(out:y)]
        { task_A(); }

        #pragma omp task
        { task_B(); }

        #pragma omp taskwait // Waits for direct child tasks to complete
    }
}

// Task Loops for massive loops:
#pragma omp taskloop grainsize(1000)
for (int i = 0; i < N; i++) { ... }
```

---

## 8. OpenMP SIMD Vectorization

```c
#pragma omp simd [safelen(N)] [simdlen(N)] [aligned(ptr:64)] [linear(i:1)]
for (int i = 0; i < N; i++) {
    c[i] = a[i] * b[i];
}

// Combined Multithreading + Vectorization:
#pragma omp parallel for simd schedule(static)
for (int i = 0; i < N; i++) { ... }
```

---

## 9. OpenMP Runtime Library APIs (`<omp.h>`)

| Function Prototype | Description |
| :--- | :--- |
| `void omp_set_num_threads(int num);` | Sets default thread count for subsequent parallel regions. |
| `int omp_get_num_threads(void);` | Returns number of threads currently in team (returns 1 in serial). |
| `int omp_get_thread_num(void);` | Returns calling thread's unique rank ($0 \dots \text{threads}-1$). |
| `int omp_get_max_threads(void);` | Returns maximum number of threads available. |
| `int omp_get_num_procs(void);` | Returns count of online hardware processor cores. |
| `int omp_in_parallel(void);` | Returns non-zero if called from inside an active parallel region. |
| `double omp_get_wtime(void);` | High-resolution wall-clock timestamp (in seconds). |
| `double omp_get_wtick(void);` | Timer precision (seconds per tick). |

### Runtime Lock APIs (`omp_lock_t`):
```c
omp_lock_t lock;
omp_init_lock(&lock);
omp_set_lock(&lock);    // Blocking acquire
omp_test_lock(&lock);   // Non-blocking acquire (returns non-zero if acquired)
omp_unset_lock(&lock);  // Release
omp_destroy_lock(&lock);
```

---

## 10. OpenMP Environment Variables

| Variable | Typical Value | Purpose |
| :--- | :--- | :--- |
| `OMP_NUM_THREADS` | `export OMP_NUM_THREADS=8` | Sets default thread count for parallel regions. |
| `OMP_SCHEDULE` | `export OMP_SCHEDULE="dynamic,16"` | Specifies schedule for `schedule(runtime)`. |
| `OMP_STACKSIZE` | `export OMP_STACKSIZE=64M` | Sets thread stack size (prevents stack overflow on large private arrays). |
| `OMP_DYNAMIC` | `export OMP_DYNAMIC=FALSE` | Enables/disables runtime dynamic thread count adjustment. |
| `OMP_WAIT_POLICY` | `export OMP_WAIT_POLICY=ACTIVE` | `ACTIVE` (spins waiting; lower latency), `PASSIVE` (sleeps; frees CPU). |

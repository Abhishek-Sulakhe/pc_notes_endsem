# OpenMP Syntax Guide: Compilable Code Examples

This reference provides self-contained, compilable C code examples illustrating every standard OpenMP syntax construct, directive, clause, and runtime API.

---

## 1. Basic Parallel Region & Thread Discovery

Demonstrates creating a team of threads, querying thread IDs, team size, and measuring wall-clock execution time.

```c
#include <stdio.h>
#include <omp.h>

int main() {
    // Set thread count programmatically (overrides OMP_NUM_THREADS)
    omp_set_num_threads(4);

    double start_time = omp_get_wtime();

    #pragma omp parallel
    {
        int tid = omp_get_thread_num();     // Unique thread rank (0 to num_threads - 1)
        int total = omp_get_num_threads();  // Current active team size

        printf("Hello from Thread %d of %d\n", tid, total);
    } // Implicit barrier: all threads synchronize here before master continues

    double end_time = omp_get_wtime();
    printf("Total parallel region execution time = %f seconds\n", end_time - start_time);

    return 0;
}
```
**Compile & Run:**
```bash
gcc -fopenmp -O2 01_basic_parallel.c -o basic_parallel
./basic_parallel
```

---

## 2. Loop Work-Sharing (`#pragma omp for`) & Loop Collapse

Demonstrates dividing loop iterations among threads and collapsing nested 2D loops into a single linear iteration space.

```c
#include <stdio.h>
#include <omp.h>

#define N 8
#define ROWS 4
#define COLS 4

int main() {
    int a[N], b[N], c[N];
    for (int i = 0; i < N; i++) { a[i] = i; b[i] = i * 2; }

    // Combined construct: forks threads AND divides loop
    #pragma omp parallel for num_threads(4)
    for (int i = 0; i < N; i++) {
        c[i] = a[i] + b[i];
        printf("[Thread %d] computes index i = %d (c[%d] = %d)\n",
               omp_get_thread_num(), i, i, c[i]);
    }

    printf("\n--- 2D Loop Collapse ---\n");
    int matrix[ROWS][COLS];

    // collapse(2) flattens 4x4 matrix (16 iterations) across threads
    #pragma omp parallel for collapse(2) num_threads(4)
    for (int r = 0; r < ROWS; r++) {
        for (int c_idx = 0; c_idx < COLS; c_idx++) {
            matrix[r][c_idx] = r * 10 + c_idx;
            printf("[Thread %d] matrix[%d][%d] = %d\n",
                   omp_get_thread_num(), r, c_idx, matrix[r][c_idx]);
        }
    }

    return 0;
}
```

---

## 3. Loop Scheduling Policies (`static`, `dynamic`, `guided`)

Illustrates the four scheduling policies and chunk assignments.

```c
#include <stdio.h>
#include <unistd.h>
#include <omp.h>

#define N 12

int main() {
    omp_set_num_threads(3);

    printf("=== 1. STATIC SCHEDULING (chunk size = 2, round-robin) ===\n");
    #pragma omp parallel for schedule(static, 2)
    for (int i = 0; i < N; i++) {
        printf("[Thread %d] static iteration %d\n", omp_get_thread_num(), i);
    }

    printf("\n=== 2. DYNAMIC SCHEDULING (work queue on-demand) ===\n");
    #pragma omp parallel for schedule(dynamic, 2)
    for (int i = 0; i < N; i++) {
        printf("[Thread %d] dynamic iteration %d\n", omp_get_thread_num(), i);
        if (i % 2 == 0) usleep(10000); // Simulate irregular workload
    }

    printf("\n=== 3. GUIDED SCHEDULING (exponentially decaying chunks) ===\n");
    #pragma omp parallel for schedule(guided, 1)
    for (int i = 0; i < N; i++) {
        printf("[Thread %d] guided iteration %d\n", omp_get_thread_num(), i);
    }

    return 0;
}
```

---

## 4. Data Scoping (`shared`, `private`, `firstprivate`, `lastprivate`, `default(none)`)

Demonstrates the exact semantic differences and variable isolation across threads.

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int shared_var = 100;
    int firstpriv_var = 50;
    int lastpriv_var = -1;

    // default(none) forces explicit scoping for safety
    #pragma omp parallel for num_threads(4) \
        default(none) \
        shared(shared_var, stdout) \
        firstprivate(firstpriv_var) \
        lastprivate(lastpriv_var)
    for (int i = 0; i < 4; i++) {
        int tid = omp_get_thread_num();

        // private_var is allocated fresh on stack (uninitialized)
        int private_var = tid * 10;

        // firstpriv_var starts initialized with 50
        firstpriv_var += tid;

        // lastpriv_var captures value from sequentially last iteration (i == 3)
        lastpriv_var = tid * 1000;

        printf("[Thread %d] private=%d, firstprivate=%d, shared=%d\n",
               tid, private_var, firstpriv_var, shared_var);
    }

    printf("\n--- After Parallel Region ---\n");
    printf("lastprivate value preserved from last iteration: %d\n", lastpriv_var);
    printf("shared_var remains: %d\n", shared_var);

    return 0;
}
```

---

## 5. Parallel Reductions (`reduction(+: sum)`, `reduction(max: peak)`)

Demonstrates race-free distributed accumulation across threads without manual locking.

```c
#include <stdio.h>
#include <omp.h>

#define N 1000

int main() {
    int arr[N];
    for (int i = 0; i < N; i++) arr[i] = i + 1;

    long long total_sum = 0;
    int max_val = -1;

    // Both sum and max reduction performed concurrently
    #pragma omp parallel for num_threads(4) \
        reduction(+: total_sum) \
        reduction(max: max_val)
    for (int i = 0; i < N; i++) {
        total_sum += arr[i];
        if (arr[i] > max_val) {
            max_val = arr[i];
        }
    }

    printf("Parallel Sum = %lld (Formula check: %lld)\n",
           total_sum, ((long long)N * (N + 1)) / 2);
    printf("Parallel Maximum = %d\n", max_val);

    return 0;
}
```

---

## 6. Functional Sections (`#pragma omp sections`)

Demonstrates task-level coarse-grained parallelism where different threads run different functions.

```c
#include <stdio.h>
#include <omp.h>

void load_database() {
    printf("[Thread %d] Loading database records...\n", omp_get_thread_num());
}

void parse_network_stream() {
    printf("[Thread %d] Parsing incoming network sockets...\n", omp_get_thread_num());
}

void render_gui() {
    printf("[Thread %d] Rendering user interface...\n", omp_get_thread_num());
}

int main() {
    #pragma omp parallel sections num_threads(3)
    {
        #pragma omp section
        { load_database(); }

        #pragma omp section
        { parse_network_stream(); }

        #pragma omp section
        { render_gui(); }
    } // Implicit barrier: all sections finish before proceeding

    printf("All independent subsystem tasks completed.\n");
    return 0;
}
```

---

## 7. Single vs. Master Execution & The `nowait` Clause

Demonstrates when to execute code on only one thread and how `nowait` removes synchronization stalls.

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int shared_config = 0;

    #pragma omp parallel num_threads(4)
    {
        // 1. Single Construct: executed by EXACTLY ONE thread (any thread)
        #pragma omp single
        {
            printf("[Thread %d] ONE thread initializes shared configuration!\n",
                   omp_get_thread_num());
            shared_config = 42;
        } // Implicit barrier: all threads wait here until single finishes

        printf("[Thread %d] sees shared_config = %d\n",
               omp_get_thread_num(), shared_config);

        // 2. Master Construct: executed by MASTER thread (rank 0) ONLY
        #pragma omp master
        {
            printf("[Thread %d] Master thread logging progress (NO implicit barrier!)\n",
                   omp_get_thread_num());
        }

        // 3. nowait Clause: prevents threads from stalling at loop barrier
        #pragma omp for nowait
        for (int i = 0; i < 4; i++) {
            printf("[Thread %d] processing fast phase iteration %d\n",
                   omp_get_thread_num(), i);
        }

        printf("[Thread %d] immediately continues without waiting on others!\n",
               omp_get_thread_num());
    }

    return 0;
}
```

---

## 8. Mutual Exclusion: `critical` vs. `atomic` vs. `ordered`

Demonstrates safe updates to shared data and sequential output ordering within parallel loops.

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int critical_counter = 0;
    int atomic_counter = 0;

    #pragma omp parallel num_threads(8)
    {
        // 1. Critical Section (Lock-based, suitable for multi-line blocks)
        #pragma omp critical(counter_lock)
        {
            critical_counter++;
        }

        // 2. Atomic Directive (Hardware-level instruction, significantly faster)
        #pragma omp atomic update
        atomic_counter++;
    }

    printf("Final Critical Counter = %d\n", critical_counter);
    printf("Final Atomic Counter   = %d\n", atomic_counter);

    printf("\n--- Ordered Execution Inside Parallel Loop ---\n");
    #pragma omp parallel for ordered num_threads(4)
    for (int i = 0; i < 6; i++) {
        // Parallel computation here...
        int square = i * i;

        // Ordered clause guarantees output in sequential index order: 0, 1, 2, 3, 4, 5
        #pragma omp ordered
        {
            printf("[Thread %d] Index %d squared = %d\n",
                   omp_get_thread_num(), i, square);
        }
    }

    return 0;
}
```

---

## 9. Task Parallelism: Recursive Tree Traversal & `taskwait`

Demonstrates irregular task parallelism (e.g., recursive divide-and-conquer Fibonacci).

```c
#include <stdio.h>
#include <omp.h>

long long fibonacci(int n) {
    if (n < 2) return n;

    long long x, y;

    // Create asynchronous tasks for subproblems
    #pragma omp task shared(x)
    { x = fibonacci(n - 1); }

    #pragma omp task shared(y)
    { y = fibonacci(n - 2); }

    // Wait until both sub-tasks have resolved
    #pragma omp taskwait
    return x + y;
}

int main() {
    int n = 15;
    long long result;

    #pragma omp parallel num_threads(4)
    {
        // Single thread initiates the root recursive task
        #pragma omp single
        {
            result = fibonacci(n);
        }
    }

    printf("Fibonacci(%d) = %lld\n", n, result);
    return 0;
}
```

---

## 10. Explicit Runtime Locks (`omp_lock_t`)

Demonstrates low-level programmer-managed mutual exclusion.

```c
#include <stdio.h>
#include <omp.h>

omp_lock_t my_lock;
int shared_resource = 0;

int main() {
    // 1. Initialize lock
    omp_init_lock(&my_lock);

    #pragma omp parallel num_threads(4)
    {
        int tid = omp_get_thread_num();

        // 2. Acquire lock (blocking)
        omp_set_lock(&my_lock);

        shared_resource += 10;
        printf("[Thread %d] Acquired lock. Shared Resource = %d\n",
               tid, shared_resource);

        // 3. Release lock
        omp_unset_lock(&my_lock);

        // 4. Test lock (non-blocking)
        if (omp_test_lock(&my_lock)) {
            printf("[Thread %d] Successfully acquired lock via test_lock!\n", tid);
            omp_unset_lock(&my_lock);
        }
    }

    // 5. Destroy lock
    omp_destroy_lock(&my_lock);
    return 0;
}
```

---

## 11. SIMD Vectorization (`#pragma omp simd`)

Demonstrates generating data-parallel hardware vector instructions without thread spawning overhead.

```c
#include <stdio.h>
#include <omp.h>

#define SIZE 64

int main() {
    float a[SIZE], b[SIZE], c[SIZE];

    for (int i = 0; i < SIZE; i++) {
        a[i] = (float)i;
        b[i] = (float)(i * 2);
    }

    // Tells compiler to vectorize loop using SIMD vector registers (AVX/SSE)
    #pragma omp simd safelen(8)
    for (int i = 0; i < SIZE; i++) {
        c[i] = a[i] * b[i] + 1.5f;
    }

    // Combined multithreading + vectorization:
    #pragma omp parallel for simd schedule(static)
    for (int i = 0; i < SIZE; i++) {
        c[i] += 10.0f;
    }

    printf("Vectorized output sample: c[10] = %f\n", c[10]);
    return 0;
}
```
**Compilation with Vectorization Optimization:**
```bash
gcc -fopenmp -O3 -mavx2 11_simd_vectorization.c -o simd_vectorization
./simd_vectorization
```

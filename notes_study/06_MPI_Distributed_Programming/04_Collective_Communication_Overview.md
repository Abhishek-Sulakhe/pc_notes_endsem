# Subtopic 06.4: Collective Communication Primitives & Topologies

---

## 1. Pedagogical Motivation & Core Problem Statement

In [Subtopic 06.2](file:///d:/Academics/3rd%20year/PC/study/notes_study/06_MPI_Distributed_Programming/02_Point_to_Point_Communication_Send_Recv.md), we mastered point-to-point communication using `MPI_Send` and `MPI_Recv`. While point-to-point primitives give us complete, fine-grained control over individual message passing between paired ranks, real-world scientific and distributed parallel algorithms rarely operate as isolated pairs of communicating nodes.

Consider common algorithmic patterns in distributed computing:
1. **Parameter Initialization:** Process 0 reads a $1\text{ GB}$ configuration file or neural network weight matrix from disk and needs to send an exact copy to all $1,024$ worker processes.
2. **Domain Decomposition:** Process 0 holds an array of $1,000,000$ mesh points and needs to partition it into equal slices, sending slice $i$ to rank $i$.
3. **Global Convergence & Aggregation:** Every rank computes a local residual error $\epsilon_i$; the algorithm terminates only when the maximum error across all nodes ($\max_i \epsilon_i$) drops below $10^{-6}$.
4. **Phase Synchronization:** All processes must finish updating their local boundary values before any process starts calculating flux for the next simulation time step.

### What Happens If We Use Point-to-Point Loops?
If a developer implements broadcast by writing a naive loop on rank 0:
```c
// NAIVE POINT-TO-POINT BROADCAST (ANTI-PATTERN)
if (rank == 0) {
    for (int dest = 1; dest < size; dest++) {
        MPI_Send(buffer, count, MPI_INT, dest, tag, MPI_COMM_WORLD);
    }
} else {
    MPI_Recv(buffer, count, MPI_INT, 0, tag, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
}
```
Three severe problems immediately cripple performance and maintainability:
1. **Network Serialization Bottleneck ($O(P)$ Complexity):** Rank 0 must push the message through its single network interface card (NIC) $P-1$ times sequentially. If $P = 1,024$, rank $1,023$ sits completely idle waiting for its data, wasting massive compute capacity.
2. **Code Clutter & Cognitive Overhead:** Managing complex point-to-point loops, non-blocking requests (`MPI_Isend`), and buffer tracking across hundreds of ranks requires hundreds of lines of fragile boilerplate.
3. **Hardware Agnosticism:** The naive loop cannot take advantage of hardware multicast, shared-memory bypass (`cross-memory attach` / `XPMEM`) within multi-core sockets, or balanced binary/binomial network tree topologies.

**The Solution:** MPI provides **Collective Communication Primitives**. A collective function is a coordinated operation that involves **all** processes within a given communicator. The MPI implementation internally optimizes the underlying network topology—using binomial trees, recursive doubling, or ring pipelines—slashing time complexity from linear $O(P)$ to logarithmic $O(\log_2 P)$ or constant-bandwidth streaming.

---

## 2. Mathematical Definition & Golden Rules of Collective Communication

Before examining the individual primitives, we must establish the formal semantics and operational constraints defined by the MPI standard.

> [!IMPORTANT]
> ### The 5 Golden Rules of MPI Collectives
> 1. **Universal Participation:** Every single process belonging to the communicator (e.g., `MPI_COMM_WORLD`) **must** call the collective routine. If rank $k$ skips the call due to an `if (rank != k)` branch, the collective will wait indefinitely for rank $k$, resulting in a permanent **Distributed Deadlock**.
> 2. **Ordered Execution:** Collective calls must be executed in the exact same sequence by all participating processes. You cannot have rank 0 call `MPI_Bcast` followed by `MPI_Reduce` while rank 1 calls `MPI_Reduce` followed by `MPI_Bcast`.
> 3. **No Message Tags:** Point-to-point calls use tags to disambiguate messages (`tag = 0, 1, ...`). Collective operations **do not take tag arguments**. Matches are made strictly based on the communicator context and the order of execution.
> 4. **Blocking Semantics (Standard Collectives):** In standard MPI (MPI-1 / MPI-2), collective routines return only when their local involvement in the operation is complete (i.e., the caller's send/receive buffers can be safely reused). *Note: Return does not necessarily guarantee that other processes have completed or received the data (except for `MPI_Barrier`).*
> 5. **Data Type and Size Consistency:** For operations involving data exchange, the count and datatype specified by senders and receivers must be mutually compatible according to the primitive's specification.

---

## 3. Taxonomy of MPI Collective Primitives

MPI collective operations are categorized into four functional classes based on data movement patterns:

```
                          MPI COLLECTIVE PRIMITIVES
                                     |
    +-------------------+------------+------------+--------------------+
    |                   |                         |                    |
1. Synchronization   2. One-to-All             3. All-to-One        4. All-to-All
    |                   |                         |                    |
  MPI_Barrier        MPI_Bcast                 MPI_Gather           MPI_Allgather
                     MPI_Scatter               MPI_Reduce           MPI_Allreduce
                     MPI_Scatterv              MPI_Gatherv          MPI_Alltoall
                                                                    MPI_Scan
```

---

## 4. In-Depth Operational Primitives & Visual ASCII Architectures

### 4.1. Global Synchronization: `MPI_Barrier`

`MPI_Barrier` creates a distributed rendezvous point. No process can return from `MPI_Barrier` until all processes in the communicator have entered it.

```c
int MPI_Barrier(MPI_Comm comm);
```

#### ASCII Architectural Flow:
```
Time  Rank 0               Rank 1               Rank 2               Rank 3
  |     |                    |                    |                    |
  |   [Local Work]         [Local Work]         [Short Task]         [Long Task]
  |     |                    |                    |                    |
  |     v                    |                    v                    |
  |   MPI_Barrier(comm)      v                  MPI_Barrier(comm)      |
  |   (Waiting...)         MPI_Barrier(comm)    (Waiting...)           |
  |   (Waiting...)         (Waiting...)         (Waiting...)           v
  |   (Waiting...)         (Waiting...)         (Waiting...)         MPI_Barrier(comm)
  |     +--------------------+--------------------+--------------------+
  |     |       BARRIER SYNCHRONIZATION POINT ACHIEVED (ALL PRESENT)   |
  v     v                    v                    v                    v
      [Proceed]            [Proceed]            [Proceed]            [Proceed]
```

> [!CAUTION]
> Avoid overusing `MPI_Barrier`. Barriers serialize processes, destroy algorithmic concurrency, and amplify the penalty of "stragglers" (the slowest process stalls the entire cluster). Use them only when strictly necessary, such as before taking global wall-clock benchmark timestamps (`MPI_Wtime`).

---

### 4.2. One-to-All Broadcast: `MPI_Bcast`

A single designated **Root Process** distributes an identical copy of its buffer to all processes in the communicator (including itself).

```c
int MPI_Bcast(
    void *buffer,          // Starting address of buffer
    int count,             // Number of entries in buffer
    MPI_Datatype datatype, // Data type of buffer elements
    int root,              // Rank of broadcast root process
    MPI_Comm comm          // Communicator
);
```

#### Memory State Transformation:
```
           BEFORE BCAST                      AFTER BCAST (root = 0)
Rank 0:  [ D0 | D1 | D2 | D3 ]            Rank 0:  [ D0 | D1 | D2 | D3 ]
Rank 1:  [  ? |  ? |  ? |  ? ]    ===>    Rank 1:  [ D0 | D1 | D2 | D3 ]
Rank 2:  [  ? |  ? |  ? |  ? ]            Rank 2:  [ D0 | D1 | D2 | D3 ]
Rank 3:  [  ? |  ? |  ? |  ? ]            Rank 3:  [ D0 | D1 | D2 | D3 ]
```

- **Root Role:** On `root`, `buffer` acts as the input/source memory.
- **Worker Role:** On non-root processes, `buffer` acts as the output/destination memory and is overwritten with data received from `root`.

---

### 4.3. One-to-All Distribution: `MPI_Scatter`

`MPI_Scatter` divides a contiguous array on the root process into $P$ equal segments and delivers the $i$-th segment to rank $i$.

```c
int MPI_Scatter(
    const void *sendbuf,   // Source array (significant only at root)
    int sendcount,         // Number of elements sent to EACH process!
    MPI_Datatype sendtype, // Datatype of send elements
    void *recvbuf,         // Destination buffer on all processes
    int recvcount,         // Number of elements in recvbuf
    MPI_Datatype recvtype, // Datatype of recv elements
    int root,              // Rank of scatter root
    MPI_Comm comm          // Communicator
);
```

#### Memory State Transformation:
```
           BEFORE SCATTER                     AFTER SCATTER (root = 0)
Rank 0:  [ Seg 0 | Seg 1 | Seg 2 | Seg 3 ]    Rank 0:  [ Seg 0 ]
Rank 1:  [   ?   ]                     ===>   Rank 1:  [ Seg 1 ]
Rank 2:  [   ?   ]                            Rank 2:  [ Seg 2 ]
Rank 3:  [   ?   ]                            Rank 3:  [ Seg 3 ]
```

> [!WARNING]
> **The `sendcount` Exam Trap:**  
> In `MPI_Scatter`, `sendcount` is **not** the total size of `sendbuf`. It specifies the number of elements dispatched to **each individual process**. If root has an array of $100$ integers and $P = 4$, `sendcount` must be $25$, **not** $100$.

---

### 4.4. All-to-One Collection: `MPI_Gather`

`MPI_Gather` is the exact inverse of `MPI_Scatter`. Each process transmits an array segment from its local buffer. The root process collects these segments in rank-order ($0, 1, \dots, P-1$) and stores them contiguously in `recvbuf`.

```c
int MPI_Gather(
    const void *sendbuf,   // Source buffer on every process
    int sendcount,         // Number of elements sent by EACH process
    MPI_Datatype sendtype, // Datatype of send elements
    void *recvbuf,         // Destination buffer (significant only at root)
    int recvcount,         // Number of elements received from EACH process!
    MPI_Datatype recvtype, // Datatype of recv elements
    int root,              // Rank of gather root
    MPI_Comm comm          // Communicator
);
```

#### Memory State Transformation:
```
           BEFORE GATHER                      AFTER GATHER (root = 0)
Rank 0:  [ Seg 0 ]                            Rank 0:  [ Seg 0 | Seg 1 | Seg 2 | Seg 3 ]
Rank 1:  [ Seg 1 ]                     ===>   Rank 1:  [   Unchanged / Garbage   ]
Rank 2:  [ Seg 2 ]                            Rank 2:  [   Unchanged / Garbage   ]
Rank 3:  [ Seg 3 ]                            Rank 3:  [   Unchanged / Garbage   ]
```

- On non-root processes, `recvbuf` is ignored and can be passed as `NULL`.
- On root, `recvbuf` must be allocated to hold at least $P \times \text{recvcount}$ elements.

---

### 4.5. All-to-One Reduction: `MPI_Reduce`

`MPI_Reduce` collects data from all processes, combines them using an associative reduction operator (e.g., sum, product, maximum, minimum), and places the final accumulated result into `recvbuf` on the root process.

```c
int MPI_Reduce(
    const void *sendbuf,   // Source data on every process
    void *recvbuf,         // Result buffer (significant only at root)
    int count,             // Number of elements in sendbuf
    MPI_Datatype datatype, // Datatype of elements
    MPI_Op op,             // Reduction operator
    int root,              // Rank of reduction root
    MPI_Comm comm          // Communicator
);
```

#### Predefined MPI Reduction Operators (`MPI_Op`):

| Operator Name | Mathematical Operation | Typical Use Case |
| :--- | :--- | :--- |
| `MPI_SUM` | $\sum x_i$ | Vector dot product, total energy calculation |
| `MPI_PROD` | $\prod x_i$ | Compound probabilities, factorial scaling |
| `MPI_MAX` | $\max(x_i)$ | Maximum error residual, peak temperature |
| `MPI_MIN` | $\min(x_i)$ | Minimum eigenvalue, shortest path distance |
| `MPI_LAND` | $x_1 \land x_2 \land \dots$ | Logical AND (global convergence check) |
| `MPI_LOR` | $x_1 \lor x_2 \lor \dots$ | Logical OR (global flag termination) |
| `MPI_MAXLOC` | Returns maximum value + rank index | Determining which node owns the peak load |
| `MPI_MINLOC` | Returns minimum value + rank index | Finding minimum coordinate and rank |

#### Memory State Transformation:
```
Rank 0:  sendbuf = [ 10 ]
Rank 1:  sendbuf = [ 20 ]   ===( MPI_Reduce with MPI_SUM, root = 0 )===>  Rank 0: recvbuf = [ 100 ]
Rank 2:  sendbuf = [ 30 ]                                                 Rank 1..3: recvbuf unmodified
Rank 3:  sendbuf = [ 40 ]
```

---

### 4.6. All-to-All Distribution & Reduction: `MPI_Allreduce` & `MPI_Allgather`

In many algorithms, every process needs the final reduced value (for instance, the updated global step size in gradient descent) or the full gathered dataset.

Instead of calling `MPI_Reduce` followed by `MPI_Bcast`, MPI provides combined primitives that execute more efficiently in a single network pass:

#### `MPI_Allreduce`:
Logically equivalent to `MPI_Reduce` to root followed by `MPI_Bcast` to all ranks.

```c
int MPI_Allreduce(
    const void *sendbuf,
    void *recvbuf,
    int count,
    MPI_Datatype datatype,
    MPI_Op op,
    MPI_Comm comm
);
```
*(Notice: There is **no root parameter** because all ranks receive the result).*

```
Rank 0:  [ 10 ]                                   Rank 0:  [ 100 ]
Rank 1:  [ 20 ]   ===( MPI_Allreduce, MPI_SUM )===> Rank 1:  [ 100 ]
Rank 2:  [ 30 ]                                   Rank 2:  [ 100 ]
Rank 3:  [ 40 ]                                   Rank 3:  [ 100 ]
```

---

## 5. Algorithmic Topologies & Network Complexity Analysis

Why are collective operations dramatically faster than linear point-to-point loops?

### 5.1. Linear Naive Loop vs. Binomial Tree Broadcast

Suppose the root process must send a message of size $m$ bytes to $P$ processes across an interconnect characterized by latency $\alpha$ (startup time per message) and inverse bandwidth $\beta$ (transfer time per byte).

#### Linear Point-to-Point Loop ($P$ steps):
Root transmits $P-1$ sequential messages.
$$T_{\text{linear}} = (P - 1) \cdot (\alpha + m \cdot \beta)$$

For large $P$ (e.g., $P = 1,024$), time scales as **$\mathcal{O}(P)$**.

#### Binomial Spanning Tree (Logarithmic Scaling):
In a binomial tree, any process that has already received the data becomes an active sender in the subsequent step.

```
Step 0:  Rank 0 holds data.
Step 1:  Rank 0 sends to Rank 1.               (Active: 0, 1)
Step 2:  Rank 0 sends to Rank 2; Rank 1 -> 3.   (Active: 0, 1, 2, 3)
Step 3:  Rank 0 -> 4; 1 -> 5; 2 -> 6; 3 -> 7.   (Active: 0..7)
```

```
                 BINOMIAL TREE BROADCAST (P = 8)

Step 0:        [ Rank 0 ]
               /    |    \
Step 1:  [ Rank 1 ] |     \
          /         |      \
Step 2: [ Rank 3 ] [ Rank 2 ] \
          |         |          \
Step 3: [ Rank 7 ] [ Rank 6 ] [ Rank 4 ]
                                 |
                               [ Rank 5 ]
```

At step $k$, the number of informed processes doubles from $2^{k-1}$ to $2^k$. The total number of steps required to reach all $P$ processes is:
$$\text{Steps} = \lceil \log_2 P \rceil$$

Total communication time:
$$T_{\text{binomial}} = \lceil \log_2 P \rceil \cdot (\alpha + m \cdot \beta)$$

For $P = 1,024$, $\lceil \log_2 1024 \rceil = 10$. The binomial tree requires only **$10$ message hops** instead of $1,023$ serial hops—a **$100\times$ speedup** in network latency!

---

### 5.2. Recursive Doubling for `MPI_Allreduce`

For small-to-medium message sizes, `MPI_Allreduce` uses **Recursive Doubling** (also known as hypercube exchange).
- In round $d$ ($d = 0, 1, \dots, \log_2 P - 1$), process $r$ exchanges data with process $r \oplus 2^d$ (bitwise XOR).
- After $\log_2 P$ rounds, all processes possess the globally reduced array.

```
Round 0 (dist 1):  0 <-> 1,   2 <-> 3,   4 <-> 5,   6 <-> 7
Round 1 (dist 2):  0 <-> 2,   1 <-> 3,   4 <-> 6,   5 <-> 7
Round 2 (dist 4):  0 <-> 4,   1 <-> 5,   2 <-> 6,   3 <-> 7
```

---

## 6. Comprehensive Compilable C Implementations

### Program 1: Distributed Array Sum & Global Average (`MPI_Scatter` + `MPI_Reduce`)

This production-grade example illustrates how rank 0 generates a global dataset, scatters equal slices to all worker ranks, each rank computes its local sum, and `MPI_Reduce` combines the local sums into a single global total.

```c
/******************************************************************************
 * File: mpi_collective_scatter_reduce.c
 * Description: Demonstrates data distribution using MPI_Scatter and 
 *              result aggregation using MPI_Reduce.
 * Compilation: mpicc -Wall -O3 mpi_collective_scatter_reduce.c -o scatter_reduce
 * Execution:   mpirun -np 4 ./scatter_reduce
 ******************************************************************************/

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

#define ELEMENTS_PER_PROC 5 // Each process handles 5 elements

int main(int argc, char *argv[]) {
    int rank, size;
    int *global_array = NULL;
    int local_array[ELEMENTS_PER_PROC];
    long long local_sum = 0;
    long long global_sum = 0;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int total_elements = size * ELEMENTS_PER_PROC;

    // Rank 0 allocates and populates the master dataset
    if (rank == 0) {
        global_array = (int *)malloc(total_elements * sizeof(int));
        if (global_array == NULL) {
            fprintf(stderr, "Rank 0: Memory allocation failed!\n");
            MPI_Abort(MPI_COMM_WORLD, 1);
        }

        printf("=== MPI Collective Communication: Scatter & Reduce ===\n");
        printf("Cluster Size = %d ranks, Total Elements = %d\n", size, total_elements);
        printf("Master Array: ");
        for (int i = 0; i < total_elements; i++) {
            global_array[i] = i + 1; // Values: 1, 2, 3, ..., total_elements
            printf("%d ", global_array[i]);
        }
        printf("\n------------------------------------------------------\n");
    }

    /*
     * MPI_Scatter:
     * - global_array is partitioned into chunks of size ELEMENTS_PER_PROC.
     * - The i-th chunk is copied into local_array on process rank i.
     * - sendcount is ELEMENTS_PER_PROC (count per process, NOT total_elements).
     */
    MPI_Scatter(
        global_array,           // Source buffer (valid on root)
        ELEMENTS_PER_PROC,      // Number of elements sent to EACH process
        MPI_INT,                // Datatype sent
        local_array,            // Destination buffer on all ranks
        ELEMENTS_PER_PROC,      // Number of elements received
        MPI_INT,                // Datatype received
        0,                      // Root rank
        MPI_COMM_WORLD          // Communicator
    );

    // Each rank computes the partial sum of its assigned slice
    for (int i = 0; i < ELEMENTS_PER_PROC; i++) {
        local_sum += local_array[i];
    }
    printf("[Rank %d] Received slice: [%d .. %d], Local Partial Sum = %lld\n",
           rank, local_array[0], local_array[ELEMENTS_PER_PROC - 1], local_sum);

    /*
     * MPI_Reduce:
     * - Gathers local_sum from all ranks.
     * - Applies MPI_SUM associative reduction.
     * - Stores final scalar into global_sum at root (rank 0).
     */
    MPI_Reduce(
        &local_sum,             // Local input buffer
        &global_sum,            // Result buffer on root
        1,                      // Count of elements
        MPI_LONG_LONG,          // Datatype
        MPI_SUM,                // Reduction operation
        0,                      // Root rank
        MPI_COMM_WORLD          // Communicator
    );

    // Root prints final verified answer
    if (rank == 0) {
        // Analytical verification: Sum of 1..N = N*(N+1)/2
        long long expected = ((long long)total_elements * (total_elements + 1)) / 2;
        double average = (double)global_sum / total_elements;

        printf("------------------------------------------------------\n");
        printf("Global Sum Collected at Rank 0 = %lld (Expected = %lld)\n", global_sum, expected);
        printf("Global Array Arithmetic Mean    = %.2f\n", average);
        printf("Verification: %s\n", (global_sum == expected) ? "SUCCESS (MATCH)" : "FAILURE");

        free(global_array);
    }

    MPI_Finalize();
    return 0;
}
```

#### Expected Terminal Output (`mpirun -np 4 ./scatter_reduce`):
```text
=== MPI Collective Communication: Scatter & Reduce ===
Cluster Size = 4 ranks, Total Elements = 20
Master Array: 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 
------------------------------------------------------
[Rank 0] Received slice: [1 .. 5], Local Partial Sum = 15
[Rank 1] Received slice: [6 .. 10], Local Partial Sum = 40
[Rank 2] Received slice: [11 .. 15], Local Partial Sum = 65
[Rank 3] Received slice: [16 .. 20], Local Partial Sum = 90
------------------------------------------------------
Global Sum Collected at Rank 0 = 210 (Expected = 210)
Global Array Arithmetic Mean    = 10.50
Verification: SUCCESS (MATCH)
```

---

### Program 2: Dynamic Convergence & Threshold Broadcast (`MPI_Bcast` + `MPI_Allreduce`)

This program demonstrates how simulation parameters are broadcast to all ranks, followed by an `MPI_Allreduce` call that lets every worker simultaneously discover whether the entire cluster has met a termination criteria.

```c
/******************************************************************************
 * File: mpi_bcast_allreduce.c
 * Description: Demonstrates runtime configuration broadcast and global
 *              convergence detection using MPI_Allreduce.
 * Compilation: mpicc -Wall -O3 mpi_bcast_allreduce.c -o bcast_allreduce
 * Execution:   mpirun -np 4 ./bcast_allreduce
 ******************************************************************************/

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    double tolerance;
    int max_iterations;
} Config;

int main(int argc, char *argv[]) {
    int rank, size;
    Config config;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // Rank 0 configures simulation parameters
    if (rank == 0) {
        config.tolerance = 0.005;
        config.max_iterations = 10;
        printf("[Master] Initialized tolerance = %f, max_iter = %d\n",
               config.tolerance, config.max_iterations);
    }

    /*
     * MPI_Bcast:
     * - Distributes config struct from rank 0 to all other ranks.
     * - We send sizeof(Config) bytes using MPI_BYTE.
     */
    MPI_Bcast(&config, sizeof(Config), MPI_BYTE, 0, MPI_COMM_WORLD);

    // Simulate iterative calculation with random local residuals
    srand(rank * 100 + 7);
    double local_error = 0.01 / (rank + 1); // Rank 0 has 0.01, Rank 1 has 0.005, etc.
    double max_global_error = 0.0;

    /*
     * MPI_Allreduce:
     * - Finds the maximum error across all nodes using MPI_MAX.
     * - Stores the result directly in max_global_error on EVERY process.
     */
    MPI_Allreduce(&local_error, &max_global_error, 1, MPI_DOUBLE, MPI_MAX, MPI_COMM_WORLD);

    // All ranks can now independently check the global convergence condition
    if (max_global_error <= config.tolerance) {
        printf("[Rank %d] Converged! Global Peak Error = %f <= Tolerance %f\n",
               rank, max_global_error, config.tolerance);
    } else {
        printf("[Rank %d] Iteration continues. Global Peak Error = %f > Tolerance %f\n",
               rank, max_global_error, config.tolerance);
    }

    MPI_Finalize();
    return 0;
}
```

---

## 7. Solved Numerical Problems

### Numerical Problem 1: Binomial Tree vs. Linear Point-to-Point Scalability

#### Problem Statement:
An MPI weather simulation cluster consists of $P = 1,024$ nodes. The master node must broadcast a boundary conditions table of size $m = 256\text{ KB}$ ($256 \times 10^3\text{ bytes}$) to all nodes.  
The network parameters are:
- Latency (startup time) $\alpha = 5\text{ }\mu\text{s} = 5 \times 10^{-6}\text{ s}$
- Network bandwidth $B = 10\text{ GB/s} = 10 \times 10^9\text{ bytes/s}$ ($\beta = \frac{1}{B} = 10^{-10}\text{ s/byte}$)

Calculate:
1. The execution time of a naive linear point-to-point broadcast loop.
2. The execution time of a binomial tree collective broadcast (`MPI_Bcast`).
3. The speedup achieved by the collective binomial tree.

---

#### Step-by-Step Solution:

**Step 1: Compute single-message transfer time ($t_{\text{msg}}$):**
$$t_{\text{msg}} = \alpha + m \cdot \beta$$
$$t_{\text{msg}} = (5 \times 10^{-6}) + (256 \times 10^3 \times 10^{-10})$$
$$t_{\text{msg}} = 5 \times 10^{-6} + 25.6 \times 10^{-6} = 30.6\text{ }\mu\text{s} = 3.06 \times 10^{-5}\text{ s}$$

---

**Step 2: Calculate Naive Linear Broadcast Time ($T_{\text{linear}}$):**
The root transmits sequentially to $P - 1 = 1,023$ worker ranks:
$$T_{\text{linear}} = (P - 1) \cdot t_{\text{msg}}$$
$$T_{\text{linear}} = 1,023 \times 30.6\text{ }\mu\text{s} = 31,303.8\text{ }\mu\text{s} \approx 31.30\text{ ms}$$

---

**Step 3: Calculate Binomial Tree Broadcast Time ($T_{\text{binomial}}$):**
Number of communication steps:
$$\text{Steps} = \lceil \log_2 P \rceil = \log_2 (1,024) = 10\text{ steps}$$
$$T_{\text{binomial}} = \lceil \log_2 P \rceil \cdot t_{\text{msg}}$$
$$T_{\text{binomial}} = 10 \times 30.6\text{ }\mu\text{s} = 306.0\text{ }\mu\text{s} \approx 0.306\text{ ms}$$

---

**Step 4: Calculate Speedup ($S$):**
$$S = \frac{T_{\text{linear}}}{T_{\text{binomial}}} = \frac{31,303.8\text{ }\mu\text{s}}{306.0\text{ }\mu\text{s}} \approx 102.3\times$$

$$\boxed{\text{Linear Time} = 31.30\text{ ms}, \quad \text{Binomial Tree Time} = 0.306\text{ ms}, \quad \text{Speedup} = 102.3\times}$$

---

### Numerical Problem 2: Buffer Allocation & Rank Offset Calculation for `MPI_Gather`

#### Problem Statement:
A matrix multiplication program executes on $P = 8$ processes. Each process generates a local output block of $C_{\text{local}}$ consisting of $N = 500$ double-precision floating-point numbers (`sizeof(double) = 8\text{ bytes}`).  
Process rank 0 calls `MPI_Gather` to assemble the entire matrix into `C_global`.

1. What is the required buffer allocation size (in bytes) for `recvbuf` on the root process?
2. What value must be passed for `sendcount` on worker processes, and what value must be passed for `recvcount` on the root process?
3. Calculate the byte offset in `recvbuf` where the data from Process rank 5 will be stored.

---

#### Step-by-Step Solution:

**Step 1: Required memory allocation on Root (`recvbuf`):**
$$\text{Total Elements} = P \times N = 8 \times 500 = 4,000\text{ elements}$$
$$\text{Total Memory} = 4,000 \times 8\text{ bytes} = 32,000\text{ bytes} = 32\text{ KB}$$

---

**Step 2: Determine `sendcount` and `recvcount`:**
According to MPI semantics, `sendcount` is the number of elements dispatched by each process, and `recvcount` is the number of elements received **from each process individually**:
$$\text{sendcount} = 500$$
$$\text{recvcount} = 500$$

*(Note: Passing $4,000$ to `recvcount` would cause an out-of-bounds memory access crash).*

---

**Step 3: Byte offset for Rank 5:**
The gathered blocks are stored contiguously in increasing rank order:
$$\text{Element Offset} = \text{rank} \times \text{count} = 5 \times 500 = 2,500\text{ elements}$$
$$\text{Byte Offset} = 2,500 \times 8\text{ bytes} = 20,000\text{ bytes}$$

$$\boxed{\text{Root Allocation} = 32\text{ KB}, \quad \text{sendcount} = 500, \quad \text{recvcount} = 500, \quad \text{Offset (Rank 5)} = 20,000\text{ bytes}}$$

---

## 8. Common Exam Pitfalls & Trap Alerts

> [!CAUTION]
> ### 1. Conditional Collective Invocations (Distributed Deadlock)
> **Faulty Pattern:**
> ```c
> if (rank == 0) {
>     MPI_Bcast(data, 10, MPI_INT, 0, MPI_COMM_WORLD); // DEADLOCK!
> }
> ```
> **Reason:** Because `MPI_Bcast` is a collective primitive, every rank in `MPI_COMM_WORLD` must participate. Process 0 waits forever for the worker ranks to check in, while ranks $1..P-1$ bypass the block and continue or terminate.  
> **Rule:** Never wrap collective communication primitives inside rank-restricted conditionals unless the communicator itself has been split (e.g., via `MPI_Comm_split`).

> [!WARNING]
> ### 2. The Total vs. Per-Process Count Confusion
> Developers frequently pass the total aggregated array size to `MPI_Scatter` or `MPI_Gather`.
> - **In `MPI_Scatter`:** `sendcount` is elements **per recipient**, not total elements in `sendbuf`.
> - **In `MPI_Gather`:** `recvcount` is elements **per sender**, not total elements in `recvbuf`.

> [!NOTE]
> ### 3. Overlapping Buffers and Aliasing
> In standard C and MPI, passing the same buffer as both `sendbuf` and `recvbuf` (e.g., `MPI_Allreduce(arr, arr, ...)`) is strictly forbidden by the standard and results in undefined memory behavior.  
> If an in-place reduction is required, use the special sentinel value `MPI_IN_PLACE`:
> ```c
> MPI_Allreduce(MPI_IN_PLACE, arr, count, MPI_INT, MPI_SUM, MPI_COMM_WORLD);
> ```

---

## 9. Comprehensive Comparison Table

| Primitive | Participation | Send Buffer Role | Recv Buffer Role | Root Parameter | Primary Application |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`MPI_Barrier`** | All ranks | None | None | None | Staging computational phases & benchmarking |
| **`MPI_Bcast`** | All ranks | Source (on root) | Destination (workers) | Yes (`root`) | Distributing configuration, inputs, weights |
| **`MPI_Scatter`** | All ranks | Partitioned array | Disjoint slice received | Yes (`root`) | Domain decomposition, work distribution |
| **`MPI_Gather`** | All ranks | Local slice sent | Reassembled array | Yes (`root`) | Collecting final simulation grid at master |
| **`MPI_Reduce`** | All ranks | Local operand sent | Accumulated scalar/array | Yes (`root`) | Global sum, error norm, dot product |
| **`MPI_Allreduce`** | All ranks | Local operand sent | Accumulated result | **No** (all ranks) | Convergence detection, iterative solvers |
| **`MPI_Alltoall`** | All ranks | Transposed slices | Transposed slices | **No** (all ranks) | Distributed matrix transpose, FFT |

---

## 10. Quick University Exam Revision Sheet

- **Collective Communication:** Coordinated communication involving all processes in a communicator. No tags are used; calls must match in sequence across all ranks.
- **Root Process:** The designated rank that acts as the sole source (in `Bcast`, `Scatter`) or sole destination (in `Gather`, `Reduce`).
- **Complexity Advantage:** Replaces naive $\mathcal{O}(P)$ linear serialization with balanced $\mathcal{O}(\log_2 P)$ binomial tree or hypercube communication schedules.
- **Essential Primitives to Memorize for Exam:**
  - `MPI_Bcast(buf, count, type, root, comm)`
  - `MPI_Scatter(sendbuf, sendcnt, sendtype, recvbuf, recvcnt, recvtype, root, comm)`
  - `MPI_Gather(sendbuf, sendcnt, sendtype, recvbuf, recvcnt, recvtype, root, comm)`
  - `MPI_Reduce(sendbuf, recvbuf, count, type, op, root, comm)`
  - `MPI_Allreduce(sendbuf, recvbuf, count, type, op, comm)`
  - `MPI_Barrier(comm)`
- **Key Formulas:**
  - Linear Broadcast Time: $T = (P - 1)(\alpha + m\beta)$
  - Binomial Tree Broadcast Time: $T = \lceil \log_2 P \rceil (\alpha + m\beta)$

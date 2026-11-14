# High-Probability University Exam Questions & Model Answers

This guide contains the most critical, high-probability theory, derivation, and code-debugging questions encountered in university Parallel Computing examinations, complete with rigorous model answers, diagrams, and evaluation criteria.

---

## Question 1: Architectural Classification & Memory Systems

> **Question:**  
> (a) Classify computer architectures according to Flynn's Taxonomy. Explain each category with real-world examples.  
> (b) Contrast Uniform Memory Access (UMA) and Cache-Coherent Non-Uniform Memory Access (ccNUMA) architectures. Draw structural block diagrams and explain the concept of the NUMA factor.

---

### Model Answer:

#### (a) Flynn's Taxonomy
Michael J. Flynn (1966) classified all computing systems into four distinct categories based on the multiplicity of hardware instruction streams and data streams:

1. **SISD (Single Instruction, Single Data):**
   - A single processor executes a single sequential instruction stream operating on a single memory data stream.
   - *Example:* Classical uniprocessors (e.g., Intel 8086, unpipelined RISC architectures).
2. **SIMD (Single Instruction, Multiple Data):**
   - A single control unit fetches and decodes an instruction, broadcasting it simultaneously to multiple processing elements. Each element executes the identical operation synchronously on distinct data elements.
   - *Example:* Modern vector extensions (Intel AVX-512, ARM SVE) and Graphics Processing Units (GPUs / Streaming Multiprocessors).
3. **MISD (Multiple Instruction, Single Data):**
   - Multiple distinct instruction streams operate concurrently on the same single data stream.
   - *Example:* Fault-tolerant flight computers (e.g., NASA Space Shuttle redundant voting systems where three independent programs process sensor data to vote on flight decisions).
4. **MIMD (Multiple Instruction, Multiple Data):**
   - Multiple autonomous processors concurrently execute different instruction streams on completely independent data streams.
   - *Example:* Multi-core CPUs (Intel Core i9, AMD EPYC) and distributed HPC clusters.

---

#### (b) UMA vs. ccNUMA Architectures

```
     UMA (Symmetric Multiprocessor)                    ccNUMA (Distributed Shared Memory)
+---------------------------------------+       +-------------------+       +-------------------+
|  [Core 0]   [Core 1] ... [Core N-1]   |       |  Socket 0 (Node)  |       |  Socket 1 (Node)  |
|     |          |             |        |       | [C0] [C1] [C2] [C3|       | [C4] [C5] [C6] [C7|
|  +---------------------------------+  |       |   | L1/L2/L3 Caches|      |   | L1/L2/L3 Caches|
|  |     UNIFORM SYSTEM BUS / XBAR   |  |       |   +---------------+       |   +---------------+
|  +---------------------------------+  |       |   | Memory (Local)|       |   | Memory (Local)|
|                   |                   |       +-------------------+       +-------------------+
|          [ Shared Main Memory ]       |                 ^                           ^
+---------------------------------------+                 |     QPI / UPI Interconnect |
                                                          +===========================+
```

#### Detailed Comparison Table:

| Architectural Metric | UMA (Uniform Memory Access) | ccNUMA (Cache-Coherent NUMA) |
| :--- | :--- | :--- |
| **Physical Memory** | Centralized physical memory pool | Physically distributed among CPU sockets |
| **Access Latency** | Identical latency for all cores to all memory addresses | **Non-uniform:** Fast local access; slow remote access across interconnect |
| **Hardware Scalability** | Severely limited (typically $\le 8$ sockets due to bus contention) | High scalability (dozens of sockets; hundreds of physical cores) |
| **Interconnect** | Shared system bus or central crossbar | High-speed point-to-point coherent links (Intel UPI, AMD Infinity Fabric) |
| **NUMA Factor** | Always equal to $1.0$ | Greater than $1.0$ ($\text{NUMA Factor} = \frac{T_{\text{remote}}}{T_{\text{local}}} \approx 2.0\text{ to }3.5$) |

---

## Question 2: Strong Scaling vs. Weak Scaling (Amdahl vs. Gustafson)

> **Question:**  
> (a) State and derive Amdahl's Law from first principles. State its physical asymptotic limit.  
> (b) Explain why Amdahl's Law is considered pessimistic for modern High-Performance Computing, and derive Gustafson-Barsis's Law for weak scaling.

---

### Model Answer:

#### (a) Derivation of Amdahl's Law (Strong Scaling)
Let $T_1$ be the sequential execution time of a program on 1 processor.  
Let the program be composed of:
- A strictly serial fraction $s$ ($0 \le s \le 1$).
- A parallelizable fraction $p = 1 - s$.

When executed on $P$ processors under the **Strong Scaling** assumption (where the problem size remains fixed):
1. The serial portion takes time: $T_{serial} = s \cdot T_1$.
2. The parallel portion scales ideally across $P$ processors: $T_{parallel} = \frac{(1 - s) \cdot T_1}{P}$.

The total parallel execution time $T_P$ is:
$$T_P = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{P} = T_1 \left( s + \frac{1 - s}{P} \right)$$

Speedup $S_P$ is defined as $\frac{T_1}{T_P}$:
$$\boxed{S_P = \frac{T_1}{T_1 \left( s + \frac{1 - s}{P} \right)} = \frac{1}{s + \frac{1 - s}{P}}}$$

**Asymptotic Limit:**  
As the number of processors approaches infinity ($P \to \infty$):
$$\boxed{S_{\infty} = \lim_{P \to \infty} \frac{1}{s + \frac{1 - s}{P}} = \frac{1}{s}}$$
*Significance:* If only $5\%$ of a code is sequential ($s = 0.05$), the maximum speedup on an infinite number of processors cannot exceed $\frac{1}{0.05} = 20\times$.

---

#### (b) Gustafson-Barsis's Law (Weak Scaling)
Amdahl's Law assumes that the problem size remains fixed as processors increase. In real-world scientific computing, researchers use larger supercomputers **not** to solve the same tiny problem faster, but to solve massively larger, higher-resolution problems in the same total time (**Weak Scaling**).

Let $T_P$ be the normalized parallel execution time on $P$ processors.  
Let $s'$ be the measured serial fraction on the parallel system:
- Parallel execution time: $T_P = s' + (1 - s') = 1$.
- On 1 processor, the parallel workload cannot be divided and must execute sequentially:
  $$T_1 = s' \cdot T_P + P(1 - s') \cdot T_P$$
  $$T_1 = [s' + P(1 - s')] \cdot T_P$$

The Scaled Speedup $S_{scaled}$ is:
$$S_{scaled} = \frac{T_1}{T_P} = s' + P(1 - s')$$
$$\boxed{S_{scaled} = P - s'(P - 1)}$$

*Conclusion:* Unlike Amdahl's Law, scaled speedup grows **linearly** with $P$. If $s' = 0.05$ on $P = 1,000$ cores:
$$S_{scaled} = 1,000 - 0.05(999) = 1,000 - 49.95 = 950.05\times$$
Gustafson's law accurately models real-world HPC scalability.

---

## Question 3: Synchronization Primitives: Mutex vs. Binary Semaphore vs. Spinlock

> **Question:**  
> Provide a comprehensive comparative analysis of a Mutex, a Binary Semaphore, and a Spinlock with respect to:  
> 1. Ownership semantics.  
> 2. CPU scheduling behavior under contention.  
> 3. Priority inversion susceptibility.  
> 4. Performance characteristics (The Contention Cliff).

---

### Model Answer:

#### Master Synchronization Comparison Table:

| Criterion | Mutex (`pthread_mutex_t`) | Binary Semaphore (`sem_t`) | Spinlock (`pthread_spinlock_t`) |
| :--- | :--- | :--- | :--- |
| **Concept / Ownership** | **Strict Ownership:** Only the thread that locked the mutex can unlock it. | **No Ownership:** Any thread (or interrupt handler) can signal (`sem_post`) to unblock a waiting thread. | **Strict Ownership:** Acquired and released by the holding thread. |
| **Contention Behavior** | **Sleep / Block:** Deschedules waiting threads into an OS wait queue; yields CPU to other processes. | **Sleep / Block:** Puts waiting threads to sleep when the counter reaches 0. | **Busy-Wait:** Executes an active polling loop on the CPU core consuming $100\%$ CPU cycles. |
| **Context Switch Overhead** | Incurs 2 OS context switches ($\sim 1\text{--}5\text{ }\mu\text{s}$) on lock contention. | Incurs 2 OS context switches on contention. | **Zero context switches;** ultra-low latency ($\approx 10\text{--}50\text{ ns}$) if lock is held briefly. |
| **Priority Inversion** | Can mitigate priority inversion using Priority Inheritance protocols (`PTHREAD_PRIO_INHERIT`). | Cannot prevent priority inversion (no thread ownership to inherit priority from). | Highly vulnerable to deadlocks if a low-priority holder is preempted by a high-priority spinner. |
| **Best Used For** | Long critical sections or I/O operations inside locks. | Signaling, producer-consumer coordination, and event notifications. | Extremely short critical sections ($\le$ context switch latency) on dedicated cores. |

#### The "Contention Cliff" of Spinlocks:
When the number of competing threads exceeds physical CPU cores ($N > \text{Cores}$), spinlocks suffer a catastrophic throughput collapse (the **Contention Cliff**). The thread holding the lock may be preempted by the OS scheduler, while all other threads spin fruitlessly, burning power, causing cache-line invalidation storms, and delaying the lock holder from being rescheduled.

---

## Question 4: Condition Variables & Spurious Wakeups

> **Question:**  
> In POSIX Pthreads:  
> (a) Why must `pthread_cond_wait()` always take a locked mutex as its second parameter?  
> (b) What is a "Spurious Wakeup"? Why is checking condition state using `if (condition)` considered a fatal programming bug? Provide the correct code pattern.

---

### Model Answer:

#### (a) Why `pthread_cond_wait` Requires a Mutex
`pthread_cond_wait(&cond, &mutex)` must perform an **atomic transition**:
1. Release the mutex lock.
2. Put the calling thread to sleep on the condition variable queue.

If these two actions were not executed atomically in hardware/kernel:
- Thread A checks `while (count == 0)` and decides to sleep.
- Before Thread A calls wait, Thread B runs, adds an item, calls `pthread_cond_signal()`, and finishes.
- Thread A now executes `wait()`. Because the signal arrived *before* Thread A was sleeping, the signal is permanently lost (**Lost Wakeup Hazard**), and Thread A sleeps forever.
Passing the locked mutex guarantees that the release and sleep registration happen atomically.

---

#### (b) Spurious Wakeups & The `while` Loop Rule
A **Spurious Wakeup** occurs when a waiting thread unblocks and returns from `pthread_cond_wait()` even though no thread explicitly signaled the condition variable. This can occur due to:
1. Low-level OS kernel signal interruptions (e.g., UNIX signals `SIGINT`, `SIGCHLD`).
2. Multi-core kernel race resolutions where waking up an extra thread is preferred over complex lock verification.
3. Multiple competing consumer threads where Thread 1 wakes up and consumes the item before Thread 2 wakes up.

#### The Fatal Bug (`if` Statement):
```c
// FATAL BUG: IF STATEMENT
if (count == 0) {
    pthread_cond_wait(&not_empty, &mutex);
}
int item = buffer[--count]; // DANGER: Buffer underflow if wakeup was spurious!
```

#### Correct Golden Pattern (`while` Loop):
```c
// CORRECT & SAFE PATTERN
while (count == 0) {
    pthread_cond_wait(&not_empty, &mutex); // Re-tests condition upon waking!
}
int item = buffer[--count]; // Guaranteed safe
```

---

## Question 5: OpenMP Loop Scheduling Policies

> **Question:**  
> Explain the four OpenMP loop scheduling policies (`static`, `dynamic`, `guided`, `auto`). Provide mathematical formulas, runtime trade-offs, and algorithmic scenarios where each policy is preferred.

---

### Model Answer:

```
STATIC SCHEDULING (Compile-time Round-Robin)
Chunk 0 -> Thread 0 | Chunk 1 -> Thread 1 | Chunk 2 -> Thread 2 | Chunk 3 -> Thread 0 ...

DYNAMIC SCHEDULING (Runtime Work Queue)
[ Work Queue: Chunk 0, 1, 2, 3, 4, ... ] ---> Assigned to first available idle thread

GUIDED SCHEDULING (Exponentially Decaying Chunks)
[ Chunk 1: Size 100 ] -> [ Chunk 2: Size 75 ] -> [ Chunk 3: Size 56 ] ... -> [ Min Chunk: 8 ]
```

### Detailed Evaluation:

#### 1. `schedule(static [, chunk])`
- **Mechanism:** Iterations are partitioned into fixed chunks of size `chunk` and assigned to threads in round-robin order before loop execution starts.
- **Overhead:** Near-zero runtime overhead; excellent cache locality.
- **Limitation:** Severe load imbalance if work per iteration varies.
- **Best For:** Perfectly uniform calculations (e.g., vector addition, matrix initialization).

#### 2. `schedule(dynamic [, chunk])`
- **Mechanism:** Chunks are placed in a shared dynamic work queue. As soon as a thread finishes its chunk, it atomically grabs the next chunk from the queue.
- **Overhead:** High lock and atomic overhead per chunk dispatch.
- **Best For:** Highly irregular, unpredictable workloads (e.g., Mandelbrot set rendering, sparse matrix calculations).

#### 3. `schedule(guided [, min_chunk])`
- **Mechanism:** Starts with large chunks to minimize scheduling overhead; chunk sizes shrink exponentially as remaining work decreases:
  $$\text{chunk} = \max\left( \left\lceil \frac{\text{Remaining Work}}{P} \right\rceil, \; \text{min\_chunk} \right)$$
- **Overhead:** Balanced; low overhead initially, dynamic load balancing near the end.
- **Best For:** Loops with decaying or monotonically decreasing/increasing workloads (e.g., triangular matrix solves).

#### 4. `schedule(auto)`
- **Mechanism:** Delegates the scheduling decision entirely to the compiler and OpenMP runtime heuristics.

---

## Question 6: OpenMP Latent Concurrency Bugs & Vectorization

> **Question:**  
> What is a "Latent Concurrency Bug" in OpenMP SIMD programming? Provide a C code snippet demonstrating how a missing data-scoping clause passes testing under `-O0` but causes catastrophic failure when compiled with `-O3 -march=native`.

---

### Model Answer:

A **Latent Concurrency Bug** is a synchronization or data-scoping defect that remains dormant during unoptimized debug builds (`-O0`) because memory access timings mask the race condition, but manifests as data corruption or incorrect results when the compiler optimizes or vectorizes code (`-O3`).

#### Defective Code Example:
```c
#include <stdio.h>
#include <omp.h>

#define N 1000

int main() {
    double a[N], b[N], c[N];
    double tmp; // INTENDED AS TEMPORARY SCRATCHPAD

    // Initialize arrays
    for (int i = 0; i < N; i++) { a[i] = i * 1.0; b[i] = i * 2.0; }

    // DANGEROUS PRAGMA: tmp is accidentally SHARED!
    #pragma omp parallel for
    for (int i = 0; i < N; i++) {
        tmp = a[i] * 2.5;         // Step 1: Write to shared tmp
        c[i] = tmp + b[i];        // Step 2: Read from shared tmp
    }

    printf("c[500] = %f\n", c[500]);
    return 0;
}
```

#### Why it passes at `-O0`:
Under `-O0`, the compiler maps `tmp` to a physical memory stack location. Thread execution interleaving on small datasets may occasionally align such that writes and reads complete sequentially before another thread touches `tmp`.

#### Why it fails catastrophically at `-O3 -march=native`:
When compiled with vectorization flags (`-O3 -fopenmp -mavx2`):
1. The compiler attempts to unroll the loop and issue SIMD vector instructions across multiple threads simultaneously.
2. Multiple CPU cores simultaneously overwrite the memory address of `tmp`.
3. Thread $A$ writes its intermediate product to `tmp`, but before it can compute `c[i]`, Thread $B$ overwrites `tmp` with its own product. Thread $A$ reads the corrupt value, producing silent numerical corruption.

#### The Correction:
Explicitly scope `tmp` as thread-private:
```c
#pragma omp parallel for private(tmp)
// OR better: declare tmp locally inside the loop scope
```

---

## Question 7: MPI Point-to-Point Deadlocks & Resolution

> **Question:**  
> (a) Illustrate the classical MPI bidirectional ping-pong deadlock scenario using standard `MPI_Send` and `MPI_Recv`.  
> (b) Explain why the deadlock occurs under the Rendezvous protocol.  
> (c) Provide two distinct programming techniques to eliminate the deadlock.

---

### Model Answer:

#### (a) Deadlock Code Pattern:
```c
// DEADLOCK SCENARIO (Circular Dependency)
if (rank == 0) {
    MPI_Send(sendbuf, count, MPI_INT, 1, 0, MPI_COMM_WORLD);
    MPI_Recv(recvbuf, count, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
} else if (rank == 1) {
    MPI_Send(sendbuf, count, MPI_INT, 0, 0, MPI_COMM_WORLD);
    MPI_Recv(recvbuf, count, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
}
```

```
               THE MPI CIRCULAR DEADLOCK STATE

+-----------------------+                         +-----------------------+
|       Rank 0          |                         |       Rank 1          |
+-----------------------+                         +-----------------------+
| Calling MPI_Send()    | ----------------------> | Calling MPI_Send()    |
| (Blocked waiting for  |                         | (Blocked waiting for  |
|  matching Recv on R1) | <---------------------- |  matching Recv on R0) |
+-----------------------+                         +-----------------------+
           |                                                 |
           v                                                 v
   [ NEVER REACHED ]                                 [ NEVER REACHED ]
    MPI_Recv(...)                                     MPI_Recv(...)
```

#### (b) Why Deadlock Occurs Under the Rendezvous Protocol:
1. For small messages below the **Eager Threshold**, MPI buffers the message in internal memory and returns from `MPI_Send` immediately, masking the bug.
2. For large messages (above eager limit), `MPI_Send` switches to the **Rendezvous Protocol**. It refuses to return until a matching `MPI_Recv` has been posted and acknowledged by the destination node.
3. Because both Rank 0 and Rank 1 call `MPI_Send` first, both block indefinitely waiting for the other to post `MPI_Recv`. Neither process can ever advance to reach its receive call, resulting in a **Permanent Distributed Deadlock**.

#### (c) Two Solutions to Eliminate the Deadlock:

#### Solution 1: Odd-Even Interleaved Pairing
Reverse the call order on one of the ranks:
```c
if (rank == 0) {
    MPI_Send(sendbuf, count, MPI_INT, 1, 0, MPI_COMM_WORLD);
    MPI_Recv(recvbuf, count, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
} else if (rank == 1) {
    MPI_Recv(recvbuf, count, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE); // Recv FIRST!
    MPI_Send(sendbuf, count, MPI_INT, 0, 0, MPI_COMM_WORLD);
}
```

#### Solution 2: Combined Atomic Exchange (`MPI_Sendrecv`)
Use `MPI_Sendrecv` to execute the send and receive simultaneously in an internally optimized non-blocking schedule:
```c
MPI_Sendrecv(
    sendbuf, count, MPI_INT, (rank == 0 ? 1 : 0), 0,
    recvbuf, count, MPI_INT, (rank == 0 ? 1 : 0), 0,
    MPI_COMM_WORLD, MPI_STATUS_IGNORE
);
```

---

## Question 8: MPI Dynamic Client-Server Connections & `ompi-server`

> **Question:**  
> Explain the architectural mechanisms provided by MPI-2 to connect two independently launched MPI programs across a network. Detail the functions of `MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect`, and explain how `ompi-server` provides a DNS-like lookup registry.

---

### Model Answer:

In standard MPI-1, all processes must be launched simultaneously within a single `mpirun` instance under a shared `MPI_COMM_WORLD`. MPI-2 introduces **Dynamic Process Management** enabling independently launched programs to establish communication dynamically.

#### Handshake Sequence & Key Primitives:
1. **Port Allocation (`MPI_Open_port`):**
   - The server process calls `MPI_Open_port(MPI_INFO_NULL, port_string)` to bind a network endpoint. The runtime returns an opaque address string (up to `MPI_MAX_PORT_NAME`) containing host IPs and communication tokens.
2. **Port Publication:**
   - The server exports the port string so external clients can locate it. This can be done via a shared file (`port.txt`) or by registering with `ompi-server`.
3. **Listening & Connection (`MPI_Comm_accept` & `MPI_Comm_connect`):**
   - **Server:** Calls `MPI_Comm_accept(port_string, ..., &inter_comm)` and blocks listening.
   - **Client:** Reads the port string and calls `MPI_Comm_connect(port_string, ..., &inter_comm)`.
   - The handshake completes, returning an **Inter-Communicator** (`inter_comm`) spanning the two disjoint process groups.
4. **Data Exchange:**
   - Both sides communicate using standard `MPI_Send` and `MPI_Recv` across the inter-communicator. In an inter-communicator, rank indices reference the peer process in the remote group.
5. **Teardown:**
   - Channels are closed via `MPI_Comm_disconnect(&inter_comm)` followed by `MPI_Close_port(port_string)`.

#### The Role of `ompi-server`:
In distributed environments lacking a shared filesystem, `ompi-server` acts as a **centralized directory lookup service** (analogous to DNS in networking):
- **DNS Analogy:** DNS resolves a human-readable domain name (e.g., `google.com`) to an IP address. Similarly, `ompi-server` maps an application service name (e.g., `"PhysicsEngine"`) to its internal MPI port string.
- **Server Registration:** Calls `MPI_Publish_name("PhysicsEngine", MPI_INFO_NULL, port_name)`.
- **Client Resolution:** Calls `MPI_Lookup_name("PhysicsEngine", MPI_INFO_NULL, port_name)`.
- **CLI Daemon Launch:**
  ```bash
  ompi-server --no-daemonize -report-uri /tmp/ompi-server.uri
  mpirun --mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./server
  ```

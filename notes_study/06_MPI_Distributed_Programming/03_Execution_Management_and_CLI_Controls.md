# MPI Execution Management, Process Mapping & CLI Controls

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 06 — Distributed-Memory Programming with MPI
SUBTOPIC:               03 — Execution Management, Process Mapping & CLI Controls
PREREQUISITES:          Subtopic 06.1 (MPI Lifecycle), Subtopic 06.2 (Point-to-Point)
SOURCE LECTURE NOTES:   IMG_1333.pdf (Pages 20–21)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Operational Reality of Distributed Execution

In traditional single-process and shared-memory programming, running an application is simple: you compile with `gcc` and run `./my_program`. The operating system kernel loads the binary, creates a virtual memory space, and dispatches threads to CPU cores.

In distributed-memory MPI computing, **execution management is radically different**:
- You may be launching $1,000$ processes across $64$ independent server racks connected via high-speed network switches.
- How does the system distribute processes across nodes, sockets, and physical CPU cores?
- What happens if you try to spawn 20 processes on a quad-core laptop?
- How do you empirically verify that MPI processes are truly independent operating system processes with separate virtual address spaces rather than threads?

In this subtopic, you will master the **runtime execution architecture** of MPI, the mechanics of the `mpirun`/`mpiexec` process orchestrator, essential command-line controls (`--display-allocation`, `--oversubscribe`, hostfiles), and Linux process table inspection.

---

## 2. Process Identification: The `getpid()` Diagnostic Proof

In Module 04, we saw that all Pthreads within a process share the same Process ID (`PID`).

How do we prove that MPI uses the **Shared-Nothing Multi-Process Model**?

As presented in your lecture notes (`IMG_1333.pdf`, Page 21), we invoke the POSIX system call `getpid()` from `<unistd.h>` inside every MPI rank:

```c
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    int rank, size;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // getpid() queries the Linux kernel for the calling process's unique OS Process ID
    printf("Hello from Rank %d out of %d | OS PID = %d\n", rank, size, getpid());
    fflush(stdout);

    // sleep(60) keeps processes alive so we can inspect them in the Linux process table
    sleep(60);

    MPI_Finalize();
    return 0;
}
```

### The Terminal Output
When compiled and run with `mpirun -np 4 ./mpi_diag`:
```text
Hello from Rank 0 out of 4 | OS PID = 41205
Hello from Rank 1 out of 4 | OS PID = 41206
Hello from Rank 2 out of 4 | OS PID = 41207
Hello from Rank 3 out of 4 | OS PID = 41208
```

$$\boxed{\textbf{The Proof of Isolation: } \text{Every single MPI rank is a \textbf{completely independent OS process}}}$$
$$\boxed{\text{possessing its own unique PID, its own private page tables, and its own heap/stack memory!}}$$

---

### Inspecting the Linux Process Table (`ps aux`)

While the processes are sleeping during `sleep(60)`, open a second terminal and query the Linux operating system process table:

```bash
ps aux | grep mpi_diag
```

```text
USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
abhishek 41205   0.0  0.1 142380  5210 pts/1    S+   21:05   0:00 ./mpi_diag
abhishek 41206   0.0  0.1 142380  5208 pts/1    S+   21:05   0:00 ./mpi_diag
abhishek 41207   0.0  0.1 142380  5212 pts/1    S+   21:05   0:00 ./mpi_diag
abhishek 41208   0.0  0.1 142380  5204 pts/1    S+   21:05   0:00 ./mpi_diag
```
This confirms that the operating system treats them as four separate, isolated programs. If Rank 1 crashes with a Segmentation Fault, Ranks 0, 2, and 3's memory remains completely uncorrupted.

---

## 3. The `mpirun` / `mpiexec` Process Orchestrator Architecture

When you type `mpirun -np 4 ./a.out`, what actually happens under the hood?

In modern HPC implementations (such as **Open MPI**), `mpirun` launches the **Open Run-Time Environment (ORTE)**:

```
========================================================================================
                  MPIRUN / PRRTE ORCHESTRATION ARCHITECTURE
========================================================================================

                               [ User Terminal ]
                                      |
                     Types: mpirun -np 4 ./simulation
                                      |
                                      v
                        +---------------------------+
                        |  HEAD DAEMON (mpirun /    |
                        |  Open RTE Controller)     |
                        +-------------+-------------+
                                      |
               Dispatches launch commands via SSH / SLURM
                                      |
             +------------------------+------------------------+
             |                                                 |
             v                                                 v
  [ Compute Node 1 ]                                [ Compute Node 2 ]
  +-------------------------------+                 +-------------------------------+
  | Local Node Daemon (orted)     |                 | Local Node Daemon (orted)     |
  |                               |                 |                               |
  | Spawns:                       |                 | Spawns:                       |
  | - Rank 0 (PID 41205) on Core 0|                 | - Rank 2 (PID 18021) on Core 0|
  | - Rank 1 (PID 41206) on Core 1|                 | - Rank 3 (PID 18022) on Core 1|
  +-------------------------------+                 +-------------------------------+
========================================================================================
```

1. **The Controller Daemon:** `mpirun` initializes and reads cluster configuration files, detecting available cores and network interfaces.
2. **Remote Dispatch:** Connects to remote compute nodes via Secure Shell (`ssh`) or job schedulers (SLURM, PBS, LSF).
3. **Local Node Daemons (`orted`):** Launches a background daemon (`orted`) on each compute node.
4. **Process Spawning & Binding:** The local daemon uses POSIX `fork()` and `execve()` to launch the application binary, binding processes to specific physical CPU cores using OS affinity masks (`sched_setaffinity`).
5. **I/O Redirection (Multiplexing):** Captures `stdout` and `stderr` streams from all distributed processes, tags them, and streams them back across the network to display in the user's local terminal.

---

## 4. Advanced `mpirun` Command-Line Controls

As detailed in your lecture notes (`IMG_1333.pdf`, Page 21), mastering the command-line flags of `mpirun` is essential for production execution and laboratory examinations:

```
+-----------------------------------------------------------------------------------+
|                        ESSENTIAL MPIRUN COMMAND-LINE FLAGS                        |
+-----------------------------------------------------------------------------------+
| Flag / Option                | Purpose & Operational Behavior                     |
| :--------------------------- | :------------------------------------------------- |
| **`-np N`** or **`-n N`**    | Spawns exactly $N$ processes across the cluster.   |
| **`--display-allocation`**   | Displays the detected hardware slots, nodes, and   |
|                              | resource allocations before launching.             |
| **`--oversubscribe`**        | Overrides core-count safety checks; permits running|
|                              | more MPI processes than physical hardware slots.   |
| **`--hostfile <file>`**      | Points to a cluster node list (hostnames & slots). |
| **`-host <node1,node2>`**    | Comma-separated list of nodes to target directly.  |
| **`--bind-to <core/socket>`**| Explicitly binds processes to specific cores/NUMA. |
| **`--map-by <core/socket>`** | Dictates process distribution order (round-robin).  |
+-----------------------------------------------------------------------------------+
```

---

### Control 1: Inspecting Resource Allocation (`--display-allocation`)

```bash
mpirun --display-allocation -np 4 ./mpi_diag
```

Instructs the Open MPI runtime to print a detailed diagnostic summary of the detected hardware environment prior to executing:
```text
====================== ALLOCATED HOSTS ======================
    node01: slots=2 max_slots=0 slots_inuse=0 state=UP
    node02: slots=2 max_slots=0 slots_inuse=0 state=UP
=============================================================
```
This confirms that Node 1 and Node 2 were successfully contacted, each allocating 2 physical core slots.

---

### Control 2: The Oversubscription Trap & `--oversubscribe`

In high-performance computing, the golden rule is **One MPI Process Per Physical CPU Core**. 

By default, modern Open MPI strictly prevents users from launching more processes than the system has hardware processing units:
- Suppose your laptop has **4 CPU cores**.
- If you run: `mpirun -np 8 ./my_program`
- Open MPI **halts execution immediately** and prints a fatal error:

```text
--------------------------------------------------------------------------
There are not enough slots available in the system to satisfy the 8
slots that were requested by the application:
  ./my_program

Either request fewer slots for your application, or make more slots available
for use by setting the --oversubscribe option.
--------------------------------------------------------------------------
```

#### Why Does Open MPI Reject Oversubscription by Default?
Because MPI processes perform **active polling (busy-waiting)** in their network communication loops (`MPI_Recv` and `MPI_Send`). If 8 CPU-hungry processes fight over 4 physical cores:
1. The CPU cores thrash violently between processes via OS context switches.
2. Interconnect latency skyrockets by $1,000\times$.
3. Cache lines are continuously evicted, destroying performance.

#### When to Use `--oversubscribe`
During development, testing, and university lab assignments, you frequently need to simulate a 16-node or 32-node cluster on a standard 4-core student workstation.

To force Open MPI to bypass the safety check, pass **`--oversubscribe`**:

```bash
mpirun --oversubscribe -np 16 ./my_program
```
Open MPI multiplexes all 16 processes onto the 4 cores using time-sharing, allowing you to debug distributed algorithms locally without owning a supercomputer.

---

## 5. Multi-Node Cluster Execution with Hostfiles

To run an MPI application across physical cluster computers in a university laboratory or data center, you use a **Hostfile**.

### What is a Hostfile?
A simple plain-text file listing the IP addresses or network hostnames of cluster computers along with their available core capacity (**slots**):

```text
# cluster_hosts.txt
192.168.1.101 slots=4
192.168.1.102 slots=4
192.168.1.103 slots=8
node04.cluster.lan slots=8
```
- `slots=4` informs `mpirun`: *"Node 1 has 4 physical cores available for MPI ranks."*

### Launching Across the Cluster

```bash
mpirun --hostfile cluster_hosts.txt -np 24 ./simulation
```
`mpirun` reads the hostfile, contacts the four machines via SSH, and distributes the 24 process ranks across them:
- Ranks 0–3 execute on Node 101.
- Ranks 4–7 execute on Node 102.
- Ranks 8–15 execute on Node 103.
- Ranks 16–23 execute on Node 04.

---

## 6. Process Mapping & Binding Policies: Core vs Socket

How should processes be distributed across multi-socket, multi-core motherboards?

Consider a dual-socket server with **2 Sockets**, each containing **8 Cores** (16 cores total). You wish to run 8 processes.

```
========================================================================================
                      PROCESS MAPPING: BY CORE vs BY SOCKET
========================================================================================

1. MAPPING BY CORE (Default / Packed):
   mpirun --map-by core -np 8 ./a.out
   - Ranks 0, 1, 2, 3, 4, 5, 6, 7 are ALL packed onto Socket 0!
   - Socket 1 remains completely idle!
   - Advantage: Maximizes shared L3 cache locality for intra-socket communication.
   - Disadvantage: All 8 processes share and choke on Socket 0's memory bus!

2. MAPPING BY SOCKET (Round-Robin / Distributed):
   mpirun --map-by socket -np 8 ./a.out
   - Ranks 0, 2, 4, 6 -> Placed on Socket 0
   - Ranks 1, 3, 5, 7 -> Placed on Socket 1
   - Advantage: Doubles available memory bandwidth by utilizing both NUMA memory buses!
   - Disadvantage: Inter-process messages must traverse slower inter-socket QPI/UPI links.
========================================================================================
```

---

## 7. Complete Compilable C Program: Cluster Topology & Process Affinity Diagnostics

Here is the complete C program demonstrating process mapping, host identification, and OS process ID inspection:

```c
/**
 * ============================================================================
 * PROGRAM:     mpi_process_inspection.c
 * DESCRIPTION: Diagnoses MPI execution mapping across cluster nodes:
 *              - Queries OS Process ID (getpid)
 *              - Queries physical hostname (MPI_Get_processor_name)
 *              - Inspects process-to-core mapping
 * COMPILATION: mpicc -Wall -O2 mpi_process_inspection.c -o mpi_inspect
 * EXECUTION:   mpirun --display-allocation -np 4 ./mpi_inspect
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    int rank, size;
    char hostname[MPI_MAX_PROCESSOR_NAME];
    int name_len;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // Query physical host machine name
    MPI_Get_processor_name(hostname, &name_len);

    // Query OS Process ID from Linux kernel
    pid_t process_id = getpid();

    // Print diagnostic process profile
    printf("[Rank %2d of %2d] Host: %-15s | OS PID: %-6d\n", 
           rank, size, hostname, (int)process_id);
    fflush(stdout);

    // Synchronize output display across cluster
    MPI_Barrier(MPI_COMM_WORLD);

    if (rank == 0) {
        printf("\n[Master] All %d processes verified running in independent memory spaces.\n", size);
    }

    MPI_Finalize();
    return 0;
}
```

---

## 8. University Examination Preparation: Solved Questions

### Question 1: Execution Mechanics (10 Marks)
**Explain the role of the `mpirun` (or `mpiexec`) process launcher in distributed-memory computing. Describe the function of the `--oversubscribe` flag. Why does Open MPI prevent oversubscription by default?**

#### Model Examination Answer:
1. **Role of `mpirun`:**  
   In distributed-memory systems, applications cannot be launched directly as standard single-process binaries. `mpirun` acts as the cluster runtime orchestrator:
   - Reads cluster hostfiles and discovers available compute resources.
   - Uses remote daemons (via SSH or job schedulers like SLURM) to deploy executable binaries across remote cluster nodes.
   - Allocates processes to physical CPU cores, sets process affinity masks, and establishes network communication sockets between all ranks to initialize `MPI_COMM_WORLD`.
   - Aggregates and multiplexes `stdout` and `stderr` streams from all distributed nodes back to the user's local terminal.
2. **The `--oversubscribe` Flag:**  
   The `--oversubscribe` option instructs `mpirun` to bypass physical hardware core constraints, allowing a user to launch more MPI processes than there are available physical CPU cores/slots on the host machine.
3. **Why Open MPI Prevents Oversubscription by Default:**  
   MPI processes are designed for high-performance computing and utilize aggressive busy-polling in their network receive and send routines. If multiple MPI processes are assigned to a single CPU core, they aggressively compete for execution time. The operating system kernel is forced to perform thousands of context switches per second, invalidating CPU L1/L2 caches and introducing massive latency spikes that degrade parallel performance by orders of magnitude.

---

### Question 2: Diagnostic Verification (5 Marks)
**How can a software developer prove empirically that MPI processes execute in separate virtual memory spaces, whereas OpenMP threads execute in a shared virtual memory space?**

#### Model Examination Answer:
- **MPI Verification:**  
  Inside an MPI program, every rank invokes the POSIX system call `getpid()`. When executed across $N$ processes via `mpirun -np N`, the operating system outputs **$N$ distinct, unique Process IDs (PIDs)**. Furthermore, inspecting the process table with `ps aux | grep binary` displays $N$ independent process entries, each with its own private virtual memory space (`VSZ`).
- **OpenMP Verification:**  
  Inside an OpenMP parallel block running across $N$ threads, invoking `getpid()` from each thread outputs the **exact same Process ID** for all threads. This proves that all OpenMP threads exist within the boundaries of a single shared operating system process and share one virtual address space.

---

## 9. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 06.3 QUICK REVISION SHEET
================================================================================

1. MPI PROCESS ISOLATION:
   - Verified via getpid(): Every rank has a distinct Linux Process ID!
   - ps aux shows separate process entries with isolated memory spaces.

2. MPIRUN ORCHESTRATION:
   - Launches local/remote daemons (orted) via SSH or SLURM.
   - Binds ranks to cores; multiplexes distributed stdout back to user terminal.

3. ADVANCED MPIRUN CLI CONTROLS:
   - --display-allocation: Prints detected hardware cores and node topology.
   - --oversubscribe: Bypasses slot limits; allows running 16 ranks on a 4-core PC.
   - --hostfile <file>: Distributes ranks across multi-node cluster machines.
   - --map-by socket: Distributes ranks round-robin across NUMA sockets.

4. GOLDEN HPC RULE:
   - Exactly 1 MPI process per physical CPU core for maximum performance!
================================================================================
```

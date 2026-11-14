# MPI Point-to-Point Communication: Send, Receive, Protocols & Deadlocks

```
================================================================================
COURSE:                 Parallel Computing & High-Performance Systems
MODULE:                 06 — Distributed-Memory Programming with MPI
SUBTOPIC:               02 — Point-to-Point Communication: Send & Receive
PREREQUISITES:          Subtopic 06.1 (MPI Architecture & Process Lifecycle)
SOURCE LECTURE NOTES:   IMG_1333.pdf (Pages 19–20)
TARGET AUDIENCE:        Computer Science & Engineering (University Level)
================================================================================
```

---

## 1. Introduction: The Point-to-Point Communication Paradigm

In Subtopic 06.1, we discovered the fundamental constraint of distributed-memory computing: **Processes do not share physical memory**.

If Process 0 on Node A computes an array of numbers that Process 1 on Node B needs, Process 0 cannot write into Process 1's RAM. Instead, Process 0 must package the data into a network message and transmit it explicitly to Process 1.

$$\boxed{\textbf{Point-to-Point Communication: } \text{A directed, pairwise data transmission involving exactly}}$$
$$\boxed{\text{\textbf{two processes}: exactly one designated \textbf{Sender} and exactly one designated \textbf{Receiver}.}}$$

```
========================================================================================
                     POINT-TO-POINT COMMUNICATION ARCHITECTURE
========================================================================================

    [ PROCESS 0 (Sender) ]                                  [ PROCESS 1 (Receiver) ]
  +------------------------+                              +------------------------+
  | Application Memory:    |                              | Application Memory:    |
  | int data = 50;         |                              | int data = ???;        |
  +-----------+------------+                              +------------^-----------+
              |                                                        |
      MPI_Send(&data, ...)                                     MPI_Recv(&data, ...)
              |                                                        |
              v                                                        |
  +------------------------+       NETWORK MESSAGE ENVELOPE    +------------------------+
  |  Network Socket / NIC  |  ==>  [ Payload: 50           ]  ==>  Network Socket / NIC  |
  |  Send Buffer           |       [ Sender Rank: 0        ]   |  Receive Buffer        |
  +------------------------+       [ Dest Rank:   1        ]   +------------------------+
                                   [ Tag ID:      0        ]
                                   [ Context:     WORLD    ]
========================================================================================
```

---

## 2. The `MPI_Send()` API Specification

The primary function used to transmit data to another process is `MPI_Send()`:

```c
int MPI_Send(
    const void *buf,       // 1. Initial memory address of data payload
    int count,             // 2. Number of elements to transmit
    MPI_Datatype datatype, // 3. MPI Datatype identifier
    int dest,              // 4. Rank of destination process
    int tag,               // 5. Message envelope tag (classification ID)
    MPI_Comm comm          // 6. Communicator context
);
```

```
+-------------------------------------------------------------------------------+
|                       MPI_SEND PARAMETER DISSECTION                           |
+-------------------------------------------------------------------------------+
| 1. const void *buf                                                            |
|    - Memory pointer to the beginning of the data payload.                     |
|    - Can point to a single scalar (`&data`) or an array (`array`).            |
|                                                                               |
| 2. int count                                                                  |
|    - The number of data elements being transmitted (NOT the byte count!).     |
|    - For an array of 5 integers, `count = 5` (MPI calculates bytes internally)|
|                                                                               |
| 3. MPI_Datatype datatype                                                      |
|    - The specific MPI data type identifier (e.g., `MPI_INT`, `MPI_DOUBLE`).   |
|    - Ensures cross-platform data conversion if machines use different byte    |
|      endianness (Big-Endian vs Little-Endian).                                |
|                                                                               |
| 4. int dest                                                                   |
|    - The unique rank of the target process in communicator `comm`.            |
|    - Must be a valid rank: $0 \le \text{dest} < \text{size}$.                 |
|                                                                               |
| 5. int tag                                                                    |
|    - An integer "subject line" or message category ($0 \le \text{tag} \le 32767$).|
|    - Allows a receiver to distinguish between different types of messages    |
|      (e.g., Tag 1 = Sensor Data, Tag 2 = Shutdown Signal).                    |
|                                                                               |
| 6. MPI_Comm comm                                                              |
|    - The communicator context (typically `MPI_COMM_WORLD`).                   |
+-------------------------------------------------------------------------------+
```

---

## 3. The `MPI_Recv()` API Specification

To receive the message sent by `MPI_Send()`, the target process must post an matching `MPI_Recv()`:

```c
int MPI_Recv(
    void *buf,             // 1. Initial memory address where data is stored
    int count,             // 2. Maximum capacity of receive buffer
    MPI_Datatype datatype, // 3. MPI Datatype identifier
    int source,            // 4. Rank of sending process (or MPI_ANY_SOURCE)
    int tag,               // 5. Matching message tag (or MPI_ANY_TAG)
    MPI_Comm comm,         // 6. Communicator context
    MPI_Status *status     // 7. Output: Metadata status object (or MPI_STATUS_IGNORE)
);
```

```
+-------------------------------------------------------------------------------+
|                       MPI_RECV PARAMETER DISSECTION                           |
+-------------------------------------------------------------------------------+
| 1. void *buf                                                                  |
|    - Memory buffer where incoming network bytes are unpacked.                 |
|                                                                               |
| 2. int count                                                                  |
|    - The MAXIMUM number of elements the receive buffer can hold.              |
|    - If the incoming message contains more elements than `count`, an          |
|      **MPI Buffer Overflow** error (`MPI_ERR_TRUNCATE`) is triggered!         |
|                                                                               |
| 3. int source                                                                 |
|    - The rank of the expected sender.                                         |
|    - Can specify an exact rank (e.g., `0`) or wildcard `MPI_ANY_SOURCE`.      |
|                                                                               |
| 4. int tag                                                                    |
|    - Matches messages with this exact tag ID.                                 |
|    - Can specify an exact tag (e.g., `0`) or wildcard `MPI_ANY_TAG`.          |
|                                                                               |
| 5. MPI_Status *status                                                         |
|    - Pointer to an `MPI_Status` struct containing metadata about the message. |
|    - If metadata is not needed, pass `MPI_STATUS_IGNORE`.                     |
+-------------------------------------------------------------------------------+
```

---

## 4. Standard MPI Datatypes vs C Datatypes

Because cluster nodes may run on different CPU architectures (e.g., x86-64 vs ARM64) with different data alignments and byte orders, **you cannot pass raw C types to MPI functions**. You must use standardized **MPI Datatypes**:

```
+-----------------------------------------------------------------------------------+
|                           MPI DATATYPE MAPPING TABLE                              |
+-----------------------------------------------------------------------------------+
| C Primitive Data Type         | Corresponding MPI Datatype Identifier             |
| :---------------------------- | :------------------------------------------------ |
| `char`                        | `MPI_CHAR`                                        |
| `signed short int`            | `MPI_SHORT`                                       |
| `signed int`                  | `MPI_INT`                                         |
| `signed long int`             | `MPI_LONG`                                        |
| `unsigned int`                | `MPI_UNSIGNED`                                    |
| `float`                       | `MPI_FLOAT`                                       |
| `double`                      | `MPI_DOUBLE`                                      |
| `unsigned char` (raw byte)    | `MPI_BYTE`                                        |
+-----------------------------------------------------------------------------------+
```

---

## 5. Wildcards & The `MPI_Status` Structure

In real-world master-worker architectures, a master process often does not know **which worker will finish first**, or **what tag a message carries**.

OpenMP provides two special **Wildcard Values**:
1. **`source = MPI_ANY_SOURCE`:** Receive a message from **any process** in the communicator. Whichever process transmits first is accepted.
2. **`tag = MPI_ANY_TAG`:** Accept a message carrying **any tag identifier**.

### Decoding `MPI_Status`
When wildcards are used, how does the receiving process find out who sent the message, what tag was attached, and how many elements were transmitted?

It inspects the **`MPI_Status` struct**:

```c
MPI_Status status;
MPI_Recv(buffer, 100, MPI_INT, MPI_ANY_SOURCE, MPI_ANY_TAG, MPI_COMM_WORLD, &status);

// 1. Query Sender's Rank:
int sender_rank = status.MPI_SOURCE;

// 2. Query Attached Tag:
int message_tag = status.MPI_TAG;

// 3. Query Actual Elements Received via MPI_Get_count():
int actual_count;
MPI_Get_count(&status, MPI_INT, &actual_count);

printf("Received %d integers from Rank %d with Tag %d\n", 
       actual_count, sender_rank, message_tag);
```

---

## 6. Under the Hood: Eager vs Rendezvous Communication Protocols

`MPI_Send()` is a **Blocking Send**. But what does "blocking" actually mean in physical hardware?

The MPI runtime implements point-to-point communication using two internal protocols:

```
========================================================================================
                     EAGER PROTOCOL vs RENDEZVOUS PROTOCOL
========================================================================================

1. EAGER PROTOCOL (For Tiny / Short Messages, typically < 64 KB):
   Sender Node                                          Receiver Node
   MPI_Send(&msg) 
        |
        v
   [ Copies data into local NIC buffer ]
   [ Transmits packet over network immediately ] =====> [ Stores in temporary buffer ]
        |
   MPI_Send RETURNS IMMEDIATELY!
   (Sender continues computing without waiting for receiver to call MPI_Recv!)

2. RENDEZVOUS PROTOCOL (For Large / Bulk Messages, typically > 64 KB):
   Sender Node                                          Receiver Node
   MPI_Send(&big_array)
        |
   [ Transmits Handshake Request "Ready to send?" ] ==>
                                                   <== [ Acknowledges "I posted Recv!" ]
   [ Direct Memory-to-Memory RDMA Transfer ] ========> [ Unpacks directly into buf ]
        |
   MPI_Send BLOCKS until Receiver acknowledges receipt!
========================================================================================
```

$$\boxed{\textbf{The Definition of Blocking: } \text{\texttt{MPI\_Send} blocks until the application's send buffer}}$$
$$\boxed{\text{is \textbf{safe to overwrite or reuse}. It does NOT mean the receiver has received it!}}$$

---

## 7. The Classic Communication Deadlock

Because `MPI_Send` and `MPI_Recv` are blocking, improper message sequencing causes **Deadlocks**.

### The Recv-Before-Send Deadlock (The Stare-Down)

Consider two processes attempting to exchange data:

```c
// CATASTROPHIC DEADLOCK:
if (rank == 0) {
    MPI_Recv(recv_buf, 10, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE); // BLOCKS waiting for Rank 1
    MPI_Send(send_buf, 10, MPI_INT, 1, 0, MPI_COMM_WORLD);
} 
else if (rank == 1) {
    MPI_Recv(recv_buf, 10, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE); // BLOCKS waiting for Rank 0
    MPI_Send(send_buf, 10, MPI_INT, 0, 0, MPI_COMM_WORLD);
}
```

- Process 0 waits for Process 1 to send.
- Process 1 waits for Process 0 to send.
- Neither process can ever reach its `MPI_Send()` call!
- **Result:** Both processes sleep forever in kernel network queues. The supercomputer cluster hangs permanently.

---

### The Three Solutions to Deadlock

#### Solution 1: Alternating Send and Receive Orders
Ensure one process sends first while the other receives first:
```c
if (rank == 0) {
    MPI_Send(send_buf, 10, MPI_INT, 1, 0, MPI_COMM_WORLD); // Sends first
    MPI_Recv(recv_buf, 10, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
} else if (rank == 1) {
    MPI_Recv(recv_buf, 10, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE); // Receives first
    MPI_Send(send_buf, 10, MPI_INT, 0, 0, MPI_COMM_WORLD);
}
```

#### Solution 2: Combined `MPI_Sendrecv()`
OpenMP provides a combined atomic send-and-receive primitive that eliminates deadlock risk:
```c
MPI_Sendrecv(send_buf, 10, MPI_INT, dest, 0,
             recv_buf, 10, MPI_INT, src,  0,
             MPI_COMM_WORLD, MPI_STATUS_IGNORE);
```

#### Solution 3: Non-Blocking Communication (`MPI_Isend` / `MPI_Irecv`)
Initiates communication asynchronously and returns immediately, using `MPI_Wait()` later to synchronize.

---

## 8. Complete Compilable C Program: Ping-Pong Benchmark

Here is the complete C program implementing the classic **Ping-Pong Benchmark** between Rank 0 and Rank 1 transcribed and enhanced from your lecture notes (`IMG_1333.pdf`, Page 19):

```c
/**
 * ============================================================================
 * PROGRAM:     mpi_ping_pong.c
 * DESCRIPTION: Point-to-Point message passing benchmark between Rank 0 and Rank 1.
 *              Demonstrates MPI_Send, MPI_Recv, MPI_Status inspection,
 *              and wall-clock round-trip latency measurement.
 * COMPILATION: mpicc -Wall -O2 mpi_ping_pong.c -o mpi_ping_pong
 * EXECUTION:   mpirun -np 2 ./mpi_ping_pong
 * ============================================================================
 */

#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

#define PING_PONG_LIMIT 5

int main(int argc, char *argv[]) {
    int rank, size;
    int ping_pong_count = 0;
    int partner_rank;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // This benchmark strictly requires exactly 2 processes
    if (size != 2) {
        if (rank == 0) {
            fprintf(stderr, "ERROR: Ping-Pong benchmark requires exactly 2 processes!\n");
        }
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }

    // Determine partner process rank: 0 talks to 1, 1 talks to 0
    partner_rank = (rank == 0) ? 1 : 0;

    double start_time = MPI_Wtime();

    while (ping_pong_count < PING_PONG_LIMIT) {
        if (rank == ping_pong_count % 2) {
            // It is MY TURN to increment and SEND
            ping_pong_count++;
            
            // MPI_Send(&buf, count, datatype, dest, tag, comm)
            MPI_Send(&ping_pong_count, 1, MPI_INT, partner_rank, 0, MPI_COMM_WORLD);
            printf("[Process %d] Sent ping_pong_count = %d to Process %d\n", 
                   rank, ping_pong_count, partner_rank);
        } else {
            // It is MY TURN to RECEIVE
            MPI_Status status;
            
            // MPI_Recv(&buf, count, datatype, source, tag, comm, &status)
            MPI_Recv(&ping_pong_count, 1, MPI_INT, partner_rank, 0, MPI_COMM_WORLD, &status);
            
            // Inspect status metadata
            int actual_elements;
            MPI_Get_count(&status, MPI_INT, &actual_elements);
            
            printf("  [Process %d] Received count = %d from Sender %d (Elements: %d)\n", 
                   rank, ping_pong_count, status.MPI_SOURCE, actual_elements);
        }
    }

    double elapsed_time = MPI_Wtime() - start_time;

    if (rank == 0) {
        printf("\n================================================================\n");
        printf("Ping-Pong exchange reached limit of %d trips successfully!\n", PING_PONG_LIMIT);
        printf("Total Elapsed Time: %.6f seconds\n", elapsed_time);
        printf("Average One-Way Latency: %.2f microseconds\n", 
               (elapsed_time / (double)PING_PONG_LIMIT) * 1e6);
        printf("================================================================\n");
    }

    MPI_Finalize();
    return 0;
}
```

---

## 9. University Examination Preparation: Solved Questions

### Question 1: Core Mechanics (10 Marks)
**Dissect the parameter lists of `MPI_Send()` and `MPI_Recv()`. Explain the role of the message `tag` and `MPI_Datatype`. Why does MPI require explicit datatypes instead of raw byte pointers?**

#### Model Examination Answer:
1. **Parameter Dissection:**  
   *(Provide the 6 parameters of `MPI_Send` and 7 parameters of `MPI_Recv` detailed in Sections 2 and 3).*
2. **Role of the Message Tag:**  
   The tag is an integer identifier ($0$ to $32767$) attached to the message envelope. It serves as a subject classifier, allowing receivers to selectively receive specific types of data (e.g., Tag 1 for matrix rows, Tag 2 for boundary halos, Tag 99 for termination commands) even if they arrive on the same communicator.
3. **Why MPI Requires Explicit Datatypes:**  
   In a heterogeneous cluster, nodes may run on different hardware microarchitectures with conflicting internal data representations (e.g., Big-Endian vs. Little-Endian byte order, 32-bit vs. 64-bit word alignment). If MPI transferred raw byte streams (`void *`), data would be corrupted upon receipt on an architecture with opposite endianness. Explicit datatypes (`MPI_INT`, `MPI_DOUBLE`) enable the MPI communication engine to automatically convert byte orders and format numbers during network transit.

---

### Question 2: Deadlock Analysis (5 Marks)
**Analyze the following MPI code snippet executing on 2 processes. Identify the concurrency defect and explain how to resolve it.**

```c
if (rank == 0) {
    MPI_Recv(buf1, 100, MPI_INT, 1, 0, MPI_COMM_WORLD, &status);
    MPI_Send(buf2, 100, MPI_INT, 1, 0, MPI_COMM_WORLD);
} else if (rank == 1) {
    MPI_Recv(buf2, 100, MPI_INT, 0, 0, MPI_COMM_WORLD, &status);
    MPI_Send(buf1, 100, MPI_INT, 0, 0, MPI_COMM_WORLD);
}
```

#### Model Examination Answer:
1. **The Defect (Recv-Before-Send Deadlock):**  
   Both Process 0 and Process 1 call `MPI_Recv()` first. Because `MPI_Recv()` is a blocking call, Process 0 halts and waits for Process 1 to send data, while Process 1 halts and waits for Process 0 to send data. Neither process can ever reach its respective `MPI_Send()` call. Both processes remain blocked forever in a permanent distributed deadlock.
2. **Resolution:**  
   Invert the communication sequence on one process so that sends and receives alternate:
   ```c
   if (rank == 0) {
       MPI_Send(buf2, 100, MPI_INT, 1, 0, MPI_COMM_WORLD); // Send first
       MPI_Recv(buf1, 100, MPI_INT, 1, 0, MPI_COMM_WORLD, &status);
   } else if (rank == 1) {
       MPI_Recv(buf2, 100, MPI_INT, 0, 0, MPI_COMM_WORLD, &status); // Recv first
       MPI_Send(buf1, 100, MPI_INT, 0, 0, MPI_COMM_WORLD);
   }
   ```
   Alternatively, replace both with `MPI_Sendrecv()`.

---

### Question 3: Wildcards & Status (5 Marks)
**Explain the use of `MPI_ANY_SOURCE` and `MPI_ANY_TAG`. How does a receiving process determine the identity of the sender and the actual number of elements received?**

#### Model Examination Answer:
1. **Wildcard Roles:**  
   - `MPI_ANY_SOURCE`: Passed as the source parameter to `MPI_Recv()`. Instructs the runtime to accept an incoming message from *any* process in the communicator on a first-come, first-served basis.
   - `MPI_ANY_TAG`: Passed as the tag parameter. Instructs the runtime to accept messages regardless of their envelope tag value.
2. **Retrieving Metadata:**  
   When wildcards are used, metadata is recovered via the `MPI_Status` output argument:
   - **Sender Rank:** Extracted directly from `status.MPI_SOURCE`.
   - **Message Tag:** Extracted directly from `status.MPI_TAG`.
   - **Element Count:** Querying actual data count requires calling the helper function:
     ```c
     int count;
     MPI_Get_count(&status, MPI_INT, &count);
     ```

---

## 10. Quick Revision & Key Takeaways

```
================================================================================
                    SUBTOPIC 06.2 QUICK REVISION SHEET
================================================================================

1. POINT-TO-POINT COMMUNICATION:
   - Pairwise directed transfer: 1 sender (dest rank), 1 receiver (source rank).
   - MPI_Send(&buf, count, datatype, dest, tag, comm)
   - MPI_Recv(&buf, count, datatype, source, tag, comm, &status)

2. PARAMETER ESSENTIALS:
   - count: Number of elements (NOT bytes!).
   - datatype: MPI_INT, MPI_DOUBLE, MPI_CHAR (handles endianness conversion).
   - tag: Integer classification label (0 to 32767).

3. WILDCARDS & MPI_STATUS:
   - Wildcards: MPI_ANY_SOURCE, MPI_ANY_TAG
   - status.MPI_SOURCE: Identifies actual sender rank.
   - status.MPI_TAG: Identifies actual tag.
   - MPI_Get_count(&status, datatype, &count): Obtains actual count of elements.

4. PROTOCOLS & DEADLOCKS:
   - Eager Protocol: Tiny messages buffered in network; Send returns fast.
   - Rendezvous Protocol: Bulk messages require receiver handshake; Send blocks!
   - Deadlock hazard: Both processes calling Recv first -> permanent hang!
   - Solutions: Alternate send/recv order, or use MPI_Sendrecv().
================================================================================
```

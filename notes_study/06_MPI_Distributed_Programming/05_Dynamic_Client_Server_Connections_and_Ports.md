# Subtopic 06.5: Dynamic Client-Server Connections, Ports & `ompi-server`

---

## 1. Pedagogical Motivation: Breaking the Static `MPI_COMM_WORLD` Barrier

In traditional MPI programming (MPI-1 standard), every parallel application is statically bound to an immutable execution harness:
```bash
mpirun -np 16 ./simulation_engine
```
When this command executes, the process manager spawns exactly 16 processes, binds them into the predefined global communicator `MPI_COMM_WORLD`, and fixes the cluster topology for the lifetime of the application. 

While static execution models are suitable for tightly coupled batch simulations (such as molecular dynamics or finite-element analysis), modern high-performance and distributed computing frequently demands **dynamic, decoupled, and long-running services**:

1. **Decoupled Client-Server Architectures:** Consider an interactive visualization dashboard (a GUI client) that periodically connects to a massive, long-running weather simulation running on a 500-node cluster to fetch the latest climate state without stopping or restarting the simulation.
2. **Independent Workflow Pipelines:** A data-ingestion pipeline produces telemetry data in one job, while a machine learning inference engine consumes that data in another independently launched job. Neither job was launched by the same `mpirun` command.
3. **Adaptive Elastic Computing:** Compute nodes may join or leave a running distributed calculation based on cloud spot pricing or computational load variations.

### The Fundamental Obstacle
Two independently launched MPI programs—e.g., `Terminal 1: mpirun -np 1 ./sender` and `Terminal 2: mpirun -np 1 ./receiver`—possess completely separate, isolated instances of `MPI_COMM_WORLD`. Neither knows of the other's existence, network IP addresses, or process IDs.

```
       TERMINAL 1 (Job A)                       TERMINAL 2 (Job B)
+-------------------------------+       +-------------------------------+
|    MPI_COMM_WORLD (Job A)     |       |    MPI_COMM_WORLD (Job B)     |
|  [ Rank 0: sender.c (PID 101) ] |       | [ Rank 0: receiver.c (PID 205)] |
+-------------------------------+       +-------------------------------+
                |                                       |
                x------------- CANNOT COMMUNICATE ------x
                      (Isolated Communicator Realms)
```

**The Solution:** The **MPI-2 Dynamic Process Model**. MPI-2 introduced primitives that allow independent programs to publish rendezvous endpoints ("MPI Ports"), establish dynamic connections across the network, and synthesize a bidirectional communication bridge known as an **Inter-Communicator**.

---

## 2. Core Architectural Concepts

To master dynamic MPI connections, three key abstractions must be understood:

### 2.1. The MPI Port Abstraction
In standard TCP/IP networking, a port is a 16-bit integer (e.g., port 8080).  
In MPI, an **MPI Port** is an opaque, system-generated character string that encodes the physical and logical network coordinates required for an external process to establish contact.

```c
char port_name[MPI_MAX_PORT_NAME];
```
- `MPI_MAX_PORT_NAME`: A predefined MPI implementation constant (typically 1024 to 4096 bytes in Open MPI and MPICH).
- A generated port string looks similar to:  
  `"365795337.0;tcp://192.168.1.10:45821+3530881966.0;tcp://192.168.1.10:45823:300"`  
  It encapsulates the host IP, internal daemon sockets, tag offsets, and security tokens.

### 2.2. Intra-Communicators vs. Inter-Communicators
This is an essential distinction for university examinations:

```
INTRA-COMMUNICATOR (e.g., MPI_COMM_WORLD)
- Single group of processes.
- Point-to-point and collective operations occur WITHIN the same group.
- Ranks are indexed from 0 to Size - 1.

INTER-COMMUNICATOR (Dynamic Client-Server Bridge)
- Connects TWO DISJOINT groups of processes (Local Group and Remote Group).
- Point-to-point communication is directional: A send from rank 0 of the local group
  targets rank 0 of the REMOTE group, NOT local rank 0.
```

```
+-------------------------------------------------------------------------------+
|                       INTER-COMMUNICATOR BRIDGE                               |
|                                                                               |
|   LOCAL GROUP (Sender)                              REMOTE GROUP (Receiver)   |
|   +-----------------------+                         +-----------------------+ |
|   | Rank 0 (Server Node)  | <=====================> | Rank 0 (Client Node)  | |
|   +-----------------------+     Bi-directional      +-----------------------+ |
|                                 Network Channel                               |
+-------------------------------------------------------------------------------+
```

---

## 3. The Dynamic Connection Protocol & Lifecycle

Establishing communication between independent MPI programs follows a five-phase handshake protocol:

```
Time     SERVER (sender.c)                             CLIENT (receiver.c)
  |              |                                              |
  |      1. MPI_Open_port()                                     |
  |              |                                              |
  |      2. Publish Port String                                 |
  |         (via port.txt or ompi-server)                       |
  |              | -------------[ port.txt / URI ]------------> |
  |              |                                              |
  |      3. MPI_Comm_accept()                           3. MPI_Comm_connect()
  |         (Blocks listening)                             (Initiates handshake)
  |              | <========== HANDSHAKE ESTABLISHED ========> |
  |              |       [ Returns Inter-Communicator ]         |
  |              |                                              |
  |      4. MPI_Send(..., client)                       4. MPI_Recv(..., server)
  |              | ------------ Data Transfer ----------------> |
  |              |                                              |
  |      5. MPI_Comm_disconnect()                       5. MPI_Comm_disconnect()
  |              |                                              |
  |      6. MPI_Close_port()                                    |
  v              v                                              v
```

### Detailed Functional Breakdown:

#### 1. Port Allocation (`MPI_Open_port`)
The server requests the MPI runtime to bind a new network service endpoint:
```c
int MPI_Open_port(
    MPI_Info info,        // Implementation hints (usually MPI_INFO_NULL)
    char *port_name       // Output buffer of size MPI_MAX_PORT_NAME
);
```

#### 2. Waiting for Incoming Client (`MPI_Comm_accept`)
The server transitions into a listening state, blocking until a client contacts the port:
```c
int MPI_Comm_accept(
    const char *port_name, // The port opened above
    MPI_Info info,         // Implementation hints (MPI_INFO_NULL)
    int root,              // Rank in comm that accepts (usually 0)
    MPI_Comm comm,         // Intra-communicator of server group (MPI_COMM_SELF)
    MPI_Comm *newcomm      // Output: Newly formed Inter-Communicator
);
```

#### 3. Client Connection Request (`MPI_Comm_connect`)
The client uses the published port string to initiate the network handshake:
```c
int MPI_Comm_connect(
    const char *port_name, // Server's published port string
    MPI_Info info,         // Implementation hints (MPI_INFO_NULL)
    int root,              // Rank in comm that initiates (usually 0)
    MPI_Comm comm,         // Intra-communicator of client group (MPI_COMM_SELF)
    MPI_Comm *newcomm      // Output: Newly formed Inter-Communicator
);
```

#### 4. Disconnection & Cleanup (`MPI_Comm_disconnect` & `MPI_Close_port`)
Because dynamic connections allocate underlying network file descriptors and communication state tables outside of standard communicator scopes, they must be gracefully unlinked:
```c
int MPI_Comm_disconnect(MPI_Comm *comm); // Synchronously unlinks both sides
int MPI_Close_port(const char *port_name); // Releases network port binding
```

---

## 4. Port Publication: Shared Files vs. Centralized Name Servers (`ompi-server`)

How does the client discover the server's dynamically generated port string?

### Approach A: Shared Filesystem (`port.txt`)
In simple single-node or shared-NFS cluster environments, the server writes the string directly to a shared text file (`port.txt`). The client reads `port.txt` upon startup.
- **Pros:** No extra infrastructure required; simple to program and debug.
- **Cons:** Requires a shared filesystem; prone to file race conditions if client starts before server finishes writing.

---

### Approach B: Centralized Enterprise Name Server (`ompi-server`)
In large multi-node production clusters where nodes do not share a common disk or where hundreds of dynamic jobs interact, MPI provides a dedicated lookup daemon: **`ompi-server`**.

> [!NOTE]
> ### The DNS Analogy (Crucial Exam Concept)
> Just as the Internet's **Domain Name System (DNS)** maps human-readable domain names (`google.com`) to physical IP addresses (`142.250.190.46`), `ompi-server` acts as a distributed directory service.  
> It allows an MPI application to publish an abstract service name (e.g., `"WeatherService"`) mapped to its complex internal MPI port string via `MPI_Publish_name()`, allowing clients to find it using `MPI_Lookup_name()`.

```
                    CENTRALIZED LOOKUP ARCHITECTURE (ompi-server)

                          +--------------------------+
                          |       ompi-server        |
                          |  (Central Lookup Daemon) |
                          +--------------------------+
                               ^                |
        1. MPI_Publish_name    |                | 2. MPI_Lookup_name
           ("WeatherService")  |                |    ("WeatherService")
                               |                v
                    +--------------+        +--------------+
                    |   SERVER     |        |   CLIENT     |
                    | (sender.c)   |        | (receiver.c) |
                    +--------------+        +--------------+
                               ^                |
                               +----------------+
                             3. Direct TCP/RDMA
                                Inter-Communicator
```

#### Starting the `ompi-server` Daemon:
```bash
ompi-server --no-daemonize -report-uri /tmp/ompi-server.uri
```
- `--no-daemonize`: Keeps the server process in the terminal foreground so its logs and activity can be monitored.
- `-report-uri /tmp/ompi-server.uri`: Instructs the daemon to output its own Process Management Interface (PMIx) Uniform Resource Identifier into `/tmp/ompi-server.uri`.

#### Linking Independent `mpirun` Jobs to `ompi-server`:
When launching independent MPI jobs, the Open MPI MCA parameter `pmix_server_uri` tells each instance where to locate the shared directory daemon:
```bash
# Terminal 1: Launch Sender
mpirun --mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./sender

# Terminal 2: Launch Receiver
mpirun --mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./receiver
```

---

## 5. Complete, Compilable C Implementations

### Program 1: The Server Process (`sender.c`)

This complete program opens a dynamic MPI port, exports the port identifier to `port.txt`, blocks until a client connects, transmits data across the inter-communicator, and cleanly tears down the channel.

```c
/******************************************************************************
 * File: sender.c
 * Description: MPI-2 Dynamic Server implementation.
 *              Opens an MPI port, saves it to disk, waits for a client,
 *              transmits an integer payload, and safely disconnects.
 * Compilation: mpicc -Wall -O2 sender.c -o sender
 * Execution:   mpirun -np 1 ./sender
 ******************************************************************************/

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

#define PORT_FILE "port.txt"

int main(int argc, char *argv[]) {
    MPI_Comm client_comm;
    char port_name[MPI_MAX_PORT_NAME];
    int data_to_send = 100;
    FILE *fp = NULL;

    // 1. Initialize MPI Execution Environment
    MPI_Init(&argc, &argv);

    printf("===================================================\n");
    printf("[SERVER: sender.c] Process Started (PID = %d)\n", getpid());
    printf("===================================================\n");

    // 2. Open an MPI network communication port
    if (MPI_Open_port(MPI_INFO_NULL, port_name) != MPI_SUCCESS) {
        fprintf(stderr, "[SERVER ERROR] Failed to open MPI port!\n");
        MPI_Abort(MPI_COMM_WORLD, 1);
    }

    printf("[SERVER] Successfully opened dynamic port:\n--> %s\n", port_name);
    fflush(stdout);

    // 3. Save the port name to a file so the client can discover it
    fp = fopen(PORT_FILE, "w");
    if (fp == NULL) {
        perror("[SERVER ERROR] Could not open port.txt for writing");
        MPI_Close_port(port_name);
        MPI_Abort(MPI_COMM_WORLD, 1);
    }
    fprintf(fp, "%s\n", port_name);
    fclose(fp);
    printf("[SERVER] Published port identifier to '%s'.\n", PORT_FILE);
    printf("[SERVER] Waiting for client to connect via MPI_Comm_accept()...\n");
    fflush(stdout);

    /*
     * 4. Accept incoming connection:
     * - Blocks until an external process calls MPI_Comm_connect with this port.
     * - Returns a new inter-communicator (client_comm).
     */
    MPI_Comm_accept(port_name, MPI_INFO_NULL, 0, MPI_COMM_SELF, &client_comm);
    printf("[SERVER] Connection established! Client handshake accepted.\n");

    /*
     * 5. Send data across the inter-communicator:
     * - Destination rank is 0 (the client's rank in its remote group).
     * - Tag is 0.
     */
    printf("[SERVER] Transmitting payload (data = %d) to client...\n", data_to_send);
    MPI_Send(&data_to_send, 1, MPI_INT, 0, 0, client_comm);
    printf("[SERVER] Payload successfully dispatched.\n");

    // 6. Graceful Disconnection and Port Cleanup
    printf("[SERVER] Disconnecting client inter-communicator...\n");
    MPI_Comm_disconnect(&client_comm);

    printf("[SERVER] Closing communication port...\n");
    MPI_Close_port(port_name);

    // Remove the temporary rendezvous file
    remove(PORT_FILE);

    printf("[SERVER] Finalizing MPI. Server shut down cleanly.\n");
    MPI_Finalize();
    return 0;
}
```

---

### Program 2: The Client Process (`receiver.c`)

This program reads the published port string from `port.txt`, initiates an active connection request to the server, receives the payload over the inter-communicator, and finalizes cleanly.

```c
/******************************************************************************
 * File: receiver.c
 * Description: MPI-2 Dynamic Client implementation.
 *              Reads server's port from disk, initiates connection,
 *              receives payload, and cleanly disconnects.
 * Compilation: mpicc -Wall -O2 receiver.c -o receiver
 * Execution:   mpirun -np 1 ./receiver
 ******************************************************************************/

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define PORT_FILE "port.txt"

int main(int argc, char *argv[]) {
    MPI_Comm server_comm;
    MPI_Status status;
    char port_name[MPI_MAX_PORT_NAME];
    int received_data = 0;
    FILE *fp = NULL;

    // 1. Initialize MPI Execution Environment
    MPI_Init(&argc, &argv);

    printf("===================================================\n");
    printf("[CLIENT: receiver.c] Process Started (PID = %d)\n", getpid());
    printf("===================================================\n");

    // 2. Read the server's published port string from disk
    printf("[CLIENT] Searching for rendezvous file '%s'...\n", PORT_FILE);
    fp = fopen(PORT_FILE, "r");
    if (fp == NULL) {
        fprintf(stderr, "[CLIENT ERROR] Could not find '%s'!\n", PORT_FILE);
        fprintf(stderr, "[HINT] Make sure the server (sender) is running first.\n");
        MPI_Abort(MPI_COMM_WORLD, 1);
    }

    if (fscanf(fp, "%s", port_name) != 1) {
        fprintf(stderr, "[CLIENT ERROR] Failed to read port string from file.\n");
        fclose(fp);
        MPI_Abort(MPI_COMM_WORLD, 1);
    }
    fclose(fp);

    printf("[CLIENT] Read server port string:\n--> %s\n", port_name);
    printf("[CLIENT] Connecting to server via MPI_Comm_connect()...\n");
    fflush(stdout);

    /*
     * 3. Connect to the server:
     * - Contacts the port opened by sender.c.
     * - Synthesizes the inter-communicator (server_comm).
     */
    MPI_Comm_connect(port_name, MPI_INFO_NULL, 0, MPI_COMM_SELF, &server_comm);
    printf("[CLIENT] Handshake confirmed! Connected to server.\n");

    /*
     * 4. Receive data across the inter-communicator:
     * - Source rank is 0 (the server's rank in its remote group).
     * - Tag is 0.
     */
    MPI_Recv(&received_data, 1, MPI_INT, 0, 0, server_comm, &status);
    printf("[CLIENT] SUCCESS: Received payload from server = %d\n", received_data);

    // 5. Graceful Disconnection and Finalization
    printf("[CLIENT] Disconnecting server inter-communicator...\n");
    MPI_Comm_disconnect(&server_comm);

    printf("[CLIENT] Finalizing MPI. Client terminated successfully.\n");
    MPI_Finalize();
    return 0;
}
```

---

## 6. Step-by-Step Compilation, Execution & Verification Workflow

### Step 1: Compilation
In your Linux or HPC terminal, compile both source files using the MPI wrapper compiler:
```bash
mpicc -Wall -O2 sender.c -o sender
mpicc -Wall -O2 receiver.c -o receiver
```

---

### Step 2: Live Execution in Two Separate Terminals

#### Terminal 1: Launch the Server
```bash
mpirun -np 1 ./sender
```
*Console Output (Server blocks waiting for receiver):*
```text
===================================================
[SERVER: sender.c] Process Started (PID = 43120)
===================================================
[SERVER] Successfully opened dynamic port:
--> 365795337.0;tcp://127.0.0.1:51243+3530881966.0;tcp://127.0.0.1:51245:300
[SERVER] Published port identifier to 'port.txt'.
[SERVER] Waiting for client to connect via MPI_Comm_accept()...
```

#### Terminal 2: Launch the Client (in another window)
```bash
mpirun -np 1 ./receiver
```
*Console Output (Client connects and receives data):*
```text
===================================================
[CLIENT: receiver.c] Process Started (PID = 43185)
===================================================
[CLIENT] Searching for rendezvous file 'port.txt'...
[CLIENT] Read server port string:
--> 365795337.0;tcp://127.0.0.1:51243+3530881966.0;tcp://127.0.0.1:51245:300
[CLIENT] Connecting to server via MPI_Comm_connect()...
[CLIENT] Handshake confirmed! Connected to server.
[CLIENT] SUCCESS: Received payload from server = 100
[CLIENT] Disconnecting server inter-communicator...
[CLIENT] Finalizing MPI. Client terminated successfully.
```

*Console Output (Server unblocks and finishes):*
```text
[SERVER] Connection established! Client handshake accepted.
[SERVER] Transmitting payload (data = 100) to client...
[SERVER] Payload successfully dispatched.
[SERVER] Disconnecting client inter-communicator...
[SERVER] Closing communication port...
[SERVER] Finalizing MPI. Server shut down cleanly.
```

---

## 7. Analysis of Timing Asynchrony: "Fast Sender vs. Slow Receiver"

An important question frequently posed in examinations and systems interviews is:
> **Question:** *What happens if the sender produces messages at high speed while the receiver is slow or asleep?*

```c
// In receiver.c:
while (1) {
    MPI_Recv(&data, 1, MPI_INT, 0, 0, server, &status);
    sleep(50); // Simulates heavy 50-second compute task
}
```

### Protocol Response:
1. **Below Eager Threshold (Small Messages):**
   - If the sender transmits small integers (e.g., 4 bytes), the MPI subsystem will buffer the message in its internal network memory (or system socket buffer) and allow `MPI_Send` on the sender to return immediately.
2. **Above Eager Threshold / Buffer Saturation:**
   - As the fast sender continues looping, the MPI network transmission buffers quickly fill up to capacity.
   - Once buffers are saturated, the runtime switches to the **Rendezvous Protocol**. Subsequent calls to `MPI_Send` on the sender **block unconditionally** until the slow receiver issues an `MPI_Recv` to drain the socket.
3. **Data Integrity Guarantee:**
   - Standard MPI will **never** silently drop or overwrite packets due to speed disparity. Either the sender is backpressured (throttled), or memory exhaustion triggers an `MPI_ERR_NO_MEM` runtime abort.

---

## 8. Solved Numerical & Engineering Problems

### Numerical Problem 1: Handshake Latency Overhead vs. Static Communication

#### Problem Statement:
A distributed monitoring service executes on an HPC cluster.
- Establishing a dynamic inter-communicator handshake (`MPI_Open_port` + `MPI_Comm_accept` + `MPI_Comm_connect`) incurs a fixed network connection latency of $T_{\text{handshake}} = 12.5\text{ ms}$.
- Once connected, individual telemetry packets of size $m = 64\text{ bytes}$ are transmitted with a latency $\alpha = 2\text{ }\mu\text{s}$ ($2 \times 10^{-3}\text{ ms}$) and bandwidth $B = 1\text{ GB/s}$ ($\beta = 10^{-6}\text{ ms/byte}$).
- In Scenario A, the client connects, sends $1$ packet, and immediately disconnects; it repeats this cycle $N = 1,000$ times.
- In Scenario B, the client establishes the dynamic connection once, streams all $N = 1,000$ packets consecutively, and then disconnects.

Calculate:
1. Total communication time for Scenario A ($T_A$).
2. Total communication time for Scenario B ($T_B$).
3. The speedup $S = \frac{T_A}{T_B}$ achieved by maintaining a persistent connection.

---

#### Step-by-Step Solution:

**Step 1: Calculate single-packet transfer time ($t_{\text{packet}}$):**
$$t_{\text{packet}} = \alpha + m \cdot \beta$$
$$t_{\text{packet}} = 2 \times 10^{-3}\text{ ms} + (64 \times 10^{-6}\text{ ms}) = 0.002064\text{ ms}$$

---

**Step 2: Total Time for Scenario A (Transient Connections):**
Each iteration incurs a full handshake plus one packet transfer:
$$T_A = N \times (T_{\text{handshake}} + t_{\text{packet}})$$
$$T_A = 1,000 \times (12.5\text{ ms} + 0.002064\text{ ms}) = 1,000 \times 12.502064\text{ ms} = 12,502.064\text{ ms} \approx 12.502\text{ s}$$

---

**Step 3: Total Time for Scenario B (Persistent Connection):**
A single handshake is performed, followed by $1,000$ continuous packet transfers:
$$T_B = T_{\text{handshake}} + (N \times t_{\text{packet}})$$
$$T_B = 12.5\text{ ms} + (1,000 \times 0.002064\text{ ms}) = 12.5\text{ ms} + 2.064\text{ ms} = 14.564\text{ ms} \approx 0.01456\text{ s}$$

---

**Step 4: Compute Speedup ($S$):**
$$S = \frac{T_A}{T_B} = \frac{12,502.064\text{ ms}}{14.564\text{ ms}} \approx 858.4\times$$

$$\boxed{T_A = 12.502\text{ s}, \quad T_B = 14.564\text{ ms}, \quad \text{Speedup } S = 858.4\times}$$

**Engineering Takeaway:** Dynamic inter-communicator handshakes carry significant connection setup overhead. In production architectures, establish the connection once during initialization and reuse the inter-communicator for ongoing data streams.

---

## 9. Common Examination Pitfalls & Traps

> [!CAUTION]
> ### 1. Rank Indexing in Inter-Communicators
> **The Trap:** When calling `MPI_Send` on the server across the inter-communicator, developers often wonder: *"What rank should I pass as the destination?"*  
> **Rule:** In an **Inter-Communicator**, ranks are evaluated relative to the **remote group**.  
> If the client job consists of 1 process, its rank in the remote group is `0`. Therefore, the server sends to `dest = 0`. It does **not** mean sending to itself!

> [!WARNING]
> ### 2. Calling `MPI_Comm_free` instead of `MPI_Comm_disconnect`
> Standard communicators created via `MPI_Comm_dup` or `MPI_Comm_split` are deallocated using `MPI_Comm_free`.  
> However, for dynamic inter-communicators established via `MPI_Comm_accept`/`connect`, calling `MPI_Comm_free` can cause dangling network connections or memory leaks. **You must call `MPI_Comm_disconnect()`**, which synchronizes both endpoints before releasing system sockets.

> [!NOTE]
> ### 3. Forgetting `MPI_Close_port()`
> Opening an MPI port allocates network sockets and kernel descriptors. If a server process loops accepting connections without calling `MPI_Close_port()`, it will eventually exhaust OS file descriptors (`EMFILE: Too many open files`).

---

## 10. Summary Comparison: Static vs. Dynamic MPI

| Architectural Feature | Static MPI (MPI-1 Standard) | Dynamic Client-Server MPI (MPI-2) |
| :--- | :--- | :--- |
| **Process Launching** | Single `mpirun` command for all nodes | Independent `mpirun` invocations at different times |
| **Communicator Scope** | `MPI_COMM_WORLD` encompasses all ranks | Separate `MPI_COMM_WORLD` per job; bridged via Inter-Communicator |
| **Process Count ($P$)** | Fixed for the lifetime of the program | Dynamic; clients can attach and detach on demand |
| **Endpoint Discovery** | Automatic rank-based indexing ($0 \dots P-1$) | Via MPI Port string (`port.txt` or `ompi-server` daemon) |
| **Primary Primitives** | `MPI_Init`, `MPI_Send`, `MPI_Recv` | `MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect` |
| **Lookup Service** | Not required | `ompi-server` (DNS equivalent for MPI ports) |
| **Teardown Routine** | `MPI_Finalize` | `MPI_Comm_disconnect` followed by `MPI_Close_port` |

---

## 11. Quick University Exam Revision Sheet

- **Why Dynamic MPI?** Enables communication between independently launched MPI jobs without sharing a common initial `MPI_COMM_WORLD`.
- **MPI Port:** An implementation-generated character string (up to `MPI_MAX_PORT_NAME`) holding internal network socket endpoints.
- **`ompi-server`:** Centralized lookup daemon acting as a DNS registry for MPI applications, enabling port publishing (`MPI_Publish_name`) and lookup (`MPI_Lookup_name`).
- **Standard Protocol Sequence:**
  1. Server calls `MPI_Open_port()`.
  2. Server publishes port string (writes to `port.txt` or registers with `ompi-server`).
  3. Server calls `MPI_Comm_accept()` (blocks listening).
  4. Client reads port string and calls `MPI_Comm_connect()`.
  5. Both acquire a connected **Inter-Communicator**.
  6. Data is exchanged via `MPI_Send`/`MPI_Recv`.
  7. Channel is unlinked via `MPI_Comm_disconnect()`.
  8. Server releases socket via `MPI_Close_port()`.
- **Key CLI Flags:**
  - `ompi-server --no-daemonize -report-uri /tmp/ompi-server.uri`
  - `mpirun --mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./sender`

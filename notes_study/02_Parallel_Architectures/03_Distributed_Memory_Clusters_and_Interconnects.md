# Distributed-Memory Systems: Clusters, Interconnects & Message Passing Networks

In Subtopic 02.2, we saw that shared-memory architectures (UMA and ccNUMA) provide a single address space, making communication between threads fast and implicit. However, physical constraints (pin counts, cache coherence traffic, and electrical bus limits) prevent shared-memory systems from scaling beyond several dozen or at most a few hundred cores.

When scientists need to simulate the global climate, model molecular dynamics with billions of atoms, or train massive neural networks, they require systems with **hundreds of thousands or millions of processor cores**. 

To reach this scale, we must abandon the shared address space entirely and build **Distributed-Memory Systems (Clusters)**. In this subtopic, you will trace the evolution of distributed memory from historical "pure" systems to modern hierarchical supercomputer clusters, examine high-speed interconnect fabrics, and understand the physical latency-bandwidth hierarchy that governs distributed computing.

---

## 1. What is a Distributed-Memory System?

### Basic Intuition: The Network of Post Offices
Imagine four independent cities across a country. Each city has its own local warehouse, its own workers, and its own records. 
- A worker in City 1 **cannot** simply walk into City 2's warehouse and take an item (there is no shared physical space).
- If City 1 needs data or materials from City 2, City 1 must write a formal letter, place the data into an envelope, hand it to the postal service, and transmit it across the national highway network.
- City 2's post office receives the envelope, unpacks the contents, and hands it to the local worker.

In computer systems:
- The **cities** are independent compute nodes.
- The **local warehouses** are private DRAM banks.
- The **postal service** is the Network Interface Card (NIC / NI).
- The **highway network** is the high-speed network switch fabric (e.g., InfiniBand).
- The **formal letters** are **messages** in the Message Passing Interface (MPI).

```
                      DISTRIBUTED-MEMORY SYSTEM
                      
      COMPUTE NODE 0                              COMPUTE NODE 1
   ┌───────────────────┐                       ┌───────────────────┐
   │ ┌───────────────┐ │                       │ ┌───────────────┐ │
   │ │ Processor (P) │ │                       │ │ Processor (P) │ │
   │ └───────▲───────┘ │                       │ └───────▲───────┘ │
   │         │         │                       │         │         │
   │ ┌───────▼───────┐ │                       │ ┌───────▼───────┐ │
   │ │ Private Memory│ │                       │ │ Private Memory│ │
   │ │      (M)      │ │                       │ │      (M)      │ │
   │ └───────▲───────┘ │                       │ └───────▲───────┘ │
   │         │         │                       │         │         │
   │ ┌───────▼───────┐ │                       │ ┌───────▼───────┐ │
   │ │Network Inter. │ │                       │ │Network Inter. │ │
   │ │     (NI)      │ │                       │ │     (NI)      │ │
   └─────────┬─────────┘                       └─────────┬─────────┘
             │                                           │
             └──────────────► INTERCONNECT ◄─────────────┘
                              COMMUNICATION
                                 NETWORK
```

---

## 2. Distributed Memory Systems: "Back in the Day" vs Today

Your lecture notes draw a profound architectural distinction between how distributed clusters were built historically ("back in the day") and how modern high-performance clusters are constructed today.

### The Historical Approach: "Pure" Distributed-Memory Systems
In the early days of cluster computing (the 1990s "Beowulf" cluster era):
- Each machine was an off-the-shelf uniprocessor PC containing:
  - A single processor ($P$) with on-chip cache ($C$).
  - An exclusive, private local memory ($M$).
  - A standard Network Interface card ($NI$).
- **Interconnection:** Standard local area network (LAN) using Ethernet.
- **Protocols:** Operating system networking stacks: **TCP/IP** and **UDP/IP**.
- **Software Model:** **One process per node**.
- **Communication:** Pure explicit **message passing** over the network. If Node 0 needed data from Node 1, it had to call OS socket routines or early MPI primitives.

```
       "PURE" DISTRIBUTED-MEMORY SYSTEM (FROM LECTURE NOTES)
       
       ┌─────────┐       ┌─────────┐       ┌─────────┐       ┌─────────┐
       │    P    │       │    P    │       │    P    │       │    P    │
       ├─────────┤       ├─────────┤       ├─────────┤       ├─────────┤
       │    C    │       │    C    │       │    C    │       │    C    │
       ├─────────┤       ├─────────┤       ├─────────┤       ├─────────┤
       │    M    │       │    M    │       │    M    │       │    M    │
       ├─────────┤       ├─────────┤       ├─────────┤       ├─────────┤
       │   NI    │       │   NI    │       │   NI    │       │   NI    │
       └────┬────┘       └────┬────┘       └────┬────┘       └────┬────┘
            │                 │                 │                 │
    ────────┴─────────────────┴─────────────────┴─────────────────┴────────
                         COMMUNICATION NETWORK
                       (TCP/IP and UDP/IP Stacks)
```

#### Why Pure Systems Became Obsolete:
1. **High Protocol Overhead:** The operating system's TCP/IP stack involves kernel context switches, packet header processing, checksum calculations, and multiple data copies in RAM, introducing massive latency ($\approx 50\text{ to } 100\ \mu\text{s}$).
2. **Silicon Inefficiency:** Having only 1 core per box meant huge physical chassis footprints, high power costs, and massive cabling complexity.

---

### Modern Distributed-Memory Systems: Hierarchical Clusters
Today, a compute node is no longer a simple single-core PC. As your notes state:
> *"Distributed memory systems today: Cluster of networked shared-memory nodes. ccNUMA architecture per node. Multiple cores per ccNUMA domain."*

```
              MODERN HIGH-PERFORMANCE CLUSTER ARCHITECTURE
              
     NODE 0 (ccNUMA Chassis)                      NODE 1 (ccNUMA Chassis)
 ┌─────────────────────────────┐              ┌─────────────────────────────┐
 │ ┌─────────────┐┌──────────┐ │              │ ┌─────────────┐┌──────────┐ │
 │ │Socket 0(UMA)││Socket 1  │ │              │ │Socket 0(UMA)││Socket 1  │ │
 │ │  16 Cores   ││ 16 Cores │ │              │ │  16 Cores   ││ 16 Cores │ │
 │ └──────┬──────┘└────┬─────┘ │              │ └──────┬──────┘└────┬─────┘ │
 │        │ Inter-socket│      │              │        │ Inter-socket│      │
 │        └──────┬─────┘       │              │        └──────┬─────┘       │
 │               ▼             │              │               ▼             │
 │        [ Host Memory ]      │              │        [ Host Memory ]      │
 │               │             │              │               │             │
 │     ┌─────────┴────────┐    │              │     ┌─────────┴────────┐    │
 │     │ PCIe Host Bus    │    │              │     │ PCIe Host Bus    │    │
 │     └─────────┬────────┘    │              │     └─────────┬────────┘    │
 │               ▼             │              │               ▼             │
 │     [ High-Speed NIC / HCA ]│              │     [ High-Speed NIC / HCA ]│
 └───────────────┬─────────────┘              └───────────────┬─────────────┘
                 │                                            │
                 └─────────────► HIGH-SPEED ◄─────────────────┘
                              INTERCONNECT FABRIC
                         (InfiniBand / Slingshot)
```

---

## 3. The Communication Latency & Bandwidth Hierarchy

In a modern supercomputer cluster, the cost of moving data varies by **several orders of magnitude** depending on where the sender and receiver threads/processes reside physically.

Your notes explicitly identify this hierarchy:
> *"Intra-socket, inter-socket, inter-node all have different latency ($\lambda$) and bandwidth ($b$)."*

$$\boxed{\lambda_{\text{intra-socket}} \ll \lambda_{\text{inter-socket}} \ll \lambda_{\text{inter-node}}}$$
$$\boxed{b_{\text{intra-socket}} \gg b_{\text{inter-socket}} \gg b_{\text{inter-node}}}$$

```
        THE THREE TIERS OF COMMUNICATION IN MODERN CLUSTERS
        
 Tier              Physical Boundary            Typical Latency (λ)    Typical Bandwidth (b)
 ─────────────────────────────────────────────────────────────────────────────────────────
 Intra-Socket      Core-to-Core on same chip    10 - 20 ns             200 - 1000 GB/s (L3)
 Inter-Socket      Chip-to-Chip on same board   80 - 150 ns            40 - 80 GB/s (Fabric)
 Inter-Node        Node-to-Node across network  1 - 5 μs (1000-5000 ns)10 - 25 GB/s (InfiniBand)
```

### Analysis of the Tiers:

1. **Tier 1: Intra-Socket Communication (On-Chip):**
   - Cores share the same silicon die.
   - Communication occurs via cache coherence across the on-chip L3 ring or mesh bus.
   - **Latency:** $\approx 10\text{ to } 20\text{ ns}$ (extremely fast).
   - **Bandwidth:** $\approx 500\text{ to } 1,000\text{ GB/s}$.

2. **Tier 2: Inter-Socket Communication (On-Motherboard):**
   - Cores reside on different CPU sockets on the same motherboard.
   - Communication traverses inter-socket point-to-point links (Intel Ultra Path Interconnect / UPI, AMD Infinity Fabric).
   - **Latency:** $\approx 100\text{ ns}$ ($5\text{ to } 10\times$ slower than intra-socket).
   - **Bandwidth:** $\approx 40\text{ to } 80\text{ GB/s}$.

3. **Tier 3: Inter-Node Communication (Across the Network Fabric):**
   - Processes reside in physically distinct chassis mounted in server racks.
   - Data must be packaged into network packets, serialized over optical fiber or copper cables, routed through switches, and deserialized at the destination.
   - **Latency:** $\approx 1.0\text{ to } 3.0\ \mu\text{s}$ ($1,000\text{ to } 3,000\text{ ns}$—**over $100\times$ slower than on-chip communication!**).
   - **Bandwidth:** $\approx 10\text{ to } 25\text{ GB/s}$ (per 100G/200G InfiniBand port).

> [!IMPORTANT]
> **Why Parallel Algorithms Must Be "Topology-Aware":**
> If an algorithm naively sends millions of tiny messages across nodes, performance is obliterated by the $1\ \mu\text{s}$ network latency. High-performance software must place closely communicating tasks on the same socket (using OpenMP/threads) and use network messages (MPI) only for coarse-grained data exchanges.

---

## 4. Modern Interconnect Technologies & RDMA

In modern HPC clusters, standard operating system TCP/IP stacks have been replaced by specialized networking hardware: **InfiniBand**, **Intel Omni-Path**, and **Cray Slingshot**.

### Remote Direct Memory Access (RDMA)
The breakthrough technology that enabled microsecond-scale cluster networking is **RDMA (Remote Direct Memory Access)**:

```
               TRADITIONAL TCP/IP vs ZERO-COPY RDMA
               
 TRADITIONAL OS NETWORKING (High Overhead, ~50 us)
 Application Buffer ──► Kernel Socket Buffer ──► NIC DMA ──► Wire
         [ Context Switch ]      [ Memory Copy ]
         
 ZERO-COPY RDMA NETWORKING (Kernel-Bypass, ~1 us)
 Application Buffer ───────────────────────────► NIC HCA ──► Wire
             (Hardware reads directly from user memory!)
```

1. **Kernel Bypass:** The network adapter (Host Channel Adapter / HCA) communicates directly with user-space memory, completely bypassing the operating system kernel and eliminating expensive context switches.
2. **Zero-Copy Transfers:** Data is transferred directly from the user memory of Node 0 into the user memory of Node 1 via DMA without intermediate staging copies in OS kernel RAM.

---

## 5. Network Topologies: Connecting Thousands of Nodes

In a distributed supercomputer containing 10,000 nodes, how should the physical network cables connect the nodes together?

Your notes state: *"On top: effects from network structure."* The geometric wiring layout of a parallel computer is called its **Interconnection Network Topology**.

### 1. Fully Connected (Crossbar Switch):
- Every node has a direct physical cable to every other node.
- **Latency:** 1 hop (ideal).
- **Cost:** Requires $\frac{N(N - 1)}{2}$ links. Physically impossible and economically unfeasible for large $N$ (e.g., $10,000\text{ nodes} \implies 50\text{ million cables!}$).

### 2. Fat-Tree Topology (The Modern Supercomputing Standard):
- Nodes are connected at the leaves of a hierarchical tree of switches.
- As you move up from leaf switches towards the root switches, the links become thicker ("fatter")—meaning more parallel cables are added to prevent bottlenecks at higher levels.
- **Advantage:** Provides **full bisection bandwidth**; any half of the supercomputer can communicate with the other half at full wire speed without congestion.

```
                         FAT-TREE TOPOLOGY
                         
                      [ ROOT SWITCHES ]
                         /    │    \
              ┌─────────┘     │     └─────────┐   (Fatter/More Cables)
              ▼               ▼               ▼
        [ SWITCH A ]    [ SWITCH B ]    [ SWITCH C ]
          /      \        /      \        /      \
         ▼        ▼      ▼        ▼      ▼        ▼
       Node 0   Node 1 Node 2   Node 3 Node 4   Node 5
```

### 3. Torus / Mesh Topology:
- Nodes are arranged in a multi-dimensional grid (e.g., 3D or 5D torus) where each node connects only to its nearest physical neighbors in the rack ($+x, -x, +y, -y, +z, -z$).
- **Advantage:** Highly scalable wiring; cables are short and uniform.
- **Disadvantage:** Communication between distant nodes takes multiple switch "hops", increasing latency.

---

## Important Comparisons

### Shared-Memory vs Distributed-Memory Paradigms

| Feature | Shared-Memory System | Distributed-Memory System (Cluster) |
| :--- | :--- | :--- |
| **Physical Memory** | Single physical memory space (UMA or ccNUMA). | Disjoint, physically isolated local memories. |
| **Address Space** | Unified global address space. | Multiple separate, private address spaces. |
| **Communication Mechanism** | Implicit read/write to shared variables (`x = 5`). | Explicit message passing (`MPI_Send` / `MPI_Recv`). |
| **Hardware Required** | Multi-core CPU or multi-socket SMP motherboard. | Network switches, network interfaces (NICs/HCAs), cables. |
| **Hardware Boundary** | Single physical chassis / motherboard. | Hundreds to thousands of independent server racks. |
| **Data Synchronization** | Locks, mutexes, condition variables, barriers. | Message rendezvous, message tags, collective operations. |
| **Scalability Ceiling** | Tens to a few hundred cores. | Millions of cores (e.g., modern Top500 supercomputers). |
| **Primary Programming API**| OpenMP, Pthreads. | Message Passing Interface (MPI). |

---

## Exam-Ready Definitions

### Q1. What is a distributed-memory parallel computer?
> **Answer:**  
> A distributed-memory parallel computer is an architectural system composed of multiple autonomous processing nodes, each possessing its own private local memory and network interface, where nodes communicate and coordinate exclusively by sending and receiving messages over a high-speed interconnection network.

### Q2. Distinguish between a historical "pure" distributed-memory system and a modern HPC cluster.
> **Answer:**  
> A historical "pure" distributed-memory system consisted of single-core processors with local memory interconnected via standard Ethernet using operating system TCP/IP stacks, running one process per node. A modern HPC cluster is a hierarchical hybrid consisting of multi-socket, multi-core ccNUMA nodes communicating over dedicated high-speed low-latency fabrics (e.g. InfiniBand) using kernel-bypass RDMA protocols.

### Q3. What is Remote Direct Memory Access (RDMA)?
> **Answer:**  
> Remote Direct Memory Access (RDMA) is a network communications technology that allows the network adapter on one compute node to read or write data directly to/from the user-space memory of a remote compute node without operating system kernel intervention or CPU involvement, drastically reducing communication latency.

### Q4. What is a Fat-Tree network topology?
> **Answer:**  
> A Fat-Tree network topology is a hierarchical tree-structured interconnect where compute nodes reside at the leaves, and the bandwidth of the communication links progressively increases ("fattens") towards the top of the tree, providing constant bisection bandwidth and eliminating routing bottlenecks near the root.

---

## Conceptual Examination Questions

### Q1. Why is multithreading described in your notes as "easier than distributed memory programming"?
> **Model Answer:**  
> In shared-memory multithreading (e.g. OpenMP or Pthreads), all threads share a common address space. A developer can parallelize existing sequential code incrementally by adding compiler directives above loops without restructuring the underlying data structures. Variables are accessed directly through memory pointers.  
> In distributed memory (MPI), because address spaces are completely disjoint, the programmer must explicitly partition global data arrays across nodes, manage local index offsets, marshal data into message buffers, and explicitly coordinate communication using paired sends and receives. Any mistake in message ordering leads to deadlocks or silent data corruption.

### Q2. Explain why the latency hierarchy ($\lambda_{\text{intra}} < \lambda_{\text{inter-socket}} < \lambda_{\text{inter-node}}$) dictates the design of hybrid parallel software.
> **Model Answer:**  
> Intra-socket communication operates at nanosecond scale ($\approx 10\text{ ns}$), inter-socket at $\approx 100\text{ ns}$, while inter-node network communication operates at microsecond scale ($\approx 1,000\text{ to } 3,000\text{ ns}$).  
> If an application relies solely on network message passing for all fine-grained interactions, the network latency dominates, severely limiting speedup. Therefore, modern high-performance applications adopt **hybrid programming models (MPI + OpenMP)**:
> 1. Coarse-grained domain decomposition across compute nodes is handled by **MPI**.
> 2. Fine-grained loop parallelism and data sharing within each ccNUMA node/socket is handled by **OpenMP threads**, ensuring that high-frequency data sharing remains on high-bandwidth, sub-microsecond on-chip buses.

---

## Quick Revision Sheet

- **Distributed Memory Fundamental Principle:** No shared address space; processors exchange data strictly via explicit **message passing**.
- **Historical Pure System:** Single-core nodes, TCP/IP over Ethernet, 1 process per box, high latency ($50\ \mu\text{s}$).
- **Modern Cluster Structure:** Networked ccNUMA nodes, multi-core sockets, hardware RDMA fabrics, low latency ($1\ \mu\text{s}$).
- **The Communication Cost Hierarchy:**
  $$\boxed{\lambda_{\text{intra-socket}} \approx 10\text{ ns} \quad \ll \quad \lambda_{\text{inter-socket}} \approx 100\text{ ns} \quad \ll \quad \lambda_{\text{inter-node}} \approx 1,000 - 3,000\text{ ns}}$$
  $$\boxed{b_{\text{intra-socket}} \approx 500\text{ GB/s} \quad \gg \quad b_{\text{inter-socket}} \approx 60\text{ GB/s} \quad \gg \quad b_{\text{inter-node}} \approx 10 - 25\text{ GB/s}}$$
- **RDMA Benefits:** Kernel bypass + Zero copy $\implies$ microsecond latency.
- **Topologies:** Fat-Tree (standard, full bisection bandwidth) vs 3D Torus (nearest-neighbor scaling).

---

## What You Should Be Able to Do After This Topic

1. [ ] Define a distributed-memory system and contrast it with shared-memory systems.
2. [ ] Draw the architectural diagram of a "pure" historical distributed-memory system from your lecture notes.
3. [ ] Explain why pure distributed systems with TCP/IP were replaced by modern hierarchical clusters.
4. [ ] State the three-tier latency and bandwidth hierarchy (intra-socket, inter-socket, inter-node) with typical numbers.
5. [ ] Explain what RDMA is and how kernel-bypass networking reduces message passing latency.
6. [ ] Describe the Fat-Tree network topology and explain why upper links must be "fatter".
7. [ ] Explain why modern supercomputing codes combine MPI between nodes and OpenMP within nodes.

---

## Source References
- `Sanjay_Notes_1.pdf`: Page 11 (Multithreading easy in OpenMP vs general purpose, Distributed memory systems "back in the day", Pure distributed memory system diagram with $P, C, M, NI$, Communication network via TCP/IP and UDP/IP, One process per node, Distributed memory systems today: clusters of networked shared-memory ccNUMA nodes, Intra-socket, inter-socket, inter-node latency and bandwidth differences, Network structure effects).
- `IMG_1333.pdf`: Pages 16–17 (Distributed memory multiprocessor architecture, Nodes, RAM, Cores, Network messaging foundation for MPI).

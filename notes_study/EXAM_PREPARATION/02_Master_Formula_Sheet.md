# Master Formula Sheet & Mathematical Models

This comprehensive mathematical formula sheet consolidates all governing equations, algorithmic models, network topology metrics, and performance laws across the entire Parallel Computing syllabus.

---

## 1. Hardware Microarchitecture & Performance Metrics

### 1.1. Clock Frequency & Execution Time
$$\text{Cycle Time: } \tau = \frac{1}{f}$$
$$\text{CPU Execution Time: } T_{CPU} = \text{Instruction Count (IC)} \times \text{CPI} \times \tau = \frac{\text{IC} \times \text{CPI}}{f}$$

### 1.2. Average Cycles Per Instruction (CPI)
$$\text{CPI}_{avg} = \sum_{i=1}^{n} \left( \text{CPI}_i \times \frac{\text{Count}_i}{\text{IC}} \right) = \frac{\sum_{i=1}^n (\text{CPI}_i \times \text{Count}_i)}{\text{IC}}$$
$$\text{IPC} = \frac{1}{\text{CPI}}$$

### 1.3. Pipelining Performance Metrics
For a pipeline of depth $k$ executing $n$ independent instructions:
$$\text{Sequential Clock Cycles: } C_{seq} = n \times k$$
$$\text{Pipelined Clock Cycles: } C_{pipe} = k + (n - 1)$$
$$\text{Ideal Speedup: } S_k = \frac{C_{seq}}{C_{pipe}} = \frac{n \cdot k}{k + n - 1}$$
$$\lim_{n \to \infty} S_k = k$$
$$\text{Pipelined Throughput: } Th = \frac{n}{(k + n - 1) \cdot \tau}$$
$$\text{Pipeline Efficiency: } \eta_k = \frac{S_k}{k} = \frac{n}{k + n - 1}$$

### 1.4. Peak Floating-Point Performance ($R_{peak}$)
$$R_{peak} = N_{sockets} \times N_{cores/socket} \times f_{clock} \times \left(\frac{\text{SIMD Vector Width}}{\text{Data Word Size}}\right) \times \text{FMA Factor}$$
- *FMA Factor:* $2$ (1 Multiply + 1 Add per clock cycle)
- *AVX-512 with 64-bit Double Precision:* $\frac{512}{64} = 8\text{ DP elements} \times 2 = 16\text{ FLOP/cycle/core}$

### 1.5. Arithmetic Intensity ($I$)
$$I = \frac{\text{Total Floating-Point Operations (FLOPs)}}{\text{Total Main Memory Traffic (Bytes Transferred)}} \quad [\text{FLOP/byte}]$$

---

## 2. Interconnection Networks & Topologies

For a network of $P$ processing nodes:

| Topology | Node Degree ($d$) | Network Diameter ($D$) | Bisection Bandwidth ($B_w$) | Total Network Links |
| :--- | :---: | :---: | :---: | :---: |
| **Linear Bus** | $1$ | $P - 1$ | $1$ | $P - 1$ |
| **Ring** | $2$ | $\lfloor P/2 \rfloor$ | $2$ | $P$ |
| **2D Mesh ($k \times k, P=k^2$)** | $4$ (interior) | $2(k - 1) = 2(\sqrt{P} - 1)$ | $k = \sqrt{P}$ | $2P - 2\sqrt{P}$ |
| **2D Torus ($k \times k, P=k^2$)** | $4$ | $2 \lfloor k/2 \rfloor = 2 \lfloor \sqrt{P}/2 \rfloor$ | $2k = 2\sqrt{P}$ | $2P$ |
| **$d$-Dimensional Hypercube ($P = 2^d$)** | $d = \log_2 P$ | $d = \log_2 P$ | $\frac{P}{2} = 2^{d-1}$ | $\frac{P \cdot d}{2}$ |
| **Fat Tree ($k$-ary $n$-tree)** | $2k$ | $2n$ | Constant bisection | Non-blocking |

---

## 3. Parallel Performance Analysis & Scaling Laws

### 3.1. Fundamental Scaling Metrics
$$\text{Speedup: } S_P = \frac{T_1}{T_P}$$
$$\text{Parallel Efficiency: } E_P = \frac{S_P}{P} = \frac{T_1}{P \cdot T_P} \quad (0 \le E_P \le 1)$$
$$\text{Parallel Overhead: } T_o(P) = P \cdot T_P - T_1$$

### 3.2. Amdahl's Law (Strong Scaling — Fixed Problem Size)
Let serial fraction be $s$, and parallelizable fraction be $p = 1 - s$:
$$T_P = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{P}$$
$$\boxed{S_P = \frac{T_1}{T_P} = \frac{1}{s + \frac{1 - s}{P}}}$$
$$\boxed{\text{Asymptotic Speedup Limit: } S_{\infty} = \lim_{P \to \infty} S_P = \frac{1}{s}}$$

### 3.3. Amdahl's Law with Parallel Communication Overhead
When overhead scales with $P$ (e.g., $T_o(P) = c \cdot P$ or $c \cdot P^2$):
$$T_P = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{P} + T_o(P)$$
$$\text{Optimal Processor Count } P^*: \quad \frac{d T_P}{dP} = 0 \implies -\frac{(1 - s) T_1}{P^2} + \frac{d T_o(P)}{dP} = 0$$
$$\text{If } T_o(P) = c \cdot P \implies P^* = \sqrt{\frac{(1 - s) T_1}{c}}$$

### 3.4. Gustafson-Barsis's Law (Weak Scaling — Scaled Workload)
Let $s'$ be the serial fraction measured on the parallel system ($s' = \frac{T_{serial}}{T_P}$):
$$T_1 = s' \cdot T_P + P(1 - s') \cdot T_P$$
$$\boxed{S_{scaled} = \frac{T_1}{T_P} = s' + P(1 - s') = P - s'(P - 1)}$$

### 3.5. The Roofline Model
$$\boxed{P_{\text{achievable}} = \min\left(R_{peak}, \; I \times B_{mem}\right)}$$
$$\text{Machine Balance (Inflection Point): } B_m = \frac{R_{peak}}{B_{mem}} \quad [\text{FLOP/byte}]$$
- If $I < B_m \implies$ **Memory-Bound Region** ($P_{\text{achievable}} = I \times B_{mem}$)
- If $I \ge B_m \implies$ **Compute-Bound Region** ($P_{\text{achievable}} = R_{peak}$)

---

## 4. OpenMP Scheduling Models

For a loop with $N$ total iterations executing on $P$ threads:

### 4.1. Static Scheduling (`schedule(static, chunk)`)
$$\text{Total Chunks: } C_{total} = \lceil \frac{N}{\text{chunk}} \rceil$$
$$\text{Iteration } i \text{ assigned to Thread: } \text{Thread ID} = \left\lfloor \frac{i}{\text{chunk}} \right\rfloor \pmod P$$

### 4.2. Guided Scheduling (`schedule(guided, min_chunk)`)
Chunk size decays dynamically as remaining iterations decrease:
$$\text{Chunk Size for Step } k: \quad \text{chunk}_k = \max\left( \left\lceil \frac{R_k}{P} \right\rceil, \; \text{min\_chunk} \right)$$
*(where $R_k$ is the number of remaining, unassigned loop iterations).*

### 4.3. Reduction Identity Values
| Operator | Directive Syntax | Identity Element |
| :--- | :--- | :---: |
| **Addition** | `reduction(+: var)` | `0` |
| **Multiplication** | `reduction(*: var)` | `1` |
| **Subtraction** | `reduction(-: var)` | `0` |
| **Bitwise AND** | `reduction(&: var)` | `~0` (All 1s) |
| **Bitwise OR** | `reduction(|: var)` | `0` |
| **Logical AND** | `reduction(&&: var)` | `1` (`true`) |
| **Logical OR** | `reduction(||: var)` | `0` (`false`) |
| **Maximum** | `reduction(max: var)` | Smallest representable value |
| **Minimum** | `reduction(min: var)` | Largest representable value |

---

## 5. Distributed Memory & MPI Communication Models

### 5.1. Point-to-Point Message Transfer Time (Hockney Model)
For message size $m$ bytes across an interconnect with latency $\alpha$ and bandwidth $B$ ($\beta = \frac{1}{B}$):
$$\boxed{T_{msg} = \alpha + m \cdot \beta = \alpha + \frac{m}{B}}$$

### 5.2. Ping-Pong Experimental Benchmark Formulation
Given measured round-trip time $\text{RTT}(m)$ for message size $m$:
$$\text{One-Way Latency: } \alpha = \frac{\text{RTT}(0)}{2}$$
$$\text{Network Bandwidth: } B = \frac{m}{\frac{\text{RTT}(m)}{2} - \alpha}$$

### 5.3. Algorithmic Collective Communication Topologies

#### 1. Broadcast (`MPI_Bcast`)
- **Naive Linear Loop:**
  $$T_{\text{linear}} = (P - 1) \cdot (\alpha + m \cdot \beta) \implies \mathcal{O}(P)$$
- **Binomial Spanning Tree:**
  $$T_{\text{binomial}} = \lceil \log_2 P \rceil \cdot (\alpha + m \cdot \beta) \implies \mathcal{O}(\log_2 P)$$
- **Van de Geijn (Scatter-Allgather) Pipeline (Large $m$):**
  $$T_{\text{pipelined}} = 2 \lceil \log_2 P \rceil \alpha + 2 \left(\frac{P - 1}{P}\right) m \beta$$

#### 2. Reduction (`MPI_Reduce`)
- **Binomial Tree Reduction:**
  $$T_{\text{reduce}} = \lceil \log_2 P \rceil \cdot (\alpha + m \cdot \beta + m \cdot \gamma_{\text{compute}})$$
  *(where $\gamma_{\text{compute}}$ is the compute time per element).*

#### 3. Allreduce (`MPI_Allreduce`)
- **Recursive Doubling (Hypercube Exchange):**
  $$T_{\text{allreduce}} = \log_2 P \cdot (\alpha + m \cdot \beta + m \cdot \gamma_{\text{compute}})$$

#### 4. Scatter & Gather (`MPI_Scatter` / `MPI_Gather`)
- Total data across cluster is $M = P \cdot m_{\text{local}}$ bytes:
  $$T_{\text{scatter}} = \lceil \log_2 P \rceil \alpha + \left(\frac{P - 1}{P}\right) M \beta$$

---

## 6. Quick University Exam Formula Matrix

| Concept | Governing Formula | Key Variable Meanings |
| :--- | :--- | :--- |
| **Amdahl's Law** | $S_P = \frac{1}{s + \frac{1-s}{P}}$ | $s = \text{serial fraction}, P = \text{processors}$ |
| **Gustafson's Law** | $S_{scaled} = P - s'(P - 1)$ | $s' = \text{measured serial fraction}$ |
| **Roofline Performance** | $P = \min(R_{peak}, I \times B_{mem})$ | $I = \text{FLOP/byte}, B_{mem} = \text{GB/s}$ |
| **Pipeline Speedup** | $S_k = \frac{n \cdot k}{k + n - 1}$ | $k = \text{stages}, n = \text{instructions}$ |
| **Point-to-Point Latency** | $T = \alpha + \frac{m}{B}$ | $\alpha = \text{latency (s)}, B = \text{bandwidth (B/s)}$ |
| **Binomial Broadcast** | $T = \lceil \log_2 P \rceil (\alpha + m\beta)$ | $\log_2 P = \text{tree depth}$ |

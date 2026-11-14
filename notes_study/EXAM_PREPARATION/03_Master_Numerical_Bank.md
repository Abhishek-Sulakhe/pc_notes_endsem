# Master Numerical Problem Bank & Step-by-Step Solutions

This numerical problem bank contains comprehensive, university-examination-level problems with full step-by-step mathematical derivations, formulas, substitutions, and boxed answers.

---

## Numerical Problem 1: Instruction Mix, Average CPI & Execution Time

### Problem Statement:
A computer program executes $1.5 \times 10^9$ instructions on a $3.0\text{ GHz}$ uniprocessor.  
The compiler generates the following instruction mix:

| Instruction Class | Clock Cycles Per Instruction (CPI) | Mix Percentage |
| :--- | :---: | :---: |
| **Arithmetic / Logic (ALU)** | $1$ | $45\%$ |
| **Data Transfer (Load/Store)** | $4$ | $30\%$ |
| **Control Transfer (Branch/Jump)** | $2$ | $15\%$ |
| **Floating-Point Arithmetic** | $6$ | $10\%$ |

Calculate:
1. The average Cycles Per Instruction ($\text{CPI}_{avg}$).
2. The overall instruction execution rate in MIPS (Million Instructions Per Second).
3. The total CPU execution time ($T_{CPU}$) in seconds.

---

### Step-by-Step Solution:

**Step 1: Compute Average CPI ($\text{CPI}_{avg}$):**
$$\text{CPI}_{avg} = \sum_{i=1}^4 (\text{CPI}_i \times f_i)$$
$$\text{CPI}_{avg} = (1 \times 0.45) + (4 \times 0.30) + (2 \times 0.15) + (6 \times 0.10)$$
$$\text{CPI}_{avg} = 0.45 + 1.20 + 0.30 + 0.60 = 2.55\text{ cycles/instruction}$$

---

**Step 2: Compute Total CPU Execution Time ($T_{CPU}$):**
Clock frequency $f = 3.0\text{ GHz} = 3.0 \times 10^9\text{ Hz}$. Cycle time $\tau = \frac{1}{f} = \frac{1}{3.0 \times 10^9}\text{ s}$.
$$T_{CPU} = \frac{\text{Instruction Count (IC)} \times \text{CPI}_{avg}}{f}$$
$$T_{CPU} = \frac{(1.5 \times 10^9) \times 2.55}{3.0 \times 10^9} = \frac{3.825 \times 10^9}{3.0 \times 10^9} = 1.275\text{ seconds}$$

---

**Step 3: Compute Instruction Execution Rate in MIPS:**
$$\text{MIPS} = \frac{\text{IC}}{T_{CPU} \times 10^6} = \frac{f}{\text{CPI}_{avg} \times 10^6}$$
$$\text{MIPS} = \frac{3.0 \times 10^9}{2.55 \times 10^6} \approx 1,176.47\text{ MIPS}$$

$$\boxed{\text{CPI}_{avg} = 2.55, \quad T_{CPU} = 1.275\text{ s}, \quad \text{Throughput} = 1,176.47\text{ MIPS}}$$

---

## Numerical Problem 2: Pipeline Speedup, Stalls & Hazard Penalties

### Problem Statement:
A 5-stage instruction pipeline ($k = 5$) operates with a clock cycle time of $\tau = 2\text{ ns}$.  
An application consisting of $n = 100,000$ instructions is executed.
1. Compute the execution time, speedup ($S_k$), and efficiency ($\eta_k$) for an **ideal pipeline** with no stalls compared to unpipelined sequential execution.
2. Suppose $20\%$ of instructions are conditional branches. The branch outcome is resolved in stage 3, incurring a **2-cycle penalty on misprediction**. The branch prediction accuracy is $80\%$. Compute the actual execution time and speedup under these branch stalls.

---

### Step-by-Step Solution:

**Part 1: Ideal Pipeline Performance**
- Sequential clock cycles: $C_{seq} = n \times k = 100,000 \times 5 = 500,000\text{ cycles}$.
- Pipelined clock cycles: $C_{pipe} = k + (n - 1) = 5 + 99,999 = 100,004\text{ cycles}$.
- Sequential execution time: $T_{seq} = 500,000 \times 2\text{ ns} = 1,000\text{ }\mu\text{s} = 1.0\text{ ms}$.
- Ideal pipelined time: $T_{pipe} = 100,004 \times 2\text{ ns} = 200.008\text{ }\mu\text{s} \approx 0.20\text{ ms}$.
- Speedup: $S_k = \frac{C_{seq}}{C_{pipe}} = \frac{500,000}{100,004} \approx 4.9998 \approx 5.0$.
- Efficiency: $\eta_k = \frac{S_k}{k} = \frac{4.9998}{5} = 99.99\%$.

---

**Part 2: Real Pipeline with Branch Hazards**
- Number of branch instructions: $N_{branch} = 0.20 \times 100,000 = 20,000\text{ branches}$.
- Misprediction rate: $1 - 0.80 = 0.20$ ($20\%$ mispredicted).
- Number of mispredicted branches: $N_{mispredict} = 20,000 \times 0.20 = 4,000\text{ branches}$.
- Total stall penalty cycles: $C_{stall} = 4,000 \times 2\text{ stall cycles} = 8,000\text{ cycles}$.
- Total pipelined cycles: $C_{actual} = C_{pipe} + C_{stall} = 100,004 + 8,000 = 108,004\text{ cycles}$.
- Actual execution time:
  $$T_{actual} = 108,004 \times 2\text{ ns} = 216,008\text{ ns} = 216.008\text{ }\mu\text{s}$$
- Actual speedup:
  $$S_{actual} = \frac{C_{seq}}{C_{actual}} = \frac{500,000}{108,004} \approx 4.63$$

$$\boxed{\text{Ideal Speedup} = 5.0\times, \quad \text{Actual Time with Stalls} = 216.01\text{ }\mu\text{s}, \quad \text{Actual Speedup} = 4.63\times}$$

---

## Numerical Problem 3: Theoretical Peak Performance ($R_{peak}$) & The Roofline Model

### Problem Statement:
A compute cluster node is equipped with a dual-socket Intel Xeon processor.
- Sockets: $2$
- Cores per socket: $18$ ($36$ physical cores total)
- Base clock frequency: $f = 2.4\text{ GHz} = 2.4 \times 10^9\text{ Hz}$
- Vector Instruction Set: AVX-512 ($512$-bit SIMD registers)
- Fused Multiply-Add (FMA) support: Enabled (counts as 2 FLOPs/cycle)
- Measured Memory Bandwidth ($B_{mem}$): $150\text{ GB/s}$

A scientific kernel executes the **DAXPY** operation ($Y[i] = a \cdot X[i] + Y[i]$) on double-precision 64-bit floating-point numbers.
1. Calculate the theoretical peak floating-point performance ($R_{peak}$) of the server in $\text{TFLOP/s}$.
2. Calculate the Arithmetic Intensity ($I$) of the DAXPY kernel in $\text{FLOP/byte}$.
3. Calculate the Machine Balance ($B_m$) of the system.
4. Using the Roofline Model, determine whether DAXPY is memory-bound or compute-bound, and calculate its maximum achievable performance in $\text{GFLOP/s}$.

---

### Step-by-Step Solution:

**Step 1: Compute Peak Floating-Point Performance ($R_{peak}$):**
Each AVX-512 vector register holds:
$$\text{DP Elements per Vector} = \frac{512\text{ bits}}{64\text{ bits/element}} = 8\text{ double-precision elements}$$
With FMA, each core executes $8 \times 2 = 16\text{ FLOPs/cycle}$.
$$R_{peak} = N_{sockets} \times N_{cores/socket} \times f \times \text{FLOPs/cycle/core}$$
$$R_{peak} = 2 \times 18 \times (2.4 \times 10^9) \times 16$$
$$R_{peak} = 36 \times 2.4 \times 10^9 \times 16 = 1.3824 \times 10^{12}\text{ FLOP/s} = 1.3824\text{ TFLOP/s}$$

---

**Step 2: Compute Arithmetic Intensity ($I$) of DAXPY:**
For each array element $i$:
- Operations: 1 floating-point multiplication ($a \cdot X[i]$) + 1 floating-point addition ($+ Y[i]$) = **$2\text{ FLOPs}$**.
- Memory Traffic:
  - Load $X[i]$: $8\text{ bytes}$
  - Load $Y[i]$: $8\text{ bytes}$
  - Store updated $Y[i]$: $8\text{ bytes}$
  - Total Memory Transferred: $8 + 8 + 8 = 24\text{ bytes}$.
$$I = \frac{2\text{ FLOPs}}{24\text{ bytes}} = \frac{1}{12} \approx 0.0833\text{ FLOP/byte}$$

---

**Step 3: Compute Machine Balance ($B_m$):**
$$B_m = \frac{R_{peak}}{B_{mem}} = \frac{1,382.4\text{ GFLOP/s}}{150\text{ GB/s}} = 9.216\text{ FLOP/byte}$$

---

**Step 4: Determine Operational Bound & Achievable Performance:**
Comparing $I$ with $B_m$:
$$I = 0.0833\text{ FLOP/byte} \ll B_m = 9.216\text{ FLOP/byte}$$
Because $I < B_m$, DAXPY is severely **Memory-Bandwidth Bound**.

Achievable performance is bounded by the memory ceiling:
$$P_{achievable} = I \times B_{mem} = \frac{1}{12}\text{ FLOP/byte} \times 150\text{ GB/s} = 12.50\text{ GFLOP/s}$$
Efficiency relative to $R_{peak}$:
$$\text{Efficiency} = \frac{12.50\text{ GFLOP/s}}{1,382.4\text{ GFLOP/s}} \approx 0.90\%$$

$$\boxed{R_{peak} = 1.3824\text{ TFLOP/s}, \quad I = 0.0833\text{ FLOP/B}, \quad \text{Bound: MEMORY-BOUND}, \quad P_{achievable} = 12.50\text{ GFLOP/s}}$$

---

## Numerical Problem 4: Interconnection Network Metrics Comparison

### Problem Statement:
An HPC cluster consists of $P = 64$ processor nodes.  
Compare the architectural properties of connecting these 64 nodes using:
1. A unidirectional Ring.
2. An $8 \times 8$ 2D Mesh ($k = 8$).
3. An $8 \times 8$ 2D Torus ($k = 8$).
4. A 6-dimensional Hypercube ($2^6 = 64$).

For each topology, calculate:
- Node degree ($d$) of interior nodes.
- Network diameter ($D$).
- Bisection width ($B_w$).
- Total number of physical bidirectional communication links ($E$).

---

### Step-by-Step Solution:

#### 1. Unidirectional / Bidirectional Ring ($P = 64$):
- **Node Degree:** $d = 2$ (each node connects to left and right neighbors).
- **Network Diameter:** $D = \lfloor \frac{P}{2} \rfloor = \lfloor \frac{64}{2} \rfloor = 32\text{ hops}$.
- **Bisection Width:** $B_w = 2$ links (cutting the ring in half cuts 2 links).
- **Total Links:** $E = P = 64\text{ links}$.

#### 2. 2D Mesh ($8 \times 8, k = 8$):
- **Node Degree:** $d = 4$ for interior nodes (corners have 2, edges have 3).
- **Network Diameter:** $D = 2(k - 1) = 2(8 - 1) = 14\text{ hops}$.
- **Bisection Width:** $B_w = k = 8\text{ links}$.
- **Total Links:** $E = 2P - 2\sqrt{P} = 2(64) - 2(8) = 128 - 16 = 112\text{ links}$.

#### 3. 2D Torus ($8 \times 8, k = 8$):
- **Node Degree:** $d = 4$ (wraparound connections make all nodes identical).
- **Network Diameter:** $D = 2 \lfloor \frac{k}{2} \rfloor = 2 \lfloor \frac{8}{2} \rfloor = 8\text{ hops}$.
- **Bisection Width:** $B_w = 2k = 2(8) = 16\text{ links}$.
- **Total Links:** $E = 2P = 2(64) = 128\text{ links}$.

#### 4. 6-Dimensional Hypercube ($d = 6, P = 2^6 = 64$):
- **Node Degree:** $d = \log_2 P = \log_2 64 = 6$.
- **Network Diameter:** $D = \log_2 P = 6\text{ hops}$.
- **Bisection Width:** $B_w = \frac{P}{2} = \frac{64}{2} = 32\text{ links}$.
- **Total Links:** $E = \frac{P \times d}{2} = \frac{64 \times 6}{2} = 192\text{ links}$.

### Master Summary Comparison Table:
| Metric | Ring | 2D Mesh ($8 \times 8$) | 2D Torus ($8 \times 8$) | 6D Hypercube |
| :--- | :---: | :---: | :---: | :---: |
| **Node Degree ($d$)** | $2$ | $4$ | $4$ | $\mathbf{6}$ |
| **Network Diameter ($D$)** | $32$ | $14$ | $8$ | $\mathbf{6}$ |
| **Bisection Width ($B_w$)** | $2$ | $8$ | $16$ | $\mathbf{32}$ |
| **Total Links ($E$)** | $64$ | $112$ | $128$ | $\mathbf{192}$ |

---

## Numerical Problem 5: Amdahl's Law vs. Gustafson-Barsis's Law

### Problem Statement:
A parallel program has a sequential fraction $s = 0.08$ ($8\%$ serial execution).
1. Under **Amdahl's Law** (Strong Scaling), calculate the theoretical speedup ($S_P$) and parallel efficiency ($E_P$) for $P = 8, 16, 64, 256$, and $\infty$.
2. Under **Gustafson-Barsis's Law** (Weak Scaling), calculate the scaled speedup ($S_{scaled}$) for the same processor counts assuming $s' = 0.08$.

---

### Step-by-Step Solution:

**Part 1: Amdahl's Law Calculations:**
Formula: $S_P = \frac{1}{0.08 + \frac{0.92}{P}}$, \quad $E_P = \frac{S_P}{P}$
- For $P = 8$:  
  $S_8 = \frac{1}{0.08 + \frac{0.92}{8}} = \frac{1}{0.08 + 0.115} = \frac{1}{0.195} \approx 5.13\times$, \quad $E_8 = \frac{5.13}{8} = 64.1\%$
- For $P = 16$:  
  $S_{16} = \frac{1}{0.08 + \frac{0.92}{16}} = \frac{1}{0.08 + 0.0575} = \frac{1}{0.1375} \approx 7.27\times$, \quad $E_{16} = \frac{7.27}{16} = 45.4\%$
- For $P = 64$:  
  $S_{64} = \frac{1}{0.08 + \frac{0.92}{64}} = \frac{1}{0.08 + 0.014375} = \frac{1}{0.094375} \approx 10.60\times$, \quad $E_{64} = \frac{10.60}{64} = 16.6\%$
- For $P = 256$:  
  $S_{256} = \frac{1}{0.08 + \frac{0.92}{256}} = \frac{1}{0.08 + 0.00359} = \frac{1}{0.08359} \approx 11.96\times$, \quad $E_{256} = \frac{11.96}{256} = 4.7\%$
- For $P = \infty$:  
  $S_{\infty} = \frac{1}{s} = \frac{1}{0.08} = 12.50\times$, \quad $E_{\infty} \to 0\%$

---

**Part 2: Gustafson-Barsis's Law Calculations:**
Formula: $S_{scaled} = P - s'(P - 1) = P - 0.08(P - 1)$
- For $P = 8$:  
  $S_{scaled} = 8 - 0.08(7) = 8 - 0.56 = 7.44\times$
- For $P = 16$:  
  $S_{scaled} = 16 - 0.08(15) = 16 - 1.20 = 14.80\times$
- For $P = 64$:  
  $S_{scaled} = 64 - 0.08(63) = 64 - 5.04 = 58.96\times$
- For $P = 256$:  
  $S_{scaled} = 256 - 0.08(255) = 256 - 20.40 = 235.60\times$

### Comparative Result Table:
| Processors ($P$) | Amdahl Speedup ($S_P$) | Amdahl Efficiency ($E_P$) | Gustafson Scaled Speedup ($S_{scaled}$) |
| :---: | :---: | :---: | :---: |
| **8** | $5.13\times$ | $64.1\%$ | $7.44\times$ |
| **16** | $7.27\times$ | $45.4\%$ | $14.80\times$ |
| **64** | $10.60\times$ | $16.6\%$ | $58.96\times$ |
| **256** | $11.96\times$ | $4.7\%$ | $235.60\times$ |
| **$\infty$** | $\mathbf{12.50\times}$ (Ceiling) | $\mathbf{0.0\%}$ | $\mathbf{\infty}$ (Scales with Workload) |

---

## Numerical Problem 6: Amdahl's Law with Overhead & Optimal Processor Count ($P^*$)

### Problem Statement:
A finite-difference simulation has a sequential runtime on 1 core of $T_1 = 200\text{ seconds}$.  
The serial fraction is $s = 0.04$ ($4\%$).  
When parallelized across $P$ processors, inter-node barrier synchronization and cache-invalidation overhead introduces a latency penalty modeled by:
$$T_o(P) = 0.005 \cdot P^2\text{ seconds}$$

1. Formulate the parallel execution time function $T_P(P)$.
2. Calculate the optimal number of processors $P^*$ that minimizes total execution time.
3. Calculate the minimum execution time $T_{min}$ and the maximum achievable speedup $S_{max}$ at $P^*$.
4. Show what happens to performance if the user over-allocates the cluster to $P = 100$ processors.

---

### Step-by-Step Solution:

**Step 1: Formulate $T_P(P)$:**
$$T_P(P) = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{P} + T_o(P)$$
$$T_P(P) = (0.04 \times 200) + \frac{(0.96 \times 200)}{P} + 0.005 P^2$$
$$T_P(P) = 8 + \frac{192}{P} + 0.005 P^2$$

---

**Step 2: Find Optimal Core Count $P^*$:**
Differentiate $T_P$ with respect to $P$ and set to zero:
$$\frac{d T_P}{dP} = -\frac{192}{P^2} + 2 \times 0.005 P = 0$$
$$\frac{192}{P^2} = 0.01 P \implies P^3 = \frac{192}{0.01} = 19,200$$
$$P^* = \sqrt[3]{19,200} \approx 26.78$$

Since processor counts must be integers, evaluate $P = 26$ and $P = 27$:
- For $P = 26$: $T_{26} = 8 + \frac{192}{26} + 0.005(26^2) = 8 + 7.3846 + 3.3800 = 18.7646\text{ s}$
- For $P = 27$: $T_{27} = 8 + \frac{192}{27} + 0.005(27^2) = 8 + 7.1111 + 3.6450 = 18.7561\text{ s}$ (Minimum!)

$$P^* = 27\text{ processors}$$

---

**Step 3: Compute Minimum Runtime and Maximum Speedup:**
$$T_{min} = T_{27} = 18.76\text{ seconds}$$
$$S_{max} = \frac{T_1}{T_{min}} = \frac{200\text{ s}}{18.7561\text{ s}} \approx 10.66\times$$

---

**Step 4: Performance at $P = 100$ (Contention Collapse):**
$$T_{100} = 8 + \frac{192}{100} + 0.005(100^2) = 8 + 1.92 + 50.0 = 59.92\text{ seconds}$$
$$S_{100} = \frac{200}{59.92} \approx 3.34\times$$

At 100 cores, synchronization overhead ($50\text{ s}$) dwarfs the actual compute work ($1.92\text{ s}$), collapsing speedup from $10.66\times$ down to $3.34\times$.

$$\boxed{P^* = 27\text{ processors}, \quad T_{min} = 18.76\text{ s}, \quad S_{max} = 10.66\times, \quad S_{100} = 3.34\times}$$

---

## Numerical Problem 7: OpenMP Guided Scheduling Chunk Allocation Trace

### Problem Statement:
An OpenMP parallel loop contains $N = 400$ iterations and is executed on $P = 4$ threads using the clause:
```c
#pragma omp for schedule(guided, 8)
```
Recall the guided scheduling chunk formula:
$$\text{chunk}_k = \max\left( \left\lceil \frac{R_k}{P} \right\rceil, \; \text{min\_chunk} \right)$$
Trace the chunk sizes assigned to threads for the first 6 assignments.

---

### Step-by-Step Solution:

- **Initial State:** Remaining iterations $R_0 = 400$, Threads $P = 4$, $\text{min\_chunk} = 8$.

1. **Assignment 1:**
   $$\text{chunk}_1 = \max\left( \lceil \frac{400}{4} \rceil, 8 \right) = \max(100, 8) = \mathbf{100}$$
   $$R_1 = 400 - 100 = 300\text{ iterations remaining}$$

2. **Assignment 2:**
   $$\text{chunk}_2 = \max\left( \lceil \frac{300}{4} \rceil, 8 \right) = \max(75, 8) = \mathbf{75}$$
   $$R_2 = 300 - 75 = 225\text{ iterations remaining}$$

3. **Assignment 3:**
   $$\text{chunk}_3 = \max\left( \lceil \frac{225}{4} \rceil, 8 \right) = \max(57, 8) = \mathbf{57}$$
   $$R_3 = 225 - 57 = 168\text{ iterations remaining}$$

4. **Assignment 4:**
   $$\text{chunk}_4 = \max\left( \lceil \frac{168}{4} \rceil, 8 \right) = \max(42, 8) = \mathbf{42}$$
   $$R_4 = 168 - 42 = 126\text{ iterations remaining}$$

5. **Assignment 5:**
   $$\text{chunk}_5 = \max\left( \lceil \frac{126}{4} \rceil, 8 \right) = \max(32, 8) = \mathbf{32}$$
   $$R_5 = 126 - 32 = 94\text{ iterations remaining}$$

6. **Assignment 6:**
   $$\text{chunk}_6 = \max\left( \lceil \frac{94}{4} \rceil, 8 \right) = \max(24, 8) = \mathbf{24}$$
   $$R_6 = 94 - 24 = 70\text{ iterations remaining}$$

$$\boxed{\text{Assigned Chunks: } [100, \; 75, \; 57, \; 42, \; 32, \; 24]}$$

---

## Numerical Problem 8: MPI Hockney Model & Ping-Pong Benchmarking

### Problem Statement:
An MPI point-to-point ping-pong benchmark executes between two compute nodes connected by an InfiniBand network:
- A $0$-byte message ping-pong yields a Round-Trip Time of $\text{RTT}(0) = 4.8\text{ }\mu\text{s}$.
- A $4\text{ MB}$ ($4 \times 10^6\text{ bytes}$) message ping-pong yields a Round-Trip Time of $\text{RTT}(4\text{ MB}) = 1,604.8\text{ }\mu\text{s}$.

1. Derive the one-way network latency $\alpha$ in microseconds.
2. Calculate the effective network bandwidth $B$ in $\text{GB/s}$ ($10^9\text{ bytes/s}$).
3. Predict the time required to send a single $500\text{ KB}$ ($500 \times 10^3\text{ bytes}$) message one-way.

---

### Step-by-Step Solution:

**Step 1: Compute One-Way Latency ($\alpha$):**
$$\text{One-Way Time } T(0) = \frac{\text{RTT}(0)}{2} = \frac{4.8\text{ }\mu\text{s}}{2} = 2.4\text{ }\mu\text{s} = 2.4 \times 10^{-6}\text{ seconds}$$

---

**Step 2: Compute Network Bandwidth ($B$):**
One-way transfer time for $4\text{ MB}$:
$$T(4\text{ MB}) = \frac{\text{RTT}(4\text{ MB})}{2} = \frac{1,604.8\text{ }\mu\text{s}}{2} = 802.4\text{ }\mu\text{s} = 8.024 \times 10^{-4}\text{ seconds}$$
Using Hockney's model $T(m) = \alpha + \frac{m}{B}$:
$$8.024 \times 10^{-4} = 2.4 \times 10^{-6} + \frac{4 \times 10^6}{B}$$
$$\frac{4 \times 10^6}{B} = 8.024 \times 10^{-4} - 0.024 \times 10^{-4} = 8.0 \times 10^{-4}\text{ seconds}$$
$$B = \frac{4 \times 10^6\text{ bytes}}{8.0 \times 10^{-4}\text{ s}} = 0.5 \times 10^{10}\text{ bytes/s} = 5.0\text{ GB/s} \quad (5 \times 10^9\text{ B/s})$$

---

**Step 3: Predict One-Way Time for $500\text{ KB}$:**
$$T(500\text{ KB}) = \alpha + \frac{m}{B}$$
$$T(500\text{ KB}) = 2.4 \times 10^{-6}\text{ s} + \frac{500 \times 10^3\text{ bytes}}{5.0 \times 10^9\text{ bytes/s}}$$
$$T(500\text{ KB}) = 2.4\text{ }\mu\text{s} + 100.0\text{ }\mu\text{s} = 102.4\text{ }\mu\text{s}$$

$$\boxed{\alpha = 2.4\text{ }\mu\text{s}, \quad B = 5.0\text{ GB/s}, \quad T(500\text{ KB}) = 102.4\text{ }\mu\text{s}}$$

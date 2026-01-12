# **System Design Case Study: Scaling a Single-Node API**

Author: Akash  
Date: January 12, 2026  
Environment: WSL 2 (Ubuntu) on Windows, 8-Core CPU

## **1\. Introduction**

The goal of this experiment was to determine the maximum throughput (Requests Per Second) of a simple HTTP "Hello World" API on a single machine. We started with a standard single-threaded implementation and iteratively identified bottlenecks, applied optimizations, and scaled the system vertically until we hit the physical limits of the hardware and network stack.

**Tools Used:**

* **Server:** Node.js (v20+), Bun (v1.x)  
* **Benchmark Tool:** wrk (HTTP benchmarking tool)  
* **OS:** Linux (Ubuntu via WSL 2\)

## ---

**2\. Phase 1: The Baseline (Single Threaded)**

We began with a standard Express.js server running on a single CPU core.

### **The Setup**

* **Code:** Basic Express.js res.send('Hello World').  
* **Configuration:** Defaults.  
* **Command:** wrk \-t12 \-c400 \-d30s ...

### **Results**

| Metric | Result |
| :---- | :---- |
| **Throughput** | \~11,000 req/sec |
| **Latency (Avg)** | 41.70 ms |
| **Errors** | 0 |

### **Bottleneck Analysis**

* **Observation:** Increasing connections from 400 to 4,000 did not increase throughput.  
* **Root Cause:** **CPU Saturation.** Node.js is single-threaded. The single CPU core was at 100% usage processing the Event Loop, limiting the server to 11k RPS.

## ---

**3\. Phase 2: Vertical Scaling (Clustering)**

To overcome the single-core limit, we utilized the Node.js cluster module to fork worker processes equal to the number of available CPU cores (4 workers utilized).

### **The Setup**

* **Code:** Added cluster module to fork workers.  
* **Command:** wrk \-t15 \-c40000 ... (Increased load to 40k connections).

### **Results**

| Metric | Result | Change |
| :---- | :---- | :---- |
| **Throughput** | \~28,136 req/sec | **\+155%** |
| **Latency (Avg)** | 345.29 ms | Increased (Queuing) |
| **Connect Errors** | **11,758** | **CRITICAL FAILURE** |

### **Bottleneck Analysis**

* **Observation:** Throughput more than doubled, proving vertical scaling worked. However, nearly 30% of connections failed immediately.  
* **Root Cause:** **Ephemeral Port Exhaustion.** The OS ran out of available ports (default range is \~28k) to assign to the test client. The OS rejected new TCP connections.

## ---

**4\. Phase 3: OS Level Tuning**

We needed to tune the Operating System to handle high concurrency networking.

### **Optimizations Applied**

1. **Increased File Descriptors:** ulimit \-n 65535 (Prevented "Too many open files").  
2. **Expanded Port Range:** sysctl \-w net.ipv4.ip\_local\_port\_range="1024 65000".  
3. **Enabled Port Reuse:** sysctl \-w net.ipv4.tcp\_tw\_reuse=1 (Allows rapid reuse of TIME\_WAIT sockets).

### **Results**

| Metric | Result | Change |
| :---- | :---- | :---- |
| **Throughput** | \~34,033 req/sec | **\+21%** |
| **Latency (Avg)** | 101.71 ms | **\-70% (Improved)** |
| **Connect Errors** | **0** | **FIXED** |

### **Bottleneck Analysis**

* **Observation:** Errors vanished. Throughput hit a stable ceiling of \~34k RPS.  
* **Root Cause:** **Software Overhead.** The hardware was fine, but the Express.js framework and Node's object allocation were now the limiting factor.

## ---

**5\. Phase 4: Code & Network Optimization**

To squeeze more performance from the CPU, we removed the Express.js framework and optimized how data was sent.

### **Optimizations Applied**

1. **Removed Express:** Switched to raw Node.js http module.  
2. **Static Buffers:** Pre-allocated Buffer.from('Hello World') to avoid creating strings/buffers on every request (CPU savings).  
3. **Increased Backlog:** server.listen(..., 65535\) to allow a larger kernel queue for bursting traffic.

### **Results**

| Metric | Result | Change |
| :---- | :---- | :---- |
| **Throughput** | \~37,934 req/sec | **\+11%** |
| **Latency (Avg)** | 62.96 ms | **\-38% (Faster)** |
| **Errors** | 0 | Stable |

### **Bottleneck Analysis**

* **Observation:** Latency dropped significantly. Throughput inched up.  
* **Root Cause:** **Runtime Limit.** We reached the maximum execution speed of the V8 JavaScript engine for HTTP parsing on this hardware.

## ---

**6\. Phase 5: Runtime Change (Switch to Bun)**

To break the limits of V8/Node.js, we switched the runtime to **Bun**, which uses a faster native HTTP implementation (written in Zig/C++).

### **The Setup**

* **Code:** Bun.serve({ fetch(req) { return new Response("Hello World"); } })  
* **Load:** Tuned to 10,000 connections to avoid network saturation (Little's Law optimization).

### **Results (The "Sweet Spot")**

| Metric | Result | Change vs Node |
| :---- | :---- | :---- |
| **Throughput** | **67,573 req/sec** | **\+78%** |
| **Latency (Avg)** | 151.14 ms | Higher (Network Queuing) |
| **Max Throughput** | \~81,000 req/sec | (Unstable with errors) |

### **Bottleneck Analysis**

* **Observation:** Throughput nearly doubled.  
* **Root Cause:** **Virtualization Limit.** The bottleneck is no longer the code or the runtime, but the **WSL 2 Virtual Network Bridge**. The VM simply cannot forward packets faster than 80k/sec without dropping them.

## ---

**7\. Comparative Summary**

| Phase | Description | RPS | Latency (Avg) | Limiting Factor |
| :---- | :---- | :---- | :---- | :---- |
| **1** | Single CPU / Default | 11,000 | 41ms | **CPU (Single Core)** |
| **2** | Clustered (4 CPUs) | 28,136 | 345ms | **OS (Ports/FDs)** |
| **3** | OS Tuned | 34,033 | 101ms | **Framework (Express)** |
| **4** | Code Optimized | 37,934 | 62ms | **Runtime (Node/V8)** |
| **5** | **Bun Runtime** | **67,573** | 151ms | **Network / VM** |

## ---

**8\. Conclusion**

This benchmark demonstrates that performance is rarely defined by a single metric; it is a series of bottlenecks.

1. **Hardware is rarely the first problem:** We 3x'd performance (11k \-\> 34k) just by fixing software and OS configuration.  
2. **The "Hidden" OS:** Limits like file descriptors and ephemeral ports are critical scaling walls that code cannot fix.  
3. **Runtimes Matter:** Switching from Node.js to Bun provided a massive 78% boost, but only *after* the OS was tuned to handle it.  
4. **Diminishing Returns:** At 67k RPS, we hit the limits of the virtualization environment (WSL). Further scaling would require **Horizontal Scaling** (adding more servers) or running on Bare Metal Linux.
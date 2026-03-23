# 🚀 Senior Golang Architect Interview Preparation Guide

This comprehensive guide is tailored for **Senior/Architect level** roles, specifically focusing on high-performance backend systems, IIoT (Industrial IoT), Edge-to-Cloud integration, and Scalable Microservices.

---

## 🏗 Part 1: General Advanced Golang (50 Questions)

### 1. Concurrency & The Scheduler
1.  **Blocking Syscalls vs. Network I/O:** How does the M:P:G model handle them?
    * *Deep Dive:* For syscalls, P detaches from M. For Network I/O, G is parked in the Netpoller (epoll/kqueue).
2.  **Work-Stealing:** Conditions under which a Processor (P) steals from others.
3.  **Lock-Free Design:** Implementing queues with `sync/atomic` and avoiding the ABA problem.
4.  **Channel Internals:** The `hchan` struct, mutex usage, and memory copying mechanics.
5.  **Signal Primivites:** When to use `sync.Cond` vs. Channels.
6.  **Leak Detection:** Using `pprof` and `runtime.NumGoroutine()` to find leaks.
7.  **Memory Model:** The "Happens-Before" guarantees in lockless code.
8.  **Context Tree:** Mechanism of recursive cancellation in `context.Context`.
9.  **Map Contention:** Implementing Sharding/Partitioning to avoid global locks.
10. **sync.Pool:** Internal "victim cache" and behavior across GC cycles.

### 2. Memory & Garbage Collection (11-20)
* Topics: Escape Analysis, Stack vs Heap, Concurrent Mark-and-Sweep, Write Barriers, Memory Alignment/Padding, `GOGC` vs `GOMEMLIMIT`, CGO overhead.

### 3. Language Mechanics & Internals (21-30)
* Topics: `iface` vs `eface` memory layout, Generics (Monomorphization), Reflection performance, Map bucket evacuation, Slice growth algorithms.

### 4. Architecture & Tooling (31-50)
* Topics: Functional Options, Hexagonal Architecture, Graceful Shutdown, Dependency Injection (no-reflect), Table-driven & Fuzz testing, Race Detector internals, Terraform Provider development.

---

## 🌐 Part 2: Role-Specific: Integration, IIoT, & Cloud (50 Questions)

### 1. Edge & IIoT Protocols (51-60)
* Topics: MQTT QoS 1/2 handling, CoAP over UDP (retransmissions), Managing 10k+ TCP conns (epoll), Store-and-forward for network partitions, mTLS in Go.

### 2. Streaming & Real-Time (61-70)
* Topics: High-throughput WebSockets, gRPC Streams vs SSE, Backpressure strategies, Kafka exactly-once semantics, In-memory ring buffers.

### 3. Data Transformation & Connectors (71-80)
* Topics: Zero-copy transformations, JSON optimization (`jsoniter`), Schema evolution in NoSQL, Batching inserts from Go Channels.

### 4. Cloud Integration & Reliability (81-100)
* Topics: Circuit Breakers (`sony/gobreaker`), Token Bucket Rate Limiting, AWS/Azure SDK vendor lock-in avoidance, Lambda cold-start optimization, Distributed Locking (Redis/Etcd).

---

## 📚 Study Resources

### Core Internals
* [Ardan Labs: Scheduling In Go (3 Part Series)](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
* [Go Runtime Source: proc.go](https://github.com/golang/go/blob/master/src/runtime/proc.go)
* [The Go Memory Model](https://go.dev/ref/mem)
* [Go Data Structures: Interfaces](https://research.swtch.com/interfaces)

### IIoT & Integration
* [MQTT Essentials](https://www.hivemq.com/mqtt-essentials/)
* [Go Concurrency Patterns: Pipelines](https://go.dev/blog/pipelines)
* [Practical Go: Real World Advice (Dave Cheney)](https://dave.cheney.net/practical-go)

### Performance & Debugging
* [Profiling Go Programs (pprof)](https://go.dev/blog/pprof)
* [High Performance Go Workshop](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris-2019.html)

---

## 💡 Key Talking Points for Technical Architects

1.  **Resiliency:** Always mention *Circuit Breakers* and *Retries with Jitter* when discussing Cloud/API connectors.
2.  **Backpressure:** In IIoT, explain how you prevent the "Fast Producer, Slow Consumer" problem using buffered channels and dropping non-critical packets.
3.  **Observability:** Mention exposing **Prometheus** metrics for Goroutine counts, GC latency, and message throughput.
4.  **Security:** For Edge devices, emphasize **mTLS** and **JWT** rotation for northbound data transfer.

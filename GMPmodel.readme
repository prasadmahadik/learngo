```markdown
# Go Scheduler G‑P‑M Model (Goroutine–Processor–Machine)

The G‑P‑M (often written G‑M‑P) model is how Go’s runtime scheduler runs millions of goroutines efficiently on a limited number of OS threads and CPU cores.[web:21][web:26][web:30]

---

## 1. Core Concepts

### G – Goroutine

- A **G** is a goroutine, represented internally by type `g` in the runtime.[web:30]  
- It contains:
  - Its own small, growable stack.[web:26]  
  - Program counter and register state, saved/restored by the scheduler during context switches.[web:26]  
  - Metadata like status (runnable, waiting, etc.).[web:26]  
- Goroutines are scheduled in user space by Go’s runtime, not directly by the OS.[web:21][web:26]  

### P – Processor

- A **P** is a logical processor context; the scheduler’s unit of execution rights.[web:35][web:38]  
- Properties:
  - There are exactly `GOMAXPROCS` Ps.[web:35][web:38]  
  - Each P maintains a local run queue of runnable goroutines.[web:35][web:36]  
  - An M must own a P to execute Go code.[web:30][web:35]  
- Ps enable work stealing and help balance load across OS threads.[web:35][web:36]  

### M – Machine (OS thread)

- An **M** is an OS thread, represented internally by type `m`.[web:30]  
- It can execute:
  - User Go code.  
  - Runtime code.  
  - System calls.  
- The scheduler’s job is to match up:
  - A G (code to execute),  
  - An M (where to execute),  
  - A P (rights/resources to execute).[web:30]  

---

## 2. How G, P, and M Work Together

### High‑level orchestration

- The scheduler maintains:
  - A pool of Gs (goroutines in various states).  
  - A fixed set of Ps (logical processors, `GOMAXPROCS`).  
  - A variable set of Ms (OS threads) that attach to Ps to run code.[web:21][web:30][web:38]  
- When an M stops executing user Go code (for example, enters a blocking syscall), it returns its P to the idle P pool.[web:30]  
- To resume executing Go code (for example, syscall returns), that M must acquire a P from the idle pool again.[web:30]  

Conceptual mapping: **P** is a parking lot holding goroutines (Gs) ready to run, **M** is the driver (OS thread), and each driver must own a parking lot key (P) to take cars out and drive them on a CPU core.[web:21][web:35][web:38]

---

## 3. Run Queues and Work Stealing

Go manages goroutine queues at two levels:[web:36]

- **Local run queues** (per P): primary queues where runnable Gs are stored.[web:35][web:36]  
- **Global run queue**: shared queue used when:
  - Local queue overflows.  
  - Some goroutines are injected from the network poller or GC wakeups.[web:36]  

Work‑stealing rules for an idle P:[web:36]

1. Pull work from its own local queue.  
2. Pull work from the global queue.  
3. Pull work from the network poller (ready I/O events).  
4. Steal work from another P’s local queue (take about half of it).  

This design:

- Reduces contention because most scheduling is done on local queues.[web:20][web:36]  
- Balances load automatically by letting idle Ps steal from busy Ps.[web:20][web:36]  

---

## 4. Static Mapping Diagram

Assume `GOMAXPROCS = 2` with 6 runnable goroutines:

```text
G1, G2, G3  →  [ P1 ]  →  M1  →  Core 1
G4, G5, G6  →  [ P2 ]  →  M2  →  Core 2
```

- `G1..G6` are runnable goroutines.[web:26][web:35]  
- `[P1]` and `[P2]` are processors holding local run queues and scheduler state.[web:35][web:36]  
- `M1` and `M2` are OS threads executing Go code on two CPU cores.[web:21][web:30]  

Only as many goroutines can run in parallel as there are Ps (`GOMAXPROCS`), even if total goroutines is much larger.[web:38]

---

## 5. Lifecycle Example: Creating and Running Goroutines

### Example code

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func worker(id int) {
    for i := 0; i < 3; i++ {
        fmt.Println("worker", id, "iteration", i)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    runtime.GOMAXPROCS(2) // at most 2 Ps → 2 goroutines can run in parallel
    for i := 0; i < 5; i++ {
        go worker(i)
    }
    time.Sleep(time.Second)
}
```

- Default `GOMAXPROCS` is number of logical CPUs; here we set it to 2 explicitly.[web:38]  
- Five goroutines (`worker(0..4)`) may be runnable, but at most 2 execute in parallel at any moment.[web:38]  

### Internal sequence (simplified)

```text
Start:
  G0 (runtime) on M0 + P0

Init:
  runtime creates main goroutine Gmain
  attach:  M0 ↔ P0
  enqueue Gmain into P0.runq

Run main:
  M0 pops Gmain from P0.runq and runs main()

Create goroutines:
  Gmain executes loop:
    go worker(0) → create G1, enqueue to P0.runq
    go worker(1) → create G2, enqueue to P0.runq
    ...
    go worker(4) → create G5, enqueue to P0.runq (or global if overflow)

Scheduling:
  When Gmain finishes or blocks:
    M0 picks next goroutine from P0.runq, e.g. G1
  Another M (M1) may be created and attached to P1 so two goroutines
  run in parallel on two cores.
```

The run loop of each M:

1. Take a runnable G from its P’s local run queue.  
2. Execute G until it blocks, yields, or is preempted.  
3. Repeat; if no local G, try global queue or work stealing.[web:20][web:26][web:36]  

---

## 6. Blocking Syscalls and P Handoff

When a goroutine performs a blocking syscall (for example, slow I/O):

- The M executing that goroutine enters the kernel and may block.[web:26][web:30]  
- To avoid stalling other goroutines, the runtime:
  - Detaches the P from that blocked M and returns it to the idle P pool.  
  - Attaches that P to another available M so other goroutines in the run queue can continue executing.[web:26][web:30]  

Text “diagram”:

```text
Before syscall:
  G1, G2, G3 → [ P1 ] → M1 → Core 1

G1 calls blocking syscall:
  M1 blocks in kernel with G1
  runtime detaches P1 from M1
  M1 (blocked) now has no P

Handoff:
  P1 is attached to another M, say M2
  M2 ↔ P1 now runs on Core 1
  M2 starts executing G2, G3 from P1’s run queue

After syscall returns:
  M1 becomes runnable again
  it tries to acquire an idle P to resume executing Go code
```

This mechanism:

- Ensures that long‑running or blocking syscalls do not freeze all goroutines.[web:26][web:30]  
- Allows many goroutines that block on I/O to coexist without creating one OS thread per goroutine.[web:21][web:26]  

---

## 7. Preemption

Go has a work‑stealing, mostly cooperative scheduler but with preemption:

- Since Go 1.14, goroutines can be preempted at safe points (asynchronous preemption), improving fairness.[web:20][web:37]  
- The scheduler uses tick counters to detect long‑running goroutines and injects preemption requests so other goroutines get CPU time.[web:20][web:37]  

Preemption details are visible in internal code and issues (for example, handling Ps in `_Psyscall` vs `_Prunning` states for preemption logic).[web:37]

---

## 8. GOMAXPROCS in Practice

`runtime.GOMAXPROCS(N)`:

- Controls the number of Ps, i.e. the maximum number of goroutines that can run *in parallel*.[web:35][web:38]  
- Default is number of logical CPUs, which is usually a good choice.[web:36][web:38]  

Example:

```go
fmt.Println(runtime.GOMAXPROCS(0)) // read current value (no change)
```

- If `GOMAXPROCS` is 4, you can have millions of goroutines, but only 4 run at the same time; others wait in queues.[web:36][web:38]  

---

## 9. Why the G‑P‑M Model Matters

Compared to a 1:1 goroutine‑to‑OS‑thread model:

- **Lower memory**: goroutines have small, growable stacks instead of large fixed thread stacks.[web:26][web:36]  
- **Faster scheduling**: user‑space scheduling (run queues + work stealing) is cheaper than kernel thread scheduling for massive concurrency.[web:20][web:26]  
- **Better blocking behavior**: blocking syscalls detach Ps and reassign them, so other goroutines keep progressing.[web:26][web:30]  

---

## 10. How to Use This Mental Model When Coding

You do not directly manipulate G, P, or M, but:

- Use `go` to create goroutines; think of them as tasks going into Ps’ run queues.[web:35][web:36]  
- Tune `GOMAXPROCS` if you have special CPU‑bound workloads or want to limit Go’s CPU usage.[web:35][web:38]  
- Prefer non‑blocking I/O and channels; use `context.Context` for cancellation and timeouts, avoiding goroutine leaks and unnecessary blocking.[web:20][web:36]  

Mental summary:

> Each core you give Go via `GOMAXPROCS` becomes a P with its own run queue. Goroutines are tasks in these queues. The runtime dynamically maps these tasks (Gs) onto OS threads (Ms) attached to Ps, handling blocking, preemption, and load balancing behind the scenes.[web:21][web:30][web:36][web:38]
```

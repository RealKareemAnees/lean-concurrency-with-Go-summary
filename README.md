# Concurrent Programming with Go — Chapters 1–3 Refresher

## Overview

These three chapters lay the conceptual and mechanical foundation for concurrent programming in Go. Chapter 1 motivates *why* concurrency matters (throughput, responsiveness, hardware trends) and introduces Go's core tools (goroutines, channels, CSP) plus the theoretical limits of parallel speedup (Amdahl's and Gustafson's laws). Chapter 2 drops down to the OS level — processes, threads, scheduling, context switches — and shows how Go's goroutines relate to (but differ from) OS threads via the M:N hybrid scheduling model. Chapter 3 introduces the first concrete inter-thread communication mechanism, memory sharing, and uses it to demonstrate the central danger of concurrent code: race conditions, along with the tooling (`-race`) used to catch them.

## Core Concepts

### Concurrency vs. Parallelism (Chapter 1 & 2)

**Definition:**
- **Concurrency** is an attribute of *program code* — structuring a program into tasks with defined boundaries and synchronization points that *could* execute independently.
- **Parallelism** is a property of the *executing program* — tasks actually running at the same physical instant, which requires multiple processing units.
- Formally: *"Concurrency is about planning how to do many tasks at the same time. Parallelism is about performing many tasks at the same time."* Parallelism is a subset of concurrency — only concurrent programs can run in parallel, but not all concurrent programs actually do (e.g., on a single core).
- **Pseudo-parallel execution**: a single-processor system frequently context-switching jobs (on a timer or blocking I/O) to give the illusion of simultaneous execution. Some textbooks debate whether I/O-in-progress alongside CPU computation counts as "true" parallelism; the convention in the text is to reserve "parallel execution" for computation, not I/O.

**How it works internally:** Whether concurrent code parallelizes depends entirely on hardware/environment (number of cores) — the code itself doesn't change.

**Example tasks that are concurrency candidates:**
- Handling one user's request
- Searching one file for text
- Calculating one row of a matrix multiplication
- Rendering one video frame

---

### Horizontal vs. Vertical Scaling (Chapter 1)

**Definition:**
- **Horizontal scaling**: improving performance by distributing load across multiple processing resources (more processors/servers).
- **Vertical scaling**: improving performance by upgrading existing resources (a faster single processor).

**Context:** Horizontal scaling only yields benefits if the code is written to exploit multiple processing units — otherwise adding cores/servers does nothing (this is literally the payroll problem in the chapter's opening story).

---

### I/O Wait and Responsiveness (Chapter 1)

**Definition:** Most programs spend a small proportion of time actually computing; the rest is spent waiting on I/O (disk, network, user input). Concurrent programming lets a program do useful work during these idle waits (e.g., spell-checking while the user is idle, or reading the next file while processing the current one in memory).

**Mechanism:** Even on a *single* CPU, an OS can pause one task, respond to input or perform another task, then resume — using interleaving via hardware interrupts and OS traps. This creates the *impression* of simultaneity via fast task-switching, not true parallel execution.

**Example (conceptual, word processor):** Keyboard-listener task, spellcheck task, document-stats task, and autosave task all appear to run "simultaneously" on one CPU by being fast-switched by the OS.

---

### Go's Concurrency Toolkit — Goroutines & Channels (Chapter 1)

**Definition:**
- **Goroutine**: Go's lightweight unit of concurrent execution, modeled as a user-level thread run on top of OS-level ("kernel-level") threads managed by Go's runtime.
- **Channel**: an abstraction allowing two or more goroutines to pass messages, enabling synchronization and information exchange.

**Philosophy:** "If you need something done concurrently, create a goroutine to do it... without worrying about resource allocation." Go's runtime and hardware handle parallelism; the programmer focuses on writing *correct* concurrent code.

**Syntax:**
```go
go someFunction(args) // launches someFunction concurrently in a new goroutine
```

**Example (fan-out/fan-in mentioned conceptually in the payroll story):** Divide payroll into employee groups, call the payroll module concurrently across goroutines, load-balance via a channel, then collect results via another channel/goroutine. (Full channel syntax deferred to a later chapter — this chapter only introduces the concept.)

---

### Communicating Sequential Processes (CSP) (Chapter 1)

**Definition:** A formal model, first described by C.A.R. Hoare in 1978, for expressing concurrent interactions where isolated processes communicate and synchronize by passing messages (as opposed to sharing memory). Influenced languages like Occam and Erlang; Go implements many CSP ideas, notably synchronized channels.

**How it helps:** Modeling concurrency as isolated goroutines that only interact via channels reduces the risk of race conditions, because there's no shared, uncontrolled mutable state to corrupt. This mirrors real-world concurrency (people/processes/machines communicating via messages).

**Important nuance:** CSP-style channel programming is *not always superior*. Classic memory-sharing primitives (mutexes, condition variables) can sometimes perform better depending on the problem. Go supports both. The book's approach: start with memory sharing/classic primitives first (Chapters 2–6ish) before covering CSP-style channels (Chapter 7+), so the reader has a solid grounding in traditional synchronization first.

---

### Building Concurrency Tools From Scratch (Chapter 1 — meta note)

**Definition:** The book's pedagogical approach is to implement common concurrency primitives (mutex, condition variables, semaphores, waitgroups, and even channels) from more basic building blocks, even where Go already provides them (e.g., Go has no built-in semaphore — the book builds one). Rationale: understanding internals (like knowing how a sort algorithm works even if you use a built-in sort function) leads to better-informed usage decisions.

---

### Amdahl's Law (Chapter 1)

**Definition:** Formulated by Gene Amdahl in 1967. States that the overall performance improvement from optimizing (parallelizing) part of a system is limited by the fraction of time that part is actually used — i.e., the *non-parallelizable (serial) portion* of a task caps the maximum possible speedup, regardless of how many processors you add.

**How it works:** For a fixed-size problem with a fixed serial fraction:
- At 95% parallel portion: ~19x speedup at 512 processors; hits a hard ceiling around 4096 processors for ~20x speedup — adding more workers beyond this wastes money.
- At 90% parallel: scalability limit reached around 512 workers; max speedup ≈ 10x.
- At 75% parallel: limit near 128 workers; max speedup ≈ 4x.
- At 50% parallel: limit near 16 workers; max speedup ≈ 2x.

**Analogy used:** Building houses with more builders — 1 builder: 8 months; 2 builders: 4 months; 4 builders: 2.5 months (and costs more overall); 8 and 16 builders: both plateau at 2 months. This plateau is the *scalability limit* — some parts (e.g., only one truck can use the single access road at a time) are inherently sequential.

**Takeaway:** Amdahl's law paints a "bleak picture" for parallel computing when even a small serial fraction exists — the more processors you add, the less relative benefit you get, and eventually zero benefit.

---

### Gustafson's Law (Chapter 1)

**Definition:** Published in 1988 by John L. Gustafson and Edwin H. Barsis as a re-evaluation of Amdahl's Law's assumptions. Core argument: in practice, **problem size grows** as more resources become available, rather than staying fixed. Extra resources can be devoted to increasing the *scope or quality* of work instead of just speeding up a fixed task.

**Two key arguments against Amdahl's pessimism:**
1. If you have surplus resources beyond what's needed, allocate them to other work rather than wastefully parallelizing further (e.g., don't put 1000 builders on a small house — send the extra builders to other projects).
2. As problem size increases, the *non-parallel portion typically does not grow proportionally* — it often stays roughly constant. Given this, speedup can scale **linearly** with available parallel resources.

**Example used:** A video game utilizing more cores over time. Eventually frame rate stops improving (Amdahl's limit reached for that fixed problem), but developers redirect the extra processing power toward richer graphics/higher resolution instead (Gustafson's law in action — the "problem" size/scope grew).

**Relationship between the two laws:** They aren't contradictory — Amdahl's law describes speedup limits for a *fixed-size* problem; Gustafson's law describes what happens when the problem size is allowed to scale with available resources.

---

### OS Job Lifecycle / Multiprocessing (Chapter 2)

**Definition:** Multiprocessing (multiprogramming) is when an OS handles more than one task at a time, keeping the CPU busy by switching to other jobs when the current one is idle (e.g., waiting on I/O).

**How it works internally — job states (using `grep 'hello' largeReadme.md` as example):**
1. User submits job.
2. OS places job in the **job queue** if not yet ready to run.
3. Job moves to the **ready queue** once ready.
4. CPU picks the job from the ready queue and executes it.
5. When job requests I/O, OS removes it from CPU and places it in the **I/O waiting queue**; another ready job takes the CPU.
6. Device completes the I/O operation.
7. Job moves back to the ready queue (waiting its turn again, since CPU might be busy).
8. CPU is free again; job resumes and continues (e.g., searching loaded text for a match).
9. An **interrupt** may be raised mid-execution — caused by: an I/O device finishing an operation, a program requesting a software interrupt, or a hardware clock/timer tick.
10. OS pauses the job, returns it to the ready queue, and schedules another job (this + step 9 together = a **context switch**).
11. Job eventually resumes again (steps 4–10 repeat as needed).
12. Job finishes and terminates.

**Definition — Context switch:** Occurs whenever the system interrupts a job and the OS steps in to schedule another. Involves overhead: saving the current job's **Process Context Block (PCB)** — program counter, CPU registers, memory info — and loading the next job's PCB.

**Note (Linux-specific):** The "ready queue" is called the **run queue** on some systems like Linux.

**Historical/side context:**
- SAGE (1950s, US military airspace monitoring system) pioneered real-time processing, distributed computing, and multiprocessing ideas, using remote computers over telephone lines.
- IBM System/360 (1960s) is often cited as the first "real" OS with true multiprocessing support, though earlier systems (batch-processing systems) existed under different names.
- Before multiprocessing, a job requiring tape I/O would halt *all* processing — hugely inefficient. Solution: load multiple jobs, give each a memory chunk, switch to another job during I/O waits.
- **Time-sharing**: Since programming is mostly a "thinking" process, only a small fraction of connected terminal users are compiling/executing at any moment — so CPU time can be shared among many interactive users, cutting the long feedback loops of the mainframe batch era.

---

### Processes (Chapter 2)

**Definition:** An abstraction representing a program currently running on the system, with its own isolated memory space and system resources.

**How it works internally:**
- Analogy: a team of painters, each with their own paper — draw separately, then merge (stick pieces onto canvas) at the end.
- Each process's isolation means a crash in one process does not affect others.
- **Downside:** isolation costs more memory (each process needs its own memory space) and startup is slower (allocating memory space + resources for a new process takes longer than a thread).
- Because processes don't share memory, they must communicate via OS tools/mechanisms: files, databases, pipes, sockets, etc. — merging results across processes is more of a challenge than with threads.

**Creating processes — Syntax/APIs:**
- Windows: `CreateProcess()` — creates the process, allocates resources, loads code, starts execution.
- UNIX: `fork()` — creates a **child** process as a complete copy of the calling (**parent**) process's memory, registers, stack, file handlers, and program counter.
  - Returns the child's process ID to the *parent*.
  - Returns `0` to the *child*.
  - Each process can branch behavior based on that return value.
  - Child can either keep the copied memory or clear/restart it.
  - Since each process has separate memory, one process changing a variable is invisible to the other.

```go
// Go's syscall package (OS-specific) exposes process primitives:
// Windows: syscall.CreateProcess()
// UNIX:    syscall.ForkExec(), syscall.StartProcess()
// Cross-platform higher-level: os/exec package's exec() abstraction
```

**Note:** Go's concurrency model does *not* typically rely on heavyweight OS processes — it favors lightweight threading/goroutines instead.

**Copy-on-Write (COW) optimization (side note):** Instead of copying the entire memory space on `fork()`, parent and child initially **share** memory pages. Only when either process *modifies* a page does the OS copy that specific page to a new location. Saves memory/time, but if a process modifies most of its memory, the OS ends up copying most pages anyway.

**Termination behavior:** When a process finishes or hits an unhandled error, the OS reclaims all its resources (memory, file handles, network connections). On UNIX/Windows, a parent finishing does **not** automatically terminate its child processes.

**Real example — multiprocessing for common tasks:**
```bash
curl -s https://www.rfc-editor.org/rfc/rfc1122.txt | wc
```
This forks two concurrent processes (`curl` and `wc`) connected via a pipe buffer. `curl` writes into the buffer; `wc` reads from it. `curl` blocks if the buffer fills; `wc` blocks if the buffer is empty. When `curl` finishes, it places an end-of-data marker in the pipe so `wc` knows to terminate too. Verified via `ps -a` showing both PIDs.

---

### Threads (Chapter 2)

**Definition:** A lightweight execution context ("microprocess") within a process. Threads share the process's memory space and resources but each has its **own private stack, registers, and program counter**.

**How it works internally:**
- Analogy: instead of separate papers (processes), the whole team paints directly on one shared canvas (shared memory).
- A process starts with one **main thread** by default; a process with more than one thread is **multithreaded**.
- Creating a thread requires the OS to allocate only stack space, registers, and a program counter — *not* a whole new memory space — making thread creation "sometimes 100 times faster" than process creation and far less resource-intensive.
- Because memory is shared, any change to a shared/global variable by one thread is immediately visible to other threads in the same process — this enables efficient collaboration but **also introduces the risk of threads stepping on each other's work**, requiring communication/synchronization.

**Stack vs. Heap (side note, expanded in Chapter 3):** Local (non-shared) variables inside a function live on that thread's private **stack**. Space shared between functions/threads via pointers lives on the **heap** (shared main memory).

**Thread states:** Mirror the job states discussed for processes (ready, running, waiting for I/O, etc.), but at a per-thread granularity.

**Termination semantics — important cross-language difference:**
- **Go:** When the **main** goroutine/thread terminates, the *entire process* terminates immediately — even if other goroutines/threads are still running.
- **Java:** By contrast, the process terminates only once *all* threads have finished.

**OS-specific thread creation APIs:**
- Windows: `CreateThread()`
- Linux: `clone()` with the `CLONE_THREAD` option
- **POSIX Threads (pthreads):** An IEEE standard API for creating/managing/synchronizing threads, implemented by various OSes (Windows, UNIX), though not universally supported by all languages.

**Language-specific thread models (side note):**
- Java: models threads as objects.
- Python: has a Global Interpreter Lock (GIL) that blocks multiple threads from executing Python bytecode in true parallel.
- Go: uses the finer-grained goroutine concept (see below).

**Multithreaded server example (sports score app):**
- A **stream reader thread** reads a live sports feed and writes into a shared `sports scores` data structure.
- Multiple **client handler threads** (in a pool) read from that shared structure to answer user requests concurrently.
- **Benefits cited:**
  1. Resource efficiency — many lightweight threads can be spun up/down cheaply, dynamically sized to traffic.
  2. Shared memory access — easy to have all threads read/write one shared data structure directly.

**Hybrid process+thread architecture example — modern browsers:**
- Threads download/render page resources concurrently, sharing memory to assemble the final rendered page.
- Heavy computation (e.g., a script) can be split across more threads on a multicore CPU.
- **Processes** are used for isolation — e.g., one process per tab/window — so that a crashing script in one tab doesn't kill other tabs (e.g., doesn't destroy an unsaved draft email in another tab).
- Real browsers cap the number of processes and start sharing processes across tabs beyond that cap, to control memory usage.

---

### Goroutines — Creation & Basic Semantics (Chapter 2)

**Definition:** Go's own concurrency construct — neither an OS thread nor a process. Managed by Go's runtime, which multiplexes many goroutines onto a smaller set of OS ("kernel-level") threads.

**Syntax:**
```go
go functionName(args...)  // launches functionName asynchronously (non-blocking call)
```
- Calling a function normally (without `go`) is a **synchronous** call — caller waits for it to return.
- Calling with `go` is **asynchronous** — caller does not wait; it proceeds to the next instruction immediately.

**Example — sequential version:**
```go
package main

import (
    "fmt"
    "time"
)

func doWork(id int) {
    fmt.Printf("Work %d started at %s\n", id, time.Now().Format("15:04:05"))
    time.Sleep(1 * time.Second) // simulates work
    fmt.Printf("Work %d finished at %s\n", id, time.Now().Format("15:04:05"))
}

func main() {
    for i := 0; i < 5; i++ {
        doWork(i)
    }
}
// Total runtime: ~5 seconds (one doWork call after another)
```

**Example — concurrent version using goroutines:**
```go
package main

import (
    "fmt"
    "time"
)

func doWork(id int) {
    fmt.Printf("Work %d started at %s\n", id, time.Now().Format("15:04:05"))
    time.Sleep(1 * time.Second)
    fmt.Printf("Work %d finished at %s\n", id, time.Now().Format("15:04:05"))
}

func main() {
    for i := 0; i < 5; i++ {
        go doWork(i) // launches concurrently; main() does not wait
    }
    time.Sleep(2 * time.Second) // needed so goroutines get a chance to run
}
// Total runtime: ~2 seconds — but output ORDER is non-deterministic across runs
```

**Key observations from the example:**
- Without the trailing `time.Sleep`, the program would print **nothing**, because `main()` (and therefore the whole process) terminates before any goroutine executes.
- Output order of concurrently-launched goroutines is **not guaranteed** and will vary between runs — never write code that depends on a particular scheduling order.

---

### User-Level Threads vs. Kernel-Level Threads (Chapter 2)

**Definition:**
- **Kernel-level threads**: threads the OS itself knows about, manages, schedules, and context-switches. The OS stores each thread's context (registers, stack, state).
- **User-level threads**: threads implemented entirely within the application's own memory space (user space), invisible to the OS, which sees only a single kernel-level thread. The application/runtime itself must implement its own internal scheduler, context-switching logic, and a table tracking each user-level thread's state.

**Tradeoffs:**
- **Advantage of user-level threads:** faster context switches (no kernel intervention needed → no cache flush, no OS scheduler invocation).
- **Disadvantage 1:** If a user-level thread performs a *blocking* I/O call, the OS sees the whole process as blocked, descheduling **all** user-level threads within it — defeating one of the main purposes of using multiple threads. Workaround: use non-blocking I/O calls, but not all devices support this.
- **Disadvantage 2:** On a multicore/multiprocessor machine, a group of user-level threads bundled under a single kernel-level thread can only ever use **one processor at a time** — no true parallelism among them.

**Historical/naming side note — Green Threads:** Coined in Java 1.1 as an implementation of user-level threads (single-core only, JVM-managed). Abandoned in Java 1.3 in favor of kernel-level threads. The term is now loosely used by many developers for any user-level thread implementation. Calling Go's goroutines "green threads" is described as **somewhat inaccurate**, since Go's runtime lets goroutines fully use multiple CPUs. A later, different Java feature (introduced in later Java versions) reused similar ideas under the name **virtual threads**.

---

### Go's M:N Hybrid Threading Model (Chapter 2)

**Definition:** Go's runtime uses a hybrid system: **M** goroutines (user-level threads) are mapped onto **N** kernel-level threads. This contrasts with:
- **N:1 model** = classic user-level threading (many user threads : one kernel thread) — limited to one processor.
- Go's **M:N model** allows true multi-core parallelism because there are multiple kernel-level threads, each of which can run on a separate physical core.

**Note:** Implementing an M:N runtime is significantly more complex than either extreme, requiring load-balancing logic to move goroutines across kernel threads.

**GOMAXPROCS — Syntax:**
```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    fmt.Println("Number of CPUs:", runtime.NumCPU())
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
}
// On an 8-core machine, prints:
// Number of CPUs: 8
// GOMAXPROCS: 8
```
- `runtime.GOMAXPROCS(0)` — passing `n < 1` just **returns** the current value without changing it.
- If unset, Go queries the OS for the number of CPUs and uses that as the default `GOMAXPROCS` value — i.e., the number of kernel-level threads Go will use to run goroutines in parallel.

**How it works internally — Run queues & work stealing:**
- Each kernel-level thread gets a **Local Run Queue (LRQ)** holding a subset of the program's goroutines.
- There's also a **Global Run Queue (GRQ)** for goroutines not yet assigned to any kernel-level thread's LRQ.
- **Work stealing:** If a kernel-level thread's goroutine makes a blocking call, Go creates a new kernel-level thread (or reuses an idle one from a pool) and moves that thread's queue of goroutines to the new one, so the blocked-on-I/O thread doesn't stall the whole queue. Work stealing *also* happens for load-balancing purposes — an idle LRQ (empty queue) will steal goroutines from another thread's LRQ so no processor sits idle while others are overloaded.

**Locking a goroutine to its OS thread — Syntax:**
```go
runtime.LockOSThread()   // binds current goroutine exclusively to its current OS thread
// ... goroutine code that must stay pinned, e.g. calling into a C library ...
runtime.UnlockOSThread() // releases the binding
```
Use case: interfacing with external C libraries where the goroutine must not migrate to a different kernel-level thread mid-execution.

---

### The Go Scheduler & Preemption (Chapter 2)

**Definition:** The Go scheduler runs in **user space** and is responsible for context-switching goroutines onto kernel-level threads. Unlike the OS scheduler (which is **preemptive**, using hardware clock interrupts to forcibly reclaim the CPU from any thread), Go's scheduler needs **user-level events** to trigger it, since it has no access to hardware interrupts directly.

**Triggering events for the Go scheduler:**
- Starting a new goroutine (`go` keyword)
- Making a system call (e.g., file read)
- Synchronizing goroutines (e.g., via channels/locks — covered in later chapters)
- Explicitly calling `runtime.Gosched()`

**Syntax — explicit yield:**
```go
runtime.Gosched() // yields the processor, allowing another goroutine a chance to run;
                   // does NOT suspend the calling goroutine — it resumes automatically afterward
```

**Example:**
```go
package main

import (
    "fmt"
    "runtime"
)

func sayHello() {
    fmt.Println("Hello")
}

func main() {
    go sayHello()
    runtime.Gosched()
    fmt.Println("Finished")
}
```
**Important caveat:** There is **no guarantee** `sayHello()` will ever actually run before `main()` finishes and the process exits. `runtime.Gosched()` only *increases the odds* the other goroutine executes — it does not force or order execution. Running this repeatedly will sometimes print only `"Finished"`.

**Warning (explicit in text):** Never write code that depends on the Go scheduler's or OS scheduler's specific execution order — it is inherently non-deterministic and can vary between runs, machine setups, and Go versions. Use proper synchronization mechanisms (covered starting Chapter 4) if execution order matters.

---

### Sharing Memory: Bus Architecture & Cache Coherency (Chapter 3)

**Definition:** Memory sharing is a form of Inter-Thread Communication (ITC) / Inter-Process Communication (IPC, for processes) where goroutines communicate by reading/writing a common region of process memory — as opposed to **message passing** (covered later via channels, Chapter 7+).

**How it works internally — architecture:**
- On a **single-processor** system, all kernel-level threads can share the same memory access straightforwardly, context-switching between them.
- On **multi-processor/multicore** systems, a **system bus** connects CPUs to main memory. A processor must wait for the bus to be idle before making a memory request.
- As core count grows, the bus becomes a bottleneck. **Caches** are added between CPU and main memory to reduce bus traffic and improve access speed — each CPU has its own local cache.
- **Cache-coherency problem:** If Thread 1 updates a cached value, and Thread 2 has an outdated cached copy of the same memory, Thread 2's cache becomes **stale**.
- **Solutions:**
  - **Write-through**: when a cache updates its value, it immediately mirrors the update back to main memory. But this alone doesn't fix *other* CPUs' stale caches.
  - **Cache listening / invalidation**: caches listen to bus update messages; upon detecting an update to a memory block they've cached, they either apply the update or **invalidate** their cache line, forcing a fresh fetch from memory next access.
  - Collectively these mechanisms are called **cache-coherency protocols** (e.g., "write-back with invalidation"). Real hardware typically mixes multiple such protocols.

**Side note — Coherency wall:** As core counts scale up, maintaining cache coherence becomes exponentially more complex and costly, and chip engineers worry this will eventually become the limiting factor on how many cores can be usefully added — this limit is called the **coherency wall**.

---

### Sharing Variables Between Goroutines (Chapter 3)

**Definition & mechanism:** Goroutines can share a variable by passing a **pointer** to it. Example: a countdown timer shared between the `main()` goroutine (reads frequently) and a `countdown()` goroutine (writes/decrements periodically).

**Full example:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    count := 5
    go countdown(&count) // shares the memory location of count

    for count > 0 {
        time.Sleep(500 * time.Millisecond) // reads more frequently than updates happen
        fmt.Println(count)
    }
}

func countdown(seconds *int) {
    for *seconds > 0 {
        time.Sleep(1 * time.Second)
        *seconds -= 1
    }
}
// Sample output (values repeat since reads happen more often than writes):
// 5 4 4 3 3 2 2 1 1 0
```

**Note:** If you remove the `go` keyword, this becomes purely sequential: `count` lives on `main`'s stack, `countdown()` blocks for the full 5 seconds updating it directly, and by the time control returns to `main()`, `count` is already `0` — so the loop body never executes at all.

---

### Escape Analysis (Chapter 3)

**Definition:** The Go compiler's decision process for whether a variable is allocated on the **stack** (fast, automatically reclaimed when the function returns) or the **heap** (garbage-collected, slight extra cost). When a variable that "looks" local ends up allocated on the heap because it's referenced outside the function's own stack frame (e.g., shared with another goroutine via a pointer), it is said to have **escaped to the heap**.

**Why it matters for goroutines:** A goroutine cannot safely use another goroutine's stack space — stack frames can be torn down at different times per goroutine's independent lifecycle. So the Go compiler detects cross-goroutine sharing and forces such variables onto the heap instead.

**Checking escape decisions — Syntax:**
```bash
go tool compile -m countdown.go
```
Sample output (concurrent version, from the text):
```
countdown.go:7:6: can inline countdown
countdown.go:7:16: seconds does not escape
countdown.go:15:5: moved to heap: count
```
Sample output (sequential version — `go` keyword removed):
```
countdown.go:7:6: can inline countdown
countdown.go:16:14: inlining call to countdown
countdown.go:7:16: seconds does not escape
```
Notice `count` no longer says "moved to heap" — it stays on the stack — **and** the compiler now performs **inlining** (replacing the function call with its body inline to avoid call overhead), an optimization that isn't applied to the concurrent/goroutine version because the function runs on a separate stack, possibly on another kernel thread.

**Cost tradeoff:** Heap-allocated memory requires garbage collection cleanup (marking unreferenced objects and reclaiming space), which is more expensive than stack memory, which is automatically reclaimed the instant a function returns. Using goroutines/shared memory means giving up some compiler optimizations (like inlining) and incurring extra heap/GC overhead — the tradeoff is potential concurrency-driven speedup.

---

### Race Conditions (Chapter 3)

**Definition:** A **race condition** occurs when multiple threads/goroutines share a resource and their timing-dependent, uncoordinated interleaving of operations produces incorrect or unpredictable results. Behavior depends on the exact, non-deterministic timing of independent events.

**Definition — Critical section:** A set of instructions that must execute without interference from other concurrent executions touching the same shared state. Race conditions occur specifically when this "no interference" guarantee is violated.

**Definition — Atomic operation:** An operation that cannot be interrupted partway through ("atomic" from Greek for "indivisible"). Compound operations like `*money += 10` are **not** atomic — they compile down to multiple machine instructions (read, modify, write), and another goroutine's instructions can interleave in between.

**Real-life analogy given:** A couple sharing a paper grocery list on the fridge; both independently decide (without telling each other) to do the shopping after work — resulting in duplicate purchases, because there was no synchronization/communication about who was already handling it.

**Real-world consequence example (cautionary story):** A fictional investment bank ("Turner Belfort") suffers a system-wide outage and corrupted client account balances due to a race condition that had existed silently in production code for a long time before being triggered.

**Full demonstration — Stingy and Spendy:**
```go
package main

import (
    "fmt"
    "time"
)

func stingy(money *int) {
    for i := 0; i < 1000000; i++ {
        *money += 10
    }
    fmt.Println("Stingy Done")
}

func spendy(money *int) {
    for i := 0; i < 1000000; i++ {
        *money -= 10
    }
    fmt.Println("Spendy Done")
}

func main() {
    money := 100
    go stingy(&money)
    go spendy(&money)
    time.Sleep(2 * time.Second)
    println("Money in bank account: ", money)
}
```
- **Expected** result: `100` (equal earning and spending should cancel out).
- **Actual** results vary wildly between runs — e.g., `4203750` or `-1127120` — because `*money += 10` and `*money -= 10` are not atomic; a goroutine can read a stale value, then another goroutine also reads the same stale value, and one goroutine's write overwrites the other's, silently dropping updates.

**Walkthrough (single-processor scenario) of exactly how updates get lost:**
1. Spendy reads 100, subtracts 10, writes 90.
2. Stingy reads 90, adds 10, writes 100.
3. Spendy reads 100 (correctly).
4. **Context switch interrupts Spendy mid-operation** before it writes back.
5. Stingy reads 100 (same stale base value Spendy already read).
6. Both Stingy and Spendy do their arithmetic based on the *same* starting value of 100.
7. Spendy writes back 90.
8. Stingy then overwrites with 110 — Spendy's subtraction is effectively lost. Net: spent $20, earned $20, but ended with $10 extra due to the lost update.

**Definition — Heisenbug:** A bug (like a race condition) that changes behavior or disappears when you try to observe/debug it — e.g., adding breakpoints slows execution, making the race less likely to reproduce. Named after Heisenberg's uncertainty principle. Because Heisenbugs are so hard to debug directly, the best strategy is *prevention* — understanding causes and applying proper synchronization from the start, rather than trying to catch them after the fact.

---

### Letter Frequency Counter — Sequential vs. Concurrent (Chapter 3)

**Purpose:** A larger, realistic demonstration of how memory sharing across many goroutines can introduce subtle race conditions (as opposed to the toy Stingy/Spendy example).

**Sequential version (conceptual reconstruction based on the text):**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "strings"
)

const allLetters = "abcdefghijklmnopqrstuvwxyz"

func countLetters(url string, frequency []int) {
    resp, _ := http.Get(url)
    defer resp.Body.Close()
    if resp.StatusCode != 200 {
        panic("Server returning error status code: " + resp.Status)
    }
    body, _ := io.ReadAll(resp.Body)
    for _, b := range body {
        c := strings.ToLower(string(b))
        cIndex := strings.Index(allLetters, c)
        if cIndex >= 0 {
            frequency[cIndex] += 1
        }
    }
    fmt.Println("Completed:", url)
}

func main() {
    var frequency = make([]int, 26)
    for i := 1000; i <= 1030; i++ {
        url := fmt.Sprintf("https://rfc-editor.org/rfc/rfc%d.txt", i)
        countLetters(url, frequency) // sequential — one at a time
    }
    for i, c := range allLetters {
        fmt.Printf("%c-%d ", c, frequency[i])
    }
}
// Deterministic: same counts every run (e.g. e-181360), but takes ~17s
```

**Concurrent version (introduces the bug deliberately, per the text's explicit warning):**
```go
func main() {
    var frequency = make([]int, 26)
    for i := 1000; i <= 1030; i++ {
        url := fmt.Sprintf("https://rfc-editor.org/rfc/rfc%d.txt", i)
        go countLetters(url, frequency) // WARNING: race condition — for demonstration only
    }
    time.Sleep(10 * time.Second) // crude wait strategy, improved in later chapters
    for i, c := range allLetters {
        fmt.Printf("%c-%d ", c, frequency[i])
    }
}
// Faster (~11s) but results are non-deterministic and UNDER-count
// e.g. e-179936 instead of e-181360 — lost updates from concurrent frequency[cIndex] += 1
```
**Note explicitly stated by the text:** `time.Sleep()` is *not a great way* to wait for goroutines to finish — later chapters cover condition variables, semaphores (Ch. 5), and waitgroups (Ch. 6) as proper solutions.

---

### Yielding Does NOT Fix Race Conditions (Chapter 3)

**Definition/point:** Adding `runtime.Gosched()` calls inside the critical section loop reduces the *frequency* of race conditions (by shrinking the window of time spent in the critical section relative to scheduling overhead, and by limiting certain compiler optimizations) — but does **not eliminate** them.

**Example:**
```go
func stingy(money *int) {
    for i := 0; i < 1000000; i++ {
        *money += 10
        runtime.Gosched()
    }
    fmt.Println("Stingy Done")
}

func spendy(money *int) {
    for i := 0; i < 1000000; i++ {
        *money -= 10
        runtime.Gosched()
    }
    fmt.Println("Spendy Done")
}
```
Results still vary between runs (e.g., `170` one run, `-190` the next) — the race condition still occurs, just less often.

**Explicit warning from the text:** *"Never rely on telling the runtime when to yield the processor to solve race conditions."* There's no guarantee another parallel thread won't interfere, and even a single-processor system with multiple kernel-level threads (via `runtime.GOMAXPROCS(n)`) could still have the OS interrupt execution at any time.

---

### The Go Race Detector (Chapter 3)

**Definition:** A built-in Go tool for detecting race conditions, enabled via the `-race` compiler/runtime flag. It instruments all memory accesses to track cross-goroutine reads/writes and reports a warning when it detects unsynchronized concurrent access to the same memory.

**Syntax:**
```bash
go run -race stingyspendy.go
```

**Sample output structure:**
```
==================
WARNING: DATA RACE
Read at 0x00c00001a0f8 by goroutine 7:
  main.spendy()
      /home/james/go/stingyspendy.go:21 +0x3b
  ...
Previous write at 0x00c00001a0f8 by goroutine 6:
  main.stingy()
      /home/james/go/stingyspendy.go:14 +0x4d
  ...
==================
Found 1 data race(s)
exit status 66
```

**Important caveats (explicit warnings):**
- The race detector only catches a race condition **if it is actually triggered** during that particular run — it is not exhaustive/infallible. Test with production-like workloads/scenarios for best coverage.
- Enabling `-race` in **production** is generally discouraged because it significantly slows performance and increases memory usage. It's a development/testing tool.

---

## Patterns & Idioms

### Fan-out / Fan-in (Chapter 1, conceptually introduced)
**When to use:** Splitting a large workload (e.g., payroll for many employees) into independent chunks processed concurrently by multiple goroutines ("fan-out"), then collecting/aggregating results back together through a channel and a collector goroutine ("fan-in").
**Why:** Maximizes use of multiple CPU cores for large parallelizable workloads; matches Jane's payroll solution in the opening story (5x speedup on multicore hardware).
**Note:** The book only *names* this pattern conceptually here (with an illustrative diagram of readers → channel → goroutines → channel → aggregator); full channel syntax for implementing it is deferred to later chapters (message passing, Ch. 7+).

### Async "fire-and-wait" via `time.Sleep` (Chapter 2 & 3 — anti-pattern flagged as temporary)
**When used in the text:** As a placeholder technique to let goroutines finish before `main()` exits, since Go terminates the whole process the instant `main()`'s goroutine returns.
```go
go doWork(i)
time.Sleep(2 * time.Second) // crude: guesses how long goroutines need
```
**Why it exists (as a stopgap):** No synchronization primitives have been introduced yet at this point in the book.
**Explicitly flagged as inferior:** The text says this is "not a great way" to wait for completion — proper mechanisms (condition variables, semaphores in Ch. 5; waitgroups in Ch. 6) are promised as replacements.

### Sharing state via pointer (Chapter 3)
**When to use:** When two or more goroutines need to cooperate on a single evolving value (counter, accumulator, table) without message-passing infrastructure.
**Why it exists:** It's the most direct/lowest-level way to share data between goroutines, and forms the conceptual basis for all classic thread-safety techniques (mutexes, atomics) explored in subsequent chapters.
```go
count := 5
go countdown(&count) // shared via pointer, forces heap allocation via escape analysis
```
**Caveat:** Unsynchronized concurrent reads/writes via shared pointers are exactly what causes race conditions (see Stingy/Spendy) — this pattern is shown specifically so the *next* chapters can introduce mutexes/synchronization to fix it.

---

## Warnings, Pitfalls & Gotchas

- **Adding hardware without concurrent code changes achieves nothing.** The payroll module in Chapter 1's story didn't speed up despite adding CPU cores/memory because the underlying C++ code wasn't written to exploit multiple cores.
- **Never rely on scheduler execution order.** Both the OS scheduler and Go's own scheduler pick the next job/goroutine unpredictably; code that assumes a specific order will break intermittently and unpredictably across runs/machines/Go versions.
- **Omitting a wait mechanism can silently drop all goroutine output.** In `go doWork(i)` examples, if `main()` has no `time.Sleep` (or later, no proper synchronization), the process exits before any goroutine executes — Go terminates the whole process when the **main** goroutine finishes, unlike Java which waits for all threads.
- **`runtime.Gosched()` yielding does not eliminate race conditions** — it can reduce their *frequency* by luck of timing/optimization side effects, but is not a synchronization mechanism and provides no correctness guarantee.
- **Compound operations like `x += 10` are not atomic.** They expand into multiple machine instructions (read-modify-write), leaving a window where another goroutine can interleave and cause lost updates.
- **Race conditions are "Heisenbugs."** Attaching a debugger/breakpoints changes timing and can make the bug disappear during debugging sessions, making them notoriously hard to catch reactively — prevention via proper design is essential.
- **User-level threads block the entire process on a blocking I/O call**, since the OS is unaware of the individual user-level threads inside — defeats the purpose of concurrency unless non-blocking I/O is used (and not all I/O supports non-blocking mode).
- **User-level threads confined to a single kernel-level thread cannot achieve true parallelism** even on multicore hardware — this is why Go's M:N model (multiple kernel threads) matters.
- **Sharing memory means potential for one goroutine to overwrite/step on another's work** — this is the core reason synchronization (covered starting Ch. 4 with mutexes) is necessary whenever memory sharing is used.
- **The Go race detector is not infallible** — it only detects races that actually manifest during a given run; test thoroughly, and don't ship `-race` builds to production (major performance and memory cost).
- **`go tool compile -m` heap-escape / inlining differences**: using goroutines forces certain variables to the heap and disables some optimizations (like inlining) that would otherwise apply in sequential code — a subtle performance cost of concurrency beyond just synchronization overhead.
- **Fork() memory-copy cost:** Naively, `fork()` duplicates a process's entire memory space, which is expensive; without copy-on-write optimization, this cost is paid immediately even for memory that's never modified.
- **Excess process creation is expensive:** Because each process needs its own memory and resource allocation (and copying that state), it's unusual (and costly) for a program to use large numbers of concurrently-cooperating OS processes on one problem — threads/goroutines are preferred for fine-grained concurrency.

---

## Important Side Notes

- **Historical hardware trend (Chapter 1):** Before multicore CPUs, performance scaled with clock speed/transistor count, doubling roughly every two years. Physical limits (overheating, power draw) plus demand for mobile devices with good battery life drove the shift to multicore processors instead of ever-faster single cores.
- **Cloud computing side note:** Cheap, easily accessible large-scale compute resources only help if your code is written to actually use multiple processing units concurrently.
- **First commercial dual-core processor:** Released by Intel in 2005 (mentioned in Chapter 2 as a historical marker for the shift to multicore consumer/server hardware).
- **Multicore processor architecture note:** Cores typically share main memory and a bus interface, but each core has its own local CPU and at least one private memory cache — this is exactly what creates the cache-coherency challenges discussed in Chapter 3.
- **Interrupt controller complexity:** On multiprocessor systems, interrupt controllers are more advanced, able to target a single processor or a group of processors depending on the interrupt's purpose.
- **The RFC document source (rfc-editor.org)** was deliberately chosen for the letter-frequency example because the documents are static, plain-text, and have predictable sequential IDs — ideal for reproducible concurrency benchmarking without dealing with dynamic/formatted content.
- **Full code listings available externally:** The book repeatedly references `http://github.com/cutajarj/ConcurrentProgrammingWithGo` for all code listings and exercise solutions.
- **Book's overall roadmap (explicitly stated):** Starts with OS/concurrency fundamentals → race conditions → memory-sharing communication and classical primitives (mutexes, condition variables — Ch. 4+) → semaphores (Ch. 5) → waitgroups (Ch. 6) → message passing / channels / CSP (Ch. 7+) → concurrency patterns, deadlocks, and advanced topics like spinning locks (later chapters).
- **Go vs. other languages' thread termination semantics** differ meaningfully (Go kills everything when main goroutine ends; Java waits for all threads) — an important portability/mental-model gotcha when coming from other languages.
- **Terminology clarification:** "Green threads" (Java-originated term) vs. Go's goroutines vs. Java's newer "virtual threads" — all conceptually related (user-level-thread-like constructs) but historically and technically distinct.

---

## Quick Reference Table

| Construct / API | Purpose | Example |
|---|---|---|
| `go f(x)` | Launch `f(x)` as a new goroutine (async, non-blocking) | `go doWork(i)` |
| `time.Sleep(d)` | Pause execution for duration `d` (also used, crudely, to wait for goroutines) | `time.Sleep(2 * time.Second)` |
| `runtime.NumCPU()` | Returns number of logical CPUs visible to Go | `fmt.Println(runtime.NumCPU())` |
| `runtime.GOMAXPROCS(n)` | Get (`n<1`) or set the number of OS threads Go uses to run goroutines in parallel | `runtime.GOMAXPROCS(0)` |
| `runtime.Gosched()` | Yield the processor to give another goroutine a chance to run (no guarantee, no suspension) | `runtime.Gosched()` |
| `runtime.LockOSThread()` / `UnlockOSThread()` | Pin/unpin current goroutine to its current OS thread (e.g., for C interop) | `runtime.LockOSThread()` |
| `go tool compile -m file.go` | Show compiler's escape-analysis and inlining decisions | `go tool compile -m countdown.go` |
| `go run -race file.go` | Run program instrumented with the Go race detector | `go run -race stingyspendy.go` |
| `fork()` (UNIX syscall) | Duplicate current process into parent + child | returns PID to parent, `0` to child |
| `CreateProcess()` (Windows syscall) | Create and start a new process | OS-specific |
| `CreateThread()` (Windows) / `clone(CLONE_THREAD)` (Linux) | Create a new kernel-level thread | OS-specific |
| Pointer sharing (`&var`) | Share a variable's memory across goroutines | `go countdown(&count)` |

---

## Self-Check Questions

1. Explain the difference between concurrency and parallelism, and describe a scenario where a program is concurrent but *not* parallel.
2. Why did adding more CPU cores and memory fail to speed up HSS's payroll module in the opening story? What would need to change?
3. Walk through Amdahl's Law and Gustafson's Law: under what assumption does each operate, and how do they reach different conclusions about the value of adding more processors?
4. Trace through the OS job lifecycle (job queue → ready queue → CPU → I/O waiting queue → ready queue → CPU → terminate) and explain what triggers each transition, including the role of interrupts and context switches.
5. Compare processes and threads in terms of memory isolation, creation cost, and crash resilience. Why might a modern browser use *both* in combination?
6. What is Go's M:N threading model, and how does `GOMAXPROCS` plus work stealing allow goroutines to achieve true parallelism where classic user-level threads (N:1) could not?
7. Using the Stingy/Spendy example, explain step-by-step how a lost update can occur even though each operation (`+= 10`, `-= 10`) looks like a single line of code.
8. Why doesn't inserting `runtime.Gosched()` calls fix the Stingy/Spendy race condition, and why is the Go race detector (`-race`) not a complete substitute for proper synchronization?

# Synchronization with Mutexes, Condition Variables, Semaphores, Waitgroups & Barriers — Chapters 4–6 Refresher

## Overview

These three chapters cover Go's classic memory-sharing synchronization toolkit (the "SRC model," per later terminology). Chapter 4 introduces mutexes for protecting critical sections and readers–writer mutexes for read-heavy workloads, including a from-scratch implementation. Chapter 5 introduces condition variables (waiting for a condition to become true) and counting semaphores (bounded concurrent access + signal-storing), using them to fix starvation problems and build a write-preferring readers–writer lock. Chapter 6 covers waitgroups (waiting for a group of tasks to finish) and barriers (repeatedly synchronizing a group of goroutines at common checkpoints), showing both Go's built-in tools and from-scratch implementations built on the primitives from earlier chapters.

## Core Concepts

### Mutexes — Protecting Critical Sections (Chapter 4)

**Definition:** A **mutex** (mutual exclusion) is a concurrency-control primitive that allows only one execution (goroutine or kernel-level thread) into a critical section at a time. If multiple goroutines request the lock simultaneously, exactly one acquires it; the rest are suspended until it's released. When several goroutines are waiting, only one is resumed when the mutex becomes free.

**How it works internally (side note, deferred to Ch. 12):**
- On a single-processor system, a naive (bad) approach would be disabling interrupts while a thread holds the lock — but this risks a badly-written program (e.g., an infinite loop while holding the lock) freezing the entire system, and doesn't work at all on multiprocessor systems (other threads could still run in parallel on other CPUs).
- Real implementations rely on hardware support for an **atomic test-and-set operation**: an execution checks a memory location, and if it matches an expected value, atomically updates it to a "locked" flag. The hardware guarantees no other execution can touch that memory location mid-operation. Early hardware enforced this by locking the entire bus during the operation. If a thread finds the location already locked, the OS blocks that thread until it changes back to "free."

**Syntax:**
```go
var mutex sync.Mutex // zero value is unlocked
mutex.Lock()
// critical section
mutex.Unlock()
```

**Example — fixing Stingy/Spendy race condition from Ch. 3:**
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func stingy(money *int, mutex *sync.Mutex) {
    for i := 0; i < 1000000; i++ {
        mutex.Lock()
        *money += 10
        mutex.Unlock()
    }
    fmt.Println("Stingy Done")
}

func spendy(money *int, mutex *sync.Mutex) {
    for i := 0; i < 1000000; i++ {
        mutex.Lock()
        *money -= 10
        mutex.Unlock()
    }
    fmt.Println("Spendy Done")
}

func main() {
    money := 100
    mutex := sync.Mutex{}
    go stingy(&money, &mutex)
    go spendy(&money, &mutex)
    time.Sleep(2 * time.Second)
    mutex.Lock()
    fmt.Println("Money in bank account: ", money)
    mutex.Unlock()
}
// Output: Money in bank account: 100 (race eliminated)
```

**Important note (explicit in text):** Even *read-only* access to shared state should be protected by the same mutex used for writes — compiler optimizations can reorder instructions, so without synchronization a goroutine might read a stale cached copy rather than the latest value. Protect all critical sections, including reads.

**Note:** Go's zero-value `sync.Mutex{}` starts in the unlocked state — no separate initialization call is needed.

---

### Mutex Placement & Amdahl's Law Trade-off (Chapter 4)

**Definition/principle:** The portion of code held under a mutex lock effectively becomes **sequential** — only one goroutine executes it at a time. Per Amdahl's Law (Ch. 1), the more time spent in this sequential portion relative to the parallelizable portion, the more your overall speedup is capped. Therefore: **minimize both the duration and the frequency of Lock()/Unlock() calls.**

**Anti-pattern example (locking too much — turns concurrent code sequential):**
```go
func CountLetters(url string, frequency []int, mutex *sync.Mutex) {
    mutex.Lock() // WRONG: locks around the slow network download too
    resp, _ := http.Get(url)
    defer resp.Body.Close()
    // ... process body, update frequency ...
    mutex.Unlock()
}
```
This effectively serializes all downloads — total runtime matches the fully sequential (non-concurrent) version, just with random completion order.

**Correct pattern (lock only the fast, shared-state-touching part):**
```go
func CountLetters(url string, frequency []int, mutex *sync.Mutex) {
    resp, _ := http.Get(url)         // slow network I/O — done concurrently, NOT locked
    defer resp.Body.Close()
    if resp.StatusCode != 200 {
        panic("Server returning error code: " + resp.Status)
    }
    body, _ := io.ReadAll(resp.Body)
    mutex.Lock()                     // lock only around the fast, shared-memory-touching loop
    for _, b := range body {
        c := strings.ToLower(string(b))
        cIndex := strings.Index(AllLetters, c)
        if cIndex >= 0 {
            frequency[cIndex] += 1
        }
    }
    mutex.Unlock()
    fmt.Println("Completed:", url, time.Now().Format("15:04:05"))
}
```
**Result:** Much faster overall execution since the expensive download happens fully in parallel; only the brief in-memory counting step is serialized.

**Tip (explicit in text):** Decide which resources actually need protecting and where critical sections truly begin/end, then minimize the number and duration of `Lock()`/`Unlock()` calls — there's a per-call performance cost to mutex operations (explored further in Ch. 12).

---

### Non-Blocking Mutex Locks — `TryLock()` (Chapter 4)

**Definition:** `TryLock()` attempts to acquire the lock but never blocks: it returns `true` immediately if the lock was acquired, or `false` immediately if another goroutine currently holds it.

**Syntax:**
```go
if mutex.TryLock() {
    // lock acquired — do work
    mutex.Unlock()
} else {
    // lock unavailable — do something else instead of blocking
}
```

**Example — monitoring shared state without disrupting workers:**
```go
func main() {
    mutex := sync.Mutex{}
    var frequency = make([]int, 26)
    for i := 2000; i <= 2200; i++ {
        url := fmt.Sprintf("https://rfc-editor.org/rfc/rfc%d.txt", i)
        go listing4_5.CountLetters(url, frequency, &mutex)
    }
    for i := 0; i < 100; i++ {
        time.Sleep(100 * time.Millisecond)
        if mutex.TryLock() {
            for i, c := range listing4_5.AllLetters {
                fmt.Printf("%c-%d ", c, frequency[i])
            }
            mutex.Unlock()
        } else {
            fmt.Println("Mutex already being used")
        }
    }
}
```

**Side note:** Added to Go's mutex in version **1.18**. Go's own documentation is quoted as saying good uses of `TryLock()` are *rare*, and its use is *"often a sign of a deeper problem"* — because in Go, spawning a new goroutine is so cheap (unlike kernel threads in other languages) that it's usually simpler to just launch another goroutine to do other work while waiting for a lock, rather than polling with `TryLock()`.

---

### Readers–Writer Mutexes (Chapter 4)

**Definition:** A variant of the mutex that distinguishes between **read access** (which can safely be concurrent, since simple reads don't interfere with each other) and **write access** (which must remain exclusive). Multiple goroutines can hold the "read lock" simultaneously; only one goroutine at a time can hold the "write lock," and while a write lock is held, no readers can proceed either.

**Key insight (explicit in text):** *"Race conditions only happen if we change the shared state without proper synchronization. If we don't modify the shared data, there is no risk of race conditions."* Giving every reader exclusive access (as a plain mutex would) is unnecessarily restrictive when reads vastly outnumber writes.

**Go's built-in type — `sync.RWMutex`:**
```go
type RWMutex
    func (rw *RWMutex) Lock()          // acquire write lock
    func (rw *RWMutex) RLock()         // acquire read lock
    func (rw *RWMutex) RLocker() Locker
    func (rw *RWMutex) RUnlock()       // release read lock
    func (rw *RWMutex) TryLock() bool
    func (rw *RWMutex) TryRLock() bool
    func (rw *RWMutex) Unlock()        // release write lock
```

**Example — read-heavy sports-score-style server (basketball match updates):**
```go
func matchRecorder(matchEvents *[]string, mutex *sync.RWMutex) {
    for i := 0; ; i++ {
        mutex.Lock()  // exclusive write access
        *matchEvents = append(*matchEvents, "Match event "+strconv.Itoa(i))
        mutex.Unlock()
        time.Sleep(200 * time.Millisecond)
    }
}

func clientHandler(mEvents *[]string, mutex *sync.RWMutex, st time.Time) {
    for i := 0; i < 100; i++ {
        mutex.RLock()                     // shared read access
        allEvents := copyAllEvents(mEvents)
        mutex.RUnlock()
        fmt.Println(len(allEvents), "events copied in", time.Since(st))
    }
}

func main() {
    mutex := sync.RWMutex{}
    var matchEvents = make([]string, 0, 10000)
    for j := 0; j < 10000; j++ {
        matchEvents = append(matchEvents, "Match event")
    }
    go matchRecorder(&matchEvents, &mutex)
    start := time.Now()
    for j := 0; j < 5000; j++ {
        go clientHandler(&matchEvents, &mutex, start)
    }
    time.Sleep(100 * time.Second)
}
```
**Measured result (text's benchmark on a 10-core machine):** Using `RWMutex` instead of a plain `Mutex` gave roughly a **threefold increase in throughput** for this read-heavy workload (~10,000 events copied in ~10s vs. ~33s).

**Note:** Go's `sync.RWMutex` is documented as **write-preferring**: once a `Lock()` call is pending, no new readers can acquire `RLock()` until that writer proceeds — this prevents write starvation (details below).

---

### Building a Read-Preferring Readers–Writer Mutex From Scratch (Chapter 4)

**Definition:** A hand-built readers–writer lock using two plain mutexes and a counter, demonstrating the internals. Called "read-preferring" because as long as at least one reader holds a lock, no writer can proceed — readers are never blocked out in favor of a waiting writer.

**Analogy:** A room with two entrances (readers' and writers'). Multiple readers can be in the room at once, tracked by a `readersCounter`. The writers' entrance is "locked from inside" via a `globalLock` mutex that the **first** reader to enter acquires (blocking all writers), and that the **last** reader to leave releases.

**Syntax/structure:**
```go
type ReadWriteMutex struct {
    readersCounter int
    readersLock    sync.Mutex // serializes access to readersCounter itself
    globalLock     sync.Mutex // acts as the "writer's entrance" lock
}
```

**`ReadLock()` / `WriteLock()`:**
```go
func (rw *ReadWriteMutex) ReadLock() {
    rw.readersLock.Lock()
    rw.readersCounter++
    if rw.readersCounter == 1 {
        rw.globalLock.Lock() // first reader locks out any writer
    }
    rw.readersLock.Unlock()
}

func (rw *ReadWriteMutex) WriteLock() {
    rw.globalLock.Lock() // writer must acquire the same global lock
}
```

**`ReadUnlock()` / `WriteUnlock()`:**
```go
func (rw *ReadWriteMutex) ReadUnlock() {
    rw.readersLock.Lock()
    rw.readersCounter--
    if rw.readersCounter == 0 {
        rw.globalLock.Unlock() // last reader releases the writer's entrance
    }
    rw.readersLock.Unlock()
}

func (rw *ReadWriteMutex) WriteUnlock() {
    rw.globalLock.Unlock()
}
```

**Explicit limitation flagged in text:** *"This implementation of the readers–writer lock is read-preferring... if we have a consistent number of readers' goroutines hogging the read part of the mutex, a writer goroutine would be unable to acquire the mutex"* — i.e., it's vulnerable to **write starvation** (addressed via condition variables in Ch. 5).

---

### Condition Variables (Chapter 5)

**Definition:** A synchronization primitive that always works **together with a mutex**, allowing a goroutine to atomically release the mutex and suspend itself while waiting for some condition on shared state to become true — avoiding both the correctness problems of unsynchronized polling and the inefficiency of manual sleep-and-retry loops.

**Motivating problem — inefficient polling with `time.Sleep()`:**
```go
func spendy(money *int, mutex *sync.Mutex) {
    for i := 0; i < 200000; i++ {
        mutex.Lock()
        for *money < 50 {
            mutex.Unlock()
            time.Sleep(10 * time.Millisecond) // arbitrary guess — too short wastes CPU, too long wastes time
            mutex.Lock()
        }
        *money -= 50
        // ...
        mutex.Unlock()
    }
}
```
**Problem explicitly stated:** Choosing the sleep duration is a lose-lose guess — too short wastes CPU cycling on an unchanged condition; too long delays reacting to a change that already happened.

**How it works internally — the canonical pattern (numbered per the text's 8-step walkthrough):**
1. Goroutine A holds the mutex and checks a condition on shared state.
2. If not met, A calls `Wait()`.
3. `Wait()` **atomically**: (a) releases the mutex, (b) suspends A.
4. Goroutine B acquires the now-free mutex and updates the shared state.
5. B calls `Signal()` or `Broadcast()`, then unlocks the mutex.
6. A wakes up and **automatically reacquires** the mutex, then rechecks the condition (steps 2–6 may repeat).
7. Condition is eventually met.
8. A proceeds with its logic.

**Critical correctness note (explicit in text):** The atomicity of "release mutex + suspend" inside `Wait()` is essential — it guarantees no other execution can sneak in between those two steps, acquire the lock, and call `Signal()` before A is actually suspended (which would cause a missed signal).

**Syntax — Go's `sync.Cond`:**
```go
type Cond
    func NewCond(l Locker) *Cond
    func (c *Cond) Broadcast()
    func (c *Cond) Signal()
    func (c *Cond) Wait()

type Locker interface {
    Lock()
    Unlock()
}
```
A `sync.Mutex` satisfies the `Locker` interface, so it's typically used to build a `Cond`.

**Example — Stingy/Spendy fixed with a condition variable (preventing negative balance):**
```go
func main() {
    money := 100
    mutex := sync.Mutex{}
    cond := sync.NewCond(&mutex)
    go stingy(&money, cond)
    go spendy(&money, cond)
    time.Sleep(2 * time.Second)
    mutex.Lock()
    fmt.Println("Money in bank account: ", money)
    mutex.Unlock()
}

func stingy(money *int, cond *sync.Cond) {
    for i := 0; i < 1000000; i++ {
        cond.L.Lock()
        *money += 10
        cond.Signal()
        cond.L.Unlock()
    }
    fmt.Println("Stingy Done")
}

func spendy(money *int, cond *sync.Cond) {
    for i := 0; i < 200000; i++ {
        cond.L.Lock()
        for *money < 50 {
            cond.Wait()
        }
        *money -= 50
        if *money < 0 {
            fmt.Println("Money is negative!")
            os.Exit(1)
        }
        cond.L.Unlock()
    }
    fmt.Println("Spendy Done")
}
```
**Note:** `cond.L` is the `Locker` (mutex) associated with the condition variable — you access/lock/unlock it through the `Cond` struct's `L` field.

**Side note — "Monitors":** A **monitor** is the general synchronization-pattern name for "a mutex with an associated condition variable" used for waiting/signaling. Some languages (e.g., Java) build a monitor into every object instance; in Go, you get this pattern any time you pair a `sync.Mutex` with a `sync.Cond`.

---

### Missed Signals (Chapter 5)

**Definition/danger:** If `Signal()` or `Broadcast()` is called while **no** goroutine is currently suspended in `Wait()`, the signal is simply **lost** — there is no buffering or memory of the signal for a future `Wait()` call to pick up.

**Demonstration of the bug:**
```go
func doWork(cond *sync.Cond) {
    fmt.Println("Work started")
    fmt.Println("Work finished")
    cond.Signal() // called WITHOUT holding the mutex — race with main's Wait()
}

func main() {
    cond := sync.NewCond(&sync.Mutex{})
    cond.L.Lock()
    for i := 0; i < 50000; i++ {
        go doWork(cond)
        fmt.Println("Waiting for child goroutine ")
        cond.Wait()
        fmt.Println("Child goroutine finished")
    }
    cond.L.Unlock()
}
```
Eventually crashes with:
```
fatal error: all goroutines are asleep - deadlock!
```
**Why:** Occasionally `doWork()`'s goroutine calls `Signal()` *before* `main()` has actually reached `cond.Wait()` — the signal fires into the void, and `main()` then blocks forever waiting for a signal that already happened. Go's runtime detects this dead end and panics.

**Tip (debugging aid from text):** Inserting `runtime.Gosched()` right before `cond.Wait()` in `main()` increases the odds of reproducing this race, since it gives the child goroutine more opportunity to run (and signal) before `main()` reaches its wait.

**Fix — always call `Signal()`/`Broadcast()` while holding the associated mutex:**
```go
func doWork(cond *sync.Cond) {
    fmt.Println("Work started")
    fmt.Println("Work finished")
    cond.L.Lock()
    cond.Signal()
    cond.L.Unlock()
}
```
**Why this works:** Because `Wait()` only releases the mutex *while suspended*, if the signaling goroutine must acquire that same mutex before calling `Signal()`, it's guaranteed the waiter is either already asleep (and will receive the signal) or hasn't yet started waiting (in which case... — the pattern still requires structuring code so the check-and-wait happens under the same lock discipline, as shown in the Stingy/Spendy fix above with the `for` loop re-checking the condition).

**Tip (explicit, repeated in text):** *"Always use Signal(), Broadcast(), and Wait() when holding the mutex lock to avoid synchronization problems."*

---

### `Signal()` vs `Broadcast()` (Chapter 5)

**Definition:**
- **`Signal()`** wakes exactly **one** arbitrarily-chosen goroutine among those suspended in `Wait()` on that condition variable. You have no control over which one.
- **`Broadcast()`** wakes **all** goroutines currently suspended in `Wait()` on that condition variable.

**Example — waiting for all players to join a multiplayer game:**
```go
func main() {
    cond := sync.NewCond(&sync.Mutex{})
    playersInGame := 4
    for playerId := 0; playerId < 4; playerId++ {
        go playerHandler(cond, &playersInGame, playerId)
        time.Sleep(1 * time.Second) // stagger connections
    }
}

func playerHandler(cond *sync.Cond, playersRemaining *int, playerId int) {
    cond.L.Lock()
    fmt.Println(playerId, ": Connected")
    *playersRemaining--
    if *playersRemaining == 0 {
        cond.Broadcast() // wake ALL waiting players at once
    }
    for *playersRemaining > 0 {
        fmt.Println(playerId, ": Waiting for more players")
        cond.Wait()
    }
    cond.L.Unlock()
    fmt.Println("All players connected. Ready player", playerId)
}
```
**Why `Broadcast()` here, not `Signal()`:** Multiple goroutines (up to 3, in this 4-player example) may simultaneously be suspended in `Wait()`; all of them need to wake up together once the last player connects — `Signal()` would only release one, leaving the others stuck.

---

### Write-Starvation & the Write-Preferring Readers–Writer Lock (Chapter 5)

**Definition — Starvation:** *"A situation where an execution is blocked from gaining access to a shared resource because the resource is made unavailable for a long time (or indefinitely) by other greedy executions."*

**Demonstration of write starvation** using the Ch. 4 read-preferring `ReadWriteMutex`:
```go
func main() {
    rwMutex := listing4_12.ReadWriteMutex{}
    for i := 0; i < 2; i++ {
        go func() {
            for {
                rwMutex.ReadLock()
                time.Sleep(1 * time.Second)
                fmt.Println("Read done")
                rwMutex.ReadUnlock()
            }
        }()
    }
    time.Sleep(1 * time.Second)
    rwMutex.WriteLock() // never succeeds — starved!
    fmt.Println("Write finished")
}
```
**Result:** Prints "Read done" forever — the two reader goroutines continuously trade off holding the read lock (never both releasing it at the same instant), so the writer can never acquire `globalLock`.

**Side note — Go's own `RWMutex` avoids this:** Go's documentation (quoted in text) states explicitly that once a `Lock()` call is pending, the implementation excludes new readers from acquiring `RLock()` — Go's `sync.RWMutex` is **write-preferring** by design, specifically to guarantee that the lock "eventually becomes available."

**Design for a write-preferring lock — required properties:**
- `readersCounter` (int, starts at 0) — active reader count.
- `writersWaiting` (int, starts at 0) — count of writers currently blocked waiting.
- `writerActive` (bool, starts at false) — whether a writer currently holds the lock.
- A `sync.Cond` (with its mutex) to coordinate waiting/waking on these fields.

**Syntax/structure:**
```go
type ReadWriteMutex struct {
    readersCounter int
    writersWaiting int
    writerActive   bool
    cond           *sync.Cond
}

func NewReadWriteMutex() *ReadWriteMutex {
    return &ReadWriteMutex{cond: sync.NewCond(&sync.Mutex{})}
}
```

**`ReadLock()` — blocks if any writer is waiting OR active (this is what gives writers priority):**
```go
func (rw *ReadWriteMutex) ReadLock() {
    rw.cond.L.Lock()
    for rw.writersWaiting > 0 || rw.writerActive {
        rw.cond.Wait()
    }
    rw.readersCounter++
    rw.cond.L.Unlock()
}
```

**`WriteLock()` — registers itself as waiting, then blocks while any readers or another writer is active:**
```go
func (rw *ReadWriteMutex) WriteLock() {
    rw.cond.L.Lock()
    rw.writersWaiting++
    for rw.readersCounter > 0 || rw.writerActive {
        rw.cond.Wait()
    }
    rw.writersWaiting--
    rw.writerActive = true
    rw.cond.L.Unlock()
}
```

**`ReadUnlock()` — last reader out broadcasts (to potentially wake a waiting writer):**
```go
func (rw *ReadWriteMutex) ReadUnlock() {
    rw.cond.L.Lock()
    rw.readersCounter--
    if rw.readersCounter == 0 {
        rw.cond.Broadcast()
    }
    rw.cond.L.Unlock()
}
```

**`WriteUnlock()` — always broadcasts (readers or writers may be waiting):**
```go
func (rw *ReadWriteMutex) WriteUnlock() {
    rw.cond.L.Lock()
    rw.writerActive = false
    rw.cond.Broadcast()
    rw.cond.L.Unlock()
}
```
**Why writers still win even when both readers and writers are woken:** Any woken reader re-checks its `for rw.writersWaiting > 0 || rw.writerActive` condition and goes back to sleep if a writer is still waiting/active — so writers effectively cut the line.

---

### Counting Semaphores (Chapter 5)

**Definition:** A concurrency-control primitive that allows up to **N** concurrent executions into a resource (rather than just 1, like a mutex). Once all **N** permits are in use, further `Acquire()` calls block until a permit is freed via `Release()`.

**Relationship to mutexes:** A mutex is functionally equivalent to a semaphore with `N = 1`. A semaphore with exactly one permit is called a **binary semaphore**.

**Important usage-pattern difference (explicit in text):** With a mutex, the goroutine that locks it is expected to be the one that unlocks it. With a semaphore, this is **not** necessarily true — a different goroutine can legitimately call `Release()` than the one that called `Acquire()` (this asymmetry is exploited later for signaling, below).

**Core operations:**
- **New semaphore** — create with X initial permits.
- **Acquire** — take one permit; block if none available.
- **Release** — return one permit, potentially unblocking a waiter.

**Building one from scratch (using a condition variable internally):**
```go
type Semaphore struct {
    permits int
    cond    *sync.Cond
}

func NewSemaphore(n int) *Semaphore {
    return &Semaphore{
        permits: n,
        cond:    sync.NewCond(&sync.Mutex{}),
    }
}

func (rw *Semaphore) Acquire() {
    rw.cond.L.Lock()
    for rw.permits <= 0 {
        rw.cond.Wait()
    }
    rw.permits--
    rw.cond.L.Unlock()
}

func (rw *Semaphore) Release() {
    rw.cond.L.Lock()
    rw.permits++
    rw.cond.Signal() // only one permit freed -> only wake one waiter
    rw.cond.L.Unlock()
}
```

**Side note:** Go's standard library doesn't bundle a semaphore type, but an extension package exists at `golang.org/x/sync` — part of the Go project but maintained under looser compatibility guarantees than the core `sync` package.

**Historical side note:** Semaphores were invented by Dutch computer scientist **Edsger Dijkstra** in his unpublished 1962 paper *"Over Seinpalen"* ("About Semaphores"). The name references early railway signaling systems, where a pivoted arm's angle conveyed different meanings to train drivers.

---

### Semaphores as Signal-Storing Alternative to Condition Variables (Chapter 5)

**Definition/insight:** Unlike a bare condition variable's `Signal()` (which is lost if no one is waiting), a semaphore's `Release()` call **permanently increments the permit counter** — so if `Release()` happens *before* the corresponding `Acquire()`, the "signal" isn't lost; the next `Acquire()` simply succeeds immediately.

**Example — reliably waiting for a goroutine to finish (fixing the missed-signal bug from earlier):**
```go
func main() {
    semaphore := listing5_16.NewSemaphore(0) // starts with 0 permits
    for i := 0; i < 50000; i++ {
        go doWork(semaphore)
        fmt.Println("Waiting for child goroutine ")
        semaphore.Acquire() // blocks only if Release() hasn't happened yet
        fmt.Println("Child goroutine finished")
    }
}

func doWork(semaphore *listing5_16.Semaphore) {
    fmt.Println("Work started")
    fmt.Println("Work finished")
    semaphore.Release() // safe even if called before Acquire()
}
```
**Why this fixes the earlier bug:** If `Release()` (analogous to the earlier `Signal()`) happens before `Acquire()` (analogous to `Wait()`), the permit count simply becomes 1, and the subsequent `Acquire()` call returns immediately rather than blocking forever.

---

### Waitgroups (Chapter 6)

**Definition:** A synchronization abstraction for waiting until a **group of concurrent tasks** all report completion — conceptually like a project manager tracking a fixed set of assigned tasks and being notified once every one is done.

**Go's built-in `sync.WaitGroup` — three operations:**
- `Add(delta int)` — increments the internal counter by `delta`.
- `Done()` — decrements the counter by 1 (equivalent to `Add(-1)`).
- `Wait()` — blocks until the counter reaches 0.

**Syntax:**
```go
var wg sync.WaitGroup
wg.Add(n)
// ... n goroutines each eventually call wg.Done() ...
wg.Wait()
```

**Example:**
```go
func main() {
    wg := sync.WaitGroup{}
    wg.Add(4)
    for i := 1; i <= 4; i++ {
        go doWork(i, &wg)
    }
    wg.Wait()
    fmt.Println("All complete")
}

func doWork(id int, wg *sync.WaitGroup) {
    i := rand.Intn(5)
    time.Sleep(time.Duration(i) * time.Second)
    fmt.Println(id, "Done working after", i, "seconds")
    wg.Done()
}
```

**Applied fix to the letter-frequency program (replacing a crude fixed `Sleep()` from earlier chapters):**
```go
func main() {
    wg := sync.WaitGroup{}
    wg.Add(31)
    mutex := sync.Mutex{}
    var frequency = make([]int, 26)
    for i := 1000; i <= 1030; i++ {
        url := fmt.Sprintf("https://rfc-editor.org/rfc/rfc%d.txt", i)
        go func() {
            listing4_5.CountLetters(url, frequency, &mutex)
            wg.Done()
        }()
    }
    wg.Wait()
    mutex.Lock()
    for i, c := range listing4_5.AllLetters {
        fmt.Printf("%c-%d ", c, frequency[i])
    }
    mutex.Unlock()
}
```
**Benefit over `time.Sleep()`:** The program now proceeds exactly when work finishes, rather than guessing a fixed wait duration.

---

### Building a Waitgroup From Semaphores (Chapter 6)

**Definition/trick:** Initialize a semaphore with `1 - size` permits (a **negative** starting value). Each `Done()` call releases one permit (moving the count toward 1); `Wait()` simply calls `Acquire()`, which blocks until the running total finally reaches a positive value (1).

```go
type WaitGrp struct {
    sema *listing5_16.Semaphore
}

func NewWaitGrp(size int) *WaitGrp {
    return &WaitGrp{sema: listing5_16.NewSemaphore(1 - size)}
}

func (wg *WaitGrp) Wait() {
    wg.sema.Acquire()
}

func (wg *WaitGrp) Done() {
    wg.sema.Release()
}
```
**Worked example (size 3):** Semaphore starts at `1 - 3 = -2`. Three `Done()` calls bring it to `-1`, `0`, then `1`. Once it's `1` (positive), `Wait()`'s `Acquire()` call succeeds.

**Explicit limitation (flagged in text):**
1. **Fixed size at creation** — you must know the total task count up front; the semaphore-backed size cannot grow later. Go's real `WaitGroup` allows calling `Add()` at any time, even *while* other goroutines are already waiting.
2. **Only one waiter is properly supported** — if multiple goroutines call `Wait()`, only one is released, since the permit count only increases to 1 total.

---

### Case Study Requiring Dynamic Waitgroup Resizing — Recursive File Search (Chapter 6)

**Definition of the problem:** A concurrent recursive directory search doesn't know in advance how many subdirectories (and thus goroutines) it will need — the waitgroup's size must grow *as the search discovers more directories*, which the semaphore-based waitgroup above cannot support, but Go's built-in `sync.WaitGroup` can (via repeated `Add()` calls).

**Example:**
```go
func fileSearch(dir string, filename string, wg *sync.WaitGroup) {
    files, _ := os.ReadDir(dir)
    for _, file := range files {
        fpath := filepath.Join(dir, file.Name())
        if strings.Contains(file.Name(), filename) {
            fmt.Println(fpath)
        }
        if file.IsDir() {
            wg.Add(1)                          // grows the waitgroup dynamically
            go fileSearch(fpath, filename, wg)  // recurse into subdirectory
        }
    }
    wg.Done()
}

func main() {
    wg := sync.WaitGroup{}
    wg.Add(1)
    go fileSearch(os.Args[1], os.Args[2], &wg)
    wg.Wait()
}
```

---

### Building a Fully-Featured Waitgroup With Condition Variables (Chapter 6)

**Definition:** A from-scratch waitgroup implementation that overcomes both limitations of the semaphore-based version — it supports dynamic resizing (`Add()` any time) and correctly supports **multiple** simultaneous waiters (via `Broadcast()`).

**Syntax/structure:**
```go
type WaitGrp struct {
    groupSize int
    cond      *sync.Cond
}

func NewWaitGrp() *WaitGrp {
    return &WaitGrp{
        cond: sync.NewCond(&sync.Mutex{}),
    }
}
```

**`Add(delta)` — simply adjusts the counter under the mutex:**
```go
func (wg *WaitGrp) Add(delta int) {
    wg.cond.L.Lock()
    wg.groupSize += delta
    wg.cond.L.Unlock()
}
```

**`Wait()` — blocks (via condition variable) while the group size is still positive:**
```go
func (wg *WaitGrp) Wait() {
    wg.cond.L.Lock()
    for wg.groupSize > 0 {
        wg.cond.Wait()
    }
    wg.cond.L.Unlock()
}
```

**`Done()` — decrements the size; the goroutine that brings it to exactly 0 broadcasts:**
```go
func (wg *WaitGrp) Done() {
    wg.cond.L.Lock()
    wg.groupSize--
    if wg.groupSize == 0 {
        wg.cond.Broadcast() // must use Broadcast, not Signal, since multiple Wait() callers may exist
    }
    wg.cond.L.Unlock()
}
```
**Why `Broadcast()` and not `Signal()` here:** Multiple goroutines could simultaneously be blocked in `Wait()` — all must be released together once the group's work is complete, which `Signal()` (waking only one) cannot guarantee.

---

### Barriers (Chapter 6)

**Definition:** A synchronization primitive that repeatedly synchronizes a **fixed group of goroutines** at common checkpoints in their execution — every participant must call `Wait()` before *any* of them are allowed to proceed past that point. Unlike waitgroups (which combine a separate `Done()`/`Wait()` pair and are typically used once), a barrier's `Wait()` **combines both roles into one atomic call**, and — depending on implementation — a barrier can be **reused** repeatedly (a reusable barrier is sometimes called a **cyclic barrier**).

**Analogy given:** A private plane departs only once all passengers have arrived at the terminal (a **barrier** — everyone waits for everyone). Contrast with a **waitgroup**: the pilot waiting for several *different* preparatory tasks (refueling, loading luggage, boarding) to each individually complete.

**How it works internally:**
- Track a `waitCount` (how many participants have currently called `Wait()`) against a fixed `size` (total number of participants).
- If `waitCount < size` after incrementing, the calling goroutine suspends on a condition variable.
- If `waitCount == size`, reset `waitCount` to 0 and `Broadcast()` — releasing all suspended goroutines together so they can proceed (and potentially call `Wait()` again on a future cycle, since the counter was reset).

**Syntax/structure (Go doesn't bundle a barrier type — built from scratch):**
```go
type Barrier struct {
    size      int
    waitCount int
    cond      *sync.Cond
}

func NewBarrier(size int) *Barrier {
    condVar := sync.NewCond(&sync.Mutex{})
    return &Barrier{size, 0, condVar}
}

func (b *Barrier) Wait() {
    b.cond.L.Lock()
    b.waitCount += 1
    if b.waitCount == b.size {
        b.waitCount = 0
        b.cond.Broadcast()
    } else {
        b.cond.Wait()
    }
    b.cond.L.Unlock()
}
```

**Example — two goroutines with different work durations staying in lockstep:**
```go
func workAndWait(name string, timeToWork int, barrier *listing6_10.Barrier) {
    start := time.Now()
    for {
        fmt.Println(time.Since(start), name, "is running")
        time.Sleep(time.Duration(timeToWork) * time.Second)
        fmt.Println(time.Since(start), name, "is waiting on barrier")
        barrier.Wait()
    }
}

func main() {
    barrier := listing6_10.NewBarrier(2)
    go workAndWait("Red", 4, barrier)
    go workAndWait("Blue", 10, barrier)
    time.Sleep(100 * time.Second)
}
```
**Observed behavior:** The faster "Red" goroutine (4s of work) reaches the barrier and waits; only once the slower "Blue" goroutine (10s of work) also reaches the barrier do *both* resume together, and the whole cycle repeats — the two stay synchronized at each round, even though they'd otherwise run at different paces.

---

### Concurrent Matrix Multiplication Using Barriers (Chapter 6)

**Definition:** A realistic multi-goroutine application demonstrating repeated barrier synchronization: load inputs → compute concurrently (one goroutine per output row) → output results → repeat.

**Background — Matrix multiplication complexity (side note):** The naive triple-nested-loop algorithm for multiplying two n×n matrices has runtime complexity **O(n³)** — doubling the matrix dimension multiplies the runtime by 2³ = 8×.

**Side note — faster algorithms:** Volker Strassen (1969) devised an O(n^2.807) algorithm — a real but only significant improvement for very large matrices. More recent algorithms have even better theoretical complexity but are so-called **"galactic algorithms"** — they only outperform simpler methods at input sizes far too large to fit in any real computer's memory, so they're not used in practice.

**Sequential baseline:**
```go
func matrixMultiply(matrixA, matrixB, result *[matrixSize][matrixSize]int) {
    for row := 0; row < matrixSize; row++ {
        for col := 0; col < matrixSize; col++ {
            sum := 0
            for i := 0; i < matrixSize; i++ {
                sum += matrixA[row][i] * matrixB[i][col]
            }
            result[row][col] = sum
        }
    }
}
```

**Concurrency design:** One goroutine per output row. For an n×n result, spawn n row-goroutines plus a barrier sized `n + 1` (rows + the `main()` goroutine).

**Per-row worker (loops forever, synchronizing via the barrier each cycle):**
```go
func rowMultiply(matrixA, matrixB, result *[matrixSize][matrixSize]int,
    row int, barrier *listing6_10.Barrier) {
    for {
        barrier.Wait() // wait for main() to finish loading new input matrices
        for col := 0; col < matrixSize; col++ {
            sum := 0
            for i := 0; i < matrixSize; i++ {
                sum += matrixA[row][i] * matrixB[i][col]
            }
            result[row][col] = sum
        }
        barrier.Wait() // signal this row is done; wait for all other rows too
    }
}

func main() {
    var matrixA, matrixB, result [matrixSize][matrixSize]int
    barrier := listing6_10.NewBarrier(matrixSize + 1)
    for row := 0; row < matrixSize; row++ {
        go rowMultiply(&matrixA, &matrixB, &result, row, barrier)
    }
    for i := 0; i < 4; i++ {
        generateRandMatrix(&matrixA)
        generateRandMatrix(&matrixB)
        barrier.Wait() // release row-workers to start computing
        barrier.Wait() // wait for all rows to finish before printing
        for i := 0; i < matrixSize; i++ {
            fmt.Println(matrixA[i], matrixB[i], result[i])
        }
        fmt.Println()
    }
}
```
**Why the barrier appears twice per cycle in `rowMultiply`:** Once to wait for fresh input data from `main()`, and again at the end of the row computation to signal completion and wait for sibling row-goroutines to also finish before the cycle repeats.

**Explicit caveat/side note — "Barriers or no barriers?":** This load→wait→collect pattern is most valuable when creating new execution units is *expensive* (e.g., kernel-level threads), since a barrier lets you *reuse* the same long-lived goroutines across cycles instead of paying creation cost repeatedly. In Go, goroutine creation is cheap, so for many use cases it's simpler to just spawn fresh goroutines each round and use a `WaitGroup` — barriers still pay off mainly when synchronizing **large numbers** of goroutines repeatedly.

---

## Patterns & Idioms

### Minimal Critical Section Pattern (Chapter 4)
**When to use:** Any time a mutex-protected critical section contains both "slow, non-shared work" (e.g., network I/O) and "fast, shared-state work" (e.g., updating a counter/slice).
**Why:** Following Amdahl's Law, minimizing the serialized portion maximizes achievable speedup. Only lock around the smallest section actually touching shared state.
```go
resp, _ := http.Get(url)  // NOT locked — independent, slow work
mutex.Lock()
frequency[cIndex] += 1     // locked — brief, shared-state work
mutex.Unlock()
```

### Read-Mostly Optimization via Readers–Writer Locks (Chapter 4)
**When to use:** Workloads where reads vastly outnumber writes (e.g., thousands of reads/sec vs. a few writes/sec, as in the basketball score server).
**Why:** Lets many readers proceed concurrently instead of serializing them like a plain mutex would, since concurrent reads of unmodified data cannot race.
```go
mutex.RLock()
data := copyData(shared)
mutex.RUnlock()
```

### Condition-Checked Retry Loop Under a Condition Variable (Chapter 5)
**When to use:** Whenever a goroutine must wait for a data-dependent condition (not just "is the lock free") before proceeding — e.g., "is there enough money," "have enough players joined."
**Why:** Always **re-check the condition in a loop** after waking (`for condition_not_met { cond.Wait() }`), rather than assuming it's true immediately after `Wait()` returns — since `Signal()`/`Broadcast()` might wake a goroutine whose specific condition still isn't satisfied (e.g., another waiter grabbed the resource first).
```go
cond.L.Lock()
for *money < 50 {
    cond.Wait()
}
*money -= 50
cond.L.Unlock()
```

### Semaphore as a Reliable One-Shot Completion Signal (Chapter 5)
**When to use:** Waiting for a single asynchronous task's completion where the "done" event might fire before the waiter starts waiting.
**Why:** Unlike a bare condition variable's `Signal()`, a semaphore's `Release()` permanently records the event as a permit — no race between "did the waiter start waiting yet."
```go
sem := NewSemaphore(0)
go func() {
    doWork()
    sem.Release() // safe even if called first
}()
sem.Acquire()     // blocks only if Release() hasn't happened yet
```

### Waitgroup for Fan-Out Task Completion (Chapter 6)
**When to use:** Spawning N independent goroutines and needing to know precisely when all N have finished, replacing fragile `time.Sleep()`-based guesses.
**Why:** Precise, event-driven completion signaling instead of arbitrary fixed delays.
```go
wg.Add(n)
for i := 0; i < n; i++ {
    go func() { defer wg.Done(); doWork() }()
}
wg.Wait()
```

### Dynamically-Growing Waitgroup for Recursive/Unknown-Size Work (Chapter 6)
**When to use:** Recursive or dynamically-discovered workloads (e.g., directory tree search) where the total task count isn't known upfront.
**Why:** Go's built-in `WaitGroup` allows `Add()` calls even after some goroutines are already running/waiting — essential for recursive fan-out.
```go
wg.Add(1)
go func() {
    // ...discover more work...
    wg.Add(1)
    go recurse(...)
    wg.Done()
}()
```

### Barrier for Cyclic Load–Compute–Collect Pipelines (Chapter 6)
**When to use:** Repeated rounds of "load data → compute concurrently across a fixed set of workers → collect/output results → repeat," especially when worker creation is costly or the same goroutine pool should be reused indefinitely.
**Why:** A single barrier call synchronizes both "signal I'm done with my part" and "wait for everyone else" in one atomic step, letting long-lived worker goroutines cycle between computing and waiting without recreating them.
```go
for {
    barrier.Wait() // wait for fresh input
    computeMyPart()
    barrier.Wait() // signal done, wait for peers
}
```

---

## Warnings, Pitfalls & Gotchas

- **Locking too broadly turns concurrent code sequential.** Wrapping a mutex around slow, independent work (e.g., a network download) alongside the shared-state update destroys the concurrency benefit — per Amdahl's Law, this caps your achievable speedup severely.
- **Even read-only access to shared state needs mutex protection** — compiler/hardware reordering and caching can otherwise serve a goroutine a stale value.
- **`TryLock()` is rarely the right tool in Go** — Go's own documentation calls correct uses "rare" and often "a sign of a deeper problem," since spawning another goroutine is usually simpler and cheaper than polling for a lock.
- **A naive read-preferring readers–writer lock can starve writers indefinitely** if readers keep overlapping their access windows — demonstrated explicitly with two continuously-cycling reader goroutines blocking `main()`'s `WriteLock()` forever.
- **Polling a condition with `time.Sleep()` is inefficient and imprecise** — too short wastes CPU, too long delays responsiveness; condition variables solve this properly.
- **Calling `Signal()`/`Broadcast()` without holding the associated mutex risks a missed signal** — if no goroutine is yet in `Wait()`, the signal is silently lost forever, potentially leaving a waiter blocked permanently (Go's runtime will detect a resulting whole-program deadlock and panic, but only if *all* goroutines end up stuck).
- **Always re-check the wait condition in a `for` loop after `Wait()` returns**, not just an `if` — a woken goroutine's specific condition might not yet be satisfied even though *some* signal arrived.
- **Semaphores differ from mutexes in unlock discipline** — with a mutex, the locking goroutine should be the one to unlock; with a semaphore, a *different* goroutine may legitimately call `Release()` (this is actually exploited for signaling patterns, but is a mental-model shift from mutex usage).
- **A semaphore-based waitgroup implementation is fixed-size at creation and supports only one waiter** — using it for scenarios needing dynamic resizing (e.g., recursive fan-out) or multiple concurrent `Wait()` callers will silently misbehave (only one waiter gets released).
- **`Done()` in a multi-waiter waitgroup must use `Broadcast()`, not `Signal()`** — otherwise only one of potentially several suspended waiters would be released when the group's work completes.
- **Barriers provide limited benefit in Go for simple one-shot fan-out/fan-in** — since goroutine creation is cheap in Go (unlike OS threads), the load→create-goroutines→waitgroup pattern is often simpler than barrier-based long-lived-worker reuse; barriers pay off mainly for repeated cycles or very large goroutine counts.
- **A barrier's `Wait()` call combines "I'm done" and "wait for others" atomically** — this is a key conceptual difference from waitgroups (`Done()` and `Wait()` are separate calls), and mixing up the two mental models can lead to incorrect synchronization logic.

---

## Important Side Notes

- **Mutex hardware implementation (deferred detail):** Real mutexes rely on hardware-guaranteed atomic test-and-set operations; naive software-only approaches (disabling interrupts) are unsafe and don't generalize to multiprocessor systems. Full implementation details (including how Go builds its own mutex) are promised for **Chapter 12**.
- **`TryLock()` was added in Go 1.18** — a relatively recent addition to the `sync.Mutex` API.
- **Go's `sync.RWMutex` is write-preferring by design** — explicitly documented by Go to prevent write starvation; the book's own from-scratch Ch. 4 implementation is read-preferring, and Ch. 5 shows how to build a write-preferring version instead.
- **Matrix multiplication complexity and "galactic algorithms":** The standard algorithm is O(n³); Strassen's 1969 algorithm achieves O(n^2.807) but only pays off at very large sizes; some newer algorithms have even better asymptotic complexity but are impractical ("galactic") because their crossover point requires matrices too large to fit in real memory.
- **Semaphore historical origin:** Coined and formalized by Edsger Dijkstra in his 1962 (unpublished) paper "Over Seinpalen," with the name borrowed from railway signal-arm systems.
- **Go's `golang.org/x/sync` extension package** contains a semaphore implementation (among other things) not included in the Go standard library — maintained by the Go project itself but with looser compatibility guarantees than core `sync`.
- **"Monitor" terminology:** The general pattern of "mutex + condition variable" is called a monitor in concurrency literature; some languages (Java) bake a monitor into every object instance, while Go achieves the same effect explicitly via `sync.Mutex` + `sync.Cond`.
- **Cross-references to later chapters:** Chapter 12 promises deeper coverage of mutex internals (atomics, OS-level implementation, Go's own mutex internals). The write-preferring readers-writer lock built in Ch. 5 directly improves on the read-preferring one built in Ch. 4.
- **Barrier terminology:** A reusable barrier is called a **cyclic barrier** in general concurrency literature — Go doesn't provide one out of the box, unlike waitgroups, so the book builds one from a condition variable.

---

## Quick Reference Table

| Construct / API | Purpose | Example |
|---|---|---|
| `sync.Mutex{}` | Zero-value mutex, starts unlocked | `var mu sync.Mutex` |
| `mu.Lock()` / `mu.Unlock()` | Enter/exit a critical section | `mu.Lock(); x++; mu.Unlock()` |
| `mu.TryLock() bool` | Non-blocking lock attempt | `if mu.TryLock() { ... }` |
| `sync.RWMutex{}` | Mutex distinguishing read vs. write access | `var rw sync.RWMutex` |
| `rw.RLock()` / `rw.RUnlock()` | Shared (concurrent) read access | `rw.RLock(); read(x); rw.RUnlock()` |
| `rw.Lock()` / `rw.Unlock()` | Exclusive write access | `rw.Lock(); x = v; rw.Unlock()` |
| `sync.NewCond(locker)` | Create a condition variable bound to a mutex | `cond := sync.NewCond(&mu)` |
| `cond.Wait()` | Atomically unlock + suspend; re-locks on wake | `for !ready { cond.Wait() }` |
| `cond.Signal()` | Wake exactly one waiter | `cond.Signal()` |
| `cond.Broadcast()` | Wake all waiters | `cond.Broadcast()` |
| `NewSemaphore(n)` (custom) | Create a semaphore with `n` permits | `sem := NewSemaphore(3)` |
| `sem.Acquire()` / `sem.Release()` | Take/return a permit | `sem.Acquire(); work(); sem.Release()` |
| `sync.WaitGroup{}` | Track completion of a group of tasks | `var wg sync.WaitGroup` |
| `wg.Add(n)` / `wg.Done()` / `wg.Wait()` | Register tasks / mark one done / block until all done | `wg.Add(3); ...; wg.Wait()` |
| `NewBarrier(size)` (custom) | Create a reusable synchronization point for `size` goroutines | `b := NewBarrier(4)` |
| `barrier.Wait()` | Block until all `size` participants call `Wait()`, then release all | `barrier.Wait()` |

---

## Self-Check Questions

1. Explain why locking a mutex around an entire network-download-plus-processing function (rather than just the processing part) can make a "concurrent" program run no faster than a purely sequential one. Connect this to Amdahl's Law.
2. Walk through the 8-step canonical pattern of using a condition variable with a mutex (check condition → Wait → another goroutine updates state → Signal/Broadcast → reacquire → recheck). Why must releasing the mutex and suspending execution inside `Wait()` happen atomically?
3. Why does calling `Signal()` or `Broadcast()` without holding the associated mutex risk a "missed signal," and how does always signaling under the lock prevent this?
4. Describe write-starvation in a read-preferring readers–writer lock, and explain the specific mechanism (the `writersWaiting` counter) that the write-preferring implementation uses to prevent it.
5. How does a semaphore's `Release()`/`Acquire()` pair avoid the "missed signal" problem that plain condition variables have, and why does initializing a semaphore with `1 - size` permits let it function as a fixed-size waitgroup?
6. What two specific limitations of the semaphore-based waitgroup implementation motivate building a waitgroup with condition variables instead, and how does the condition-variable version solve each one?
7. Explain the conceptual difference between a waitgroup and a barrier — specifically, why a barrier's `Wait()` call is described as combining a waitgroup's `Done()` and `Wait()` into a single atomic operation.
8. In the concurrent matrix multiplication example, why does each row-goroutine call `barrier.Wait()` twice per cycle, and what would go wrong if one of those two calls were removed?

# Communication Using Message Passing & Channels — Chapters 7–9 Refresher

## Overview

These three chapters shift from memory-sharing concurrency (Ch. 1–6) to **message passing** — Go's implementation of ideas from Hoare's Communicating Sequential Processes (CSP). Chapter 7 introduces Go's channel primitive itself (creation, buffering, direction, closing, result-collection) and even builds a channel from scratch using semaphores. Chapter 8 covers the `select` statement for multiplexing operations across multiple channels (blocking, non-blocking, timeouts, disabling cases with `nil`), plus a framework for choosing memory sharing vs. message passing. Chapter 9 assembles these primitives into reusable concurrency patterns (quit channels, pipelines, fan-in/fan-out, broadcast, dynamic pipelines) that are idiomatic in real Go concurrent programs.

## Core Concepts

### Message Passing / Inter-Thread Communication (ITC) (Chapter 7)

**Definition:** An alternative to memory sharing for goroutine communication. Goroutines send/receive discrete messages instead of directly manipulating shared variables. Each goroutine works with its own isolated memory, greatly reducing race-condition risk since nothing is being concurrently modified in shared state.

**Note (side reference):** In distributed systems (multiple machines, no shared memory), message passing (e.g., over HTTP) is the *only* way applications can communicate — this is analogous to why Go favors channels for cross-goroutine communication even on a single machine.

---

### Go Channels — Creation & Basic Send/Receive (Chapter 7)

**Definition:** A channel is a typed conduit that lets two or more goroutines exchange messages. Conceptually a "direct line" between goroutines.

**Syntax:**
```go
ch := make(chan string)   // create an unbuffered channel of type string
ch <- "HELLO"             // send (channel on the left of <-)
msg := <-ch               // receive (channel on the right of <-)
```

**Example:**
```go
package main

import "fmt"

func receiver(messages chan string) {
    msg := ""
    for msg != "STOP" {
        msg = <-messages
        fmt.Println("Received:", msg)
    }
}

func main() {
    msgChannel := make(chan string)
    go receiver(msgChannel)
    fmt.Println("Sending HELLO...")
    msgChannel <- "HELLO"
    fmt.Println("Sending THERE...")
    msgChannel <- "THERE"
    fmt.Println("Sending STOP...")
    msgChannel <- "STOP"
}
```
**Important gotcha demonstrated:** The final `"STOP"` message is never printed by the receiver in the output, because `main()` sends it and then immediately terminates — and when the main goroutine ends, the whole process exits before the receiver goroutine gets a chance to print it.

---

### Channel Synchronicity (Blocking Behavior) (Chapter 7)

**Definition:** By default, Go channels are **synchronous** (unbuffered): a sender blocks until a receiver is ready to consume the message, and a receiver blocks until a sender provides a message.

**How it works internally:** If a sender pushes onto a channel and no goroutine is receiving, the sending goroutine suspends indefinitely (or until a receiver appears). Symmetric behavior applies to receivers with no sender.

**Example — blocked sender (no receiver ever consumes):**
```go
func receiver(messages chan string) {
    time.Sleep(5 * time.Second) // never actually reads from channel
    fmt.Println("Receiver slept for 5 seconds")
}
// main() sends "HELLO" and blocks forever waiting for a receiver
```
Output:
```
Sending HELLO...
Receiver slept for 5 seconds
fatal error: all goroutines are asleep - deadlock!
```

**Example — blocked receiver (no sender ever sends):**
```go
func sender(messages chan string) {
    time.Sleep(5 * time.Second) // never actually sends
}
// main() tries: msg := <-msgChannel and blocks forever
```
Same `deadlock!` fatal error results.

**Note:** Go's runtime has built-in **deadlock detection** — when it determines that *all* goroutines are permanently asleep with no possibility of resuming, it raises `fatal error: all goroutines are asleep - deadlock!` rather than hanging silently forever. (Deadlocks are explored more deeply in a later chapter, per the text.)

---

### Buffered Channels (Chapter 7)

**Definition:** A channel configured with a capacity, allowing it to store a limited number of messages before a sender blocks. As long as buffer space remains, a sender does not block; once full, further sends block until a receiver frees space by consuming a message.

**How it works internally:**
- Sender writes to a channel with available buffer space → message stored, sender doesn't block.
- Buffer fills up → sender blocks until a receiver consumes a message and frees a slot.
- Receiver reads from a non-empty buffer → doesn't block, even if the sender has stopped sending.
- Buffer empties with no sender producing more → receiver blocks.
- Messages are always delivered **in the order they were sent** (FIFO), even when there's a backlog.

**Syntax:**
```go
ch := make(chan int, 3)   // buffered channel, capacity 3
size := len(ch)           // check current number of buffered messages
```

**Example:**
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func receiver(messages chan int, wGroup *sync.WaitGroup) {
    msg := 0
    for msg != -1 {
        time.Sleep(1 * time.Second) // slow consumer: 1 msg/sec
        msg = <-messages
        fmt.Println("Received:", msg)
    }
    wGroup.Done()
}

func main() {
    msgChannel := make(chan int, 3) // buffer capacity of 3
    wGroup := sync.WaitGroup{}
    wGroup.Add(1)
    go receiver(msgChannel, &wGroup)
    for i := 1; i <= 6; i++ {
        size := len(msgChannel)
        fmt.Printf("%s Sending: %d. Buffer Size: %d\n",
            time.Now().Format("15:04:05"), i, size)
        msgChannel <- i
    }
    msgChannel <- -1
    wGroup.Wait()
}
```
**Observed behavior:** The fast sender fills the 3-slot buffer immediately (sends 1–4 rapidly, blocking on the 4th until space frees), then the sender and the slow (1 msg/sec) receiver proceed in lockstep as the buffer refills and drains.

---

### Channel Direction (Chapter 7)

**Definition:** Channels are **bidirectional** by default (usable for both sending and receiving). Go lets you restrict a channel parameter's direction in a function signature to **send-only** or **receive-only**, enforced at compile time.

**Syntax:**
```go
func receiver(messages <-chan int) { ... }  // receive-only channel
func sender(messages chan<- int)   { ... }  // send-only channel
```

**Example:**
```go
func receiver(messages <-chan int) {
    for {
        msg := <-messages
        fmt.Println(time.Now().Format("15:04:05"), "Received:", msg)
    }
}

func sender(messages chan<- int) {
    for i := 1; ; i++ {
        fmt.Println(time.Now().Format("15:04:05"), "Sending:", i)
        messages <- i
        time.Sleep(1 * time.Second)
    }
}

func main() {
    msgChannel := make(chan int)
    go receiver(msgChannel)
    go sender(msgChannel)
    time.Sleep(5 * time.Second)
}
```
**Compile-time enforcement:** Attempting `messages <- 99` inside `receiver()` (a receive-only channel) fails to compile:
```
invalid operation: cannot send to receive-only channel messages (variable of type <-chan int)
```

---

### Closing Channels (Chapter 7)

**Definition:** Instead of relying on a special **sentinel value** (a.k.a. **poison pill** in distributed-systems terminology) sent through the channel to signal "no more data," Go allows explicitly **closing** a channel.

**Syntax:**
```go
close(ch)
```

**Semantics:**
- After closing, you must **not** send any more messages on the channel — doing so raises a runtime error/panic.
- Reading from a **closed** channel never blocks — it immediately returns the **zero value** for the channel's type (e.g., `0` for `int`, `""` for `string`) forever.
- Reading returns a second boolean value indicating whether the channel is still open:
```go
msg, more := <-messages   // `more` is false once the channel is closed and drained
```

**Example — problem with using default zero value alone:**
```go
func receiver(messages <-chan int) {
    for {
        msg := <-messages
        fmt.Println(time.Now().Format("15:04:05"), "Received:", msg)
        time.Sleep(1 * time.Second)
    }
}
```
Output after the sender sends 1, 2, 3 and then closes the channel:
```
17:19:50 Received: 1
17:19:51 Received: 2
17:19:52 Received: 3
17:19:53 Received: 0
17:19:54 Received: 0
17:19:55 Received: 0    // keeps reading zero values forever
```
**Why this is problematic:** The zero value returned by a closed channel might be a legitimately meaningful value in your domain (e.g., a temperature of `0`), making it ambiguous whether `0` means "closed" or "actual reading of zero."

**Fix — using the open-flag:**
```go
func receiver(messages <-chan int) {
    for {
        msg, more := <-messages
        fmt.Println(time.Now().Format("15:04:05"), "Received:", msg, more)
        time.Sleep(1 * time.Second)
        if !more {
            return
        }
    }
}
```

**Cleaner idiom — `for range` over a channel:**
```go
func receiver(messages <-chan int) {
    for msg := range messages {   // automatically stops when channel closes
        fmt.Println(time.Now().Format("15:04:05"), "Received:", msg)
        time.Sleep(1 * time.Second)
    }
    fmt.Println("Receiver finished.")
}
```
This is the idiomatic way to "consume everything until the sender is done."

---

### Receiving Function Results via Channels (Chapter 7)

**Definition:** A pattern where a function is run concurrently (often as an anonymous goroutine), and its return value is delivered later via a channel instead of a normal function return, letting the caller do other work in the meantime.

**Example:**
```go
func findFactors(number int) []int {
    result := make([]int, 0)
    for i := 1; i <= number; i++ {
        if number%i == 0 {
            result = append(result, i)
        }
    }
    return result
}

func main() {
    resultCh := make(chan []int)
    go func() {
        resultCh <- findFactors(3419110721) // runs concurrently, sends result when done
    }()
    fmt.Println(findFactors(4033836233))   // runs on main goroutine at the same time
    fmt.Println(<-resultCh)                // blocks here until the first call finishes
}
```
**Behavior:** If the first `findFactors()` call (running in the anonymous goroutine) isn't finished by the time `main()` tries to read `<-resultCh`, `main()` simply blocks until the result arrives — an easier alternative to manually managing a shared variable + waitgroup.

---

### Building a Channel From Scratch (Chapter 7)

**Definition:** A demonstration of channel internals: a buffered channel behaves like a thread-safe fixed-size FIFO queue that blocks receivers when empty and blocks senders when full. The book reconstructs this using: a **queue** (linked list), a **mutex** (protects the queue), and **two semaphores** — a *capacity semaphore* (blocks sender when full) and a *buffer-size semaphore* (blocks receiver when empty).

**How it works internally — components:**
```go
type Channel[M any] struct {
    capacitySema *listing5_16.Semaphore // permits = free buffer slots remaining
    sizeSema     *listing5_16.Semaphore // permits = messages currently in buffer
    mutex        sync.Mutex             // protects the shared buffer
    buffer       *list.List             // linked-list-backed FIFO queue
}

func NewChannel[M any](capacity int) *Channel[M] {
    return &Channel[M]{
        capacitySema: listing5_16.NewSemaphore(capacity), // starts full of permits
        sizeSema:     listing5_16.NewSemaphore(0),         // starts with 0 permits
        buffer:       list.New(),
    }
}
```

**`Send()` — 3 steps:**
1. Acquire a permit from the **capacity semaphore** (blocks if buffer is full — 0 permits left).
2. Push the message onto the buffer, protected by the mutex.
3. Release a permit on the **size semaphore** (wakes a blocked receiver, if any).

```go
func (c *Channel[M]) Send(message M) {
    c.capacitySema.Acquire()
    c.mutex.Lock()
    c.buffer.PushBack(message)
    c.mutex.Unlock()
    c.sizeSema.Release()
}
```

**`Receive()` — 3 steps:**
1. Release a permit on the **capacity semaphore** (wakes a blocked sender, if any, giving it a free slot).
2. Acquire a permit from the **size semaphore** (blocks if buffer is empty — 0 permits left).
3. Pop the front message from the buffer, protected by the mutex.

```go
func (c *Channel[M]) Receive() M {
    c.capacitySema.Release()
    c.sizeSema.Acquire()
    c.mutex.Lock()
    v := c.buffer.Remove(c.buffer.Front()).(M)
    c.mutex.Unlock()
    return v
}
```

**Important design note:** Releasing the capacity semaphore permit *first* in `Receive()` (before acquiring the size semaphore) is deliberate — it makes the implementation also correctly support **zero-capacity (synchronous) channels**, where sender and receiver must "rendezvous" and wait for each other directly.

**Side note — actual Go implementation differs:** Go's real channel implementation (in `runtime/chan.go`) does **not** use a two-semaphore design. Instead it integrates directly with the scheduler using two linked lists (one of suspended senders, one of suspended receivers) plus a buffer. This gives **fairness**: the first goroutine suspended is the first one resumed — a guarantee the book's simplified two-semaphore version does not necessarily provide.

---

### The `select` Statement — Reading From Multiple Channels (Chapter 8)

**Definition:** `select` lets a goroutine wait on multiple channel operations simultaneously, blocking until *any* one of them is ready, then executing that case's code.

**Syntax:**
```go
select {
case msg1 := <-channelA:
    // handle msg1
case msg2 := <-channelB:
    // handle msg2
}
```

**Example:**
```go
func writeEvery(msg string, seconds time.Duration) <-chan string {
    messages := make(chan string)
    go func() {
        for {
            time.Sleep(seconds)
            messages <- msg
        }
    }()
    return messages
}

func main() {
    messagesFromA := writeEvery("Tick", 1*time.Second)
    messagesFromB := writeEvery("Tock", 3*time.Second)
    for {
        select {
        case msg1 := <-messagesFromA:
            fmt.Println(msg1)
        case msg2 := <-messagesFromB:
            fmt.Println(msg2)
        }
    }
}
```

**Important note on tie-breaking:** *If multiple `select` cases are simultaneously ready, Go picks one at random.* Code must never depend on a particular case being preferred/ordered.

**Note — Channels as first-class objects:** Because Go channels can be returned from functions, stored in variables, and passed as arguments, functions like `writeEvery()` above can be composed together as reusable building blocks — a recurring theme throughout Ch. 9.

---

### `select` with a `default` Case — Non-Blocking Operations (Chapter 8)

**Definition:** Adding a `default` case to `select` executes that case's code immediately if none of the other channel operations are currently ready, avoiding blocking entirely. Directly analogous to a non-blocking `tryLock()` on a mutex.

**Syntax:**
```go
select {
case msg := <-messages:
    // handle message
default:
    // executed immediately if no message is ready
}
```

**Example:**
```go
func main() {
    messages := sendMsgAfter(3 * time.Second)
    for {
        select {
        case msg := <-messages:
            fmt.Println("Message received:", msg)
            return
        default:
            fmt.Println("No messages waiting")
            time.Sleep(1 * time.Second)
        }
    }
}
```
Output:
```
No messages waiting
No messages waiting
No messages waiting
Message received: Hello
```

---

### Side Note: Historical Origins of `select` (Chapter 8)

- UNIX's `select()` system call accepts a set of file descriptors and blocks until any becomes ready for I/O — useful for monitoring multiple files/sockets from one thread.
- Go's `select` statement takes its name from the **Newsqueak** programming language's `select` command. Newsqueak (unrelated to Orwell's fictional "Newspeak") also derived its concurrency model from Hoare's CSP.
- It's speculated (but not confirmed) that Newsqueak's `select` was itself named after the UNIX `select()` syscall, which was built for multiplexed I/O on the Blit graphics terminal in 1983.
- Functionally, UNIX's `select()` and Go's `select` are analogous: both multiplex several blocking operations into one execution point.

---

### Brute-Force Concurrent Computation with `default` Case (Chapter 8)

**Definition:** A pattern combining the `default` case (to perform ongoing computational work) with a separate `case` watching a shared "stop" channel — enabling many goroutines to do useful work in a tight loop while remaining instantly responsive to a stop signal.

**How it's used:** `close()` on a channel acts as a **broadcast signal** to all consumers simultaneously — every goroutine selecting on that channel unblocks the moment it's closed. This is explicitly noted as a general technique: *"We can use the `close()` operation on a channel to act like a signal being broadcast to all consumers."*

**Example (password brute-force across a divided search space):**
```go
func guessPassword(from int, upto int, stop chan int, result chan string) {
    for guessN := from; guessN < upto; guessN += 1 {
        select {
        case <-stop:
            fmt.Printf("Stopped at %d [%d,%d)\n", guessN, from, upto)
            return
        default:
            if toBase27(guessN) == passwordToGuess {
                result <- toBase27(guessN)
                close(stop) // broadcasts "stop" to every other goroutine's select
                return
            }
        }
    }
    fmt.Printf("Not found between [%d,%d)\n", from, upto)
}

func main() {
    finished := make(chan int)
    passwordFound := make(chan string)
    for i := 1; i <= 387_420_488; i += 10_000_000 {
        go guessPassword(i, i+10_000_000, finished, passwordFound)
    }
    fmt.Println("password found:", <-passwordFound)
    close(passwordFound)
    time.Sleep(5 * time.Second)
}
```

---

### Timing Out on Channels (Chapter 8)

**Definition:** Using `select` combined with `time.After(duration)` to block on a channel operation only up to a maximum wait time, after which the timeout branch fires instead.

**Syntax:**
```go
select {
case msg := <-messages:
    // handle message
case t := <-time.After(timeoutDuration):
    // timeout fired; t is the time the timeout message was sent
}
```

**Mechanism:** `time.After(duration)` returns a channel on which a single message (the current time) is sent after `duration` elapses — implemented internally via `time.Timer`, so you don't need to build your own timer goroutine.

**Example:**
```go
func main() {
    t, _ := strconv.Atoi(os.Args[1])
    messages := sendMsgAfter(3 * time.Second)
    timeoutDuration := time.Duration(t) * time.Second
    select {
    case msg := <-messages:
        fmt.Println("Message received:", msg)
    case tNow := <-time.After(timeoutDuration):
        fmt.Println("Timed out. Waited until:", tNow.Format("15:04:05"))
    }
}
```
**Use case cited:** Financial trading applications — raise an alert if a stock price update doesn't arrive within a time window.

---

### Writing to Channels with `select` (Chapter 8)

**Definition:** `select` cases aren't limited to receive operations — a case can be a **send** operation, and it fires only when that send would succeed without blocking (i.e., a receiver or buffer slot is ready).

**Example (feeding random numbers into a prime filter while collecting results in the same loop):**
```go
func primesOnly(inputs <-chan int) <-chan int {
    results := make(chan int)
    go func() {
        for c := range inputs {
            isPrime := c != 1
            for i := 2; i <= int(math.Sqrt(float64(c))); i++ {
                if c%i == 0 {
                    isPrime = false
                    break
                }
            }
            if isPrime {
                results <- c
            }
        }
    }()
    return results
}

func main() {
    numbersChannel := make(chan int)
    primes := primesOnly(numbersChannel)
    for i := 0; i < 100; {
        select {
        case numbersChannel <- rand.Intn(1000000000) + 1: // send case
        case p := <-primes:                                // receive case
            fmt.Println("Found prime:", p)
            i++
        }
    }
}
```

---

### Disabling `select` Cases with `nil` Channels (Chapter 8)

**Definition:** A `nil`-valued channel **always blocks** on both send and receive. This isn't useless — it's a deliberate technique to **disable** a specific `select` case (e.g., once its underlying data source has closed), so the `select` loop effectively continues operating on only the remaining live channels.

**Basic behavior demonstration:**
```go
var ch chan string = nil
ch <- "message" // blocks forever
fmt.Println("This is never printed")
```
Result: Go's deadlock detector fires:
```
fatal error: all goroutines are asleep - deadlock!
goroutine 1 [chan send (nil chan)]:
```

**Critical warning (explicit in text):** *"When we use a select case on a closed channel, that case will always execute"* — because reading from a **closed** channel never blocks (it returns the zero value immediately). If you don't handle this, a `select` loop combining a closed channel with a still-open one will busy-loop, endlessly executing the closed-channel case and receiving zero values.

**Fix — reassign the channel variable to `nil` once it's detected closed:**
```go
func main() {
    sales := generateAmounts(50)
    expenses := generateAmounts(40)
    endOfDayAmount := 0
    for sales != nil || expenses != nil {
        select {
        case sale, moreData := <-sales:
            if moreData {
                fmt.Println("Sale of:", sale)
                endOfDayAmount += sale
            } else {
                sales = nil // disables this case going forward
            }
        case expense, moreData := <-expenses:
            if moreData {
                fmt.Println("Expense of:", expense)
                endOfDayAmount -= expense
            } else {
                expenses = nil // disables this case going forward
            }
        }
    }
    fmt.Println("End of day profit and loss:", endOfDayAmount)
}
```
**Effect:** Once a source channel closes, setting it to `nil` disables that `select` case permanently (since `nil` channels always block, they're never "ready"), letting the loop naturally drain whichever channel(s) remain open. The loop condition `sales != nil || expenses != nil` exits once both sources are exhausted.

**Note:** This pattern of merging two (or more) fixed data sources into one consumption point is a form of the **fan-in pattern** (formally introduced in Ch. 9) — but this `select`-based version only works for a **fixed, known number of sources**. Chapter 9 covers a fan-in variant that supports a *dynamic* number of sources.

---

### Choosing Between Message Passing and Memory Sharing (Chapter 8)

**Definition:** A decision framework — neither approach is universally superior; the choice depends on code-simplicity goals, coupling concerns, memory constraints, and communication volume/size.

**Factor 1 — Code simplicity:** Message-passing code tends to have well-defined modules with clear input/output channels, making data flow easier to follow. Memory-sharing code (mutexes, semaphores scattered through critical sections) is more low-level, verbose, and harder to trace data flow through.

**Factor 2 — Coupling:**
- **Tightly coupled** software: changing one component ripples into many others.
- **Loosely coupled** software: components have clear boundaries and few dependencies; changes stay local.
- Memory sharing tends to produce **more tightly coupled** systems, because any execution can read/write the same shared memory with no clear contract.
- Message passing makes it **easier** (not automatic/guaranteed) to build loosely coupled systems, because channels act as explicit input/output contracts. Text explicitly notes: *"This is not to say that all code that uses message passing is loosely coupled. Nor is all software that uses memory sharing tightly coupled."*

**Factor 3 — Memory consumption:** Message passing tends to consume **more memory**, since each goroutine keeps its own isolated copy/state (e.g., each downloader building its own local frequency slice instead of sharing one slice) — results are only merged afterward.

**Example contrast (letter-frequency app, from Ch. 3, rewritten as message passing):**
```go
func countLetters(url string) <-chan []int {
    result := make(chan []int)
    go func() {
        defer close(result)
        frequency := make([]int, 26) // LOCAL slice, not shared
        resp, _ := http.Get(url)
        defer resp.Body.Close()
        if resp.StatusCode != 200 {
            panic("Server returning error code: " + resp.Status)
        }
        body, _ := io.ReadAll(resp.Body)
        for _, b := range body {
            c := strings.ToLower(string(b))
            cIndex := strings.Index(allLetters, c)
            if cIndex >= 0 {
                frequency[cIndex] += 1
            }
        }
        result <- frequency
    }()
    return result
}

func main() {
    results := make([]<-chan []int, 0)
    totalFrequencies := make([]int, 26)
    for i := 1000; i <= 1030; i++ {
        url := fmt.Sprintf("https://rfc-editor.org/rfc/rfc%d.txt", i)
        results = append(results, countLetters(url))
    }
    for _, c := range results {
        frequencyResult := <-c
        for i := 0; i < 26; i++ {
            totalFrequencies[i] += frequencyResult[i]
        }
    }
    for i, c := range allLetters {
        fmt.Printf("%c-%d ", c, totalFrequencies[i])
    }
}
```
No mutex needed (each goroutine only touches its own slice), but memory usage grows with the number of concurrent goroutines/slices — acceptable here since each slice is tiny (size 26), but potentially costly for large payloads.

**Factor 4 — Communication efficiency / message size and volume:**
- Message passing copies data on every send — expensive for **large messages** (e.g., images/video). Memory sharing avoids the copy cost entirely.
- Message passing also degrades when goroutines are extremely **"chatty"** — needing to exchange a huge number of messages with each other repeatedly (example given: a grid-based weather-forecasting simulation where every cell's goroutine must exchange partial results with every other cell's goroutine on every iteration). In such cases, memory sharing (e.g., a shared 2D array with reader–writer locks) is likely more efficient.

---

### Communicating Sequential Processes (CSP) — Deeper Dive (Chapter 9)

**Definition:** A formal concurrency model proposed by C.A.R. Hoare (1978), based on message passing between isolated sequential processes via **named, unbuffered channels**, rather than shared memory.

**Key distinction (explicit in text):** A "CSP process" is **not** the same as an OS process (Chapter 2) — it's simply an independent sequential execution with its own isolated internal state, communicating only via channels.

**Why CSP/message-passing exists as an alternative to the SRC model:**
- Memory-sharing concurrency (mutexes, semaphores, condition variables) is called, per the text, the **SRC model**, named after Andrew D. Birrell's 1989 paper *"An Introduction to Programming with Threads"* (Systems Research Center).
- SRC-style code is criticized as too low-level: scheduling is non-deterministic, and combined with shared memory this creates race-condition risk that requires careful manual synchronization.
- In critical domains (health, infrastructure), this non-determinism makes correctness hard to *prove*.

**Immutability as a partial fix:** If shared data is **never modified** after creation, there's no race-condition risk (races require concurrent writes to the same location). When an update is needed, create a **new copy** with the changes instead of mutating in place. This raises the question of *how to distribute* the new copy — which is exactly what message passing/channels solve.

**Go's CSP implementation vs. the original model:**
- Like CSP, Go's channels are **synchronized and unbuffered by default**.
- **Key Go innovation over pure CSP:** channels are **first-class objects** — they can be stored in variables, passed as function arguments/return values, or even sent over other channels. This allows *dynamic* topologies (channels created/destroyed at runtime) rather than CSP's originally *static* topology of fixed, pre-wired processes.

**Side note — CSP in other languages:**
- **Erlang:** processes communicate by sending messages, but there's no channel concept, and messages are **not synchronous**.
- **Java/Scala (Akka framework):** uses the **Actor model** — units called "actors" with isolated memory pass messages to each other; again, no channels, and message passing is not synchronous.

---

## Patterns & Idioms

### Quitting Channels Pattern (Chapter 9)
**When to use:** When a goroutine consumes from more than one channel and needs a clean, explicit way to be told "stop now," rather than relying only on `close()` of its primary input channel.
**Why:** Using a dedicated `quit` channel with `select` lets any part of a system broadcast a stop signal, decoupled from the data-flow channels themselves. The data type of the quit channel is irrelevant since only the close signal matters.
```go
func printNumbers(numbers <-chan int, quit chan int) {
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-numbers)
        }
        close(quit)
    }()
}

func main() {
    numbers := make(chan int)
    quit := make(chan int)
    printNumbers(numbers, quit)
    next := 0
    for i := 1; ; i++ {
        next += i
        select {
        case numbers <- next:
        case <-quit:
            fmt.Println("Quitting number generation")
            return
        }
    }
}
```

### Pipelining Pattern (Chapter 9)
**When to use:** Chaining multiple processing stages together, where each stage is a goroutine that accepts an input channel and returns an output channel.
**Why:** Creates a uniform, composable interface — any stage can be swapped, extended, or reordered because they all share the same "accept input chan(s), return output chan" shape. Data flows through the pipeline without shared memory between stages.
```go
func generateUrls(quit <-chan int) <-chan string { /* returns chan of URLs */ }
func downloadPages(quit <-chan int, urls <-chan string) <-chan string { /* returns chan of page bodies */ }
func extractWords(quit <-chan int, pages <-chan string) <-chan string { /* returns chan of words */ }

func main() {
    quit := make(chan int)
    defer close(quit)
    results := extractWords(quit, downloadPages(quit, generateUrls(quit)))
    for result := range results {
        fmt.Println(result)
    }
}
```
**Limitation observed:** In this simple chained form, downloads happen **sequentially** (one page fully processed before the next starts) — motivating the fan-out pattern below for concurrency within a stage.

### Fan-Out Pattern (Chapter 9)
**Definition:** Multiple goroutines read from the **same** input channel, load-balancing the work among themselves — analogous to several baristas serving from one shared queue.
**When to use:** When you want to parallelize processing of a stream of work items and **don't care about output order**.
```go
const downloaders = 20

pages := make([]<-chan string, downloaders)
for i := 0; i < downloaders; i++ {
    pages[i] = downloadPages(quit, urls) // 20 goroutines share the same `urls` input channel
}
```
**Caveat (explicit):** *"The fan-out pattern makes sense only if we don't care about the order of the incoming messages"* — since concurrent, non-deterministic scheduling means items complete in unpredictable order.

### Fan-In Pattern (Chapter 9)
**Definition:** Merging the contents of multiple channels into a single output channel.
**When to use:** After fanning out (many goroutines each with their own output channel), to feed a single downstream consumer that expects one input channel.
**Why the close-timing problem exists:** With many-to-one merging, you can't simply close the output when *any one* source closes — another source might still be producing. Solution: use a `sync.WaitGroup`, one increment per source goroutine, and only close the merged output once **all** sources have signaled completion.
```go
func FanIn[K any](quit <-chan int, allChannels ...<-chan K) chan K {
    wg := sync.WaitGroup{}
    wg.Add(len(allChannels))
    output := make(chan K)
    for _, c := range allChannels {
        go func(channel <-chan K) {
            defer wg.Done()
            for i := range channel {
                select {
                case output <- i:
                case <-quit:
                    return
                }
            }
        }(c)
    }
    go func() {
        wg.Wait()
        close(output)
    }()
    return output
}
```
**Note:** This example uses **Go generics** (`[K any]`) so the same fan-in implementation works across any channel element type.

### Flushing Results on Close (Chapter 9)
**When to use:** When a stage needs to accumulate *all* incoming data before producing any output (e.g., computing a top-N result), rather than streaming output continuously.
**Why:** The stage watches for the input channel to close (via the `moreData` flag pattern), and only *then* performs a final computation (e.g., sorting) and emits a single summary result.
```go
func longestWords(quit <-chan int, words <-chan string) <-chan string {
    longWords := make(chan string)
    go func() {
        defer close(longWords)
        uniqueWordsMap := make(map[string]bool)
        uniqueWords := make([]string, 0)
        moreData, word := true, ""
        for moreData {
            select {
            case word, moreData = <-words:
                if moreData && !uniqueWordsMap[word] {
                    uniqueWordsMap[word] = true
                    uniqueWords = append(uniqueWords, word)
                }
            case <-quit:
                return
            }
        }
        sort.Slice(uniqueWords, func(a, b int) bool {
            return len(uniqueWords[a]) > len(uniqueWords[b])
        })
        longWords <- strings.Join(uniqueWords[:10], ", ")
    }()
    return longWords
}
```
**Note on safety:** Since the internal map/slice here is only ever touched by this one goroutine (never shared), there's no race-condition risk despite looking like mutable shared state.

### Broadcast Pattern (Chapter 9)
**Definition:** Replicates every message from one input channel out to **multiple** output channels (as opposed to fan-out, which *splits/load-balances* messages across consumers). Each consumer sees **every** message, not a subset.
**When to use:** When multiple independent downstream computations (e.g., `longestWords()` and `frequentWords()`) both need the complete stream of the same data.
```go
func Broadcast[K any](quit <-chan int, input <-chan K, n int) []chan K {
    outputs := CreateAll[K](n)
    go func() {
        defer CloseAll(outputs...)
        var msg K
        moreData := true
        for moreData {
            select {
            case msg, moreData = <-input:
                if moreData {
                    for _, output := range outputs {
                        output <- msg
                    }
                }
            case <-quit:
                return
            }
        }
    }()
    return outputs
}
```
**Important caveat (explicit):** *"A slow consumer from this broadcast implementation would slow all consumers to the same rate"* — because the broadcast only reads its next input message after successfully writing the current message to **every** output channel; a single slow/blocked receiver stalls the whole broadcast.

### Closing Channels After a Condition — `Take(n)` (Chapter 9)
**When to use:** To terminate part of a pipeline early once a certain number of items have been processed (e.g., stop after the first 10,000 words), without stopping the *entire* application.
**Why:** By using a **separate quit channel** scoped only to the upstream portion of the pipeline (before the `Take(n)` stage), you can shut down just the producers feeding that stage while letting downstream consumers finish processing what's already flowed through.
```go
func Take[K any](quit chan int, n int, input <-chan K) <-chan K {
    output := make(chan K)
    go func() {
        defer close(output)
        moreData := true
        var msg K
        for n > 0 && moreData {
            select {
            case msg, moreData = <-input:
                if moreData {
                    output <- msg
                    n--
                }
            case <-quit:
                return
            }
        }
        if n == 0 {
            close(quit) // signals upstream producers to stop
        }
    }()
    return output
}
```
**Wiring:** Uses **two separate quit channels** — `quitWords` for everything before `Take(n)` (generators, downloaders, word extractor), and a separate `quit` for everything after (broadcast, longest/frequent-word finders) — so that stopping the upstream doesn't prematurely kill the downstream still processing buffered data.

### Dynamic Pipeline via First-Class Channels — Prime Sieve (Chapter 9)
**When to use:** When the pipeline's *shape* (number of stages) needs to grow based on runtime data, not just its data flow.
**Why it matters:** This is presented as Go's concrete improvement over Hoare's original CSP paper, which described a **static** linear pipeline (fixed max prime search range). Because Go channels are first-class objects that can be created and wired up dynamically inside running goroutines, the pipeline can **extend itself** as new primes are discovered.
```go
func primeMultipleFilter(numbers <-chan int, quit chan<- int) {
    var right chan int
    p := <-numbers          // first number received becomes this filter's prime
    fmt.Println(p)
    for n := range numbers {
        if n%p != 0 {        // not a multiple of p -> could be prime or pass further right
            if right == nil {
                right = make(chan int)
                go primeMultipleFilter(right, quit) // dynamically spawn next stage
            }
            right <- n
        }
    }
    if right == nil {
        close(quit)
    } else {
        close(right)
    }
}

func main() {
    numbers := make(chan int)
    quit := make(chan int)
    go primeMultipleFilter(numbers, quit)
    for i := 2; i < 100000; i++ {
        numbers <- i
    }
    close(numbers)
    <-quit
}
```
**Attribution note:** The pipeline-based prime-sieve algorithm (sieve of Eratosthenes via a chain of filters) is attributed to mathematician/programmer **Douglas McIlroy**, though it appears in Hoare's CSP paper.
**Simplification noted in the text:** For clarity, the example checks divisibility against *all* primes less than `c` rather than only up to `√c` (the mathematically sufficient check), keeping the listing shorter.

---

## Warnings, Pitfalls & Gotchas

- **Sending on a channel with no receiver blocks forever** (or until the runtime detects deadlock and panics) — Go's channels are synchronous by default.
- **Receiving from a channel with no sender blocks forever**, symmetric to the above.
- **The last message sent right before `main()` returns may never be observed/printed**, because the whole process terminates the instant the main goroutine finishes, regardless of other goroutines' progress.
- **Sending on a closed channel raises an error/panics.** Once `close(ch)` is called, no further sends are legal.
- **Reading from a closed channel never blocks and returns the zero value** — this is easy to mistake for legitimate data (e.g., an actual `0` reading) unless you check the second boolean return value (`msg, more := <-ch`).
- **A `select` case reading from an already-closed channel will *always* be immediately ready**, causing a busy-loop that endlessly executes that case with zero values if not handled — the fix is to detect closure and set that channel variable to `nil` to disable the case.
- **`nil` channels block forever on both send and receive** — accidentally using an uninitialized/nil channel variable will silently deadlock (though Go's deadlock detector will usually catch and report it if *all* goroutines are stuck).
- **`select` picks randomly among simultaneously-ready cases** — never write code that assumes a particular case will be preferred or that cases are checked in listed order.
- **Passing pointers on channels re-introduces shared-memory risk.** The text's guideline: pass **copies** of data on channels wherever possible; if a pointer must be passed, either treat the referenced data as immutable, or ensure the sender never touches it again after sending.
- **Mixing message passing and memory sharing in the same application is discouraged** — it creates confusion about which paradigm governs which part of the system, undermining the safety benefits of each.
- **Message passing can silently bloat memory usage** since each goroutine holds isolated copies of data (no sharing) — fine for small payloads (e.g., a 26-element frequency slice), potentially costly for large ones.
- **Message passing is a poor fit for very large payloads** (e.g., images, video frames) since every send **copies** the data — memory sharing avoids this overhead.
- **Message passing degrades badly under high message *volume* / "chatty" communication patterns** — e.g., algorithms where every worker must exchange partial results with every other worker on every iteration (like grid-based weather simulation) can end up dominated by message-copying overhead; memory sharing with fine-grained locks (e.g., reader–writer locks) may be more appropriate.
- **Fan-out only makes sense when message order doesn't matter** — concurrent, non-deterministic processing means completion order across fanned-out goroutines is unpredictable.
- **A naive many-to-one fan-in that closes its output as soon as any one source closes risks premature closure** while other sources are still producing — must wait for *all* sources (e.g., via a `sync.WaitGroup`) before closing the merged output.
- **A broadcast implementation that writes to all outputs before reading the next input will have its overall throughput capped by its slowest consumer** — one blocked/slow receiver stalls delivery to every other receiver.
- **Building a `Take(n)`-style early-termination stage requires care with quit-channel scoping** — using a single shared quit channel for the whole pipeline can prematurely kill downstream stages that still need to process already-buffered data; separate quit channels for upstream vs. downstream segments avoid this.

---

## Important Side Notes

- **Distributed systems and message passing:** When applications run on separate machines with no shared memory, message passing (e.g., via HTTP) is the *only* communication option — Go's channel-based model mirrors this at the single-machine, cross-goroutine level.
- **`time.Timer` / `time.After`:** Provided by the standard library so developers don't need to hand-roll their own timeout/timer goroutines for use with `select`.
- **The SRC model naming:** Comes from Andrew D. Birrell's 1989 paper *"An Introduction to Programming with Threads"* (Systems Research Center) — the classic memory-sharing-with-primitives approach to concurrency.
- **CSP paper source:** C.A.R. Hoare, *"Communicating Sequential Processes,"* 1978 (available at https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf) — origin of the formal model Go's concurrency design borrows from.
- **Languages/frameworks influenced by CSP:** Erlang, Occam, Go, Scala's Akka framework, Clojure's `core.async`, among others — though each implements the ideas differently (e.g., Erlang and Akka lack the "channel" concept and aren't synchronous).
- **Go's key departure from classic CSP:** treating channels as **first-class objects** (storable, passable, sendable-on-other-channels) — this single feature is what enables *dynamic* pipeline topologies (e.g., the growing prime-sieve pipeline) instead of CSP's originally static, fixed-size process networks.
- **Historical/mathematical attribution:** The concurrent prime-sieve-via-pipeline algorithm is attributed to Douglas McIlroy, even though it's presented in Hoare's CSP paper.
- **`go build` vs. `go run` compile-time errors:** Channel-direction violations (e.g., sending on a receive-only channel) are caught by the Go **compiler**, not at runtime — demonstrated via `go build directional.go` producing a compile error.
- **RFC documents reused as test data:** Chapters 3, 4, and 9 all reuse the same static, plain-text RFC documents from rfc-editor.org for concurrency demonstrations (letter frequency, word frequency, longest/most-frequent words) — chosen for their predictable URLs and unchanging content.
- **Generics usage:** Several Ch. 9 utilities (`FanIn[K any]`, `Broadcast[K any]`, `Take[K any]`, exercise signatures like `TakeUntil[K any]`, `Print[T any]`, `Drain[T any]`) use Go generics so the same pipeline-stage function can be reused across different channel element types without duplicating code.

---

## Quick Reference Table

| Construct / API | Purpose | Example |
|---|---|---|
| `make(chan T)` | Create an unbuffered (synchronous) channel of type `T` | `ch := make(chan string)` |
| `make(chan T, n)` | Create a buffered channel with capacity `n` | `ch := make(chan int, 3)` |
| `ch <- v` | Send value `v` on channel `ch` | `msgChannel <- "HELLO"` |
| `v := <-ch` | Receive a value from channel `ch` | `msg := <-messages` |
| `v, ok := <-ch` | Receive plus an "is channel still open" flag | `msg, more := <-messages` |
| `close(ch)` | Close a channel; further sends panic, reads return zero value | `close(msgChannel)` |
| `for v := range ch` | Consume all messages until the channel closes | `for msg := range messages { ... }` |
| `chan<- T` | Send-only channel type (function parameter) | `func sender(m chan<- int)` |
| `<-chan T` | Receive-only channel type (function parameter) | `func receiver(m <-chan int)` |
| `len(ch)` | Number of currently buffered messages | `size := len(msgChannel)` |
| `select { case ...: }` | Block until one of several channel ops is ready | `select { case m := <-a: ... }` |
| `select` + `default` | Non-blocking channel attempt | `select { case m := <-ch: ...; default: ... }` |
| `time.After(d)` | Channel that receives a value after duration `d` | `case t := <-time.After(3*time.Second):` |
| `nil` channel | Always blocks on send/receive — used to disable a `select` case | `sales = nil // disables case` |
| Fan-out | Multiple goroutines share one input channel | `for i:=0;i<20;i++ { go worker(sharedInput) }` |
| Fan-in | Merge multiple channels into one, close after all done via `WaitGroup` | `FanIn(quit, ch1, ch2, ch3)` |
| Broadcast | Replicate one input to N output channels | `Broadcast(quit, input, 2)` |

---

## Self-Check Questions

1. Explain why the final "STOP" message is missing from the receiver's output in the basic message-passing example (Listing 7.1/7.2), and how this relates to how `main()` terminates a Go program.
2. What's the difference in blocking behavior between an unbuffered channel and a buffered channel with 3 slots of capacity? At what points does each block a sender vs. a receiver?
3. Why is reading the raw zero-value from a closed channel potentially dangerous, and what two mechanisms does Go provide to distinguish "closed" from "legitimately zero"?
4. Walk through why a `select` case on an already-closed channel will always fire, and explain the `nil`-channel technique used to prevent this from causing a busy loop.
5. Describe the difference between the fan-out and fan-in patterns, and explain why a naive fan-in implementation risks closing its merged output channel too early.
6. Under what conditions (message size, message volume, coupling needs) would you prefer memory sharing over message passing, and vice versa? Give a concrete example scenario for each.
7. How does treating channels as "first-class objects" in Go allow the prime-sieve pipeline to grow dynamically, compared to the original static CSP model described in Hoare's 1978 paper?
8. In the broadcast pattern implementation, why does a single slow consumer end up throttling all the other consumers, and how might you address that limitation?

# Concurrency Patterns, Deadlocks, and Low-Level Synchronization — Chapters 10–12 Refresher

## Overview

These final three chapters move from "how to synchronize" to "how to design and reason about" concurrent systems. Chapter 10 covers decomposition strategies (task vs. data, granularity) and four reusable implementation patterns (loop-level parallelism, fork/join, worker pools, pipelining). Chapter 11 tackles deadlocks: how to identify them via resource allocation graphs, and three strategies to handle them (detection, avoidance via algorithms like the banker's algorithm, and prevention via lock ordering) — including deadlocks specific to channels. Chapter 12 goes underneath all the higher-level primitives to show how mutexes are actually built, from atomic variables up through spin locks, futexes, and finally Go's real production mutex implementation.

## Core Concepts

### Decomposition (Chapter 10)

**Definition:** Decomposition is the process of subdividing a program into tasks and identifying which of these tasks are largely independent — i.e., can run concurrently without affecting each other.

**Task dependency graphs:** A tool for visualizing which steps can run in parallel, using the flat-tire-changing analogy — nodes are tasks, and a directed edge from A to B means "B depends on A." Reading such a graph tells you which tasks are free to be parallelized (e.g., unloading a spare tire vs. loosening wheel nuts) and which must remain sequential.

---

### Task Decomposition (Chapter 10)

**Definition:** Breaking a program down by asking, *"What are the different parallel **actions** we can perform to accomplish the job more quickly?"* — the pilots-splitting-cockpit-duties analogy: both pilots see the same instrument data but perform different actions concurrently.

**Example (from Ch. 9's longest-words pipeline):** The three tasks — downloading web pages, extracting words, finding the longest words — form a task decomposition, where each has a dependency on the previous one (can't extract words before downloading).

---

### Data Decomposition (Chapter 10)

**Definition:** Breaking a program down by asking, *"How can we organize the **data** in our program so that we can execute more work in parallel?"*

**Input data decomposition:** Dividing the program's *input* and feeding pieces to multiple concurrent executions. **Example:** Ch. 3's letter-frequency program — each input URL is handed to a separate goroutine.

**Output data decomposition:** Using the program's *output* structure to distribute work. **Example:** Ch. 6's matrix multiplication — each goroutine is responsible for computing one output row.

**Note (explicit in text):** *"Task and data decomposition are principles that should be applied together when designing a concurrent program. Most concurrent applications apply a mixture of task and data decomposition to achieve an efficient solution."*

---

### Task Granularity (Chapter 10)

**Definition:** How large or small the subtasks/data-chunks are when distributing work across concurrent executions.
- **Fine-grained**: problem broken into many small tasks.
- **Coarse-grained**: problem broken into few large tasks.

**Trade-offs (explicit in text, illustrated via a software team building an online shop):**
- **Too coarse:** Few large tasks → possible load imbalance (some workers idle after finishing quickly while others are still grinding through a big task); not enough tasks to distribute to everyone.
- **Too fine:** More even distribution and more available parallelism, but excessive time is spent on **communication and synchronization overhead** (e.g., "developers wasting time in meetings"), which can actually *reduce* overall efficiency.

**The "sweet spot":** Somewhere between the extremes maximizes speedup; its exact location depends on team/resource size, communication overhead, and — most importantly — how much the problem's *inherent* dependencies constrain parallelization.

**Tip (explicit in text):** *"Concurrent solutions that require very little communication and synchronization (due to the nature of the problem being solved) generally allow us to have finer-grained solutions and achieve bigger speedups."*

---

### Loop-Level Parallelism (Chapter 10)

**Definition:** Transforms each iteration of a sequential loop into an independent concurrent task, when the collection being processed has no inter-item dependencies.

**Example — file hashing per directory (independent iterations):**
```go
func FHash(filepath string) []byte {
    file, _ := os.Open(filepath)
    defer file.Close()
    sha := sha256.New()
    io.Copy(sha, file)
    return sha.Sum(nil)
}

func main() {
    dir := os.Args[1]
    files, _ := os.ReadDir(dir)
    wg := sync.WaitGroup{}
    for _, file := range files {
        if !file.IsDir() {
            wg.Add(1)
            go func(filename string) {
                fPath := filepath.Join(dir, filename)
                hash := FHash(fPath)
                fmt.Printf("%s - %x\n", filename, hash)
                wg.Done()
            }(file.Name())
        }
    }
    wg.Wait()
}
```

**Definition — Loop-carried dependence:** When a step in one loop iteration depends on a step from a *different* iteration of the *same* loop — this breaks the naive loop-level parallelism approach above.

**Example — hashing an entire directory (loop-carried dependence):** Combining each file's hash into a running directory-level hash requires strict ordering — step *i* must incorporate step *i-1*'s result before proceeding.

**Solution pattern:** Split each iteration into an **independent part** (computing the per-file hash, safe to parallelize) and a **dependent part** (combining into the running total, which must stay ordered) — using a chain of channels to pass a "ready" signal from each iteration to the next:
```go
func main() {
    dir := os.Args[1]
    files, _ := os.ReadDir(dir)
    sha := sha256.New()
    var prev, next chan int
    for _, file := range files {
        if !file.IsDir() {
            next = make(chan int)
            go func(filename string, prev, next chan int) {
                fpath := filepath.Join(dir, filename)
                hashOnFile := FHash(fpath)      // independent part — runs concurrently
                if prev != nil {
                    <-prev                       // wait for previous iteration's turn
                }
                sha.Write(hashOnFile)            // dependent part — must stay ordered
                next <- 0                        // signal next iteration
            }(file.Name(), prev, next)
            prev = next
        }
    }
    <-next
    fmt.Printf("%x\n", sha.Sum(nil))
}
```
**Important requirement (explicit in text):** `os.ReadDir()` returns entries in a fixed directory order — this ordering guarantee is essential, since an undefined iteration order would make the combined hash non-reproducible.

---

### Fork/Join Pattern (Chapter 10)

**Definition:** Spawn ("fork") one execution per independent task, then wait for and merge ("join") all their results before proceeding — appropriate when there's an initial fully-parallel phase followed by a single aggregation step, **with no ordering requirement** on how results are combined.

**Example — finding the source file with the deepest nested code block:**
```go
type CodeDepth struct{ file string; level int }

func deepestNestedBlock(filename string) CodeDepth {
    code, _ := os.ReadFile(filename)
    max, level := 0, 0
    for _, c := range code {
        if c == '{' {
            level += 1
            max = int(math.Max(float64(max), float64(level)))
        } else if c == '}' {
            level -= 1
        }
    }
    return CodeDepth{filename, max}
}

// FORK
func forkIfNeeded(path string, info os.FileInfo, wg *sync.WaitGroup, results chan CodeDepth) {
    if !info.IsDir() && strings.HasSuffix(path, ".go") {
        wg.Add(1)
        go func() {
            results <- deepestNestedBlock(path)
            wg.Done()
        }()
    }
}

// JOIN
func joinResults(partialResults chan CodeDepth) chan CodeDepth {
    finalResult := make(chan CodeDepth)
    max := CodeDepth{"", 0}
    go func() {
        for pr := range partialResults {
            if pr.level > max.level {
                max = pr
            }
        }
        finalResult <- max
    }()
    return finalResult
}

func main() {
    dir := os.Args[1]
    partialResults := make(chan CodeDepth)
    wg := sync.WaitGroup{}
    filepath.Walk(dir, func(path string, info os.FileInfo, err error) error {
        forkIfNeeded(path, info, &wg, partialResults)
        return nil
    })
    finalResult := joinResults(partialResults)
    wg.Wait()
    close(partialResults)
    result := <-finalResult
    fmt.Printf("%s has the deepest nested code block of %d\n", result.file, result.level)
}
```
**Note (explicit in text):** Unlike the directory-hash example, this problem does **not** rely on the order of partial results (finding a max is order-independent) — this lack of ordering requirement is exactly what allows the simpler fork/join pattern to apply cleanly, versus needing the more complex chained-channel approach.

---

### Worker Pool Pattern (Chapter 10)

**Definition:** A fixed (or dynamically resizable) number of pre-created goroutines pull work items from a shared queue (implemented as a channel); each worker is either idle waiting for work or busy processing a task. Also known by other names: **thread pool pattern, replicated workers, master/worker, worker-crew model.**

**Example — minimal HTTP server:**
```go
func handleHttpRequest(conn net.Conn) {
    buff := make([]byte, 1024)
    size, _ := conn.Read(buff)
    if r.Match(buff[:size]) {
        file, err := os.ReadFile(fmt.Sprintf("../resources/%s", r.FindSubmatch(buff[:size])[1]))
        if err == nil {
            conn.Write([]byte(fmt.Sprintf("HTTP/1.1 200 OK\r\nContent-Length: %d\r\n\r\n", len(file))))
            conn.Write(file)
        } else {
            conn.Write([]byte("HTTP/1.1 404 Not Found\r\n\r\n<html>Not Found</html>"))
        }
    } else {
        conn.Write([]byte("HTTP/1.1 500 Internal Server Error\r\n\r\n"))
    }
    conn.Close()
}

func StartHttpWorkers(n int, incomingConnections <-chan net.Conn) {
    for i := 0; i < n; i++ {
        go func() {
            for c := range incomingConnections {
                handleHttpRequest(c)
            }
        }()
    }
}

func main() {
    incomingConnections := make(chan net.Conn)
    StartHttpWorkers(3, incomingConnections)
    server, _ := net.Listen("tcp", "localhost:8080")
    defer server.Close()
    for {
        conn, _ := server.Accept()
        incomingConnections <- conn // load-balanced to whichever worker is free
    }
}
```

**Extension — limiting load with `select`'s `default` case ("server busy" response):**
```go
select {
case incomingConnections <- conn:
default:
    fmt.Println("Server is busy")
    conn.Write([]byte("HTTP/1.1 429 Too Many Requests\r\n\r\n<html>Busy</html>\n"))
    conn.Close()
}
```

**Note (explicit, important caveat for Go specifically):** *"The worker pool pattern is especially useful when creating new threads of execution is expensive... In Go, creating goroutines is a very fast process, so this pattern doesn't bring much benefit in terms of performance."* However, worker pools remain useful in Go specifically **to cap concurrency** so servers/programs don't exhaust resources under unexpected load spikes — not primarily for avoiding goroutine-creation cost.

---

### Pipelining Pattern (Chapter 10)

**Definition:** Used when tasks have a strict linear dependency chain (each step's output feeds the next step's input) — modeled after a manufacturing assembly line. Different workers each specialize in one stage, and multiple items flow through the pipeline concurrently (item N can be at stage 3 while item N+1 is at stage 1).

**Example — cupcake factory (5-stage pipeline: prepare tray → pour mixture → bake → add toppings → box):**
```go
func AddOnPipe[X, Y any](q <-chan int, f func(X) Y, in <-chan X) chan Y {
    output := make(chan Y)
    go func() {
        defer close(output)
        for {
            select {
            case <-q:
                return
            case input := <-in:
                output <- f(input)
            }
        }
    }()
    return output
}

func main() {
    input := make(chan int)
    quit := make(chan int)
    output := AddOnPipe(quit, Box,
        AddOnPipe(quit, AddToppings,
            AddOnPipe(quit, Bake,
                AddOnPipe(quit, Mixture,
                    AddOnPipe(quit, PrepareTray, input)))))
    go func() {
        for i := 0; i < 10; i++ {
            input <- i
        }
    }()
    for i := 0; i < 10; i++ {
        fmt.Println(<-output, "received")
    }
}
```
**Measured result:** Sequential production of 10 boxes took ~131s (13s/box × 10); pipelined production took ~59s — a substantial speedup, though not a full 10x, because of the pipeline properties below.

---

### Pipelining Properties — Throughput vs. Latency (Chapter 10)

**Definition — Latency:** The total time for **one item** to travel from start to finish through the entire pipeline (sum of all stage times).

**Definition — Throughput:** The **rate** at which completed items emerge from the pipeline over time — dictated by the **slowest stage** (the bottleneck), since every other stage must wait for the bottleneck to free up capacity.

**Key finding #1 (explicit in text):** *"In a pipeline, the throughput rate is dictated by the slowest step. The latency of the system is the sum of the time it takes to perform every step along the way."*

**Experiment 1 — speeding up all *non-bottleneck* steps (everyThingElseTime: 2s → 1s):** Overall time to produce 10 boxes barely changed (~59s → ~56s) — because the 5-second **baking** step remained the bottleneck; four workers now work twice as fast but throughput is still capped by the oven.

**Tip (explicit):** *"To increase the throughput of a system, it's always best to focus on the bottleneck of that system."* However, speeding up the non-bottleneck steps **does** meaningfully reduce **latency** (single-item time dropped from 13s to 9s).

**Tip (explicit):** *"To reduce the latency of a pipeline system, we need to improve the speed of most steps in the pipeline."*

**Experiment 2 — speeding up the bottleneck itself (ovenTime: 5s → 2s, everyThingElseTime back to 2s):** Total time for 10 boxes dropped dramatically (~59s → ~30s) — because now every stage takes roughly the same time, keeping all workers constantly busy with minimal idle time, directly boosting **throughput**.

**Takeaway:** Improving the bottleneck stage improves throughput dramatically; improving non-bottleneck stages mainly improves per-item latency, with only marginal throughput gains.

---

### Deadlocks — Definition & Conditions (Chapter 11)

**Definition:** A deadlock occurs when executions block indefinitely, each waiting for another to release a resource it needs, so that none can proceed.

**Coffman conditions (1971 paper "System Deadlocks" by Coffman et al.) — all four must hold simultaneously for a deadlock to occur:**
1. **Mutual exclusion** — each resource is either free or held by exactly one execution.
2. **Wait-for condition** (a.k.a. hold-and-wait) — an execution holding one or more resources can request additional ones.
3. **No preemption** — a resource cannot be forcibly taken from the execution holding it; only that execution can release it voluntarily.
4. **Circular wait** — there exists a cycle of two or more executions, each waiting on a resource held by the next one in the cycle.

**Example — minimal two-goroutine, two-mutex deadlock:**
```go
func red(lock1, lock2 *sync.Mutex) {
    for {
        lock1.Lock()
        lock2.Lock()
        lock1.Unlock(); lock2.Unlock()
    }
}

func blue(lock1, lock2 *sync.Mutex) {
    for {
        lock2.Lock()  // NOTE: acquires in the OPPOSITE order from red()
        lock1.Lock()
        lock1.Unlock(); lock2.Unlock()
    }
}

func main() {
    lockA := sync.Mutex{}
    lockB := sync.Mutex{}
    go red(&lockA, &lockB)
    go blue(&lockA, &lockB)
    time.Sleep(20 * time.Second)
    fmt.Println("Done")
}
```
**Root cause:** `red()` acquires lock1-then-lock2, while `blue()` acquires lock2-then-lock1 — the opposite order. If `red()` grabs lock1 while `blue()` grabs lock2 at nearly the same time, each then blocks waiting for the other's lock — permanently.

**Note (explicit, important non-determinism caveat):** *"Due to the non-deterministic nature of concurrent executions, running listings 11.1 and 11.2 will not always result in a deadlock."* Adding `Sleep()` calls between the two lock acquisitions increases the odds of triggering it reliably for demonstration/testing purposes.

---

### Resource Allocation Graphs (RAGs) (Chapter 11)

**Definition:** A directed graph used to visualize and formally detect deadlocks. **Nodes** are either executions (circles) or resources (rectangles). **Edges**:
- Execution → Resource (dashed) = the execution is **requesting** that resource.
- Resource → Execution (solid) = the execution is currently **holding** that resource.

**Key detection rule (explicit in text):** *"Whenever a resource allocation graph contains such a cycle, it means that a deadlock has occurred."* A cycle means you can trace a path of edges from any node back to itself.

**Real-world analogy:** A rail-crossing network where trains need to reserve multiple crossings simultaneously — each train acquiring crossings one at a time can create the exact same circular-wait structure as goroutines fighting over mutexes, illustrating that deadlocks are a general resource-allocation phenomenon, not something unique to software.

**Example — 3-goroutine deadlock in a bank ledger app:** With `Transfer()` locking a source account then a target account (in that call order), concurrent random transfers between 4 accounts can create a cycle among any subset of the goroutines (e.g., goroutines 0, 2, and 3), even while leaving other goroutines (e.g., goroutine 1) merely *blocked* (not part of the cycle itself, but stuck waiting on a resource held within it).

---

### Detecting Deadlocks (Chapter 11)

**Go's built-in detection — how it works and its limitation:** Go's runtime checks which goroutine to run next; if it finds **every single goroutine** blocked waiting on a resource (mutex, channel, etc.), it throws:
```
fatal error: all goroutines are asleep - deadlock!
```
**Critical limitation (explicit in text):** This only fires when **all** goroutines are simultaneously blocked. If even one goroutine remains runnable (e.g., sleeping via `time.Sleep()` rather than blocked on a lock), Go's detector **will not** notice the deadlock among the others — the program will simply hang forever in that subset without any error.

**Demonstration of working around (accidentally defeating) Go's detector:**
```go
func main() {
    lockA, lockB := sync.Mutex{}, sync.Mutex{}
    wg := sync.WaitGroup{}
    wg.Add(2)
    go lockBoth(&lockA, &lockB, &wg)
    go lockBoth(&lockB, &lockA, &wg)
    go func() {
        wg.Wait() // this goroutine blocks, but...
        fmt.Println("Done waiting on waitgroup")
    }()
    time.Sleep(30 * time.Second) // ...main() itself is only sleeping, not blocked
    fmt.Println("Done")
}
```
Because `main()` is merely sleeping (a runnable state, just delayed) rather than blocked, Go's runtime never sees "all goroutines asleep," so no deadlock error is ever printed — the program silently hangs (in the two locking goroutines) for the full 30 seconds and then exits normally, masking the underlying deadlock entirely.

**General/algorithmic deadlock detection:** Build a resource allocation graph programmatically and run a **modified depth-first search** to detect cycles (tracking visited nodes during traversal; revisiting an already-visited node signals a cycle). This is the approach used by systems like databases.

**Side note — real-world example:** MySQL detects deadlocks between concurrent transactions and returns: `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction` — the client can then roll back and retry.

**Side note — why Go doesn't do full graph-based detection:** Maintaining a live resource allocation graph and running cycle-detection on every resource request/release would be a costly performance overhead for a general-purpose runtime with potentially huge numbers of goroutines and locks. This overhead is *not* prohibitive for databases, though, because "the detection algorithm is fast relative to the slow database operations" already being performed.

**Response strategies once a deadlock is detected (either approach used by different systems):**
1. **Terminate** the stuck executions (Go's approach — though Go terminates the *entire process*, not just the deadlocked goroutines).
2. **Return an error** to the requesting executions so they can decide to retry (common database approach — e.g., roll back and retry the transaction).

**Side note — "the ostrich method":** *Doing nothing* about deadlocks is itself a legitimate (if named somewhat pejoratively) strategy — appropriate only when deadlocks are known to be rare and low-consequence in your specific system. (Text notes this term stems from a popular misconception about ostrich behavior.)

---

### Avoiding Deadlocks — The Banker's Algorithm (Chapter 11)

**Definition:** An algorithm (developed by Edsger Dijkstra) for deciding whether it's *safe* to grant a resource request, given full advance knowledge of: (1) the maximum number of each resource type each execution might ever request, (2) what each execution currently holds, and (3) how many of each resource remain available.

**Definition — Safe vs. unsafe state:** A system state is **safe** if there exists *some* order of scheduling remaining requests such that every execution can eventually complete (even if each requests its declared maximum). Otherwise the state is **unsafe** — an unsafe state doesn't guarantee a deadlock will happen, but it means one *could* happen depending on future requests.

**How the algorithm works:** Before granting any resource request, simulate granting it and check whether the resulting state is still safe. If yes, grant it. If no, **suspend** the requesting execution until the request can be safely granted (i.e., until other executions release enough resources).

**Example applied to the train-crossing scenario:** By knowing each train's planned route (which crossings it will need) in advance, a smart dispatcher can hold back trains 2 and 4 at their first crossing request (since granting it would risk an eventual unsafe/deadlocked state), while letting trains 1 and 3 proceed uninterrupted (since their full crossing needs don't create risk).

**Applied to the ledger application — a custom "arbitrator":** Since the full set of resources each goroutine needs (exactly two specific bank accounts) is known **in advance** for each transaction, a simplified custom scheme suffices instead of the full banker's algorithm:
```go
type Arbitrator struct {
    accountsInUse map[string]bool
    cond          *sync.Cond
}

func NewArbitrator() *Arbitrator {
    return &Arbitrator{
        accountsInUse: map[string]bool{},
        cond:          sync.NewCond(&sync.Mutex{}),
    }
}

func (a *Arbitrator) LockAccounts(ids ...string) {
    a.cond.L.Lock()
    for allAvailable := false; !allAvailable; {
        allAvailable = true
        for _, id := range ids {
            if a.accountsInUse[id] {
                allAvailable = false
                a.cond.Wait()
            }
        }
    }
    for _, id := range ids {
        a.accountsInUse[id] = true
    }
    a.cond.L.Unlock()
}

func (a *Arbitrator) UnlockAccounts(ids ...string) {
    a.cond.L.Lock()
    for _, id := range ids {
        a.accountsInUse[id] = false
    }
    a.cond.Broadcast()
    a.cond.L.Unlock()
}
```
**Key idea:** Never partially lock accounts — either **all** required accounts become available together and are locked atomically as a set (via the condition-variable retry loop), or the goroutine waits entirely. This structurally prevents the circular-wait condition, since no goroutine ever holds one account while blocked on another.

**Side note — why the banker's algorithm isn't used in real OS/runtime schedulers:** It requires advance knowledge of the *maximum* resource needs of every execution — unrealistic for general-purpose operating systems or language runtimes, where processes/threads/goroutines request resources dynamically and unpredictably. It also assumes a *fixed* set of executions, which doesn't hold for systems where processes are constantly starting and terminating.

---

### Preventing Deadlocks — Resource Ordering (Chapter 11)

**Definition:** If the full set of exclusive resources an execution will need is known in advance, **always acquire them in a consistent, predefined global order** (e.g., alphabetical). This eliminates the possibility of circular wait entirely, since no two executions can ever be holding-one-while-waiting-for-the-other's resource in opposite directions.

**Fixing the red/blue deadlock via consistent ordering:**
```go
func red(lock1, lock2 *sync.Mutex) {
    for {
        lock1.Lock()
        lock2.Lock()
        lock1.Unlock(); lock2.Unlock()
    }
}

func blue(lock1, lock2 *sync.Mutex) {
    for {
        lock1.Lock() // SAME order as red() now — no more deadlock possible
        lock2.Lock()
        lock1.Unlock(); lock2.Unlock()
    }
}
```
**Why this works:** With both goroutines always requesting lock1 before lock2, whichever goroutine gets lock1 first will also be the only one able to proceed to request lock2 — the other is fully blocked at the very first lock, never holding anything while waiting, so no cycle can form.

**Applied to the ledger app — sort accounts by ID before locking:**
```go
func (src *BankAccount) Transfer(to *BankAccount, amount int, tellerId int) {
    accounts := []*BankAccount{src, to}
    sort.Slice(accounts, func(a, b int) bool {
        return accounts[a].id < accounts[b].id
    })
    accounts[0].mutex.Lock() // always lock alphabetically-first account first
    accounts[1].mutex.Lock()
    src.balance -= amount
    to.balance += amount
    accounts[1].mutex.Unlock()
    accounts[0].mutex.Unlock()
}
```
**Extended rule for dynamically-discovered resource needs (conditional/fallback logic):** When you don't know all needed resources upfront (e.g., "pay from Amy's account; if insufficient, use Mia's account instead"), the rule becomes: *never acquire a lower-order resource while already holding a higher-order one.* If a situation requires acquiring an out-of-order resource, **release everything currently held first**, then re-acquire everything (including the new resource) in the correct order from scratch.

---

### Deadlocks With Channels (Chapter 11)

**Definition/insight:** Deadlocks aren't limited to mutexes — a channel's capacity (or its unbuffered rendezvous requirement) can also be modeled as a mutually-exclusive resource that a goroutine "holds" (is blocked trying to use) while requesting another. A **circular wait between channel operations** produces the same deadlock structure as with mutexes.

**Conceptual model:** An unbuffered channel starts with zero "read resources" and zero "write resources" available. A pending write operation makes one read resource available (and consumes a write resource, in the sense that the writer is now occupying/blocking on it); a pending read makes one write resource available.

**Example — circular deadlock between a file-handler and directory-handler goroutine:**
```go
func handleDirectories(dirs <-chan string, files chan<- string) {
    for fullpath := range dirs {
        filesInDir, _ := os.ReadDir(fullpath)
        for _, file := range filesInDir {
            files <- filepath.Join(fullpath, file.Name()) // blocks if handleFiles is busy elsewhere
        }
    }
}

func handleFiles(files chan string, dirs chan string) {
    for path := range files {
        file, _ := os.Open(path)
        fileInfo, _ := file.Stat()
        if fileInfo.IsDir() {
            dirs <- path // blocks if handleDirectories is busy elsewhere
        } else {
            // print file info
        }
    }
}
```
**Deadlock scenario:** `handleDirectories` is blocked trying to push a file onto `files` (waiting for `handleFiles` to read), while at the same moment `handleFiles` is blocked trying to push a directory onto `dirs` (waiting for `handleDirectories` to read) — each is stuck sending while the other is also stuck sending, and neither is currently in a position to receive.

**Ineffective "fixes" explicitly called out as only postponing the problem:**
- **Adding a buffer** to one channel — deadlock still occurs once the buffer fills, just delayed to a directory with more files/subdirectories than the buffer size.
- **Adding more worker goroutines** to `handleFiles` — deadlock still occurs once you hit a directory with more files than available worker goroutines.

**Actual fix #1 — break the circular wait by delegating the send to a fresh goroutine:**
```go
func handleDirectories(dirs <-chan string, files chan<- string) {
    for fullpath := range dirs {
        filesInDir, _ := os.ReadDir(fullpath)
        for _, file := range filesInDir {
            go func(fp string) {
                files <- fp // now runs in its own goroutine, doesn't block the loop
            }(filepath.Join(fullpath, file.Name()))
        }
    }
}
```

**Actual fix #2 — use `select` to simultaneously offer to send AND receive:**
```go
func handleDirectories(dirs <-chan string, files chan<- string) {
    toPush := make([]string, 0)
    appendAllFiles := func(path string) {
        filesInDir, _ := os.ReadDir(path)
        for _, f := range filesInDir {
            toPush = append(toPush, filepath.Join(path, f.Name()))
        }
    }
    for {
        if len(toPush) == 0 {
            appendAllFiles(<-dirs)
        } else {
            select {
            case fullpath := <-dirs:
                appendAllFiles(fullpath)
            case files <- toPush[0]:
                toPush = toPush[1:]
            }
        }
    }
}
```
**Why this works:** The `select` lets the goroutine remain responsive to **either** direction of traffic at once — if it can't currently send (no receiver ready), it can still receive, and vice versa — breaking the strict "blocked only on sending" circular dependency.

**Note (explicit, important design-level takeaway):** *"Having deadlocks in message-passing programs is often a sign of bad program design. Having a deadlock while using channels means that we have programmed a circular flow of messages going through the same goroutines. Most of the time, we can avoid possible deadlocks by designing our programs so that the flow of messages is not circular."*

---

### Atomic Variables (Chapter 12)

**Definition:** Atomic variables support certain operations (add, compare-and-swap, load, store, swap) that execute **without interruption** — no other goroutine can observe or interfere with a partially-completed atomic operation. This allows safe concurrent updates to a **single** variable without needing a full mutex.

**Syntax — `sync/atomic` package (32-bit example; equivalents exist for other integer sizes and other types like booleans/unsigned integers):**
```go
func AddInt32(addr *int32, delta int32) (new int32)
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func LoadInt32(addr *int32) (val int32)
func StoreInt32(addr *int32, val int32)
func SwapInt32(addr *int32, new int32) (old int32)
```

**Example — Stingy/Spendy without any mutex at all:**
```go
func stingy(money *int32) {
    for i := 0; i < 1000000; i++ {
        atomic.AddInt32(money, 10)
    }
}

func spendy(money *int32) {
    for i := 0; i < 1000000; i++ {
        atomic.AddInt32(money, -10)
    }
}

func main() {
    money := int32(100)
    wg := sync.WaitGroup{}
    wg.Add(2)
    go func() { stingy(&money); wg.Done() }()
    go func() { spendy(&money); wg.Done() }()
    wg.Wait()
    fmt.Println("Money in account: ", atomic.LoadInt32(&money)) // always 100
}
```
**Best practice (explicit in text):** Even though reading after `wg.Wait()` guarantees the goroutines are done, it's still "good practice to use atomic load operations when working with atomics to ensure that we read the latest value from main memory and not an outdated cached value."

**Critical limitation (explicit in text):** Atomic operations only guarantee atomicity for **one variable at a time**. If you need to update **multiple** related variables together as a single logical unit (e.g., the flight-booking scenario below), atomics alone cannot help — you need a mutex or similar tool.

**Performance cost (measured via Go's benchmarking tool):** The text's own micro-benchmark showed atomic addition on a 64-bit integer running **more than 3x slower** than a plain (non-atomic) increment — because atomics forfeit compiler/CPU caching optimizations, requiring the system to keep caches across cores consistent (flushing to main memory, invalidating other caches) on every atomic operation.

**Syntax — Go benchmark tests:**
```go
func BenchmarkNormal(bench *testing.B) {
    for i := 0; i < bench.N; i++ {
        total += 1
    }
}
func BenchmarkAtomic(bench *testing.B) {
    for i := 0; i < bench.N; i++ {
        atomic.AddInt64(&total, 1)
    }
}
```
Run via: `go test -bench=. -count 3`

---

### `CompareAndSwap()` (Chapter 12)

**Definition:** An atomic operation that checks whether a variable currently holds an expected `old` value; if so, it atomically updates it to `new` and returns `true`. If the current value doesn't match `old`, nothing is changed and it returns `false`.

**Example:**
```go
number := int32(17)
result := atomic.CompareAndSwapInt32(&number, 17, 19)
// number is now 19, result is true

number = int32(23)
result = atomic.CompareAndSwapInt32(&number, 17, 19)
// number is unchanged (still 23), result is false
```
This is the foundational primitive used to build spin locks (below).

---

### Spin Locks (Chapter 12)

**Definition:** A mutex implementation built entirely in user space using an atomic integer as a "locked/unlocked" flag and a tight retry loop using `CompareAndSwap()`. Called a **spin lock** because a goroutine trying to acquire an already-locked mutex loops ("spins") repeatedly attempting to acquire it, rather than being suspended by the OS/runtime.

**Syntax/implementation:**
```go
type SpinLock int32

func (s *SpinLock) Lock() {
    for !atomic.CompareAndSwapInt32((*int32)(s), 0, 1) {
        runtime.Gosched() // yields, giving other goroutines a chance to run
    }
}

func (s *SpinLock) Unlock() {
    atomic.StoreInt32((*int32)(s), 0)
}

func NewSpinLock() sync.Locker {
    var lock SpinLock
    return &lock
}
```
**Application — flight-booking system requiring atomic multi-variable updates:** Since a customer's full itinerary might span multiple flights, and each flight's `SeatsLeft` must be checked/decremented as an all-or-nothing unit, a per-flight `Locker` (backed by this spin lock) is used, with all flights in a booking sorted alphabetically and locked in that fixed order (to prevent deadlock, per Ch. 11's ordering technique) before checking/updating seat counts:
```go
func Book(flights []*Flight, seatsToBook int) bool {
    bookable := true
    sort.Slice(flights, func(a, b int) bool {
        return flights[a].Origin+flights[a].Dest < flights[b].Origin+flights[b].Dest
    })
    for _, f := range flights {
        f.Locker.Lock()
    }
    for i := 0; i < len(flights) && bookable; i++ {
        if flights[i].SeatsLeft < seatsToBook {
            bookable = false
        }
    }
    for i := 0; i < len(flights) && bookable; i++ {
        flights[i].SeatsLeft -= seatsToBook
    }
    for _, f := range flights {
        f.Locker.Unlock()
    }
    return bookable
}
```

**Definition — Resource contention:** When an execution uses a resource in a way that blocks/slows other executions wanting the same resource.

**Explicit limitation of spin locks:** Under high contention, waiting goroutines burn CPU cycles endlessly re-checking `CompareAndSwap()` in a loop — wasted cycles that could otherwise do useful work. `runtime.Gosched()` mitigates this somewhat by yielding, but the fundamental busy-waiting cost remains.

---

### Futexes (Chapter 12)

**Definition:** "Futex" = **fast userspace mutex** (a name the text calls *misleading*, since "futexes are not mutexes at all"). A futex is an OS-provided **wait-queue primitive** accessible from user space, letting you suspend/awaken an execution tied to a specific memory address — used as a building block for efficient mutexes, semaphores, and condition variables.

**Conceptual API (pseudocode, since Go doesn't expose futex syscalls directly):**
- `futex_wait(address, value)` — if the value currently at `address` equals `value`, suspend the caller and enqueue it on a per-address wait queue. If the value differs, return immediately (no wait).
- `futex_wake(address, count)` — wake up to `count` suspended executions waiting on `address` (taken from the front of that address's queue); `count = 0` wakes **all** of them.

**Side note — real OS implementations:**
- **Linux:** both `futex_wait`/`futex_wake` implemented via `syscall(SYS_futex, ...)` with `FUTEX_WAIT` / `FUTEX_WAKE` parameters.
- **Windows:** `WaitOnAddress()` for waiting; `WakeByAddressSingle()` or `WakeByAddressAll()` for waking.

**Naive futex-based mutex (pseudo-Go, since real futex syscalls aren't exposed in Go):**
```go
type FutexLock int32

func (f *FutexLock) Lock() {
    for !atomic.CompareAndSwapInt32((*int32)(f), 0, 1) {
        futex_wait((*int32)(f), 1)
    }
}

func (f *FutexLock) Unlock() {
    atomic.StoreInt32((*int32)(f), 0)
    futex_wakeup((*int32)(f), 1)
}
```
**Race-avoidance detail (explicit in text):** Passing `1` to `futex_wait()` avoids a race where the lock is released *just after* the failed `CompareAndSwap()` but *before* `futex_wait()` executes — if that happens, `futex_wait` finds the value is no longer `1` and returns immediately instead of sleeping forever, so the loop simply retries.

**Improvement — avoiding unnecessary wake-up system calls:** Since `futex_wakeup()` is an expensive syscall (context-switches into and back out of the kernel), always calling it on every `Unlock()` — even when nobody is waiting — wastes performance in the low-contention case. Fix: give the lock variable a **third state**, `2` = "locked with waiters," so `Unlock()` only calls `futex_wakeup()` when it's actually needed:
```go
type FutexLock int32

func (f *FutexLock) Unlock() {
    oldValue := atomic.SwapInt32((*int32)(f), 0)
    if oldValue == 2 { // only wake if there were waiters
        futex_wakeup((*int32)(f), 1)
    }
}

func (f *FutexLock) Lock() {
    if !atomic.CompareAndSwapInt32((*int32)(f), 0, 1) {
        for atomic.SwapInt32((*int32)(f), 2) != 0 {
            futex_wait((*int32)(f), 2)
        }
    }
}
```
**Note (explicit — conservative design choice):** After waking from `futex_wait()`, the goroutine always re-marks the lock as `2` ("locked with waiters") even if it can't be sure any other goroutine is actually still waiting — "we play it safe... at the cost of occasionally doing an unnecessary futex_wakeup() system call," trading a rare wasted syscall for correctness simplicity.

**State meanings summary:** `0` = unlocked, `1` = locked (no known waiters), `2` = locked with waiters present.

---

### Go's Actual Mutex Implementation (Chapter 12)

**Key architectural fact (explicit in text):** Since Go uses a **user-level threading model** (goroutines multiplexed onto kernel threads — recall Ch. 2's M:N model), calling a real futex `wait` would suspend an entire **kernel-level thread**, not just the individual goroutine — wasteful, since other goroutines sharing that kernel thread would also be blocked. So **Go's `sync.Mutex` does not use OS futexes directly.**

**What Go does instead:** Implements its **own internal semaphore** (in the `runtime` package, not directly exposed to user code) that queues and suspends *goroutines* (not kernel threads) entirely within Go's own user-space runtime — mirroring what a futex-based system would do, but at the goroutine level instead of the OS-thread level. Source: `https://github.com/golang/go/blob/master/src/runtime/sema.go`.

**Pseudocode for the internal `semacquire`-style logic:**
```go
func semaphoreAcquire(permits *int32, queueAtTheBack bool) {
    for {
        v := atomic.LoadInt32(permits)
        if v != 0 && atomic.CompareAndSwapInt32(permits, v, v-1) {
            break
        }
        // Queue functions only actually queue/park the goroutine if permits == 0
        if queueAtTheBack {
            queueAndSuspendGoroutineAtTheEnd(permits)
        } else {
            queueAndSuspendGoroutineInFront(permits)
        }
    }
}
```
**Note:** This internal semaphore supports queueing a goroutine either at the **back** (normal) or the **front** (priority) of its wait queue — this capability is used by the starvation-avoidance mechanism below.

**`sync.Mutex` layered design:** `sync.Mutex` first attempts a cheap `CompareAndSwap()` directly (like a spin lock), and only falls back to the internal semaphore (to actually suspend the goroutine) if that fails — combining spin-lock-like speed in the uncontended case with proper suspension in the contended case.

**Two operating modes — normal vs. starvation:**

- **Normal mode:** Waiting goroutines queue normally (FIFO, back of queue) on the internal semaphore. **Problem (explicit in text):** A goroutine resumed from the front of the queue must *re-compete* via `CompareAndSwap()` against brand-new goroutines that just called `Lock()` and haven't queued yet — and newly-arriving goroutines have a structural advantage (they're already running/scheduled), so a resumed goroutine can lose repeatedly, risking **starvation**.
- **Starvation mode:** If a goroutine at the front of the queue fails to acquire the lock even after being resumed, it's put back at the **front** (priority position) rather than the back. If this keeps happening and the goroutine has been waiting more than **1 millisecond**, the mutex switches into starvation mode. In this mode, the lock is handed **directly** to the front-of-queue goroutine on unlock — newly-arriving goroutines don't even attempt `CompareAndSwap()`; they go straight to the back of the queue. The mutex reverts to normal mode once the queue empties, or once a waiter acquires the lock in under 1ms.

**Why this two-mode design exists (explicit rationale):** Normal mode is very efficient under **low contention** (goroutines can grab the lock quickly without queueing overhead). Starvation mode kicks in specifically to guarantee fairness and prevent indefinite starvation under **high contention**, at some cost to raw throughput.

**Source reference:** `https://go.dev/src/sync/mutex.go`

---

## Patterns & Idioms

### Task Dependency Graph Analysis (Chapter 10)
**When to use:** Before parallelizing any multi-step process, to identify which steps have hard ordering constraints vs. which are free to run concurrently.
**Why:** Prevents both under-parallelizing (missing legitimate concurrency opportunities) and over-parallelizing (incorrectly running dependent steps out of order).

### Loop-Level Parallelism with Dependency Chaining (Chapter 10)
**When to use:** A loop where most of each iteration's work is independent, but a small final step must be applied in strict iteration order.
**Why:** Lets you capture most of the available parallelism (the independent, often slower part) while still guaranteeing correctness for the small ordered part, via a chain of hand-off channels between iterations.

### Fork/Join for Order-Independent Aggregation (Chapter 10)
**When to use:** An initial fully-parallel computation phase followed by a single merge/reduce step where the merge order doesn't matter (sum, max, count, etc.).
**Why:** Simpler than loop-carried-dependency chaining — a plain `WaitGroup` + a single aggregating goroutine consuming a shared results channel suffices.

### Worker Pool for Bounded, Reusable Concurrency (Chapter 10)
**When to use:** Handling a variable/unpredictable stream of incoming work (e.g., HTTP requests) where you want to cap maximum concurrency.
**Why (in Go specifically):** Not for goroutine-creation cost savings (cheap in Go) but for **resource protection** — capping how much work can be in flight at once, with the option to reject excess load gracefully via `select`+`default`.

### Pipeline for Strictly Sequential Multi-Stage Work (Chapter 10)
**When to use:** A process where each stage's output is the next stage's required input, and many items need processing over time (so different items can occupy different stages simultaneously).
**Why:** Maximizes utilization of all stages concurrently; but remember throughput is capped by the slowest stage — profile and optimize the bottleneck first.

### Resource Ordering to Prevent Deadlocks (Chapter 11)
**When to use:** Any time multiple exclusive resources must be acquired together and the full resource set is knowable in advance (or can be sorted by a stable key).
**Why:** Structurally eliminates circular wait — the root cause of deadlock — by guaranteeing all executions request shared resources in the same global order.
```go
sort.Slice(resources, func(a, b int) bool { return resources[a].ID < resources[b].ID })
for _, r := range resources { r.Lock() }
// ... critical section ...
for _, r := range resources { r.Unlock() }
```

### Arbitrator / All-or-Nothing Resource Acquisition (Chapter 11)
**When to use:** When resource ordering alone is insufficient or inconvenient, and you know the exact resource set needed per operation.
**Why:** A condition-variable-backed "all or nothing" acquisition guarantees no goroutine ever holds a strict subset of its needed resources while waiting for the rest — eliminating circular wait without requiring a fixed lock-acquisition order.

### Breaking Circular Wait in Channel Pipelines (Chapter 11)
**When to use:** Two (or more) goroutines that both send *and* receive from channels connecting them, risking a circular blocked-send condition.
**Why:** Using `select` to simultaneously offer a send and a receive (or spawning a fresh goroutine per send) ensures a goroutine is never *purely* blocked in one direction, breaking the cycle. Best practice, though, is to design message flow to avoid circularity in the first place.

### Layered Lock Implementation (Fast Path + Slow Path) (Chapter 12)
**When to use:** Building any custom high-performance lock.
**Why:** Attempt a cheap atomic `CompareAndSwap()` first (fast path for the uncontended case); only fall back to a more expensive suspension mechanism (semaphore/futex) when contention is actually detected (slow path). This is exactly the design of Go's own `sync.Mutex`.

---

## Warnings, Pitfalls & Gotchas

- **Choosing task granularity poorly hurts performance in either direction:** too coarse wastes idle capacity and creates load imbalance; too fine drowns useful work in synchronization/communication overhead.
- **Loop-carried dependence breaks naive loop-level parallelism** — you cannot simply spawn one goroutine per iteration if a later iteration's correctness depends on an earlier iteration's result; you must isolate the independent portion and chain-synchronize the dependent portion.
- **Non-deterministic ordering can silently corrupt "deterministic" aggregation** — e.g., relying on `os.ReadDir()`'s stable ordering was explicitly called out as a *requirement* for the directory-hash example to produce reproducible results; an unordered iteration would produce different hashes on different runs even with unchanged directory contents.
- **Worker pools bring little to no performance benefit in Go** (unlike languages with expensive OS-thread creation) — using them purely for "efficiency" is often unwarranted; their real value in Go is **capping concurrency**, not amortizing creation cost.
- **In a pipeline, optimizing the wrong stage wastes effort** — speeding up non-bottleneck stages barely moves throughput; you must identify and target the actual slowest stage to meaningfully increase throughput.
- **Deadlocks are non-deterministic and hard to reproduce** — "as with race conditions, we can have a program that runs without hitches for a long time, and then suddenly the execution halts" — the same run may or may not deadlock depending on timing.
- **Go's built-in deadlock detector only fires when *all* goroutines are simultaneously blocked** — if even one goroutine remains merely "sleeping" or otherwise runnable, a genuine deadlock among the rest will go completely undetected and the program will simply hang silently.
- **Buffering a channel or adding more worker goroutines only *postpones* a channel-based deadlock**, it doesn't fix it — the same circular-wait structure will eventually be triggered once load/data exceeds the buffer or worker count.
- **Full deadlock-avoidance algorithms (banker's algorithm) are impractical for general OS/runtime schedulers** — they require unrealistic advance knowledge of every execution's maximum resource needs, and assume a fixed, unchanging set of executions.
- **Atomic operations only cover a single variable** — attempting to use atomics alone to keep *multiple* related variables consistent (e.g., transferring money between two accounts, or booking seats across multiple flights) will not prevent race conditions across that multi-variable operation; a mutex or other broader synchronization tool is required.
- **Atomic operations are meaningfully slower than plain operations** (the text's benchmark showed >3x slower for a simple 64-bit add) — don't reach for atomics reflexively; they trade raw speed for lock-free safety on single variables.
- **Spin locks waste CPU under high contention** — a goroutine stuck spinning on `CompareAndSwap()` burns cycles that could otherwise do useful work; this is the direct motivation for futexes.
- **Calling an OS-level `futex_wakeup()` unconditionally on every unlock is wasteful** — since syscalls are expensive (context switch into and out of the kernel), a naive futex-based mutex that always wakes on unlock is slower than a spin lock in the *uncontended* case; the three-state (`0`/`1`/`2`) design specifically avoids this cost when there are no waiters.
- **Go's `sync.Mutex` cannot use raw OS futexes** because Go's goroutines are user-level constructs multiplexed onto OS threads — suspending via a real futex would block an entire kernel thread (and any other goroutines sharing it), not just the intended goroutine; this is why Go implements its own internal semaphore/queueing system instead.
- **Naive FIFO-queue mutex designs are vulnerable to starvation** — newly-arriving goroutines have a structural speed advantage over goroutines being resumed from a wait queue, since the resumed goroutine has to be rescheduled while the new arrival is already running; without Go's starvation-mode fallback, a queued goroutine could in theory wait indefinitely while new arrivals keep cutting in line.

---

## Important Side Notes

- **Different names for the worker pool pattern:** thread pool pattern, replicated workers, master/worker, worker-crew model — all describe essentially the same concept.
- **Galactic algorithms reference (carried over conceptually from Ch. 6, relevant to decomposition discussions):** not directly repeated here, but the granularity discussion in Ch. 10 echoes the same theme of theoretical vs. practical efficiency trade-offs.
- **Coffman et al.'s 1971 paper "System Deadlocks"** is the origin of the four necessary-and-sufficient deadlock conditions (mutual exclusion, wait-for, no preemption, circular wait) — foundational to all deadlock analysis since.
- **Real-world deadlock analogies given:** relationship conflicts, negotiations, road traffic — and the text notes that "road engineers spend a great deal of time and effort designing systems to minimize the risks of traffic deadlocks," underscoring that deadlock avoidance is a general systems-design problem, not just a computing one.
- **MySQL's deadlock handling** is cited as a concrete real-world example of full graph-based deadlock detection combined with an error-and-retry response strategy — contrasted with Go's simpler (and less complete) "detect-and-crash-the-whole-process" approach.
- **The banker's algorithm was developed by Edsger Dijkstra** — the same computer scientist credited (per Ch. 5) with inventing semaphores; both are foundational contributions to concurrent systems theory.
- **Futex is a misleading acronym** — "fast userspace mutex" sounds like it describes a mutex, but the text is explicit that "futexes are not mutexes at all"; they are a lower-level wait-queue primitive used *to build* mutexes (and semaphores, condition variables, etc.).
- **Platform-specific futex-equivalent APIs:** Linux uses `syscall(SYS_futex, ...)` with `FUTEX_WAIT`/`FUTEX_WAKE`; Windows uses `WaitOnAddress()` / `WakeByAddressSingle()` / `WakeByAddressAll()`.
- **Go's internal semaphore and mutex source code are both in the public Go repository** (`runtime/sema.go` and `sync/mutex.go` respectively) — the text explicitly points readers there for the full, edge-case-laden implementation beyond the simplified pseudocode presented.
- **Go's mutex starvation-mode threshold is hardcoded to 1 millisecond** — a specific, concrete implementation detail: a goroutine that's been waiting and repeatedly losing the race to newly-arrived goroutines for more than 1ms triggers the switch to starvation (fair-queueing) mode.

---

## Quick Reference Table

| Construct / API | Purpose | Example |
|---|---|---|
| Task dependency graph | Visualize step ordering constraints before parallelizing | nodes = tasks, edges = "depends on" |
| Loop-level parallelism | Parallelize independent loop iterations | `go func(item) { process(item) }(x)` per iteration |
| Loop-carried dependence chaining | Preserve ordering for a dependent sub-step across iterations | chain of per-iteration channels signaling "ready" |
| Fork/join | Parallel phase + order-independent merge | `WaitGroup` + goroutine draining a shared results channel |
| Worker pool | Bounded concurrency draining a shared work queue | N goroutines all `range`-ing over one channel |
| Pipeline (`AddOnPipe`) | Chain sequential-dependency stages concurrently | `stage3(stage2(stage1(input)))` via channels |
| Resource allocation graph (RAG) | Detect deadlocks visually/algorithmically | cycle in the graph ⇒ deadlock |
| Lock ordering | Prevent deadlock via consistent acquisition order | sort resources by ID, lock in that order |
| Arbitrator (condition-variable based) | All-or-nothing multi-resource locking | `LockAccounts(id1, id2)` / `UnlockAccounts(id1, id2)` |
| `select` for channel deadlock-breaking | Avoid circular blocked-send between goroutines | `select { case ch1<-x: ...; case y:=<-ch2: ... }` |
| `sync/atomic` functions | Lock-free single-variable updates | `atomic.AddInt32(&x, 1)` |
| `atomic.CompareAndSwapInt32` | Conditional atomic update | `CompareAndSwapInt32(&v, old, new) bool` |
| Spin lock (`CompareAndSwap` + loop) | User-space mutex, busy-waits under contention | `for !CAS(&lock,0,1) { Gosched() }` |
| Futex (`futex_wait`/`futex_wake`) | OS-level suspend/resume tied to a memory address | `futex_wait(&addr, expectedVal)` |
| Go's `sync.Mutex` | Production mutex: fast-path CAS + internal semaphore queue + starvation mode | `mu.Lock(); ...; mu.Unlock()` |

---

## Self-Check Questions

1. Explain the difference between task decomposition and data decomposition, and give an example of each from earlier chapters' programs (letter-frequency counter vs. matrix multiplication).
2. Walk through why "loop-carried dependence" breaks naive loop-level parallelism, and describe the channel-chaining technique used to still parallelize the independent part of each iteration.
3. In the cupcake-factory pipeline, explain why doubling the speed of every non-baking step barely changed total production time, while doubling the speed of the baking step nearly halved it. Connect this to the general throughput-vs-bottleneck principle.
4. State the four Coffman conditions required for a deadlock, and explain how "lock ordering" as a prevention technique specifically defeats the "circular wait" condition without touching the other three.
5. Why does Go's built-in deadlock detector fail to catch every deadlock, and what specific coding pattern (shown in the chapter) can accidentally hide a genuine deadlock from that detector?
6. Describe how a channel-based deadlock can occur between two goroutines that only use channels (no mutexes), and explain why simply adding a buffer or more worker goroutines only delays rather than fixes the problem.
7. Trace through why atomic operations alone cannot solve the flight-booking problem (multiple seats across multiple flights), and explain what additional technique (spin locks) was layered on top of atomics to solve it.
8. Explain the progression from spin locks → naive futex-based mutex → optimized 3-state futex-based mutex → Go's actual `sync.Mutex`. What specific inefficiency does each successive design fix, and why can't Go's mutex use real OS futexes directly?

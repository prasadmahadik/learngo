Go provides automatic memory management with a garbage collector (GC), so you do not manually manage memory with malloc/free, but understanding how it works is important for performance and correctness. [go](https://go.dev/doc/gc-guide)

1. Memory model overview  
- Go divides memory conceptually into stack and heap. [deepu](https://deepu.tech/memory-management-in-golang/)
- Stack: per‑goroutine, small, fast, grows and shrinks as needed, automatically reclaimed when functions return. [slideshare](https://www.slideshare.net/slideshow/memory-management-in-go-stack-heap-garbage-collector/265963594)
- Heap: shared, dynamically allocated memory for data that must outlive the current function or be shared across goroutines, managed by the GC. [tip.golang](https://tip.golang.org/doc/gc-guide)

2. Stack vs heap and escape analysis  
- The compiler decides whether a variable is placed on the stack or heap using escape analysis. [deepu](https://deepu.tech/memory-management-in-golang/)
- If a value never “escapes” its function (no reference returned or stored globally), it usually stays on the stack. [youtube](https://www.youtube.com/watch?v=F4YISvNxn3o)
- If a pointer to a value outlives the function (returned from the function, captured in a closure, stored in a global or heap object), the value escapes to the heap. [youtube](https://www.youtube.com/watch?v=F4YISvNxn3o)
- Stack allocation is extremely cheap; heap allocation is more expensive and adds GC pressure. [slideshare](https://www.slideshare.net/slideshow/memory-management-in-go-stack-heap-garbage-collector/265963594)

3. Garbage collector high‑level design  
- Go uses a concurrent, tri‑color, mark‑and‑sweep garbage collector. [bytegoblin](https://bytegoblin.io/blog/exploring-the-inner-workings-of-garbage-collection-in-golang-tricolor-mark-and-sweep)
- Mark‑and‑sweep: the GC first marks all reachable (live) objects, then sweeps and reclaims unreachable (garbage) objects. [bytegoblin](https://bytegoblin.io/blog/exploring-the-inner-workings-of-garbage-collection-in-golang-tricolor-mark-and-sweep)
- Tri‑color algorithm: objects are classified as white (unknown, potentially garbage), gray (known reachable, not fully scanned), and black (known reachable and fully scanned). [blog.stackademic](https://blog.stackademic.com/basics-of-golang-gc-explained-tri-color-mark-and-sweep-and-stop-the-world-cc832f99164c)
- The GC runs mostly concurrently with the program (mutator), with short stop‑the‑world (STW) pauses at the start and end of collections. [go](https://go.dev/doc/gc-guide)

4. Roots and marking  
- GC starts from a set of roots: global variables, stack variables and pointers on goroutine stacks, and some runtime structures. [deepu](https://deepu.tech/memory-management-in-golang/)
- Initially, all heap objects are white. [bytegoblin](https://bytegoblin.io/blog/exploring-the-inner-workings-of-garbage-collection-in-golang-tricolor-mark-and-sweep)
- The GC finds objects referenced by roots and colors them gray, putting them into a mark queue. [deepu](https://deepu.tech/memory-management-in-golang/)
- A GC worker repeatedly takes gray objects, scans their pointers, colors any referenced white objects gray, and then colors the scanned object black. [blog.stackademic](https://blog.stackademic.com/basics-of-golang-gc-explained-tri-color-mark-and-sweep-and-stop-the-world-cc832f99164c)
- When there are no gray objects left, all reachable objects are black; remaining white objects are unreachable (garbage). [bytegoblin](https://bytegoblin.io/blog/exploring-the-inner-workings-of-garbage-collection-in-golang-tricolor-mark-and-sweep)

5. Sweep phase  
- In the sweep phase, the GC walks heap spans and frees memory for white objects, returning it to the allocator. [github](https://github.com/golang/go/blob/master/src/runtime/mgc.go)
- Sweeping can be concurrent and often happens lazily, span by span, when new allocations need memory. [tip.golang](https://tip.golang.org/src/runtime/mgc.go?m=text)

6. Concurrency, STW, and write barriers  
- To keep latency low, Go’s GC is mostly concurrent but still needs short STW phases to start and end a collection. [go](https://go.dev/doc/gc-guide)
- During concurrent marking, the program continues to mutate pointers, which can break the tri‑color invariants if not handled. [blog.stackademic](https://blog.stackademic.com/basics-of-golang-gc-explained-tri-color-mark-and-sweep-and-stop-the-world-cc832f99164c)
- Go uses a write barrier: on relevant pointer writes, the runtime informs the GC so newly reachable objects are marked correctly. [blog.stackademic](https://blog.stackademic.com/basics-of-golang-gc-explained-tri-color-mark-and-sweep-and-stop-the-world-cc832f99164c)
- This ensures correctness but introduces a small overhead on pointer assignments during GC. [bytegoblin](https://bytegoblin.io/blog/exploring-the-inner-workings-of-garbage-collection-in-golang-tricolor-mark-and-sweep)

7. GOGC and heap growth control  
- `GOGC` is an environment variable that controls how aggressively the GC runs. [tip.golang](https://tip.golang.org/doc/gc-guide)
- It is defined as a percentage of live heap growth: after a GC, if live heap is L and GOGC is G, the next GC is targeted when heap grows to approximately L × (1 + G/100). [tip.golang](https://tip.golang.org/doc/gc-guide)
- Higher GOGC (e.g. 200–500) allows the heap to grow more before GC, reducing GC frequency but using more memory. [go](https://go.dev/doc/gc-guide)
- Lower GOGC (e.g. 50) triggers GC more often, reducing peak memory but increasing GC CPU overhead. [tip.golang](https://tip.golang.org/doc/gc-guide)
- You can set it, for example, with `GOGC=200 ./app` or disable GC with `GOGC=off` for special cases. [go](https://go.dev/doc/gc-guide)

8. Heap layout, spans, and allocation  
- Internally, Go’s runtime organizes heap memory into spans: contiguous chunks that hold objects of specific size classes. [github](https://github.com/golang/go/blob/master/src/runtime/mgc.go)
- Each P has its own local caches for allocations to reduce contention and speed up small allocations. [github](https://github.com/golang/go/blob/master/src/runtime/mgc.go)
- When spans become sufficiently empty during sweep, they can be reused for future allocations. [tip.golang](https://tip.golang.org/src/runtime/mgc.go?m=text)
- Many tiny allocations can stress the allocator and GC, so reducing allocation rate is a key optimization. [github](https://github.com/golang/go/blob/master/src/runtime/mgc.go)

9. Common pitfalls with memory and GC  
- Goroutine leaks: goroutines that never exit, staying blocked on channels or network calls, keep their stacks and referenced heap data alive. [dev](https://dev.to/doziestar/mastering-memory-the-art-of-memory-management-and-garbage-collection-in-go-5292)
- Unbounded maps/slices: growing collections that are never cleared or limited can cause unbounded heap growth. [dev](https://dev.to/doziestar/mastering-memory-the-art-of-memory-management-and-garbage-collection-in-go-5292)
- Dangling slices: holding a small slice pointing into a large backing array prevents the entire array from being collected. [deepu](https://deepu.tech/memory-management-in-golang/)
- Excessive heap allocations: unnecessary boxing, short‑lived heap objects, or frequent interface conversions create GC pressure. [dev](https://dev.to/doziestar/mastering-memory-the-art-of-memory-management-and-garbage-collection-in-go-5292)

10. Best practices to work well with Go’s GC  
- Reduce unnecessary allocations: reuse buffers, consider `sync.Pool` for frequently reused objects, and avoid creating temporary objects in hot loops when possible. [go](https://go.dev/doc/gc-guide)
- Pay attention to escape analysis: use `go build -gcflags="-m"` to see which variables escape to the heap and adjust code if appropriate. [deepu](https://deepu.tech/memory-management-in-golang/)
- Limit lifetime of references: clear map entries and slice elements when no longer needed so references do not keep large structures alive. [dev](https://dev.to/doziestar/mastering-memory-the-art-of-memory-management-and-garbage-collection-in-go-5292)
- Avoid goroutine leaks: always provide cancellation (e.g. `context.Context`, done channels) and ensure goroutines eventually return. [youtube](https://www.youtube.com/watch?v=F4YISvNxn3o)
- Profile memory: use `pprof` and heap profiles to see where allocations happen and what objects are consuming memory. [youtube](https://www.youtube.com/watch?v=F4YISvNxn3o)

11. Mental model summary  
- Stack: fast, per‑goroutine, automatically freed on function return; best for short‑lived, non‑escaping data. [slideshare](https://www.slideshare.net/slideshow/memory-management-in-go-stack-heap-garbage-collector/265963594)
- Heap: shared, garbage‑collected, for values that must outlive current functions or be shared. [slideshare](https://www.slideshare.net/slideshow/memory-management-in-go-stack-heap-garbage-collector/265963594)
- GC: concurrent tri‑color mark‑and‑sweep with short STW phases, guided by GOGC to balance memory usage versus GC CPU cost. [tip.golang](https://tip.golang.org/doc/gc-guide)
- Your job: write code that avoids leaks and unnecessary allocations; Go’s runtime handles the rest efficiently in most cases. [dev](https://dev.to/doziestar/mastering-memory-the-art-of-memory-management-and-garbage-collection-in-go-5292)

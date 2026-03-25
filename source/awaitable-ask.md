# What We Want for I/O in C++

## 1. The Ask

Two things.

**First**, we want standard I/O operations to return IoAwaitable, the protocol defined in [P4003R0](https://isocpp.org/files/papers/P4003R0.pdf) "Coroutines for I/O."

**Second**, we want [P4126R0](https://isocpp.org/files/papers/P4126R0.pdf) "A Universal Continuation Model" - a mechanism that lets a struct produce a `coroutine_handle` without allocating a coroutine frame. This gives sender/receiver pipelines zero-allocation access to every awaitable ever written.

Together, these two changes make coroutines and senders both first-class citizens of the I/O stack.

## 2. Why

`std::execution` provides compile-time sender composition, structured concurrency guarantees, and a customization point model that enables heterogeneous dispatch. These are achievements for real domains - GPU dispatch, work-graph pipelines, heterogeneous execution.

Coroutines serve a different domain. They cannot express compile-time work graphs or target heterogeneous dispatch. What they do is serial byte-oriented I/O - reads, writes, timers, DNS lookups, TLS handshakes - the work that networked applications spend most of their time on. And coroutines bring a property to this domain that senders cannot match.

`await_suspend` takes a `coroutine_handle<>`. The consumer type is already erased. An awaitable's size is known when the stream is constructed - it can be preallocated once and reused across every operation. No per-operation allocation. The benchmark (Appendix A) measures the cost: `capy::any_read_stream` adds 4.8 ns and zero allocations under type erasure.

Sender/receiver's `connect(receiver)` produces an `op_state` whose type depends on both the sender and the receiver. When either side is type-erased, the operation state size is unknown at construction time. It must be heap-allocated per operation. The benchmark measures this too: ~23 ns and one allocation per operation under type erasure. The cost is structural - it follows from the template-dependent nature of `connect`.

Coroutines already pay for the frame allocation. That is the price of compiler-managed state, and it is a fair price. But if I/O operations return senders, coroutines pay the frame allocation and the sender overhead on every I/O call. The entire tax falls on one model.

We propose the alternative: let I/O return awaitables. Coroutines consume them natively - zero additional cost. Senders consume them through the bridge that P4126 enables - at a cost, but a cost that falls on the sender side. Both models pay for what they use.

## 3. What Happens If We Do Not Do This

Without IoAwaitable, standard I/O returns senders. Three consequences follow.

**Coroutines pay a compounding tax.** Every `co_await` of an I/O sender inside `std::execution::task` goes through `connect`/`start`/`state<Rcvr>`. Coroutines already pay for the frame. Stacking sender overhead on top makes them uncompetitive for the workloads they are best at - serial byte-oriented I/O with type-erased streams.

**The teachable model carries a performance penalty.** Coroutines are the natural first contact with asynchrony in C++. A `for` loop that reads from a socket is something a student can follow. The sender equivalent - `repeat_effect_until` composed with `let_value` and `just` - works, but it is not something most developers will write or maintain. If the model that developers can learn carries a penalty, new users conclude that C++ async is slow. That conclusion, once formed, is difficult to reverse.

**The domain partition goes unserved.** Senders excel at compile-time work graphs, structured concurrency, and heterogeneous dispatch. Coroutines excel at serial byte-oriented I/O, type-erased streams, and deep call chains. The current path optimizes one side of this partition and taxes the other. Both models deserve to be optimized for the workloads they serve. Right now, only one of them is.

## 4. Supporting Papers

- [P4126R0](https://isocpp.org/files/papers/P4126R0.pdf) "A Universal Continuation Model" - purely additive; gives senders zero-allocation access to every awaitable ever written
- [P4092R0](https://isocpp.org/files/papers/P4092R0.pdf) "Consuming Senders from Coroutine-Native Code" - the sender-to-awaitable bridge, working implementation
- [P4093R0](https://isocpp.org/files/papers/P4093R0.pdf) "Producing Senders from Coroutine-Native Code" - the awaitable-to-sender bridge, working implementation
- [P4123R0](https://isocpp.org/files/papers/P4123R0.pdf) "The Cost of Senders for Coroutine I/O" - structural cost analysis, every concession granted
- [P4088R0](https://isocpp.org/files/papers/P4088R0.pdf) "The Case for Coroutines" - five properties that make coroutines the right substrate for I/O
- [P4003R0](https://isocpp.org/files/papers/P4003R0.pdf) "Coroutines for I/O" - defines the IoAwaitable protocol
- [P4127R0](https://isocpp.org/files/papers/P4127R0.pdf) "The Coroutine Frame Allocator Timing Problem" - the frame allocator must arrive before the frame exists

## 5. The Prize

The C++ Alliance has built the libraries, written the papers, and found the adopters.

- [P4125R0](https://isocpp.org/files/papers/P4125R0.pdf) "Field Experience: Porting a Derivatives Exchange to Coroutine-Native I/O" - a commercial derivatives exchange is porting from Asio callbacks to coroutine-native I/O. Early results: it works.
- [P4100R0](https://isocpp.org/files/papers/P4100R0.pdf) "The Network Endeavor: Coroutine-Native I/O for C++29" - thirteen papers, two libraries, three independent adopters, and a timeline through 2028.
- [P4048R0](https://isocpp.org/files/papers/P4048R0.pdf) "Networking for C++29: A Call to Action" - a production pipeline, a continuous workflow, and an open invitation to the people whose expertise can make it succeed.

Twenty-one years is long enough.

---

## Appendix A: I/O Read Stream Benchmark

This benchmark is the subject of ongoing optimization. Absolute numbers will change as implementations improve. The fundamental cost asymmetry - per-operation allocation for senders under type erasure, zero for awaitables - is structural and will not change, because sender/receiver's `connect` is template-dependent while `coroutine_handle` is type-erased.

The benchmark source is public: [github.com/sgerbino/capy/.../bench/beman](https://github.com/sgerbino/capy/tree/pr/beman-bench/bench/beman). Anyone is invited to inspect the code, suggest improvements, and help make it better. The architects of P2300 are especially welcome - their expertise would strengthen the comparison.

### Overview

This benchmark compares three execution models for asynchronous I/O across three stream abstraction levels and two I/O return types. Each cell executes 100,000,000 `read_some` calls on a single thread using a no-op stream, isolating execution model overhead from I/O latency.

### Methodology

**Execution models** (one per table):

- **capy::task** - Capy's coroutine task, driven by `capy::thread_pool`. Natively consumes IoAwaitables.
- **beman::execution::task** - Beman's P2300 coroutine task, driven by `sender_thread_pool`. Natively consumes senders.
- **sender/receiver pipeline** - Pure sender pipeline using `repeat_effect_until` + `let_value`. No coroutines. Driven by `sender_thread_pool` via `sync_wait`.

**Stream abstraction levels** (one per row):

- **Native** - Concrete stream type, fully visible to the compiler. No virtual dispatch or type erasure.
- **Abstract** - Virtual base class. The caller sees an interface; the implementation is hidden behind virtual dispatch.
- **Type erased** - Value-type erasure. `capy::any_read_stream` for awaitables (zero steady-state allocation via cached awaitable storage); `sndr_any_read_stream` for senders (heap-allocated stream, sender type erasure via SBO).

**I/O return types** (one per column):

- **Column A** - The execution model's native I/O type. No bridge needed.
- **Column B** - The opposite I/O type, consumed through a bridge. `await_sender` bridges senders into IoAwaitables; `as_sender` bridges IoAwaitables into senders via a synthetic coroutine frame (`frame_cb`).

### Results

#### Table 1: `capy::task`

|                       | A: awaitable (native) |          | B: sender (bridge) |          |
|-----------------------|----------------------:|---------:|--------------------:|---------:|
|                       | ns/op                 | al/op    | ns/op               | al/op    |
| ioaw_read_stream      | 31.5                  | 0        | 75.4                | 1        |
| ioaw_io_read_stream   | 32.5                  | 0        | 99.6                | 2        |
| capy::any_read_stream | 36.3                  | 0        | 99.8                | 2        |

#### Table 2: `beman::execution::task`

|                       | A: sender (native) |          | B: awaitable (bridge) |          |
|-----------------------|-------------------:|---------:|----------------------:|---------:|
|                       | ns/op              | al/op    | ns/op                 | al/op    |
| sndr_read_stream      | 31.8               | 0        | 90.9                  | 1        |
| sndr_io_read_stream   | 55.2               | 1        | 90.8                  | 1        |
| sndr_any_read_stream  | 55.2               | 1        | 98.6                  | 1        |

#### Table 3: sender/receiver pipeline

|                       | A: sender (native) |          | B: awaitable (bridge) |          |
|-----------------------|-------------------:|---------:|----------------------:|---------:|
|                       | ns/op              | al/op    | ns/op                 | al/op    |
| sndr_read_stream      | 33.7               | 0        | 45.6                  | 1        |
| sndr_io_read_stream   | 44.3               | 1        | 45.4                  | 1        |
| sndr_any_read_stream  | 56.2               | 1        | 54.2                  | 1        |

### Analysis

#### Native performance is equivalent

Both execution models achieve ~31-32 ns/op with zero allocations when consuming their native I/O type on a concrete stream. There is no inherent speed advantage to either model at the baseline.

#### Type erasure costs diverge fundamentally

The benchmark's central finding. When the stream is type-erased:

- **capy::any_read_stream** adds 4.8 ns/op and **zero allocations**. The awaitable is preallocated at stream construction and reused across every `read_some` call. This is possible because `await_suspend` takes a type-erased `coroutine_handle<>` - the consumer type is already erased, so the awaitable's size is known at construction time.

- **sndr_io_read_stream** / **sndr_any_read_stream** add ~23 ns/op and **one allocation per operation**. The sender's `connect(receiver)` produces an `op_state` whose type depends on both the sender and the receiver. Since either may be erased, the operation state must be heap-allocated. This allocation cannot be eliminated - it is structural to the sender/receiver model.

The allocation counts tell the story:

| Stream type           | capy::task | bex::task | pipeline |
|-----------------------|-----------:|----------:|---------:|
| Native                |          0 |         0 |        0 |
| Abstract              |          0 |         1 |        1 |
| Type erased           |          0 |         1 |        1 |

Capy achieves zero-allocation type erasure. Sender/receiver cannot.

#### Bridges are expensive but reveal structural costs

Column B measures the tax of consuming the "wrong" I/O type through a bridge:

- **await_sender** (sender to awaitable, used in Table 1 Col B): Adds ~44 ns and 1 allocation per operation. The bridge connects the sender to a `bridge_receiver`, heap-allocates the operation state, and posts the coroutine's continuation back through the executor.

- **as_sender** (awaitable to sender, used in Tables 2-3 Col B): Adds ~59 ns and 1 allocation for bex::task. The bridge constructs a synthetic coroutine frame (`frame_cb`) and threads the executor through `io_env`. For the sender pipeline, the overhead is smaller (~12 ns) because the pipeline avoids coroutine frame allocation.

The bridge allocations are additive with type erasure allocations. In Table 1 Col B, the abstract/type-erased rows show 2 al/op: one from the bridge and one from the sender's type-erased `connect`.

#### The sender pipeline has no looping primitive

The sender/receiver pipeline (Table 3) uses `repeat_effect_until` composed with `let_value` and `just` to implement a loop. This is not part of beman::execution - it was adapted from an example implementation. The loop works without stack overflow because each iteration posts to the executor, but the algorithm machinery (`let_value`, `just`, `repeat_effect_until`) adds overhead compared to a coroutine's native `for` loop.

At the native level, the pipeline (33.7 ns/op) is comparable to the coroutine models (~31-32 ns/op). The gap widens under type erasure because the pipeline's `connect` on each iteration must allocate, whereas a coroutine reuses its frame.

#### What the bridge columns demonstrate

The bridge columns represent the real cost that arises when a library returns one I/O type but the application uses the other execution model. A networking library built on IoAwaitables will pay the `as_sender` tax when consumed from a sender pipeline. Conversely, a sender-based I/O library will pay the `await_sender` tax when consumed from `capy::task`.

The asymmetry in bridge cost reflects a deeper asymmetry in the models:

- **IoAwaitable to sender** (`as_sender`): Uses `frame_cb` to synthesize a coroutine handle. Zero-allocation bridge (the frame is inline in the op_state). The allocation comes from the sender side's `connect`, not from the bridge itself.

- **sender to IoAwaitable** (`await_sender`): Must `connect` the sender to a `bridge_receiver` and heap-allocate the operation state. The allocation is inherent to consuming any sender.

### Trade-off summary

| Property                    | capy IoAwaitable       | P2300 sender/receiver                         |
|-----------------------------|------------------------|-----------------------------------------------|
| Native concrete performance | ~31 ns/op, 0 al/op    | ~32 ns/op, 0 al/op                           |
| Type erasure cost           | +5 ns/op, 0 al/op     | +23 ns/op, 1 al/op                           |
| Type erasure mechanism      | Preallocated awaitable | Heap-allocated op_state                       |
| Why erasure allocates       | It does not            | op_state depends on sender AND receiver types |
| Looping                     | Native `for` loop      | Requires `repeat_effect_until`                |
| Bridge to other model       | ~59 ns/op + 1 al/op   | ~44 ns/op + 1 al/op                          |
| Composability               | Coroutine chains       | Sender algorithm pipelines                    |

# What We Want for I/O in C++

## 1. The Ask

Two things.

**First**, we want standard I/O operations to return IoAwaitable, the protocol defined in [P4003R0](https://isocpp.org/files/papers/P4003R0.pdf) "Coroutines for I/O."

**Second**, we want [P4126R0](https://isocpp.org/files/papers/P4126R0.pdf) "A Universal Continuation Model" - a mechanism that lets a struct produce a `coroutine_handle` without allocating a coroutine frame. This gives sender/receiver pipelines zero-allocation access to every awaitable ever written.

Together, these two changes make coroutines and senders both first-class citizens of the I/O stack.

## 2. Why

`std::execution` provides compile-time sender composition, structured concurrency guarantees, and a customization point model that enables heterogeneous dispatch. These are achievements for real domains - GPU dispatch, work-graph pipelines, heterogeneous execution.

Coroutines serve a different domain. They cannot express compile-time work graphs or target heterogeneous dispatch. What they do is serial byte-oriented I/O - reads, writes, timers, DNS lookups, TLS handshakes - the work that networked applications spend most of their time on. And coroutines bring a property to this domain that senders cannot match.

`await_suspend` takes a `coroutine_handle<>`. The consumer type is already erased. An awaitable's size is known when the stream is constructed - it can be preallocated once and reused across every operation. No per-operation allocation. The benchmark (Appendix A) measures the cost: `capy::any_read_stream` adds ~5 ns and zero allocations under type erasure.

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

This benchmark compares three execution models for asynchronous I/O across three stream abstraction levels and two I/O return types. Each cell executes 100,000,000 `read_some` calls on a single thread using a no-op stream, isolating execution model overhead from I/O latency. Each configuration is measured over 5 independent runs; tables report mean +/- standard deviation.

### Methodology

**Execution models** (one per table):

- **sender/receiver pipeline** - Pure sender pipeline using `repeat_effect_until` + `let_value`. No coroutines. Driven by `sender_thread_pool` via `sync_wait`.
- **capy::task** - Capy's coroutine task, driven by `capy::thread_pool`. Natively consumes IoAwaitables.
- **beman::execution::task** - Beman's P2300 coroutine task [1], driven by `sender_thread_pool`. Natively consumes senders.

**Stream abstraction levels** (one per row):

- **Native** - Concrete stream type, fully visible to the compiler. No virtual dispatch or type erasure.
- **Abstract** - Virtual base class. The caller sees an interface; the implementation is hidden behind virtual dispatch.
- **Type-erased** - Value-type erasure. `capy::any_read_stream` for awaitables (zero steady-state allocation via cached awaitable storage); `sndr_any_read_stream` for senders (heap-allocated stream, sender type erasure via SBO).

**I/O return types** (one per column):

- **Column A** - Awaitable I/O type.
- **Column B** - Sender I/O type.

Each execution model natively consumes one I/O type and bridges the other. The native column is shown in **bold**.

**Thread pools:**

Both thread pools inherit from `boost::capy::execution_context`, providing the same recycling memory resource [8] for coroutine frame allocation. Both use intrusive work queues, mutex + condition variable synchronization, and identical outstanding-work tracking with `std::atomic<std::size_t>` and `memory_order_acq_rel`.

- **capy::thread_pool** - Used in Table 2 Col A. Posts `continuation&` objects via intrusive linked list (zero allocation per post).
- **sender_thread_pool** - Used in all other cells. Posts `work_item*` intrusively when the sender's operation state inherits `work_item` (zero allocation). Posts `coroutine_handle<>` by wrapping in a heap-allocated `coro_work` object when the caller provides a bare handle (one allocation per post via `sender_executor::post`).

The `coro_work` allocation path is used when IoAwaitables post through the executor adapter (Tables 1 and 3 Col A). This is a cross-protocol adaptation cost: the IoAwaitable hands off a `coroutine_handle<>`, and `sender_executor` must wrap it in a `work_item` for its intrusive queue. Real P2300 execution contexts (stdexec's `run_loop`, `static_thread_pool`) use the same intrusive queue pattern internally - the operation state IS the queue node, so no wrapping is needed in the native path [6].

**Allocation tracking:**

Global `operator new`/`operator delete` are overridden to count heap allocations. Both `capy::task` and `beman::task` use Capy's recycling memory resource for coroutine frame allocation. Recycled frames bypass `operator new` after their initial allocation, so coroutine frame costs are amortized to near zero in steady state. The allocation counts reflect per-operation heap allocations only.

**Compiler optimization:**

Each `co_await` suspends the coroutine and posts to the thread pool's work queue, acquiring a mutex, pushing to the intrusive queue, and signaling a condition variable. These are observable side effects that prevent the compiler from eliminating the benchmark loops.

### Bridge Implementations

#### `await_sender` (sender to IoAwaitable)

Used in Table 2 Col B. Wraps a P2300 sender so it can be `co_await`'d inside a `capy::task`.

**Mechanism:** The bridge creates a `sender_awaitable` that placement-constructs the sender's operation state into a stack-allocated buffer. A `bridge_receiver` stores the sender's completion result in a `std::variant` discriminated by completion channel (value, error_code, exception_ptr, stopped).

**Synchronous completion safety:** The bridge uses an `std::atomic<bool>` exchange protocol. Both `await_suspend` (after calling `start()`) and the receiver's completion function call `done_.exchange(true, memory_order_acq_rel)`. Whichever side arrives second (sees `true` from the exchange) is responsible for resuming the coroutine. If the sender completes synchronously during `start()`, `await_suspend` detects this and returns the coroutine handle for symmetric transfer - the coroutine never actually suspends, avoiding stack corruption [5]. This is the same pattern used by beman::execution's `sender_awaitable` [1], which uses `atomic<thread::id>` for the same purpose.

**Result routing:** The bridge inspects the sender's error completion signatures at compile time. If the sender can complete with `set_error(std::error_code)`, `await_resume` returns `io_result<T>` so the error code is a value, not an exception. Otherwise, `await_resume` returns the value directly and rethrows exceptions.

**Zero bridge allocations:** The operation state lives on the coroutine frame (via placement new into a sized buffer). The receiver resumes the coroutine directly - no posting through the executor. The 0 al/op for native senders confirms this.

#### `as_sender` (IoAwaitable to sender)

Used in Tables 1 and 3 Col A. Wraps an IoAwaitable so it can be consumed by the P2300 sender/receiver model.

**Mechanism:** The bridge constructs a synthetic coroutine frame (`frame_cb`) - a 24-byte struct whose first two members (resume/destroy function pointers) match the coroutine frame ABI layout used by MSVC, GCC, and Clang. `coroutine_handle<>::from_address(&cb_)` produces a valid handle whose `.resume()` calls the bridge's completion callback. This avoids allocating an actual coroutine frame, unlike P2300's `connect-awaitable` which creates a bridge coroutine with a heap-allocated frame that is "not generally eligible for the heap-allocation elision optimization (HALO)" [3].

**Executor query:** The bridge obtains a Capy-compatible executor from the P2300 environment using the standard query forwarding mechanism [2, sec. 33.9.4]. It defines a `get_io_executor` query CPO marked as a forwarding query (`forwarding_query(get_io_executor_t{})` returns `true`), ensuring it propagates through sender adapter chains via `FWD-ENV` [2, sec. 33.9.3.5]. Since `starts_on` injects `sched_env<Scheduler>` (which only answers `get_scheduler` and `get_domain`), the bridge queries `get_scheduler(env)` - which IS forwarded - then queries the scheduler itself: `scheduler.query(get_io_executor_t{})`. The scheduler returns a Capy executor by value, which the bridge stores in the operation state. No benchmark-specific types appear in the bridge code.

**`as_awaitable` customization:** The `awaitable_sender` provides an `as_awaitable(Promise&)` member, which is the first-priority dispatch in `[exec.as.awaitable]` [2, sec. 33.9.11.8]. When `co_await`'d inside a `bex::task`, beman's `await_transform` calls this member instead of wrapping the sender in `sender_awaitable`. The member creates a standard awaitable that calls the IoAwaitable's 2-argument `await_suspend(handle, io_env const*)` directly, adapting it to the standard 1-argument protocol. This eliminates a double bridge (IoAwaitable to sender to `sender_awaitable` to awaitable) that would otherwise add connect/start/variant/atomic overhead.

**Completion routing:** The `frame_cb` callback calls `await_resume()` on the IoAwaitable and routes the result through P2300 completion channels based on the return type: `void` to `set_value()`, `error_code` to `set_value()`/`set_error(ec)`, other types to `set_value(T)`.

### Results

All values are mean +/- stddev over 5 runs. **Bold** = native execution model.

#### Table 1: sender/receiver pipeline

|             | A: awaitable (bridge) |       | B: sender (native) |       |
|-------------|----------------------:|------:|--------------------:|------:|
|             | ns/op                 | al/op | ns/op               | al/op |
| Native      | 44.1 +/- 0.1         | 1     | **30.6 +/- 0.1**   | **0** |
| Abstract    | 44.0 +/- 0.1         | 1     | **44.4 +/- 0.0**   | **1** |
| Type-erased | 54.2 +/- 0.1         | 1     | **56.3 +/- 0.1**   | **1** |

#### Table 2: `capy::task`

|             | A: awaitable (native) |       | B: sender (bridge) |       |
|-------------|----------------------:|------:|--------------------:|------:|
|             | ns/op                 | al/op | ns/op               | al/op |
| Native      | **31.4 +/- 0.2**     | **0** | 45.8 +/- 0.3       | 0     |
| Abstract    | **32.1 +/- 0.1**     | **0** | 69.8 +/- 0.1       | 1     |
| Type-erased | **36.4 +/- 0.1**     | **0** | 69.7 +/- 0.2       | 1     |

#### Table 3: `beman::execution::task`

|             | A: awaitable (bridge) |       | B: sender (native) |       |
|-------------|----------------------:|------:|--------------------:|------:|
|             | ns/op                 | al/op | ns/op               | al/op |
| Native      | 39.5 +/- 0.1         | 1     | **30.0 +/- 0.3**   | **0** |
| Abstract    | 39.6 +/- 0.1         | 1     | **53.5 +/- 0.2**   | **1** |
| Type-erased | 44.9 +/- 0.2         | 1     | **53.4 +/- 0.1**   | **1** |

### Analysis

#### Native performance is equivalent

Both execution models achieve ~30-34 ns/op with zero allocations when consuming their native I/O type on a concrete stream. There is no inherent speed advantage to either model at the baseline.

#### Type erasure costs diverge fundamentally

The benchmark's central finding. When the stream is type-erased:

- **capy::any_read_stream** adds ~5 ns/op and **zero allocations**. The awaitable is preallocated at stream construction and reused across every `read_some` call. This is possible because `await_suspend` takes a type-erased `coroutine_handle<>` - the consumer type is already erased, so the awaitable's size is known at construction time.

- **sndr_io_read_stream** / **sndr_any_read_stream** add ~23 ns/op and **one allocation per operation**. The sender's `connect(receiver)` produces an `op_state` whose type depends on both the sender and the receiver. Since either may be erased, the operation state must be heap-allocated. This allocation cannot be eliminated - it is structural to the sender/receiver model [3].

The allocation counts (from the native/bold columns) tell the story:

| Stream type | capy::task | bex::task | pipeline |
|-------------|------------|-----------|----------|
| Native      | 0          | 0         | 0        |
| Abstract    | 0          | 1         | 1        |
| Type-erased | 0          | 1         | 1        |

capy::task is the only solution which offers a zero-allocation path for every stream type.

#### Bridges are competitive

The non-bold column in each table measures the cost of consuming the opposite I/O type through a bridge. Both bridges use universally correct protocols - not optimized for this benchmark's specific senders.

- **await_sender** (sender to IoAwaitable, Table 2 Col B): Adds ~14 ns and **zero allocations** for native senders. The bridge connects the sender to a bridge receiver and uses an atomic exchange protocol to handle synchronous and asynchronous completion uniformly. The receiver resumes the coroutine directly - no posting through the executor. The 1 al/op on abstract/type-erased rows comes from the sender's own type-erased `connect`, not from the bridge.

- **as_sender** (IoAwaitable to sender, Tables 1 and 3 Col A): For `beman::execution::task` (Table 3), the `awaitable_sender`'s `as_awaitable` member lets beman's `await_transform` [2, sec. 33.9.11.8] call the IoAwaitable directly, bypassing beman's `sender_awaitable` wrapping. The overhead is ~10 ns over the native sender path. For the sender pipeline (Table 1), the bridge constructs a synthetic coroutine frame (`frame_cb`) - zero allocation from the bridge itself.

#### Bridge allocation sources

The bridged columns' allocations deserve clarification, as none come from the bridges themselves:

| Cell                     | al/op | Source                                                                    |
|--------------------------|------:|---------------------------------------------------------------------------|
| T1 Col A all rows        | 1     | `sender_executor::post` heap-allocates `coro_work` (executor adapter)     |
| T2 Col B native          | 0     | No allocation anywhere                                                    |
| T2 Col B abstract/erased | 1     | Sender's type-erased `connect` heap-allocates `concrete_op`               |
| T3 Col A all rows        | 1     | `sender_executor::post` heap-allocates `coro_work` (executor adapter)     |

The `coro_work` allocation in Tables 1 and 3 Col A arises because `sender_executor::post(coroutine_handle<>)` must wrap the handle in a `work_item` subclass for the pool's intrusive queue. This is a cross-protocol adaptation cost - the IoAwaitable posts a `coroutine_handle<>`, while the pool's queue accepts `work_item*` nodes. Real P2300 execution contexts avoid this in the native path because the operation state IS the queue node [6]. Capy's `thread_pool` avoids it because `continuation` objects are already intrusive nodes. Neither model inherently requires per-operation allocation for work posting.

#### P2300 bridge asymmetry

P2300 provides asymmetric support for bridging between senders and awaitables [2, 3]:

**Sender to Awaitable:** The `as_awaitable` customization point [2, sec. 33.9.11.8] is the first-priority dispatch when a sender is `co_await`'d. A sender can provide an optimized awaitable representation via a member function, completely bypassing the generic `sender_awaitable` wrapping (connect + start + result variant + atomic). The benchmark's sender streams use this to provide an awaitable that inherits `work_item` and enqueues itself directly - single round-trip, zero allocation. This is a legitimate and expected customization [7].

**Awaitable to Sender:** There is no equivalent customization point. When `connect()` encounters an awaitable, it falls back to `connect-awaitable`, which creates a bridge coroutine with a heap-allocated frame. P2006R1 explicitly notes this frame is "not generally eligible for the heap-allocation elision optimization (HALO)" [3]. Capy's `as_sender` bridge avoids this allocation entirely by using a synthetic `frame_cb` instead of a real coroutine. This is a novel technique not available through P2300's standard mechanisms.

#### The sender pipeline has no looping primitive

The sender/receiver pipeline (Table 1) uses `repeat_effect_until` composed with `let_value` and `just` to implement a loop. This is not part of beman::execution - it was adapted from an example implementation. The loop works without stack overflow because each iteration posts to the executor, but the algorithm machinery adds overhead compared to a coroutine's native `for` loop.

At the native level, the pipeline (30.6 ns/op) is comparable to the coroutine models (~30-31 ns/op). The gap widens under type erasure because the pipeline's `connect` on each iteration must allocate, whereas a coroutine reuses its frame.

#### What the bridge columns demonstrate

The bridged columns represent the real cost that arises when a library returns one I/O type but the application uses the other execution model. A networking library built on IoAwaitables will pay the `as_sender` tax when consumed from a sender pipeline. Conversely, a sender-based I/O library will pay the `await_sender` tax when consumed from `capy::task`.

Both bridges are designed for universal correctness:

- **await_sender** uses an atomic exchange protocol that safely handles senders completing synchronously during `start()`, asynchronously on the same thread, or asynchronously on a different thread.

- **as_sender** uses the P2300 environment query mechanism [2, sec. 33.9.4] to obtain its executor, provides an `as_awaitable` member for coroutine integration [2, sec. 33.9.11.8], and provides a `connect` path for sender pipelines - each using the most efficient mechanism available for that context.

The bridge overhead is modest - both directions add 10-14 ns for native streams with zero bridge allocations. The allocations visible in the bridged columns come from the target model's own machinery (type-erased `connect`, executor adapter posting), not from the bridges.

### Trade-off summary

| Property                       | capy IoAwaitable (Col A)                                      | P2300 sender/receiver (Col B)                   |
|--------------------------------|---------------------------------------------------------------|--------------------------------------------------|
| Native concrete performance    | ~31 ns/op, 0 al/op                                           | ~30 ns/op, 0 al/op                              |
| Type erasure cost              | +5 ns/op, 0 al/op                                            | +23 ns/op, 1 al/op                              |
| Type erasure mechanism         | Preallocated awaitable                                        | Heap-allocated op_state                          |
| Why erasure allocates          | It does not                                                   | op_state depends on sender AND receiver types [3] |
| Looping                        | Native `for` loop                                             | Requires `repeat_effect_until`                   |
| Bridge to other model (native) | ~14 ns/op, 0 al/op                                           | ~14 ns/op, 0 al/op                              |
| Bridge to other model (erased) | ~(-2) ns/op, 1 al/op                                         | ~33 ns/op, 1 al/op                              |
| Sender to awaitable bridge     | Zero-alloc synthetic frame (`frame_cb`)                       | `as_awaitable` customization point [2]           |
| Awaitable to sender bridge     | No standard mechanism; `connect-awaitable` allocates [3]      | N/A (native)                                     |
| Composability                  | Coroutine chains                                              | Sender algorithm pipelines                       |

### References

[1] Beman Project. *execution26: Beman.Execution*. https://github.com/bemanproject/execution

[2] P2300R10. *std::execution*. Niebler, Baker, Hollman, et al. https://wg21.link/P2300

[3] P2006R1. *Eliminating heap-allocations in sender/receiver with connect()/start() as basis operations*. Baker, Niebler, et al. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2006r1.pdf

[4] P3187R1. *Remove ensure_started and start_detached from P2300*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3187r1.pdf

[5] P3552R3. *Add a Coroutine Task Type*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3552r3.html

[6] NVIDIA. *stdexec: NVIDIA's reference implementation of P2300*. https://github.com/NVIDIA/stdexec

[7] C++ Working Draft. *[exec.as.awaitable]*. https://eel.is/c++draft/exec.as.awaitable

[8] P2079R6. *System execution context*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2079r6.html

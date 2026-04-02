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

The benchmark source is public: [github.com/cppalliance/capy/.../bench/beman](https://github.com/cppalliance/capy/tree/develop/bench/beman) [14]. Anyone is invited to inspect the code, suggest improvements, and help make it better. The architects of P2300 are especially welcome - their expertise would strengthen the comparison.

### Overview

This benchmark compares three execution models for asynchronous I/O across three stream abstraction levels and two I/O return types. Each cell executes 20,000,000 `read_some` calls on a single thread using a no-op stream, isolating execution model overhead from I/O latency. Each configuration is measured over 5 independent runs preceded by a warmup pass; tables report mean +/- standard deviation.

### Methodology

**Execution models** (one per table):

- **sender/receiver pipeline** - Pure sender pipeline using `repeat_effect_until` + `let_value`. No coroutines. Driven by `sender_thread_pool` via `sync_wait`.
- **capy::task** - Capy's coroutine task, driven by `capy::thread_pool`. Natively consumes IoAwaitables.
- **beman::execution::task** - Beman's P2300 coroutine task [1], driven by `sender_thread_pool`. Natively consumes senders. **Note:** `bex::task`'s `await_transform` checks `as_awaitable` on the sender (first-priority dispatch per [exec.as.awaitable]). When the sender provides an `as_awaitable` member - as the benchmark's `sndr_read_stream` does - the task calls it directly, bypassing `connect`/`start` entirely. Table 3's native sender column (Col B) therefore measures the `as_awaitable` path, not the full sender protocol. This is the best-case scenario for senders in coroutines. See *Table 3 and `as_awaitable`* in the Analysis section for implications.

**Stream abstraction levels** (one per row):

- **Native** - Concrete stream type, fully visible to the compiler. No virtual dispatch or type erasure.
- **Abstract** - Virtual base class. The caller sees an interface; the implementation is hidden behind virtual dispatch.
- **Type-erased** - Value-type erasure. `capy::any_read_stream` for awaitables (zero steady-state allocation via cached awaitable storage); `sndr_any_read_stream` for senders (heap-allocated stream, sender type erasure via SBO).

**I/O return types** (one per column):

- **Column A** - Awaitable I/O type.
- **Column B** - Sender I/O type.

Each execution model natively consumes one I/O type and bridges the other. The native column is shown in **bold**.

**Thread pools:**

Both thread pools inherit from `boost::capy::execution_context`, providing the same recycling memory resource for coroutine frame allocation. Both use intrusive work queues, mutex + condition variable synchronization, and identical outstanding-work tracking with `std::atomic<std::size_t>` and `memory_order_acq_rel`.

- **capy::thread_pool** - Used in Table 2 Col A. Posts `continuation&` objects via intrusive linked list (zero allocation per post).
- **sender_thread_pool** - Used in all other cells. Posts `work_item*` intrusively when the sender's operation state inherits `work_item` (zero allocation). Has no `post(coroutine_handle<>)` - P2300 execution contexts only expose `schedule()`, which returns a sender. To resume a coroutine on the scheduler, the caller must go through `schedule()`, `connect()`, `start()`, heap-allocating the operation state (one allocation per post).

The `schedule`/`connect`/`start` allocation path is used when IoAwaitables post through the executor adapter (Tables 1 and 3 Col A). This is a cross-protocol adaptation cost: the IoAwaitable produces a `coroutine_handle<>`, but P2300 has no way to accept a bare handle. The adapter must create a `scheduled_resume` operation state - `connect(schedule(sched), resume_receiver)` - and heap-allocate it because the coroutine is suspended and cannot host it. The operation state IS the queue node (inherits `work_item`), so no additional wrapping is needed, but the allocation is unavoidable. Real P2300 execution contexts (stdexec's `run_loop`, `static_thread_pool`) use the same intrusive queue pattern [6].

**Operation state recycling:**

Type-erased senders allocate their operation state (`concrete_op`) via `op_base::operator new`, which is overridden to use the same recycling memory resource used for coroutine frames. After warmup, these allocations are served from a thread-local free list in O(1) without calling global `operator new`. Both the coroutine frame recycler and the op_state recycler use the same `boost::capy::get_recycling_memory_resource()`, providing equivalent amortized allocation cost. The recycling allocator is functionally equivalent to what P3433R1 [9] proposes for allocator support in operation states.

This means the benchmark shows both models at their best: coroutine frames are recycled (standard practice for coroutine-based systems), and sender operation states are recycled (the strongest available mitigation for the structural allocation). The remaining performance differences reflect irreducible overhead - allocator fast-path cost, factory dispatch, virtual calls - not allocation policy. The al/op counts in the tables reflect allocation *calls* (including recycled), not global heap hits, so the structural allocation demand is visible even when the recycler eliminates the malloc cost.

**Allocation tracking:**

All allocation paths go through a single counter. Global `operator new` increments `g_alloc_count` before calling `malloc`. The recycling memory resource is wrapped in a `counting_memory_resource` proxy that increments the same counter before delegating - both for type-erased sender operation states (`op_base::operator new`) and for coroutine frame allocations (`polymorphic_allocator` passed to `bex::task`). This means al/op reflects *allocation calls per operation* regardless of whether they hit the global heap or the recycler's free list. The counter measures structural allocation demand, not allocation policy.

**Warmup:**

The first complete pass through all cells is a warmup (results discarded). This eliminates instruction cache, branch predictor, and CPU frequency scaling effects from the first execution model measured. The 5 measured runs begin from a thermally stable state.

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

**Mechanism:** The bridge constructs a synthetic coroutine frame (`frame_cb`) - a 24-byte struct whose first two members (resume/destroy function pointers) match the coroutine frame ABI layout used by MSVC, GCC, and Clang. `coroutine_handle<>::from_address(&cb_)` produces a valid handle whose `.resume()` calls the bridge's completion callback. This avoids allocating an actual coroutine frame, unlike P2300's `connect-awaitable` which creates a bridge coroutine with a heap-allocated frame that is "not generally eligible for the heap-allocation elision optimization (HALO)" [3]. P4126R0 [10] proposes standardizing this technique as a "universal continuation model."

**Executor query:** The bridge obtains a Capy-compatible executor from the P2300 environment using the standard query forwarding mechanism [2, sec. 33.9.4]. It defines a `get_io_executor` query CPO marked as a forwarding query (`forwarding_query(get_io_executor_t{})` returns `true`), ensuring it propagates through sender adapter chains via `FWD-ENV` [2, sec. 33.9.3.5]. Since `starts_on` injects `sched_env<Scheduler>` (which only answers `get_scheduler` and `get_domain`), the bridge queries `get_scheduler(env)` - which IS forwarded - then queries the scheduler itself: `scheduler.query(get_io_executor_t{})`. The scheduler returns a Capy executor by value, which the bridge stores in the operation state. No benchmark-specific types appear in the bridge code.

**`as_awaitable` customization:** The `awaitable_sender` provides an `as_awaitable(Promise&)` member, which is the first-priority dispatch in `[exec.as.awaitable]` [2, sec. 33.9.11.8]. When `co_await`'d inside a `bex::task`, beman's `await_transform` calls this member instead of wrapping the sender in `sender_awaitable`. The member creates a standard awaitable that calls the IoAwaitable's 2-argument `await_suspend(handle, io_env const*)` directly, adapting it to the standard 1-argument protocol. This eliminates a double bridge (IoAwaitable to sender to `sender_awaitable` to awaitable) that would otherwise add connect/start/variant/atomic overhead.

**Completion routing:** The `frame_cb` callback calls `await_resume()` on the IoAwaitable and routes the result through P2300 completion channels based on the return type: `void` to `set_value()`, `error_code` to `set_value()`/`set_error(ec)`, other types to `set_value(T)`.

### Results

All values are mean +/- stddev over 5 runs (warmup excluded). **Bold** = native execution model. al/op counts allocation calls per operation, including recycled allocations. The bridge column (Col A) in Tables 1 and 3 shows 1 al/op - the `scheduled_resume` operation state when IoAwaitables post through `schedule()`, `connect()`, `start()`.

#### Table 1: sender/receiver pipeline

|                | A: awaitable (bridge) |          | B: sender (native)    |          |
|----------------|----------------------:|---------:|----------------------:|---------:|
|                | ns/op                 | al/op    | ns/op                 | al/op    |
| Native         | 46.0 +/- 0.1         | 1        | **32.8 +/- 0.2**     | **0**    |
| Abstract       | 46.0 +/- 0.1         | 1        | **45.1 +/- 0.2**     | **1**    |
| Type-erased    | 55.7 +/- 0.2         | 1        | **57.1 +/- 0.3**     | **1**    |
| Synchronous    | N/A                   |          | N/A                   |          |

#### Table 2: `capy::task`

|                | A: awaitable (native)  |          | B: sender (bridge) |          |
|----------------|-----------------------:|---------:|--------------------:|---------:|
|                | ns/op                  | al/op    | ns/op               | al/op    |
| Native         | **31.4 +/- 0.0**      | **0**    | 48.3 +/- 0.5       | 0        |
| Abstract       | **32.4 +/- 0.4**      | **0**    | 72.1 +/- 0.1       | 1        |
| Type-erased    | **36.3 +/- 0.0**      | **0**    | 72.3 +/- 0.1       | 1        |
| Synchronous    | **1.0 +/- 0.1**       | **0**    | 19.7 +/- 0.2       | 0        |

#### Table 3: `beman::execution::task`

|                | A: awaitable (bridge) |          | B: sender (native)    |          |
|----------------|----------------------:|---------:|----------------------:|---------:|
|                | ns/op                 | al/op    | ns/op                 | al/op    |
| Native         | 43.5 +/- 0.2         | 1        | **32.2 +/- 0.2**     | **0**    |
| Abstract       | 43.3 +/- 0.3         | 1        | **55.3 +/- 0.2**     | **1**    |
| Type-erased    | 48.8 +/- 0.1         | 1        | **55.3 +/- 0.2**     | **1**    |
| Synchronous    | 2.7 +/- 0.3          | 0        | **1.1 +/- 0.3**      | **0**    |

### Analysis

#### Native performance is equivalent

Both execution models achieve ~31-33 ns/op with zero allocations when consuming their native I/O type on a concrete stream. There is no inherent speed advantage to either model at the baseline.

#### Type erasure costs diverge

- **capy::any_read_stream** (type-erased awaitable): **36.3 ns/op, 0 al/op**. The awaitable is preallocated at stream construction and reused across every `read_some` call. No allocator path is invoked per operation - placement construct into existing storage.

- **sndr_any_read_stream** (type-erased sender): **55.3-57.1 ns/op, 1 al/op**. Each operation traverses the recycling allocator fast path (TLS lookup, size-class bucketing, free-list pop/push), the factory lambda, `concrete_op` construction/destruction, virtual `start()`/`execute()` dispatch, and `unique_ptr` management.

The ~19-21 ns gap and the 1 al/op difference are irreducible with the current sender/receiver architecture. The allocation call represents the minimum structural cost of the `connect`/`start` protocol under type erasure: the operation state's type is erased, so it must be dynamically allocated - even with a recycling allocator.

The allocation counts (from the native/bold columns):

| Stream type           | pipeline | capy::task | bex::task |
|-----------------------|---------:|-----------:|----------:|
| Native                | 0        | 0          | 0         |
| Abstract              | 1        | 0          | 1         |
| Type-erased           | 1        | 0          | 1         |

The IoAwaitable column (capy::task) shows 0 al/op at all abstraction levels. The sender columns show 1 al/op once the stream is abstracted - the type-erased `concrete_op` allocation that the recycler serves from its free list.

#### Bridges are competitive

The non-bold column in each table measures the cost of consuming the opposite I/O type through a bridge. Both bridges use universally correct protocols - not optimized for this benchmark's specific senders.

- **await_sender** (sender to IoAwaitable, Table 2 Col B): Adds ~17 ns and **zero allocations** for native senders. The bridge connects the sender to a bridge receiver and uses an atomic exchange protocol to handle synchronous and asynchronous completion uniformly. The receiver resumes the coroutine directly - no posting through the executor. Abstract and type-erased senders show 1 al/op - the type-erased `concrete_op` allocation from the sender side, not the bridge.

- **as_sender** (IoAwaitable to sender, Tables 1 and 3 Col A): For `beman::execution::task` (Table 3), the `awaitable_sender`'s `as_awaitable` member lets beman's `await_transform` [2, sec. 33.9.11.8] call the IoAwaitable directly, bypassing beman's `sender_awaitable` wrapping. The overhead is ~11 ns over the native sender path. For the sender pipeline (Table 1), the bridge constructs a synthetic coroutine frame (`frame_cb`). Both paths incur 1 al/op from the `scheduled_resume` operation state required by P2300's `schedule()`, `connect()`, `start()` protocol.

#### The bridged awaitable outperforms native senders under abstraction

In Table 3 (`beman::execution::task`), the bridged awaitable column (Col A) is **faster** than the native sender column (Col B) for abstract and type-erased streams:

- Table 3 abstract: awaitable bridge 43.3 ns (1 al/op) vs sender native **55.3 ns (1 al/op)**
- Table 3 type-erased: awaitable bridge 48.8 ns (1 al/op) vs sender native **55.3 ns (1 al/op)**

Both sides now show 1 al/op at the abstract/type-erased level, but the awaitable bridge is still 7-12 ns faster. The bridged awaitable's performance is remarkably flat across abstraction levels (43.5/43.3/48.8 ns), while the native sender jumps sharply from 32.2 ns (native) to 55.3 ns (abstract). This occurs because the bridge cost is constant - the IoAwaitable's `await_suspend` always follows the same path regardless of stream abstraction - while the sender model's virtual dispatch and type erasure machinery scale with abstraction level.

In Table 1 (sender pipeline), the native sender is slightly slower: bridge 55.7 ns vs native 57.1 ns at the type-erased level - both at 1 al/op.

#### P2300 bridge asymmetry

P2300 provides asymmetric support for bridging between senders and awaitables [2, 3]:

**Sender to Awaitable:** The `as_awaitable` customization point [2, sec. 33.9.11.8] is the first-priority dispatch when a sender is `co_await`'d. A sender can provide an optimized awaitable representation via a member function, completely bypassing the generic `sender_awaitable` wrapping (connect + start + result variant + atomic). The benchmark's sender streams use this to provide an awaitable that inherits `work_item` and enqueues itself directly - single round-trip, zero allocation. This is a legitimate and expected customization [7].

**Awaitable to Sender:** There is no equivalent customization point. When `connect()` encounters an awaitable, it falls back to `connect-awaitable`, which creates a bridge coroutine with a heap-allocated frame. P2006R1 explicitly notes this frame is "not generally eligible for the heap-allocation elision optimization (HALO)" [3]. Capy's `as_sender` bridge avoids this allocation entirely by using a synthetic `frame_cb` instead of a real coroutine. P4126R0 [10] proposes standardizing this technique.

#### Table 3 and `as_awaitable`

Table 3's native sender column (Col B) benefits from `bex::task`'s `as_awaitable` dispatch. When a sender provides an `as_awaitable` member, `bex::task`'s `await_transform` calls it directly - the sender's `connect` and `start` methods are never invoked. The benchmark's `sndr_read_stream::read_sender` provides exactly this: an `as_awaitable` that returns a lightweight `work_item` awaitable, identical in cost to the IoAwaitable path.

This explains why Table 3 native sender (32.2 ns/op) matches Table 2 native awaitable (31.4 ns/op) - both are measuring the awaitable path, not the sender protocol.

The existing P2300 networking implementation in beman::net [13] does not use `bex::task`. Its examples use a custom `demo::task` whose `await_transform` always creates a `sender_awaiter` that calls `connect` + `start` - with no `as_awaitable` check. Every `co_await net::async_receive(...)` in beman::net pays the full sender protocol cost. For beman::net users, Table 1 (sender pipeline) is more representative of actual per-operation overhead than Table 3.

Senders that do not provide `as_awaitable` - which includes most senders produced by P2300 algorithms like `let_value`, `then`, `when_all`, etc. - also go through the full `connect`/`start` path in `bex::task` via its generic `sender_awaitable` bridge. The `as_awaitable` optimization is only available to leaf senders that implement it explicitly.

#### Synchronous completions: the hot path senders cannot optimize

In real networking I/O, many operations complete without waiting for the kernel: reads from a socket with data already in the receive buffer, writes to a non-full send buffer, DNS cache hits, TLS session resumptions, io_uring completions already batched in the completion queue. In a high-throughput server, this is the common case - a busy connection often has data waiting before the application reads it.

The Synchronous row measures this scenario. The I/O operation completes immediately - no executor posting, no thread pool round-trip.

|                | capy::task (awaitable) | beman::task (sender via as_awaitable) | sender pipeline |
|----------------|:----------------------:|:-------------------------------------:|:---------------:|
| Synchronous    | 1.0 ns/op              | 1.1 ns/op                             | N/A             |

Both coroutine models achieve ~1 ns/op through symmetric transfer - `await_suspend` returns the coroutine handle and the compiler performs a tail call. The stack stays flat regardless of how many operations complete synchronously in sequence.

The sender/receiver pipeline cannot run this benchmark. `start()` is void - it cannot signal synchronous completion. The only way to deliver a result is through the receiver (`set_value`), which recurses into the next iteration's `connect`/`start`. Without a trampoline, this causes stack overflow. A trampoline requires fundamental changes to every loop algorithm (`repeat_effect_until`, `retry`, `for_each`) and interacts poorly with sender composition (`let_value`, `starts_on`). The benchmark's attempt to add a trampoline to `repeat_effect_until` resulted in an infinite loop when composed with `let_value` and `starts_on`.

Coroutines handle synchronous completions for free through two mechanisms, neither of which has a sender equivalent:

- **`await_ready`** - The awaitable can perform the I/O (e.g., `recvmsg`) in `await_ready` and return `true` if data is available. The coroutine never suspends - no handle manipulation, no symmetric transfer, no atomic exchange. This is the fastest possible path for inline completions. A sender cannot do this because `start()` is called inside `await_suspend`, after the coroutine has already suspended. The work cannot be moved earlier.

- **`await_suspend` return value** - If `await_ready` returns `false`, `await_suspend` can still complete the I/O and return the coroutine handle for symmetric transfer. The compiler performs a tail call - the stack stays flat regardless of how many operations complete synchronously in sequence.

The sender model would need an equivalent mechanism (a way for `start()` to indicate "completed synchronously, here is the result"), which does not exist in P2300 and would be a fundamental change to the operation state protocol.

#### Compile-time safety

The IoAwaitable protocol's 2-argument `await_suspend(coroutine_handle<>, io_env const*)` structurally enforces that the execution environment is provided at suspension time. The dependency is in the function signature - the compiler rejects any call site that does not provide it.

In the sender/receiver model, environment availability is checked when a sender queries the receiver's environment inside `start()`. This check IS compile-time (it fails template instantiation if the query is unsupported), but it is opt-in: each sender must explicitly constrain its `connect` method. If the sender author forgets the constraint, the error appears as a deep template instantiation failure rather than a clear signature mismatch. P3164R4 [11] and P3557R2 [12] are addressing diagnostic quality for these errors but are not yet part of the C++26 standard.

#### The sender pipeline has no looping primitive

The sender/receiver pipeline (Table 1) uses `repeat_effect_until` composed with `let_value` and `just` to implement a loop. This is not part of beman::execution - it was adapted from an example implementation. The loop works without stack overflow because each iteration posts to the executor, but the algorithm machinery adds overhead compared to a coroutine's native `for` loop.

At the native level, the pipeline (32.8 ns/op) is comparable to the coroutine models (~31-32 ns/op). The gap widens under type erasure because the pipeline's `connect` on each iteration traverses the factory + allocator path (1 al/op), whereas a coroutine reuses its frame (0 al/op).

#### What the bridge columns demonstrate

The bridged columns represent the real cost that arises when a library returns one I/O type but the application uses the other execution model. A networking library built on IoAwaitables will pay the `as_sender` tax when consumed from a sender pipeline. Conversely, a sender-based I/O library will pay the `await_sender` tax when consumed from `capy::task`.

Both bridges are designed for universal correctness:

- **await_sender** uses an atomic exchange protocol that safely handles senders completing synchronously during `start()`, asynchronously on the same thread, or asynchronously on a different thread.

- **as_sender** uses the P2300 environment query mechanism [2, sec. 33.9.4] to obtain its executor, provides an `as_awaitable` member for coroutine integration [2, sec. 33.9.11.8], and provides a `connect` path for sender pipelines - each using the most efficient mechanism available for that context.

The bridge overhead is modest - both directions add 11-17 ns for native streams. The `await_sender` bridge (Table 2 Col B) incurs zero allocation calls for native senders; the `as_sender` bridge (Tables 1 and 3 Col A) incurs 1 al/op from the `scheduled_resume` operation state required by P2300's `schedule()`, `connect()`, `start()` protocol.

#### Scope and limitations

This benchmark measures per-operation overhead for sequential I/O in a tight loop. It does not measure:

- **Concurrent composition** - `when_all` over N streams, fan-out patterns.
- **Real I/O latency** - io_uring submit/complete cycles, network round-trips.
- **Multi-threaded work distribution** - cross-thread scheduling, work stealing, NUMA-aware dispatch.
- **Compile time and diagnostic quality** - template instantiation depth, error message clarity.

### Trade-off summary

| Property                          | capy IoAwaitable                      | P2300 sender/receiver                         |
|-----------------------------------|---------------------------------------|-----------------------------------------------|
| Native concrete performance       | ~31 ns/op, 0 al/op                   | ~32 ns/op, 0 al/op                           |
| Type erasure cost (with recycler) | +5 ns/op, 0 al/op                    | +23-24 ns/op, 1 al/op                        |
| Type erasure mechanism            | Preallocated awaitable                | Recycled op_state (factory + virtual dispatch)|
| Why the gap persists              | No allocator path, no allocation call | Allocator fast path + factory + unique_ptr [3]|
| Synchronous completion            | ~1 ns/op (symmetric transfer)         | N/A (stack overflow without trampoline)       |
| Inline completion (await_ready)   | I/O in `await_ready`, no suspend      | No equivalent; `start()` is post-suspend      |
| Looping                           | Native `for` loop                     | Requires `repeat_effect_until`                |
| Bridge to other model (native)    | ~10-11 ns/op, 1 al/op                | ~17 ns/op, 0 al/op                           |
| Bridge to other model (erased)    | Faster in bex::task, equal in pipeline| ~36 ns/op, 1 al/op                           |
| Sender to awaitable bridge        | Zero-alloc synthetic frame (`frame_cb`) [10] | `as_awaitable` customization point [2] |
| Awaitable to sender bridge        | No standard mechanism; `connect-awaitable` allocates [3] | N/A (native)           |
| `as_awaitable` bypass             | N/A (native protocol)                 | Only leaf senders with explicit member [7]    |
| Compile-time env safety           | Structural (in function signature)    | Opt-in (per-sender constraint) [11, 12]       |
| Composability                     | Coroutine chains (`when_all`, `when_any`, `timeout`) | Sender algorithm pipelines   |

### References

[1] Beman Project. *execution26: Beman.Execution*. https://github.com/bemanproject/execution

[2] P2300R10. *std::execution*. Niebler, Baker, Hollman, et al. https://wg21.link/P2300

[3] P2006R1. *Eliminating heap-allocations in sender/receiver with connect()/start() as basis operations*. Baker, Niebler, et al. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2006r1.pdf

[4] P3187R1. *Remove ensure_started and start_detached from P2300*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3187r1.pdf

[5] P3552R3. *Add a Coroutine Task Type*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3552r3.html

[6] NVIDIA. *stdexec: NVIDIA's reference implementation of P2300*. https://github.com/NVIDIA/stdexec

[7] C++ Working Draft. *[exec.as.awaitable]*. https://eel.is/c++draft/exec.as.awaitable

[8] P2079R6. *System execution context*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2079r6.html

[9] P3433R1. *Allocator Support for Operation States*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3433r1.pdf

[10] P4126R0. *A Universal Continuation Model*. https://isocpp.org/files/papers/P4126R0.pdf

[11] P3164R4. *Early Diagnostics for Sender Expressions*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3164r4.html

[12] P3557R2. *High-Quality Sender Diagnostics with Constexpr Exceptions*. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3557r2.html

[13] Beman Project. *net: Beman.Net - P2300-based networking*. https://github.com/bemanproject/net

[14] Gerbino, S. *I/O Read Stream Benchmark source*. https://github.com/cppalliance/capy/tree/develop/bench/beman

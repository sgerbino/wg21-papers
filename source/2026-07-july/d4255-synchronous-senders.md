---
title: "Awaitables And Senders For Synchronous I/O"
document: P4255R0
date: 2026-07-01
intent: info
audience: SG1, LEWG
reply-to:
  - "Vinnie Falco <vinnie.falco@gmail.com>"
---

## Abstract

The awaitable protocol handles synchronous I/O at zero cost. The sender protocol does not.

C++20 awaitables provide two mechanisms for synchronous I/O: recompilation (swap the awaitable type, the algorithm goes from async to sync) and relinking (compile once against a type-erased stream, swap the object file). Both work today. This paper implements the simplest possible synchronous write operation under both models, grants senders every affordance, and sets the two implementations side by side.

---

## Revision History

### R0: July 2026 (post-Brno mailing)

- Initial revision.

---

## 1. Disclosure

The author provides information and serves at the pleasure of the committee.

The author developed and maintains [Capy](https://github.com/cppalliance/capy)<sup>[1]</sup> and [Corosio](https://github.com/cppalliance/corosio)<sup>[2]</sup>, coroutine-native I/O libraries under the C++ Alliance. The author has a stake in the coroutine model's adoption.

Coroutine-native I/O cannot express compile-time work graphs. This is a genuine limitation.

This paper asks for nothing.

## 2. The Abstraction

A synchronous write stream has one operation: accept a string and store it. No error codes, no byte counts, no partial writes. Two concrete types implement it.

`string_sink` appends to a `std::string`. The operation is synchronous. The data is already in memory. No kernel transition occurs.

```cpp
class string_sink
{
    std::string& out_;

public:
    explicit string_sink(std::string& s)
        : out_(s) {}

    auto write(std::string_view sv)
    {
        out_.append(sv.data(), sv.size());
        // returns an awaitable or sender
    }
};
```

`tcp_sink` writes to a TCP socket. The operation is asynchronous. The kernel accepts the data, the coroutine suspends, the reactor resumes it when the write completes.

Both expose the same `write(std::string_view)` signature. The return type differs. The algorithm that calls `co_await sink.write(...)` does not.

## 3. Recompilation

A generic algorithm written as a coroutine template:

```cpp
template<class Sink>
task<> log_lines(Sink& sink,
    std::span<std::string_view> lines)
{
    for (auto line : lines)
        co_await sink.write(line);
}
```

Compile against `tcp_sink`. The awaitable returned by `write` suspends. The reactor resumes. The algorithm is asynchronous.

Recompile against `string_sink`. The awaitable returned by `write` has `await_ready() == true`. No suspension occurs. The algorithm is synchronous.

The source is identical. The awaitable type varies. The execution model is selected at compile time.

## 4. Relinking

The same algorithm compiled once against a type-erased stream:

```cpp
class write_stream
{
public:
    virtual ~write_stream() = default;
    virtual /* IoAwaitable */
        write(std::string_view sv) = 0;
};

task<> log_lines(write_stream& sink,
    std::span<std::string_view> lines)
{
    for (auto line : lines)
        co_await sink.write(line);
}
```

The algorithm's object code is fixed. It does not know whether the stream is synchronous or asynchronous. It does not need to know.

Link against an object file that provides `tcp_sink` behind the vtable. The algorithm is asynchronous.

Link against a different object file that provides `string_sink` behind the vtable. The algorithm is synchronous.

No recompilation. One indirect call per write. Zero allocation per write.

**The algorithm was compiled once. The execution model was chosen by the linker.**

## 5. What Senders Provide

Before examining the sender path for synchronous I/O, three genuine achievements of `std::execution` deserve recognition.

**Zero-allocation composition.** Sender pipelines collapse into a single `operation_state` at compile time. No heap allocation, no virtual dispatch, no reference counting. This is a real property that coroutines do not match for multi-stage pipelines.<sup>[3]</sup>

**Compile-time work graphs.** The sender algebra encodes DAGs of work at the type level. `when_all`, `then`, `let_value` compose into a static structure the optimizer can see through. Domain customization via `transform_sender` retargets the same graph to CPU or GPU by swapping the scheduler.<sup>[4]</sup>

**Structured concurrency.** `counting_scope` tracks dynamically spawned work and prevents scope destruction until all work completes.<sup>[3]</sup>

The comparison that follows grants senders every affordance: an `inline_scheduler`-scheduled task so `await_transform` applies no affinity wrap, synchronous completion inside `start`, and the minimal `completion_signatures<set_value_t()>`. No conforming task implementation can avoid this path; the sender protocol imposes it.

## 6. The Sender Path

`string_sink::write` returns a sender:

```cpp
class string_sink
{
    std::string& out_;

public:
    explicit string_sink(std::string& s)
        : out_(s) {}

    auto write(std::string_view sv)
    {
        out_.append(sv.data(), sv.size());
        return write_sender{};
    }

private:
    struct write_sender
    {
        using sender_concept =
            std::execution::sender_t;
        using completion_signatures =
            std::execution::completion_signatures<
                std::execution::set_value_t()>;

        template<class Receiver>
        struct state
        {
            using operation_state_concept =
                std::execution::
                    operation_state_t;
            Receiver rcvr_;

            void start() & noexcept
            {
                std::execution::set_value(
                    std::move(rcvr_));
            }
        };

        template<class Receiver>
        state<std::decay_t<Receiver>>
            connect(Receiver&& rcvr) const
        {
            return {std::forward<Receiver>(
                rcvr)};
        }
    };
};
```

The sender completes synchronously. `start` calls `set_value` on the receiver immediately. No kernel transition. No suspension on the sender side. This is the most favorable implementation possible.

A coroutine returning `execution::task` consumes it:

```cpp
execution::task<> log_lines(
    string_sink& sink,
    std::span<std::string_view> lines)
{
    for (auto line : lines)
        co_await sink.write(line);
}
```

The analysis below takes the task's scheduler to be `inline_scheduler`. A task's default scheduler is `task_scheduler`,<sup>[5]</sup> under which `await_transform` wraps each awaited sender in `affine_on`; `inline_scheduler` is the configuration that grants the sender its most favorable synchronous path, with no affinity wrap.

What happens inside `co_await sink.write(line)`, per the specification:

1. `await_transform` receives the sender.<sup>[5]</sup> The promise type's `await_transform` is constrained to `sender`.<sup>[5]</sup> `[task.promise]` p10. Because the task's scheduler is `inline_scheduler`, `await_transform` forwards the sender to `as_awaitable` directly; with a non-inline scheduler it would first wrap the sender in `affine_on`.<sup>[6]</sup> `[exec.affine.on]`.

2. `as_awaitable` checks the sender for a member `as_awaitable`; this sender provides none, so it constructs a `sender-awaitable`.<sup>[3]</sup> `[exec.as.awaitable]`.

3. The `sender-awaitable` constructor calls `connect(sndr, awaitable-receiver)`.<sup>[3]</sup> The operation state is materialized. The receiver is wired.

4. `await_ready()` returns `false`.<sup>[3]</sup> Unconditionally. The coroutine suspends.

5. `await_suspend` calls `start(state)`.<sup>[3]</sup> Inside `start`, `set_value(receiver)` fires synchronously.

6. The receiver stores the result in a `variant` and calls `.resume()` on the coroutine handle.<sup>[3]</sup> The coroutine resumes.

7. `await_resume()` extracts the value from the `variant`.<sup>[3]</sup>

Seven protocol steps. One suspension and one resumption. One operation state construction. One receiver instantiation. One `variant` emplacement. To append bytes to a string.

## 7. The Awaitable Path

`string_sink::write` returns an IoAwaitable:

```cpp
class string_sink
{
    std::string& out_;

public:
    explicit string_sink(std::string& s)
        : out_(s) {}

    auto write(std::string_view sv)
    {
        out_.append(sv.data(), sv.size());
        return immediate{};
    }

private:
    struct immediate
    {
        bool await_ready() const noexcept
        {
            return true;
        }

        void await_suspend(
            std::coroutine_handle<>,
            io_env const*) noexcept
        {
        }

        void await_resume() noexcept {}
    };
};
```

A coroutine returning a task type that satisfies the IoAwaitable protocol<sup>[7]</sup> consumes it:

```cpp
task<> log_lines(
    string_sink& sink,
    std::span<std::string_view> lines)
{
    for (auto line : lines)
        co_await sink.write(line);
}
```

What happens inside `co_await sink.write(line)`:

1. `await_transform` delegates to `transform_awaitable`, which wraps the IoAwaitable in a `transform_awaiter`.<sup>[1]</sup>

2. `await_ready()` returns `true`. The coroutine does not suspend.

3. `await_resume()` returns.

Three protocol steps. No suspension. No operation state. No receiver. No `variant`. The bytes were appended to the string.

## 8. Comparison

| Property | Awaitable | Sender |
| --------------------------------- | --------- | ------ |
| Protocol steps per write | 3 | 7 |
| Coroutine suspensions | 0 | 1 |
| Coroutine resumptions | 0 | 1 |
| Operation state constructions | 0 | 1 |
| Receiver instantiations | 0 | 1 |
| `variant` emplacements | 0 | 1 |
| Type erasure cost | 1 vtable call, 0 allocations | `any_sender`: 0-1 allocations |

## 9. Interoperation

The awaitable protocol and the sender protocol are not mutually exclusive. An IoAwaitable can be wrapped as a sender and consumed by sender pipelines.

[P4093R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4093r1.pdf)<sup>[8]</sup> provides `as_sender`, which wraps any IoAwaitable as a `std::execution` sender:

```cpp
auto sndr = as_sender(sink.write(line))
    | ex::then([] { /* next step */ })
    | ex::upon_error(
        [](std::error_code ec) {
            // reachable
        });
```

The sender algebra works. `when_all` composes bridged IoAwaitables into parallel work. `let_value` sequences them. `upon_error` handles failures. The IoAwaitable is a leaf node in the sender's work graph. Structured concurrency is inherited from the sender pipeline.

Without callback handles, the bridge allocates one coroutine frame per bridged operation - the frame exists only to produce a `coroutine_handle<>`, the only type the awaitable protocol accepts. [P4126R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4126r1.pdf)<sup>[9]</sup> shows this allocation is eliminable. A callback handle - three pointers matching the coroutine frame prefix, zero heap allocation - gives senders a `coroutine_handle<>` without allocating a frame. The bridge cost drops to zero.

IoAwaitables own the I/O layer. Sender pipelines own the composition layer. The bridge connects them. With callback handles, the bridge is free.

## 10. The Case for Coroutine I/O

Section 9 shows IoAwaitables entering sender pipelines via `as_sender`.<sup>[8]</sup> [P4092R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4092r1.pdf)<sup>[10]</sup> shows senders consumed from coroutine-native code without `execution::task`. The bridge goes both ways. The broader design fork is documented in [P4088R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4088r1.pdf)<sup>[11]</sup>.

The question is not which model is more powerful. It is which implementation shape minimizes total cost when both consumers exist.

| Consumer / I/O shape | Awaitable | Sender |
| -------------------- | --------- | ------ |
| **Coroutine** | | |
| Synchronous | Zero (no suspend) | 7-step ceremony (Section 6) |
| Asynchronous | Zero protocol overhead (inherent suspend only) | Inherent suspend + ceremony |
| **Sender pipeline** | | |
| Synchronous | Zero ([P4126R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4126r1.pdf))<sup>[9]</sup> | Zero |
| Asynchronous | Zero ([P4126R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4126r1.pdf))<sup>[9]</sup> | Zero |

The awaitable column is four zeros. For synchronous I/O, the sender column carries the full seven-step ceremony of Section 6. For asynchronous I/O, the sender protocol adds ceremony - `connect`, receiver wiring, `variant` emplacement, affinity wrapping - atop the inherent suspend. The ceremony is not inherent to the async operation. It is inherent to the sender protocol.

For asynchronous I/O these added steps are a step count, not a separately observable runtime cost: once the operation suspends to a scheduler, the suspension dominates and the steps are not measurable above it. The case this paper isolates is synchronous completion, where no suspension absorbs them.

The sender pipeline cells in the awaitable column depend on P4126R1<sup>[9]</sup> callback handles. Without callback handles, senders consuming an awaitable allocate one coroutine frame per operation. Two of the awaitable column's four zeros require P4126R1.

The awaitable is the implementation shape where neither consumer pays a protocol tax.

## 11. Composed I/O

A composed I/O algorithm calls lower-level operations in a loop. `read` fills a buffer by looping `read_some`. TLS decrypts by looping encrypted reads. HTTP sequences header parsing with body reads. Each layer is a coroutine composing awaitables. These algorithms are generic - constrained on concepts, agnostic to execution context. This is the domain where composition matters.

Under the sender protocol, each iteration of such a loop pays the ceremony of Section 6 independently - even when the operation completes synchronously. Each synchronous completion constructs an operation state, instantiates a receiver, suspends the coroutine, calls `start`, fires `set_value` on the receiver, emplaces the result into a `variant`, and resumes the coroutine. For a 64 KB read with a 4 KB kernel buffer, this is sixteen iterations. On a buffered stream where most completions are synchronous, that is sixteen operation states, sixteen receivers, sixteen suspensions, sixteen resumptions. To copy bytes that are already in user-space memory.

What if the protocol could detect that the result is already available? Then the coroutine need not suspend. Remove the suspension and the resumption. What if, when the result is available, the coroutine skipped connection entirely? No operation state. No receiver. Remove the machinery that exists to shuttle a value across a suspension boundary when no boundary exists. What if the protocol expressed readiness through a single query - a boolean - true: the value is here, take it directly; false: the value requires work, suspend, resume when ready?

This is `await_ready`.

```cpp
template <typename S, typename MB>
  requires ReadStream<S> && MutableBufferSequence<MB>
auto
read(S& stream, MB buffers) ->
        io_task<std::size_t>
{
    auto consuming = buffer_slice(buffers);
    std::size_t const total_size =
        buffer_size(buffers);
    std::size_t total_read = 0;

    while(total_read < total_size)
    {
        auto [ec, n] = co_await stream.read_some(
            consuming.data());
        consuming.remove_prefix(n);
        total_read += n;
        if(ec)
            co_return {ec, total_read};
    }

    co_return {{}, total_read};
}
```

The algorithm composes `read_some` into `read` through `co_await`; the result is itself awaitable, and TLS composes `read`, and HTTP composes TLS - composition nests without sender algebra. When `read_some` completes synchronously, no allocation occurs, no operation state is constructed, no receiver is wired; the protocol adds nothing to what the hardware delivers. The requirement surface at each `co_await` is three members: `await_ready`, `await_suspend`, `await_resume`. The protocol is conditionally lazy: `await_ready() == false` defers, `await_ready() == true` proceeds - a protocol that cannot express "the result is ready, proceed without deferring" is not lazy but unconditionally indirect. The concept constraint defines the interface, the awaitable defines execution semantics, the coroutine body defines composition logic - three concerns, no coupling. The algorithm accepts any type satisfying `ReadStream`, works across execution contexts without recompilation or runtime overhead, and imposes minimal requirements on user types. The coroutine frame outlives every `co_await` within it; activations nest, RAII works, cancellation propagates downward.

When the stream is synchronous, the loop runs without suspension - `await_ready()` returns `true` at every iteration, no scheduler is consulted, no operation state is constructed, and the generic algorithm has the same cost as a hand-written `while` loop calling `memcpy`.

Stepanov's iterator concepts do not impose indirection when dereferencing a pointer. A `T*` satisfies `random_access_iterator` and dereferences in one instruction. The concept does not require constructing an intermediate state object, wiring a callback, or performing a two-phase access protocol - even though some iterators require all of those internally. The cost is proportional to what the underlying data access requires.

The awaitable protocol has this property. `await_ready() == true` is the pointer dereference: the value is there, take it. `await_ready() == false` is the disk-backed iterator: the value requires work, suspend, resume when ready. The cost tracks the operation, not the protocol. The sender protocol does not have this property. It imposes the complex-case machinery on every invocation, including the case where the bytes are already in memory. An iterator protocol that required constructing a state machine, wiring a continuation, and performing a two-phase dispatch to read a value from contiguous memory would not have succeeded as the basis for generic programming.

## 12. Closing The Gap

The sender model, as specified, does not match the awaitable model for synchronous I/O through the generic `sender-awaitable` path that every sender inherits. A concrete sender can sidestep that path by hand, by providing its own member `as_awaitable`,<sup>[3]</sup> a manual customization that is lost under type erasure. The modifications below are what would lift the costs from the generic protocol itself, for every sender, type-erased included. Each is presented in order.

### 12.1. A Readiness Query

`sender-awaitable::await_ready()` returns `false` unconditionally.<sup>[3]</sup><sup>[12]</sup> To skip suspension for senders that complete synchronously, a readiness query is required. The sender must advertise, at compile time or at run time, that its `start` will call `set_value` before returning.

[P3552R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3552r3.html)<sup>[5]</sup>'s `await_transform` could detect synchronous senders before they enter `as_awaitable` and bypass the `sender-awaitable` path entirely. It does not.

This requires a new concept requirement on senders. A trait, a tag, or a constexpr query. The sender model now has a readiness query.

### 12.2. Conditional Suspension

With the readiness query in place, `sender-awaitable::await_ready()` returns `true` when the sender advertises synchronous completion. The coroutine no longer suspends.

But `connect` was already called in the `sender-awaitable` constructor.<sup>[3]</sup> The operation state was already materialized. The receiver was already wired. The `variant` was already allocated. The suspension was saved. The ceremony was not.

### 12.3. Deferred Connection

To skip the ceremony, `connect` must be moved from the `sender-awaitable` constructor into `await_suspend`, where it can be bypassed when `await_ready()` returns `true`.

But the value needs to come from somewhere. `await_resume` must return the result. If `connect` and `start` did not execute, no receiver received the value. The sender needs a second value-delivery mechanism - a `get_value()` member, a direct extraction path, a way to produce the result without constructing an operation state, wiring a receiver, calling `start`, routing through `set_value`, and emplacing into a `variant`.

The sender model now has two value-delivery mechanisms: channels for asynchronous completion, direct extraction for synchronous completion.

### 12.4. Conditional Affinity Wrapping

`await_transform` wraps a sender in `affine_on` when the task's scheduler is not `inline_scheduler`.<sup>[6]</sup> For a sender that completes synchronously - whose bytes are already in the string before `co_await` evaluates - the affinity wrap serves no purpose.

[P3552R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3552r3.html)<sup>[5]</sup> already skips the wrap when the task's scheduler is `inline_scheduler`. But that condition is type-based - it recognizes one built-in scheduler type. A sender that always completes synchronously while awaited in a task carrying a real scheduler still pays the wrap, because `await_transform` sees a non-inline scheduler and cannot tell the sender will not suspend. To skip the wrap in that case, the sender must advertise synchronous completion - through the readiness query of Section 12.1 or a dedicated completion-behavior tag - so `await_transform` can bypass `affine_on` regardless of the task's scheduler type.

This adds no new mechanism: it reuses the readiness query of Section 12.1 to generalize an affinity bypass that already exists, lifting it from a single built-in scheduler type to any sender that advertises synchronous completion.

### 12.5. Zero-Allocation Type Erasure

`any_sender::connect` produces a type-erased operation state whose size is unknown at compile time. The current implementations use small-buffer optimization (64 bytes in stdexec) or heap allocation.<sup>[3]</sup> The per-operation cost is zero or one allocation.

Measured in the Capy benchmark suite<sup>[1]</sup> (`bench/beman`) on a type-erased no-op read, single thread, 20,000,000 operations per cell, clang 22.1.5 release build: the type-erased awaitable consumed by a coroutine allocates zero times per operation; the type-erased `any_sender` consumed by a coroutine allocates once. Wall-clock was 37 ns per operation for the awaitable and 55 ns for the sender; that figure spans two coroutine frameworks and is reported for context, while the allocation count is the structural result.

The awaitable model's type erasure is one virtual function call and zero allocations. To match this, the sender needs a base class with a virtual function that returns the value directly - without constructing an operation state, without wiring a receiver, without calling `start`.

The sender model now has virtual dispatch.

### 12.6. The Result

The sender model, modified to match the awaitable model for synchronous I/O:

```cpp
struct sync_ready_sender
{
    using sender_concept = sender_t;
    using completion_signatures =
        completion_signatures<set_value_t()>;

    // 12.1: readiness query
    static constexpr            // cf. await_ready()
        bool is_synchronous = true;

    // 12.3: direct extraction (bypass connect)
    void get_value()            // cf. await_resume()
        const noexcept;

    // 12.4: completion-behavior tag
    static constexpr auto       // cf. !await_suspend()
        completion_behavior =
        completion_behavior_t::always_inline;

    // 12.5: virtual base for type erasure
    virtual void                // cf. virtual
        get_value_erased()      //     await_resume()
        const;

    // original protocol (retained for async)
    template<class Receiver>
    struct state { /* ... */ };

    template<class Receiver>
    state<Receiver> connect(Receiver&&) const;
};
```

The awaitable:

```cpp
struct immediate
{
    bool await_ready() const noexcept;
    void await_suspend(
        std::coroutine_handle<>,
        io_env const*) noexcept;
    void await_resume() noexcept;
};
```

## 13. Concerns

**"P4126R1 is unshipped. The bridge cost is hypothetical."** Two of the awaitable column's four zeros in Section 10 depend on [P4126R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4126r1.pdf)<sup>[9]</sup>. The dependency is real. It is also an argument for collaboration, not against the design. The architects of `std::execution` should work with the author to move P4126R1 forward - so that sender pipelines can consume all awaitables for free. The core finding (Sections 6-8) stands on shipped specification alone; the bridge zeros are the prize collaboration unlocks.

**"A sender can provide a member `as_awaitable` and skip the ceremony. No protocol change is needed."** True. `[exec.as.awaitable]` uses a sender's own `as_awaitable` when the sender provides one, in preference to constructing the generic `sender-awaitable`:<sup>[3]</sup>

```cpp
// [exec.as.awaitable], reduced to the relevant branch
template<class Expr, class Promise>
decltype(auto) as_awaitable(Expr&& e, Promise& p)
{
    if constexpr (requires { e.as_awaitable(p); })
        return e.as_awaitable(p);        // the sender's own awaitable
    else
        return sender_awaitable{e, p};   // the seven steps of Section 6
}
```

In `execution::task` with an `inline_scheduler`, `await_transform` forwards the sender to `as_awaitable` with no `affine_on` wrap, and `as_awaitable` honors a sender's own member `as_awaitable` when present, so such a sender is used directly.<sup>[5]</sup> A sender whose `as_awaitable` returns a synchronous awaitable then takes the path of Section 7. `connect`, the receiver, `start`, and the `variant` are never instantiated. Three protocol steps, not seven.

This is the paper's thesis arrived at from the sender side. The synchronous fast path the sender reaches here is an awaitable: the sender hands one back, and the awaitable does the work. Closing the gap for one concrete sender, awaited from a coroutine, is one existing customization point returning the three-member struct of Section 7.

Two costs remain. The member is manual and per-sender; a sender that omits it inherits the seven-step path. And it is lost under type erasure: `any_sender` erases the concrete sender and the member with it, and `any_sender::connect` materializes the operation state of Section 12.5. Type erasure is the one sender-specific cost no `as_awaitable` member reaches.

The scope is the coroutine consumer. A sender pipeline never enters `as_awaitable`; Section 10 records zero for both synchronous pipeline cells.

**"The protocol cannot know at compile time whether a given co_await will always complete synchronously. The operation state must be constructed because the protocol must handle the general case."** The awaitable protocol handles this case exactly. `await_ready()` is evaluated at runtime: if the result is available, return `true` - no suspension; if work is required, return `false` - suspend. The protocol does not need compile-time knowledge. It asks the operation at the point of evaluation. For senders that are always synchronous (like `string_sink::write_sender`), the property is known at compile time - a constexpr trait could express it. The sender model has no such trait; Section 12.1 proposes one. The "cannot know" argument applies equally to awaitables, yet an awaitable whose `await_ready()` depends on runtime state handles both cases through the same three-member protocol: when ready, no suspension, no operation state, no receiver; when not ready, suspend, resume when ready. One protocol, two behaviors, selected at the point of evaluation. The sender protocol forces the heavy path regardless.

**"The optimizer eliminates the ceremony."** The suspension is observable behavior independent of optimization. When `await_ready()` returns `false`, the coroutine suspends and other coroutines in the executor's queue may run. The scheduling interleave is not a "specification mechanic" - it is a behavioral property that `std::execution::task` already ships. A conforming implementation cannot return `true` from `sender-awaitable::await_ready()` when `[exec.as.awaitable]` specifies `false`.<sup>[3]</sup> An implementation that does so is non-conforming. An implementation that wishes to do so requires a specification change - which is Section 12.1.

**"Operation state construction delivers structured concurrency guarantees."** Genuine for asynchronous operations where the coroutine suspends and work executes concurrently. For a synchronous write where the data is in the string before `co_await` evaluates, there is no concurrent lifetime to manage. The operation state guarantees a property that was never at risk.

**"Protocol step counts are not runtime costs."** True for `connect`, `start`, and `set_value` when sender and receiver are fully visible to the optimizer and the optimizer is sufficiently aggressive. Not true for the suspension/resumption pair, which remains observable regardless of inlining. Not true across type-erasure boundaries, where `any_sender::connect` materializes an operation state the compiler cannot see through.

**"Awaitables don't compose into work graphs."** They do, through the bridge. Section 9 shows IoAwaitables consumed by sender pipelines via `as_sender`.<sup>[8]</sup> The sender algebra - `when_all`, `let_value`, `upon_error` - works. The bridge cost is eliminable.<sup>[9]</sup>

**"Section 10 grants awaitables a hypothetical bridge while measuring senders against literal spec text."** The paper's core comparison (Sections 6-8) depends on no hypothetical. The seven-step ceremony is measured against normative text. The three-step awaitable is measured against C++20 `await_ready()`. Both are shipped. P4126R1 enters only in Section 10's "sender pipeline consuming an awaitable" cells. Two of the awaitable column's four zeros depend on it; the coroutine-consumption cells do not. The core finding - that coroutines consuming synchronous I/O pay a protocol tax under senders but not under awaitables - stands on shipped specification alone.

**"Unconditional suspension is the sound default."** The paper does not argue the default is unsound. It argues the sender protocol provides no override. The awaitable protocol solved this in C++20 with a single boolean. The question is not whether the default is correct but why the protocol has no conditional path.

**"The composed algorithm is sequential - real composition requires parallelism."** Sequential composition over a single stream is expressed as a coroutine loop. Parallel composition across multiple streams - scatter-gather, concurrent requests, fan-out/fan-in - is expressed through the sender algebra via the bridge of Section 9. The paper does not argue that all composition is sequential. It argues that sequential I/O composition - the dominant pattern in protocol implementations (TLS, HTTP, WebSocket, SMTP, DNS resolution) - is a coroutine composing awaitables, and each inner await that completes synchronously pays the ceremony independently under the sender protocol. The multiplier is proportional to the protocol's depth: HTTP over TLS over TCP is three layers of composed coroutines, each with its own `read_some` loop, each iteration paying independently.

**"The composed read loop has no cancellation propagation."** Stop tokens propagate transparently through `io_env` - the execution environment bundle passed to every IoAwaitable via `await_suspend(coroutine_handle<>, io_env const*)`.<sup>[7]</sup> Each child operation inherits the caller's stop token without explicit wiring. Every stream operation observes the stop token and may complete early with an operation-cancelled error. The mechanism is defined in [P4003R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4003r3.pdf)<sup>[7]</sup>.

**"The bridge concedes the dependency."** The bridge goes both directions. [P4093R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4093r1.pdf)<sup>[8]</sup> bridges IoAwaitables into sender pipelines. [P4092R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4092r1.pdf)<sup>[10]</sup> bridges senders into coroutine-native code without `execution::task`. Section 10 shows the cost is asymmetric: if I/O is an awaitable, both consumers pay zero; if I/O is a sender, coroutines pay the ceremony.

**"The comparison measures the wrong case."** Synchronous completion is not a corner case in I/O. Buffered writes, cached reads, DNS cache hits, and in-memory operations complete synchronously. A protocol that penalizes the common fast path compounds the cost across thousands of operations per connection.

**"Senders retarget via scheduler swap; awaitables require recompilation."** Section 4 demonstrates retargeting by relinking. One vtable call, zero allocations. The linker swaps the object file.

**"The modifications in Section 12 are natural evolution."** The modifications introduce a readiness query, a second value-delivery path, and virtual dispatch for type erasure; they also generalize the existing scheduler-affinity bypass from one built-in type to any synchronous sender. The awaitable protocol provides the same capability with three members.

**"The type erasure comparison is asymmetric."** Both paths use type erasure at the same boundary. The awaitable path produces one indirect call and zero allocations. `any_sender::connect` materializes an operation state the compiler cannot see through.

**"The falsification criteria measure senders on the awaitable's home turf."** The paper says so in Section 1: "Coroutine-native I/O cannot express compile-time work graphs. This is a genuine limitation." Section 5 credits senders with three achievements awaitables do not match. Section 10 covers both synchronous and asynchronous I/O. The falsification criteria are scoped to the paper's claim, not to a universal comparison.

## 14. Falsification

The observations documented in this paper would be discharged if any of the following were demonstrated:

- A sender protocol mechanism, equivalent to `await_ready`, that skips `connect` and `start` for trivially-ready senders without introducing a second value-delivery path.

- A `sender-awaitable` implementation in which `await_ready()` returns `true` when the sender is known to complete synchronously, without requiring `connect` to have already executed.

- A type-erasure mechanism for senders that achieves virtual-dispatch cost - one indirect call, zero allocation per operation - without reintroducing virtual dispatch.

## Acknowledgements

Eric Niebler, Kirk Shoop, Lewis Baker, and their collaborators for `std::execution` and the sender algebra. Dietmar K&uuml;hl and Maikel Nadolski for [P3552R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3552r3.html) (`std::execution::task`). Robert Leahy for the AIO-to-sender bridge. Mungo Gill for [P2583R4](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p2583r4.pdf) (symmetric transfer).

## References

[1] [Capy](https://github.com/cppalliance/capy) (C++ Alliance).

[2] [Corosio](https://github.com/cppalliance/corosio) (C++ Alliance).

[3] [P2300R10](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html) - "std::execution" (Eric Niebler, Micha&lstrok; Dominiak, Lewis Baker, Lucian Radu Teodorescu, Lee Howes, Kirk Shoop, Michael Garland, Bryce Adelstein Lelbach, 2024).

[4] [NVIDIA/stdexec](https://github.com/NVIDIA/stdexec) - Reference implementation of `std::execution` (NVIDIA, 2024).

[5] [P3552R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3552r3.html) - "Add a Coroutine Task Type" (Dietmar K&uuml;hl, Maikel Nadolski, 2025).

[6] [P3941R2](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p3941r2.html) - "Scheduler Affinity" (Dietmar K&uuml;hl, 2026).

[7] [P4003R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4003r3.pdf) - "Ask: A Minimal Coroutine Execution Model" (Vinnie Falco, 2026).

[8] [P4093R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4093r1.pdf) - "Producing Senders from Coroutine-Native Code" (Vinnie Falco, Steve Gerbino, 2026).

[9] [P4126R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4126r1.pdf) - "A Universal Continuation Model" (Vinnie Falco, Klemens Morgenstern, 2026).

[10] [P4092R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4092r1.pdf) - "Consuming Senders from Coroutine-Native Code" (Vinnie Falco, Steve Gerbino, 2026).

[11] [P4088R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4088r1.pdf) - "What C++20 Coroutines Already Buy The Standard" (Vinnie Falco, 2026).

[12] [P2583R4](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p2583r4.pdf) - "Symmetric Transfer and Sender Composition" (Mungo Gill, Vinnie Falco, 2026).

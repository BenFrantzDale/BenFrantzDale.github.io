---
layout: post
title: "Sender Intuition: Senders Don't Send"
date: 2024-10-01 12:00:00 +0000
categories: blog
---

> TL;DR: https://godbolt.org/z/4d7r4Ea8r


I’ve been keeping an eye on the [P2300 “Senders” proposal](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html#intro) for generic asynchrony for many years, but felt like I never quite “got” it. 
I know I’m not the only one who has found it challenging to grok, 
leading to questions like "[is `then(f)` a sender–receiver?](https://www.youtube.com/watch?v=nQpXOx0D7I8&t=1390s)" (It's not; see below.) Yet, I could tell a while ago that it showed amazing promise, taking inspiration from Stepanov with the ambitious goal of generically abstracting asynchronous programming with the goal of zero runtime abstraction overhead. The promise is that this will allow us to write code that works equally well for describing asynchronous algorithms on a microcontroller (without allocation or exceptions) as it does for distributing the processing of terabytes of data across a cluster of GPUs or other supercomputers. It’s a lofty goal, but as far as I can tell the goal is being met! Somehow my confusion dissolved on my drive home from the airport after CppCon 2024. Let me share the step-by-step understanding that finally made it click.

## Background

P2300 describes asynchronous work as “senders” – things which do some work (or none at all) and “send” the answer (or non-answer or stopped signal or an error) on to another stage, ensuring that the resources needed for that stage are kept alive for its duration, and making all data transfer explicit, avoiding raw synchronization primitives. This provides a C++ API – arguably a domain-specific language – that lets us write code like this:  
```  
auto result = sync_wait(  
    when_all(schedule(sch) | then(f),  
             schedule(sch) | then(g))  
).value();  
```  
To compute `f` on and `g` potentially in parallel, without even allocating memory!

But what is the computer actually doing? As a C++ programmer, I want intuition for what’s going on behind the abstraction! By that I mean like when I see   
```  
auto x = std::vector{1, 2, 3};  
```  
in my head I can picture   
```  
auto __ptr = new int[]{1, 2, 3};   
auto x = Vec<int>{.data_ = __ptr, .end_ = __ptr + 3, .cap_ = __ptr + 3};  
// ...  
delete x.data_; //< RAII: We won’t leak!  
```  
To me the beauty of C++ is the ability to have those zero-cost abstractions, allowing for clean code where I can still reason about the underlying details. I want a similar level of intuition when looking at Senders code.

Because it has a multitude of design goals, the Senders design has a lot of pieces to it. I want to focus on two parts:

1. A domain-specific language for describing asynchronous operations using expression templates.  
2. A customizable system for dispatching asynchronous work.

For me, intuition for (1) requires that I understand at least the basics of (2) so that I know approximately how the asynchronous code will actually be executed. But if you step into implementations, you quickly find yourself in a sea of underscores, namespaces, and customization-point objects. There are good reasons for all of these things, but they hindered my understanding.

Conceptually a sender is something you can “connect” to a “receiver” and “press start” and know that eventually the receiver you connected will be notified of the result (or get a stopped signal or an error value).

![Not stricktly correct: Conceptually a sender is connected to a reciever allowing you to call `start()`.](/assets/images/conceptually-connected.png)

When people say “this sender does that and sends its result…”, that’s what they mean, but as we’ll see below, that’s not exactly what happens, and if you are expecting that, you’ll get lost.

So what’s really going on? Let’s consider some expressions you’ll see in Senders code:

* `just()` is the simplest sender: It represents doing nothing and “sending” nothing onward. (A “value completion signature of `()`”.)  
* `just(42)` is similar: it’s a sender that completes with the integer 42.  
* `then(just(), f)` is a sender. It represents calling `f()` and sending the result onward.  
* `then(f)` is *not* a sender – it’s an adapter (a “sender adapter closure” if you want to be precise) that, with another sender piped in from the left, produces a sender. The existence of the type that is `then(f)` is really just to permit the pipe syntax.  
* `just() | then(f)` is a sender, equivalent to `then(just(), f)`, but providing left-to-right reading for sanity.  
* `continues_on(sch)` is not a sender. It is another sender adapter.  
* `continues_on(snd, sch)` (equivalently `snd | continues_on(sch)`) is a sender. It represents doing the work of `snd` then transferring the result data to `sch` and any future work after that will be scheduled on `sch`.  
* `just(1) | then(f) | then(g)` (equivalently: `then(then(just(1), f), g)`) is a sender. It is a nested structure that sends the value of `g(f(1))` onward.  
* `schedule(sch)` is a sender that represents starting work with a particular scheduler – we’ll revisit this later.

So, what can you do with senders? I heard they send things and connect to things. I’ve heard about receivers. Are some senders also receivers? Certainly that must be the case, right? (I had thought `then(f)` was somehow a sender and a receiver.) We are making a pipeline of senders, right? (We are not.) Senders get connected to receivers? (Yes and no.) 

> The one thing you can do with a sender is call `connect(snd, rec)`. That’s it.

OK, so then what is a receiver? A receiver is a callback that, conceptually, a sender can send to. We take one sender (which can represent a whole chain of work), and plug it into a receiver, producing an object that’s ready to run – where we can call `start()` and know that eventually the receiver will receive the result (or a stop or error signal). But here’s the rub: While we might say “you connect a sender to a receiver”, that’s not precisely true. At least not in the same sense that in an object-oriented framework you might connect an object to a callback, modifying the source object. Remember, we want to be able to avoid allocation, and these are value types assembled as expression templates, so we don’t mutate the senders (i.e., **you will never find a sender that is connected to a receiver**), instead, connection produces a new object: an operation state. Typically this will have a copy of the receiver in it, but in general it won’t include the sender that was passed to `connect`. If this feels like a lot of indirection without getting to the “do parallel work” part, keep in mind that this design needs to admit basically all asynchrony, and so there are reasons for these layers. My goal here is just to clarify what the layers are.

So what does connecting do? As a diagram, it’s more like this:

![Calling `connect(sender, receiver)` produces an operation state containing the receiver and exposing a `start()` member function.`](/assets/images/opstate.png)

But diagrams are handwavy. Let’s look at a very simple example with code. We can make our own receiver that logs the call to `set_value`:  
```cpp
//! A receiver that accepts the value channel and prints it.  
struct printing_receiver {  
    using receiver_concept = ex::receiver_t;  
    void set_value(auto&&... args) noexcept {  
        fmt::println("set_value{}", std::tie(args…));  
    }  
    // Could similarly have `void set_stopped() noexcept;` and `void set_error(auto&& e) noexcept;`.  
};  
```  
[https://godbolt.org/z/aWWPTEoqd](https://godbolt.org/z/aWWPTEoqd)  
Writing `connect(ex::just("foo"), printing_receiver{}).start()` is a very round-about (and dicy) way to write `fmt::println("set_value{}", std::tuple(“foo”));`, but it’s illustrative: A receiver is a simple thing: it’s the end of these pipelines. As it turns out, they do more than that, some of which we’ll see below.

So what happens? Where does the result of `connect` come from? The `connect` customization–point object is customizable for any sender and receiver, but here it’s `just(x)` that provided it: `just(x)` is a sender – it doesn’t have a corresponding receiver type, but it does have an operation state that conceptually is  
```cpp
//! Operation state for unary `just(x)`.
//! Starting it calls set_value on the downstream receiver.
template <typename T, ex::receiver Downstream>
struct JustOpState {
    T x;
    Downstream downstream;
    void start() noexcept { ex::set_value(this->downstream, this->x); }
};
```  
Where `connect` is customized so that `auto opState = connect(just(42), printing_receiver{});` turns into  
```cpp
auto opState = JustOpState{.x = 123, .downstream = printing_receiver{}};  
```  
And then `opState.start()` just calls `downstream_.set_value(123)` on the `printing_receiver`.  
Notice that there are no senders left after `opState` has been constructed: the temporary that is the result of `just(42)` is gone. While you will hear people say we “connect a sender to a receiver” – and at a high level that’s true – we actually created an operation state corresponding to that sender connected to that receiver.

Let’s build on that: `just(x) | then(f)` (equivalently `then(just(x), f)`) is a sender that we say “completes” with a value of `decltype(f(x))`. (Again, strictly speaking *it* doesn’t complete: it gets transformed into something else that ultimately calls `set_value` on the receiver we called `connect` with it.) The expression `then(just(x), f)`, is a sender. Looking at the right end of the pipeline it’s a `then` sender (that is, a sender produced by the `then` adapter). That sender itself contains (a copy of) the `just(x)` sender). What happens when we connect it? Well, a then sender needs an operation state for connecting it to a receiver. Here’s what a then sender looks like:  
```cpp
//! Sender for the `then` adapter.
//! Wraps an upstream sender with a function call.
template <stdexec::sender Upstream, typename Fn>
struct ThenSender {
    // Explicit concept opt-in:
    using sender_concept = ex::sender_t;
    Upstream upstream;
    Fn fn;

    auto connect(ex::receiver auto downstream) {
        return ex::connect(
            this->upstream, 
            ThenReceiver{.downstream = downstream, .fn = this->fn}
        );
    }
    
    ...
};  
```  
Note that it encapsulates everything to the *left* in the pipeline (the `just`), so in this case `Upstream` is `JustSender<int>`, so it’s basically `ThenSender{.upstream_ = JustSender{x}, .fn_ = f}`. 

Let's look at `connect`: Basically it **turns nested nested sender-adapter "onion" inside out**.
The recursive call to `ex::connect(this->upstream, ...)` will produce `upstream`'s operation state wrapped around `ThisReceiver` with `downstream` inside.
So in the case of `then(just(x), f)`, calling `connect` returns the result of connecting `just(x)` to a `ThenReceiver`. So what’s that?  
```cpp
//! A receiver for the `then` sender adapter.
//! Wraps a downstream receiver with a function call.
template <typename Fn, ex::receiver Downstream>
struct ThenReceiver {
    // Explicit concept opt-in:
    using receiver_concept = ex::receiver_t;

    Fn fn;
    Downstream downstream;

    void set_value(auto x) noexcept { ex::set_value(this->downstream, this->fn(x)); }
};
```  
So putting it all together: we started with `then(just(x), f)` which is essentially  
```cpp
ThenSender{.upstream = JustSender{x}, .fn = f}  
```
and when we call `auto opState = connect(then(just(x), f), printing_receiver{});` it evaluates to  
```cpp
auto opState = JustOpState{
    .x = x, 
    .downstream =  
        ThenReceiver{.fn = f, .downstream = printing_receiver{}
    }
};
```
Let’s review what happened here:

1. `opState` has no senders in it.  
2. There is only one operation state (`JustOpState`), corresponding to the “start” of the sender pipeline – to “starting” `then(just(x), f)`.  
3. The nested sender-adapter “onion” has been turned inside-out, putting the final receiver at the inside, and the “just” part on the outside.  
4.  The sender adapter `then(f)` first transformed into a sender (`then(just(x), f)`), but ultimately, since it’s an adapter, it gets represented by a receiver. There is no operation state corresponding to `then(f)`, just for `then(just(x), f)` – and that one is a “just” operation state (containing a “then” receiver containing the final receiver).  
5. `just` doesn’t have a corresponding receiver – it starts a chain of senders and so has an associated operation state but no receiver.  
6. Both `ThenReceiver` and `JustOpState` could easily have access to `printing_receiver`, so with some additional API it’s not hard to imagine that they could read information out of it, like if it wanted to provide a `stop_token`. This was not true of the sender passed to `connect`. This seems to be a key to the design: by going from a sender (possibly composed of adaptors around other senders) to an operation state where the nesting is inverted, the design separates composition from the runtime details that the ultimate receiver can provide.

## Scheduling

The above is just a long way to compose functions (as Sean Parent points out in his NYC++ talk linked above). Let’s distribute the work. There’s a sender factory function, `schedule` such that `schedule(sch)` is a sender that we might say “completes with no values on a given execution resource”. What does that mean, intuitively? It’s very similar to `just()` except it has to hold the resource. The sender is simple:  
```cpp
template <ex::scheduler Sch>  
struct ScheduleSender {  
    Sch scheduler;  
};

//! Operation state for scheduling:  
template <ex::receiver Downstream>  
struct MyPoolSchedulerOpState {  
    MyPoolScheduler sch;  
    Downstream downstream;  
    void start() {  
        sch.getResource().enqueue([&] { downstream_.set_value(); });  
    }  
};

auto connect(ScheduleSender<MyPoolScheduler> snd, ex::receiver auto downstream) {  
    return MyPoolSchedulerOpState{snd.sch, downstream};  
}  
```  
So when connected, we get a `MyPoolSchedulerOpState<...>`, and when started, the calling thread executes `sch_.getResource().enqueue([&] { downstream_.set_value(); });` and immediately returns, causing the thread pool to wake up and eventually `call downstream_.set_value();`. In contrast, I’ve heard talks say things like “When the `schedule(sch)` sender starts, it’s going to start on a thread in that thread pool.” That’s conceptually correct a very high level, but that language can trip people up: objects that model the `std::execution::sender` concept don’t ever actually start, and in as much as they do, they start on the thread that called `opState.start()`.

I want to highlight two things here:

1. The lifetime of the operation state is managed by the caller of `start()` – that caller had better have some way of knowing when it’s OK to destroy the operation state! For this reason, library code typically hides the receivers and the call to `start()`, and the final receiver likely has a condition variable or some similar way to indicate that the chain has completed.  
2. This `enqueue` as written here may have to allocate in some way. However, `MyPoolSchedulerOpState` is specialized for the execution resource, so the operation state could hold the node of a linked list of work to do.

## Recap

We’ve just walked through simple pipelines and scheduling in excruciating detail. I hope this starts to give you some intuition for what Senders code is doing at asynchronous execution time.  
Stated explicitly:

1. Senders don’t send. Conceptually, they represent the idea of doing some work and transferring data and control flow to a receiver, but strictly speaking, they are placeholder objects that contain just enough information to describe the work they need to do. I’m sure I’ll start to describe them as sending, but that’s shorthand for “when connected to a receiver, they correspond to all or part of an operation state that sends to a receiver”.  
2. Pipelines of senders with sender adapters are left-associative (it’s `operator|` after all), meaning it’s a layered object with the “source” sender at the core and the sender created by the rightmost sender-adapter (i.e., the combined sender for the whole pipeline) on the outside: `(just(x) | then(f)) | then(g)`.  
3. Like Eigen and C++20 range adapters, all of this stuff is based around the idea of expression templates. This makes room for a ton of compiler optimization.  
4. “Connecting” a sender to a receiver doesn’t produce a connected sender-receiver. It produces an operation state, which is conceptually the connected sener-receiver pair, but is a different type. In addition, this operation state is a core of this design because it outlives its operation, so it’s a non-allocating place to store information needed for that operation. This is part of what is meant by “structured concurrency”.  
5. Calling `connect` turns the nesting inside-out, leaving nested receivers with the ultimate sink at the core, but where the “skin” of the onion is the operation state for the initial sender (e.g., for `just()` or for `schedule(sch)`). After calling `connect`, we are done with the sender we passed in. It doesn’t need to still be alive when `start()` is called.  
6. An operation state object is opaque, immovable, and conceptually has just one “button” on it you can “press”: `start()`. If you press it, it’s your job to ensure its lifetime extends beyond completion.

## Applying Intuition: `when_all`

The above gives the basic intuition for what `schedule(sch) | then(f) | then(g)` means and how, given a receiver, it would be transformed into an operation state that includes a receiver “onion”. For me, once I understood these things, the intuition for the other senders feels pretty accessible. As a quick example, consider `when_all`, which takes senders and sends their combined results as a tuple, doing reasonable things about cancelation and errors. Since `when_all(s0, s1)` is a sender (not an adapter) so it doesn’t have a corresponding receiver, but it needs an operation state. The senders we pass in must complete, so `when_all`’s must provide receivers for each of the two senders, and a place to store each result, and `when_all`, once `connect`ed will produce an and operation state that includes the operation state `s0` and `s1` connected to those internal receivers. It also needs a stop source (which can be a `std::inplace_stop_source` since the operation state provides a safe non-heap place to put such resources!) so if any of the included senders stops, no further senders are started, and probably an atomic counter so the last thread out can send the results onward. That’s a bit of handwaving, but now I can at least imagine it.

## Further Intuition

There a bunch more questions I haven’t yet explored in depth, but could be their own articles:

* What happens at the end of `when_all(schedule(sch) | then(f), schedule(sch) | then(g))`? A thread from `sch` must realize it’s the “last one out”. Then what?  
  “Why all of these abstraction layers?” One inkling: By transforming a sender chain to a receiver chain at the point of connection to the final receiver, there are customization points to allow information to travel in both directions through the chan, so what started life as `just()` turns into something that knows what execution resource it is running on, if it needs to report cancelation, etc.  
* “What's this `let_value` thing for?” It lets you put off creating a sender until mid-execution, which is useful for various reasons.
* “How does cancelation work?” I haven’t talked about the error or stopped channels, but an interesting aspect of P2300’s design, where then “sender onion” gets turned inside out into a “receiver onion” is that it lets the “receiver onion” look inside itself to ask if cancellation is even possible, removing potential overhead.
* “What's an environment? Why would I query it?” For example, it lets you know if you need to worry about cancelation.
* “Why is everything a customization point?” Well, for moving to a GPU you need to actually move data around. The goal here is to decouple the description of asynchronous dependencies and data-flow, so you could write `just(std::move(data)) | continues_on(sch) | bulk(...)` and execute that on a CPU, but if `sch` is a GPU, that same pipeline is still correct, and that the definition of the GPU scheduler would inject the code to transport the data to the GPU.  
* “These operation states seem very simple. Why bother with them?” They are more interesting for `when_all`, `let_value`, and others, and their lifetime invariants are core to the “structured” part of “structured concurrency”.  
* What is `then(f) | then(g)` on its own? I thought pipelines were left-associative? Yes, but through trickery, the authors have set it up so you can build adaptors using the pipe syntax and have them work as you’d expect.  
* “The P2300 set of senders and sender-adapters doesn’t seem complete.” It isn’t. There are more. Some can be written in terms of the provided functionality, but for now, for some operations, you will have to write your own.

If there’s enough interest, I’d consider digging into these or other topics in future posts.

## Conclusion
There's a lot of machinery in Senders, but at its core, it's solving a very hard problem. Could it be simpler? Maybe?
But I think we primarily need more documentation of the the details so we can all have an intuition for what code-gen will result from composed sender algorithms.
See https://godbolt.org/z/4d7r4Ea8r to explore for yourself.


Thanks to the following people:

* Dietmar Kühl for the spirited face-to-face conversation (and allowing me to nerd-snipe him about cancellation at CppCon).  
* Daisy Hollman, whose [CppCon talk](https://cppcon2024.sched.com/event/1gZgc/ranges-are-output-range-adaptors-the-next-iteration-of-c-ranges) about “[Daisy Chains](https://github.com/dhollman/daisychains)” got me thinking about push-versus-pull pipelines, making me realize that with the right API, it should be possible to transform the lazy view `r | std::views::transform(f) | std::views::filter(p)` into a “pushable” object – essentially exactly what `connect(snd, rec)` does.  
* Ville Voutilainen for his comments on a draft.  
* Eric Niebler for designing this system, refining it such as renaming `continues_on` and fixing cancelation after split, and for his encouragement to write this article.


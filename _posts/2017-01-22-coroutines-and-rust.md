---
layout: post
title: Coroutines and Rust (Part 1)
---

Anyone who has ever used the Lua, Python, Javascript, Ruby, or C# language has had a good chance of coming across some sort of `yield` operator which 1) suspends the state of a running function and 2) yields back a value to whoever the caller is. These "generators" or "coroutines" fairly often implement some kind of iterable interface so they can be used in for-loops. Here's a trivial generator in Python.

```python
def range(n):
    i = 0
    while i < n:
        yield i
        i += 1
```

Generators have the benefit of being extraordinarily straightforward to write and refactor compared to the equivalent state machine. Similarly, async/await in C# is another example of an easy to use abstraction that compiles to a state machine which runs until it is blocked on some long running task, yielding control back to the thread pool scheduler until such time that the data is available. In fact there are a great many interesting and useful abstractions which can all be implemented with only the ability to suspend a function's state and return a value. If one also includes the ability to asynchronously provide input, one can write streaming parsers and sinks, which siphon from streams until they have enough input to produce a value.

Many people are interested in bringing such a feature to Rust, so I think it's worth writing this post to outline some of the things that have been said so far, pitfalls, and directions for the future, as well as compare to Ecmascript's 6's generator design. This post is the first of many and is intended both as a detail heavy introduction to people who are only vaguely familiar with the discussion or have not heard of coroutines before, and also as a kind of summary of the discussion in the last two RFCs so as to provide a good bounding point for the future. (If you have read the current RFCs for this feature, you can probably skip the "First Attempt" section.)

What this is not is:

- an RFC, or even a proto-RFC
- an authoritative document in any way
- even the best way to do any of this, as there are tradeoffs to every design decision
- a short read

# Prior Art
The community has been very vigorous in this area and so there is a lot of acknowledgment to be had. Apologies if I have forgotten someone who deserves to be mentioned.

There was a [huge issue in the Rust repo](https://github.com/rust-lang/rfcs/issues/1081) on Async IO as well as [an equally large issue on C# style yielding](https://github.com/rust-lang/rfcs/issues/388) which convinced a lot of people that we should have some kind of generator feature in Rust, be it sugar for a state machine or something more sophisticated. That led to erickt's very impressive [Stateful](https://github.com/erickt/stateful), a compiler plugin and library which allows you to write state machines using some macros. It gets you very far but is only meant as a temporary solution because this kind of feature deserves to be in the compiler proper. Nonetheless it laid the groundwork and was a great proof of concept.

There are currently two RFCs to this end, which I encourage you to read or skim through: [Zoxc's generator proposal](https://github.com/Zoxc/rfcs/blob/gen/text/0000-generators.md) and [vadimcn's stackless coroutines](https://github.com/vadimcn/rfcs/blob/coroutines2/text/0000-coroutines.md), which is where a lot of the following discussion occurred and pitfalls were discovered (EDIT: The pull request for vadimcn's RFC has been removed since this post was written but the rendered RFC is still provided her for historical purposes). The similarity between these two RFCs is that both allow you to write suspendable functions which yield values until they return a final value, as well as accept input on every resumption. There are many major differences but the most important in my view are that 1) zoxc's enum representing the result of advancing a generator contains three states while vadimcn's only has two, 2) zoxc's rfc explicitly formalizes the notion of an executor in the RFC, and 3) vadimcn's generators are mostly indistiguishable from `FnMut`, while zoxc proposes a new trait. I think zoxc's enum actually results in a more flexible API and better ergonomics, and I make this case later. Also, vadimcn's proposal is substantially more minimal, which makes it easier to comprehend and identifies the core elements well, opting to implement control flow abstractions with macros over `yield`. I draw from both of these later.

Lastly, [Eduard-Mihai Burtescu](https://github.com/eddyb) from the Rust compiler team has been invested and involved in this community-wide discussion since the very beginning and has been instrumental in this getting as far as it's gotten, as well as providing the useful perspective of someone who knows the compiler internals well. If you click on any of the links I've provided, you'll see his comments, advice, and direction throughout. 

# Design Requirements
Any good generator design should satisfy the following requirements.

* Allow one to yield values
* Allow one to return a value
* Allow values to be moved out of the generator's environment, such as a sink moving a buffer as its return value
* Allow one to send values into the generator
* Be reasonably ergonomic
* Not require any heap allocations
* Be flexible enough to write iterators, futures, streams, sinks, and protocols like Deflate
* Be composable, i.e. it should be painless to combine various abstractions with minimal or no reimplementation of macros
* Allow streams to be used with for loops
* Allow iterators to be used with for loops
* Be usable with Futures-rs with minimal to no API redesign on the library-side. 

Some optional requirements are

* Allow one to delegate to an interior coroutine until it returns (like python's `yield from`)
* Have a robust and flexible set of "generator combinators" (for instance, imagine an `ignore_all` which takes a generator that yields `Y` and returns `R` and turns it into one that yields `()` and returns `R`). 
* Reduce the amount of enums users have to write (this is vague; more on this in a bit)

# First Attempt
We start with a trait, because this is how we encode a uniform interface across a variety of types in Rust.

```rust
enum CoResult<Y, R> {
    Yield(pub Y), Return(pub R)
}

trait Generator {
    type Return;
    type Yield;
    fn next(&mut self) -> CoResult<Yield, Return>;
}
```

The `CoResult` is what we get back from calling our generator. Once we get a `Return`, we shouldn't call the generator anymore (I would expect it to panic, in fact). Any function which we expect to create a generator should implement this trait, and the compiler should autogenerate the next method and do the state tracking for us. I almost put a `Sized` bound here for simplicity but it would be interesting to explore unsized generators that are "stackful", i.e. have an unbounded execution space. 

We need three syntactic components. The first is a way to mark a function as returning a generator. The second is a `yield` keyword. The third is a type signature that includes the yield type. It's unimportant exactly what this syntax looks like as long as eventually there is some. Translating the previous python example into Rust directly might look like:

```rust
fn* range(n: usize) -> yield usize {
    let mut i = 0;
    while i < n {
        yield i;
        i += 1;
    }
}
```

Or more idiomatically

```rust
fn* range(n: usize) -> yield usize {
    for i in 0..n {
        yield i;
    }
}
```

The `*` after `fn` lets the compiler know that this is a "generator constructor", meaning that it's alright to use the `yield` keyword in this context. If we didn't have this and inferred this from the presence of `yield`, it would be possible to introduce copy and paste errors that greatly change the semantics and type inference of a function. This is even less apparent if a new user uses a macro which obscures the yield. While there are some benefits, the language team has expressed preference for explicit marking.

The `yield` keyword here suspends the generator at that statement, returns a value to whoever called it, and stores the execution state somewhere in the environment of the generator. You can think of this as similar to how a mutable closure can change its environment. In this way both generators and closures are stored on the stack, unboxed. However, while a closure only needs to store the environment that it closes over, a generator needs to have room for both the environment it closes over and the maximum extent of its own execution stack. Notice I said maximum: the stack space required for executing such a function must be bounded. That means there can be no recursive calls because it would be impossible to determine how much stack space to store for the generator to execute in. Just like with recursive datatypes, the best way to get around this is with pointer indirection.

The type signature syntax here is Eduard's. In general, `R yield Y` occurring in the return type of a generator constructor is shorthand for `impl Generator<Return=R, Yield=Y>` (for those unfamiliar with the `impl Trait` RFC, this is a compiler generated type which only guarantees that it implements some trait, and was motivated by returning unboxed closures). If you omit the return type as in `yield i32`, then this is shorthand for having a return type of `()`.

Here's how we might use this manually.

```rust
let mut i = 0;
let mut r = range(n);      // This creates the generator. No code is run at this point as the generator is "newborn". 
while let Yield(i) = r() { // Calling the generator like a function advances it toward the next yield.
    sum += i;
}
```

Eventually the generator call will return `Return(())` and we stop our iteration. It's simple enough to wire this to the existing Iterator trait so we can use them in `for`-loops:

```rust
// This impl depends on the "trait-specialization" RFC so that it doesn't cause coherence issues.
impl<G: Generator> Iterator for G {
    type Item = G::Yield;

    fn next(&mut self) -> Option<G::Yield> {
        match self() {
            Yield(y) => Some(y),
            _ => None,
        }
    }
}
```

It's notable that these are just as efficient as Rust iterators, and they are unboxed, so no heap allocations have been incurred. The stack space of the function sits there with the rest of the stack, and because it's bounded, allows you to continue doing other work while the function remains unfinished. In fact I like to think of this as a *reified stack frame* - a handle to some executing function that I can resume on-demand until it finally finishes.

Because we've written this blanket impl for generators, we can rewrite our `range` function to return something that only exposes an iterator interface, hiding the low-level implementation detail that this is an execution-suspending function that returns a `CoResult`.

```rust
fn* range(n: usize) -> impl Iterator<Item=usize> { ... }
```

As a convenience, the Python language includes a `yield from` construct which delegates to an inner generator and yields all of its values for you. Javascript includes this as `yield*`. While the Javascipt notation is ambiguous in Rust with `yield *expr`, and `from` is not a reserved word, presumably this could appear in our language as `yield while`. 

["Futures"](https://tokio.rs/docs/getting-started/futures/) are a datatype representing long-running computations that will eventually result in a value. You `poll` any type that implements `Future<Item=T, Error=E>` and it returns either `Ok(Async::Ready(T))`, `Ok(Async::NotReady)`, or `Err(E)`. Currently futures are written much like iterators are: you define a struct that contains the essential data and you implement poll yourself. The `tokio` library contains many great combinators and primitives and is extremely useful, today, for writing asynchronous code, but let's try writing our own future anyway using a generator and a fake API.

```rust
fn* async_task(db: DatabaseHandle) -> Result<Record, E> yield () {
    let mut f = db.lookup("Linus Torvalds"); // Returns an impl Future<Item=Record, Error=DBErr>
    let r = loop {
        match f.poll() {
            Ok(Async::NotReady) => { yield; continue; },
            Ok(Async::Ready(r)) => Ok(r),
            Err(e) => Err(e),
        }
    }

    return r;
}
```
This is the first time we've seen `return` in a generator. Returning is what makes generators useful for much more than just implementing iterators. Note that we move the `Record` out of this function, which is why we can't run a generator again once it's already returned.

Again, writing a blanket impl by hoisting the result out:

```rust
impl<G, T, E> Future for G 
    where G: Generator<Return=Result<T, E>>
{
    type Item = T;
    type Error = E;

    fn poll(&mut self) -> Result<Async<T>, E> {
        match self() {
            Yield(_) => Ok(Async::NotReady),
            Return(Ok(t)) => Ok(Async::Ready(t)),
            Return(Err(e)) => Err(e),
        }
    }
}
```

The type signature of our last function becomes a simple `fn* async_task(db: DatabaseHandle) -> impl Future<Item=Record, Error=DBErr>`, which is an improvement over the raw generator signature. The looping is unfortunate, but trivially remedied with a small macro (whose definition is an exercise for the reader) that expands into the same pattern.

```rust
fn* async_task(db: DatabaseHandle) -> impl Future<Item=Record, Error=DBErr> {
    await!(db.lookup("Linus Torvalds"))
}
```
Thus a user who only knows about the `Future` trait and the `await!` macro is able to write his own asynchronous code without having to know anything about generators and yielding.

In general, this is the predicted usage pattern for generators: library writers will write generators and then write traits and macros which abstract over their usage. Library users will either use the preconfigured components or have the ability to make their own components by writing a generator that yields and returns the correct types.

[This is where the smooth sailing ends in the design.](/2017/01/22/coroutines-and-rust-ii.html)
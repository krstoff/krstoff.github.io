---
layout: post
title: Coroutines and Rust (Part 2)
---
[(If you haven't read part 1, click here.)](/2017/01/22/coroutines-and-rust.html)

# Unforeseen Pitfall 1 - It doesn't compose
`Future` has a sequence counterpart called `Stream`, which could be thought of as "many futures". You poll it repeatedly and on occasion it gives you back a value wrapped in an option. When you've hit `None`, you've reached the end of the stream. A `Stream<Item=I, Error=E>` should be obtainable from any `Generator<Yield = Result<Async<I>, E>, Return = ()>`. Annoyingly, even though we were able to get rid of the `Async` type in our future, we need it in the stream because we need a way to signal that we're still waiting on a value to become available. We don't need the option because we expect to return unit at the end of our generator. I don't need to explicitly write out the blanket impl; you can imagine how onerous the datatype origami is here.

Let's write a simple stream.

```rust
fn* stream() -> impl Stream<Item=&'static str, Error=()> {
    let strings = vec!["Three", "Blind", "Mice"];
    for s in &strings {
        yield Ok(Async::Ready(s));
    }
}
```
Besides the nesting of types, this works just fine. But this isn't especially asynchronous. Let's throw in a query that is bottlenecked by IO.

```rust
fn* matches<'a>(search_string: &'a str) -> impl Stream<Item=Static, Error=()> {
    let matches = await!(find_matches(search_string)); // web request to some server, possibly
    for string in matches {
        yield Ok(Async::Ready(string));
    }
}
```

Here we've attempted to use a future in a stream, but this won't compile at all. Because we've defined futures to have a yield type of `()`, the await macro attempts to suspend the stream with `Yield(())`, which is not what the stream wants to yield. Ok, we can change the generator that maps to futures to be `Generator<Yield=Result<Async<R>>, Return=Result<Async<R>>>`. Now, we change the await macro to yield `Ok(Async::NotReady)`. The type signatures here feel very wrong, and are sure to scare beginning users - but this does work. Let's continue.

Suppose we want to inject the value of a future right into a stream.

```rust
fn* statuses() -> impl Stream<Item=Status, ()> {
    for num in 0..NUM_MACHINES {
        yield await!(check_status(num)).unwrap();
    }
}
```

Now, this isn't the best code. It unnecessarily serializes what should have been a parallel dispatch (I also punted on error handling). But by all appearances this code should be correct, and yet it isn't. The problem is that our await macro lifts the value out of the `Async` wrapper because we didn't need it anymore, but the stream yields polls, so it expects us to wrap in a `Ready`. 

This problem is not isolated. Our abstraction is too simple and thus does not compose well at all. Unfortunately, it seems that every combination of abstractions of generator-like traits results in having to write more macros, and we end up deeply nested in these enumerations. This is why I think Zoxc's addition of a third variant to represent suspension or waiting is a better design choice. We start with this alternative definition of `CoResult`:

```rust
enum CoResult<Y, R> { 
    Yield(pub Y), Return(pub R), Wait
}
```

Wait is the value returned by a `yield` with no values. (Zoxc's RFC actually parameterizes his suspension variant to contain a "reason" for waiting, but I think that complicates the API too much and we lose the dead-simple composability I'm about to show. Besides, futures-rs already uses TLS and some other machinery to communicate wait-reasons when it installs wakeup callbacks, so I think we're alright here without it.) Now we redefine our impl of Futures:

```rust
impl<G, T, E> Future for G 
    where G: Generator<Return=Result<T, E>, Yield=!>
{
    type Item = T;
    type Error = E;

    fn poll(&mut self) -> Result<Async<T>, E> {
        match self() {
            Wait => Ok(Async::NotReady),
            Return(Ok(t)) => Ok(Async::Ready(t)),
            Return(Err(e)) => Err(e),
            _ => unreachable!(),
        }
    }
}
```

Not much change here. We use `!`, the diverging type, to indicate that this branch can never happen because futures don't yield. We've also regained our sensible type signatures. What does the await macro look like now?

```rust
// await $expr expands into...
{
    let mut e = $expr;
    loop {
        match e.poll() {
            Ok(Async::NotReady) => { yield; continue; },
            Ok(Async::Ready(r)) => Ok(r),
            Err(e) => Err(e),
        }
    }
}
```
If this is what you wrote before, congratulations. We have not yet lost intuitive behavior. The real test now is to use a future in a stream and to inject one into a stream. First we define the impl over streams...

```rust
impl<G, E> Stream for G
    where G: Generator,
          G::Return = Option<E>,
{
    type Item = G::Yield;
    type Error = E;

    fn poll(&mut self) -> Result<Async<Option<T>>, E> {
        match self() {
            Yield(y) => Ok(Async::Ready(Some(y))),
            Wait => Ok(Async::NotReady),
            Return(None) => Ok(None),
            Return(Some(err)) => Err(err),
        }
    }
} 
```
The type of generator we use has actually dramatically simplified since we now only have to worry about yielding values instead of signalling in the type that we're not ready to yield one. Now let's see using a future in a stream.

```rust
fn* matches<'a>(search_string: &'a str) -> impl Stream<Item=Static, Error=()> {
    let matches = await!(find_matches(search_string)).unwrap(); // web request to some server, possibly
    for string in matches {
        yield string;
    }
}
```
It's actually simpler than what we tried writing before. This is because our suspensions (due to waiting on the future) are handled by the `Wait` variant. The await macro is robust enough to be used in this situation. In fact this is what a novice would try writing and - surprise - it works just fine. This is a big usability improvement and a win for ergonomics.  In fact I would make the argument that a future version of tokio or futures should scrap their datatypes and just use the `CoResult` directly, since it matches so cleanly with their semantics. This would avoid the unnecessary translation.

Let's try injecting a future into a stream.

```rust
fn* statuses() -> impl Stream<Item=Status, ()> {
    for num in 0..NUM_MACHINES {
        yield await!(check_status(num)).unwrap();
    }
}
```

Identical to what we wrote before, even though this expands into a double nested yield. The inner yield in the await's loop tells the generator to notify that it's waiting. The visible yield finally yields back the result of the value. We seamlessly combined two features here, and we're writing asynchronous streams as simply as we were writing iterators earlier.

I talked about `yield while` in the last section. Let's revisit this to find something surprising (at least , I didn't expect it). If this were written as a macro, what would we expect it to expand to? Probably something like this:

```rust
let generator = $expr;
loop {
    match generator() {
        Wait => yield;                      // If our sub-generator waits, so do we
        Yield(y) => { yield y; continue; }  // Whatever our sub-generator yields, we yield
        Return(r) => r                      // The last result of a generator is often useful, so we return this as the value of the expression.
                                            // (This is what ES6 does)
    }
}
```
We place the restriction of course that one can only `yield while` a generator that has the same yield type as the calling generator. Looking at this hypothetical expansion is interesting because it almost exactly resembles our await macro. The only two differences are that ours passes back a yield value while the future generator never yields a value (remember that the yield type was `!`), and that ours operates directly on a generator. In fact, *if* generators were the exposed interface for the futures library, we could simply expand `await!(expr)` as `yield while expr`! There is probably something more fundamental going on here (like, on a categorical level) that yield-while feels like a sister operation to yield. The point I'm trying to make here is that the `CoResult` type with a `Wait` variant is not convenient for our API by accident: it more directly allows us to express the relationships between "asynchronous" processes that we wanted. `await!` over a future is only useful for the one abstraction that exposes `poll`; with `yield-while` and raw generators we can "await" any datatype/process that we want that we know will eventually provide `Return`, such as a stream which finishes with a `StreamCloseReason`, for instance.

Some people might want to prevent an interior generator from providing values to the exterior one to yield. For instance, if we were awaiting another generator to finish which also functioned as a stream of values, we wouldn't necessarily want it to inject its own values into our streams. I mentioned combinators earlier. I provide a list of useful and interesting combinators I've come up with later but here's one called `ignore_all` which converts any `Generator<Yield=Y>` into a `Generator<Yield=!>`:

```rust
fn* ignore_all<G: Generator>(g: G) -> G::Return yield ! {
    loop {
        match g() {
            Wait | Yield(_) => { yield; },
            Return(r)       => { return r; }
        }
    }
}
```
This kind of selective mapping from one type parameter to another (in this case `G::Yield` to `!`) should come naturally to anyone who has ever used `Result`, `Iterator::map`, `Option`, etc.

There is so much more I have to say about the things that fall out of this design, like what this means for `Streams` and `IntoIterator/Iterator`, but I want to save that for future posts... but there's one final point I want to make on this subtopic. I just made a fairly long argument for exposing the generator trait directly to people who want to use and create futures by claiming that it simplifies the API (I called this "type-origami" earlier because of all of the nesting, which can be considered user-hostile) and gives us more composable tools. This is true. But I also claimed earlier that functions shouldn't expose the "low-level" generator interface and expose an easier to read trait that obscures unnecessary type information (for instance, most people who use streams really do not want a return value, and most people who use futures don't want a yield value). This is also true! As it stands, these are not especially reconcilable. The current version of `conservative impl trait` is a "tightly-sealing abstraction mechanism" for any trait which is not an OIBIT (like Copy, Send, etc.), so there's no way to use any `impl Trait Future<Item=I>` as a generator that returns `I` and never yields. There are two solutions for this, as far as I can tell. The first is to allow `Generator` to be a leaking abstraction type like `Copy` and `Send` are. This is what I would prefer as it makes things simpler. The second is to have a second blanket impl that actually implements `Generator<Yield=!, Return=I>` for any `Future`. I don't know if this will actually work due to coherency (and I think impl specialization has to form a DAG) and it isn't especially nice because it feels like an ouroboros of abstraction that should be simplified to begin with. At every step of the way we should be thinking about the end user - even if we can justify byzantine layers of abstraction to ourselves, will a newcomer to Rust be as forgiving? There is a third option similar to the second one to introduce `IntoGenerator`, have the await macro expand into a call that turns *anything* that implements this into a generator, and then does what it does here. I could envision tokio adopting this as the most flexible option because it allows them to maintain their API, and it also results in very simple error messages (`await!(2)` errors with "Error: i32 does not implement IntoGenerator"). ...And a fourth option of course is to just break the existing library entirely, write it from the ground up, and introduce trait synonyms to the language. I don't think anyone wants this. :)

# Unforeseen Pitfall 2 - Sending values isn't as easy
I haven't mentioned input at all yet. I imagine that many people might ask "why would I want to send input to a generator?", especially if you have only ever heard of generators in the context of futures, streams, and iterators. I conjecture that there are actually vastly more use-cases for asynchronously receiving values than there are for yielding or returning them. ES6 refers to this as the "observer pattern", so I transliterate many of their disseminated examples here, but I describe their generator implementation at length in the next post.

vadimcn's RFC tackles this by treating generators almost indistinguishably from `FnMut`. In fact, any generator is essentially an `FnMut(Input) -> CoResult<Y, R>`. This is elegant, both conceptually and implementation-wise. The functions we've been writing as `fn*` all desugar into returning a closure, and the traits we've been implementing are implemented over mutable closures returning CoResults. On to sending values: the idea is that the initial argument to the closure is the first input, and future inputs are returned by `yield`, i.e. yield becomes both an expression and control-flow. To see what I mean, consider this code:

```rust
let mut concat = |s: Option<&str>| {
    match s {
        None => { return String::new(); }
        Some(s) => {
            let string = String::from(s);
            while let Some(s) = yield {
                string += s;
            }

            return string;
        }
    }
}
```

This generator concatenates slices onto an internal string until we send it `None`, signaling that we've ended our function. In fact, we can actually write any iterator consumer as an `FnMut(Option<Item>) -> CoResult<!, R>`. Here's a hypothetical method on the `Iterator` trait:

```
pub fn feed_into<F, R>(self, f: F) -> R where F: Fn(Option<Self::Item>) -> CoResult<!, R> {
    let mut r;
    for i in self {
        r = f(i);
    }

    match r {
        Return(ret) => ret,
        _ => panic!("Did not receive enough input."),
    }
}
```

I want to stress that this input type is actually distinct from the type of our constructor arguments. Our constructor constructs a generator but does not begin running it on anything. Doing so would be hazardous. For instance, as I've mentioned, futures use thread local storage liberally, such as to install callbacks and unpark, but this isn't possible if the thread isn't running an executor (like your main thread won't be while you're building up future-computations). It would be impossible to use futures at all if they immediately started running while you're building new futures with combinators. 

Thus far we have always assumed we have input to begin with. But it's often the case that we don't have any input when we start our generator and wait for more. Here's a "logger" in javascript.

```javascript
// Prints anything it receives.
function* dataConsumer() {
    var i = 1;
    while (true) {
        console.log(`${i}. ${yield}`);
        i++;
    }
}
```

Our constructor definitely doesn't start off with any string to print and we probably don't want to initialize it with one anyway. If I were using this in a real application, I'd prefer to have this declared somewhere in main, pass it down my app as necessary, and send a string when I'm very deep in the logic already and need to log something. A poor solution to this problem is to force the api to take in `Option<String>`. But this doesn't really match the semantics that we want because we always know that there won't be input to start with, and we're forced to unwrap all later input. It also becomes more onerous to call from user code. You might object that we've already done something like this previously, with our string concatenator. But we should be motivated to remove as many unnecessary enumeration types as possible and reduce the complexity of our interfaces. Furthermore, even our string concatenator benefits from these improved semantics:

```rust
// Sorry in advance for this type signature.
fn* concat() -> for<'a> impl Generator<&'a str, Yield=!, Return=String> {
    let string = String::new();
    while let Some(s) = yield {
        string += s;
    }
    return string;
}
```

An entire layer of nesting has been removed from our generator and we didn't need to unroll the first case. There is also an argument here for generality. Let's call a generator scheme that requires input "input requiring generators" and let's call a generator scheme that doesn't "input-optional generators". Every input-requiring generator can be written in the input-optional scheme by immediately yielding on the first line of the generator, but in order to write an input-optional generator in an input-requiring scheme, you need to introduce an `Option` and unwrap on every usage.

So we've established that input is desirable and we don't necessarily want input at the beginning of the generator's execution. Now we have to complicate our trait a little.

```rust
trait Generator<Input> {
    type Yield;
    type Return;

    fn start(&mut self, Input) -> CoResult<Yield, Return>;
    fn next(&mut self) -> CoResult<Yield, Return>;
}
```

Again, though, we've run into a little bit of trouble. What happens if we start a generator that's already executing? I'd recommend panicking, since that's what we do when we try to advance a generator that's already returned. What happens when we try to advance a newborn generator that hasn't been started? Again, I'd recommend panicking here, but this is far more precarious because when you've been given a generator from somewhere, you don't know if someone has already started it for you. This is actually a problem in Javascript: you start a generator with `next()` to "prime it" and then advance it with `next(args)`. This is annoying for observer adapters, so there is a helper function that primes your generator for you, but I'm unconvinced that's the right approach. I'm inclined to say that "Observers" (state machines which read values and might return but never yield) and "Generators" (state machines which yield and return but never read) are distinct enough to warrant having different traits. One argument for that is that having a design which covers every case but complicates usage is strictly worse than having multiple abstractions that are simple to use and do their job well. You see this in the design of Rust's error handling, where `Result` types are encoded at the type level for exceptional but expected situations, and hard-panics are for unrecoverable errors, vs. the typical approach of having everything implemented with "exceptions".

In (Exploring ES6)[http://exploringjs.com/es6/ch_generators.html], Dr. Axel Rauschmayer gives a few examples of generators which are both observers but also yield values. In my view, none of these examples are really relevant or expressible in Rust. For instance, there is a kind of fake y-combinator pattern where a generator expects a reference to itself as an input so that it can call next on itself later, thereby driving its own execution. In Rust this would be illegal (and memory unsafe) self-borrowing. Also, there is an example of an implementation of javascript's communicating sequential processes which uses yield to provide a reason-for-suspending to the green thread executor, but our futures library already does this just fine without any notion of (further) complicating the return type of `poll`. But it still feels like the wrong decision to not have the ability to both receive and send input asynchronously, so I think this deserves some more consideration. 

The last thing before I end this sub-section is `yield while`. I've been referring to it as if it were an operator but we can certainly implement it as a macro. For ergonomic reasons, and because we expect it to run a generator to completion, we expect the generator to be newborn. We also want it to send input sent to the exterior generator down into to inner one. Thus this code:

```rust
let value = yield_from!(generator());
```

should translate to

```rust
let value = {
    let mut value;
    let g = generator();
    let mut next = g.start();
    loop {
        let input = match next {
            Wait => yield,
            Yield(y) => (yield y),
            Return(r) => { value = r; break; },
        }
        next = g(input);
    }

    value
}
```

# Unforeseen Pitfall 3 - Suspending while an interior borrow is held

The two RFCs I mentioned in the prior art section have one specific, important restriction: no borrows to anything in the execution space of a generator must persist across suspension points. To see why this is a problem, consider this trivial code.

```rust
fn* gen() -> yield {
    let x = 2;
    let r = &x;
    yield;
    let y = *r;
    return;
}

let g = gen();
g.start();
return g;
```

After I call the start method, there's a borrow in g pointing somewhere to the stack. Then I move the generator. This pointer now points to garbage. So the RFCs explicitly disallow this. But this is unfortunate and undesirable for at least three reasons.

The first is that it makes writing generator functions really hard compared to writing a normal function. You have to move your operations around and assign temporaries so that no borrows exist. This is a substantial break from what people are used to and makes for a poor experience. 

The second is that there is a lot of code I want to write that should be correct but isn't. For instance, here's a perfectly reasonable future that fills a buffer:

```rust
fn* task() -> impl Future<Item=Vec<u8>> {
    let mut v = Vec::new();
    
    await!(task1(&mut v));
    await!(task2(&mut v));
    await!(task3(&mut v));

    return v;
}
```

Or, a common problem in using futures is that with the current combinators, you can't share state in what is essentially a sequential chain of futures without using an `Arc`. With generators, this is just a normal matter of scope, not sharing. 

The third is that in practice, a lot of generators aren't really going to be partially run and then moved, anyway. Take a future. Typically futures that are run on a queue are boxed once, and then live on the heap for the entirety of their execution, so they are never moved anyway. Or, if you run a future with the blocking `wait` executor, it consumes the future and runs it in place until it finally returns. Or take a for loop. The for loop always consumes the iterator, so you never have an opportunity to run it anyway. Writing generators that borrow their own interiors isn't a problem as long as they're used in the way that we typically use them right now, it's just that Rust isn't smart enough to understand immovable values. [This recent RFC by Zoxc](https://github.com/rust-lang/rfcs/pull/1858) attempts to solve this problem with a new `Move` marker trait that some types don't implement. The compiler will know that once you mutate this type, you shouldn't move it. This also should allow Rust to have better support for and safer interfaces for "intrusive datastructures", which just means that the interior components are aware of and have pointers to the surrounding datatype.

`impl Trait` currently doesn't support multiple bounds, but if it eventually does, then you can write something like this:

```rust
fn* count(n: usize) -> impl Iterator<Item=usize> + Move {
    for i in 0..n { yield i; }
}
```

and the compiler will yell at you if you have an interior borrow somewhere. Rust's safety and marker traits really shine here. This is simultaneously the most important roadblock and also the shortest section because it is so easy to solve. 

# Conclusion
If you weren't familiar with the full range of use-cases for generators, you probably are now. We've made a straightforward but naive first attempt and seen the drawbacks. Hopefully this post is useful as a good starting point for anyone who wants to contribute to what is a community wide discussion. These issues were discovered by trying to build usable abstractions with the proposals. The next post (which is already written) goes further with this philosophy. We'll try to implement an Actors library and a Communicating Sequential Processes library and learn some lessons. The third post will get into much more detail into `futures-rs`, as well as the standard library and IO (such as providing non-blocking methods to currently blocking datastructures like `Mutex`). And if I don't get repetitive strain injury, I'll write a fourth post about some more exotic extensions to generators like generators which have multiple methods and unsized/stackful generators (for tree iterators).
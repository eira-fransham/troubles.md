---
title: "RustFest Paris Workshop: Fastware"
date: 2018-05-22T11:22:48+02:00
draft: false
---

> "In almost every computation a great variety of arrangements for the succession of the processes is possible, and various considerations must influence the selection amongst them for the purposes of a Calculating Engine. One essential object is to choose that arrangement which shall tend to reduce to a minimum the time necessary for completing the calculation." 
> 
> ### Ada Lovelace, 1843

## Prerequisites

You should get these from your package manager where possible, but if you don't have a package manager installed then you can download them from the linked websites. Your package manager will be [choco][choco] on Windows, [homebrew][brew] on macOS, and on Linux it could be one of `apt-get`, `pacman`, `rpm`, `nix`, `cron`+`wget`, your secretary logging into your computer and downloading unsigned binaries while you sleep, etc.

- You'll need [Rust][rust] to compile your code, of course.
- The sample code for this is [here][rustfest-sample-code].
- To compare results we can use `cargo-benchcmp`, just run `cargo install cargo-benchcmp` to get it.
- To generate performance traces you'll need [Valgrind][valgrind], this requires macOS or Linux. If you're on Windows then you can use my Valgrind traces, see below.
- To view the results you can use [KCachegrind][kcachegrind].

[choco]: https://chocolatey.org/
[brew]: https://brew.sh/
[rustfest-sample-code]: https://github.com/Vurich/rustfest-perf-workshop
[rust]: https://www.rust-lang.org/
[valgrind]: http://valgrind.org/
[kcachegrind]: https://kcachegrind.github.io/html/Home.html

You'll probably have seen `perf`+`flamegraph` recommended before by [certain very smart and handsome fellows][warp-speed] but I've actually had more luck with `callgrind` (included in Valgrind). You can try to use `perf` but although it can give very good outputs for some programs it's quite inconsistent. Also, using `callgrind` means you can use KCachegrind, which is the best thing since sliced bread. In the linked article I said that Valgrind was scary, but that was one of my patented Uninformed Opinions&trade;. It's actually very intuitive and, honestly, easier to use than the "simple" solution of using `perf`.

[warp-speed]: {{< ref "posts/rust-optimization.md" >}}

## Finding our slow sections

It's often said[^often-said] that the slowest code is that which has been optimised without benchmarks. You wouldn't expect your code to work if you never ran it, so why should you expect it to be fast if you never benchmarked it? Writing good benchmarks is a bit of an art, because it's really easy to accidentally write benchmarks that make your code seem fast, when really the compiler is applying some optimisations that work in the side-effect-free world of the benchmark but can no longer get applied when you put it out into the wild.

[^often-said]: By me. So far I haven't got it to catch on.

You should have a few different kinds of benchmarks:

1. Ones that operate on real-world data - if you spend all your time optimising but your users never see that speedup then what was the point of doing all that work in the first place?
2. Ones that exercise degenerate cases to make sure that your code isn't fast in the general case but incredibly slow in edge-cases.
3. Ones that exercise real-world cases in confinement, this can help pinpoint where your code is choking.

You can see where I added the benchmarks in [this commit][benchmark-commit] in the sample code repo. I've heavily commented the benchmarks so you can see my thinking, and I highly recommend you read the comments and the code in that commit to get an idea of what a decent benchmark suite looks like. It's not perfect, more and more-accurate benchmarks could be written, but the point is that our language has very few primitives (function calls, variables, assigment, literals) and so we need benchmarks that exercise each of those.

[benchmark-commit]: https://github.com/Vurich/rustfest-perf-workshop/commit/f839ff3cd76343e7371eec73de61997fa000f1eb

Probably you already know [how to run benchmarks in Rust][benchmarks], but doing so isn't actually terribly useful in isolation. Benchmarks alone are essentially meaningless, since they are affected by everything from the current PC setup to the other currently-running programs. Their power comes in comparison.

If you haven't already, you need to make sure that you're using the nightly release of Rust, since the stable release doesn't allow you to run benchmarks. If you're using [`rustup`][rustup], this is simple - just do `rustup override set nightly`. If you're not using `rustup`, then use `rustup`. Sorry, it's really the only convenient way to use nightly Rust.

To get a baseline benchmark, you can output the results of `cargo bench` to a file. On Linux and macOS you can just run `cargo bench > baseline.bench`. On each change you can output the new benchmarks to a new file by running `cargo bench > new.bench`, then compare the two with `cargo benchcmp baseline.bench new.bench`. If you do it now, without making any changes, you should get something that looks like the following:

[rustup]: https://rustup.rs/

```text
name                           baseline.bench ns/iter  new.bench ns/iter  diff ns/iter  diff %  speedup 
benches::parse_deep_nesting    52,923                  53,618                      695   1.31%   x 0.99 
benches::parse_literals        34,167                  34,623                      456   1.33%   x 0.99 
benches::parse_many_variables  182,052                 180,939                  -1,113  -0.61%   x 1.01 
benches::parse_nested_func     36,798                  37,222                      424   1.15%   x 0.99 
benches::parse_real_code       4,578                   4,574                        -4  -0.09%   x 1.00 
benches::run_deep_nesting      2,705                   2,722                        17   0.63%   x 0.99 
benches::run_many_variables    42,622                  42,193                     -429  -1.01%   x 1.01 
benches::run_nested_func       4,563                   4,698                       135   2.96%   x 0.97 
benches::run_real_code         141,153                 141,211                      58   0.04%   x 1.00 
```

You'll notice that even when you run the exact same code twice you can still get slightly different results. You shouldn't trust speedups of less than 2% unless you can very reliably reproduce them.

So, where do we start? Well, that's what our trusty friend Valgrind is for. Valgrind is an execution framework that allows plugins to analyse a compiled program at quite a deep level. It ships with tools to [detect memory safety bugs][memcheck], to [report possible deadlocks and race conditions][helgrind] and - most importantly for us - to [analyse the time taken by individual functions][callgrind]. It can't produce output that's very useful to humans without some help, however. Specifically, it needs debuginfo enabled. For release builds (`cargo build --release`) you can add that to the `Cargo.toml` like so:

```toml
[profile.release]
debug = true
```

[memcheck]: http://valgrind.org/docs/manual/mc-manual.html
[helgrind]: http://valgrind.org/docs/manual/hg-manual.html
[callgrind]: http://valgrind.org/docs/manual/cl-manual.html

We need it for benchmarks, however. For that `cargo` uses the `bench` profile, and so we need to add the following to `Cargo.toml`:

```toml
[profile.debug]
debug = true
```

This is added to the sample project already (see [this commit][debuginfo-commit]). If you run `cargo bench` it will first build the benchmarking binary and then run it, displaying the path on stdout. When I run `cargo bench` it looks like this:

```text
jef@jef-pc ~/D/G/V/rustfest> cargo bench
   Compiling rustfest v0.1.0 (file:///home/jef/Documents/GitHub/Vurich/rustfest)
    Finished release [optimized] target(s) in 11.84s
     Running target/release/deps/rustfest-5c3dd55c20998644

running 9 tests
test benches::parse_deep_nesting   ... bench:      53,041 ns/iter (+/- 18,369)
test benches::parse_literals       ... bench:      34,433 ns/iter (+/- 5,025)
test benches::parse_many_variables ... bench:     185,630 ns/iter (+/- 36,507)
test benches::parse_nested_func    ... bench:      38,114 ns/iter (+/- 6,402)
test benches::parse_real_code      ... bench:       4,758 ns/iter (+/- 842)
test benches::run_deep_nesting     ... bench:       2,793 ns/iter (+/- 243)
test benches::run_many_variables   ... bench:      46,430 ns/iter (+/- 7,715)
test benches::run_nested_func      ... bench:       5,050 ns/iter (+/- 597)
test benches::run_real_code        ... bench:     152,798 ns/iter (+/- 46,472)

test result: ok. 0 passed; 0 failed; 0 ignored; 9 measured; 0 filtered out
```

[debuginfo-commit]: https://github.com/Vurich/rustfest-perf-workshop/commit/24a8fab60674dbc70b2ad18df8baba3e2edace65

You can see that it displays the binary path on the third line, under `Running`. You can then run callgrind on this binary. For me this is `target/release/deps/rustfest-5c3dd55c20998644` but for you it could be something else, so use the output of `cargo bench` on your machine to see what the binary's name is.

> ### Important!
> 
> The binary's name will change when you change your code, as it's based on a hash of the source code and cargo's build parameters. Therefore it's very important to rerun `cargo bench` and run callgrind on the _new_ binary name. Otherwise you'll be running it on the old binary and it will look like nothing has changed!

## Contents:
- [Exercise 1](#exercise-1)
- [Exercise 2](#exercise-2)
- [Exercise 3](#exercise-3)
- [Exercise 4](#exercise-4-intermediate-and-advanced-only)
- [Further work 1](#further-work-1)
- [Further work 2](#further-work-2)
- [Further work 3](#further-work-3-advanced-only)

# Exercise 1

If you're on a platform that Valgrind supports, you can follow the next steps to get a trace. Otherwise, you can use [my trace][valgrind-trace]. To run callgrind, execute Valgrind with `--tool=callgrind` on the benchmark binary. In order to make the benchmark binary actually run the benchmarks (by default it will run tests) you have to additionally pass `--bench` as an argument to the binary itself, like so:

[valgrind-trace]: https://gist.githubusercontent.com/Vurich/5fc8d700ceaa85c0b185eb493a3d5125/raw/28894cfd8869b377fbb96e7a0a4f1c9e204a213b/callgrind.out.6359

```text
valgrind --tool=callgrind target/release/deps/rustfest-5c3dd55c20998644 --bench
```

Callgrind actually has lots of options which you can see with `--help`, and they're really useful for debugging slowness within functions (as opposed to which functions are slow), but for now we'll just run it with the defaults. This will generate a file with a name like `callgrind.out.1234` where `1234` is a unique number for each run of callgrind. Run KCachegrind (it's a GUI program, so you probably don't want to start it from the command-line) and open the generated `callgrind.out` file. You should see something that looks like so:

![KCachegrind's home screen](/rustfest-2018-workshop/kcachegrind.png "KCachegrind's home screen")

This is probably pretty overwhelming. We can whittle down the functions to just our code by typing the name of our project (if you're using the sample project then this will be `rustfest`) into the search bar on the left. If we look at the list of functions that take the most time, we can see that after the generated `main` function the most time is spent in `eval`. Probably you guessed this, what with it being one of only two functions in our module, but hey, it's nice to get confirmation. Let's click on `eval` and see what its cost centers are. In the bottom-right, you'll see the list of functions called by `eval` in order of their total cost - these are called the "callees". You can see a visualisation of the callees in a tree-like structure by clicking "callee map" at the top, but depending on your tolerance for soulless coloured rectangles the amount of use you can get from this may vary. In the list of callees, you can see that a _lot_ of time is spent in `HashMap::clone` and `HashMap::drop`[^rawtable].  Essentially, we keep creating new `HashMap`s and then destroying them again:

[^rawtable]: Actually `HashMap` is a wrapper around `RawTable` and so you'll see `RawTable` in the callee map, `HashMap` has been totally inlined at this point and so doesn't appear in the stack trace. If you see a `std` item that you don't recognise then try looking it up in Rust's [stdlib documentation][std-doc]. Unfortunately, that doesn't help here (`RawTable` is private) but it will help for many other types. For example, the "raw" inner type for `Vec` [is public and listed in the stdlib docs][rawvec].

[rawvec]: https://doc.rust-lang.org/1.9.0/alloc/raw_vec/struct.RawVec.html
[std-doc]: https://doc.rust-lang.org/1.9.0/std/

![`eval`'s callees with `HashMap::clone` and `HashMap::drop` at the top](/rustfest-2018-workshop/kcachegrind-callees.png "`eval`'s callees with `HashMap::clone` and `HashMap::drop` at the top")

So where in `eval` are we cloning hashmaps? If we look at the source code, the answer is obvious:

```rust
pub fn eval(program: Ast, variables: &mut HashMap<String, Value>) -> Value {
    // ..snip..
    match program {
        // ..snip..
        Call(func, arguments) => {
            let func = eval(*func, variables);

            match func {
                Function(args, body) => {
                    // Start a new scope, so all variables defined in the body of the
                    // function don't leak into the surrounding scope.
                    let mut new_scope = variables.clone();
                    //       Clone here ^---------------^

                    // ..snip..
                }
                // ..snip..
            }
        }
        // ..snip..
    }
}
```

If we look at the callees for `clone` and `drop` by double-clicking on them we can see that the `HashMap` is spending a lot of time recursively cloning and dropping its elements. This is because not only is the `HashMap` heap-allocated, but so are its elements. `String` is a heap-allocated string that must be allocated, cloned and deallocated, and `Value` contains `Vec`s and `String`s - both heap-allocated. These values are all immutable and so we don't actually need owned, heap-allocated versions of these. There are a couple of optimisations that you can apply here, and I've seperated them into beginner, intermediate and advanced.

## Beginner

Convert our uses of `String` to be `Rc<String>`. This means that the clone and drop are just adding and subtracting 1 to the reference count, respectively. Run the benchmarks before and after making this change and use `cargo benchcmp` to verify that this has made an improvement.

## Intermediate

Same as above, but notice that `Rc<String>` actually has _two_ heap allocations - the `Rc` just points to a _pointer_ to the actual string data. Figure out if there's a way to avoid this. Hint: `Rc`'s type parameter is `?Sized`.

## Advanced

Do these strings need to be allocated on the heap at all? Combine actually has [an API for zero-copy parsing][combine-zero-copy]. It requires ensuring that the input can have pointers constructed to it - not possible if you're parsing from an iterator over bytes, for example. To ensure this you can add the following to the parser definition:

[combine-zero-copy]: https://docs.rs/combine/3.3.0/combine/parser/range/fn.recognize.html

```diff
diff --git a/src/lib.rs b/src/lib.rs
index 359915d..6db1190 100644
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -91,7 +91,8 @@ pub fn eval(program: Ast, variables: &mut HashMap<String, Value>) -> Value {
 }
 
 parser! {
-    pub fn expr[I]()(I) -> Ast where [I: combine::Stream<Item = char>] {
+    pub fn expr['a, I]()(I) -> Ast where [
+        I: combine::Stream<Item = char, Range = &'a str> +
+        combine::RangeStreamOnce
+    ] {
         use combine::parser::char::*;
         use combine::*;
```

I'll leave it to you to figure out the rest.

# Exercise 2

Benchmark your code before and after making the changes above and compare the results with `cargo benchcmp`. Did it help? If so, how much by?

Even after making cloning our strings cheaper, we still do a lot of clones of values that are expensive to duplicate. We call `clone` on `Ast` and `Value`, both of which can contain `Vec`s and `Box`s - expensive to clone. Let's try to make those clones a little cheaper.

A cheap way to avoid clones is as above - use `Rc`. It is not necessarily a good thing, though. For a start, it's less ergonomic - only allowing immutable access. It's not necessarily faster in all cases either, since it adds an extra heap allocation and it means that dropping the value incurs a branch. Also, although `Rc` supports weak references it won't actually drop the value until all weak references have been destroyed - a possible source of memory leaks. `&` pointers do not have this issue - although the compiler gives them special powers, at runtime they're just integers. Both can be used to avoid clones.

## Beginner

We take an owned `Ast` as an argument to `eval`, which requires cloning the whole `Ast` every time we call `eval` again in the loop - this isn't necessary. We can take a borrowed `Ast` (`&Ast`) and only clone the parts that we need. This should be faster than cloning the entire `Ast` on the chance that we need part of it as an owned value.

## Intermediate

What other clones are unnecessary? Most of our code is pure and only requires immutable access. However, we still clone the `Value`s in our scope every time we access them. Is there a way around this?

## Advanced

Same as above, but note that most of `Value` is actually very cheap to clone. What can we do to take advantage of this fact?

# Exercise 3

Again, make sure to run benchmarks to ensure you're making your code faster and not slower.

Check KCachegrind again - just underneath `clone` and `drop` then next most time-consuming function is `insert`. This is because Rust's default implementation of `HashMap` is not as fast as it could be - by design.

The reasoning behind this is best illustrated with an example. Let's say that we're selling flapjacks on the internet and we want to keep a mapping from customer name to address, in order to know where to send their delicious flapjacks[^flapjacks]. Since we're a small business that has yet to reach [Web Scale][mongodb] we've decided to just use an in-memory `HashMap` like so:

[mongodb]: https://www.youtube.com/watch?v=b2F-DItXtZs

[^flapjacks]: This example works with American or British flapjacks but just to make sure we've all equally confused, let's say that it's a new kind of flapjack that combines the benefits of both British and American kinds.

```rust
struct ServerState {
    customers: HashMap<String, String>,
}
```

Except that the way that `HashMap` is implemented is to seperate the storage into some number of "buckets", and so when you insert or retrieve from the `HashMap` you first find the bucket that it's in and then iterate over that bucket until you find the entry with the right key[^hashmap]. It looks something like this:

[^hashmap]: The real libstd `HashMap` is _much_ more complicated than this, using a cool algorithm known as "Robin Hood hashing" - because it steals from the "rich" (the full buckets) and gives to the "poor" (the less-full buckets). There's a really good explanation of this algorithm on the always-fantastic [Accidentally Quadratic blog][accidentally-quadratic-robin-hood]. For the purposes of this explanation though, this simplistic implementation is good enough.

[accidentally-quadratic-robin-hood]: https://accidentallyquadratic.tumblr.com/post/153545455987/rust-hash-iteration-reinsertion

```rust
struct HashMap<K, V> {
    buckets: Vec<Vec<(K, V)>>,
}

impl<K: Hash + Eq, V> HashMap {
    // ..snip..

    fn get(&self, k: &K) -> Option<&V> {
        let bucket = self.hash(k) % self.buckets.len();

        for &(ref existing_k, ref v) in &self.buckets[bucket] {
            if existing_k == k {
                Some(v)
            }
        }

        None
    }
}
```

If you're an attacker trying to take down this innocent flapjack website and you can manipulate which bucket your customer name goes into, you can simply put all of your entries into the _same_ bucket. This means that every new insertion or retreival to that one bucket is extremely slow, because it has to do much more comparisons than you normally would have to. The default `HashMap` used in Rust has two defences against this. Firstly, it initialises the hashing algorithm with a random seed, which means that because you don't know the initial hasher state you can't run the hashing algorithm many times on your own computer, find some customer names that would go into the same bucket, and then send those to the server. Second, the hashing algorithm is cryptographically secure. This means that there is no way to guess the initial state by just running the hasher many times, measuring how the output changes, and then deriving the state with maths. The default hashing algorithm is actually pretty fast, but is bad on small keys (like the short variable names in our programs). To improve performance we can use `FnvHash` from the [`fnv`][fnv] crate.

[fnv]: https://crates.io/crates/fnv

## Beginner

Just add `FnvHashMap` to your crate, and check the benchmarks.

```rust
extern crate fnv;

use fnv::FnvHashMap as HashMap;
```

## Intermediate/Advanced

There are other "fast hashmaps" for Rust, like [`fxhash`][fxhash]. Which one is faster for these inputs?

[fxhash]: https://crates.io/crates/fxhash

# Exercise 4 (Intermediate and advanced only)

Although we made cloning the scope a lot cheaper, we still clone it in cases that are not necessary. For example, if we call a function with no arguments that doesn't define any internal variables, we don't need to clone the scope at all. We can avoid that by either passing a reference to an owned `HashMap` if possible, and an immutable reference otherwise. If it turns out that we need to clone the hashmap then we can replace the reference with an owned `HashMap` and mutate it from there. The standard library has a type to make this easy: [`Cow`][std-borrow-cow].

[std-borrow-cow]: https://doc.rust-lang.org/std/borrow/enum.Cow.html
[cow-to-mut]: https://doc.rust-lang.org/std/borrow/enum.Cow.html#method.to_mut

## Intermediate

Use `std::borrow::Cow` to avoid cloning the scope unless absolutely necessary.

## Advanced

Explore the use of [persistent data structures][rpds] to maximise sharing. Do they help?

# Further work 1

If you did the beginner or intermediate tasks for the previous exercises, why not try going back and doing the task a level above? If you did the advanced level, are there any more optimisations that you could apply to this? If so, implement them and see if they improve your results.
# Further work 2

Try doing this same process on one of your own projects. Add debuginfo to the benchmarks, run it with `valgrind --tool=cachegrind /path/to/benchmarks --bench` and then open the results in KCachegrind. See if there's some way to reduce unnecessary copies, unnecessary work, and/or unnecessary allocation. Good targets for optimisation are virtual machines, image processing libraries and games. For games, though, if you're using a game engine then your traces might show most of your time taken in the engine's code and not yours.

# Further work 3 (Advanced only)

Most modern languages don't operate directly on the AST of their program, they compile to bytecode first. One important thing to notice about this language is that (because `if` statements aren't lazily evaluated and there are no loops) it's impossible for a program to conditionally define a variable. Therefore if a variable is defined and accessed within the same function you can just access it by index instead of by name. Unfortunately, because our language is [dynamically scoped][dynamic-vs-lexical-scope], we can't do the same when a variable is accessed from the surrounding scope. In this case, we must access it by name. If we're assuming a stack-based virtual machine this would look something like so:

[dynamic-vs-lexical-scope]: https://stackoverflow.com/a/22395580

```rust
enum OpCode {
    // ..snip..

    /// Access a variable in the current function's scope.
    PushVariableFromCurrentScope {
        /// The stack slot to access. If we have 4 variables called `a`, `b`,
        /// `c`, and `d`, we can access `a` with stack slot `0`, `b` with stack
        /// slot `1`, etc. This is much faster than looking up by name.
        stack_slot: usize,
    },
    /// Access a variable in the parent scope. Because this language is
    /// dynamically-scoped we can't apply the same optimisations as if we had
    /// made it lexically scoped. As a result, we must look up the variable
    /// by name each time.
    PushVariableFromSurroundingScope {
        /// The name of the variable. This would be `&str` but I've chosen
        /// `String` here to illustrate my point.
        variable_name: String,
    }

    // ..snip..
}
```

Then you can convert function calls to jumps (first-class functions would get converted to passing function pointers) and the entire `eval` function becomes one loop. Much faster than recursive calls. Probably you won't finish this task by the end of the session but hey, writing compilers is fun.

[rpds]: https://github.com/orium/rpds
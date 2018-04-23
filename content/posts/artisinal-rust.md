---
title: "Writing safe zero-cost abstractions in Rust"
date: 2018-04-23T12:03:56+02:00
draft: true
---

OUTLINE:
Zero-cost abstractions conceptually
Zero-cost abstractions in the stdlib
How to write your own zero-cost abstractions
Unsafe fastest-possible implementation with a safe, checked wrapper (so that others can create their own ZCAs)

Rust, to put it bluntly, is really fast. Speed is one of its major tenets, in fact. When I came to Rust, however, I had pretty much no interest in fast code or systems. I had written some C before but much more as an intellectual exercise, as one would write code in Brainfuck.

Rust's insistence on zero-cost abstractions got me interested, however, and after reading discussions by the Rust team, reading resources on compilers and even writing some code directly in LLVM's intermediate representation I started to understand what made Rust so much faster than code written in a language like Python or Haskell or JavaScript, languages that Rust takes so much influence from and shares a lot of superficial similarities with, while being so much safer than a language like C.

For example, let's look at some really simple-looking code in Rust:

```rust
fn print<T: Display>(to_print: &T) {
    println!("{}", to_print);
}

for val in &[1u8, 2, 3, 4, 5] {
    print(val);
}
```

This looks pretty similar to equivalent code in Python or JavaScript, and is much safer than the C idiom:

```c
unsigned char values[5] = {1, 2, 3, 4, 5},
    *a = (unsigned char*)&values;
int i,
    length = 5;

for (i = 0; i < length; i++) {
    printf("%u", a[i]);
}
```

(What if I change the length of `values` but not `length`? What if `values` is not an array of `char` but of some other type? The compiler won't tell you because you're doing a pointer cast and it assumes you know what you're doing)

As we'll see, though, the Rust code is actually more efficient than the unsafe C.

You see, the Rust code above is actually equivalent to:

```rust
let mut iter = [1u8, 2, 3, 4, 5].iter();
while let Some(val) = iter.next() {
    print(val);
}
```

So, what does `[1u8, 2, 3, 4, 5].iter()` return? Looking at the docs, we see it creates a `std::slice::Iter`. So, what does `slice::Iter` look like? It looks like this:

```rust
pub struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>,
}
```

The `PhantomData` is just to allow us to bind this type to the lifetime of the slice (we'd get errors about `'a` being unused otherwise), the actual data is in `ptr` and `end`. So, the former is the current position in the array, the latter is the end of the array (i.e. the first invalid index into this slice, so `iter.next()` returns `None` when `iter.ptr == iter.end`). Each call to `iter.next()` looks like this:

```rust
fn next(&mut self) -> Option<&'a T> {
    unsafe {
        if self.ptr == self.end {
            None
        } else {
            let current = self.ptr;
            self.ptr = self.ptr.offset(1);
            Some(&*current)
        }
    }
}
```

This means that when we strip away all our abstractions, our code above is actually:

```rust
let array: &[u8] = &[1, 2, 3, 4, 5];
let mut iter_ptr = array.as_ptr();
let iter_end = iter_ptr.offset(array.len() as isize);

loop {
    if iter_ptr == iter_end { break; }

    let val = unsafe { &*iter_ptr };

    print(val);

    let current = iter_ptr;
    iter_ptr = iter_ptr.offset(1);
}
```

This is actually pretty simple to compile to assembly by hand, but we'll get to that later.
So why do I say this is faster than the C above? Because of operator strength. So although `a < b` and `a == b` are both a single instruction that on the CPU, that doesn't mean they take the same amount of time. `==` is faster for the CPU to evaluate than `<`. Not only this, but the C version does an extra addition for each iteration of the loop. Can you spot it? It's in `a[i]`. This is actually just a shorthand for `a + i`, you can even write `i[a]` and it has the same effect (yes, really, go try it). This means that each iteration of the C loop the CPU executes:

- One comparison: `i < length`.
- Two additions: `i++` and `a[i]`.

For each iteration of the Rust loop the CPU executes:

- One equality: `iter.ptr != iter.end`.
- One increment: `iter.ptr = iter.ptr.offset(1)`, this is compiled to a single instruction.

Of course, modern C compilers recognise this idiom and so both compilers end up producing similar code, but Rust manages to achieve this without sacrificing safety.

There's one element of the example we glossed over though, and that's generics. In dynamic languages everything is generic by default, since everything dispatches on the type of the value at runtime. In C there are a few options: either macros (which is essentially copy-pasting the code so it gets type-checked separately at each callsite), passing them in some form without a type (like a union or a void pointer) along with some kind of metadata that allows the function to convert it back to some usable type, and having a separate function for each type (possibly itself generated with a macro). The last method is the fastest, since it's easier to optimise for the compiler.

So what does Rust do? Actually, it supports safe versions of all three techniques that C supports. Macros are macros, Rust's obviously work very differently to C's but that's a topic for another article. The void-pointer method is made safe in basically the same way as slices - for slices you store the length along with the pointer, for `Any` you store a globally-unique identifier representing the type along with the pointer (so that `<Box<Any>>::downcast` can test whether or not the type ID matches). Rust's generics are similar to macro-generated type-specific functions in that they create one function in the binary for each type they are used with. They are much faster than the version found in dynamic languages since no runtime work has to be done and they are more ergonomic than the equivalent in C since you don't need to manually call the macro with each type you want to use (it's done automatically for you). Safer, too, since the compiler checks that the type satisfies the constraints on the function.

The following code:

```rust
fn add<T: Add<Output = T>>(a: T, b: T) -> T {
    a + b
}

let a = add(1u32, 2u32);
let b = add(1u64, 2u64);
```

is converted to something like:

```rust
fn add_u32(a: u32, b: u32) -> u32 {
    u32::add(a, b)
}

fn add_u64(a: u64, b: u64) -> u64 {
    u64::add(a, b)
}

let a = add_u32(1u32, 2u32);
let b = add_u64(1u64, 2u64);
```

The alternative would be to create a single function which takes a pointer to the `add` function and jumps to the function that this points to, but this is slower since the CPU finds it hard to predict "dynamic jumps", as these are called. Most CPUs won't attempt to predict dynamic jumps at all. This matters because it causes the CPU to clear the [instruction pipeline](https://en.wikipedia.org/wiki/Instruction_pipelining). However, it can lead to unexpected results. For example, it's possible to accidentally bloat the size of your binary with unnecessary copies of the same function by using a function that takes a `T: Into<Foo>`. Going back to our previous example:

```rust
fn really_long_function<T: Into<Foo>>(a: T) {
    let a = a.into();
    
    // Lots of complicated stuff here
}

really_long_function(Bar);
really_long_function(Baz);
```

Will be compiled to

```rust
fn really_long_function_Bar(a: Bar) {
    let a = a.into();
    
    // Lots of complicated stuff here
}

fn really_long_function_Baz(a: Baz) {
    let a = a.into();
    
    // Lots of complicated stuff here
}

really_long_function_Bar(Bar);
really_long_function_Baz(Baz);
```

Meaning the really complicated stuff gets duplicated despite (in theory) being precisely the same. This is bad for binary size but also prevents the CPU's [branch predictor](https://en.wikipedia.org/wiki/Branch_predictor) from learning which branches to take, potentially slowing down your code, not to mention the effects on compile time. A better method would be to have a monomorphic function that the polymorphic function delegates to, like so:

```rust
fn really_long_function<T: Into<Foo>>(a: T) {
    fn really_long_function_inner(a: Foo) {
        // Lots of complicated stuff here
    }
    
    really_long_function_inner(a.into())
}

really_long_function(Bar);
really_long_function(Baz);
```

Converting to

```rust
fn really_long_function_inner(a: Foo) {
    // Lots of complicated stuff here
}

fn really_long_function_Bar(a: Bar) {
    really_long_function_inner(a.into())
}

fn really_long_function_Baz(a: Baz) {
    really_long_function_inner(a.into())
}

really_long_function(Bar);
really_long_function(Baz);
```

Which means that the complicated code is only compiled once.

Now we can do some hand-compilation and get code that looks like this. This isn't much different than the assembly produced by `cargo build --release`:

```gas
.main:
    # Save the existing values of `%rax` and `%rbx` since we want to use
    # them. The `q` suffix on all of these instructions means that they
    # operate on 64-bit registers. The CPU cares about this, of course,
    # but you can just use `push` and let the assembler infer the size
    # of the operands.
    pushq %rax
    pushq %rbx

    # Set `%rax` to be a pointer to the start of the array.
    leaq  .array, %rax
    
    # Set `%rbx` to be `5 + %rax`
    movq  $5,     %rbx
    addq  %rax,   %rbx

    # If `%rax` is the same as `%rbx`, jump past the loop (this is
    # technically unnecessary but it makes the assembly match the Rust).
    cmpq  %rax,   %rbx
    je    .loop-end
    
.loop:
    # Move `%rax` into a `d` register to pass it to `print`. `print` can
    # then just use `%rdi` to refer to the parameter.
    movq  %rax,   %rdi
    callq print
    
    # Increment the pointer (this is an array of bytes and so we increase
    # by 1, if this was an array of `u64`s we'd increment by 8).
    addq  $1,     %rax
    
    # Compare `%rax` and `%rbx` and jump back to the start of the loop if
    # they're still not equal.
    cmpq  %rax,   %rbx
    jne   .loop
    
.loop-end:
    # Restore the value of the intermediate registers that we used
    popq  %rbx
    popq  %rax
    retq

.array:
    .byte 1, 2, 3, 4, 5
```


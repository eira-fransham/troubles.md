---
title: "WebAssembly Troubles part 1: WebAssembly Is Not a Stack Machine"
date: 2019-01-30T14:13:48+01:00
draft: false
---

> ### Preamble
> 
> This is part 1 of a 4-part miniseries on issues with WebAssembly and proposals to fix them.
> <br/><br/>
> This article assumes some familiarity with virtual machines, compilers and WebAssembly, but I'll try to link to relevant information where necessary so even if you're not you can follow along.
> <br/><br/>
> Also, this series is going to come off as if I dislike WebAssembly. I love WebAssembly! I wrote a [whole article about how great it is][wasm-on-the-blockchain]! In fact, I love it so much that I want it to be the best that it can be, and this series is me working through my complaints with the design in the hope that some or all of these issues can be addressed soon, while the ink is still somewhat wet on the specification.

[wasm-on-the-blockchain]: {{< ref "/posts/why-wasm.md" >}}

I'm sure you're all familiar with WebAssembly by now. It's seeing use everywhere, from plugins to blockchain smart contracts to, of course, the web. If you go to the Wikipedia article for WebAssembly right now, you'll get a great overview of the technology, except for one thing: the "Design" section states:

> WebAssembly code is intended to be run on a portable abstract structured stack machine

Nope.

## Stack machines vs register machines

The difference between a stack and a register machine is essentially this: a stack machine pops values from and pushed values onto a stack, so for example a `+` operation will pop the last two values off of the stack, add them together, and then push the result back onto the stack. A register machine has some number of places where values can be stored, and operations reads and writes to those registers. So `+` will take 3 arguments, 2 that represent the registers to read the operands from and 1 that represents the register to write the result to[^acktually].

[^acktually]: Yes, I know that many register machines such as x86 output into one of the input registers, but that's still conceptually the same as taking separate arguments for input and output, just more limited.

The problem with registers is that there is no liveness analysis. If you have some code like so:

```
%0 = 1
%1 = 2
%2 = add %0, %1
...
```

Are `%0` and `%1` used after the addition? Without lookahead, you have no way of knowing. Even in SSA[^ssa] register machines like LLVM IR (where each register is written to exactly once) the liveness of each register must be continuously recalculated. This means that you have overhead associated with compilation - knowing the liveness of variables is extremely important for generating efficient assembly, but instead of the liveness being calculated when creating the IR and stored as a part of it you have to recalculate this data every time.

[^ssa]: if you don't know what SSA form is you can see [the Wikipedia article on it][wikipedia-ssa]

[wikipedia-ssa]: https://en.wikipedia.org/wiki/Static_single_assignment_form

If the IR is entirely in-memory this isn't so bad, especially if your code is in SSA form, but it means that you have to do extra work for the same quality of code and it means that a streaming compiler (like FireFox's Wasm baseline compiler) is unable to produce good code.

You can, of course, add liveness analysis metadata to the machine, but that liveness analysis is only useful if your code is in SSA form - if it's not SSA form then the liveness is _extremely_ coarse-grained.

Well, that's where WebAssembly comes in. See, WebAssembly has these things called locals. Locals are mutable variables that live for the lifetime of a function. Since WebAssembly blocks can't take arguments, they're the only way for blocks to receive data from outside. This includes stuff like loop counters, the classic example for SSA-defeating mutable variables, but it also includes regular blocks. For example:

```lisp
(func (result i32)
  (add (i32.const 1) (i32.const 2))
  (block (result i32)
    ; Can't read the result of the add above
  ))
```

You have to do this:

```lisp
(func (result i32)
  (local i32)
  (local.set 0 (add (i32.const 1) (i32.const 2)))
  (block (result i32)
    (local.get 0)
  ))
```

This is a problem. Usually a stack machine can be trivially converted to SSA form with liveness analysis - an item on the stack is dead when it is used as the argument to an operator - no ifs, no buts. Even a `pick` operator[^pick-operator] can be thought of in these terms as long as the argument to `pick` is a compile-time constant, although generally the SSA conversion of a stack machine that includes a `pick` instruction would be implemented using refcounting - meaning that reading a variable doesn't involve creating an unnecessary new one. However, locals are a problem. They're mutable, so you can't trivially convert them to SSA, and they always live for the lifetime of the function.

[^pick-operator]: A `pick` operator duplicates the nth item of the stack to the top of the stack. Unfortunately, it's not available in WebAssembly.

This poses a problem for optimisation. SSA form is one of the most powerful optimisation tools. The fact that locals are mutable defeats that, and the fact that they're global to the function defeats liveness analysis. For an example of why this is an issue, say you're trying to compile WebAssembly into some native architecture. You'll compile your code into one that uses some number of arguments in registers, but now unless you do your own liveness analysis on that function those registers are permanently marked "in use" by those arguments. Without extra analysis you don't know when the last usage of that argument is, and so you can't ever reuse the register used by that argument for something else. Even a function that never uses any of its arguments still can't reuse the space used by those arguments for anything else.

This essentially makes WebAssembly a register machine without liveness analysis, but not only that, it's a register machine that isn't even in SSA form - both of the tools at our disposal to do optimisation are unavailable. In a true, optimising compiler we can recreate that information, but WebAssembly was already emitted by a compiler that generated that information once. There's no technical reason why we should have to do that work again, it's just a deficiency in the language. A compiler that has to act on a stream of WebAssembly has less ability to recreate this information and will end up generating significantly worse code for essentially no good reason.

## Why?

The developers of the WebAssembly spec aren't dumb. For the most part it's an extremely well-designed specification. However, they are weighed down by WebAssembly's legacy. WebAssembly started out not as a bytecode, but more like a simplified binary representation for asm.js. Essentially it was originally designed to be source code, like JavaScript. It would be a more-efficient representation thereof but it still wasn't a proper virtual machine instruction set. Then, it became a register machine[^infinite-registers], and only at the last minute did it become a stack machine. At that point, concepts like locals were quite entrenched in the spec. Not only that, but for the most part the WebAssembly specification team were flying blind. No streaming compiler had yet been built, hell, no compiler had yet been built. It wasn't clear that having locals would be problematic - after all, C gets by just fine using local variables that the compiler constructs the SSA graph for.

[^infinite-registers]: I believe that it had infinite registers and so it was possible to generate it in SSA form, but it had no liveness analysis.

It's only by working on a streaming WebAssembly compiler for integration with [Wasmtime][wasmtime] that I realised these issues, as well as more issues that I'll get to in later articles in this series. Before then I considered WebAssembly's design to be utterly rock-solid, and in fact I still strongly believe that most of the decisions made were the right ones. Although it has problems, it's incredible how much the WebAssembly working group got right considering it was such relatively unknown territory at the time of the specification's writing.

[wasmtime]: https://github.com/CraneStation/wasmtime

## What's the alternative?

So currently locals represent both function arguments and local variables. There is [a proposal to allow blocks to have arguments and both functions and blocks to return more than one value][multi-return]. With this, it becomes possible to get rid of locals entirely. Loop counters are implemented by having the counter be given as an argument, and then when you jump to the header of the loop you must have the new value of the counter on the stack. This acts like a phi function but is much conceptually simpler. Since it's impossible to get the address of locals in WebAssembly there's no need for `alloca` in any form - unlike in LLVM, you don't need to have locals at all. The only thing left is to implement function arguments by having the function start with its arguments on the stack and you no longer need locals at all.

[multi-return]: https://github.com/WebAssembly/multi-value/blob/master/proposals/multi-value/Overview.md

This massively reduces the complexity of the WebAssembly specification and compilers for it without reducing the expressivity, and allows the compilers generating the WebAssembly to encode more of their knowledge about the original source into the WebAssembly itself. Although optimising compilers will probably not generate better code from this, they will become much simpler and easier to maintain, and streaming compilers will become significantly simpler and produce significantly better code.

Finally, not only is a stack machine like this automatically in SSA form, but it's automatically in _strict_ SSA form. This means that it is statically impossible to access undefined variables. In WebAssembly as it exists now, you have to zero locals when entering a function unless you can statically prove that the local is always set before it is accessed. Again, the compilers almost always have this information, and removing locals makes accessing undefined memory statically impossible.

Join me next time where I tackle WebAssembly's problematic control flow.

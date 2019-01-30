---
title: "WebAssembly Troubles part 2: Why Do We Need the Relooper Algorithm, Again?"
date: 2019-01-30T21:28:10+01:00
draft: false
---

> ### Preamble
> 
> This is part 2 of a 3-part miniseries on issues with WebAssembly and proposals to fix them. [Part 1][part-1].
> <br/><br/>
> This article assumes some familiarity with virtual machines, compilers and WebAssembly, but I'll try to link to relevant information where necessary so even if you're not you can follow along.
> <br/><br/>
> Also, this series is going to come off as if I dislike WebAssembly. I love WebAssembly! I wrote a [whole article about how great it is][wasm-on-the-blockchain]! In fact, I love it so much that I want it to be the best that it can be, and this series is me working through my complaints with the design in the hope that some or all of these issues can be addressed soon, while the ink is still somewhat wet on the specification.

[wasm-on-the-blockchain]: http://troubles.md/posts/why-wasm/
[part-1]: http://troubles.md/posts/wasm-is-not-a-stack-machine/

So if there's one thing people know about WebAssembly control flow, it's that it doesn't allow `goto`. `goto`, we're told, is [considered harmful][goto-considered-harmful], and so WebAssembly only implements simple control flow constructs:

* `block`, which you can break to the end of;
* `loop`, which you can jump to the header of;
* `if`, which allows you to choose either the `then` or `else` branch depending on a condition;
* `br`, which unconditionally jumps to a block's "target" (so the end for a `block` or `if`..`else` and the beginning for a loop);
* `br_table`, which can jump to the target of one of many blocks based on a runtime value;
* and `br_if`, which either jumps to the block's target or continues depending on a condition.

[goto-considered-harmful]: https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf

## Redundancy

The eagle-eyed among you may have noticed that there is already some redundancy in those constructs. `if`..`else` can be implemented easily using `block` and `br_if`. It looks something like so:

```lisp
; Version using `if`
(if (result i32) some_condition
  (then
    ; ..then branch..
  )
  (else
    ; ..else branch..
  ))
; Version using `block` and `br_if`
(block (param $n i32) (result i32)
  (block (param $n i32) (result i32)
    (br_if 0 $n)
    ; ..then branch..
    (br 1))
  ; ..else branch..
)
```

Except whoops! No it isn't! In WebAssembly, blocks can't take arguments. I touch on this in my previous article but there's [a proposal to fix this][multi-return]. Let's say this gets fixed though, and blocks gain the ability to take arguments. In this case, not only are these two pieces of code identical, but even a baseline non-optimising compiler will produce identical assembly for both of them. The second one takes more bytes to represent but it's a simple recurring pattern, which makes it easy to write domain-specific compression for it[^macros].

[multi-return]: https://github.com/WebAssembly/multi-value/blob/master/proposals/multi-value/Overview.md

[^macros]: The simplest compression method would be to implement macros - in the header you define a macro that takes some arguments and produces one or more instructions. Probably you'd want to restrict it to only generate balanced block start and end tags but that wouldn't be necessary for correct behaviour.

## Deeper down the rabbit hole

It's not news to most programmers that really all of these constructs are redundant if WebAssembly had `goto`. If you had `goto` and some `goto_if` variant that conditionally jumped to a label you could implement all of these constructs. So why doesn't WebAssembly support `goto`?

The classic argument against `goto` is that it leads to spaghetti code. No-one's writing raw WebAssembly though (or rather, most people aren't writing raw WebAssembly), so what's the problem? Well, WebAssembly does have some constraints, specifically it must produce code that is valid. WebAssembly is a stack machine, and you can't jump to just any label since that label might point to code that pops too many values off the stack, or pops the wrong types off the stack, or pushes too many values onto the stack. Essentially raw `goto`s allow you to do all kinds of evil stuff. What you really want is a list of `block`s like WebAssembly has now, each taking some number of arguments and returning some number of values onto the stack, and that you can call like function. Now you can have arbitrary flow between these blocks, like a typed version of `goto`.

Now I'm not some visionary genius and I didn't come up with this idea, nor was I the first to apply this to WebAssembly. The concept has been implemented in compilers for a very long time, where it is known as a [control flow graph][https://en.wikipedia.org/wiki/Control_flow_graph] or CFG. A CFG is usually combined with a code representation in SSA form, and if you've read my previous article you probably know where this is going by now but let's work through it together anyway. As for implementing it in WebAssembly, I was introduced to this as it applies to WebAssembly by the author of the [WebAssembly Funclets proposal][funclets], which implements this basic idea.

[funclets]: https://github.com/WebAssembly/funclets/blob/master/proposals/funclets/Overview.md

## Why bother?

CFGs are the backbone of almost every static language compiler out there. Both LLVM and GCC use a CFG to combine the different control flow constructs like `for`, `while`, `if` and even `goto` into a single representation that can be used to define them all. This causes a problem for compiling to WebAssembly. Since WebAssembly doesn't allow arbitrary control-flow graphs, compilers have to detect when some set of nodes in the CFG can be represented using WebAssembly's control flow constructs and emit them, falling back to a complicated algorithm like [Relooper][relooper] or Stackifier (there's no documentation online, it's only mentioned by this name in LLVM internal discussions) to emulate complex control flow. Essentially the fallback algorithm converts the CFG into a the equivalent of a loop combined with a switch statement, which can be extremely slow.

[relooper]: http://mozakai.blogspot.com/2012/05/reloop-all-blocks.html

Currently compilers that convert WebAssembly to native code cannot generate good code for this "fallback" code, despite the fact that supporting arbitrary CFGs in WebAssembly comes with no downside in terms of runtime performance, security or correctness. Supporting arbitrary CFGs would massively simplify both the generation of WebAssembly by LLVM and the compilation of WebAssembly to native code[^compilation-to-native]. WebAssembly control flow is a lossy conversion - we start with a CFG in LLVM which gets converted to WebAssembly control flow, then the native compiler converts that control flow back into a less-efficient CFG which then has optimisations applied and is then compiled to native. It would also allow GCC to generate WebAssembly - developers simply gave up implementing a WebAssembly backend for GCC as it was much too difficult to correctly implement a relooper-style algorithm over their CFG representation.

[^compilation-to-native]: I should note that it will only actually simplify the compilation to native for simple or streaming compilers, since for production-grade optimising compilers they convert the WebAssembly control flow back into a CFG.

## Why is this not already implemented?

Well for a start it requires arguments to blocks and multiple returns to be supported, neither of which are in the MVP. As for why those aren't supported in the MVP, I don't really have a good answer. As with my previous article, the question really comes down to WebAssembly's history as a statically-typed binary encoding of JavaScript and the lack of serious implementations while the specification was being written. Unlike the previous article, I find it utterly baffling that the issues with compiling to WebAssembly's inexpressive control flow from any modern compiler weren't so glaringly obvious as to prevent the release of the MVP until they were fixed. I wasn't there for the development of the specification though and so I can't speak for what was going on behind the scenes.

Join us next time for a much more positive post with my ideas about how to fix some of these issues and more in the medium-term.

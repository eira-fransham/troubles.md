---
title: "In which the CPU changes my data under my nose"
date: 2018-05-15T12:19:37+02:00
draft: false
---

IEEE 754 has been the source of jokes and confusion for a while because of its [imprecise representation of decimal numbers][decimal-numbers], but the truth is that it's an easily-implementable, efficient representation of floating-point numbers and a lot of applications that don't need high precision couldn't easily do without it. It's not without its peculiarities though. Or rather, pecularity, singular. Specifically, the existence of NaN. NaN is a weird beast, taking up [16777208][nan-values] possible values that could have otherwise been used for real data (although some sneaky people get away with using them [for special values][nan-boxing]) and [forcing Rust to split its comparison traits into "partial" and "total" variants][comparisons]. A lot of developer time and effort has been spent working out precisely how to handle NaN, and this is the story of my recent personal contribution to that time and effort.

[decimal-numbers]: https://0.30000000000000004.com/
[nan-values]: https://en.wikipedia.org/wiki/Single-precision_floating-point_format#Exponent_encoding
[nan-boxing]: https://softwareengineering.stackexchange.com/questions/185406/what-is-the-purpose-of-nan-boxing
[comparisons]: https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html

This particular story about NaN starts with a strange bug in [`wasmi`][wasmi], our experimental WebAssembly interpreter for blockchains here at [Parity][parity]. The tests were failing when compiled for 32-bit, but not for 64-bit. After rewriting our WebAssembly test harness to correctly report failures, I narrowed it down to a few test cases that looked something like the following:

[wasmi]: https://github.com/paritytech/wasmi
[parity]: https://github.com/paritytech/

```lisp
(module
  ;; ...
  (func (export "f32.reinterpret_i32") (param $x i32)
                                       (result f32)
        (f32.reinterpret/i32 (get_local $x)))
  (func (export "i32.reinterpret_f32") (param $x f32)
                                       (result i32)
        (i32.reinterpret/f32 (get_local $x)))
  ;; ...
)

;; ...

(assert_return (invoke "f32.reinterpret_i32"
                       (i32.const 0x7fa00000))
               (f32.const nan:0x200000)) 
(assert_return (invoke "f32.reinterpret_i32"
                       (i32.const 0xffa00000))
               (f32.const -nan:0x200000))
(assert_return (invoke "i32.reinterpret_f32"
                       (f32.const nan:0x200000))
               (i32.const 0x7fa00000)) 
(assert_return (invoke "i32.reinterpret_f32"
                       (f32.const -nan:0x200000))
               (i32.const 0xffa00000))
```

All other tests were succeeding, but doing a bitwise cast from integer to float and vice-versa was failing. Furthermore, these tests all succeeded on x86_64 (i.e. when compiled for 64-bit), they only failed on x86 (i.e. when compiled for 32-bit). Lastly, the difference between the expected value and the received value was only one bit. Instead of returning a float with the bit pattern `0x7fa00000` it would return `0x7fe00000` instead. If you've worked with NaNs before, you might have noticed something:

```
Expected: 0b11111111010000000000000000000000000
Got:      0b11111111110000000000000000000000000
                    ^ This bit is different
```

That's right, it's the [quiet bit][snan]! For those unaware, the quiet bit determines whether or not the number should raise an exception when you attempt to use it. This is because the [IEEE 754 standard][ieee754] was written at a time when it wasn't necessarily expected that a programming language would include exceptions. The standard includes provisions to allow hardware-level exceptions when, for example, dividing zero by zero. The idea is that when the hardware tries to execute an invalid floating point operation it is allowed to generate a "signalling NaN", or sNaN. This is what we see above - the quiet bit is set to 0, meaning that the NaN is signalling[^ieee-x86]. When you try to use this value (for example, by multiplying it with another number) the hardware is allowed to generate an exception and then return a "quiet NaN", or qNaN, which has the quiet bit set to 1. I say "allowed to" rather than "does" or "should" because IEEE 754 leaves this unspecified, and in fact as far as I can tell x86 and ARM never generate sNaNs, the only way to create them is to reinterpret an integer with the signalling bit set. Originally, I thought the source of the problem was obvious: [Rust was quieting NaN when casting an int to a float][rust-cast]. I remember seeing [tomaka][tomaka]'s[^troubles] talk at RustFest 2017 on binding C libraries where he mentioned that Rust guarantees that all NaNs are quiet and that you have to explicitly handle that when using floats returned from C (where sNaNs are allowed). So I checked the source of `f32::from_bits` and... [nope][from_bits]. It just does a normal reintepret. Turns out tomaka's advice was out of date, and Rust [now allows sNaNs][rust-snans]

[snan]: https://en.wikipedia.org/wiki/NaN#Signaling_NaN
[ieee754]: https://en.wikipedia.org/wiki/IEEE_754
[rust-cast]: https://github.com/rust-lang/rust/pull/39271
[tomaka]: https://github.com/tomaka
[from_bits]: https://github.com/rust-lang/rust/blob/8010604b2d888ac839147fe27de76cdcc713aa1b/src/libcore/num/f32.rs#L270-L275
[rust-snans]: https://github.com/rust-lang/rust/pull/46012/commits/439576fd7bedf741db5fb6a21c902e858d51f2a0

[^ieee-x86]: Quick note: this is only true for x86, the IEEE 754 spec leaves it unspecified whether 0 or 1 represents signalling, as long as the bit mask is the same (`1 << (sizeof(float) * 8 - 6)`). Why they thought it was necessary to leave this unspecified instead of just picking one and sticking with it is beyond me. Maybe on some CPUs it's faster to unset a bit and on others it's faster to set it (let's be real here though, probably not).

[^troubles]: Tomaka inspired the name of this blog with his file [TROUBLES.md][troubles-md]. I originally bought this domain because he mentioned that he was annoyed that TROUBLES.md kept being brought up in discussions about Rust language design. Yeah, this blog basically started as a way for me to troll my mate.

[troubles-md]: https://github.com/vulkano-rs/vulkano/blob/master/TROUBLES.md

So the hunt was on - where are we operating on this float value? Looking at the code it looked like we just take the number directly from the WebAssembly bytecode and interpret it as a float, no operations required. The bytecode must be correct - after all, the same code compiled for 64-bit worked correctly, and they both read the same bytecode. Since reading the code wasn't helping me at all, I opened up the LLDB debugger[^lldb-debugger] and attached it to the test binary that is output by `cargo test --no-run`. Stepping through functions and checking their return value, I eventually found the culprit:

[^lldb-debugger]: Yes, [that's the official name][lldb], assumably DB stands for something other than debugger, like decibel or Deutsche Bahn.

[lldb]: https://lldb.llvm.org/

```rust
fn run_reinterpret<T, U>(
  &mut self,
  context: &mut FunctionContext
) -> Result<InstructionOutcome, TrapKind>
where
  RuntimeValue: From<U>,
  T: FromRuntimeValue,
  T: TransmuteInto<U>
{
  let v = context
    .value_stack_mut()
    .pop_as::<T>();

  let v = v.transmute_into();
  //      ^-This call here-^

  context.value_stack_mut().push(v.into())?;

  Ok(InstructionOutcome::RunNextInstruction)
}
```

That `v.transmute_into()` call was returning `0x7fe00000` instead of `0x7fa00000`! Hang on, like the name implies, `v.transmute_into()` should just be a transmute, it should just reinterpret the bits. I looked at the source code, and sure enough:

```rust
impl TransmuteInto<f32> for i32 {
  fn transmute_into(self) -> f32 {
    f32::from_bits(self)
  }
}
```

We're back where we started. As we saw before, `f32::from_bits` is just a transmute. Originally I thought it was an old version of the standard library, so I wrote a quick test:

```rust
use test::black_box as bb;

#[test]
fn it_works() {
  assert_eq!(
    bb(0xffa00000),
    bb(
      bb(f32::from_bits(bb(0xffa00000)))
        .to_bits()
    )
  );
}
```

The `black_box`/`bb` function just ensures that the compiler doesn't optimise anything out. This particular test succeeds, though, even on x86 (i.e. 32-bit, the same architecture where our `wasmi` test suite fails). So I went back to `f32::from_bits` in the compiled output for `wasmi` and disassembled it with LLDB.

```gas
core::f32::<impl core::num::Float for f32>::from_bits:
  pushl  %ebp
  movl   %esp,        %ebp
  subl   $0x10,       %esp
  movl   0x8(%ebp),   %eax
  movl   0x8(%ebp),   %ecx
  movl   %ecx,        -0x4(%ebp)
  movss  -0x4(%ebp),  %xmm0
  movl   %eax,        -0xc(%ebp)
  movss  %xmm0,       -0x10(%ebp)
  movss  -0x10(%ebp), %xmm0
  movss  %xmm0,       -0x8(%ebp)
  flds   -0x8(%ebp)
  addl   $0x10,       %esp
  popl   %ebp
  retl   
```

At first glance, that looks like it's doing a bunch more stuff than just a transmute, but it's actually just moving data about because the test harness is compiled without optimisations by default. If you turn optimisations on, this just compiles to:

```gas
core::f32::<impl core::num::Float for f32>::from_bits:
  flds   0x4(%esp)
  retl   
```

Hang on, what's this `flds` instruction? We see it in the debug version above, too. It's not there if you disassemble the build for 64-bit:

```gas
core::f32::<impl core::num::Float for f32>::from_bits:
  movd   %edi, %xmm0
  retq   
```

So what does it do? I'm pretty familiar with x86 assembly, but I'd never seen this instruction before. Googling it brings up [this page in the x86 reference][fld], which says that it pushes the argument onto the FPU register stack. See, what most people call x86 is actually an extension known as [x87][x87], which implementes floating point operations on top of x86. One common [calling convention][calling-conv] for x87 is to return integer arguments in `%eax` and floating point arguments on the FPU stack, using the `fstps` instruction to load the returned value after control is returned to the caller. This is what `flds` is doing here, returning the argument as a float. Ah, but whenever the CPU operates on a float it's allowed to quieten it, and sure enough `fld` quietens sNaNs that were passed to it. With LLDB we can read the state of the registers and memory, so let's check out what happens between `flds` and `fstps`.

[fld]: https://c9x.me/x86/html/file_module_x86_id_100.html
[x87]: https://en.wikipedia.org/wiki/X87
[calling-conv]: https://en.wikipedia.org/wiki/X86_calling_conventions

```
(lldb) run
## SNIP ##
->  0x5655b5e0 <+0>: flds   0x4(%esp)
    0x5655b5e4 <+4>: retl   
## SNIP ##
```

You can see that we've stopped on our problematic `flds` instruction. Let's check out the data pointed to by `0x4(%esp)` (i.e. `%esp` plus `0x4`).

```
(lldb) register read esp
    esp = 0xf77fee3c
(lldb) x/xw '0xf77fee3c + 0x4'
0xf77fee40: 0xffa00000
```

There we go, that's our signalling NaN. So it's passed into the function OK. Let's see what happens in the calling code:

```
(lldb) si
## SNIP ##
->  0x5655b55c <+44>: fstps  0x14(%esp)
    0x5655b560 <+48>: addl   $0x18, %esp
    0x5655b563 <+51>: popl   %ebx
    0x5655b564 <+52>: retl   
```

So we're taking the top of the floating point stack and storing it in `0x14(%esp)`. Let's step one more time (to let the `fstps` instruction run) and read the value of that memory address:

```
(lldb) si
## SNIP ##
->  0x5655b560 <+48>: addl   $0x18, %esp
    0x5655b563 <+51>: popl   %ebx
    0x5655b564 <+52>: retl   
    0x5655b565:       nop    
(lldb) register read esp
     esp = 0xf77fee40
(lldb) x/xw '0xf77fee40 + 0x14'
0xf77fee54: 0xffe00000
```

There we go, that's a quiet NaN. That's the first time that I've seen _the very act of returning something from a function_ cause tests to fail where they would have succeeded if it had been done inline.

So the solution? I wrote [a wrapper around `u32`/`u64`][npf] that converts to `f32`/`f64` when doing operations. This essentially fools the CPU into thinking that these values are integers unless it's doing binary operations on them, and so when you pass them in and out of functions it uses the `%eax` register and preserves their bit patterns. Since WebAssembly requires that you quieten NaNs when doing binary operations, this doesn't cause a problem.

If I've learned anything from this, it's that no matter how close to the metal you go, you can never truly have total control over the execution of your code. I truly believe that the ability to do CPU-level debugging and to read disassembly is still important even in the age of high-level languages, because [all abstractions are leaky][leaky].

[npf]: https://github.com/Vurich/nan-preserving-float/blob/master/src/lib.rs
[leaky]: https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/

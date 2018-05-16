---
title: "How a Rust upgrade more than tripled the speed of my code"
date: 2018-05-11T16:07:31+02:00
draft: false
---

I'd like to share a quick story about the sheer power of LLVM and the benefits of using higher-level languages over assembly.

I work at Parity Technologies, who maintains the [Parity Ethereum client](https://github.com/paritytech/parity). In this client we have a need for performant 256-bit arithmetic, which we have to emulate in software since no modern hardware supports it natively.

For a long time we've maintained parallel implementations of arithmetic, one in Rust for stable builds and one in inline assembly (which is automatically used when you compile with the nightly compiler). We do this because we store these 256-bit numbers as arrays of 64-bit numbers and there is no way to multiply two 64-bit numbers to get a more-than-64-bit result in Rust (since Rust's integer types only go up to `u64`). This is despite the fact that x86_64 (our main target platform) natively supports 128-bit results of calculations with 64-bit numbers. So, we resort to splitting the 64-bit numbers into two 32-bit numbers (because we _can_ multiply two 32-bit numbers to get a 64-bit result).

```rust
impl U256 {
  fn full_mul(self, other: Self) -> U512 {
    let U256(ref me) = self;
    let U256(ref you) = other;
    let mut ret = [0u64; U512_SIZE];


    for i in 0..U256_SIZE {
      let mut carry = 0u64;
      // `split` splits a 64-bit number into upper and lower halves
      let (b_u, b_l) = split(you[i]);

      for j in 0..U256_SIZE {
        // This process is so slow that it's faster to check for 0 and skip
        // it if possible.
        if me[j] != 0 || carry != 0 {
          let a = split(me[j]);

          // `mul_u32` multiplies a 64-bit number that's been split into
          // an `(upper, lower)` pair by a 32-bit number to get a 96-bit
          // result. Yes, 96-bit (it returns a `(u32, u64)` pair).
          let (c_l, overflow_l) = mul_u32(a, b_l, ret[i + j]);

          // Since we have to multiply by a 64-bit number, we have to do
          // this twice.
          let (c_u, overflow_u) = mul_u32(a, b_u, c_l >> 32);
          ret[i + j] = (c_l & 0xffffffff) + (c_u << 32);

          // Then we have to do this complex logic to set the result. Gross.
          let res = (c_u >> 32) + (overflow_u << 32);
          let (res, o1) = res.overflowing_add(overflow_l + carry);
          let (res, o2) = res.overflowing_add(ret[i + j + 1]);
          ret[i + j + 1] = res;

          carry = (o1 | o2) as u64;
        }
      }
    }

    U512(ret)
  }
}
```

You don't even have to understand all of the code to see how non-optimal this is. Inspecting the output of the compiler shows that the generated assembly is extremely suboptimal. It does much more work than necessary essentially just to work around limitations in the Rust language. So we wrote an inline assembly version. The important thing about using inline assembly here is that x86_64 natively supports multiplying two 64-bit values into a 128-bit result. When Rust does `a * b` when `a` and `b` are both `u64` the CPU actually multiplies them to create a 128-bit result and then Rust just throws away the upper 64 bits. We want the upper 64 in this case though, and the only way to access it efficiently is by using inline assembly.

As you can imagine, our assembly implementation was much faster:

```text
 name            u64.bench ns/iter  inline_asm.bench ns/iter  diff ns/iter   diff %  speedup 
 u256_full_mul   243,159            197,396                        -45,763  -18.82%   x 1.23 
 u256_mul        268,750            95,843                        -172,907  -64.34%   x 2.80 
 u256_mul_small  1,608              789                               -819  -50.93%   x 2.04 
```

`u256_full_mul` tests the function above, `u256_mul` multiplies two 256-bit numbers to get a 256-bit result (in Rust, we just create a 512-bit result and then throw away the top half but in assembly we have a seperate implementation), and `u256_mul_small` multiplies two small 256-bit numbers. As you can see, the assembly implementation is up to 65% faster. This is way, way better. Unfortunately, it only works on nightly, and even then only on x86_64. The truth is that it was a lot of effort and a number of thrown-away implementations to even get the Rust code to "only" half the speed of the assembly, too. There was simply no good way to give the compiler the information necessary.

All that changed with [Rust 1.26][rust-126]. Now we can do `a as u128 * b as u128` and the compiler will use x86_64's native u64-to-u128 multiplication (even though you cast both numbers to `u128` it knows that they're "really" just `u64`, you just want a `u128` result). That means our code now looks like this:

[rust-126]: https://blog.rust-lang.org/2018/05/10/Rust-1.26.html

```rust
impl U256 {
  fn full_mul(self, other: Self) -> U512 {
    let U256(ref me) = self;
    let U256(ref you) = other;
    let mut ret = [0u64; U512_SIZE];

    for i in 0..U256_SIZE {
      let mut carry = 0u64;
      let b = you[i];

      for j in 0..U256_SIZE {
        let a = me[j];

        // This compiles down to just use x86's native 128-bit arithmetic
        let (hi, low) = split_u128(a as u128 * b as u128);

        let overflow = {
          let existing_low = &mut ret[i + j];
          let (low, o) = low.overflowing_add(*existing_low);
          *existing_low = low;
          o
        };

        carry = {
          let existing_hi = &mut ret[i + j + 1];
          let hi = hi + overflow as u64;
          let (hi, o0) = hi.overflowing_add(carry);
          let (hi, o1) = hi.overflowing_add(*existing_hi);
          *existing_hi = hi;

          (o0 | o1) as u64
        }
      }
    }

    U512(ret)
  }
}
```

Although it's almost certainly not as fast as using the LLVM-native `i256` type, the speed is much, much better. Here it is compared to the original Rust implementation:

```text
 name            u64.bench ns/iter  u128.bench ns/iter  diff ns/iter   diff %  speedup 
 u256_full_mul   243,159            73,416                  -169,743  -69.81%   x 3.31 
 u256_mul        268,750            85,797                  -182,953  -68.08%   x 3.13 
 u256_mul_small  1,608              558                       -1,050  -65.30%   x 2.88 
```

Which is great, we now get a speed boost on stable. Since we only compile the binaries for the Parity client on stable the only people who could use the assembly before were those who compiled from source, so this is an improvement for a lot of users. But wait, there's more! The new compiled code actually manages to beat the assembly implementation by a significant margin, even beating the assembly on the benchmark that multiplies two 256-bit numbers to get a 256-bit result. This is despite the fact that the Rust code still produces a 512-bit result first and then discards the upper half, where the assembly implementation does not:

```text
 name            inline_asm.bench ns/iter  u128.bench ns/iter  diff ns/iter   diff %  speedup 
 u256_full_mul   197,396                   73,416                  -123,980  -62.81%   x 2.69 
 u256_mul        95,843                    85,797                   -10,046  -10.48%   x 1.12 
 u256_mul_small  789                       558                         -231  -29.28%   x 1.41 
```

For the full multiplication that's an absolutely massive improvement, especially since the original code used highly-optimised assembly incantations from our resident cycle wizard. Here's where the faint of heart might want to step out for a moment, because I'm about to dive into the generated assembly.

Here's the hand-written assembly. I've presented it without comment because I want to comment the assembly that is actually emitted by the compiler (since, as you'll see, the `asm!` macro hides more than you'd expect):

```rust
impl U256 {
  /// Multiplies two 256-bit integers to produce full 512-bit integer
  /// No overflow possible
  pub fn full_mul(self, other: U256) -> U512 {
    let self_t: &[u64; 4] = &self.0;
    let other_t: &[u64; 4] = &other.0;
    let mut result: [u64; 8] = unsafe { ::core::mem::uninitialized() };
    unsafe {
      asm!("
        mov $8, %rax
        mulq $12
        mov %rax, $0
        mov %rdx, $1
        mov $8, %rax
        mulq $13
        add %rax, $1
        adc $$0, %rdx
        mov %rdx, $2
        mov $8, %rax
        mulq $14
        add %rax, $2
        adc $$0, %rdx
        mov %rdx, $3
        mov $8, %rax
        mulq $15
        add %rax, $3
        adc $$0, %rdx
        mov %rdx, $4
        mov $9, %rax
        mulq $12
        add %rax, $1
        adc %rdx, $2
        adc $$0, $3
        adc $$0, $4
        xor $5, $5
        adc $$0, $5
        xor $6, $6
        adc $$0, $6
        xor $7, $7
        adc $$0, $7
        mov $9, %rax
        mulq $13
        add %rax, $2
        adc %rdx, $3
        adc $$0, $4
        adc $$0, $5
        adc $$0, $6
        adc $$0, $7
        mov $9, %rax
        mulq $14
        add %rax, $3
        adc %rdx, $4
        adc $$0, $5
        adc $$0, $6
        adc $$0, $7
        mov $9, %rax
        mulq $15
        add %rax, $4
        adc %rdx, $5
        adc $$0, $6
        adc $$0, $7
        mov $10, %rax
        mulq $12
        add %rax, $2
        adc %rdx, $3
        adc $$0, $4
        adc $$0, $5
        adc $$0, $6
        adc $$0, $7
        mov $10, %rax
        mulq $13
        add %rax, $3
        adc %rdx, $4
        adc $$0, $5
        adc $$0, $6
        adc $$0, $7
        mov $10, %rax
        mulq $14
        add %rax, $4
        adc %rdx, $5
        adc $$0, $6
        adc $$0, $7
        mov $10, %rax
        mulq $15
        add %rax, $5
        adc %rdx, $6
        adc $$0, $7
        mov $11, %rax
        mulq $12
        add %rax, $3
        adc %rdx, $4
        adc $$0, $5
        adc $$0, $6
        adc $$0, $7
        mov $11, %rax
        mulq $13
        add %rax, $4
        adc %rdx, $5
        adc $$0, $6
        adc $$0, $7
        mov $11, %rax
        mulq $14
        add %rax, $5
        adc %rdx, $6
        adc $$0, $7
        mov $11, %rax
        mulq $15
        add %rax, $6
        adc %rdx, $7
        "
      : /* $0 */ "={r8}"(result[0]),  /* $1 */ "={r9}"(result[1]),  /* $2 */ "={r10}"(result[2]),
        /* $3 */ "={r11}"(result[3]), /* $4 */ "={r12}"(result[4]), /* $5 */ "={r13}"(result[5]),
        /* $6 */ "={r14}"(result[6]), /* $7 */ "={r15}"(result[7])

      : /* $8 */ "m"(self_t[0]),   /* $9 */ "m"(self_t[1]),   /* $10 */  "m"(self_t[2]),
        /* $11 */ "m"(self_t[3]),  /* $12 */ "m"(other_t[0]), /* $13 */ "m"(other_t[1]),
        /* $14 */ "m"(other_t[2]), /* $15 */ "m"(other_t[3])
      : "rax", "rdx"
      :
      );
    }

    U512(result)
  }
}
```

And here's what that generates. I've heavily commented it so you can understand what's going on even if you've never touched assembly in your life, but you will need to know basic low-level details like the difference between memory and registers. If you want to get a primer on the structure of a CPU, the [Wikipedia article on structure and implementation of CPUs][cpu-wiki] is a good place to start:

[cpu-wiki]: https://en.wikipedia.org/wiki/Central_processing_unit#Structure_and_implementation

```gas
bigint::U256::full_mul:
    ;; Function prelude - this is generated by Rust
    pushq  %r15
    pushq  %r14
    pushq  %r13
    pushq  %r12
    subq   $0x40, %rsp

    ;; Load the input arrays into registers...
    movq   0x68(%rsp), %rax
    movq   0x70(%rsp), %rcx
    movq   0x78(%rsp), %rdx
    movq   0x80(%rsp), %rsi
    movq   0x88(%rsp), %r8
    movq   0x90(%rsp), %r9
    movq   0x98(%rsp), %r10
    movq   0xa0(%rsp), %r11

    ;; ...and then immediately back into memory
    ;; This is done by the Rust compiler. There is a way to avoid
    ;; this happening but I'll get to that later
    ;; These four are the first input array
    movq   %rax, 0x38(%rsp)
    movq   %rcx, 0x30(%rsp)
    movq   %rdx, 0x28(%rsp)
    movq   %rsi, 0x20(%rsp)
    ;; These four are the output array, which is initialised to be
    ;; the same as the second input array.
    movq   %r8,  0x18(%rsp)
    movq   %r9,  0x10(%rsp)
    movq   %r10, 0x8(%rsp)
    movq   %r11, (%rsp)

    ;; This is the main loop, you'll see the same code repeated many
    ;; times since it's been unrolled so I won't go over it every time.
    ;; This takes the form of a loop that looks like:
    ;;
    ;; for i in 0..U256_SIZE {
    ;;     for j in 0..U256_SIZE {
    ;;         /* Loop body */
    ;;     }
    ;; }

    ;; Load the `0`th element of the input array into the "%rax"
    ;; register so we can operate on it. The first element is actually
    ;; already in `%rax` at this point but it gets loaded again anyway.
    ;; This is because the `asm!` macro is hiding a lot of details, which
    ;; I'll get to later.
    movq   0x38(%rsp), %rax
    ;; Multiply it with the `0`th element of the output array This operates
    ;; on memory rather than a register, and so is significantly slower than
    ;; if the same operation had been done on a register. Again, I'll get to
    ;; that soon.
    mulq   0x18(%rsp)
    ;; `mulq` multiplies two 64-bit numbers and stores the low and high
    ;; 64 bits of the result in `%rax` and `%rdx`, respectively. We move
    ;; the low bits into `%r8` (the lowest 64 bits of the 512-bit result)
    ;; and the high bits into `%r9` (the second-lowest 64 bits of the
    ;; result).
    movq   %rax, %r8
    movq   %rdx, %r9

    ;; We do the same for `i = 0, j = 1`
    movq   0x38(%rsp), %rax
    mulq   0x10(%rsp)

    ;; Whereas above we moved the values into the output registers, this time
    ;; we have to add the results to the output.
    addq   %rax, %r9

    ;; Here we add 0 because the CPU will use the "carry bit" (whether or not
    ;; the previous addition overflowed) as an additional input. This is
    ;; essentially the same as adding 1 to `rdx` if the previous addition
    ;; overflowed.
    adcq   $0x0, %rdx

    ;; Then we move the upper 64 bits of the multiplication (plus the carry bit
    ;; from the addition) into the third-lowest 64 bits of the output.
    movq   %rdx, %r10

    ;; Then we continue for `j = 2` and `j = 3`
    movq   0x38(%rsp), %rax
    mulq   0x8(%rsp)
    addq   %rax,       %r10
    adcq   $0x0,       %rdx
    movq   %rdx,       %r11
    movq   0x38(%rsp), %rax
    mulq   (%rsp)
    addq   %rax,       %r11
    adcq   $0x0,       %rdx
    movq   %rdx,       %r12

    ;; Then we do the same for `i = 1`, `i = 2` and `i = 3`
    movq   0x30(%rsp), %rax
    mulq   0x18(%rsp)  
    addq   %rax,       %r9
    adcq   %rdx,       %r10
    adcq   $0x0,       %r11
    adcq   $0x0,       %r12

    ;; This `xor` just ensures that `%r13` is zeroed. Again, this is
    ;; non-optimal (we don't need to zero these registers at all) but
    ;; I'll get to that.
    xorq   %r13,       %r13
    adcq   $0x0,       %r13
    xorq   %r14,       %r14
    adcq   $0x0,       %r14
    xorq   %r15,       %r15
    adcq   $0x0,       %r15
    movq   0x30(%rsp), %rax
    mulq   0x10(%rsp)  
    addq   %rax,       %r10
    adcq   %rdx,       %r11
    adcq   $0x0,       %r12
    adcq   $0x0,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x30(%rsp), %rax
    mulq   0x8(%rsp)   
    addq   %rax,       %r11
    adcq   %rdx,       %r12
    adcq   $0x0,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x30(%rsp), %rax
    mulq   (%rsp)      
    addq   %rax,       %r12
    adcq   %rdx,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x28(%rsp), %rax
    mulq   0x18(%rsp)  
    addq   %rax,       %r10
    adcq   %rdx,       %r11
    adcq   $0x0,       %r12
    adcq   $0x0,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x28(%rsp), %rax
    mulq   0x10(%rsp)  
    addq   %rax,       %r11
    adcq   %rdx,       %r12
    adcq   $0x0,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x28(%rsp), %rax
    mulq   0x8(%rsp)   
    addq   %rax,       %r12
    adcq   %rdx,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x28(%rsp), %rax
    mulq   (%rsp)      
    addq   %rax,       %r13
    adcq   %rdx,       %r14
    adcq   $0x0,       %r15
    movq   0x20(%rsp), %rax
    mulq   0x18(%rsp)  
    addq   %rax,       %r11
    adcq   %rdx,       %r12
    adcq   $0x0,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x20(%rsp), %rax
    mulq   0x10(%rsp)  
    addq   %rax,       %r12
    adcq   %rdx,       %r13
    adcq   $0x0,       %r14
    adcq   $0x0,       %r15
    movq   0x20(%rsp), %rax
    mulq   0x8(%rsp)   
    addq   %rax,       %r13
    adcq   %rdx,       %r14
    adcq   $0x0,       %r15
    movq   0x20(%rsp), %rax
    mulq   (%rsp)      
    addq   %rax,       %r14
    adcq   %rdx,       %r15

    ;; Finally, we move everything out of registers so we can
    ;; return it on the stack
    movq   %r8,   (%rdi)
    movq   %r9,   0x8(%rdi)
    movq   %r10,  0x10(%rdi)
    movq   %r11,  0x18(%rdi)
    movq   %r12,  0x20(%rdi)
    movq   %r13,  0x28(%rdi)
    movq   %r14,  0x30(%rdi)
    movq   %r15,  0x38(%rdi)
    movq   %rdi,  %rax
    addq   $0x40, %rsp
    popq   %r12
    popq   %r13
    popq   %r14
    popq   %r15
    retq   
```

So as you can see from my comments, there are a lot of inefficiencies in this code. We multiply on variables from memory instead of from registers, we do superfluous stores and loads, also the CPU has to do many stores and loads before even getting to the "real" code (the multiply-add loop), which is important because although the CPU can do loads and stores in parallel with calculations, the way that this code is written requires it to wait for everything to be loaded before it starts doing calculations. This is because the `asm` macro hides a lot of details. Essentially you're telling the compiler to put the input data wherever it likes, and then to substitute wherever it put the data into your assembly code with string manipulation. The compiler stores everything into registers, but then we instruct it to put the input arrays in memory (with the `"m"` before the input parameters) so it loads it back into memory again. There are ways that you could write this code to remove the inefficiencies in it, but it is clearly very difficult for even a seasoned professional to write the correct code here. This code is bug-prone - if you hadn't zeroed the output registers with the series of `xor` instructions then the code would fail sometimes but not always, with seemingly-random values that depended on the calling function's internal state. It could probably be sped up by replacing `"m"` with `"r"` here (I hadn't tested that because I only realised that this is a problem while investigating why the old assembly was so much slower in the course of writing this article), but that's not clear from reading the source code of the program and only someone with quite in-depth knowledge of LLVM's assembly syntax would realise that when looking at the code.

By comparison, the Rust code that uses `u128` is about as say-what-you-mean as you can get. Even if your goal was not optimisation you would probably write something similar to it as the simplest solution to the problem, but the code that LLVM produces is very high-quality. You can see already that it's not too different to our hand-written code, but it addresses some of the issues (commented below) while also including a couple more optimisations that I wouldn't have even thought of. I couldn't find any significant optimisations that it missed.

Here's the generated assembly:

```gas
bigint::U256::full_mul:
    ;; Function prelude
    pushq  %rbp
    movq   %rsp, %rbp
    pushq  %r15
    pushq  %r14
    pushq  %r13
    pushq  %r12
    pushq  %rbx
    subq   $0x48, %rsp

    movq   0x10(%rbp), %r11
    movq   0x18(%rbp), %rsi

    movq   %rsi, -0x38(%rbp)

    ;; I originally thought that this was a missed optimisation,
    ;; but it actually has to do this (instead of doing
    ;; `movq 0x30(%rbp), %rax`) because the `%rax` register gets
    ;; clobbered by the `mulq` below. This means it can multiply
    ;; the first element of the first array by each of the
    ;; elements of th without having to reload it from memory
    ;; like the hand-written assembly does.
    movq   0x30(%rbp), %rcx
    movq   %rcx,       %rax

    ;; LLVM multiplies from a register instead of from memory
    mulq   %r11

    ;; LLVM moves `%rdx` (the upper bits) into a register, since
    ;; we need to operate on it further. It moves `%rax` (the
    ;; lower bits) directly into memory because we don't need
    ;; to do any further work on it. This is better than moving
    ;; in and out of memory like we do in the previous code.
    movq   %rdx, %r9
    movq   %rax, -0x70(%rbp)
    movq   %rcx, %rax
    mulq   %rsi
    movq   %rax, %rbx
    movq   %rdx, %r8

    movq   0x20(%rbp), %rsi
    movq   %rcx,       %rax
    mulq   %rsi

    ;; LLVM uses `%r13` as an intermediate because it needs this
    ;; value in `%r13` later to operate on it anyway.
    movq   %rsi,       %r13
    movq   %r13,       -0x40(%rbp)

    ;; Again, we have to operate on both the low and high bits
    ;; so LLVM moves them both into registers.
    movq   %rax,       %r10
    movq   %rdx,       %r14
    movq   0x28(%rbp), %rdx
    movq   %rdx,       -0x48(%rbp)
    movq   %rcx,       %rax
    mulq   %rdx
    movq   %rax,       %r12
    movq   %rdx,       -0x58(%rbp)
    movq   0x38(%rbp), %r15
    movq   %r15,       %rax
    mulq   %r11
    addq   %r9,        %rbx
    adcq   %r8,        %r10

    ;; These two instructions store the flags into the `%rcx`
    ;; register.
    pushfq 
    popq   %rcx
    addq   %rax, %rbx
    movq   %rbx, -0x68(%rbp)
    adcq   %rdx, %r10

    ;; This stores the flags from the previous calculation into
    ;; `%r8`.
    pushfq 
    popq   %r8

    ;; LLVM takes the flags back out of `%rcx` and then does an
    ;; add including the carry flag. This is smart. It means we
    ;; don't need to do the weird-looking addition of zero since
    ;; we combine the addition of the carry flag and the addition
    ;; of the number's components together into one instruction.
    ;;
    ;; It's possible that the way LLVM does it is faster on modern
    ;; processors, but storing this in `%rcx` is unnecessary,
    ;; because the flags would be at the top of the stack anyway
    ;; (i.e. you could remove the `popq %rcx` above and this
    ;; `pushq %rcx` and it would act the same). If it is slower
    ;; then the difference will be negligible.
    pushq  %rcx
    popfq  
    adcq   %r14, %r12

    pushfq 
    popq   %rax
    movq   %rax,        -0x50(%rbp)
    movq   %r15,        %rax
    movq   -0x38(%rbp), %rsi
    mulq   %rsi
    movq   %rdx,        %rbx
    movq   %rax,        %r9
    addq   %r10,        %r9
    adcq   $0x0,        %rbx
    pushq  %r8
    popfq  
    adcq   $0x0,        %rbx

    ;; `setb` is used instead of explicitly zeroing registers and
    ;; then adding the carry bit. `setb` just sets the byte at the
    ;; given address to 1 if the carry flag is set (since this is
    ;; basically a `mov` it's faster than zeroing and then adding)
    setb   -0x29(%rbp)
    addq   %r12, %rbx

    setb   %r10b
    movq   %r15, %rax
    mulq   %r13
    movq   %rax, %r12
    movq   %rdx, %r8
    movq   0x40(%rbp), %r14
    movq   %r14, %rax
    mulq   %r11
    movq   %rdx, %r13
    movq   %rax, %rcx
    movq   %r14, %rax
    mulq   %rsi
    movq   %rdx, %rsi
    addq   %r9, %rcx
    movq   %rcx, -0x60(%rbp)

    ;; This is essentially a hack to add `%r12` and `%rbx` and store
    ;; the output in `%rcx`. It's one instruction instead of the two
    ;; that would be otherwise required. `leaq` is the take-address-of
    ;; instruction, so this line is essentially the same as if you did
    ;; `&((void*)first)[second]` instead of `first + second` in C. In
    ;; assembly, though, there are no hacks. Every dirty trick is fair
    ;; game.
    leaq   (%r12,%rbx), %rcx

    ;; The rest of the code doesn't have any new tricks, just the same
    ;; ones repeated.
    adcq   %rcx,        %r13
    pushfq 
    popq   %rcx
    addq   %rax,        %r13
    adcq   $0x0,        %rsi
    pushq  %rcx
    popfq  
    adcq   $0x0,        %rsi
    setb   -0x2a(%rbp)
    orb    -0x29(%rbp), %r10b
    addq   %r12,        %rbx
    movzbl %r10b,       %ebx
    adcq   %r8,         %rbx
    setb   %al
    movq   -0x50(%rbp), %rcx
    pushq  %rcx
    popfq  
    adcq   -0x58(%rbp), %rbx
    setb   %r8b
    orb    %al,         %r8b
    movq   %r15,        %rax
    mulq   -0x48(%rbp)
    movq   %rdx,        %r12
    movq   %rax,        %rcx
    addq   %rbx,        %rcx
    movzbl %r8b,        %eax
    adcq   %rax,        %r12
    addq   %rsi,        %rcx
    setb   %r10b
    movq   %r14,        %rax
    mulq   -0x40(%rbp)
    movq   %rax,        %r8
    movq   %rdx,        %rsi
    movq   0x48(%rbp),  %r15
    movq   %r15,        %rax
    mulq   %r11
    movq   %rdx,        %r9
    movq   %rax,        %r11
    movq   %r15,        %rax
    mulq   -0x38(%rbp)
    movq   %rdx,        %rbx
    addq   %r13,        %r11
    leaq   (%r8,%rcx),  %rdx
    adcq   %rdx,        %r9
    pushfq 
    popq   %rdx
    addq   %rax,        %r9
    adcq   $0x0,        %rbx
    pushq  %rdx
    popfq  
    adcq   $0x0,        %rbx
    setb   %r13b
    orb    -0x2a(%rbp), %r10b
    addq   %r8,         %rcx
    movzbl %r10b,       %ecx
    adcq   %rsi,        %rcx
    setb   %al
    addq   %r12,        %rcx
    setb   %r8b
    orb    %al,         %r8b
    movq   %r14,        %rax
    movq   -0x48(%rbp), %r14
    mulq   %r14
    movq   %rdx,        %r10
    movq   %rax,        %rsi
    addq   %rcx,        %rsi
    movzbl %r8b,        %eax
    adcq   %rax,        %r10
    addq   %rbx,        %rsi
    setb   %cl
    orb    %r13b,       %cl
    movq   %r15,        %rax
    mulq   -0x40(%rbp)
    movq   %rdx,        %rbx
    movq   %rax,        %r8
    addq   %rsi,        %r8
    movzbl %cl,         %eax
    adcq   %rax,        %rbx
    setb   %al
    addq   %r10,        %rbx
    setb   %cl
    orb    %al,         %cl
    movq   %r15,        %rax
    mulq   %r14
    addq   %rbx,        %rax
    movzbl %cl,         %ecx
    adcq   %rcx,        %rdx
    movq   -0x70(%rbp), %rcx
    movq   %rcx,        (%rdi)
    movq   -0x68(%rbp), %rcx
    movq   %rcx,        0x8(%rdi)
    movq   -0x60(%rbp), %rcx
    movq   %rcx,        0x10(%rdi)
    movq   %r11,        0x18(%rdi)
    movq   %r9,         0x20(%rdi)
    movq   %r8,         0x28(%rdi)
    movq   %rax,        0x30(%rdi)
    movq   %rdx,        0x38(%rdi)
    movq   %rdi,        %rax
    addq   $0x48,       %rsp
    popq   %rbx
    popq   %r12
    popq   %r13
    popq   %r14
    popq   %r15
    popq   %rbp
    retq   
```

Although there are a few more instructions in the LLVM-generated version, the slowest type of instruction (loads and stores) are minimised, it (for the most part) avoids redundant work and it applies many cheeky optimisations on top. The end result is that the code runs significantly faster.

This is not the first time that a carefully-written Rust implementation has outperformed our assembly code - some months ago I rewrote the Rust implementations of addition and subtraction, making them outperform the assembly implementation by 20% and 15%, respectively. Those didn't require 128-bit arithmetic to beat the assembly (to get the full power of the hardware in Rust you only need `u64::checked_add`/`checked_sub`), although who knows - maybe in a future PR we'll use 128-bit arithmetic and see the speed improve further still.

You can see the code from this PR [here][u128-pr] and the code from the addition/subtraction PR [here][add-sub-pr]. I should note that although the latter PR shows multiplication already outperforming the assembly implementation, this was actually due to a benchmark that mostly multiplied numbers with 0. Whoops. If there's something we can learn from that, it's that there can be no informed optimisation without representative benchmarks.

My point is not that we should take what we've learnt from the LLVM-generated code and write a new version of our hand-rolled assembly. The point is that optimising compilers are _really good_. There are very smart people working on them and computers are really good at this kind of optimisation problem (in the mathematic sense) in a way that humans find quite difficult. It's the job of language designers to give us the tools we need to inform the optimiser as best we can as to what our true intent is, and larger integer sizes are another step towards that. Rust has done a great job of allowing programmers to write programs that are easily understandable by humans and compilers alike, and it's just that power that has largely driven its success.

[u128-pr]: https://github.com/paritytech/bigint/pull/38
[add-sub-pr]: https://github.com/paritytech/bigint/pull/26

---
title: "Improving SmallVec's speed by 60% and why that shouldn't matter to you"
date: 2018-05-17T14:44:51+02:00
draft: false
---

`smallvec` is a library by the Servo team for reducing the number of allocations for dynamic arrays in the case that most of those arrays are below a certain size. For example, if you're creating a vector that normally has less than 100 items but could have more, you can reduce the load on the allocator by using a `SmallVec` like so:

```rust
fn print_iterator<T>(i: T)
where
  T: Iterator,
  T::Item: Debug,
{
  // Before...
  let vec: Vec<T::Item> = i.collect();
  // After
  let vec: SmallVec<[T::Item; 100]> = i.collect();
  
  println!("{:?}", vec);
}
```

The implementation is conceptually very simple, essentially looking like the following:

```rust
// Implemented for `[T; 1]`, `[T; 2]`, etc.
trait Array {
    type Item;

    fn ptr(&self) -> *const Item;
    fn ptr_mut(&self) -> *mut Item;
    fn size() -> usize;
}

enum SmallVec<T: Array> {
    Inline {
        len: usize,
        buffer: T,
    },
    Heap(Vec<T::Item>),
}
```

The real implementation is more complex for optimisation purposes, but it's essentially what you see above. If the length is less than `T::size()` it will store the elements inline (on the stack), if it becomes larger than that then it will copy the elements onto the heap - incurring a cost to allocate and to copy the existing elements. The main benefits to this are _not_ from avoiding allocation, but from better cache efficiency when you store `SmallVec` values in an array.

Because [`malloc` is fast][malloc-is-fast], for many cases it's actually slower to use `SmallVec` than just using `Vec` because the one-time cost of the initial allocation is dwarfed by the lifetime cost of `SmallVec`'s increased complexity. You can see that switching to `Vec` actually improves speed on many of `SmallVec`'s own benchmarks[^smallvec-benches]:

[malloc-is-fast]: http://voices.canonical.com/jussi.pakkanen/2011/09/27/is-malloc-slow/

[^smallvec-benches]: To make the benchmarks fair I preallocated the vector with the same number of elements that the `SmallVec` preallocated on the stack (16). I believe this is fair because in a real program you can always easily replace calls to `Vec::new` with `Vec::with_capacity(N)` where `N` was the number that you would have put into `SmallVec::<[T; N]>::new`. To see the exact methodology you can see [this commit][vec-benches].

[vec-benches]: https://github.com/servo/rust-smallvec/commit/291eb90ed1eaabf0e02987de9c3c76d1b85b6a53

```
 name                     smallvec.bench ns/iter  vec.bench ns/iter  diff ns/iter   diff %  speedup
 extend_from_slice        52                      65                           13   25.00%   x 0.80
 extend_from_slice_small  23                      29                            6   26.09%   x 0.79
 extend                   200                     69                         -131  -65.50%   x 2.90
 extend_small             39                      28                          -11  -28.21%   x 1.39
 from_slice               199                     36                         -163  -81.91%   x 5.53
 from_slice_small         38                      30                           -8  -21.05%   x 1.27
 insert                   1,222                   1,231                         9    0.74%   x 0.99
 insert_small             121                     114                          -7   -5.79%   x 1.06
 macro_from_elem          218                     40                         -178  -81.65%   x 5.45
 macro_from_elem_small    60                      32                          -28  -46.67%   x 1.88
 push                     357                     369                          12    3.36%   x 0.97
 push_small               57                      53                           -4   -7.02%   x 1.08
 macro_from_list          47                      33                          -14  -29.79%   x 1.42
 pushpop                  306                     253                         -53  -17.32%   x 1.21
```

`Vec` is only slower for one method - `extend_from_slice`. Weirdly `SmallVec` is even faster in the case that `extend_from_slice` must allocate. From a cursory glance at the source code for `std` it looks like `SmallVec` might get an edge on inlining, since `Vec::extend_from_slice` isn't marked `#[inline]`.

Even with the improvement on `extend_from_slice`, it's still a tough sell to use `SmallVec`. Even in the case where `SmallVec` is on the stack - the case where it's supposed to have an edge - it's still slower.

Wait, though, isn't the only difference between the two the fact that `SmallVec` has to check whether it's inline or heap-allocated? Why is `from_elem` (which tests the `vec![val; N]` macro and the equivalent `smallvec![val; N]` macro) more than 5 times slower for `SmallVec`, and why is `extend` almost 3 times as slow, even being slower when it's on the stack? Well it turns out that apparently this library had some serious inefficiencies that are _not_ inherent to the design. Let's check out what `SmallVec::from_elem` and `SmallVec::extend` look like:

```rust
impl<A: Array> SmallVec<A> {
  pub fn from_elem(elem: A::Item, n: usize) -> Self {
    let mut v = SmallVec::with_capacity(n);
    v.insert_many(0, (0..n).map(|_| elem.clone()));
    v
  }

  // ...
}

impl<A: Array> Extend<A::Item> for SmallVec<A> {
  fn extend<I>(&mut self, iterable: I)
  where
    I: IntoIterator<Item=A::Item>
  {
    let iter = iterable.into_iter();
    let (lower_size_bound, _) = iter.size_hint();

    let target_len = self.len + lower_size_bound;

    if target_len > self.capacity() {
       self.grow(target_len);
    }

    for elem in iter {
      self.push(elem);
    }
  }
}
```

Both seem pretty clear, for `extend` you reserve the expected number of elements and then do a `for` loop over the iterator, pushing each element to the new `SmallVec`. For `from_elem` you create a `SmallVec` preallocated with the number of elements you need and then insert `n` clones of `elem`. Hang on, though, what does the source code of `insert_many` look like?

```rust
impl<A: Array> SmallVec<A> {
  // ...

  pub fn insert_many<I: IntoIterator<Item=A::Item>>(&mut self, index: usize, iterable: I) {
    let iter = iterable.into_iter();
    let (lower_size_bound, _) = iter.size_hint();
    assert!(lower_size_bound <= std::isize::MAX as usize);  // Ensure offset is indexable
    assert!(index + lower_size_bound >= index);  // Protect against overflow
    self.reserve(lower_size_bound);

    unsafe {
      let old_len = self.len;
      assert!(index <= old_len);
      let ptr = self.as_mut_ptr().offset(index as isize);
      ptr::copy(ptr, ptr.offset(lower_size_bound as isize), old_len - index);

      for (off, element) in iter.enumerate() {
        if off < lower_size_bound {
          ptr::write(ptr.offset(off as isize), element);
          self.len = self.len + 1;
        } else {
          // Iterator provided more elements than the hint.
          assert!(index + off >= index);  // Protect against overflow.
          self.insert(index + off, element);
        }
      }

      let num_added = self.len - old_len;
      if num_added < lower_size_bound {
        // Iterator provided fewer elements than the hint
        ptr::copy(ptr.offset(lower_size_bound as isize), ptr.offset(num_added as isize), old_len - index);
      }
    }
  }
}
```

Gah! That's a _lot_ of code. Let's go through this bit-by-bit (I've removed assertions to make the logic clearer):

```rust
let (lower_size_bound, _) = iter.size_hint();

self.reserve(lower_size_bound);
```

This reserves enough space for the iterator's self-reported lower bound on its number of produced elements. Well hang on, we already know that we have enough space since we just created it with `with_capacity` and then used an iterator with a fixed size. So this is useless. We don't need to reserve any more space.

```rust
let old_len = self.len;

let ptr = self.as_mut_ptr().offset(index as isize);

ptr::copy(ptr, ptr.offset(lower_size_bound as isize), old_len - index);
```

This optimistically copies any existing elements to the end of the array. This is because this function allows you to insert the new elements at any point - not just at the end - and so if there are existing elements it needs to move them first[^insert-is-unsound]. Again, these lines are useless. If we're calling this from `from_elem`, we _know_ that both `self.len` and `index` are both 0, so this will always copy 0 bytes (a no-op). We still waste cycles figuring this out, though.

[^insert-is-unsound]: During the writing of this article I realised that this function is actually unsound. One interesting thing about writing unsafe Rust is that safe code is never to be trusted. You can use values from safe code as hints but it should be impossible to cause unsoundness by returning incorrect values from safe code. This means, for example, that `Iterator::size_hint` can return essentially whatever it wants. Also, it should be possible for safe code to panic at any point without causing unsoundness (although it's allowed to cause arbitrarily bad behaviour otherwise, like infinite loops or leaking data). You can see a visual explanation of this bug in [this gist][this-gist]. At the time of writing it's still unfixed but [I opened an issue][unsound-issue] where you can track its progress.

[this-gist]: https://gist.github.com/Vurich/ffa55d8e69377a64c9708e07c2db2a1b
[unsound-issue]: https://github.com/servo/rust-smallvec/issues/96

```rust
for (off, element) in iter.enumerate() {
  if off < lower_size_bound {
    ptr::write(ptr.offset(off as isize), element);
    self.len = self.len + 1;
  } else {
    self.insert(index + off, element);
  }
}
```

For every element we do a branch. This means that the compiler can't optimise this into a [`memcpy`][memcpy] (the fastest possible method of copying the bytes) because for each element we need to check whether or not we've reached the lower size bound. Technically the compiler _could_ optimise this by turning it into two loops (known as [loop fission][loop-fission]) and then noticing that the first loop is equivalent to a `memcpy`, but relying on it to do so is inadvisable because it requires multiple optimisation passes to be applied in the correct order. Again, when we're calling this function from `from_elem` we know that this can only ever go through the first branch and never the second, so this prevents optimisations and wastes time checking `off < lower_size_bound` for no benefit.

[loop-fission]: https://en.wikipedia.org/wiki/Loop_fission_and_fusion
[memcpy]: http://man7.org/linux/man-pages/man3/memcpy.3.html

```rust
let num_added = self.len - old_len;
if num_added < lower_size_bound {
  // Iterator provided fewer elements than the hint
  ptr::copy(
    ptr.offset(lower_size_bound as isize),
    ptr.offset(num_added as isize),
    old_len - index
  );
}
```

This is another branch that's always skipped when we're doing `from_elem`, since we know that the iterator produces the right amount of elements every time. More wasted cycles.

I'm not going to go line-by-line into precisely how `Vec` does it, but you can check out the code [here][vec-from-elem]. The important thing is that for many types it's about as fast as is reasonably possible. A _lot_ of work has been put into optimising this and it shows. There is even a special method that uses the operating system's ability to return zeroed memory "magically" if the element you pass to the function is equal to 0 (i.e. `Vec` requests a pointer to zeroed memory and the OS returns a pointer to memory that it knows _should_ be zeroed, but doesn't actually zero it until you try to access it). It only increases the vector's length once, after all elements have been added. This is much better than our method of increasing the vector's length by 1 each iteration of the loop because you only have to do one addition. The bigger the `Vec`, the bigger difference this will make. The biggest benefit is that it uses specialisation to use `memcpy` for any element that implements `Copy`. This means that when possible it will copy using SIMD.

[vec-from-elem]: https://github.com/rust-lang/rust/blob/9e3432447a9c6386443acdf731d488c159be3f66/src/liballoc/vec.rs#L1561-L1662 

The optimisations that require specialisation are actually impossible to implement in `SmallVec` without requiring a nightly compiler, but it turns out that it doesn't matter. You can just reuse `Vec::from_elem` (which is what `vec![elem; n]` desugars to) when you know it's going to end up on the heap, and do a simple loop-and-write when you know it will be on the stack. Here's what that looks like:

```rust
pub fn from_elem(elem: A::Item, n: usize) -> Self {
  if n > A::size() {
    vec![elem; n].into()
  } else {
    unsafe {
      let mut arr: A = ::std::mem::uninitialized();
      let ptr = arr.ptr_mut();

      for i in 0..n as isize {
        ::std::ptr::write(ptr.offset(i), elem.clone());
      }

      SmallVec {
        data: Inline { array: arr },
        len: n,
      }
    }
  }
}
```

We don't need to use the OS's magical ability to conjure up zeroed memory when we're building things on the stack since that works at the granularity of a page - 4kb by default on Linux - and if you're creating an array that large then the cost of allocation is absolutely dwarfed by the cost of initialisation. LLVM can [trivially optimise][llvm-loop-opt] the `for` loop into a `memcpy` and even a `memset` when `A::Item = u8` (the comment at the header of the file lists this optimisation as a `TODO` but if you look at the code you can see that it's already been implemented). We get all the benefits of `Vec`'s complex implementation with the only cost being a `n > A::size()` check, which is especially cheap because `A::size()` is a constant. You can see the generated assembly [here][generated-assembly], I haven't commented it because of the sheer amount of code that was generated before, but just the difference in the amount of code should make it clear that this method is significantly more efficient.

[llvm-loop-opt]: https://github.com/llvm-mirror/llvm/blob/master/lib/Transforms/Scalar/LoopIdiomRecognize.cpp
[generated-assembly]: https://gist.github.com/Vurich/2d180d83821fc4f129b176a9a0f1dc63

After this, we see a huge improvement:

```
 name             smallvec.bench ns/iter  vec.bench ns/iter  diff ns/iter   diff %  speedup 
 macro_from_elem  66                      50                 -16            -24.24%   x 1.32 
```

`Vec` is still faster, but now the difference is only 16ns instead of 130ns. I do wonder if there's some way to speed up the conversion from `Vec` to `SmallVec`, because that's the only place that the difference could be coming from and it should be essentially instant.

Now for the fun one. Let's see `SmallVec::extend` again:

```rust
impl<A: Array> Extend<A::Item> for SmallVec<A> {
  fn extend<I>(&mut self, iterable: I)
  where
    I: IntoIterator<Item=A::Item>
  {
    let iter = iterable.into_iter();
    let (lower_size_bound, _) = iter.size_hint();

    let target_len = self.len + lower_size_bound;

    if target_len > self.capacity() {
       self.grow(target_len);
    }

    for elem in iter {
      self.push(elem);
    }
  }
}
```

So what's wrong with this? Well, let's take a look at the source of `push`:

```rust
pub fn push(&mut self, value: A::Item) {
  let cap = self.capacity();

  if self.len == cap {
    self.grow(cmp::max(cap * 2, 1))
  }

  unsafe {
    let end = self.as_mut_ptr()
      .offset(self.len as isize);

    ptr::write(end, value);
    let len = self.len;
    self.set_len(len + 1)
  }
}
```

It's the same problem again - we know that we don't need to grow for the first `lower_size_bound` iterations (because we already preallocated that many elements) but we do anyway. Plus we're making LLVM's job hard - it _could_ figure out that because we set `self.capacity` and then check it against `self.len` in each iteration of the loop then it can split the loop in two and omit the check for the first `self.capacity - self.len` iterations, and then from _that_ figure out that the writes can be vectorised, but it's again relying on a specific set of optimisations in a specific order. Also, we store to `self.len` every iteration of the loop and although LLVM could load `self.len` to a register, increment it there, and then store it again, that is not a simple enough optimisation for us to expect it to be reliably performed. Besides, that optimisation cannot be performed for an iterator that may panic, since in that case we need to ensure that `self.len` is correct after each iteration of the loop. This is because other code could observe it after the panic (even without `catch_panic`, code in implementations of `Drop::drop` could observe it). We can do this optimisation ourself and make the code easier to understand by LLVM:

```rust
fn extend<I>(&mut self, iterable: I)
where
  I: IntoIterator<Item=A::Item>
{
  let mut iter = iterable.into_iter();
  let (lower_size_bound, _) = iter.size_hint();

  let target_len = self.len + lower_size_bound;

  if target_len > self.capacity() {
    self.grow(target_len);
  }

  // This section is new
  unsafe {
    let ptr = self.as_mut_ptr()
      .offset(self.len() as isize);

    let len = self.len();
    let mut count = 0;
    for p in 0..lower_size_bound as isize {
      if let Some(out) = iter.next() {
        ::std::ptr::write(ptr.offset(p), out);
        count += 1;
      } else {
        break;
      }
    }

    self.set_len(len + count);
  }

  for elem in iter {
    self.push(elem);
  }
}
```

You see that in the fast path (the path where `iter.size_hint()` tells the truth) we just continuously `ptr::write`. We branch in the loop, but for most iterators the branch could either easily be removed by the compiler after monomorphisation  - for example, for slices, vectors and wrappers around either of those that preserve `size_hint`. For those where it isn't, the `else` branch is taken at most once per run of the loop. This makes it extremely predictable even for CPUs with only basic branch predictors. We then do a "real" store to the `len` field only once, and `count` can easily be put into a register. This optimisation isn't defeated by an iterator that may panic, either. Since we only store to memory after the loop, if it panics during the loop then the calling code will just see the value of `self.len` from before we entered the loop. We can do this transformation because we know that leaking data when panicking is fine, but LLVM cannot because it would change the meaning of the program[^set-len-on-drop].

[^set-len-on-drop]: It's actually possible for LLVM to do the increment in a register and just add the code to actually store the length into the panic cleanup code. `Vec` actually does this explicitly with a `SetLenOnDrop` struct. Again though, it's a complex optimisation and you can't rely on LLVM applying it.

We can see that the difference after this optimisation is large:

```
 name          smallvec.bench ns/iter  vec.bench ns/iter  diff ns/iter   diff %  speedup 
 extend        102                     70                 -32            -31.37%   x 1.46 
 extend_small  25                      28                   3             12.00%   x 0.89 

```

We're much closer to `Vec`'s speed than before, and at last `SmallVec` is faster for the case where it doesn't have to allocate.

I think that the important takeaways from this are: you don't need to perform every optimisation yourself, but you should make your code as simple and optimisable as possible. It is necessary, however, manually perform any optimisations that changes the meaning of your program. Even if an optimisation seems like it'd have a small effect, if you can't expect the compiler to perform it then I think it's worth trying. It's very possible that it can unlock more compiler optimisations. Above all else, though, it's important to actually benchmark your optimisations. You wouldn't expect your code to be correct without testing it (even if that testing is by running the program and clicking around) so why would you expect your code to be fast if you don't benchmark it. `SmallVec` actually had no comparisons to `Vec` in its test suite until I wrote some for the purposes of this article. There are a _lot_ of crates out there that currently use `smallvec` - [crates.io lists 9 pages of crates using it at time of writing][reverse-deps]. If you're using `smallvec` for its performance benefits, it's worth benchmarking your program using `Vec` with `Vec::with_capacity` instead. Usually the simpler implementation is the faster one.

[reverse-deps]: https://crates.io/crates/smallvec/reverse_dependencies

---
title: "Why PhantomData"
date: 2018-06-05T13:41:30+02:00
draft: false
---

If you saw [the recent blog post on tagged keys][tagged-keys] you might have wondered: why can't I generalise this pattern? Why not have a type that looks like this:

[tagged-keys]: https://matklad.github.io/2018/05/24/typed-key-pattern.html

```rust
struct Tagged<T>(usize);
```

Well if you try that now, you'll end up with the following error:

```
error[E0392]: parameter `T` is never used
 --> src/main.rs:1:15
  |
1 | struct Tagged<T>(usize);
  |               ^ unused type parameter
  |
  = help: consider removing `T` or using a marker such as `std::marker::PhantomData`
```

That's right, we're not allowed to have a type parameter that goes unused. If we want to have a type that looks like the one above we have to add a marker to it like so:

```rust
struct Tagged<T>(usize, PhantomData<T>);
```

This is obviously less ergonomic (we can no longer create a `Tagged` with just `Tagged(my_index)`) and raises the question as to why it's necessary. Why not just have this be a warning, like with unused variables or unused arguments?

The problem, as with many problems, lies with inheritance. Well, not quite, but it lies with subtyping - the most well-known kind of which is Java-style inheritance. In Java, you can create a class that inherits from some other class and then the new class can be used anywhere the inherited class could have been. For example:

```java
public class Dog {
  public void makeNoise() {
    System.out.println("Rurf");
  }
}

public class Spaniel extends Dog {
  public void makeNoise() {
    System.out.println("Bork");
  }
}

public class Main {
  // Even though we declare that this takes a `Dog`,
  // we can also take a `Spaniel` because it inherits
  // from `Dog`.
  public static void squeeze(Dog somedog) {
    somedog.makeNoise();
  }
  
  public static void main(String args[]) {
    Main.squeeze(new Dog()); // Prints "Rurf"
    Main.squeeze(new Spaniel()); // Prints "Bork"
  }
}
```

I'm sorry to have subjected you to Java. You can stop weeping and screaming now, you're not in the first year of your computer science course anymore. Unless you are, in which case I'm so sorry.

Anyway, the upshot is that if A is a subtype of B, then any place expecting a B can also be given an A. However, a place expecting an A can _not_ be given a B - what if you had added a `eatCigaretteOffFloor` method to `Spaniel`? The `Dog` class wouldn't have this method. Even though Rust doesn't have the concept of Java-style inheritance[^inheritance] it does still have subtyping for lifetimes.

[^inheritance]: Just to be clear, this is a good thing. Java-style implementation inheritance is just dumping functions into your class's namespace and has no place in a civilised society. We should have abolished implementation inheritance some time around the era where we stopped shitting where we ate.

```rust
fn order_from_chinese_restaurant<'a>(
  first: &'a usize,
  second: &'a usize,
) {
  println!(
    "So that's a {} and a {}, that'll be with you shortly",
    first,
    second,
  );
}

let a = 1;
let b = 2;

static C: &'static usize = &42;

// This works because `a` and `b` have the same lifetime
order_from_chinese_restaurant(&a, &b);
// Even though `a` has a shorter lifetime than `C` we can still
// pass it here, because `'static` is a subtype of `'a`
order_from_chinese_restaurant(&a, C);

fn find_out_the_truth(truth: &'static usize) {
  println!(
    "Ah, it was {} all along",
    truth,
  )
}

find_out_the_truth(C); // This is OK
find_out_the_truth(a); // ERROR
```

You can see why this is the case - it's OK to pass a longer reference where a shorter one is expected, but the other way around is not true. For example, if a function expects a reference with the `'static` lifetime, it can store that reference in a global mutex. This would be invalid for a reference with a shorter lifetime and so Rust won't let us do it. So what does this mean for type variables? Let's say we have a struct:

```rust
struct PointlessWrapper<T>(T);
```

Is `PointlessWrapper<Foo>` a subtype of `PointlessWrapper<Bar>` well, it is if `Foo` is a subtype of `Bar`. You can pass a `PointlessWrapper<&'static str>` where you expected a `PointlessWrapper<&'a str>`. What about this, though:

```rust
struct FunkyWrapper<T>(fn(T) -> ());
```

Is `FunkyWrapper<Foo>` a subtype of `FunkyWrapper<Bar>`? It's a subtype if `Foo` is a subtype of `Bar`, right? Well no. Actually it goes the _other direction_. `FunkyWrapper<Foo>` is a subtype of `FunkyWrapper<Bar>` _if `Bar` is a subtype of `Foo`_. This is because if you have a `FunkyWrapper<&'static str>`, that means you are wrapping a function that takes a `&'static str`. This function could, again, store the reference in a global mutex or do all kinds of other freaky stuff. You can't pass a `FunkyWrapper<&'static str>` where a `FunkyWrapper<&'a str>` is expected, because if you get given a `FunkyWrapper<&'static str>` and pass an `&'a str` to it then the wrapped function could put that reference with a short lifetime into a global mutex - but the reference would be invalidated after the lifetime `'a` ends. You might be starting to see why we need a marker type now, but I'll give you one more example:

```rust
struct MagicWrapper<T>(T, fn(T) -> ());
```

Is `MagicWrapper<Foo>` a subtype of `MagicWrapper<Bar>`? No, it's not. You can only pass `MagicWrapper<Foo>` where `MagicWrapper<Foo>` is expected. This is because a function taking `MagicWrapper<&'static str>` could both rely on the fact that the first field has the static lifetime and the fact that the second field takes a static lifetime. We can't pass a `MagicWrapper<&'static str>` where a `MagicWrapper<&'a str>` is expected, nor can we pass a `MagicWrapper<&'a str>` where a `MagicWrapper<&'static str>` is expected. Now let's go back to our first example:

```rust
struct Tagged<T>(usize);
```

Is `Tagged<Foo>` a subtype of `Tagged<Bar>`? We can't know. `PhantomData` is a way to have a field that acts like the type parameter for subtyping, but has no runtime cost. Rust told us to make our struct look like so:

```rust
struct Tagged<T>(usize, PhantomData<T>);
```

This means that `Tagged<Foo>` is a subtype of `Tagged<Bar>` if `Foo` is a subtype of `Bar`. If we needed it to go the other direction we'd need to do:

```rust
struct Tagged<T>(usize, PhantomData<fn(T) -> ()>);
```

This would mean that `Tagged<Foo>` is a subtype of `Tagged<Bar>` if `Bar` is a subtype of `Foo`.

For most code this doesn't matter - if you're using `PhantomData` it's probably for concrete "marker" types that don't include lifetimes - but for unsafe code this is important. You could rely on your type parameter's lifetimes in order for your code to be correct and to avoid use-after-free bugs. Even though you probably won't need to know this for any real-life code, it's just interesting to explore the reasoning behind some of Rust's seemingly-odd design choices. Most of the time there's good reasoning behind them.

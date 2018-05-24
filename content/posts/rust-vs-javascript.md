---
title: "JavaScript: a view from the bottom"
date: 2018-05-17T11:38:07+02:00
draft: true
---

There are a lot of benchmarks out there that show code written in some dynamically-typed languages being as fast as systems languages, with some people taking that as proof that "fast languages" vs "slow languages" is a false dichotomy and that it's fine to write end-user software like desktop applications or [a terminal emulator][js-term] in JavaScript since V8 is smart enough to make it run fast. I'm going to show why I believe that this is misleading. For the purposes of this article I will continuously refer to Rust as an example of a system language, but obviously the same principles apply to any language with no or minimal runtime, like C, C++, FORTRAN or D (if you don't mind [avoiding most of D's benefits over C++][avoiding-druntime]).

[js-term]: https://hyper.is/
[avoiding-druntime]: https://www.auburnsounds.com/blog/2016-11-10_Running-D-without-its-runtime.html

I should make it clear that I don't dislike JavaScript. I have written it professionally and unprofessionally and although it's not [my favourite dynamic language][lua], in my opinion it's more enjoyable to use than most "PL theory is for nerds" languages like Java and Go. This is specifically about performance and related concerns like memory usage and battery usage. Let's start with a simple program:

[lua]: https://www.lua.org/

```javascript
function factorial( n ) {
  return equal( n, 1 ) ? n : multiply( n, factorial( n - 1 ) );
}

function multiply( x, y ) {
  return x * y;
}

function equal( a, b ) {
  return a === b;
}

var i = 0;

while ( i++ < 1e7 ) {
  factorial((i % 100) + 1);
}
```

We can run this with `d8`, 

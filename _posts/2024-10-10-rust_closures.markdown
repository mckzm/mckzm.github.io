---
layout: post
title:  "Rust Closures and Recursion"
date:   2024-10-14 00:00:00 +0900
categories: rust
tags: rust closures recursion self_reference
---

I keep forgetting that Rust closures do not (straightforwardly[^1]) support recursion. Sometimes I
remember in the midst of writing code, in other instances rust-analyzer or
Clippy duly serve me a reminder to that effect - with a side-order of déjà vu.
Maybe posting about this will help it (finally) sink in.

As an example, the code below ([playground
link](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=9ec9bbb60b88669d64af29682d4724c8)),
which aims to have an implementation of [the Ackermann function, in its
Péter/Robinson
incarnation](https://en.wikipedia.org/wiki/Ackermann_function#Definition:_as_m-ary_function),
tally the number of its invocations by using a closure, does not compile:

 ```Rust
fn main() {
    let mut invocations = 0;
    let ackermann = |m, n| {
        invocations += 1;
        match (m, n) {
            (0, _) => n + 1,
            (_, 0) => ackermann(m - 1, 1),
            _ => ackermann(m - 1, ackermann(m, n - 1)),
        }
    };
}
```

Specifically, rustc tells us we used [an unresolved name, resulting in
E0425](https://doc.rust-lang.org/stable/error_codes/E0425.html):
``` 
Checking playground v0.0.1 (/playground)
error[E0425]: cannot find function `ackermann` in this scope
 --> src/main.rs:7:23
  |
7 |             (_, 0) => ackermann(m - 1, 1),
  |                       ^^^^^^^^^ not found in this scope

error[E0425]: cannot find function `ackermann` in this scope
 --> src/main.rs:8:35
  |
8 |             _ => ackermann(m - 1, ackermann(m, n - 1)),
  |                                   ^^^^^^^^^ not found in this scope

error[E0425]: cannot find function `ackermann` in this scope
 --> src/main.rs:8:18
  |
8 |             _ => ackermann(m - 1, ackermann(m, n - 1)),
  |                  ^^^^^^^^^ not found in this scope

For more information about this error, try `rustc --explain E0425`.
error: could not compile `playground` (bin "playground") due to 3 previous errors
```

This contretemps is never a showstopper in my experience[^2]. In this specific
instance, there are a few ways to achieve our goal using a function item
instead of a closure - for example by pairing `n` with an extra parameter meant
to record invocations, and returning that same pair:

```Rust
fn ackermann(m: u32, (n, invocations): (u32, u32)) -> (u32, u32) {
    match (m, n) {
        (0, _) => (n + 1, invocations + 1),
        (_, 0) => ackermann(m - 1, (1, invocations + 1)),
        _ => ackermann(m - 1, ackermann(m, (n - 1, invocations + 1))),
    }
}

fn main() {
    println!("{:?}", ackermann(3, (4, 0)));
}
```
([playground
link](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=38a8048a70de3c398cb3e3aa00398ca5))

We could wrap calls to the function above within another that exposes the usual
Ackermann interface, too. And arguably not fiddling with enclosing scopes is A
Good Thing. Still, I find Rust's closure syntax, being much lighter than its
counterpart for function items, a pleasure to use, in particular for scratch
code. A construct similar to `let rec` or `let/where` in OCaml/Haskell (resp.)
would be lovely.

# Escape Hatch for [`Fn` Closures](https://doc.rust-lang.org/std/ops/trait.Fn.html)
[`Fn` closures don't mutate their environment when they are
called](https://doc.rust-lang.org/book/ch13-01-closures.html#:~:text=Fn%20applies%20to);
indeed some may not capture anything from their enclosing scope. They can be
recursively called if wrapped within a struct, which I learned [thanks to
asherlau and
steffhan in an URLO
thread](https://users.rust-lang.org/t/how-to-do-recursive-closure-and-mutate-the-captured-env/114975).
Reprising the Ackermann/Péter/Robinson example (without the invocation counter,
since we're using a `Fn`), this gives something like:
```Rust
struct FnWrapper<'a> {
    f: &'a dyn Fn(u32, u32, &FnWrapper) -> u32,
}

fn main() {
    let ackermann = FnWrapper {
        f: &|m: u32, n: u32, wrap: &FnWrapper| match (m, n) {
            (0, _) => n + 1,
            (_, 0) => (wrap.f)(m - 1, 1, wrap),
            _ => (wrap.f)(m - 1, (wrap.f)(m, n - 1, wrap), wrap),
        },
    };
    println!("{}", (ackermann.f)(3, 4, &ackermann));
}
```
([playground code](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=86d4a1bd3ecb2a8b458f1aebd9e4579e))

Unfortunately such code is neither very straightforward to write nor to read:
the comparative advantage of the more lightweight closure syntax has well and
truly disappeared. Still, I'd be lying if I didn't admit I find it a little
fun!

[^1]: More about that qualification below.
[^2]: Please let me know if yours differs!

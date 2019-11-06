---
layout:     post
title:      "nightly-09-11 引入的 async 块 `Send` 约束问题"
subtitle:   "原因及解决方案？"
date:       2019-11-07 00:00:00
author:     "Hexi"
header-img: "img/bg/2019-11-07-async-block-send.jpg"
tags:
    - rust
    - async
---


This blog aims to summarize and explain [rust-lang/rust#64477](https://github.com/rust-lang/rust/issues/64477) and [ rust-lang/rust#64856](https://github.com/rust-lang/rust/pull/64856).

### Introduction

If you are using rust nightly toolchain between 2019-09-11 and 2019-11-05(latest), you will fail to compile this code:

```rust
// examples/format.rs
async fn foo(_: String) {}

fn bar() -> impl Send {
    async move {
        foo(format!("")).await;
    }
}

fn main() {}
```

Run it:

```bash
cargo +nightly-2019-09-11 run --example format
```

Compiler complains:

```bash
error[E0277]: `*mut (dyn std::ops::Fn() + 'static)` cannot be shared between threads safely
 --> examples/format.rs:3:13
  |
3 | fn bar() -> impl Send {
  |             ^^^^^^^^^ `*mut (dyn std::ops::Fn() + 'static)` cannot be shared between threads safely
  |
  = help: within `core::fmt::Void`, the trait `std::marker::Sync` is not implemented for `*mut (dyn std::ops::Fn() + 'static)`
  = note: required because it appears within the type `std::marker::PhantomData<*mut (dyn std::ops::Fn() + 'static)>`
  = note: required because it appears within the type `core::fmt::Void`
  = note: required because of the requirements on the impl of `std::marker::Send` for `&core::fmt::Void`
  = note: required because it appears within the type `std::fmt::ArgumentV1<'_>`
  = note: required because it appears within the type `[std::fmt::ArgumentV1<'_>; 0]`
  = note: required because it appears within the type `for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6, 't7, 't8, 't9, 't10, 't11, 't12, 't13> {fn(std::string::String) -> impl std::future::Future {foo}, for<'t14> fn(std::fmt::Arguments<'t14>) -> std::string::String {std::fmt::format}, fn(&'r [&'r str], &'r [std::fmt::ArgumentV1<'r>]) -> std::fmt::Arguments<'r> {std::fmt::Arguments::<'r>::new_v1}, [&'s str; 0], &'t0 [&'t1 str; 0], &'t2 [&'t3 str; 0], &'t4 [&'t5 str], (), [std::fmt::ArgumentV1<'t6>; 0], &'t7 [std::fmt::ArgumentV1<'t8>; 0], &'t9 [std::fmt::ArgumentV1<'t10>; 0], &'t11 [std::fmt::ArgumentV1<'t12>], std::fmt::Arguments<'t13>, std::string::String, impl std::future::Future}`
  = note: required because it appears within the type `[static generator@examples/format.rs:4:16: 6:6 for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6, 't7, 't8, 't9, 't10, 't11, 't12, 't13> {fn(std::string::String) -> impl std::future::Future {foo}, for<'t14> fn(std::fmt::Arguments<'t14>) -> std::string::String {std::fmt::format}, fn(&'r [&'r str], &'r [std::fmt::ArgumentV1<'r>]) -> std::fmt::Arguments<'r> {std::fmt::Arguments::<'r>::new_v1}, [&'s str; 0], &'t0 [&'t1 str; 0], &'t2 [&'t3 str; 0], &'t4 [&'t5 str], (), [std::fmt::ArgumentV1<'t6>; 0], &'t7 [std::fmt::ArgumentV1<'t8>; 0], &'t9 [std::fmt::ArgumentV1<'t10>; 0], &'t11 [std::fmt::ArgumentV1<'t12>], std::fmt::Arguments<'t13>, std::string::String, impl std::future::Future}]`
  = note: required because it appears within the type `std::future::GenFuture<[static generator@examples/format.rs:4:16: 6:6 for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6, 't7, 't8, 't9, 't10, 't11, 't12, 't13> {fn(std::string::String) -> impl std::future::Future {foo}, for<'t14> fn(std::fmt::Arguments<'t14>) -> std::string::String {std::fmt::format}, fn(&'r [&'r str], &'r [std::fmt::ArgumentV1<'r>]) -> std::fmt::Arguments<'r> {std::fmt::Arguments::<'r>::new_v1}, [&'s str; 0], &'t0 [&'t1 str; 0], &'t2 [&'t3 str; 0], &'t4 [&'t5 str], (), [std::fmt::ArgumentV1<'t6>; 0], &'t7 [std::fmt::ArgumentV1<'t8>; 0], &'t9 [std::fmt::ArgumentV1<'t10>; 0], &'t11 [std::fmt::ArgumentV1<'t12>], std::fmt::Arguments<'t13>, std::string::String, impl std::future::Future}]>`
  = note: required because it appears within the type `impl std::future::Future`
  = note: the return type of a function must have a statically known size

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: Could not compile `async-block-send`.

To learn more, run the command again with --verbose.
```



So, what's the problem of that code?

### Spurious Send Required

Let's look at a simpler code:

```rust
// examples/spurious-send.rs
use std::future::Future;
use std::pin::Pin;

fn f<T>(_: &T) -> Pin<Box<dyn Future<Output = ()> + Send>> {
    unimplemented!()
}

pub fn g<T: Sync>(x: &'static T) -> impl Future<Output = ()> + Send {
    async move { f(x).await }
}

fn main() {}
```

Run it:

```bash
cargo +nightly-2019-09-11 run --example spurious-send
```

Compiler complains:

```bash
error[E0277]: `T` cannot be sent between threads safely
 --> examples/spurious-send.rs:8:37
  |
8 | pub fn g<T: Sync>(x: &'static T) -> impl Future<Output = ()> + Send {
  |                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `T` cannot be sent between threads safely
  |
  = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `T`
  = help: consider adding a `where T: std::marker::Send` bound
  = note: required because it appears within the type `for<'r, 's, 't0, 't1> {for<'t2> fn(&'t2 T) -> std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 'static)>> {f::<T>}, &'r T, T, &'s T, std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 't0)>>, std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 't1)>>, ()}`
  = note: required because it appears within the type `[static generator@examples/spurious-send.rs:9:16: 9:30 x:&T for<'r, 's, 't0, 't1> {for<'t2> fn(&'t2 T) -> std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 'static)>> {f::<T>}, &'r T, T, &'s T, std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 't0)>>, std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 't1)>>, ()}]`
  = note: required because it appears within the type `std::future::GenFuture<[static generator@examples/spurious-send.rs:9:16: 9:30 x:&T for<'r, 's, 't0, 't1> {for<'t2> fn(&'t2 T) -> std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 'static)>> {f::<T>}, &'r T, T, &'s T, std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 't0)>>, std::pin::Pin<std::boxed::Box<(dyn std::future::Future<Output = ()> + std::marker::Send + 't1)>>, ()}]>`
  = note: required because it appears within the type `impl std::future::Future`
  = note: the return type of a function must have a statically known size

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: Could not compile `async-block-send`.

To learn more, run the command again with --verbose.
```

It's strange, isn't it? As we all know,  `T: Sync => &T: Send`, that code  should works. `T: Send` is unnecessary because there is no `T` value in async block.

This bug is introduced in nightly-09-11 and fixed by [rust-lang/rust#64584](https://github.com/rust-lang/rust/pull/64584), that code works now:

```bash
cargo +nightly-2019-11-05 run --example spurious-send
```

However, you still cannot use format in `await` expression.

### format

What's the problem with this code:

```rust
// examples/format.rs
async fn foo(_: String) {}

fn bar() -> impl Send {
    async move {
        foo(format!("")).await;
    }
}

fn main() {}
```

macro format will  expand `format!("")` into:

```rust
::alloc::fmt::format(::core::fmt::Arguments::new_v1(
            &[],
            &match () {
                () => [],
            },
        )))
```

This expression generate some temporaries that do not implement `Send`, like `std::fmt::ArgumentV1`, `&core::fmt::Void`. And, these temporaries stay alive at the end of current expression.

```rust
foo(::alloc::fmt::format(::core::fmt::Arguments::new_v1(
            &[],
            &match () {
                () => [],
            },
        )))).await;
^^^^^^^^^^^^^^^^^^^^^^ temporaries stay alive
```

So, these temporaries across 'await', which means they will be stored as field of a stackless generator (the `GenFuture` returned by async block).

> Read this [blog](https://github.com/Hexilee/async-io-demo#generator) for more contents about generator.

Eventually, `GenFuture: !Send` conflicts against `impl Send`.

### Solution

- Spurious send required was fixed by [rust-lang/rust#64584](https://github.com/rust-lang/rust/pull/64584), which was merged into master branch.

- Temporaries scoped `format` was implemented by [rust-lang/rust#64856](https://github.com/rust-lang/rust/pull/64856), but merging is blocked.

  You can override `format` by yourself, there is a polyfill:

  ```rust
  // examples/format-override.rs
  extern crate alloc;
  
  macro_rules! format {
      ($($arg:tt)*) => {{
          let res = alloc::fmt::format(alloc::__export::format_args!($($arg)*));
          res
      }}
  }
  
  async fn foo(_: String) {}
  
  fn bar() -> impl Send {
      async move {
          foo(format!("")).await;
      }
  }
  
  fn main() {}
  ```

  Run it:

  ```bash
  cargo +nightly-2019-11-05 run --example format-override
  ```

  Pass.

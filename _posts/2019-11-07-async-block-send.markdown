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

本文旨在总结并解释  [rust-lang/rust#64477](https://github.com/rust-lang/rust/issues/64477) 和 [ rust-lang/rust#64856](https://github.com/rust-lang/rust/pull/64856)。本文代码地址：[Github](https://github.com/Hexilee/async-block-send)。

### 引言

如果你正在使用 2019-09-11 到 2019-11-05（最新）版本的 rust nightly 工具链，你会发现下面这段代码无法通过编译：

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

尝试运行:

```bash
cargo +nightly-2019-09-11 run --example format
```

编译报错:

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



所以，这段代码到底有啥毛病呢？

### 多余的 `Send` 约束

让我们来看一段更简单的代码：

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

尝试运行:

```bash
cargo +nightly-2019-09-11 run --example spurious-send
```

编译报错:

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

这段报错信息令人匪夷所思。众所周知，如果 `T: Sync`，则有 `&T: Send`，所以这段代码应该是没问题的。`T: Send` 是不必要的，因为 async 块中不存在 `T` 类型的变量。

这个 bug 是 nightly-09-11 中引入的，并且已被 [rust-lang/rust#64584](https://github.com/rust-lang/rust/pull/64584) 修复，所以最新的工具链是能够编译运行这段代码的：

```bash
cargo +nightly-2019-11-05 run --example spurious-send
```

不过，在含 `.await` 的语句中依然无法使用 `format` 宏。

### format

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

`format` 宏会把 `format("")`展开成：

```rust
::alloc::fmt::format(::core::fmt::Arguments::new_v1(
            &[],
            &match () {
                () => [],
            },
        )))
```

这串表达式尝试了一些没有实现 `Send` 的临时变量，像  `std::fmt::ArgumentV1`, `&core::fmt::Void` 类型的变量。并且，这些临时变量会在当前语句结束之后才会被析构。

```rust
foo(::alloc::fmt::format(::core::fmt::Arguments::new_v1(
            &[],
            &match () {
                () => [],
            },
        )))).await;
^^^^^^^^^^^^^^^^^^^^^^ temporaries stay alive
```

所以，这些临时变量的生命期跨过了一次 `await`，这意味着它们会被作为无栈协程（由 async 块返回的 `GenFuture`）的一个字段存下来。

> 可以阅读这篇[博客](https://hexilee.me/2018/12/17/rust-async-io/)来了解有关 generator 的更多内容。

最终，这些没有实现 `Send`的临时变量会导致 `GenFuture: !Send`，这与函数返回类型  `impl Send`产生了冲突。

### 解决方案

- 多余的 `Send`约束问题已被  [rust-lang/rust#64584](https://github.com/rust-lang/rust/pull/64584) 解决，目前这个 PR　已经合入了 master 分区，并已经在最新的 nightly 工具链中生效。

-  [rust-lang/rust#64856](https://github.com/rust-lang/rust/pull/64856) 提出了一份新的 format 实现，该 PR 目前合并进程被阻塞，但你可以手动实现一份来覆盖标准库实现：

  {% raw %}
  
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

  {% endraw %}
  
  尝试运行：

  ```bash
  cargo +nightly-2019-11-05 run --example format-override
  ```
  
  通过。

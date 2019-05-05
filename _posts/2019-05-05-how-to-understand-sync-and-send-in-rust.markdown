---
layout:     post
title:      "如何理解 Sync 和 Send?"
subtitle:   "如何保证线程安全？"
date:       2019-05-05 13:52:00
author:     "Hexi"
tags:
    - rust
    - 线程安全
---
### 如何理解 Sync、Send？

`Sync` 和 `Send` 是 rust 安全并发中两个至关重要的 `marker`，但绝大多数的文档或书籍每当谈到它们就只是直接抛出它们的语义：

 - 实现了 `Send` 的类型，可以安全地在线程间传递所有权。也就是说， 可以跨线程移动。

 - 实现了 `Sync` 的类型， 可以安全地在线程间传递不可变借用。也就是说，可以跨线程共享。

这两句话的确很重要（没看过的读者可以多看几遍再继续阅读下文）。但如果只把这个拿出来，像我这样不熟练的 rust 用户可能会觉得似懂非懂，很多概念混杂在一起 —— rust 中关于可变不可变的讨论太多了。

#### 导火索 RwLock

我之所以决定彻底搞清楚这两个东西是因为我使用标准库中的 `RwLock` 遇到了一些问题，查看源码之后发现这两行（先不管 `Send`）：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<T: ?Sized + Send> Send for RwLock<T> {}
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<T: ?Sized + Send + Sync> Sync for RwLock<T> {}
```

稍懂 rust 的同学应该就可以看懂，这代码的意思是，只有当类型 `T` 实现了 `Sync`，`RwLock<T>` 才会实现 `Sync`。

欸！？我就纳闷了，读写锁读写锁，怎么说也是个锁。锁不就是把不 `Sync` 的类型变 `Sync` 的存在吗？

我马上又去看了一下 `Mutex`，果然不出我所料：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<T: ?Sized + Send> Send for Mutex<T> { }
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<T: ?Sized + Send> Sync for Mutex<T> { }
```

这 `Mutex` 看起来才像锁，`RwLock` 根本不符合我对锁的印象。但我又仔细想想，互斥锁和读写锁到底差在哪儿，导致了这种情况呢？—— 读写锁允许并行地读。

所以答案很明了了，如果 `T` 不 `Sync`，就不能让多个线程同时拿到 `T` 类型对象的**不可变引用**。

#### 并行读？内存不安全？

所以，并行只读会导致内存不安全吗？这似乎不符合直觉。那到底是啥原因呢？
这里可以思考一下，rust 的不可变引用真的“只读”吗？当然不是了，大家耳熟能详的 `Cell`、`RefCell` 就是拿不可变引用改变内部数据的典型用例。

所以，这个问题的本质是，rust 的不可变引用并没有对内部可变性做过强的约束。当然我最初期望的是完全内部不可变的，而事实也如此，当你完全使用 “safe rust” 的时候。

比如像这样：

```rust
struct A {
    data: i32
}

struct B {
    a: A
}

impl B {
    fn set_data(&self, data: i32) {
        self.a.data = data;
    }
}
```

是 rust 类型系统不允许的，你永远不能从不可变的 `B` 对象上**安全地**借到一个可变的 `data` 引用。

当然, 你使用 `unsafe` 就可以达到目的。

```rust
struct A {
    data: i32,
}

struct B {
    a: *mut A,
}

impl B {
    fn set_data(&self, data: i32) {
        unsafe { (*self.a).data = data };
    }
}
```

> 在实际遇到这种场景时应该使用 `Cell`, `RefCell`, `RwLock` 等标准库封装好的结构（当然它们内部实现还得 `unsafe`）。

所以只要 unsafe rust 还存在，你就不能假定不可变引用一定是“内部不可变”的。

#### 我们为什么需要使用 Unsafe

这就是一个历史悠久的问题了，答案可以分很多方面，比如要 `ffi`，再比如在某些场景使用 dirty 操作减少拷贝提升性能。但回到我们的前文，我们为什么要让不可变引用“内部可变”呢？

我们先思考另一个问题，如果我们不使用 `unsafe`，在 rust 类型系统中，一个对象的可变引用永远只能同时存在一个，这样的话我们如果想在多个线程中使用可变引用要怎么写呢？

只能像踢球一样把可变引用在线程间传来传去，当然因为引用的生命周期问题我们一般选择把所有权在线程间传递。那怎么传呢？如果用 `channel` 的话，`sender`、`receiver` 本身是不是就得**共享**可变引用呢？

最后的结论就是我们不得不用，我们迫真地需要让不可变引用“内部可变”的操作。那既然这个需求不可避免，我们又要怎样保证 rust 的内存安全呢？

#### Sync: make unsafe rust safe

我们再回到 `Sync` 的定义：

- 实现了 `Sync` 的类型， 可以安全地在线程间传递不可变借用。也就是说，可以跨线程共享。

所以，符合这个要求的类型有两种：

第一种类型你永远不能通过它的不可变引用改变它的内部，它所有的 `pub field` 都是 `Sync` 的，然后所有的以 `&self` 作为 `receiver` 的 `pub method` 也都不改变自身（或返回内部可变引用）。

第二种类型当然所有的 `pub field` 都得是 `Sync` 的，但它可能存在以 `&self` 作为 `receiver` 的 `pub method` 能改变自身（或返回内部可变引用），只不过这些方法本身得自己保证多线程访问的安全性，比如使用锁或者原子。

其它的类型都是 `!Sync` 的。

我们可以举几个栗子看看：

- 自然实现 `Sync` 的类型，即 `field` 全部 `Sync`，那它所有的 `pub field` 显然都是 `Sync` 的。其以 `&self` 为 `receiver` 的 `pub method` 也只能使用 `field` 的 `pub method`，显然都是不可变的。
- 上文中的 `B` 类型，如果让 `set_data` 方法 `pub` 的话它就不应该 `Sync` （即不应该 `unsafe impl Sync for B {}`）。
- `RefCell<T>`，多个线程可以同时通过其不可变引用持有 `T` 的“可变引用”（当然这会导致 panic），进而改变其内部，`!Sync`。
- `RwLock<T>`，多个线程不能同时通过其不可变引用持有 `T` 的“可变引用”，也不可能同时持有“可变引用”和“不可变引用”，但可以同时持有“不可变引用”。所以：
    - 如果 `T: Sync`，则 `RwLock<T>: Sync`
    - 如果 `T: !Sync`，则 `RwLock<T>: !Sync` 

所以，如果你自己写了一个并不天然实现 `Sync` 的类型，不妨对照这两条要求考虑要不要给它实现一个 `Sync`。

#### Send: 安全共享

关于 `Send` 和 `Sync` 的联系，大多数文档都会说 “只要实现了 `Sync` 的类型，其不可变借用就可以 安全地在线程间共享”。相信看到这里的读者肯定就能理解，为什么 `&T: Send` 的要求是 `T: Sync`；而 `&mut T: Send` 的要求却更宽松，只需要 `T: Send`。

因为 `&mut T` 的 `Send` 意味着 `move`，而 `&T` 的 `Send` 意味着 `share`。要想多线程共享 `&T`， `T` 就必须 `Sync`。

包括像 `Arc<T>` 这样的 "`shared_pointer`" 也是如此：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<T: ?Sized + Sync + Send> Send for Arc<T> {}
```

当然，对于所有的非引用类型来说，大部分情况下并不需要开发者考虑它是否应该实现 `Send`，除非你要写智能指针之类的东西，不然你只需要考虑给它实现一个 `Sync` 从而让它的引用类型实现 `Send`。


#### 总结

所以，我之所以觉得 `RwLock` 应该把 `!Sync` 的类型包装成 `Sync` 的类型本质上是因为我错误地理解了 `Sync` 的语义。rust 的可变引用要求过于严苛导致我们很多时候必须使用不可变引用来改变自身，所以 `Sync` 是用来标记不可变借用可线程安全地访问的。

至于可变引用，因为永远只同时存在一个可变引用，且其不与不可变引用共存，所以以可变引用为 `receiver` 的方法永远是线程安全的，无需其它的约束。

对于 `RwLock<T>`， 你应该保证 `T` 是 Sync 的，否则它就和 `RefCell<T>` 没啥区别。只不过原本因为 `RefCell` 会 panic 的逻辑现在会因为 `RwLock` 死锁。



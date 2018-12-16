---
layout:     post
title:      rust asynchronous io
subtitle:   "从 mio 到 coroutine"
date:       2018-12-15 00:27:00
author:     "Hexi"
header-img: "img/bg/2018-12-15-rust-async-io.jpg"
tags:
    - rust
    - asynchronous
    - io
---

### 引言

2018 年接近尾声，`rust` 团队勉强立住了异步 `IO` 的 flag，`async` 成为了关键字，`Pin`, `Future`, `Poll` 和 `await!` 也进入了标准库。不过一直以来因为实际项目用不到，所以也没有主动去了解这套东西。

最近心血来潮想用 `rust` 写点东西，但并找不到比较能看的文档（可能是因为 `rust` 发展太快了，很多都过时了），最后参考[这篇文章](https://cafbit.com/post/tokio_internals/)和 `"new tokio"`( [romio](https://github.com/Hexilee/async-io-demo) ) 写了几个 `demo`，并基于 `mio` 在 `coroutine` 中实现了简陋的异步 `IO`。

最终效果如下：

```rust
// examples/async-echo.rs

#![feature(async_await)]
#![feature(await_macro)]

#[macro_use]
extern crate log;

use asyncio::executor::{block_on, spawn, TcpListener};

fn main() {
    env_logger::init();
    block_on(async {
        let mut listener = TcpListener::bind(&"127.0.0.1:7878".parse().unwrap()).expect("TcpListener bind fail");
        info!("Listening on 127.0.0.1:7878");
        while let Ok((mut stream, addr)) = await!(listener.accept()) {
            info!("connection from {}", addr);
            spawn(async move {
                let client_hello = await!(stream.read()).unwrap();
                let read_length = client_hello.len();
                let write_length = await!(stream.write(client_hello)).unwrap();
                assert_eq!(read_length, write_length);
                stream.close();
            });
        }
    })
}
```

写这篇文章的主要目的是梳理和总结，同时也希望能给对这方面有兴趣的 `Rustacean` 作为参考。本文代码以易于理解为主要编码原则，某些地方并没有太考虑性能，还请见谅；但如果文章和代码中有明显错误，欢迎指正。

本文代码仓库在 [Github](https://github.com/Hexilee/async-io-demo)，所有 `examples` 在 `nightly-x86_64-apple-darwin 2018 Edition` 上均能照常运行。运行 `example/async-echo`  时设置 `RUST_LOG` 为 `info` 可以在 terminal 看到基本的运行信息，`debug` 则可见事件循环中的事件触发顺序。

### 异步 `IO` 的基石 - `mio`

`mio` 是一个极简的底层异步 `IO` 库，如今 `rust` 生态中几乎所有的异步 `IO` 程序都基于它。

随着 `channel`, `timer` 等 `sub module` 在 `0.6.5` 版本被标为 `deprecated`，如今的 mio 提供的唯二两个核心功能分别是：

- 对操作系统异步网络 `IO` 的封装
- 用户自定义事件队列

第一个核心功能对应到不同操作系统分别是

- `Linux(Android) => epoll`
- `Windows => iocp`
- `MacOS(iOS), FreeBSD => kqueue` 
- `Fuchsia => <unknown>`

mio 把这些不同平台上的 API 封装出了一套 `epoll like` 的异步网络 API，支持 `udp 和 tcp`。

> 除此之外还封装了一些不同平台的拓展 API，比如 `uds`，本文不对这些 API 做介绍。

#### 异步网络 IO

下面是一个 `tcp` 的 `demo`

```rust
// examples/tcp.rs

use mio::*;
use mio::net::{TcpListener, TcpStream};
use std::io::{Read, Write, self};
use failure::Error;
use std::time::{Duration, Instant};

const SERVER_ACCEPT: Token = Token(0);
const SERVER: Token = Token(1);
const CLIENT: Token = Token(2);
const SERVER_HELLO: &[u8] = b"PING";
const CLIENT_HELLO: &[u8] = b"PONG";

fn main() -> Result<(), Error> {
    let addr = "127.0.0.1:13265".parse()?;

// Setup the server socket
    let server = TcpListener::bind(&addr)?;

// Create a poll instance
    let poll = Poll::new()?;

// Start listening for incoming connections
    poll.register(&server, SERVER_ACCEPT, Ready::readable(),
                  PollOpt::edge())?;

// Setup the client socket
    let mut client = TcpStream::connect(&addr)?;

    let mut server_handler = None;

// Register the client
    poll.register(&client, CLIENT, Ready::readable() | Ready::writable(),
                  PollOpt::edge())?;

// Create storage for events
    let mut events = Events::with_capacity(1024);

    let start = Instant::now();
    let timeout = Duration::from_millis(10);
    'top: loop {
        poll.poll(&mut events, None)?;
        for event in events.iter() {
            if start.elapsed() >= timeout {
                break 'top
            }
            match event.token() {
                SERVER_ACCEPT => {
                    let (handler, addr) = server.accept()?;
                    println!("accept from addr: {}", &addr);
                    poll.register(&handler, SERVER, Ready::readable() | Ready::writable(), PollOpt::edge())?;
                    server_handler = Some(handler);
                }

                SERVER => {
                    if event.readiness().is_writable() {
                        if let Some(ref mut handler) = &mut server_handler {
                            match handler.write(SERVER_HELLO) {
                                Ok(_) => {
                                    println!("server wrote");
                                }
                                Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                                err => {
                                    err?;
                                }
                            }
                        }
                    }
                    if event.readiness().is_readable() {
                        let mut hello = [0; 4];
                        if let Some(ref mut handler) = &mut server_handler {
                            match handler.read_exact(&mut hello) {
                                Ok(_) => {
                                    assert_eq!(CLIENT_HELLO, &hello);
                                    println!("server received");
                                }
                                Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                                err => {
                                    err?;
                                }
                            }
                        }
                    }
                }
                CLIENT => {
                    if event.readiness().is_writable() {
                        match client.write(CLIENT_HELLO) {
                            Ok(_) => {
                                println!("client wrote");
                            }
                            Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                            err => {
                                err?;
                            }
                        }
                    }
                    if event.readiness().is_readable() {
                        let mut hello = [0; 4];
                        match client.read_exact(&mut hello) {
                            Ok(_) => {
                                assert_eq!(SERVER_HELLO, &hello);
                                println!("client received");
                            }
                            Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                            err => {
                                err?;
                            }
                        }
                    }
                }
                _ => unreachable!(),
            }
        }
    };
    Ok(())
}
```

这个 `demo` 稍微有点长，接下来我们把它一步步分解。

直接看主循环

```rust
fn main() {
    // ...
    loop {
        poll.poll(&mut events, None).unwrap();
        // ...
    }
}
```

每次循环都得执行 `poll.poll`，第一个参数是用来存 `events` 的 `Events`， 容量是 `1024`；

```rust
let mut events = Events::with_capacity(1024);
```

第二个参数是 `timeout`，即一个 `Option<Duration>`，超时会直接返回。返回类型是 `io::Result<usize>`。

> 其中的 `usize` 代表 `events` 的数量，这个返回值是 `deprecated` 并且会在之后的版本移除，仅供参考

这里我们设置了 `timeout = None`，所以当这个函数返回时，必然是某些事件被触发了。让我们遍历 `events`：

```rust
  match event.token() {
      SERVER_ACCEPT => {
          let (handler, addr) = server.accept()?;
          println!("accept from addr: {}", &addr);
          poll.register(&handler, SERVER, Ready::readable() | Ready::writable(), PollOpt::edge())?;
          server_handler = Some(handler);
      }

      SERVER => {
          if event.readiness().is_writable() {
              if let Some(ref mut handler) = &mut server_handler {
                  match handler.write(SERVER_HELLO) {
                      Ok(_) => {
                          println!("server wrote");
                      }
                      Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                      err => {
                          err?;
                      }
                  }
              }
          }
          if event.readiness().is_readable() {
              let mut hello = [0; 4];
              if let Some(ref mut handler) = &mut server_handler {
                  match handler.read_exact(&mut hello) {
                      Ok(_) => {
                          assert_eq!(CLIENT_HELLO, &hello);
                          println!("server received");
                      }
                      Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                      err => {
                          err?;
                      }
                  }
              }
          }
      }
      CLIENT => {
          if event.readiness().is_writable() {
              match client.write(CLIENT_HELLO) {
                  Ok(_) => {
                      println!("client wrote");
                  }
                  Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                  err => {
                      err?;
                  }
              }
          }
          if event.readiness().is_readable() {
              let mut hello = [0; 4];
              match client.read_exact(&mut hello) {
                  Ok(_) => {
                      assert_eq!(SERVER_HELLO, &hello);
                      println!("client received");
                  }
                  Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
                  err => {
                      err?;
                  }
              }
          }
      }
      _ => unreachable!(),
  }
```

我们匹配每一个 `event` 的 `token`，这里的 `token` 就是我用来注册的那些 `token`。比如我在上面注册了 `server`

```rust
// Start listening for incoming connections
poll.register(&server, SERVER_ACCEPT, Ready::readable(),
                  PollOpt::edge()).unwrap();

```

第二个参数就是 `token`

```rust
const SERVER_ACCEPT: Token = Token(0);
```

这样当 `event.token() == SERVER_ACCEPT` 时，就说明这个事件跟我们注册的 `server` 有关，于是我们试图 `accept` 一个新的连接并把它注册进 `poll`，使用的 `token` 是 `SERVER`。

```rust
let (handler, addr) = server.accept().unwrap();
println!("accept from addr: {}", &addr);
poll.register(&handler, SERVER, Ready::readable() | Ready::writable(), PollOpt::edge()).unwrap();
server_handler = Some(handler);
```

这样我们之后如果发现 `event.token() == SERVER`，我们就认为它和注册的 `handler` 有关：

```rust
if event.readiness().is_writable() {
    if let Some(ref mut handler) = &mut server_handler {
        match handler.write(SERVER_HELLO) {
            Ok(_) => {
                println!("server wrote");
            }
            Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
            err => {
                err?;
            }
        }
    }
}
if event.readiness().is_readable() {
    let mut hello = [0; 4];
    if let Some(ref mut handler) = &mut server_handler {
        match handler.read_exact(&mut hello) {
            Ok(_) => {
                assert_eq!(CLIENT_HELLO, &hello);
                println!("server received");
            }
            Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
            err => {
                err?;
            }
        }
    }
}
```

这时候我们还需要判断 `event.readiness()`，这就是 `register` 函数的第三个参数，叫做 `interest`，顾名思义，就是“感兴趣的事”。它的类型是 `Ready`，一共四种，`readable, writable, error 和 hup`，可进行并运算。

在上面我们给 `handler` 注册了 `Ready::readable() | Ready::writable()`，所以 `event` 可能是 `readable` 也可能是 `writable`，所以我们要经过判断来执行相应的逻辑。注意这里的判断是

```rust
if ... {
    ...
}

if ... {
    ...
}
```

而非

```rust
if ... {
    ...
} else if ... {
    ...
}
```

因为一个事件可能同时是 `readable` 和 `writable`。

#### 容错性原则

大概逻辑先讲到这儿，这里先讲一下 `mio` 的“容错性原则”，即不能完全相信 `event`。

可以看到我上面有一段代码是这么写的 

```rust
match event.token() {
     SERVER_ACCEPT => {
         let (handler, addr) = server.accept().unwrap();
         println!("accept from addr: {}", &addr);
         poll.register(&handler, SERVER, Ready::readable() | Ready::writable(), PollOpt::edge()).unwrap();
         server_handler = Some(handler);
     }
```

`server.accept()` 返回的是 `io::Result<(TcpStream, SocketAddr)>`。如果我们选择完全相信 `event` 的话，在这里 `unwrap()` 并没有太大问题 —— 如果真的有一个新的连接就绪，`accept()` 产生的 `io::Result` 是我们无法预料且无法处理的，我们应该抛给调用者或者直接 `panic`。

但问题就是，我们可以认为 `event` 的伪消息是可预料的，可能并没有一个新的连接准备就绪，这时候我们 `accept()` 会引发 `WouldBlock Error`。但我们不应该认为 `WouldBlock` 是一种错误 —— 这是一种友善的提醒。`server` 告诉我们：“并没有新的连接，请下次再来吧。”，所以在这里我们应该忽略（可以打个 `log`）它并重新进入循环。

像我后面写的那样：

```rust
match client.write(CLIENT_HELLO) {
   Ok(_) => {
       println!("client wrote");
   }
   Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
   err => {
       err?;
   }
}
```

#### Poll Option

好了，现在我们可以运行：

```bash
[async-io-demo] cargo run --example tcp
```

terminal 里打印出了

```bash
client wrote
accept from addr: 127.0.0.1:53205
client wrote
server wrote
server received
...
```

我们可以发现，在短短的 `10 millis` 内（`let timeout = Duration::from_millis(10);`），`server` 和 `client` 分别进行了数十次的读写！

如果我们不想进行这么多次读写呢？比如，我们只想让 `server` 写一次。在网络比较通畅的情况下，`client` 和 `server` 几乎一直是可写的，所以 `Poll::poll` 在数微秒内就返回了。

这时候就要看 `register` 的第四个参数了。


```rust
poll.register(&server, SERVER_ACCEPT, Ready::readable(),
                  PollOpt::edge()).unwrap();

```

`PollOpt::edge()` 的类型是 `PollOpt`，一共有 `level, edge, oneshot` 三种，他们有什么区别呢？

比如在我上面的代码里，

```rust
if event.readiness().is_readable() {
    let mut hello = [0; 4];
    match client.read_exact(&mut hello) {
        Ok(_) => {
            assert_eq!(SERVER_HELLO, &hello);
            println!("client received");
        }
        Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => continue,
        err => {
            err?;
        }
    }
}
```

我在收到一个 `readable readiness` 时，只读了四个字节。如果这时候缓冲区里有八字节的数据，那么：

- 如果我注册时使用 `PollOpt::level()`，我在下次 `poll` 时 **一定** 还能收到一次 `readable readiness event` （只要我没有主动执行 `set_readiness(Read::empty())`）；
- 如果我注册时使用 `PollOpt::edge()`，我在下次 `poll` 时 **不一定** 还能收到一次 `readable readiness event`；

所以，使用 `PollOpt::edge()` 时有一个“排尽原则（`Draining readiness`）”，即每次触发 `event` 时一定要操作到资源耗尽返回 `WouldBlock`，即上面的代码要改成：

```rust
if event.readiness().is_readable() {
    let mut hello = [0; 4];
    loop {
        match client.read_exact(&mut hello) {
            Ok(_) => {
                assert_eq!(SERVER_HELLO, &hello);
                println!("client received");
            }
            Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => break,
            err => {
                err?;
            }
        }
    }
}
```

那么，`oneshot` 又是怎样的行为呢？让我们回到上面的问题，如果我们只想让 `handler` 写一次，怎么办 —— 注册时使用 `PollOpt::oneshot()`，即

```rust
let (handler, addr) = server.accept().unwrap();
println!("accept from addr: {}", &addr);
poll.register(&handler, SERVER, Ready::readable(), PollOpt::edge()).unwrap();
poll.register(&handler, SERVER_WRITE, Ready::writable(), PollOpt::oneshot()).unwrap();
server_handler = Some(handler);
```

这样的话，你只能收到一次 `SERVER_WRITE` 事件，除非你使用 `Poll::reregister` 重新注册 `handler`。

> `Poll::reregister` 可以更改 `PollOpt` 和 `interest`


#### Still Block

其实上面这个 `demo` 还存在一个问题，即我们在回调代码块中使用了同步的 `IO` 操作 `println!`。我们要尽可能避免在回调的代码块里使用耗时的 `IO` 操作。

考虑到文件 `IO` (包括 `Stdin, Stdout, Stderr`) 速度很慢，我们只需要把所有的文件 `IO` 交给一个线程进行即可。

```rust
use std::sync::mpsc::{Sender, Receiver, channel, SendError};

#[derive(Clone)]
pub struct Fs {
    task_sender: Sender<Task>,
}

impl Fs {
    pub fn new() -> Self {
        let (sender, receiver) = channel();
        std::thread::spawn(move || {
            loop {
                match receiver.recv() {
                    Ok(task) => {
                        match task {
                            Task::Println(ref string) => println!("{}", string),
                            Task::Exit => return
                        }
                    },
                    Err(_) => {
                        return;
                    }
                }
            }
        });
        Fs { task_sender: sender }
    }

    pub fn println(&self, string: String) {
        self.task_sender.send(Task::Println(string)).unwrap()
    }
}

pub enum Task {
    Exit,
    Println(String),
}
```

之后，可以使用 `Fs::println` 替换所有的 `println!`。

#### 自定义事件


上面我们实现异步 `println` 比较简单，这是因为 `println` 并没有返回值，不需要进行后续操作。设想一下，如果要我们实现 `open` 和 `ready_to_string`，先异步地 `open` 一个文件，然后异步地 `read_to_string`，最后再异步地 `println`, 我们要怎么做？

最简单的写法是回调，像这样：

```rust
// src/fs.rs

use crossbeam_channel::{unbounded, Sender};
use std::fs::File;
use std::io::Read;
use std::boxed::FnBox;
use std::thread;
use failure::Error;

#[derive(Clone)]
pub struct Fs {
    task_sender: Sender<Task>,
}

pub struct FsHandler {
    io_worker: thread::JoinHandle<Result<(), Error>>,
    executor: thread::JoinHandle<Result<(), Error>>,
}

pub fn fs_async() -> (Fs, FsHandler) {
    let (task_sender, task_receiver) = unbounded();
    let (result_sender, result_receiver) = unbounded();
    let io_worker = std::thread::spawn(move || {
        loop {
            match task_receiver.recv() {
                Ok(task) => {
                    match task {
                        Task::Println(ref string) => println!("{}", string),
                        Task::Open(path, callback, fs) => {
                            result_sender
                                .send(TaskResult::Open(File::open(path)?, callback, fs))?
                        }
                        Task::ReadToString(mut file, callback, fs) => {
                            let mut value = String::new();
                            file.read_to_string(&mut value)?;
                            result_sender
                                .send(TaskResult::ReadToString(value, callback, fs))?
                        }
                        Task::Exit => {
                            result_sender
                                .send(TaskResult::Exit)?;
                            break;
                        }
                    }
                }
                Err(_) => {
                    break;
                }
            }
        }
        Ok(())
    });
    let executor = std::thread::spawn(move || {
        loop {
            let result = result_receiver.recv()?;
            match result {
                TaskResult::ReadToString(value, callback, fs) => callback.call_box((value, fs))?,
                TaskResult::Open(file, callback, fs) => callback.call_box((file, fs))?,
                TaskResult::Exit => break
            };
        };
        Ok(())
    });

    (Fs { task_sender }, FsHandler { io_worker, executor })
}

impl Fs {
    pub fn println(&self, string: String) -> Result<(), Error> {
        Ok(self.task_sender.send(Task::Println(string))?)
    }

    pub fn open<F>(&self, path: &str, callback: F) -> Result<(), Error>
        where F: FnOnce(File, Fs) -> Result<(), Error> + Sync + Send + 'static {
        Ok(self.task_sender.send(Task::Open(path.to_string(), Box::new(callback), self.clone()))?)
    }

    pub fn read_to_string<F>(&self, file: File, callback: F) -> Result<(), Error>
        where F: FnOnce(String, Fs) -> Result<(), Error> + Sync + Send + 'static {
        Ok(self.task_sender.send(Task::ReadToString(file, Box::new(callback), self.clone()))?)
    }

    pub fn close(&self) -> Result<(), Error> {
        Ok(self.task_sender.send(Task::Exit)?)
    }
}

impl FsHandler {
    pub fn join(self) -> Result<(), Error> {
        self.io_worker.join().unwrap()?;
        self.executor.join().unwrap()
    }
}

type FileCallback = Box<FnBox(File, Fs) -> Result<(), Error> + Sync + Send>;
type StringCallback = Box<FnBox(String, Fs) -> Result<(), Error> + Sync + Send>;

pub enum Task {
    Exit,
    Println(String),
    Open(String, FileCallback, Fs),
    ReadToString(File, StringCallback, Fs),
}

pub enum TaskResult {
    Exit,
    Open(File, FileCallback, Fs),
    ReadToString(String, StringCallback, Fs),
}

```

```rust
// examples/fs.rs

use asyncio::fs::fs_async;
use failure::Error;

const TEST_FILE_VALUE: &str = "Hello, World!";

fn main() -> Result<(), Error> {
    let (fs, fs_handler) = fs_async();
    fs.open("./examples/test.txt", |file, fs| {
        fs.read_to_string(file, |value, fs| {
            assert_eq!(TEST_FILE_VALUE, &value);
            fs.println(value)?;
            fs.close()
        })
    })?;
    fs_handler.join()?;
    Ok(())
}
```

测试

```bash
[async-io-demo] cargo run --example fs
```

这样写在逻辑上的确是对的，但是负责跑 `callback` 的 `executor` 线程其实被负责 `io` 的线程阻塞住了（`result_receiver.recv()`）。那我们能不能在 `executor` 线程里跑一个事件循环，以达到不被 `io` 线程阻塞的目的呢？（即确定 `result_receiver` 中有 `result` 时，`executor` 才会进行 `result_receiver.recv()`）.

这就到了体现 `mio` 强大可拓展性的时候：注册用户态的事件队列。

把上面的代码稍加修改，就成了这样：

```rust
// src/fs_mio.rs

use crossbeam_channel::{unbounded, Sender, TryRecvError};
use std::fs::File;
use std::io::{Read};
use std::boxed::FnBox;
use std::thread;
use failure::Error;
use std::time::Duration;
use mio::*;

#[derive(Clone)]
pub struct Fs {
    task_sender: Sender<Task>,
}

pub struct FsHandler {
    io_worker: thread::JoinHandle<Result<(), Error>>,
    executor: thread::JoinHandle<Result<(), Error>>,
}

const FS_TOKEN: Token = Token(0);

pub fn fs_async() -> (Fs, FsHandler) {
    let (task_sender, task_receiver) = unbounded();
    let (result_sender, result_receiver) = unbounded();
    let poll = Poll::new().unwrap();
    let (registration, set_readiness) = Registration::new2();
    poll.register(&registration, FS_TOKEN, Ready::readable(), PollOpt::oneshot()).unwrap();
    let io_worker = std::thread::spawn(move || {
        loop {
            match task_receiver.recv() {
                Ok(task) => {
                    match task {
                        Task::Println(ref string) => println!("{}", string),
                        Task::Open(path, callback, fs) => {
                            result_sender
                                .send(TaskResult::Open(File::open(path)?, callback, fs))?;
                            set_readiness.set_readiness(Ready::readable())?;
                        }
                        Task::ReadToString(mut file, callback, fs) => {
                            let mut value = String::new();
                            file.read_to_string(&mut value)?;
                            result_sender
                                .send(TaskResult::ReadToString(value, callback, fs))?;
                            set_readiness.set_readiness(Ready::readable())?;
                        }
                        Task::Exit => {
                            result_sender
                                .send(TaskResult::Exit)?;
                            set_readiness.set_readiness(Ready::readable())?;
                            break;
                        }
                    }
                }
                Err(_) => {
                    break;
                }
            }
        }
        Ok(())
    });

    let executor = thread::spawn(move || {
        let mut events = Events::with_capacity(1024);
        'outer: loop {
            poll.poll(&mut events, Some(Duration::from_secs(1)))?;
            for event in events.iter() {
                match event.token() {
                    FS_TOKEN => {
                        loop {
                            match result_receiver.try_recv() {
                                Ok(result) => {
                                    match result {
                                        TaskResult::ReadToString(value, callback, fs) => callback.call_box((value, fs))?,
                                        TaskResult::Open(file, callback, fs) => callback.call_box((file, fs))?,
                                        TaskResult::Exit => break 'outer
                                    }
                                }
                                Err(e) => {
                                    match e {
                                        TryRecvError::Empty => break,
                                        TryRecvError::Disconnected => Err(e)?
                                    }
                                }
                            }
                        }
                        poll.reregister(&registration, FS_TOKEN, Ready::readable(), PollOpt::oneshot())?;
                    }
                    _ => unreachable!()
                }
            }
        };
        Ok(())
    });
    (Fs { task_sender }, FsHandler { io_worker, executor })
}

impl Fs {
    pub fn println(&self, string: String) -> Result<(), Error> {
        Ok(self.task_sender.send(Task::Println(string))?)
    }

    pub fn open<F>(&self, path: &str, callback: F) -> Result<(), Error>
        where F: FnOnce(File, Fs) -> Result<(), Error> + Sync + Send + 'static {
        Ok(self.task_sender.send(Task::Open(path.to_string(), Box::new(callback), self.clone()))?)
    }

    pub fn read_to_string<F>(&self, file: File, callback: F) -> Result<(), Error>
        where F: FnOnce(String, Fs) -> Result<(), Error> + Sync + Send + 'static {
        Ok(self.task_sender.send(Task::ReadToString(file, Box::new(callback), self.clone()))?)
    }

    pub fn close(&self) -> Result<(), Error> {
        Ok(self.task_sender.send(Task::Exit)?)
    }
}

impl FsHandler {
    pub fn join(self) -> Result<(), Error> {
        self.io_worker.join().unwrap()?;
        self.executor.join().unwrap()
    }
}

type FileCallback = Box<FnBox(File, Fs) -> Result<(), Error> + Sync + Send>;
type StringCallback = Box<FnBox(String, Fs) -> Result<(), Error> + Sync + Send>;

pub enum Task {
    Exit,
    Println(String),
    Open(String, FileCallback, Fs),
    ReadToString(File, StringCallback, Fs),
}

pub enum TaskResult {
    Exit,
    Open(File, FileCallback, Fs),
    ReadToString(String, StringCallback, Fs),
}

```

```rust
// examples/fs-mio.rs

use asyncio::fs_mio::fs_async;
use failure::Error;

const TEST_FILE_VALUE: &str = "Hello, World!";

fn main() -> Result<(), Error> {
    let (fs, fs_handler) = fs_async();
    fs.open("./examples/test.txt", |file, fs| {
        fs.read_to_string(file, |value, fs| {
            assert_eq!(TEST_FILE_VALUE, &value);
            fs.println(value)?;
            fs.close()
        })
    })?;
    fs_handler.join()?;
    Ok(())
}
```

测试

```bash
[async-io-demo] cargo run --example fs-mio
```

#### Callback is evil

既然文件 `IO` 的 `executor` 不再会被 `io worker` 线程阻塞了，那我们来试试让 `fs` 和 `tcp`  共用一个 `poll` 然后建立一个简单的文件服务器吧。

但是我已经开始觉得写 `callback` 有点难受了 —— 如果我们还想处理错误的话，会觉得更难受，像这样

```rust
use asyncio::fs_mio::fs_async;
use failure::Error;

const TEST_FILE_VALUE: &str = "Hello, World!";

fn main() -> Result<(), Error> {
    let (fs, fs_handler) = fs_async();
    fs.open("./examples/test.txt", 
        |file, fs| {
            fs.read_to_string(file, 
                |value, fs| {
                    assert_eq!(TEST_FILE_VALUE, &value);
                    fs.println(value, 
                        |err| {
                            ...
                        }
                    );
                    fs.close()
                },
                |err| {
                    ...
                }
            )
        },
        |err| {
            ...
        }
    )?;
    fs_handler.join()?;
    Ok(())
}
```

而且对 `rust` 来说，更加艰难的是闭包中的生命周期问题（闭包几乎不能通过捕获来借用环境变量）。这就意味着，如果我要借用环境中的某个变量，我要么 `clone` 它（如果它实现了 `Clone` 的话），要么把它作为闭包参数传入（意味着你要根据需要改每一层回调函数的签名，这太屎了）。

考虑到各种原因，`rust` 最终选择用 `coroutine` 作为异步 `IO` 的 `API` 抽象。

### coroutine

这里所说的 `coroutine` 是指基于 `rust generator` 的 `stackless coroutine` 而非早期被 `rust` 抛弃的 `green thread(stackful coroutine)`。

#### generator

`rust` 大概在今年五月份引入了 `generator`，但到现在还是 unstable 的 —— 虽说也没多少人用 stable（误

一个典型的斐波那契 `generator` 如下

```rust
// examples/fab.rs

#![feature(generators, generator_trait)]

use std::ops::{Generator, GeneratorState};

fn main() {
    let mut gen = fab(5);
    loop {
        match unsafe { gen.resume() } {
            GeneratorState::Yielded(value) => println!("yield {}", value),
            GeneratorState::Complete(ret) => {
                println!("return {}", ret);
                break;
            }
        }
    }
}

fn fab(mut n: u64) -> impl Generator<Yield=u64, Return=u64> {
    move || {
        let mut last = 0u64;
        let mut current = 1;
        yield last;
        while n > 0 {
            yield current;
            let tmp = last;
            last = current;
            current = tmp + last;
            n -= 1;
        }
        return last;
    }
}
```

由于 `generator` 的“中断特性”，我们很自然的可以想到，如果用 `generator` 搭配 `mio`，给每个 `generator` 分配一个 `token`，然后 `poll mio` 的事件循环，收到一个唤醒事件就 `resume` 相应的 `generator`；每个 `generator` 在要阻塞的时候拿自己的 `token` 注册一个唤醒事件然后 `yield`，不就实现了“同步代码”的异步 `IO` 吗？

这样看来原理上来说已经稳了，但 `rust` 异步 `IO` 的天空依旧漂浮着两朵乌云。

#### 自引用

第一朵乌云和 `rust` 本身的语言特性有关。

如果你写出这样的 `generator`

```rust
fn self_ref_generator() -> impl Generator<Yield=u64, Return=()> {
    || {
        let x: u64 = 1;
        let ref_x: &u64 = &x;
        yield 0;
        yield *ref_x;
    }
}
```

`rust` 一定会给你抛个错然后告诉你 "borrow may still be in use when generator yields"。编译器没有教我怎么修正让我有些恐慌，我赶紧去不存在的搜索引擎上查了查，发现这和 `generator` 的实现有关。

前文中提到，`rust generator` 是 `stackless` 的，即它并不会保留一个完整的栈，而是根据需要保留变量。如果你把上面的代码改成

```rust
fn no_ref_generator() -> impl Generator<Yield=u64, Return=()> {
    || {
        let x: u64 = 1;
        let ref_x: &u64 = &x;
        yield *ref_x;
        yield 0;
    }
}
```

在第一次 `yield` 结束之后，编译器会发现 `generator` 唯一需要保留的是字面量 `0`，所以这段代码可以顺利编译通过。但是，对于前面的 `generator`，第一次 `yield` 过后，编译器发现你需要同时保留 `x` 和它的引用 `ref_x`，这样的话 `generator` 就会变成类似这样的结构：

```rust

enum SomeGenerator {
    ...

    ...

}

```



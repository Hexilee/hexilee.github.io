---
layout:     post
title:      "小心使用 golang 的 init 函数"
subtitle:   "原因及解决方案？"
date:       2022-01-13 19:00:00
author:     "Hexi"
header-img: "img/bg/2022-01-13-be-careful-with-go-init.jpg"
tags:
    - golang
---

- ## init 函数简要介绍
  init 函数是 go 中的 package 初始化函数。例如有一个 http 客户端包 A：
  ``` go
	  package A
	  
	  import "fmt"
	  
	  var Dial func(string) (io.ReadWriteCloser, error)
	  
	  func Get(address string) (*Response, error) {
	    if Dial == nil {
	      return fmt.Errorf("Dial is not registered")
	    }
	    
	    ...
	  }
  ```
  它需要传输层插件库来实现 `Dial` 函数，比如有一个这样的库叫 B，Dial 函数注入方式如下：
  ``` go
	  package B
	  
	  import (
	    "A"
	    "net"
	  )
	  
	  func init() {
	    if A.Dial != nil {
	      panic("Dial is already registered")
	    }
	    A.Dial = func(address string) (io.ReadWriteCloser, error) {
	      return net.Dial("tcp", address )
	    }
	  }
  ```
  这样 A 库的用户就可以自由选择传输层的具体实现：
  ``` go
	  package main
	  
	  import (
	    "A"
	    _ "B" // 引入传输层插件库 B
	  )
	  
	  func main() {
	    resp, err := A.Get("http://google.com")
	    ...
	  }
  ```
- ## 带来的问题
	- ### 不同插件的冲突
	  这样的插件库设计带来的首要问题就是不同插件之间的冲突。比如我们有一个跟 B 类似实现的插件库 C，那用户同时引入时就会运行时 panic：
	  ``` go
	  		  package main
	  		  
	  		  import (
	  		    "A"
	  		    _ "B" // 引入传输层插件库 B
	  		    _ "C" // 引入另一个传输层插件库 C
	  		  )
	  		  
	  		  func main() {
	  		    resp, err := A.Get("http://google.com")
	  		    ...
	  		  }
	  ```
	  ``` bash
	  		  > go run main.go
	  		  panic: Dial is already registered
	  ```
	  当然，这种情况下 panic 属于预期行为，大部分的用户不会傻到一个库里面引入两个互相冲突的插件，但如果是间接引入呢？比如，有另一个依赖 D，它引入了插件 C：

	  ``` go
	  		  package main
	  		  
	  		  import (
	  		    "A"
	  		    _ "B" // 引入传输层插件库 B
	  		    "D"
	  		  )
	  		  
	  		  func main() {
	  		    resp, err := A.Get("http://google.com")
	  		    D.AssertNil(err)
	  		    ...
	  		  }
	  ```
	  这同样会导致运行时 panic，并且 panic 的栈调用是 init 函数的调用栈，必须查依赖图才可能找到是哪个库引入了插件 C。
	- ### 插件自冲突
	  另外插件也可能跟自己冲突，准确地说是不同版本之间的冲突。比如用户想把 B 升级到 B/v2 （B 和 B/v2 依赖了互相兼容的 A 版本），就得将所有依赖库的 B 都升级到 B/v2（可能花费巨大的工作量）。
- ## 实际案例分析
  有读者可能会说：这还不好解决，我就在注册逻辑里防止冲突，加一张表，每个插件用各自的名字加版本注册不就行了吗？比如这样写：
  ``` go
	  package A
	  
	  import "fmt"
	  
	  var DialPlugins = map[string]func(string) (io.ReadWriteCloser, error) {}
	  
	  func GetWith(address string, plugin string) (*Response, error) {
	    if dial, ok := DialPlugins[plugin]; !ok {
	      return fmt.Errorf("plugin %s is not registered", plugin)
	    }
	    ...
	  }
  ```
  ``` go
	  package B
	  
	  import (
	    "A"
	    "net"
	  )
	  
	  func init() {
	    if _, ok := DialPlugins["B/v1"]; ok {
	      panic("B" + " plugin is already registered")
	    }
	    A.DialPlugins["B/v1"] = func(address string) (io.ReadWriteCloser, error) {
	      return net.Dial("tcp", address )
	    }
	  }
  ```
  这个方案确实能解决上述插件系统的冲突问题，但会导致插件/版本迁移很麻烦，用户需要在每个调用的地方更改插件名字和版本。如果你是库作者，肯定是希望能平滑迁移的 —— 这就导致了一个显示的多版本冲突案例。
	- ### ginkgo 的迁移问题
	  如果你最近打算将你使用的 ginkgo 迁移到 ginkgo/v2，大概会遇到这样的运行时 panic：
	  ``` log
	  		  xxx.test flag redefined: ginkgo.seed
	  		  panic: xxx.test flag redefined: ginkgo.seed
	  		  
	  		  goroutine 1 [running]:
	  		  flag.(*FlagSet).Var(0xc000218120, 0x1df5fc8, 0x2351f80, 0xc000357640, 0xb, 0x1cd87a3, 0x2a)
	  		         /golang/1.16.8/go/src/flag/flag.go:871 +0x637
	  		  flag.(*FlagSet).Int64Var(...)
	  		         /golang/1.16.8/go/src/flag/flag.go:682
	  		  github.com/onsi/ginkgo/v2/types.bindFlagSet(0xc0003cc000, 0x20, 0x21, 0x1bbd540, 0xc0003b86f0, 0x232f420, 0xd, 0xd, 0x0, 0x0, ...)
	  		          /go/pkg/mod/github.com/onsi/ginkgo/v2@v2.0.0/types/flags.go:161 +0x15e5
	  		  github.com/onsi/ginkgo/v2/types.NewAttachedGinkgoFlagSet(...)
	  		          /go/pkg/mod/github.com/onsi/ginkgo/v2@v2.0.0/types/flags.go:113
	  		  github.com/onsi/ginkgo/v2/types.BuildTestSuiteFlagSet(0x2351f80, 0x23519a0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, ...)
	  		          /go/pkg/mod/github.com/onsi/ginkgo/v2@v2.0.0/types/config.go:346 +0x6e8
	  		  github.com/onsi/ginkgo/v2.init.0()
	  		          /github.com/onsi/ginkgo/v2@v2.0.0/core_dsl.go:47 +0x8f
	  		  ginkgo run failed
	  ```
	  根据 [ginkgo#875](https://github.com/onsi/ginkgo/issues/875) 的讨论，我们可以知道是因为 v1 和 v2 的 ginkgo 在 init 函数里定义了同名的 flag（显然是为了用户平滑迁移），从而导致了 flag 多次定义的 panic。

	  而解决方案就是将所有引入（import） ginkgo/v1 的依赖库（仅由 *_test.go 引入的除外）都升级到 ginkgo/v2。
	- ### ginkgo 暴露出 golang 的其它问题
	  我们都知道 ginkgo 是一个测试库，只应该被其它测试库引入，所以所有依赖库 ginkgo 版本升级的实际工作量并不大。但是，**go module 并没有区分 dependencies 和 test-dependencies**，这导致我需要额外工作来排查一个库是否在非测试文件中引入了它。

	  比如，`go mod graph` 告诉我 `k8s.io/apimachinery@v0.21.3`依赖了 `github.com/onsi/ginkgo@v1.11.0`，但它实际上并没有在非测试文件中引入 ginkgo/v1，所以我不需要升级 `k8s.io/apimachinery`。
	-
- ## 作为库作者如何避免类似的问题
	1. **尽量不要在 init 函数中造成外部副作用（即尽量只改变库内部的变量**
	2. **如果需要造成外部副作用，不要追求平滑升级大版本（即升级大版本时使用版本号作为副作用的 namespace）**
	3. **慎重在非测试文件中引入违反上述原则的库**

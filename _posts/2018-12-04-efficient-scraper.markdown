---
layout:     post
title:      Efficient Scraper
subtitle:   "高效地提取网页数据"
date:       2018-12-04 17:49:00
author:     "Hexi"
header-img: "img/bg/2018-12-04-efficient-scraper.jpg"
tags:
    - go
    - kotlin
    - rust
    - scraper
    - deserialization
    - HTML
---

前几个月打算开个爬虫的坑，然后开出了一坨新坑。

其实最早的坑很容易: 拿到 `HTML` -> 解析 `HTML`，但总觉得前人的做法过于原始，要糊一堆模板代码。于是我构思了一下，我只要能做到这两件事，就能高效地完成我的工作。

- 高效地拿到 `HTML`
- 高效地解析 `HTML`

目前来看，最接近我第一个需求的项目就是 [`retrofit`](https://github.com/square/retrofit)，而能满足第二个需求的好像并找不到。

找不到就只能自己造咯，由于最早不想写 `Java`，我两个月前造出了 `unhtml`（见[上一篇文章](https://hexilee.me/2018/10/01/unhtml/)）和 [`gotten`](https://github.com/Hexilee/gotten)。

`unhtml` 就不多介绍了，`gotten` 大概用法如下

```go
package example

import (
	"fmt"
	"github.com/Hexilee/gotten"
	"net/http"
	"time"
)

type (
	SimpleParams struct {
		Id   int `type:"path"`
		Page int `type:"query"`
	}

	Item struct {
		TypeId      int
		IId         int
		Name        string
		Description string
	}

	SimpleService struct {
		GetItems func(*SimpleParams) (gotten.Response, error) `method:"GET";path:"itemType/{id}"`
	}
)

var (
	creator, err = gotten.NewBuilder().
		SetBaseUrl("https://api.sample.com").
		AddCookie(&http.Cookie{Name: "clientcookieid", Value: "121", Expires: time.Now().Add(111 * time.Second)}).
		Build()

	simpleServiceImpl = new(SimpleService)
)

func init() {
	err := creator.Impl(simpleServiceImpl)
	if err != nil {
		panic(err)
	}
}

func InYourFunc() {
	resp, err := simpleServiceImpl.GetItems(&SimpleParams{1, 1})
	if err == nil && resp.StatusCode() == http.StatusOK {
		result := make([]*Item, 0) 
		err = resp.Unmarshal(&result)
		fmt.Printf("%#v\n", result)
	}
}
```

用起来还不错，我用它写了个练手项目 [`box-go-sdk`](https://github.com/QSCTech/box-sdk-go)。但总觉得 `API` 不是很优美而且有额外的运行时开销，于是我又糊出了 [`http-service`](https://github.com/rady-io/http-service)，大概这么用：

先安装 `go get github.com/rady-io/http-service`，然后确保它在 `PATH` 里再：

```go
package test

import (
	"io"
	"net/http"
	"time"
)

//go:generate http-service Service

/*
@HttpService
 */
type (
	/*
	@Base {scheme}://box.zjuqsc.com/item
	@Header(User-Agent) {userAgent}
	@Cookie(ga) {ga}
	@Cookie(qsc_session) secure_7y7y1n570y
	*/
	Service interface {
		/*
		@Get /get/{token}?page={page}&limit={limit}
		 */
		GetItem(token int, page int, limit int) (*http.Response, error)

		/*
		@Post /upload
		@Body multipart
		@Header(Content-Type) {contentType}
		@Cookie(ga) {cookie}
		@File(avatar) /var/log/{path}
		 */
		UploadItem(path string, contentType string, cookie string, video io.Reader) (*http.Response, error)

		/*
		@Put /change/{id}
		@Body json
		@Cookie(ga) {cookie}
		@Result json
		 */
		UpdateItem(id int, cookie string, data *time.Time, apiKey string) (result *UploadResult, statusCode int, err error)

		/*
		@Post /stat/{id}
		@SingleBody json
		 */
		StatItem(id int, body *StatBody) (*http.Response, error)

		/*
		@Post /stat/{id}
		@SingleBody json
		 */
		StatByReader(id int, body io.Reader) (*http.Response, error)

		/*
		@Post
		@Body form
		@Param(name) {firstName}.Lee
		 */
		PostInfo(id int, firstName string) (*http.Request, error)
	}
)

type UploadResult struct {
}

type StatBody struct {
}
```

再执行 `go generate`，然后同目录下就会出现 `service_impl.go` 文件，里面有个 `serviceImpl` 结构体和 `NewService` 方法，直接拿来用就好了。

看起来挺优雅的，我又糊了个练手项目 [`box-go-sdk-v2`](https://github.com/QSCTech/box-sdk-go/tree/master/v2)。

但毕竟 `go` 没有 `annotation` 更没有 `annotation processor`，解析注释也不过是旁门左道。

“还是去写 `Java` 吧” —— 我发出了真香的声音。

当然并不能就这样向 `Java` 屈服，我选择了写 `Kotlin`。然后我糊出了 [`unhtml.kt`](https://github.com/Hexilee/unhtml.kt) 。

这个专门给 `Kotlin data class` 用的 `HTML Deserializer`，由于比较屎我近期不打算维护/重写所以我没有给文档也没有上传到任何一个包仓库。这里给个例子，感兴趣的人可以去看 `test`

```kotlin
package test
import me.hexilee.unhtml.annotations.Selector
import me.hexilee.unhtml.annotations.Value
import me.hexilee.unhtml.Deserializer
import org.junit.Assert.*

fun simpleDataTest() {
  val user = Deserializer("""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <div id="test">
        <div>
            <p>Hexilee</p>
            <p>20</p>
            <p>true</p>
        </div>
    </div>
</body>
</html>""", "#test").new<SimpleUser>()
  assertEquals("Hexilee", user.name)
  assertEquals(20, user.age)
  assertTrue(user.likeLemon)
}
  
data class SimpleUser(
  @Selector("p:nth-child(1)")
  @Value
  val name: String,

  @Selector("p:nth-child(2)")
  @Value
  val age: Int,

  @Selector("p:nth-child(3)")
  @Value
  val likeLemon: Boolean
)

```

所以我说它屎到底屎在哪儿呢？

主要问题是泛型擦除，其次是基础类型的封装类没有实现同一个 `FromString` 之类的 `interface`，我得一个个特判并用相应的 `String.toXXX` 方法（这个问题在 `go` 里面更严重）。

第一个问题貌似可以用 `annotation processor 解决`，但我不是很喜欢那个接口而且 —— 既然要代码生成我为什么不写宏呢？

于是就有了 [`unhtml.rs`](https://github.com/Hexilee/unhtml.rs) ，这个项目用起来大概是这样：

```rust
#[macro_use]
extern crate unhtml_derive;
extern crate unhtml;
use unhtml::{self, FromHtml};

#[derive(FromHtml)]
#[html(selector = "#test")]
struct SingleUser {
    #[html(selector = "p:nth-child(1)", attr = "inner")]
    name: String,

    #[html(selector = "p:nth-child(2)", attr = "inner")]
    age: u8,

    #[html(selector = "p:nth-child(3)", attr = "inner")]
    like_lemon: bool,
}

let user = SingleUser::from_html(r#"<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <div id="test">
        <div>
            <p>Hexilee</p>
            <p>20</p>
            <p>true</p>
        </div>
    </div>
</body>
</html>"#).unwrap();
assert_eq!("Hexilee", &user.name);
assert_eq!(20, user.age);
assert!(user.like_lemon);
```

这个不管从结果和实现上来说都比较让我满意。主要是 `rust` 给所有的 `buildin type` 实现了 `FromStr`，并且支持包外 `impl trait`。过程宏现在也 stable 了，还有逐渐 stable 的 `nll` （不过貌似没人关心它是不是 stable），非常稳。

先说这么多吧，`rust` 的 `http client` 还有一坨坑等着踩呢......



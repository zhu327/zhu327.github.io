---
title: "像OpenResty一样使用Golang开发Web App"
date: 2018-06-26T21:35:01+08:00
tags: ["go"]
---

<https://github.com/zhu327/glualor>

最近在公司内网读过一篇Gopher Lua的文章, 感觉在Golang中使用Lua VM的模式跟OpenResty是一样一样的. 在Github上找了一圈net/http的到Gopher Lua的绑定, 然而并没有. 造轮子的机会来了 ^_^

### gluaweb

虽然看过几本Golang的书, 也读过几个开源项目的代码, 来腾讯后还上过两门Golang的课, 但是却没有写过一个Golang的项目, 从新读过net/http标准库的文档, 再看看Gopher Lua的例子就写了[gluaweb](https://github.com/zhu327/gluaweb)

<!--more-->

```go
package main

import (
	"github.com/yuin/gopher-lua"
	"github.com/zhu327/gluaweb"
	"net/http"
)

type helloHandler struct{}

func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	L := lua.NewState()
	defer L.Close()
	ctx := gluaweb.NewWebContext(w, r).WebContext(L)
	L.SetGlobal("gluaweb", ctx)
	if err := L.DoString(`

        gluaweb.Write("hello world!")

    `); err != nil {
		panic(err)
	}
}

func main() {
	http.Handle("/", &helloHandler{})
	http.ListenAndServe(":8080", nil)
}
```

使用很简单, 每个请求进来启动一个Lua VM, 然后注入gluaweb上下文, 在lua里面调用gluaweb绑定的Request或者ResponseWriter的方法即可.

到这一步基本上跟OpenResty其实就差不多了, OpenResty里Request与Response相关的方法都在ngx下, 而在gluaweb里gluaweb就相当于OpenResty的ngx.

### glualor

有了最基本的Request/Response的方法还不够, 还需要Web FrameWork, 就如同Gin之于net/http. 在OpenResty那边有一个[lor](https://github.com/sumory/lor)框架, 代码很简洁, 使用方式类似于Python的Flask. 同样是Lua写的, 只需要阅读代码, 把其中使用ngx相关的部分改写成gluaweb对应的方法即可. 包括request.lua respnse.lua cookie.lua.

lor还支持json编解码, 在OpenResty里使用的cjson, 在Github找到了[gopher-json](https://github.com/layeh/gopher-json), 在main.go里面创建Lua VM时加载json module, 并且修改lor里面utils.lua json相关的代码从cjson改为gogher-json.

除了json还有template, lor使用了lua-resty-template这个库, 在Gopher Lua这边可以直接使用Golang html/template, 同样去Github上面搜索找到一个text/template到Gopher Lua的绑定, 由于text/template提供的接口与html/template是一样的所以fork到[gluatemplate](https://github.com/zhu327/gluatemplate), 改import把text改为html. 再修改lor里面view.lua把lua-resty-template替换为gluatemplate.

main.go

```go
package main

import (
	"github.com/yuin/gopher-lua"
	"github.com/zhu327/gluatemplate"
	"github.com/zhu327/gluaweb"
	luajson "layeh.com/gopher-json"
	"net/http"
)

type helloHandler struct{}

func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	L := lua.NewState()
    luajson.Preload(L)
	L.PreloadModule("template", gluatemplate.Loader)
	defer L.Close()
	ctx := gluaweb.NewWebContext(w, r).WebContext(L)
	L.SetGlobal("gluaweb", ctx)
	if err := L.DoFile("main.lua"); err != nil { // 加载 lua 代码
		panic(err)
	}
}

func main() {
	http.Handle("/", &helloHandler{})
	http.ListenAndServe(":8080", nil)
}
```

main.lua

```lua
local lor = require("lor.index")

local app = lor()

app.get("/", function(err, req, res, next) {
	res:send("Hello world!")            
})

app.run()
```

经过以上改造, 已经能在Gopher Lua上愉快的使用lor快速开发web app了, 然后ab测试一下性能, 简直不忍直视, 继续改造.

每一个请求进来都新建一个Lua VM其实是没有必要的, 可以全局创建一个VM池, 每个请求进来从池里取出一个VM运行代码即可. Gopher Lua的README上也介绍了一个goroutine安全的pool实现直接拷贝过来用就好了. 除了VM复用还需要Lua代码的复用, 而不是每一次请求都Dofile加载一次所有的代码, 这样就需要把Lua中的某个函数暴露给Golang, 在Golang中调用已有的Lua函数来处理请求, 在lor里面就是app.run这个方法, 把app声明成全局变量就可以在Golang中调用app.run了.

再ab测一下, 性能相对于原生Golang的Hello world有大概17%的损失, 相对于开发的便利性, 还是可以接受的.

有了框架还要测试下到底能不能用, lor有一个官方的TODO示例, 直接用这个跑一下试试, 最终的完整实例就是[glualor](https://github.com/zhu327/glualor)

### 扩展

Web开发当然不只是Web Framework, OpenResty提供了全套开发库, MySQL Redis一应俱全, 但是Gopher Lua的生态还不完善, Github搜一圈并没有找到MySQL与Redis的绑定, 其实自己写也是很简单的, 有空再扩展一下^_^
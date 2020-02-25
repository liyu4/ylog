---
title: "[GO]web中间件(1)"
date: 2020-02-25T11:46:08+08:00
draft: true
---

web中间件常见的用途是一些公共方法的执行，譬如，获取用户的信息，压缩等。
在讲这些之前我们先实现一个go的web server。我们知道在go里面实现一个web服务器是相对容易的事情。
<!--more-->

下面是一个例子：

```
package main

import (
	"net/http"
)

func love(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("i love golang"))
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/love", love)
	http.ListenAndServe(":1314", mux)
}


```
在你的终端输入 curl http://:1314/love你会得到下面的输出， 或者在你的浏览器窗口输入http://127.0.0.1/love, 会输出 "i love golang"
[![iCbIuF.png](https://s1.ax1x.com/2018/09/07/iCbIuF.png)](https://imgchr.com/i/iCbIuF)

### Handler

Hanlder在net/http标准包中的定义，可以看到它时一个接口类型，里面包含了一个ServerHTTP的方法，任何实现该方法的类型，都可以被当作时Handler


```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// Handle registers the handler for the given pattern.
// If a handler already exists for pattern, Handle panics.
func (mux *ServeMux) Handle(pattern string, handler Handler) { // handler直接可以注册到mux里面
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	mux.m[pattern] = muxEntry{h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}

```

mux也叫做router，它的作用是，注册request path和它对应的handler，handler负责处理具体的逻辑，比如获取一个用户的个人信息，或者退出一个系统等。

基于此我们可以写出另外一种形式的mini http server,  下面是一个用自己实现的Handler类型，然后注册到mux，并且监听外部输入的例子:

```
package main

import (
	"net/http"
)

type love struct {
	lang string
}

func (l *love) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("i love " + l.lang))
}

func main() {
	mux := http.NewServeMux()
	l := &love{lang: "golang"}
	mux.Handle("/handler", l) // l是Hander类型
	http.ListenAndServe(":1314", mux)
}

```
在终端中输入curl http://:1314/handler, 输出如下。

![](https://ws1.sinaimg.cn/large/9b6074eegy1fv14y0kocrj20gh01sjre.jpg)


### 函数即类型 HandlerFunc
golang中函数也可以作为一种类型，HandlerFunc就是绝佳的实践，它实现了ServeHTTP方法，也就是说它属于上面我们说的Handler类型，那么也就是说假如我们有一个函数类似于：f(w, r)， 就可以利用类型之间的转换将其升级为(HandlerFunc(f)) HandlerFunc类型，然后其自动继承了ServeHTTP方法。

```
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

```

写下这个有趣的例子吧。

```
package main

import (
	"net/http"
)

func love(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("i love golang"))
}


func main() {
	mux := http.NewServeMux()
	l := http.HandlerFunc(love) // 类型变成HandlerFunc, 也可以认为是Handler类型
	http.Handle("/handlerfunc",l) // 所以我们这样注册
	http.ListenAndServe(":1314", mux)
}

其实main还这样写

func main() {
	mux := http.NewServeMux()
	http.HandlerFunc("/handlerfunc", love)  // net/http提供了HandlerFunc的直接注册
	http.ListenAndServe(":1314", mux)
}

```

为什么可以这样写呢？下面的代码可以解释, 其实它的内部实现了一个mux.Handle, 并且将我们实现的handler函数转换成Handler，然后完成注册工作。

```
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}

```

### DefaultServeMux
mux := http.NewServeMux()我们拿到了一个默认的mux, 它的结构如下，其中mu是一把读写互斥锁， m存储了请求路径对应的Handler。相信这个结构是很清晰明的。

```
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler
	pattern string
}

```

###扯了这么久，要说到middlerware怎么写了

假如我们需要在每一个请求的前面都输出一句"我喜欢和朱颖在一起"，我们该怎么做？

让我们来试一试吧。

```
package main

import (
	"log"
	"net/http"
)

// wantToBeWithYou是一个中间件函数
func wantToBeWithYou(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("zhuying want to be with you") // 中间业务在这里，可以设置ctx，鉴权等
		next.ServeHTTP(w, r)  // 这里本质上执行的下面的rain方法
	})
}

// 某一个具体的业务Handler
func rain(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("漕河泾正下着淅淅沥沥的秋雨"))
}

func main() {
	mux := http.NewServeMux()
	h := http.HandlerFunc(rain)
	mux.Handle("/middleware", wantToBeWithYou(h))
	http.ListenAndServe(":1314", mux)
}


输出如下
╰─$ go run test.go                                                                 1 ↵
2018/09/07 16:59:23 zhuying want to be with you
2018/09/07 16:59:23 漕河泾正下着淅淅沥沥的秋雨

```

如上我们实现了一个简单的中间件功能，我们成功的打印了log信息，并且执行了我们rain方法。一般而言我们会在wantToBeWithYou做一些更具体的事情，比如你可能想在ctx中设置一些value，甚至你想检查session是否合理，这些都可以在middleware中做。


未完待续！！！！！！

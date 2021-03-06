---
title: "[GO]web中间件(2)"
date: 2020-02-25T11:48:49+08:00
draft: true
---

书接上文

<!--more-->


##### 定义属于你的MyHandler接口, 任何实现其ServeHTTP方法的类型都可以当作是MyHandler。

```
type MyHandler interface {
	ServeHTTP(w http.ResponserWriter, r *http.Request, next http.HandlerFunc) 
}
next是下一个需要执行的http Handler，它作为一个参数被传递进去。
```

##### 仿照net/http定义一个函数类型MyHandlerFunc, 实现ServeHTTP方法。

```
type MyHandlerFunc func (w http.ResponseWriter, r *http.Request, next http.HandlerFunc)

func (h MyHandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	h(w, r, next) // 调用ServeHTTP就是执行具体的MyHandlerFunc, 需要手动执行next函数
}
```



##### 定义一个中间件middleware， middlerware的数据结构像是一个单向链表，可以包裹n个前置服务，也可以将需要执行的逻辑放在最后的位置。

```
type middleware struct {
	handler MyHandler // 当前Handler
	next middleware   // 下一个需要执行的middleware
}

// 实现ServeHTTP方法,也就是实现了http Handler.
func (m *middlerware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// m.next.ServeHTTP 自己调用自己，可以理解成进入下一个middleware
	m.handler.ServeHTTP(w, r, m.next.ServeHTTP) 
}
```

如上，midllerware就是一个Handler, 可在mux中注册，我们举个例子:

```
package main

import (
	"fmt"
	"net/http"
)

type MyHandler interface {
	ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc)
}

type MyHandlerFunc func(w http.ResponseWriter, r *http.Request, next http.HandlerFunc)

func (h MyHandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	h(w, r, next) // 调用ServerHTTP就是执行具体的MyHandlerFunc, 需要手动执行next函数
}

type middleware struct {
	handler MyHandler   // 当前Handler
	next    *middleware // 下一个需要执行的middleware
}

// 实现ServeHTTP方法,也就是实现了http Handler.
func (m middleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// m.next.ServeHTTP 自己调用自己，可以理解成进入下一个middleware
	m.handler.ServeHTTP(w, r, m.next.ServeHTTP)
}

func build(myHandlers []MyHandler) middleware {
	var next middleware

	if len(myHandlers) == 0 {
		return voidMiddleware()
	} else if len(myHandlers) > 1 {
		next = build(myHandlers[1:])
	} else {
		next = voidMiddleware()
	}

	return middleware{myHandlers[0], &next}
}

func voidMiddleware() middleware {
	return middleware{
		MyHandlerFunc(func(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc) {}),
		&middleware{},
	}
}

var myHandlerOne = MyHandlerFunc(func(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	fmt.Println("use myHandlerOne")
	next(w, r)
})

var myHandlerTwo = MyHandlerFunc(func(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	fmt.Println("use myHandlerTwo")
	next(w, r)
})

var loveFunc = func(w http.ResponseWriter, r *http.Request) {
	fmt.Println("you come you see")
	w.Write([]byte("i love golang"))
}

func get(handler http.HandlerFunc) MyHandlerFunc {
	return func(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
		handler.ServeHTTP(w, r) // http.HandlerFunc实际调用f(w, r)
		next(w, r)              // 后面执行的handler,当然你也写成next.ServeHTTP(w, r),这和next(w, r)本质是一样的
	}
}

func main() {
	myHandlers := []MyHandler{myHandlerOne, myHandlerTwo, get(loveFunc)}
	m := build(myHandlers)
	mux := http.NewServeMux()
	mux.Handle("/youqu", m)
	http.ListenAndServe(":1234", mux)
}

```

##### 看看效果
![](http://ww1.sinaimg.cn/large/9b6074eegy1fvpcqu6vafj21s40cq102.jpg)

#### 为什么要手动执行一下next(w, r)

显然我们看到myHandlerTwo，myHandlerTwo在函数的结尾处调用next(w, r), 其传参数为m.next.ServeHTTP,其本身为函数类型即：func, m.handler.ServeHTTP(w, r, m.next.ServeHTTP), 这里做了匿名转换，将其转换为http.HandlerFunc, 当然显示的转换是更加容易理解的， m.handler.ServeHTTP(w, r, http.HandlerFunc(m.next.ServeHTTP))。 当我们执行next(w, r)的时候其实执行就是m.next.ServeHTTP(w, r), 由此可见，中间件才能往下执行。

#### 总结

灵魂在于ServeHTTP函数和Handler接口，一个请求进来会调用ServeHTTP, 然后找到对应的handler, 对应下图的handle（w, req, ps）
![](http://ww1.sinaimg.cn/large/9b6074eegy1fvpcjt25d8j21dq0f2dlu.jpg) 
handle如何执行呢？ 他会调用Handler的ServeHTTP, 在我们的例子里就是middlerware的ServeHTTP.
![](http://ww1.sinaimg.cn/large/9b6074eegy1fvpcm4omj6j21a60cu79w.jpg)

大概中间件就是这样运行起来的, 如果你完全懂了，那就可以实战了。

完结。











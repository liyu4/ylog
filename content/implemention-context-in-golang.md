---
title: "[GO]Context的golang实现"
date: 2020-02-21T15:32:54+08:00
draft: true
---

context是golang里面的标准库，在go里面到处都可以找到使用ctx的例子，网上有很多context的科普文章
其中也有比较写的比较好的，这部分我会贴在文章末尾。今天我们主要是带着大家如何从0到1实现一个ctx，有了这种实践之后
具体在go源码中的使用场景分析，我会放在下一篇blog中，并且结合[golang数据库连接池实现](http://www.youmakemeday.com/parse-implementation-of-golang-database-sql-connection/)一起探究其中的奥秘，确保能一举深刻的理解ctx。
<!--more-->


context在源码中的[定义](https://golang.org/src/context/context.go?s=7884:7906#L61)
```
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

包含了四个方法
* Deadline, 最后截止时间
* Done, channel类型
* Err, 返回错误
* Value, 返回携带的k/v值

ctx总是从Background或者TODO开始，它们返回一个非nil的empty ctx，它没有value，永远不会取消
没有deadline，总是位于req.ctx的顶层。让我们来实现一个emptyCtx.

```
// emptyCtx不会被取消, 没有值, 没有deadline. 不能是struct{}，因为类型对应的值必须要用不同的地址
type emptyCtx int

// 实现Deadline，返回空
func (*emptyCtx) Deadline() (dealline time.Time, ok bool) {
	return
}

// 实现Done，返回空
func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

// 实现Err返回空
func (*emptyCtx) Err() error {
	return nil
}
// 实现Value，返回空
func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

* emptyCtx实现了Context接口
* emptyCtx的方法返回的都是“空值”
* emptyCtx构成了最基本的，context.Background/context.TODO

让我们实现Background和TODO

```
(
    var background = new(emptyCtx)
    var todo =       new(emptyCtx)
)


func Background() *emptyCtx {
    return background
}

func TODO() *empty {
    return todo
}
```

* Background就是空的context实现
* TODO也是空的context实现，在你实在不知道传什么ctx的时候，**传TODO是比nil更好的选择**


用过context的都知道，常用的几个方法是
* func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {//}
* func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {//}
* func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {//}
* func WithValue(parent Context, key, val interface{}) Context {//}


实战中我们也会去使用，类似于这样
```go
// 设置ctx，超时时间5秒
// 作为参数开启tx
// 如果数据库操作五秒没有执行完成
// 将返回错误
ctx, cancel := context.WithTimeout(ctx, time.Second*5)
	defer cancel()
	tx, err := _sqlServer.BeginTx(ctx, nil)
	if err != nil {
		return
	}
```

可见，实际使用的过程中是不需要关心底层是怎么实现的，接下来我们实现一个WithCancel
```
type cancelCtx struct {
	Context               // 继承parent
	done    chan struct{} // done
	err     error         // error
}
```
* 定义cancelCtx结构体
* 包含了Context,从上层的parent继承方法
* 自定义done，实现Done()
* 自定义err，实现Err()

如下代码段所示：
```
func (ctx *cancelCtx) Done() chan struct{} { return ctx.done }

func (ctx *cancelCtx) Err() error { return ctx.err }
```

定义cancel函数类型,和cancelctx取消的错误

```
type CancelFunc func()

var Canceled = errors.New("context canceled")
```


对应用层暴露的WithCancel, 输入参数是parent，可以认为是上层传下来的
返回一个新的ctx和cancel函数方法
```
func WithCancel(parent Context) (Context, CancelFunc) {
	ctx := &cancelCtx{
		Context: parent,
		done:    make(chan struct{}),
	}

	cancel := func() { ctx.cancel(Canceled）}

	go func() {
		select {
		case <-parent.Done():
			// 上层取消
			ctx.cancel(parent.Err())
		case <-ctx.Done():
			// 下层取消
		}
	}()

	return ctx, cancel
}

func (ctx *cancelCtx) cancel(err error) {
	if ctx.err != nil {
		return
	}
	ctx.err = err
	close(ctx.done)
}

```

WithDeadline的实现，继承了cancelCtx，还加了deadline类型
初始化的时候需要带上时间，调用链超时之后会报 context deadline
```
// 定义deadlineCtx
type deadlineCtx struct {
	*cancelCtx
	deadline time.Time
}

// 返回截止日期
func (ctx *deadlineCtx) Deadline() (deadline time.Time, ok bool) {
	return ctx.deadline, true
}

var DeadlineExceeded = deadlineExceededErr{}

type deadlineExceededErr struct{}

func (deadlineExceededErr) Error() string   { return "deadline exceeded " }
func (deadlineExceededErr) Timeout() bool   { return true }
func (deadlineExceededErr) Temporary() bool { return true }

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
	cctx, cancel := WithCancel(parent)

	ctx := &deadlineCtx{
		cancelCtx: cctx.(*cancelCtx),
		deadline:  deadline,
	}

	// the line below has been replaced to work with version previous to go 1.8.
	// timeout := time.Until(deadline)
	timeout := deadline.Sub(time.Now())
	t := time.AfterFunc(timeout, func() {
		ctx.cancel(DeadlineExceeded)
	})

	stop := func() {
		t.Stop()
		cancel()
	}

	return ctx, stop
}

```

WithTimeout,是超过了预设的时间之后，就会报错，它只需要对WithDeadline进行封装即可。
```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```


这是对ctx的一个基本实现，当前go版本的实现比这个稍微多一点，但是代码量依然很少

golang context是非常优秀的实现，从中可以获益良多。

https://www.flysnow.org/2017/05/12/go-in-action-go-context.html
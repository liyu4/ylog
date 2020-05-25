---
title: "[GO]Context在golang数据库连接池中的实践"
date: 2020-02-24T14:56:50+08:00
draft: true
---

我们都知道使用ctx可以控制goroutine，比如取消goroutine树，或者给ctx设置一个超时，以便整个ctx树可以控制在预期的时间内执行完，不管【成功/失败】，另外context也内置了deadline属性，开发者可以方便的控制自己开启的goroutine情况 。
<!--more-->


# transaction and ctx
golang在database/sql中的连接池和ctx深度结合，为了理解在日常后端开发中如何使用ctx这种特性帮助大家更好的开发。举一个常见的例子：
有一个需求，
1: 如果一个事物执行的时间超过了5s就返回超时错误。
2: 回滚事物，释放资源。

网络世界总是极端复杂的，网络抖动，数据库block，机房掉电等等，所以当超过预期时间了没有返回结果，程序可以自主的做一些事情，那么带有上下文的事物至少有以下两个 好处
* 前台用户不需要等待更久的时间【正常应该是1s执行完的程序，如果5s都没有返回】
* 一个执行不会一直占住一个连接（connection） 【数据库的连接是有限的】


我们看以下上面说的，具体的代码示例
```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	fmt.Print("start\n")
	// connect to mysql
	db, err := sql.Open("mysql", "root:@tcp(127.0.0.1:3306)/test")
	if err != nil {
		panic(err)
    }
	ctx, _ := context.WithTimeout(context.Background(), time.Second*5)
	//start transaction with context
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		fmt.Printf("start transaction failed: [%v]", err)
	}
	defer tx.Rollback()
	insertStatement1 := `insert into foo(id, name, age) values(?,?,?)`
	_, err = tx.Exec(ctx, insertStatement1, 2, "foo", 20)
	if err != nil {
		fmt.Printf("exec insert statement 1 failed: [%v]", err)
		return
	}
	// ctx will time out
	time.Sleep(5 * time.Second)
	insertStatement2 := `insert into bar(id, foo_id) values(?,?)`
	_, err = tx.Exec(ctx, insertStatement2, 2, 2)
	if err != nil {
		// output: sql: transaction has already been committed or rolled back
		fmt.Printf("exec insert statement 2 failed: [%v]", err)
		return
	}
	err = tx.Commit()
	if err != nil {
		fmt.Printf("tx commit failed: [%v]", err)
	}
	time.Sleep(2 * time.Second)
	fmt.Print("end\n")
}

```

*  连接mysql，默认的database为db
*  新增一个ctx，从Background开始，带有5s的超时，五秒之后ctx.Done会关闭，ctx.err = `context deadline exceeded`
*  `BeginTx(ctx, nil)`, tx bind ctx
*  无论如何都执行rollback，确保资源释放
*  执行一条sql语句
*  强制sleep五秒
*  再次执行一条sql，这个时候会报错，因为ctx超时了。
*  错误信息 `sql: transaction has already been committed or rolled back`



那么问题就是，为什么会发生这样的错误，这样的错误不符合我们的预期，因为不是ctx超时后的错误，这个我们后面再讲为什么会这样。

首先`BeginTx(...)`注入了ctx，注入的ctx做了如下的工作。

```go
// 继承begintx传递进来的ctx
ctx, cancel := context.WithCancel(ctx)
tx = &Tx{
	db:          db,
	dc:          dc,
	releaseConn: release,
	txi:         txi,
	cancel:      cancel,
    ctx:         ctx,
}

// 开启携程block住
go tx.awaitDone()

// 当ctx close done之后
// 执行rollback
func (tx *Tx) awaitDone() {
	<-tx.ctx.Done()
	tx.rollback(true)
}

// 执行rollback， 设置tx.Done为1
// 也就是意味着，这个tx被“销毁”了
func (tx *Tx) rollback(discardConn bool) error {
	if !atomic.CompareAndSwapInt32(&tx.done, 0, 1) {
		return ErrTxDone
    }
    // ......
}
```

* `BeginTx`开启了了带有ctx的tx，然后监控【go awaitDone()】ctx是否done 
* 如果`ctx.Done`, 那么则回滚tx，释放资源，设置`tx.Done = 1`


那么获得了`tx.Done = 1`有什么意义呢？ 之前我们出错的地方是exec的地方。让我们看下这段代码发生了什么。

```go
func (tx *Tx) grabConn(ctx context.Context) (*driverConn, releaseConn, error) {
	select {
    default:
    // 如果ctx不是background那么
    // ctx，cancel之后再这里返回错误
    // 也就是说不会去创建新的或者占用连接
	case <-ctx.Done():
		return nil, nil, ctx.Err()
	}
    tx.closemu.RLock()
    // 如果ctx是emptyctx的话
    // 那么由于tx.Done设置为1
    // 然后返回ErrTxDone
	if tx.isDone() {
		tx.closemu.RUnlock()
		return nil, nil, ErrTxDone
	}
	if hookTxGrabConn != nil { 
		hookTxGrabConn()
	}
	return tx.dc, tx.closemuRUnlockRelease, nil
}
```

* 首先尝grab一个connection
* 然后检查ctx的状态
* 分为设置了ctx和默认的ctx
* 如果是设置了ctx那么返回ctx的err
* 如果是没有设置ctx那么返回database/sql的自定义错误

# 事务/exce/query都带上ctx
让我们来看下，如果执行ExecContext的话，就会返回ctx的错误，比如在上面的case中我们设置的是ctx.WithTimeout(parent, duration), 翻看context.go源码可以看到内置的错误是`context deadline exceeded` 。


```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// connect to mysql
	db, err := sql.Open("mysql", "root:@tcp(127.0.0.1:3306)/test")
	if err != nil {
		panic(err)
	}
	ctx, _ := context.WithTimeout(context.Background(), time.Second*5)
	//start transaction with context
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		fmt.Printf("start transaction failed: [%v]", err)
	}
	defer tx.Rollback()
	insertStatement1 := `insert into foo(id, name, age) values(?,?,?)`
	_, err = tx.ExecContext(ctx, insertStatement1, 2, "foo", 20)
	if err != nil {
		fmt.Printf("exec insert statement 1 failed: [%v]", err)
		return
	}
	// ctx will time out
	time.Sleep(5 * time.Second)
	insertStatement2 := `insert into bar(id, foo_id) values(?,?)`
	_, err = tx.ExecContext(ctx, insertStatement2, 2, 2)
	if err != nil {
		// output: context deadline exceeded
		fmt.Printf("exec insert statement 2 failed: [%v]", err)
		return
	}
	err = tx.Commit()
	if err != nil {
		fmt.Printf("tx commit failed: [%v]", err)
	}
	time.Sleep(2 * time.Second)
}
```
针对开始的代码只是替换了`Exec()-->ExecContext()`。后者的第一个参数时候ctx，在上面我们设置的ctx为`WithTimeout()` 超时之后输出的错误`context deadline exceeded`， 对照源码来讲，就是直接执行了`select{}`，对比来说就是少执行了几行代码，然后输出的错误也有些不同。
```go
	select {
    default:
	case <-ctx.Done():
		return nil, nil, ctx.Err()
	}
```
* <-ctx.Done()执行，输出ctx.Err()
* ctx.Err() = `context deadline exceeded`
* 如果是Exec方法则输出`sql: transaction has already been committed or rolled back`



# 总结
* 如果是数据库tx操作，建议带上ctx参数，设置超时之后，可以主动控制程序的资源。
* 真正执行的函数，建议按照xxxContext()的方式执行,这样父节点ctx取消之后，所有子节点的ctx都能感应到。
* 就算不是事务执行，如果对于时间有限制的话，也建议带上ctx，比如长查询。
* ctx通过WithTimeout/WithDeadline等copy parent之后获得了child ctx，内部会一直检查父ctx是否有效，如果无效，则取消所有的子节点，并且close ctx.done， 设置ctx.err。
* 内部包，如果使用了ctx，自身需要监控ctx.Done状态。


# Link
* https://gist.github.com/liyu4/e199578085cdff5d29fcb36b04e08b2b
* https://gist.github.com/liyu4/83a5a8aa671db9df46f356a1239fcfb1
---
title: "golang数据库连接池的实现[SQLServer]"
date: 2020-02-17T17:11:44+08:00
draft: true
---
Golang的database/sql包定义了常用的操作数据的方法，他提供了一个抽象，具体的driver依赖不同的数据库。
比如mysql驱动比较有名的[MySql](https://github.com/go-sql-driver/mysql "")。
还有Sql Server使用较为广泛的库是 [SqlServer](https://github.com/denisenkom/go-mssqldb "")。
使用者只需要提供DSN(data source name)或者tcp连接, open() db之后会返回一个*sql.DB对象。DB本身没有连接数据库，
只有当Query/Exec之后才会连接数据库。
<!--more-->



假设需要从数据库中查询数据
```go
package main

import(
    "database/sql"
    _ "github.com/denisenkom/go-mssqldb"
)

func main(){
    db, err := sql.Connect("mssql", dataSourceName)
	if err != nil {
		panic(fmt.Sprintf("[db.GetSqlServer] open sql fail:%s", err.Error()))
	}

    if err != nil {
        Println(err)
        return
    }
    defer db.Close()

    rows,err := db.Query("select top 1 foo, bar from test_table")

    var foo bar string
    for rows.Next(){
        _,err := row.Scan(&foo, &bar)
        if  err  !=  nil {
            // do something
        }
        break
    }
    // ...
}

```

---


引入了两个包，一个是`database/sql`, 另外一个是`_ "github.com/denisenkom/go-mssqldb"`为什么需要引入这两个包？
我们知道在Golang中包前面有下划线意味着只引入其中的`init()`方法, 那么go-mssqldb的init方法做了些什么

```go
// 第三方包定义的driver实体
var driverInstance = &Driver{processQueryText: true}
var driverInstanceNoProcess = &Driver{processQueryText: false}

func init() {
    // 注册到sql中，稍后我们来看，注册一个结构体进去有什么用
	sql.Register("mssql", driverInstance)
    sql.Register("sqlserver", driverInstanceNoProcess)
    // 初始化一个函数， 用作发起网络调用
	createDialer = func(p *connectParams) Dialer {
		return netDialer{&net.Dialer{KeepAlive: p.keepAlive}}
	}
}

```

如上的注册一个mssql驱动到sql中封装的非常简洁，我们进去看看里面做了什么

---



```go

func Register(name string, driver driver.Driver) {
	driversMu.Lock()
	defer driversMu.Unlock()
	if driver == nil {
		panic("sql: Register driver is nil")
	}
	if _, dup := drivers[name]; dup {
		panic("sql: Register called twice for driver " + name)
	}
	drivers[name] = driver
}

```

`sql.Register(xxx)`函数的实现如下，有两个输入参数, 第一个是需要注册的数据库name，另外一个是实现了Driver的具体类型。

---


```go
type Driver interface {
	Open(name string) (Conn, error)
}

type DriverContext interface {
	OpenConnector(name string) (Connector, error)
}

```

`database/sql`定义了一个interface `Driver`, `DriverContext`，分别定义了方法 `Open()`和`OpenConnector()`



```go
type Driver struct {
	log optionalLogger
	processQueryText bool
}
// 实现了DriverContext
func (d *Driver) OpenConnector(dsn string) (*Connector, error) {
	params, err := parseConnectParams(dsn)
	if err != nil {
		return nil, err
	}
	return &Connector{
		params: params,
		driver: d,
	}, nil
}
// 实现了Driver
func (d *Driver) Open(dsn string) (driver.Conn, error) {
	return d.open(context.Background(), dsn)
}
```

`driverInstance`和`driverInstanceNoProcess`我们只需要关注其中一个即可。 那么在go-mssqldb中
`driverInstance`对应了什么？ 可以看到源码如上:

---


```go
func Open(driverName, dataSourceName string) (*DB, error) {
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
    }
    // 拿到带有ctx的DB，这里其实兼容了之前的版本
	if driverCtx, ok := driveri.(driver.DriverContext); ok {
		connector, err := driverCtx.OpenConnector(dataSourceName)
		if err != nil {
			return nil, err
		}
		return OpenDB(connector), nil
    }
    // 拿到driver版本的DB
	return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}
```

databse/sql拿到了go-mssqldb提供的driver之后做了什么？

看代码可知，拿到了一个`*DB`实体，这个实体的driver部分由第三方提供，对于go-mssqldb来讲，具体的说是提供了driver以及里面包含的Connector


---

```go
// database/sql实现了通用的exec
func (db *DB) Exec(query string, args ...interface{}) (Result, error) {
	return db.ExecContext(context.Background(), query, args...)
}

// 进到db.ExecContext
func (db *DB) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error) {
	var res Result
    var err error
    // 重试
	for i := 0; i < maxBadConnRetries; i++ {
        // 执行方法db.exec
		res, err = db.exec(ctx, query, args, cachedOrNewConn)
		if err != driver.ErrBadConn {
			break
		}
    }
    // 重试
	if err == driver.ErrBadConn {
		return db.exec(ctx, query, args, alwaysNewConn)
	}
	return res, err
}

// 显然最重要的方法是db.exec
// 进到db.exec

func (db *DB) exec(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (Result, error) {
    // 根据策略获取数据库连接
    // 注意 ！！！这个时候才真正的去获取conn
    dc, err := db.conn(ctx, strategy)
	if err != nil {
		return nil, err
    }
    // 执行query语句 select top 1 foo, bar from test_table
	return db.execDC(ctx, dc, dc.releaseConn, query, args)
}

```


日常开发过程当中常用的两类操作是Query/Exec, 就拿文章开头的`db.Query("select top 1 foo, bar from test_table")`来分析整个流程

我们的期望是获得foo和bar对应coloum上的value。

---

![image](/img/golang连接池过程分析.jpg)

通过业务场景的带入 ，我们了解了一个查询最终会走到如上的`db.conn()`和`db.exexDC()`，

那么它们经历就是上图所示的流程图逻辑。仔细看这张图，如果整个连接池为空的时候，必然需要新创建conn,


```go
type DB struct {
    connector driver.Connector
    // ignore ...
}
```

这是database/sql中`*DB`的定义，重点是看connector，通过connector发起`Connect()`, 才能创建有效的conn

---


```go
type Driver struct {
	log optionalLogger
	processQueryText bool
}
// 实现了DriverContext
func (d *Driver) OpenConnector(dsn string) (*Connector, error) {
    // 忽略具体代码
}
// 实现了Driver
func (d *Driver) Open(dsn string) (driver.Conn, error) {
	return d.open(context.Background(), dsn)
}
```

这里就稍微有一点不同了，从代码中可以看出, 第三方提供的driver有两个，并且返回的参数都不一致
一个直接提供了connector，另外一个支持提供有效的conn

---


```go
func Open(driverName, dataSourceName string) (*DB, error) {
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}

	if driverCtx, ok := driveri.(driver.DriverContext); ok {
        // 支持提供了connector
		connector, err := driverCtx.OpenConnector(dataSourceName)
		if err != nil {
			return nil, err
		}
		return OpenDB(connector), nil
	}
    // 通过dsnConnector提供connector
	return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}
```

注册到`*DB`实体中的connector是不同的，drivercontext直接提供了Connector，并且其实现了和数据库通信的Connect()方法

但是Driver并没有提供Connector，这是因为在database/sql中包了一层，也就是dsnConnector

这里的差距我认为是代码迭代过程中的兼容所导致的

---


```go
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
    // ...
    ci, err := db.connector.Connect(ctx)
    // ...
}
```

可以看见具体的`Connect()`数据库的方法都是通过注入到database/sql的connector实现的

具体到driverctx和driver显然是不同的

---


```go
// Connect to the server and return a TDS connection.
func (c *Connector) Connect(ctx context.Context) (driver.Conn, error) {
	conn, err := c.driver.connect(ctx, c, c.params)
	if err == nil {
		err = conn.ResetSession(ctx)
	}
	return conn, err
}
```
driverCtx直接可以通过`Connect(ctx context.Context)`获取有效的conn，

---


```go
type dsnConnector struct {
	dsn    string
	driver driver.Driver
}

func (t dsnConnector) Connect(_ context.Context) (driver.Conn, error) {
	return t.driver.Open(t.dsn)
}
```

driver如何获取有效的conn呢，那就要从dsnConnector来讲了，转换一下 `db.connector.Connect() == db.dsnConnector.Connect()`
可以看到最终调用了t.driver.Open(t.dsn)， 也就是说又回到了第三方提供的 driver上（注意不是driverctx），
从database/sql的driver定义可知道，第三方driver（go-mssqldb的注册的driver），必须实现了`Open()`，事实上也确认实现了这个方法。
也就是:

---



```go

// t.driver.Open(t.dsn)发起
func (d *Driver) Open(dsn string) (driver.Conn, error) {
	return d.open(context.Background(), dsn)
}

// 分析数据库参数， 执行连接程序，获取 conn
func (d *Driver) open(ctx context.Context, dsn string) (*Conn, error) {
	params, err := parseConnectParams(dsn)
	if err != nil {
		return nil, err
	}
	return d.connect(ctx, nil, params)
}
```


终于得到了一个conn， 并且database/sql维护了一个conn的连接池，前文中提到的db.execDC, 就负责将参数注入到cmder（是我生造的词）中，

cmder解析成sql server协议定义的字节流，go-mssqldb负责send/receiver， database/sql负责生成数据，比如foo， bar

---


```go
 // 执行query语句 select top 1 foo, bar from test_table
	return db.execDC(ctx, dc, dc.releaseConn, query, args)
```


综上，就是如何使用第三方库进行查询的流程，写入是一样的原理，这里不赘述，如果你看到了这里，你肯定会有一些自己的看法 。

我觉得database/sql提供了通用的功能，比如维护连接池，执行query/exec，注册，数据转换
第三库，其实也可以叫做package，实现database/sql定义的Driver, 然后Register,就轻松的连接到了sql上

细节上的协议解析，发起连接（长连接），都依靠自身来解决

这就像是OSI网络模型， 又分层的感觉在里面， 任何一层有改动只需要动其中一层即可

但是不能说这样是没有问题的，比如driver和driverctx显然是兼容之后的结果，导致阅读代码的时候需要来回去看

总体而言golang database/sql的实现是非常有借鉴和学习的意义.
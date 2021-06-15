+++
title = "database/sql 源码学习"
date = "2019-10-17"
slug = "2019/10/17/golang-database-sql"
Categories = ["Golang"]
+++

### 数据结构

#### ColumnType

contains the name and type of a column. 用于表示数据库表字段的基本信息

#### Conn

represents a single database connection rather than a pool of database connections. 只代表一个数据库连接，适用于只能对一个连接操作的场景，比如开启事务。

```go
  type Conn struct {
      db *DB

      // closemu prevents the connection from closing while there
      // is an active query. It is held for read during queries
      // and exclusively during close.
      closemu sync.RWMutex

      // dc is owned until close, at which point
      // it's returned to the connection pool.
      dc *driverConn

      // done transitions from 0 to 1 exactly once, on close.
      // Once done, all operations fail with ErrConnDone.
      // Use atomic operations on value when checking value.
      done int32
  }
```

#### DB

a database handle representing a pool of zero or more underlying connections. It's safe for concurrent use by multiple goroutines. 代表一个数据库连接池

The sql package creates and frees connections automatically; it also maintains a free pool of idle connections. 连接池由包自动管理

Once DB.Begin is called, the returned Tx is bound to a single connection.Once Commit or Rollback is called on the transaction, that transaction's connection is returned to DB's idle connection pool. 如果是事务的话，在事务提交或回滚前始终用的是一个连接

```go
  type DB struct {
      // Atomic access only. At top of struct to prevent mis-alignment
      // on 32-bit platforms. Of type time.Duration.
      waitDuration int64 // Total time waited for new connections.

      connector driver.Connector
      // numClosed is an atomic counter which represents a total number of
      // closed connections. Stmt.openStmt checks it before cleaning closed
      // connections in Stmt.css.
      numClosed uint64

      mu           sync.Mutex // protects following fields
      freeConn     []*driverConn
      connRequests map[uint64]chan connRequest
      nextRequest  uint64 // Next key to use in connRequests.
      numOpen      int    // number of opened and pending open connections
      // Used to signal the need for new connections
      // a goroutine running connectionOpener() reads on this chan and
      // maybeOpenNewConnections sends on the chan (one send per needed connection)
      // It is closed during db.Close(). The close tells the connectionOpener
      // goroutine to exit.
      openerCh          chan struct{}
      resetterCh        chan *driverConn
      closed            bool
      dep               map[finalCloser]depSet
      lastPut           map[*driverConn]string // stacktrace of last conn's put; debug only
      maxIdle           int                    // zero means defaultMaxIdleConns; negative means 0
      maxOpen           int                    // <= 0 means unlimited
      maxLifetime       time.Duration          // maximum amount of time a connection may be reused
      cleanerCh         chan struct{}
      waitCount         int64 // Total number of connections waited for.
      maxIdleClosed     int64 // Total number of connections closed due to idle.
      maxLifetimeClosed int64 // Total number of connections closed due to max free limit.

      stop func() // stop cancels the connection opener and the session resetter.
  }
```

#### driverConn

wraps a driver.Conn with a mutex, to be held during all calls into the Conn 封装了 driver.Conn

```go
  type driverConn struct {
      db        *DB
      createdAt time.Time

      sync.Mutex  // guards following
      ci          driver.Conn
      closed      bool
      finalClosed bool // ci.Close has been called
      openStmt    map[*driverStmt]bool
      lastErr     error // lastError captures the result of the session resetter.

      // guarded by db.mu
      inUse      bool
      onPut      []func() // code (with db.mu held) run when conn is next returned
      dbmuClosed bool     // same as closed, but guarded by db.mu, for removeClosedStmtLocked
  }
```

#### Rows

the result of a query. Its cursor starts before the first row of the result set. Use Next to advance from row to row. 代表查询结果集

```go
  type Rows struct {
      dc          *driverConn // owned; must call releaseConn when closed to release
      releaseConn func(error)
      rowsi       driver.Rows
      cancel      func()      // called when Rows is closed, may be nil.
      closeStmt   *driverStmt // if non-nil, statement to Close on close

      // closemu prevents Rows from closing while there
      // is an active streaming result. It is held for read during non-close operations
      // and exclusively during close.
      //
      // closemu guards lasterr and closed.
      closemu sync.RWMutex
      closed  bool
      lasterr error // non-nil only if closed is true

      // lastcols is only used in Scan, Next, and NextResultSet which are expected
      // not to be called concurrently.
      lastcols []driver.Value
  }
```

总结一下：

DB对象代表了一个连接池，它来维护这个池子的生长

driverConn代表了一个具体的数据库连接，每一条sql的执行最终都会落实到一个具体的driverConn

一次Query查询会返回一个Rows对象，代表的是一个查询集合，里面是一行一行的记录

其它公开的数据结构（Stmt、Tx）都是构建于上面说的结构之上的


### 接口

`database/sql`包里没有对任何一个数据库协议的具体实现，它只是用一堆接口描述了任何一个想要接入该包的数据库驱动需要实现的函数

#### driver.Conn

a connection to a database. It is not used concurrently by multiple goroutines.

```go
  type Conn interface {
      // Prepare returns a prepared statement, bound to this connection.
      Prepare(query string) (Stmt, error)

      // Close invalidates and potentially stops any current
      // prepared statements and transactions, marking this
      // connection as no longer in use.
      //
      // Because the sql package maintains a free pool of
      // connections and only calls Close when there's a surplus of
      // idle connections, it shouldn't be necessary for drivers to
      // do their own connection caching.
      Close() error

      // Begin starts and returns a new transaction.
      //
      // Deprecated: Drivers should implement ConnBeginTx instead (or additionally).
      Begin() (Tx, error)
  }
```

注意，Conn 接口要求必须实现Prepare方法

### 流程分析

从官方文档的一个例子开始

```go
package main

import (
	"context"
	"database/sql"
	"flag"
	"log"
	"os"
	"os/signal"
	"time"
)

var pool *sql.DB // Database connection pool.

func main() {
	id := flag.Int64("id", 0, "person ID to find")
	dsn := flag.String("dsn", os.Getenv("DSN"), "connection data source name")
	flag.Parse()

	if len(*dsn) == 0 {
		log.Fatal("missing dsn flag")
	}
	if *id == 0 {
		log.Fatal("missing person ID")
	}
	var err error

	// Opening a driver typically will not attempt to connect to the database.
	pool, err = sql.Open("driver-name", *dsn)
	if err != nil {
		// This will not be a connection error, but a DSN parse error or
		// another initialization error.
		log.Fatal("unable to use data source name", err)
	}
	defer pool.Close()

	pool.SetConnMaxLifetime(0)
	pool.SetMaxIdleConns(3)
	pool.SetMaxOpenConns(3)

	ctx, stop := context.WithCancel(context.Background())
	defer stop()

	appSignal := make(chan os.Signal, 3)
	signal.Notify(appSignal, os.Interrupt)

	go func() {
		select {
		case <-appSignal:
			stop()
		}
	}()

	Ping(ctx)

	Query(ctx, *id)
}

// Ping the database to verify DSN provided by the user is valid and the
// server accessible. If the ping fails exit the program with an error.
func Ping(ctx context.Context) {
	ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
	defer cancel()

	if err := pool.PingContext(ctx); err != nil {
		log.Fatalf("unable to connect to database: %v", err)
	}
}

// Query the database for the information requested and prints the results.
// If the query fails exit the program with an error.
func Query(ctx context.Context, id int64) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	var name string
	err := pool.QueryRowContext(ctx, "select p.name from people as p where p.id = :id;", sql.Named("id", id)).Scan(&name)
	if err != nil {
		log.Fatal("unable to execute search query", err)
	}
	log.Println("name=", name)
}
```

首先用`Open`生成了一个数据库连接池对象，我们看看它是怎么做的

```go
  var (
       // 一把读写锁
      driversMu sync.RWMutex
      // 全局的驱动映射map
      drivers   = make(map[string]driver.Driver)
  )
  
    func Open(driverName, dataSourceName string) (*DB, error) {
      driversMu.RLock()
      driveri, ok := drivers[driverName]
      driversMu.RUnlock()
      if !ok {
          return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
      }

      if driverCtx, ok := driveri.(driver.DriverContext); ok {
          connector, err := driverCtx.OpenConnector(dataSourceName)
          if err != nil {
              return nil, err
          }
          return OpenDB(connector), nil
      }

      return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
  }
  
    func OpenDB(c driver.Connector) *DB {
      ctx, cancel := context.WithCancel(context.Background())
      db := &DB{
          connector:    c,
          openerCh:     make(chan struct{}, connectionRequestQueueSize),
          resetterCh:   make(chan *driverConn, 50),
          lastPut:      make(map[*driverConn]string),
          connRequests: make(map[uint64]chan connRequest),
          stop:         cancel,
      }
      // 注意这里起了两个goroutine
      
      // 上面的DB的结构体的openerCh字段的注释里说明了
      // 这个goroutine的处理逻辑
      go db.connectionOpener(ctx)
      go db.connectionResetter(ctx)

      return db
  }
```

那么一次查询是如何进行的呢？

DB.Query 最终会调用到 Conn.Query

```go
  // QueryContext executes a query that returns rows, typically a SELECT.
  // The args are for any placeholder parameters in the query.
  func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error) {
      var rows *Rows
      var err error
      for i := 0; i < maxBadConnRetries; i++ {
          rows, err = db.query(ctx, query, args, cachedOrNewConn)
          if err != driver.ErrBadConn {
              break
          }
      }
      if err == driver.ErrBadConn {
          return db.query(ctx, query, args, alwaysNewConn)
      }
      return rows, err
  }

  // Query executes a query that returns rows, typically a SELECT.
  // The args are for any placeholder parameters in the query.
  func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
      return db.QueryContext(context.Background(), query, args...)
  }
  
    func (db *DB) query(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (*Rows, error) {
      // 获取一个driverConn
      dc, err := db.conn(ctx, strategy)
      if err != nil {
          return nil, err
      }

      return db.queryDC(ctx, nil, dc, dc.releaseConn, query, args)
  }


  // QueryContext executes a query that returns rows, typically a SELECT.
  // The args are for any placeholder parameters in the query.
  func (c *Conn) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error) {
      dc, release, err := c.grabConn(ctx)
      if err != nil {
          return nil, err
      }
      return c.db.queryDC(ctx, nil, dc, release, query, args)
  }
```

先获取一个connection，这个过程我们暂且不看，继续往下追

从queryDC开始进入驱动的代码

```go
  // queryDC executes a query on the given connection.
  // The connection gets released by the releaseConn function.
  // The ctx context is from a query method and the txctx context is from an
  // optional transaction context.
  func (db *DB) queryDC(ctx, txctx context.Context, dc *driverConn, releaseConn func(error), query string, args []interface{}) (*Rows, error) {
      // 判断驱动的连接对象是否实现了Queryer接口
      queryerCtx, ok := dc.ci.(driver.QueryerContext)
      var queryer driver.Queryer
      if !ok {
          queryer, ok = dc.ci.(driver.Queryer)
      }
      if ok {
          var nvdargs []driver.NamedValue
          var rowsi driver.Rows
          var err error
          withLock(dc, func() {
              nvdargs, err = driverArgsConnLocked(dc.ci, nil, args)
              if err != nil {
                  return
              }
              rowsi, err = ctxDriverQuery(ctx, queryerCtx, queryer, query, nvdargs)
          })
          if err != driver.ErrSkip {
          // 说明用户没有要求用Prepared Statement
              if err != nil {
                  releaseConn(err)
                  return nil, err
              }
              // Note: ownership of dc passes to the *Rows, to be freed
              // with releaseConn.
              // 注意：这里将驱动返回的行信息rowsi在这里封装成了
              // sql.Rows
              rows := &Rows{
                  dc:          dc,
                  releaseConn: releaseConn,
                  rowsi:       rowsi,
              }
              rows.initContextClose(ctx, txctx)
              return rows, nil
          }
      }
      // 如果驱动没有实现Queryer接口或者用户指定要使用Prepared Statement(返回了driver.ErrSkip），则使用Prepared Statement来做查询
      
      var si driver.Stmt
      var err error
      withLock(dc, func() {
          // prepare
          si, err = ctxDriverPrepare(ctx, dc.ci, query)
      })
      if err != nil {
          releaseConn(err)
          return nil, err
      }

      ds := &driverStmt{Locker: dc, si: si}
      // 执行查询
      rowsi, err := rowsiFromStatement(ctx, dc.ci, ds, args...)
      if err != nil {
          ds.Close()
          releaseConn(err)
          return nil, err
      }

      // Note: ownership of ci passes to the *Rows, to be freed
      // with releaseConn.
      rows := &Rows{
          dc:          dc,
          releaseConn: releaseConn,
          rowsi:       rowsi,
          closeStmt:   ds,
      }
      rows.initContextClose(ctx, txctx)
      return rows, nil
  }
```

根据驱动或者用户配置不同，以上代码会有不同的执行路径，假设最终走直接查询的路径（即不走Prepared Statement），则会继续执行如下逻辑：

```go
// 执行到ctxDriverQuery函数

func ctxDriverQuery(ctx context.Context, queryerCtx driver.QueryerContext, queryer driver.Queryer, query string, nvdargs []driver.NamedValue) (driver.Rows, error) {
    if queryerCtx != nil {
        return queryerCtx.QueryContext(ctx, query, nvdargs)
    }
    dargs, err := namedValueToValue(nvdargs)
    if err != nil {
        return nil, err
    }
    
    // 注意这里ctx的用法，根据ctx是否已被取消，提早返回 
    select {
    default:
    case <-ctx.Done():
        return nil, ctx.Err()
    }
    // 从这里以后就会到驱动的代码了
    return queryer.Query(query, dargs)
}
```
如果使用的是mysql的驱动，会继续执行以下代码：

```go
  func (mc *mysqlConn) Query(query string, args []driver.Value) (driver.Rows, error) {
      return mc.query(query, args)
  }
  
    func (mc *mysqlConn) query(query string, args []driver.Value) (*textRows, error) {
      if mc.closed.IsSet() {
          errLog.Print(ErrInvalidConn)
          return nil, driver.ErrBadConn
      }
      if len(args) != 0 {
          // 是否转义sql中的参数来防止sql注入
          // 如果指定不转义的话会使用 prepared statement
          if !mc.cfg.InterpolateParams {
              return nil, driver.ErrSkip
          }
          // try client-side prepare to reduce roundtrip
          prepared, err := mc.interpolateParams(query, args)
          if err != nil {
              return nil, err
          }
          query = prepared
      }
      // Send command
      err := mc.writeCommandPacketStr(comQuery, query)
      if err == nil {
          // Read Result
          var resLen int
          resLen, err = mc.readResultSetHeaderPacket()
          if err == nil {
              rows := new(textRows)
              rows.mc = mc

              if resLen == 0 {
                  rows.rs.done = true

                  switch err := rows.NextResultSet(); err {
                  case nil, io.EOF:
                      return rows, nil
                  default:
                      return nil, err
                  }
              }

              // Columns
              rows.rs.columns, err = mc.readColumns(resLen)
              return rows, err
          }
      }
      return nil, mc.markBadConn(err)
  }
```
上面只是读出了“表头”，拿到了row对象，下面要开始读取具体的记录了（对应一开始的例子中的Scan语句）

对于只返回一行的query直接调用Scan就可以了(其实，在暗地里帮你调用了Next)，返回多行时需要先调用Next，然后再调用Scan

```go
  func (rs *Rows) Next() bool {
      var doClose, ok bool
      withLock(rs.closemu.RLocker(), func() {
          doClose, ok = rs.nextLocked()
      })
      if doClose {
          rs.Close()
      }
      return ok
  }
  
  func (rs *Rows) nextLocked() (doClose, ok bool) {
      if rs.closed {
          return false, false
      }

      // Lock the driver connection before calling the driver interface
      // rowsi to prevent a Tx from rolling back the connection at the same time.
      rs.dc.Lock()
      defer rs.dc.Unlock()

      if rs.lastcols == nil {
          rs.lastcols = make([]driver.Value, len(rs.rowsi.Columns()))
      }
      // 从此进入驱动的代码
      rs.lasterr = rs.rowsi.Next(rs.lastcols)
      if rs.lasterr != nil {
          // Close the connection if there is a driver error.
          if rs.lasterr != io.EOF {
              return true, false
          }
          nextResultSet, ok := rs.rowsi.(driver.RowsNextResultSet)
          if !ok {
              return true, false
          }
          // The driver is at the end of the current result set.
          // Test to see if there is another result set after the current one.
          // Only close Rows if there is no further result sets to read.
          if !nextResultSet.HasNextResultSet() {
              doClose = true
          }
          return doClose, false
      }
      return false, true
  }
  
  // mysql驱动的代码，位于rows.go中
  func (rows *textRows) Next(dest []driver.Value) error {
    if mc := rows.mc; mc != nil {
        if err := mc.error(); err != nil {
            return err
        }

        // Fetch next row from stream
        // 这里会读取MySQL的查询结果
        // 想了解MySQL的client与server间的交互协议的可以看这个函数
        return rows.readRow(dest)
    }
    return io.EOF
}
```

执行Next之后，会将从mysql读到的行数据保存在Rows对象的一个私有字段里(lastcols)，然后终于来到了Scan阶段

```go
  func (rs *Rows) Scan(dest ...interface{}) error {
      rs.closemu.RLock()

      if rs.lasterr != nil && rs.lasterr != io.EOF {
          rs.closemu.RUnlock()
          return rs.lasterr
      }
      if rs.closed {
          err := rs.lasterrOrErrLocked(errRowsClosed)
          rs.closemu.RUnlock()
          return err
      }
      rs.closemu.RUnlock()

      if rs.lastcols == nil {
          return errors.New("sql: Scan called without calling Next")
      }
      if len(dest) != len(rs.lastcols) {
          return fmt.Errorf("sql: expected %d destination arguments in Scan, not %d", len(rs.lastcols), len(dest))
      }
      // 遍历这一行的每一个字段
      for i, sv := range rs.lastcols {
          // 转成Golang内部类型
          // 对于数据库里的某个字段类型会被转成Golang里哪种类型
          // 有疑问的可以看这个函数的实现
          err := convertAssignRows(dest[i], sv, rs)
          if err != nil {
              return fmt.Errorf(`sql: Scan error on column index %d, name %q: %v`, i, rs.rowsi.Columns()[i], err)
          }
      }
      return nil
  }
```

至此，一次完整的查询过程就分析完了。

下面想具体了解下`database/sql`包是如何维护数据库的连接池的。

> 有兴趣的可以看一下Goang 从1.0到现在的1.12，`database/sql`包是如何演化的，如果你去看1.1的源码会发现那时候的连接池实现很简陋，那会儿的连接池没有最大连接数的限制

这个连接池有几个特征参数：

- maxLifetime
    - maximum amount of time a connection may be reused
- maxIdle
    - 允许的最大空闲连接数
- maxOpen
    - 允许的打开的最大连接数

连接池中的连接数最大会增长到maxOpen，当请求高峰过后，连接池的可用连接数又会慢慢缩回到maxIdle

```go
  func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
      return db.QueryContext(context.Background(), query, args...)
  }

  func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error) {
      var rows *Rows
      var err error
      // 优先用池子里的连接
      for i := 0; i < maxBadConnRetries; i++ {
          rows, err = db.query(ctx, query, args, cachedOrNewConn)
          if err != driver.ErrBadConn {
              break
          }
      }
      // 当然池子里的连接有可能会过期，如果重试两次还取不到可以用的连接
      // 那就从数据库请求一个新的
      if err == driver.ErrBadConn {
          return db.query(ctx, query, args, alwaysNewConn)
      }
      return rows, err
  }
  
    func (db *DB) query(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (*Rows, error) {
      dc, err := db.conn(ctx, strategy)
      if err != nil {
          return nil, err
      }

      return db.queryDC(ctx, nil, dc, dc.releaseConn, query, args)
  }
```

下面就是获取连接的逻辑了，流程如下：

- 如果策略是优先使用缓存连接且连接池中还有空闲连接，则直接从连接池中取一个连接返回
- 如果连接池里当前已打开的连接数已超出maxOpen限制，则阻塞一直等待有连接归还到连接池后再取用
- 否则就新建一个连接返回

```go
  // conn returns a newly-opened or cached *driverConn.
  func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
      db.mu.Lock()
      if db.closed {
          db.mu.Unlock()
          return nil, errDBClosed
      }
      // 检查是否已被取消
      // Check if the context is expired.
      select {
      default:
      case <-ctx.Done():
          db.mu.Unlock()
          return nil, ctx.Err()
      }
      // 一个连接的最大生命周期
      lifetime := db.maxLifetime

      // Prefer a free connection, if possible.
      numFree := len(db.freeConn)
      if strategy == cachedOrNewConn && numFree > 0 {
          conn := db.freeConn[0]
          // copy(dst, src []Type) int
          // The copy built-in function copies elements from a source slice into a destination slice
          // The source and destination may overlap.
          // 这是在更新空闲的数据库连接数组，最开始的那个连接现在已经要被占用了
          copy(db.freeConn, db.freeConn[1:])
          // 移除掉末尾那个无用的元素
          db.freeConn = db.freeConn[:numFree-1]
          // 通过以上两步，成功从freeConn中删掉了开头的第一个连接
          conn.inUse = true
          db.mu.Unlock()
          // 检查连接是否已经过期
          // 注意如果过期并没有尝试取下一个，而是直接返回了错误
          // 由上层来继续发起连接，我理解这样会保持不同层之间逻辑的干净
          if conn.expired(lifetime) {
              conn.Close()
              return nil, driver.ErrBadConn
          }
          // Lock around reading lastErr to ensure the session resetter finished.
          // 这里应该是看下这个连接是不是被session resetter给置为badConn了
          // 如果是，那也不能用了
          // session resetter 是一个单独的goroutine在检查每个连接的状态
          conn.Lock()
          err := conn.lastErr
                    conn.Unlock()
          if err == driver.ErrBadConn {
              conn.Close()
              return nil, driver.ErrBadConn
          }
          return conn, nil
      }

      // Out of free connections or we were asked not to use one. If we're not
      // allowed to open any more connections, make a request and wait.
      // 如果当前打开的连接数已经超过了db.maxOpen
      if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
          // Make the connRequest channel. It's buffered so that the
          // connectionOpener doesn't block while waiting for the req to be read.
          req := make(chan connRequest, 1)
          reqKey := db.nextRequestKeyLocked()
          db.connRequests[reqKey] = req
          db.waitCount++
          db.mu.Unlock()

          waitStart := time.Now()

          // Timeout the connection request with the context.
          select {
          // 注意这个select没有default分支，所以会一直阻塞在这两个分支上
          // 直到任何一个分支的条件满足，即：1. query被取消了 2. 有free连接可用了
          case <-ctx.Done():
              // Remove the connection request and ensure no value has been sent
              // on it after removing.
              db.mu.Lock()
              delete(db.connRequests, reqKey)
              db.mu.Unlock()

              atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

              select {
              default:
              case ret, ok := <-req:
                  if ok && ret.conn != nil {
                      db.putConn(ret.conn, ret.err, false)
                  }
              }
              return nil, ctx.Err()
          case ret, ok := <-req:
              atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

              if !ok {
                  return nil, errDBClosed
              }
              if ret.err == nil && ret.conn.expired(lifetime) {
                  ret.conn.Close()
                  return nil, driver.ErrBadConn
              }
              if ret.conn == nil {
                  return nil, ret.err
              }
              // Lock around reading lastErr to ensure the session resetter finished.
              ret.conn.Lock()
              err := ret.conn.lastErr
              ret.conn.Unlock()
              if err == driver.ErrBadConn {
                  ret.conn.Close()
                  return nil, driver.ErrBadConn
              }
              return ret.conn, ret.err
          }
      }
      // 如果没有空闲连接且尚未达到最大连接数的限制
      // 那就开一个新连接
      db.numOpen++ // optimistically
      db.mu.Unlock()
      ci, err := db.connector.Connect(ctx)
      if err != nil {
          db.mu.Lock()
          db.numOpen-- // correct for earlier optimism
          db.maybeOpenNewConnections()
          db.mu.Unlock()
          return nil, err
      }
      db.mu.Lock()
      dc := &driverConn{
          db:        db,
          createdAt: nowFunc(),
          ci:        ci,
          inUse:     true,
      }
      db.addDepLocked(dc, dc)
      db.mu.Unlock()
      return dc, nil
  }
```

有借就有还，再借才不难，下面是归还连接的逻辑：

- 如果有connRequests，那么优先满足它
- 否则就要准备归还到池子中了，但这里还有一个check，如果当前空闲连接书freeConn大于允许的最大空闲连接限制的话就不归还了，会把连接直接关闭掉

```go
  // putConn adds a connection to the db's free pool.
  // err is optionally the last error that occurred on this connection.
  func (db *DB) putConn(dc *driverConn, err error, resetSession bool) {
      db.mu.Lock()
      if !dc.inUse {
          if debugGetPut {
              fmt.Printf("putConn(%v) DUPLICATE was: %s\n\nPREVIOUS was: %s", dc, stack(), db.lastPut[dc])
          }
          panic("sql: connection returned that was never out")
      }
      if debugGetPut {
          db.lastPut[dc] = stack()
      }
      dc.inUse = false

      for _, fn := range dc.onPut {
          fn()
      }
      dc.onPut = nil

      if err == driver.ErrBadConn {
          // Don't reuse bad connections.
          // Since the conn is considered bad and is being discarded, treat it
          // as closed. Don't decrement the open count here, finalClose will
          // take care of that.
          db.maybeOpenNewConnections()
          db.mu.Unlock()
          dc.Close()
          return
      }
      if putConnHook != nil {
          putConnHook(db, dc)
      }
      if db.closed {
          // Connections do not need to be reset if they will be closed.
          // Prevents writing to resetterCh after the DB has closed.
          resetSession = false
      }
      if resetSession {
          if _, resetSession = dc.ci.(driver.SessionResetter); resetSession {
              // Lock the driverConn here so it isn't released until
              // the connection is reset.
              // The lock must be taken before the connection is put into
              // the pool to prevent it from being taken out before it is reset.
              dc.Lock()
          }
      }
      added := db.putConnDBLocked(dc, nil)
      db.mu.Unlock()

      if !added {
          if resetSession {
              dc.Unlock()
          }
          dc.Close()
          return
      }
      if !resetSession {
          return
      }
      select {
      default:
          // If the resetterCh is blocking then mark the connection
          // as bad and continue on.
          dc.lastErr = driver.ErrBadConn
          dc.Unlock()
      case db.resetterCh <- dc:
      }
  }
  
    func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
      if db.closed {
          return false
      }
      if db.maxOpen > 0 && db.numOpen > db.maxOpen {
          return false
      }
      // 如果有connRequests（连接需求），优先满足它
      if c := len(db.connRequests); c > 0 {
          var req chan connRequest
          var reqKey uint64
          for reqKey, req = range db.connRequests {
              break
          }
          delete(db.connRequests, reqKey) // Remove from pending requests.
          if err == nil {
              dc.inUse = true
          }
          req <- connRequest{
              conn: dc,
              err:  err,
          }
          return true
      } else if err == nil && !db.closed {
          if db.maxIdleConnsLocked() > len(db.freeConn) {
              // 将连接放回池子
              db.freeConn = append(db.freeConn, dc)
              db.startCleanerLocked()
              return true
          }
          // 为啥直接增加了maxIdleClosed而没有做什么呢？
          // 因为这种情况下本函数会返回false，进而导致
          // 连接被关闭
          db.maxIdleClosed++
      }
      return false
  }
```

以下逻辑实现连接的生命周期的管理，基本逻辑为：

定期遍历连接池中的每一个连接，检查其是存活时间是否超出了设定的maxLifetime，若超出则将它从池子中删除并关闭它

```go
  // startCleanerLocked starts connectionCleaner if needed.
  func (db *DB) startCleanerLocked() {
      // 尽管每次putConnDBLocked的时候都会被调用，但注意它的条件db.cleanerCh == nil
      // 这意味着其实db.connectionCleaner只会被启动一次
      if db.maxLifetime > 0 && db.numOpen > 0 && db.cleanerCh == nil {
          db.cleanerCh = make(chan struct{}, 1)
          go db.connectionCleaner(db.maxLifetime)
      }
  }
   // 清理过期连接的逻辑
   func (db *DB) connectionCleaner(d time.Duration) {
      const minInterval = time.Second
      // 精度最高为秒级
      if d < minInterval {
          d = minInterval
      }
      t := time.NewTimer(d)

      for {
          // 这个写法很有意思
          select {
          case <-t.C:
          case <-db.cleanerCh: // maxLifetime was changed or db was closed.
          }

          db.mu.Lock()
          d = db.maxLifetime
          if db.closed || db.numOpen == 0 || d <= 0 {
              db.cleanerCh = nil
              db.mu.Unlock()
              return
          }

          expiredSince := nowFunc().Add(-d)
          var closing []*driverConn
          for i := 0; i < len(db.freeConn); i++ {
              c := db.freeConn[i]
              if c.createdAt.Before(expiredSince) {
                  closing = append(closing, c)
                  last := len(db.freeConn) - 1
                  // 数组中经典的删除元素的方法
                  db.freeConn[i] = db.freeConn[last]
                  db.freeConn[last] = nil
                  db.freeConn = db.freeConn[:last]
                  // 后退一步是为了可以检查刚刚置换过来的元素
                  i--
              }
          }
          db.maxLifetimeClosed += int64(len(closing))
          db.mu.Unlock()

          for _, c := range closing {
              c.Close()
          }

          if d < minInterval {
              d = minInterval
          }
          t.Reset(d)
        }
  }
```

### Prepared Statement

根据MySQL[文档](https://dev.mysql.com/doc/refman/5.7/en/sql-syntax-prepared-statements.html)，prepared statement 只能绑定在一个连接（session）上，即如果在一个连接上执行了prepare后，后续的exec还是要在同一个连接上才行，以下引自MySQL文档：

> A prepared statement is specific to the session in which it was created. If you terminate a session without deallocating a previously prepared statement, the server deallocates it automatically.


但在Golang的`database/sql`包中，连接并没有直接暴露给开发者（从1.9以后暴露了一个单独的连接对象Conn），开发者看到的是一个连接池，那么`database/sql`是如何在只暴露连接池的情况下来支持prepared statement的呢？

是这么做的：

1. Prepare的阶段从连接池中取出一个连接来发起prepare的操作，执行完后就把连接放回连接池了，但此时要记住是在这个连接上发起了Prepare
2. 在execute阶段，会从连接池中需要当时发起prepare的那个连接，找到后再继续发起execute操作即可。但这里就有个问题，这时候可能找不到当时那个连接了（要么这个连接正在被别人占用，要么这个连接已经被关闭了），那么此时就需要从池子中再取一个连接出来，重新发起prepare。
    - 这里有个坑：在高并发的情况下，会在服务端创建大量重复的prepared statement，可能会耗尽服务端的最大prepared statement限制

参考：[Using Prepared Statements](http://go-database-sql.org/prepared.html)

下面从代码上具体分析一下：

首先是数据结构：

```go
  // Stmt is a prepared statement.
  // A Stmt is safe for concurrent use by multiple goroutines.
  //
  // If a Stmt is prepared on a Tx or Conn, it will be bound to a single
  // underlying connection forever. If the Tx or Conn closes, the Stmt will
  // become unusable and all operations will return an error.
  // If a Stmt is prepared on a DB, it will remain usable for the lifetime of the
  // DB. When the Stmt needs to execute on a new underlying connection, it will
  // prepare itself on the new connection automatically.
  type Stmt struct {
      // Immutable:
      db        *DB    // where we came from
      query     string // that created the Stmt
      stickyErr error  // if non-nil, this error is returned for all operations

      closemu sync.RWMutex // held exclusively during close, for read otherwise.

      // If Stmt is prepared on a Tx or Conn then cg is present and will
      // only ever grab a connection from cg.
      // If cg is nil then the Stmt must grab an arbitrary connection
      // from db and determine if it must prepare the stmt again by
      // inspecting css.
      cg   stmtConnGrabber
      cgds *driverStmt

      // parentStmt is set when a transaction-specific statement
      // is requested from an identical statement prepared on the same
      // conn. parentStmt is used to track the dependency of this statement
      // on its originating ("parent") statement so that parentStmt may
      // be closed by the user without them having to know whether or not
      // any transactions are still using it.
      parentStmt *Stmt

      mu     sync.Mutex // protects the rest of the fields
      closed bool

      // css is a list of underlying driver statement interfaces
      // that are valid on particular connections. This is only
      // used if cg == nil and one is found that has idle
      // connections. If cg != nil, cgds is always used.
      css []connStmt

      // lastNumClosed is copied from db.numClosed when Stmt is created
      // without tx and closed connections in css are removed.
      lastNumClosed uint64
  }
```

prepare阶段，分别可以从DB、Conn和Tx上发起Prepare，先分析在DB上的Prepare

```go
  func (db *DB) PrepareContext(ctx context.Context, query string) (*Stmt, error) {
      var stmt *Stmt
      var err error
      for i := 0; i < maxBadConnRetries; i++ {
          stmt, err = db.prepare(ctx, query, cachedOrNewConn)
          if err != driver.ErrBadConn {
              break
          }
      }
      if err == driver.ErrBadConn {
          return db.prepare(ctx, query, alwaysNewConn)
      }
      return stmt, err
  }
  
    func (db *DB) prepare(ctx context.Context, query string, strategy connReuseStrategy) (*Stmt, error) {
      // TODO: check if db.driver supports an optional
      // driver.Preparer interface and call that instead, if so,
      // otherwise we make a prepared statement that's bound
      // to a connection, and to execute this prepared statement
      // we either need to use this connection (if it's free), else
      // get a new connection + re-prepare + execute on that one.
      // 先从池子中获取一个连接
      dc, err := db.conn(ctx, strategy)
      if err != nil {
          return nil, err
      }
      return db.prepareDC(ctx, dc, dc.releaseConn, nil, query)
  }
  
    func (db *DB) prepareDC(ctx context.Context, dc *driverConn, release func(error), cg stmtConnGrabber, query string) (*Stmt, error) {
      var ds *driverStmt
      var err error
      defer func() {
          release(err)
      }()
      // 这里是调用驱动来向server发起Prepare命令
      withLock(dc, func() {
          ds, err = dc.prepareLocked(ctx, cg, query)
      })
      if err != nil {
          return nil, err
      }
      stmt := &Stmt{
          db:    db,
          query: query,
          cg:    cg,
          cgds:  ds,
      }

      // When cg == nil this statement will need to keep track of various
      // connections they are prepared on and record the stmt dependency on
      // the DB.
      // 记住这个stmt是在哪个conn上Prepare的，便于后续exec时再继续用
      if cg == nil {
          stmt.css = []connStmt{{dc, ds}}
          stmt.lastNumClosed = atomic.LoadUint64(&db.numClosed)
          db.addDep(stmt, stmt)
      }
      return stmt, nil
  }
```

执行阶段

```go
  func (s *Stmt) ExecContext(ctx context.Context, args ...interface{}) (Result, error) {
      s.closemu.RLock()
      defer s.closemu.RUnlock()

      var res Result
      strategy := cachedOrNewConn
      for i := 0; i < maxBadConnRetries+1; i++ {
          if i == maxBadConnRetries {
              strategy = alwaysNewConn
          }
          // 这里是尝试取回当时进行Prepare的连接以待继续操作
          dc, releaseConn, ds, err := s.connStmt(ctx, strategy)
          if err != nil {
              if err == driver.ErrBadConn {
                  continue
              }
              return nil, err
          }
          // 从驱动读取执行结果
          res, err = resultFromStatement(ctx, dc.ci, ds, args...)
          releaseConn(err)
          if err != driver.ErrBadConn {
              return res, err
          }
      }
      return nil, driver.ErrBadConn
  }
  
    func (s *Stmt) connStmt(ctx context.Context, strategy connReuseStrategy) (dc *driverConn, releaseConn func(error), ds *driverStmt, err error) {
      if err = s.stickyErr; err != nil {
          return
      }
      s.mu.Lock()
      if s.closed {
          s.mu.Unlock()
          err = errors.New("sql: statement is closed")
          return
      }

      // In a transaction or connection, we always use the connection that the
      // stmt was created on.
      // 在Tx或者Conn上Prepare的情况
      if s.cg != nil {
          s.mu.Unlock()
          dc, releaseConn, err = s.cg.grabConn(ctx) // blocks, waiting for the connection.
          if err != nil {
              return
          }
          return dc, releaseConn, s.cgds, nil
      }

      s.removeClosedStmtLocked()
      s.mu.Unlock()
      // 以下是在DB上Prepare时的处理逻辑
      // 简单粗暴，直接从池子里取一个连接，
      // 然后判断这个连接是不是之前prepare过，
      // 若prepare过，则直接返回这个连接即可，
      // 若没有，则直接在这个连接上重新prepare后再返回
      dc, err = s.db.conn(ctx, strategy)
      if err != nil {
          return nil, nil, nil, err
      }

      s.mu.Lock()
      for _, v := range s.css {
          if v.dc == dc {
              s.mu.Unlock()
              return dc, dc.releaseConn, v.ds, nil
          }
      }
      s.mu.Unlock()
      
      // No luck; we need to prepare the statement on this connection
      withLock(dc, func() {
          ds, err = s.prepareOnConnLocked(ctx, dc)
      })
      if err != nil {
          dc.releaseConn(err)
          return nil, nil, nil, err
      }

      return dc, dc.releaseConn, ds, nil
  }
```

这里有个小插曲，在对照源码的各个版本的`database/sql`包的实现时，偶然发现connStmt方法的实现经历了不少的改变呢，想知道这改变背后的原因，于是对提交历史考据了一番，还真找到了：

相关的commit: 1b61a97811626d4c7a8332c107f1e091253d1b2e

对应的github issue: [https://github.com/golang/go/issues/9484](https://github.com/golang/go/issues/9484)

一开始的实现，相对精细，导致锁冲突有点严重，后来又改成简单粗暴了，同时也减少了锁冲突，提高了性能，但高并发时会有很多重新Prepare的情况。

关闭阶段

```go
  // Close closes the statement.
  func (s *Stmt) Close() error {
      s.closemu.Lock()
      defer s.closemu.Unlock()

      if s.stickyErr != nil {
          return s.stickyErr
      }
      s.mu.Lock()
      if s.closed {
          s.mu.Unlock()
          return nil
      }
      s.closed = true
      txds := s.cgds
      s.cgds = nil

      s.mu.Unlock()

      if s.cg == nil {
          return s.db.removeDep(s, s)
      }

      if s.parentStmt != nil {
          // If parentStmt is set, we must not close s.txds since it's stored
          // in the css array of the parentStmt.
          return s.db.removeDep(s.parentStmt, s)
      }
      // txds是驱动端的stmt，最终调了驱动端的Close来关闭
      return txds.Close()
  }
```

参考：

- [Prepared剖析](https://www.jianshu.com/p/ee0d2e7bef54)
- [Prepared SQL Statement ](https://dev.mysql.com/doc/refman/5.7/en/sql-syntax-prepared-statements.html)
- [Using Prepared Statements](http://go-database-sql.org/prepared.html)

### Tx

下面分析下对数据库事务的支持。同prepared statement一样，在数据库层面事务也只能在一个连接上进行，但在底层实现上，Tx的实现与prepared statement有明显不同，它底层从始至终只使用了一个连接。不过，我不太明白prepared statement为啥没有这样实现。

开启事务

```go
  func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error) {
      var tx *Tx
      var err error
      for i := 0; i < maxBadConnRetries; i++ {
          tx, err = db.begin(ctx, opts, cachedOrNewConn)
          if err != driver.ErrBadConn {
              break
          }
      }
      if err == driver.ErrBadConn {
          return db.begin(ctx, opts, alwaysNewConn)
      }
      return tx, err
  }
  
    func (db *DB) begin(ctx context.Context, opts *TxOptions, strategy connReuseStrategy) (tx *Tx, err error) {
      // 依然是先从池子中取出一个空闲连接
      dc, err := db.conn(ctx, strategy)
      if err != nil {
          return nil, err
      }
      return db.beginDC(ctx, dc, dc.releaseConn, opts)
  }
  
    func (db *DB) beginDC(ctx context.Context, dc *driverConn, release func(error), opts *TxOptions) (tx *Tx, err error) {
      var txi driver.Tx
      withLock(dc, func() {
          // 调用driver来开启事务
          txi, err = ctxDriverBegin(ctx, opts, dc.ci)
      })
      if err != nil {
          release(err)
          return nil, err
      }

      // Schedule the transaction to rollback when the context is cancelled.
      // The cancel function in Tx will be called after done is set to true.
      ctx, cancel := context.WithCancel(ctx)
      // 注意返回的tx对象记了启动事务的连接dc
      // 后续所有的操作都会在这一个dc上进行
      tx = &Tx{
          db:          db,
          dc:          dc,
          releaseConn: release,
          txi:         txi,
          cancel:      cancel,
          ctx:         ctx,
      }
      go tx.awaitDone()
      return tx, nil
  }
```

执行阶段，exec和query过程类似，下面只分析exec的过程

```go
  func (tx *Tx) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error) {
      // 取出之前保存在tx上的连接dc
      dc, release, err := tx.grabConn(ctx)
      if err != nil {
          return nil, err
      }
      // 其实就是db.queryDC的逻辑
      // 现在知道为什么tx上要记一个db对象了
      return tx.db.queryDC(ctx, tx.ctx, dc, release, query, args)
  }
  
    func (tx *Tx) grabConn(ctx context.Context) (*driverConn, releaseConn, error) {
      select {
      default:
      case <-ctx.Done():
          return nil, nil, ctx.Err()
      }

      // closeme.RLock must come before the check for isDone to prevent the Tx from
      // closing while a query is executing.
      // 这个读写锁的目的是保证当tx被关闭时已经没有任何其它的查询在进行了
      // 注意到一般每一个资源对象都有这样一个锁，比如Row、Stmt
      tx.closemu.RLock()
      if tx.isDone() {
          tx.closemu.RUnlock()
          return nil, nil, ErrTxDone
      }
      if hookTxGrabConn != nil { // test hook
          hookTxGrabConn()
      }
      // 返回的是tx上记的那个dc
      return tx.dc, tx.closemuRUnlockRelease, nil
  }
```

### 参考

- [借 Go 语言 database/sql 包谈数据库驱动和连接池设计](https://chenjiayang.me/2019/08/10/go-sql-database-pool/)
- [golang之database/sql与go-sql-driver](http://luodw.cc/2016/08/28/golang02/)

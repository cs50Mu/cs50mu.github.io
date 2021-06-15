+++
title = "database with sqlx in Golang"
date = "2019-05-09"
slug = "2019/05/09/database-with-sqlx-in-golang"
Categories = ["Golang", "database"]
+++

## database with sqlx in Golang

### Parameterized Queries

翻译为参数化查询？是个什么概念呢？

Bind parameters—also called dynamic parameters or bind variables—are an alternative way to pass data to the database. Instead of putting the values directly into the SQL statement, you just use a placeholder like ?, :name or @name and provide the actual values using a separate API call.

原来说的是这种形式：

```go
rows, err := db.Query("select * from users where name = ?", "linuxfish")
```

与之比较的是还有一种写法，这种是臭名昭著的sql拼接，之前一直理解上面的那种写法也是sql拼接，看来是错怪它了：

```go
// 这样写是有被sql注入的危险的
myName := getNameFromUser()
rows, err := db.Query("select * from users where name = " + myName)
```

那么 Parameterized Queries 的好处是什么？

- 安全
    - the best way to prevent SQL injection.
- 性能高
    - 数据库会缓存执行计划，但必须是一模一样的sql，差一点都不行，那么在实际应用中这个缓存的作用就大打折扣了，因为`select * from users where id = 2`和`select * from users where id = 3`在数据库看来也是不一样的，尽管它们本质上是一个sql。
    - 用Parameterized Queries可以解决这个问题，因为变化的值变成参数了，对于数据库而言，请求一直是`select * from users where id = ?`
    - 但其实并没有那么完美，使用Parameterized Queries后，尽管sql不变了，但相对于真正执行的sql，它缺失了一部分信息，导致优化器无法作出最优的执行计划，因为参数的不同，最终的执行计划也可能是不一样的，因此，使用Parameterized Queries会性能高，是个伪命题
    - 这又是个取舍的问题（tradeoff），从总体来看，使用Parameterized Queries的好处还是大于坏处，所以，尽量还是使用Parameterized Queries
    - 参考：[Parameterized Queries](https://use-the-index-luke.com/sql/where-clause/bind-parameters)

### what sqlx brings us

#### StructScan

这是下面说的 Get 和 Select 实现的基础，支持把select出来的字段一下scan进一个struct

The primary extension on sqlx.Rows is StructScan(), which automatically scans results into struct fields. Note that the fields must be exported (capitalized) in order for sqlx to be able to write into them, something true of all marshallers in Go. You can use the db struct tag to specify which column name maps to each struct field, or set a new default mapping with db.MapperFunc(). The default behavior is to use strings.Lower on the field name to match against the column names.

那么是根据什么规则scan的呢？如上面所说，优先会找字段有没有设置`db` tag，如果设置了就用它，如果没有设置，那么默认使用的是`strings.Lower(fieldName)`，当然这个转换机制是可以定制的，你可以传入自己的处理函数。对应的实现代码应该在这里：

```go
var NameMapper = strings.ToLower

// ...

// mapper returns a valid mapper using the configured NameMapper func.
func mapper() *reflectx.Mapper {
    mprMu.Lock()
    defer mprMu.Unlock()

    if mpr == nil {
        mpr = reflectx.NewMapperFunc("db", NameMapper)
    } else if origMapper != reflect.ValueOf(NameMapper) {
        // if NameMapper has changed, create a new mapper
        mpr = reflectx.NewMapperFunc("db", NameMapper)
        origMapper = reflect.ValueOf(NameMapper)
    }
    return mpr
}

// 定制自己的mapper
import "github.com/jmoiron/sqlx/reflectx"
 
// Create a new mapper which will use the struct field tag "json" instead of "db"
db.Mapper = reflectx.NewMapperFunc("json", strings.ToLower)
```

#### Get and Select

把 query 和 scan 结合到一步了！

```go
p := Place{}
pp := []Place{}
 
// this will pull the first place directly into p
err = db.Get(&p, "SELECT * FROM place LIMIT 1")
 
// this will pull places with telcode > 50 into the slice pp
err = db.Select(&pp, "SELECT * FROM place WHERE telcode > ?", 50)
 
// they work with regular types as well
var id int
err = db.Get(&id, "SELECT count(*) FROM place")
 
// fetch at most 10 place names
var names []string
err = db.Select(&names, "SELECT name FROM place LIMIT 10")
```
risks 潜在风险点

Select can save you a lot of typing, but beware! It's semantically different from Queryx, since **it will load the entire result set into memory at once.** If that set is not bounded by your query to some reasonable size, it might be best to use the classic Queryx/StructScan iteration instead. 作者建议如果返回的数据集大小不确定，还是使用经典的for Next Scan的模式

##### Todo： 了解下 Get 和 Select 的原理

#### Transactions

Exec and all other query verbs will ask the DB for a connection and then return it to the pool each time. There's no guarantee that you will receive the same connection that the BEGIN statement was executed on. To use transactions, you must therefore use `DB.Begin()` 必须先 Begin 一个transaction

```go
tx, err := db.Begin()
err = tx.Exec(...)
err = tx.Commit()
```

Since transactions are connection state, the Tx object must bind and control a single connection from the pool. A Tx will maintain that single connection for its entire life cycle, releasing it only when Commit() or Rollback() is called. You should take care to call at least one of these, or else the connection will be held until garbage collection. 只有当`Commit()`或`Rollback()`时数据库连接才被放回连接池，记得调这两个函数，否则会有连接不释放的问题

Because you only have one connection to use in a transaction, you can only execute one statement at a time; the cursor types Row and Rows must be Scanned or Closed, respectively, before executing another query. If you attempt to send the server data while it is sending you a result, it can potentially corrupt the connection. 事务中只有一个连接可以使用，因此查询（Query）必须被Scan完或者Close后，后续的查询才能继续执行。

#### Named Queries

Named queries are common to many other database packages. They allow you to use a bindvar syntax which refers to the names of struct fields or map keys to bind variables a query, rather than having to refer to everything positionally. 这个类似于 Python format中的这种写法：`print("网站名：{name}, 地址 {url}".format(name="菜鸟教程", url="www.runoob.com"))` 好处是不用在意参数的位置了。

```go
// named query with a struct
p := Place{Country: "South Africa"}
rows, err := db.NamedQuery(`SELECT * FROM place WHERE country=:country`, p)
 
// named query with a map
m := map[string]interface{}{"city": "Johannesburg"}
result, err := db.NamedExec(`SELECT * FROM place WHERE city=:city`, m)
```
Named query support is implemented by parsing the query for the :param syntax and replacing it with the bindvar supported by the underlying database, then performing the mapping at execution, so it is usable on any database that sqlx supports. 实现原理

#### Advanced Scanning

##### Scan Destination Safety

扫描的destination变量里的字段必须是select的字段的超集，否则会报错，当然也可以忽略这个错误

```go
var p Person
// err here is not nil because there are no field destinations for columns in `place`
err = db.Get(&p, "SELECT * FROM person, place LIMIT 1;")
 
// this will NOT return an error, even though place columns have no destination
udb := db.Unsafe()
err = udb.Get(&p, "SELECT * FROM person, place LIMIT 1;")
```

##### Alternate Scan Types

In addition to using Scan and StructScan, an sqlx Row or Rows can be used to automatically return a slice or a map of results，还能scan成slice和map！！:

```go
rows, err := db.Queryx("SELECT * FROM place")
for rows.Next() {
    // cols is an []interface{} of all of the column results
    cols, err := rows.SliceScan()
}
 
rows, err := db.Queryx("SELECT * FROM place")
for rows.Next() {
    results := make(map[string]interface{})
    err = rows.MapScan(results)
}
```

##### Custom Types / 自定义类型

the examples above all used the built-in types to both scan and query with, but database/sql provides interfaces to allow you to use any custom types. database/sql包提供了接口让你来扩展你的自定义类型，让它们也可以被Scan和Query

参考：[Built In Interfaces](http://jmoiron.net/blog/built-in-interfaces) 

#### The Connection Pool

database/sql 包内置连接池，而且提供了几个函数来定制连接池的行为：

- DB.SetMaxIdleConns(n int)
    -  sets the maximum number of connections in the idle connection pool
    -  The default max idle connections is currently 2. This may change in a future release. 默认2个
- DB.SetMaxOpenConns(n int)
    - sets the maximum number of open connections to the database.
    - 默认无限制
- DB.SetConnMaxLifetime(d time.Duration)
    - sets the maximum amount of time a connection may be reused.

### 参考

- [Illustrated guide to SQLX](http://jmoiron.github.io/sqlx/)

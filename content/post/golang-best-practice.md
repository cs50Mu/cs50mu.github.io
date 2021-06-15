+++
title = "Golang工程最佳实践"
date = "2021-01-26"
slug = "2021/01/26/golang-best-practice"
Categories = ["Golang", "best practice"]
+++

### examples

- [Building Go Web Applications and Microservices Using Gin](https://semaphoreci.com/community/tutorials/building-go-web-applications-and-microservices-using-gin) 有实现，有测试，有重构

### 项目分层

- model
    - 所有的数据模型，被其它层所依赖
- repo
    - 数据层，屏蔽所有数据存取的细节
    - 它的底层可以是数据库，也可以是**其它服务**
- business
    - 业务逻辑层，依赖数据层
- controller
    - 负责参数输入的验证、统一返回结果，依赖业务逻辑层

思考：

理想情况下，一个项目应该有上面几层，但又想到，其实应该按照一个项目的复杂度来决定它的分层，当项目的复杂度不高时，非要分成上述这么多层，反而会比较繁琐（毕竟，逻辑写在一个文件里最清楚了。。）。

还有一个问题值得思考，就是我们是否应该将每层都定义成接口，然后层与层之间只是接口依赖，不依赖于具体的实现？这样做，

- 好处：
    - 更灵活，每一层的实现都是可随便替换的；
    - 更容易写单元测试，基于接口可以很容易写一个测试专用的mock实现。
- 坏处：
    - 如果不需要写单元测试（写针对接口的集成测试），上面的好处也没多大意义了
    - 如果实现（比如MySQL --> PG）基本不可能换，上面的好处也没多大意义

单纯从易于测试这个问题来说，基于接口的集成测试所需的时间肯定比mock依赖的单元测试要长，但感觉目前遇到的项目的复杂度并不高，执行一次集成测试的时间也还可以接受，因此暂且仅仅使用针对接口的集成测试看看效果，后续如果感觉需要加单元测试再慢慢迭代吧，随着认知的成长，越来越认识到，任何事物（项目、个人、公司等）的发展都有一个过程，迭代是个美妙的词，我不应该苛求每次工作刚开始做就要达到一个很高的标准，应该慢慢迭代，第一期达到基本可用，第二期增加xx功能，等等等等。

参考：

- [Testing Go services using interfaces](https://deliveroo.engineering/2019/05/17/testing-go-services-using-interfaces.html)
- [一个例子](https://itnext.io/beautify-your-golang-project-f795b4b453aa)
- [Trying Clean Architecture on Golang](https://hackernoon.com/golang-clean-archithecture-efd6d7c43047)  非常好的一个示例，典范，应当时不时回过头来看看
- [go-clean-arch](https://github.com/bxcodec/go-clean-arch)  上面帖子对应的代码
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) Uncle Bob

#### repo / 数据层如何支持数据库事务？

想了好久，参考了一些别人的做法，总算有了一个相对完美的方案。关键点如下：

- 需要抽象一个接口`queryUpdater`来代表`sql.DB`和`sql.TX`

```go
// queryer make sql query
type queryer interface {
    NamedExec(string, interface{}) (sql.Result, error)
    Get(interface{}, string, ...interface{}) error
    Select(interface{}, string, ...interface{}) error
}

// execer execute a sql
type execer interface {
    Exec(string, ...interface{}) (sql.Result, error)
}

type queryExecer interface {
    queryer
    execer
}
```

- dao对象里既有底层的sql.DB也有这个抽象的接口

```go
// DAO dao
type DAO struct {
    db     *sqlx.DB
    dbConn queryExecer
}

// NewDAO new DAO
func NewDAO() *DAO {
    return &DAO{
        db:     db,
        dbConn: db,
    }
}
```

- 提供一个`WithTx` wrapper函数，只要是包在这个函数里的都是在一个数据库事务里执行

```go
// WithTx wrap txFunc in a db transaction
func (dao *DAO) WithTx(txFunc func(dao *DAO)) {
    tx, _ := dao.db.Beginx()

    txDAO := &DAO{
        dbConn: tx,
    }

    defer func() {
        // TODO: what if err in defer?
        // eg, tx.Rollback returns error
        if r := recover(); r != nil {
            err := fmt.Errorf("%v", r)
            // err := r.(error)
            txErr := tx.Rollback()
            if txErr != nil {
                fmt.Printf("tx rollback failed: %+v\n", txErr)
            }
            fmt.Println("tx rollback")
            // re-panic
            panic(err)
        } else {
            tx.Commit()
            fmt.Println("tx committed")
        }
    }()

    txFunc(txDAO)
}
```

事务的用法如下：

```go
    jccoinDAO := dao.NewDAO()

    var topup *model.TopUp

    jccoinDAO.WithTx(func(dao *dao.DAO) {
        // https://stackoverflow.com/questions/31981592/assign-struct-with-another-struct
        topup = &model.TopUp{
            BaseTopUp:   req.BaseTopUp,
            PaidTotal:   req.TopUpAmount,
            TimeCreated: time.Now(),
            Remark:      req.Remark,
        }

        // new topup
        // idempotence support
        if isDup := dao.NewTopUp(c, topup); isDup {
            topup = dao.GetTopUpByOrderNo(c, topup.TopUpNo, topup.Channel)
            return
        }

        // get buyer_account, update balance
        buyerAccount := model.Account{
            UserID:            topup.BuyerID,
            AccountType:       "BUYER",
            DeviceType:        topup.DeviceType,
            SystemCode:        topup.SysCode,
            IsPlatformAccount: false,
            AccountNumber:     "",
            Status:            "OPEN",
            Currency:          "JCC",
            TimeCreated:       nowFunc(),
            TimeUpdated:       nowFunc(),
        }
        buyerGiftAccount := buyerAccount
        buyerGiftAccount.AccountType = "BUYERGIFT"

        fmt.Printf("buyerAccount: %+v\n", buyerAccount)
        ba := dao.GetOrCreateBuyerAccount(c, &buyerAccount)
        if !topup.TopUpAmount.IsZero() {
            buyerBalance := ba.Balance.Add(topup.TopUpAmount)
            dao.UpdateAccountBalance(c, ba.ID, buyerBalance)
        }

        // get buyer_gift_account, update balance
        fmt.Printf("giftAccount: %+v\n", buyerGiftAccount)
        ga := dao.GetOrCreateBuyerAccount(c, &buyerGiftAccount)
        if !topup.GiftAmount.IsZero() {
            giftBalance := ga.Balance.Add(topup.GiftAmount)
            dao.UpdateAccountBalance(c, ga.ID, giftBalance)
        }
    })
```

非事务的查询就是直接在dao上查就行：

```go
    buyerBalance := jccoinDAO.GetBuyerTotalBalance(c, req.BuyerID, req.DeviceType)
    if req.CoinsAmount.GreaterThan(buyerBalance) {
        err = ErrInsufficientBalance
        return
    }
```

完美实现了代码的复用，如果不这样的话，使用事务和不使用事务时同样的查询函数要写两遍。。或者需要在参数里把`sql.DB`和`sql.Tx`都带上

参考：

- [Go Microservice with Clean Architecture: Transaction Support](https://medium.com/@jfeng45/go-microservice-with-clean-architecture-transaction-support-61eb0f886a36)  这也是个完整的方案，但感觉太复杂了。。
- [db transaction in golang](https://stackoverflow.com/questions/26593867/db-transaction-in-golang)
- [database/sql Tx - detecting Commit or Rollback](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)
- [agungdwiprasetyo/agungdpcms](https://github.com/agungdwiprasetyo/agungdpcms/blob/master/src/resume/usecase/usecase_impl.go#L77) 起初是这位大哥的代码给了我希望
- [Transaction #27](https://github.com/bxcodec/go-clean-arch/issues/27) 而起初是在go-clean-arch的这个issue里看到他的回复的

#### cache / 缓存层？

利用 [Proxy Pattern](https://github.com/tmrts/go-patterns/blob/master/structural/proxy.md)，cache 层也实现repo层的接口，把repo包在里面，具体参考这个[回答](https://github.com/bxcodec/go-clean-arch/issues/11#issuecomment-594679205)

### 代码Lint

当前使用了[revive](https://github.com/mgechev/revive)，号称 drop-in replacement for golint，确实名副其实，比golint强大多了~ 支持lint指定文件（夹）、排除文件夹等，支持lint参数定制化。当前支持的lint项在[这里](https://revive.run/r#redefines-builtin-id)能看到，而且它还暴露了接口让你能自定义lint项

参考：

- [Simple tools to improve your Go code](https://remy.io/blog/simple-tools-to-improve-your-go-code/)
- [awesome-go-linters](https://github.com/golangci/awesome-go-linters)
- [revive](https://github.com/mgechev/revive)  当前使用的，非常不错

### go design patterns

Curated list of Go design patterns, recipes and idioms

[go-patterns](https://github.com/tmrts/go-patterns)

### cmd / 临时脚本？

常常有这样的需求，有一些脚本性质的任务，偶尔需要临时执行一下，这种需求在Golang项目中该怎么做呢？

参考了几个大型项目，比如 k8s，可以这么来做，此类任务统统放在`cmd`目录下，这个目录下可以继续建子目录，脚本就写在子目录中，但注意脚本里声明的`package`需要是`main`，如果需要执行脚本的话，可以直接执行`go run <脚本路径>`即可（或者把它build成一个可执行文件）

比如在项目中我有如下脚本：

```
cmd
└── kafka
    └── kafka_consume.go
```
那么，可以直接这样来执行它：

`go run cmd/kafka/kafka_consume.go`

> 学到的

`package`的名字可以跟包含它的文件夹的名字不一样

参考：

- [golang-standards / project-layout](https://github.com/golang-standards/project-layout/tree/master/cmd)
- [kubernetes/cmd](https://github.com/kubernetes/kubernetes/tree/master/cmd)

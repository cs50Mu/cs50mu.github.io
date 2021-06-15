+++
title = "Redis中set元素设置过期？"
date = "2019-03-24"
slug = "2019/03/24/expire-elements-in-redis-set"
Categories = []
+++

### 需求

项目中遇到一个需求，要实现一个黑白名单的服务，对外暴露向名单添加元素和检查元素是否在名单中的接口，这个用 Redis 实现是很容易的，有现成的类型支持，但还有个
需求是名单中的某些元素要支持自动过期，即过了一定时间后，元素自动从名单中消失了，再去 check 该元素是否在名单时，应该返回 false。Redis 是支持过期的，但它只
支持 key 级别的过期。

### 方案

看了下，早有人给 Redis 项目提出类似的issue：要求支持元素级别的过期。项目的维护者也早已指出：不可能支持这样的 feature，因为违背了 Redis 的设计理念：简单、高效。
不过，在 Google Group 上看到 Redis 的作者针对这类需求给出了3个实现方案：

- 用 redis 的普通 set 类型实现。把时间戳 encode 进元素名称中，比如平常只是 add 一个元素 foo，现在需要 add 元素名：`foo:<timestamp>`。那么每次需要 check 这个元素
的时候先获取一下当前的时间戳跟保存的时间戳比较一下，如果已经过期，则删除它。这个方案的缺点是：如果 add 了一个元素后，一直不再访问它，那么尽管给它设置了过期时间，
那么它还是会一直存在。

- 使用 redis 的 sorted set来实现。score 是元素过期的时间戳，value 是元素名。在代码中每秒执行一次`zremrangebyscore`来清除已过期的元素。

- 第三种没看懂。。

在 stackoverflow 上看到有人给出了类似上面说的第二种方案的具体实现，参考文末链接。

### 参考

- [Pattern for expiring set members](https://groups.google.com/forum/#!topic/redis-db/rXXMCLNkNSs)
- [how to handle session expire basing redis?](https://stackoverflow.com/questions/11810020/how-to-handle-session-expire-basing-redis/11815594#11815594)

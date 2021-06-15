+++
title = "如何处理数据库大结果集?"
date = "2017-11-26"
slug = "2017/11/26/big-resultsets-handling-in-django"
Categories = []
+++

问题的起因是想知道在Django ORM中如何处理大数据集的返回，比如怎么避免进程由于内存占用过多被kill掉。由于数据库使用的是MySQL，讨论是从MySQL开始的。

MySQL协议是半双工的，同一时间只能有一方在说话，一旦服务器开始返回数据，客户端能做的只能是把数据接收完，不存在让服务器停止这一说。。（因为还没轮到它说嘛。。）所以，写sql的时候注意加limit是很重要的！

那么对于一次sql查询的返回，MySQL客户端有两个选择：

1、MySQL客户端默认会一次性先把服务器返回的数据先缓存起来，再给它的客户去用。这样做的好处是，能尽快解放数据库相关线程，让它们去做更重要的事，比如服务其它请求（因为服务器通常需要等所有数据都发送完后才释放这条查询相关的资源的），坏处是如果返回的结果很大，客户端需要花费很多时间和内存来接收它，更气人的是，在这期间，做为客户端的用户，你啥也干不了，只能等。

2、MySQL客户端还有一个选择，就是不自己先缓存啦，一边从server接收一边就返回给它的用户了~这样做的好处是，对于MySQL客户端的用户来说，它看起来反应更快了（因为没有先缓存所有数据集），而且貌似还便于做内存占用的优化？比如一边接收、处理，一边删除已处理过的数据，使得内存占用始终保持在一个很小的数量。坏处是有一个处理很慢的用户可能会拖垮数据库，并且在接收完之前客户端不能做任何其它的事情！这其实是一个Unbuffered Cursor，在MySQL客户端实现里叫SSCursor，即server side cursor，但其实它不是真正的server side cursor

Django ORM的queryset在被使用的时候会触发对应sql查询被执行，像上面说的，默认情况下结果集会被客户端先缓存，这已经是一个内存占用，然后会被Django转成对应的model instance，这又是一个内存占用，如果内存没有被及时回收，这其实是一份数据的双倍内存占用。

queryset的一个方法iterator能实现的一个优化是，将数据集转成model instance的过程改为generator模式，减少第二步的内存占用（注意：这只是针对MySQL来说的）

对于会返回大数据集的查询，一个处理办法是拆分使用limit来多次接收。每次接收一定量的数据（比如1000），内存回收后再接收下一个1000，对应的Django实现：

```python
import gc

def queryset_iterator(queryset, chunksize=1000):
    '''''
    Iterate over a Django Queryset ordered by the primary key

    This method loads a maximum of chunksize (default: 1000) rows in it's
    memory at the same time while django normally would load all rows in it's
    memory. Using the iterator() method only causes it to not preload all the
    classes.

    Note that the implementation of the iterator does not support ordered query sets.
    '''
    pk = 0
    last_pk = queryset.order_by('-pk')[0].pk
    queryset = queryset.order_by('pk')
    while pk < last_pk:
        for row in queryset.filter(pk__gt=pk)[:chunksize]:
            pk = row.pk
            yield row
        gc.collect()
```

参考：

- [https://stackoverflow.com/questions/4856882/limiting-memory-use-in-a-large-django-queryset?rq=1](https://stackoverflow.com/questions/4856882/limiting-memory-use-in-a-large-django-queryset?rq=1)
- [https://stackoverflow.com/questions/14144408/memory-efficient-constant-and-speed-optimized-iteration-over-a-large-table-in?noredirect=1&lq=1](https://stackoverflow.com/questions/14144408/memory-efficient-constant-and-speed-optimized-iteration-over-a-large-table-in?noredirect=1&lq=1)
- [https://djangosnippets.org/snippets/1949/](https://djangosnippets.org/snippets/1949/)
- [https://docs.djangoproject.com/en/1.11/ref/models/querysets/#iterator](https://docs.djangoproject.com/en/1.11/ref/models/querysets/#iterator)

下面讲一下真正的server side cursor，postgresql支持真正意义上的server side cursor，可以给一个查询指定一个cursor，然后在这个cursor上操作，fetch多行、move(移动）cursor、更新当前cursor所在的记录等等，可参考[文档](https://www.postgresql.org/docs/9.2/static/plpgsql-cursors.html)。看MySQL[官方文档](https://dev.MySQL.com/doc/refman/5.7/en/cursors.html)，它也支持一个很简陋的server side cursor，而且只能在存储过程里用。

顺便，google到一个对比Client-Side Cursors和Server-Side Cursors区别的文档，竟然来自微软。。

Client-Side Cursors

With a non-keyset client-side cursor, the server sends the entire result set across the network to the client machine. The client machine provides and manages the temporary resources needed by the cursor and result set. The client-side application can browse through the entire result set to determine which rows it requires.

Static and keyset-driven client-side cursors may place a significant load on your workstation if they include too many rows. While all of the cursor libraries are capable of building cursors with thousands of rows, applications designed to fetch such large rowsets may perform poorly. There are exceptions, of course. For some applications, a large client-side cursor may be perfectly appropriate and performance may not be an issue.

One obvious benefit of the client-side cursor is quick response. After the result set has been downloaded to the client machine, browsing through the rows is very fast. Your application is generally more scalable with client-side cursors because the cursor's resource requirements are placed on each separate client and not on the server.

Server-Side Cursors

With a server-side cursor, the server manages the result set using resources provided by the server machine. The server-side cursor returns only the requested data over the network. This type of cursor can sometimes provide better performance than the client-side cursor, especially in situations where excessive network traffic is a problem.

Server-side cursors also permit more than one operation on the connection. That is, once you create the cursor, you can use the same connection to make changes to the rows — without having to establish an additional connection to handle the underlying update queries.

However, it's important to point out that a server-side cursor is — at least temporarily — consuming precious server resources for every active client. You must plan accordingly to ensure that your server hardware is capable of managing all of the server-side cursors requested by active clients. Also, a server-side cursor can be slow because it provides only single row access — there is no batch cursor available.

Server-side cursors are useful when inserting, updating, or deleting records. With server-side cursors, you can have multiple active statements on the same connection. With SQL Server, you can have pending results in multiple statement handles.

参考：

- [Client-Side Cursors Versus Server-Side Cursors](https://msdn.microsoft.com/en-us/library/aa266531(v=vs.60).aspx)
- [Very Large Result Sets in Django using PostgreSQL](http://thebuild.com/blog/2010/12/13/very-large-result-sets-in-django-using-postgresql/)
- [postgresql cursors](https://www.postgresql.org/docs/9.2/static/plpgsql-cursors.html)
- [Retrieving million of rows from MySQL](http://techualization.blogspot.com/2011/12/retrieving-million-of-rows-from-MySQL.html)
- [MySQL cursors](https://dev.MySQL.com/doc/refman/5.7/en/cursors.html)
- [Django中对server side cursor的支持](https://code.djangoproject.com/ticket/16614#no1)

+++
title = "job queue in Golang"
date = "2019-07-20"
slug = "2019/07/20/job-queue-in-golang"
Categories = ["Golang"]
+++

异步任务，还是蛮常见的

### 算不上 job queue的形式

`go process(job)`

这其实算不上一个queue，但简单。同时，少了一些对并发的控制，比如控制同时执行的任务数等。

### 最简单的 job queue

```go
func worker(jobChan <-chan Job) {
    for job := range jobChan {
        process(job)
    }
}

// make a channel with a capacity of 100.
jobChan := make(chan Job, 100)

// start the worker
go worker(jobChan)

// enqueue a job
jobChan <- job
```
注释已经很清楚了，通过向channel发消息来提交任务，worker 从 channel 中取任务做。注意 jobChan 是一个固定长度的 channel，这能够实现 producer throtting，当 queue 中已经有100个 task 时，此时的 enqueue 操作会阻塞。

### 非阻塞式 enqueue

如果 enqueue 时不想阻塞呢？比如我想如果队列满了，就直接给 client 端返回失败，告诉它等会再试试。可以这样：

```go
// TryEnqueue tries to enqueue a job to the given job channel. Returns true if
// the operation was successful, and false if enqueuing would not have been
// possible without blocking. Job is not enqueued in the latter case.
func TryEnqueue(job Job, jobChan <-chan Job) bool {
    select {
    case jobChan <- job:
        return true
    default:
        return false
    }
}

// then you can do this
if !TryEnqueue(job, chan) {
    http.Error(w, "max capacity reached", 503)
    return
}
```

### 停止 worker

如果没有任务需要做了，那么可以：

`close(jobChan)`

因为 worker 是通过`for job := range jobChan {...}` 这种形式来取任务的，当 channel 被关闭后，for loop 会停止循环，继而结束 worker。

需要注意的是：即使 channel 被 close 的时候，channel 里还有尚未被消费的 task，这些 task 照样会被正常消费完

### 等待 worker 退出

`close` channel 只会通知 worker 当前已无更多任务，但并不会等待 worker 把任务做完，所以我们需要一种等待 worker 的机制：

```go
// use a WaitGroup 
var wg sync.WaitGroup

func worker(jobChan <-chan Job) {
    defer wg.Done()

    for job := range jobChan {
        process(job)
    }
}

// increment the WaitGroup before starting the worker
wg.Add(1)
go worker(jobChan)

// to stop the worker, first close the job channel
close(jobChan)

// then wait using the WaitGroup
wg.Wait()
```

### 带超时时间的等待

如果 worker 的任务一直没有做完，那么`wg.Wait()` 会无休止的等待下去，如果我们无法承受一直等待怎么办呢？

可以把`wg.Wait()`封装一下，给它增加 `timeout` 的功能

```go
// WaitTimeout does a Wait on a sync.WaitGroup object but with a specified
// timeout. Returns true if the wait completed without timing out, false
// otherwise.
func WaitTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
    ch := make(chan struct{})
    go func() {
        wg.Wait()
        close(ch)
    }()
    select {
    case <-ch:
            return true
    case <-time.After(timeout):
            return false
    }
}

// now use the WaitTimeout instead of wg.Wait()
WaitTimeout(&wg, 5 * time.Second)
``` 

### 取消 worker

上面的代码中，如果我们发出退出的信号，worker 们会做完当前正在做的任务然后再退出，如果我们想让它立即退出该怎么办呢？

可以利用`context.Context`

```go
// create a context that can be cancelled
ctx, cancel := context.WithCancel(context.Background())

// start the goroutine passing it the context
go worker(ctx, jobChan)

func worker(ctx context.Context, jobChan <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return

        case job := <-jobChan:
            process(job)
        }
    }
}

// Invoke cancel when the worker needs to be stopped. This *does not* wait
// for the worker to exit.
cancel()
```

但这里有一个小坑，当在收到退出信号时，同时也有job可取，那么 select 会**随机**选择一个路径来执行，并不会优先现在退出的路径，如果你想优先退出的话，需要这样：

```go
var flag uint64

func worker(ctx context.Context, jobChan <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return

        case job := <-jobChan:
            process(job)
            if atomic.LoadUint64(&flag) == 1 {
                return
            }
        }
    }
}

// set the flag first, before cancelling
atomic.StoreUint64(&flag, 1)
cancel()
```

或者这样：

```go
func worker(ctx context.Context, jobChan <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return

        case job := <-jobChan:
            process(job)
            if ctx.Err() != nil {
                return
            }
        }
    }
}

cancel()
```
(译者注：我觉得这样仍然不能保证退出路径优先被执行呀）

### 不用 context 也能取消 worker

如下，在某些场景下可能写法还更简洁，当然原理是一样的

```go
// create a cancel channel
cancelChan := make(chan struct{})

// start the goroutine passing it the cancel channel 
go worker(jobChan, cancelChan)

func worker(jobChan <-chan Job, cancelChan <-chan struct{}) {
    for {
        select {
        case <-cancelChan:
            return

        case job := <-jobChan:
            process(job)
        }
    }
}

// to cancel the worker, close the cancel channel
close(cancelChan)
```

### worker 池

最简单的就是启动多个 worker，让它们读取同一个 channel

```go
for i:=0; i<workerCount; i++ {
    go worker(jobChan)
}
```

如果想要等待 worker 退出

```go
for i:=0; i<workerCount; i++ {
    wg.Add(1)
    go worker(jobChan)
}

// wait for all workers to exit
wg.Wait()
```

取消 worker

```go
// create cancel channel
cancelChan := make(chan struct{})

// pass the channel to the workers, let them wait on it
for i:=0; i<workerCount; i++ {
    go worker(jobChan, cancelChan)
}

// close the channel to signal the workers
close(cancelChan)
```

### 参考

- [JOB QUEUES IN GO](https://disqus.com/embed/comments/?base=default&f=opsdash&t_u=https%3A%2F%2Fwww.opsdash.com%2Fblog%2Fjob-queues-in-go.html&t_d=Job%20Queues%20in%20Go%20-%20OpsDash&t_t=Job%20Queues%20in%20Go%20-%20OpsDash&s_o=default#version=5c281b90be9cbae86fbebcbaed6c8c9b)
- [Golang Workers / Job Queue](https://gist.github.com/harlow/dbcd639cf8d396a2ab73)
- [Writing worker queues, in Go](http://nesv.github.io/golang/2014/02/25/worker-queues-in-go.html)
- [Handling 1 Million Requests per Minute with Go](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/)

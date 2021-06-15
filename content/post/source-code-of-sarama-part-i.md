+++
title = "sarama源码分析Part I"
date = "2021-01-22"
slug = "2021/01/22/source-code-of-sarama-part-i"
Categories = []
+++

[Shopify / sarama](https://github.com/Shopify/sarama)

以下代码分析基于此快照的代码：2a5254c26ef606cd9a587

### 生产者如何发送消息

这一篇帖子已经分析的非常好了：[Sarama生产者是如何工作的](https://juejin.cn/post/6866316565348876296#heading-6)

在evernote的备份：[Sarama生产者是如何工作的](https://www.evernote.com/shard/s112/nl/12260165/5ab3f175-2851-4069-b3b4-b6dc20f7f203/)

也可以看下官方wiki里关于producer实现的说明：[Producer implementation](https://github.com/Shopify/sarama/wiki/Producer-implementation)

#### 从提问题中来学习

> kafka client 是如何与 broker server 交互的？

以 获取 metadata 为例（一个client启动后先要做的就是获取metadata，比如，有哪些topic，某个topic有几个partition，哪个broker是leader等）：

可以看到内部调用的是`sendAndReceive`这个方法

```go
//GetMetadata send a metadata request and returns a metadata response or error
func (b *Broker) GetMetadata(request *MetadataRequest) (*MetadataResponse, error) {
  response := new(MetadataResponse)

  err := b.sendAndReceive(request, response)

  if err != nil {
      return nil, err
  }

  return response, nil
}
```

```go
  func (b *Broker) sendAndReceive(req protocolBody, res versionedDecoder) error {
      // send 返回一个 promise
      promise, err := b.send(req, res != nil)
      if err != nil {
          return err
      }

      // 当配置`RequiredAcks`=0(即不关注server返回)时，返回的promise会是nil
      if promise == nil {
          return nil
      }

      // 然后就等待这个 promise
      select {
      case buf := <-promise.packets:
          return versionedDecode(buf, res, req.version())
      case err = <-promise.errors:
          return err
      }
  }
```
既然是返回了promise，说明`send`方法是异步的：

```go
// 为节省篇幅，省略了一些错误处理的展示
  func (b *Broker) send(rb protocolBody, promiseResponse bool) (*responsePromise, error) {
      b.lock.Lock()
      defer b.lock.Unlock()

....
      // request的correlationID
      req := &request{correlationID: b.correlationID, clientID: b.conf.ClientID, body: rb}
      buf, err := encode(req, b.conf.MetricRegistry)
      if err != nil {
          return nil, err
      }

      requestTime := time.Now()
      // Will be decremented in responseReceiver (except error or request with NoResponse)
      b.addRequestInFlightMetrics(1)
      // 关键在这里了，这里把请求发出去了
      // 最终会调用`conn.Write`
      bytes, err := b.write(buf)
      b.updateOutgoingCommunicationMetrics(bytes)
      if err != nil {
          b.addRequestInFlightMetrics(-1)
          return nil, err
      }
      b.correlationID++

      if !promiseResponse {
          // Record request latency without the response
          b.updateRequestLatencyAndInFlightMetrics(time.Since(requestTime))
          return nil, nil
      }

      // 组装了一个promise，然后把它发给了一个channel：b.responses
      // 注意这个promise里也有两个channel，它是用来返回结果的
      
      // 关联上了req.correlationID
      // 通过这个correlationID来关联req和response
      promise := responsePromise{requestTime, req.correlationID, make(chan []byte), make(chan error)}
      b.responses <- promise

      return &promise, nil
  }
```

那么，很明显的，一定有人在监听这个`b.responses`了，是在`responseReceiver`方法中：

```go
  func (b *Broker) responseReceiver() {
      var dead error
      header := make([]byte, 8)

      // 注意是一个循环读取消息，即使没有消息
      // 也会等待在这里，除非有人关闭了这个channel
      for response := range b.responses {
          ...
          // 读header
          bytesReadHeader, err := b.readFull(header)
          requestLatency := time.Since(response.requestTime)
          ....

          decodedHeader := responseHeader{}
          err = decode(header, &decodedHeader)
          ...
          
          // 这里会校验收到的消息是否是所预期的，若对不上直接抛弃，继续处理下一条
          if decodedHeader.correlationID != response.correlationID {
              b.updateIncomingCommunicationMetrics(bytesReadHeader, requestLatency)
              // TODO if decoded ID < cur ID, discard until we catch up
              // TODO if decoded ID > cur ID, save it so when cur ID catches up we have a response
              dead = PacketDecodingError{fmt.Sprintf("correlation ID didn't match, wanted %d, got %d", response.correlationID, decodedHeader.correlationID)}
              response.errors <- dead
              continue
          }
          
          // 读body
          buf := make([]byte, decodedHeader.length-4)
          bytesReadBody, err := b.readFull(buf)
          b.updateIncomingCommunicationMetrics(bytesReadHeader+bytesReadBody, requestLatency)
          if err != nil {
              dead = err
              response.errors <- err
              continue
          }

          // 将读到的body字节通过 channel 返回
          response.packets <- buf
      }
      close(b.done)
  }
```

那么，这个`responseReceiver`方法是什么时候调用的呢？是在open 一个 broker 的时候：

```go
  func (b *Broker) Open(conf *Config) error {
      if !atomic.CompareAndSwapInt32(&b.opened, 0, 1) {
          return ErrAlreadyConnected
      }

      if conf == nil {
          conf = NewConfig()
      }

      err := conf.Validate()
      if err != nil {
          return err
      }

      b.lock.Lock()

      go withRecover(func() {
          defer b.lock.Unlock()

          dialer := net.Dialer{
              Timeout:   conf.Net.DialTimeout,
              KeepAlive: conf.Net.KeepAlive,
              LocalAddr: conf.Net.LocalAddr,
          }

          ...
          
          b.conn = newBufConn(b.conn)

          b.conf = conf
          
          ....
          
          if b.id >= 0 {
              b.registerMetrics()
          }

          ....

          b.done = make(chan bool)
          b.responses = make(chan responsePromise, b.conf.Net.MaxOpenRequests-1)

          if b.id >= 0 {
              Logger.Printf("Connected to broker at %s (registered as #%d)\n", b.addr, b.id)
          } else {
              Logger.Printf("Connected to broker at %s (unregistered)\n", b.addr)
          }
          // 这里单独起了一个协程来处理
          go withRecover(b.responseReceiver)
      })          
```

什么时候会open broker呢？在新建Client的时候

```go
func NewClient(addrs []string, conf *Config) (Client, error) {
    Logger.Println("Initializing new client")
    ...
    if conf.Metadata.Full {
        // do an initial fetch of all cluster metadata by specifying an empty list of topics
        // 这里会从broker拉取一次metadata
        // 在此过程中首先会先open一个broker
        err := client.RefreshMetadata()
        switch err {
        case nil:
            break
        case ErrLeaderNotAvailable, ErrReplicaNotAvailable, ErrTopicAuthorizationFailed, ErrClusterAuthorizationFailed:
            // indicates that maybe part of the cluster is down, but is not fatal to creating the client
            Logger.Println(err)
        default:
            close(client.closed) // we haven't started the background updater yet, so we have to do this manually
            _ = client.Close()
            return nil, err
        }
    }
    // 这里启动后台定时拉取metadata的任务
    go withRecover(client.backgroundMetadataUpdater)

    Logger.Println("Successfully initialized new client")
    
    return client, nil    
    }
```
总结一下，模式是：制造任务，扔进队列（channel），新启动一个协程来监听队列（channel），有新任务就做，任务的完成情况是通过任务结构体中的channel来跟调用方完成通信的

> 通过 kafka client 生产(produce)一条消息，需要经过哪些步骤？

sarama 有两种 producer: syncProducer 和 asyncProducer，syncProducer 只是 asyncProducer 的一个简单封装，下面只介绍 asyncProducer 的发送消息的流程：

一条消息在真正被发到网络上之前，会流经以下结构：

asyncProducer(singleton) --> topicProducer(one per topic) --> partitionProducer(one per partition) --> brokerProducer(one per broker)

消息的传递是通过 channel，每个结构体在处理完消息后，会将它丢进下一个结构体在监听的channel中。

重点关注下`BrokerProducer`，这里起了两个goroutine，`bp.run`负责将消息打包（压缩）成一个set，另一个goroutine负责将打包好的set真正发出去

```go
// one per broker; also constructs an associated flusher
func (p *asyncProducer) newBrokerProducer(broker *Broker) *brokerProducer {
    var (
        input     = make(chan *ProducerMessage)
        bridge    = make(chan *produceSet)
        responses = make(chan *brokerProducerResponse)
    )

    bp := &brokerProducer{
        parent:         p,
        broker:         broker,
        input:          input,
        output:         bridge,
        responses:      responses,
        stopchan:       make(chan struct{}),
        buffer:         newProduceSet(p),
        currentRetries: make(map[string]map[int32]error),
    }
    go withRecover(bp.run)

    // minimal bridge to make the network response `select`able
    // 注意这里是按set来发的，消息走到这里时已经被整合成一个batch了
    go withRecover(func() {
        for set := range bridge {
            request := set.buildRequest()

            response, err := broker.Produce(request)

            responses <- &brokerProducerResponse{
                set: set,
                err: err,
                res: response,
            }
        }
        close(responses)
    })

    if p.conf.Producer.Retry.Max <= 0 {
        bp.abandoned = make(chan struct{})
    }

    return bp
}
```


参考：[Message Flow](https://github.com/Shopify/sarama/wiki/Producer-implementation#message-flow)

> 如何保持发送的消息有序？

基本思想是：需要重发的消息会被重新扔到队列中，不过producer对于重发的消息和正常的消息的处理策略不一样：

- 从发现有重发的消息起，优先发送`retriedMsg`
- 正常的消息会被缓存起来
- 当确认所有的`retriedMsg`都发送成功后，再将这期间缓存的正常消息一次性发送出去

代码其实比刚刚的描述复杂多了，它还有一个“水位线(Watermark)”的概念，允许重试多次，对于重试同样次数的消息被放在同一个level的buffer中

Maintaining Order

Maintaining the order of messages when a retry occurs is an additional challenge. When a brokerProducer triggers a retry, the following events occur, strictly in this order:

- the messages to retry are sent to the retryHandler

- the brokerProducer sets a flag for the given topic/partition; while this flag is set any further such messages (which may have already been in the pipeline) will be immediately sent to the retryHandler

```go
// 此处逻辑位于 func (bp *brokerProducer) handleSuccess 方法中
        sent.eachPartition(func(topic string, partition int32, pSet *partitionSet) {
            block := response.GetBlock(topic, partition)
            if block == nil {
                // handled in the previous "eachPartition" loop
                return
            }

            switch block.Err {
            case ErrInvalidMessage, ErrUnknownTopicOrPartition, ErrLeaderNotAvailable, ErrNotLeaderForPartition,
                ErrRequestTimedOut, ErrNotEnoughReplicas, ErrNotEnoughReplicasAfterAppend:
                Logger.Printf("producer/broker/%d state change to [retrying] on %s/%d because %v\n",
                    bp.broker.ID(), topic, partition, block.Err)
                if bp.currentRetries[topic] == nil {
                    bp.currentRetries[topic] = make(map[int32]error)
                }
                // 这里是所谓的`sets a flag for the given topic/partition`
                // 用于标识这个topic/partition是有问题的
                bp.currentRetries[topic][partition] = block.Err
                // 以下为将消息发送至 retryHandler
                if bp.parent.conf.Producer.Idempotent {
                    go bp.parent.retryBatch(topic, partition, pSet, block.Err)
                } else {
                    bp.parent.retryMessages(pSet.msgs, block.Err)
                }
                // dropping the following messages has the side effect of incrementing their retry count
                bp.parent.retryMessages(bp.buffer.dropPartition(topic, partition), block.Err)
            }
        })
```


```go
// 此处逻辑位于 func (bp *brokerProducer) run 方法
            if reason := bp.needsRetry(msg); reason != nil {
                bp.parent.retryMessage(msg, reason)

                if bp.closing == nil && msg.flags&fin == fin {
                    // we were retrying this partition but we can start processing again
                    delete(bp.currentRetries[msg.Topic], msg.Partition)
                    Logger.Printf("producer/broker/%d state change to [closed] on %s/%d\n",
                        bp.broker.ID(), msg.Topic, msg.Partition)
                }

                continue
            }
            
func (bp *brokerProducer) needsRetry(msg *ProducerMessage) error {
    if bp.closing != nil {
        return bp.closing
    }

    return bp.currentRetries[msg.Topic][msg.Partition]
}
```

- eventually the first retried message reaches its partitionProducer

- the partitionProducer sends off a special "chaser" message and releases its reference to the old broker

```go
// 此处逻辑位于 func (pp *partitionProducer) dispatch 中
        if msg.retries > pp.highWatermark {
            // a new, higher, retry level; handle it and then back off
            pp.newHighWatermark(msg.retries)
            pp.backoff(msg.retries)
        }

func (pp *partitionProducer) newHighWatermark(hwm int) {
    Logger.Printf("producer/leader/%s/%d state change to [retrying-%d]\n", pp.topic, pp.partition, hwm)
    pp.highWatermark = hwm

    // send off a fin so that we know when everything "in between" has made it
    // back to us and we can safely flush the backlog (otherwise we risk re-ordering messages)
    pp.retryState[pp.highWatermark].expectChaser = true
    pp.parent.inFlight.Add(1) // we're generating a fin message; track it so we don't shut down while it's still inflight
    // 注意此处的retries做了减一操作
    // 这样的话，当brokerProducer收到这个fin消息，并重新retry到partitionProducer中时
    // 才会满足msg.retries == highWatermark 从而被识别
    pp.brokerProducer.input <- &ProducerMessage{Topic: pp.topic, Partition: pp.partition, flags: fin, retries: pp.highWatermark - 1}

    // a new HWM means that our current broker selection is out of date
    Logger.Printf("producer/leader/%s/%d abandoning broker %d\n", pp.topic, pp.partition, pp.leader.ID())
    pp.parent.unrefBrokerProducer(pp.leader, pp.brokerProducer)
    pp.brokerProducer = nil
}
```

- the partitionProducer updates its metadata, opens a connection to the new broker, and sends the retried message down the new path

```go
// 依然在 func (pp *partitionProducer) dispatch 中
        // if we made it this far then the current msg contains real data, and can be sent to the next goroutine
        // without breaking any of our ordering guarantees

        if pp.brokerProducer == nil {
            if err := pp.updateLeader(); err != nil {
                pp.parent.returnError(msg, err)
                pp.backoff(msg.retries)
                continue
            }
            Logger.Printf("producer/leader/%s/%d selected broker %d\n", pp.topic, pp.partition, pp.leader.ID())
        }

        // Now that we know we have a broker to actually try and send this message to, generate the sequence
        // number for it.
        // All messages being retried (sent or not) have already had their retry count updated
        // Also, ignore "special" syn/fin messages used to sync the brokerProducer and the topicProducer.
        if pp.parent.conf.Producer.Idempotent && msg.retries == 0 && msg.flags == 0 {
            msg.sequenceNumber, msg.producerEpoch = pp.parent.txnmgr.getAndIncrementSequenceNumber(msg.Topic, msg.Partition)
            msg.hasSequence = true
        }

        pp.brokerProducer.input <- msg
    }
```

- the partitionProducer continues handling incoming messages - retried messages get sent to the new broker, while new messages are held in a queue to preserver ordering

```go
// 依然在 func (pp *partitionProducer) dispatch 中

        } else if pp.highWatermark > 0 {
            // we are retrying something (else highWatermark would be 0) but this message is not a *new* retry level
            if msg.retries < pp.highWatermark {
                // in fact this message is not even the current retry level, so buffer it for now (unless it's a just a fin)
                if msg.flags&fin == fin {
                    pp.retryState[msg.retries].expectChaser = false
                    pp.parent.inFlight.Done() // this fin is now handled and will be garbage collected
                } else {
                // 新消息暂存在buffer中
                    pp.retryState[msg.retries].buf = append(pp.retryState[msg.retries].buf, msg)
                }
                continue

        
        // 重试的消息会发往新开启的broker中
```

- the brokerProducer sees the chaser message; it clears the flag it originally sent, and "retries" the chaser message

```go
// 此处逻辑位于 func (bp *brokerProducer) run 方法
            if reason := bp.needsRetry(msg); reason != nil {
                // ”重发”
                bp.parent.retryMessage(msg, reason)
                // clear flag
                if bp.closing == nil && msg.flags&fin == fin {
                    // we were retrying this partition but we can start processing again
                    delete(bp.currentRetries[msg.Topic], msg.Partition)
                    Logger.Printf("producer/broker/%d state change to [closed] on %s/%d\n",
                        bp.broker.ID(), msg.Topic, msg.Partition)
                }

                continue
            }
```

- the partitionProducer sees the retried chaser message (indicating that it has seen the last retried message)  这是最后一条 retried msg

- the partitionProducer flushes the backlog of "new" messages to the new broker and resumes normal processing

```go
// 此处逻辑在 func (pp *partitionProducer) dispatch 中
            } else if msg.flags&fin == fin {
                // this message is of the current retry level (msg.retries == highWatermark) and the fin flag is set,
                // meaning this retry level is done and we can go down (at least) one level and flush that
                pp.retryState[pp.highWatermark].expectChaser = false
                pp.flushRetryBuffers()
                pp.parent.inFlight.Done() // this fin is now handled and will be garbage collected
                continue
            }

// 是怎么刷缓存的消息的            
func (pp *partitionProducer) flushRetryBuffers() {
    Logger.Printf("producer/leader/%s/%d state change to [flushing-%d]\n", pp.topic, pp.partition, pp.highWatermark)
    for {
        pp.highWatermark--

        if pp.brokerProducer == nil {
            if err := pp.updateLeader(); err != nil {
                pp.parent.returnErrors(pp.retryState[pp.highWatermark].buf, err)
                goto flushDone
            }
            Logger.Printf("producer/leader/%s/%d selected broker %d\n", pp.topic, pp.partition, pp.leader.ID())
        }
        // 可以看到是一层一层开始刷的
        // 从最外层（即重试次数最多的消息）开始刷起
        for _, msg := range pp.retryState[pp.highWatermark].buf {
            pp.brokerProducer.input <- msg
        }

    flushDone:
        pp.retryState[pp.highWatermark].buf = nil
        if pp.retryState[pp.highWatermark].expectChaser {
            Logger.Printf("producer/leader/%s/%d state change to [retrying-%d]\n", pp.topic, pp.partition, pp.highWatermark)
            break
        } else if pp.highWatermark == 0 {
            Logger.Printf("producer/leader/%s/%d state change to [normal]\n", pp.topic, pp.partition)
            break
        }
    }
}
```

> 同步发送与异步发送的区别？

和一般的理解不一样，`同步发送`并不代表消息是同步发出去的，`异步发送`也只是提供了一个异步的表象。查看源码可以知道，消息的异步同步与否，最关键的是看`RequiredAcks`配置的是啥。

`RequiredAcks`有3种选择：

- 0 表示不需要等待kafka server确认   # 完全的异步 
- 1 表示需要等待分区的Leader确认后才可以  # 依然可能丢消息
- -1 表示需要等待分区的所有副本都确认后才可以   # 完全的同步

也就是说，即便你使用了`同步发送`，但`RequiredAcks`配置为0，那么也是可能丢消息的（因为client发送消息时并不关注server的返回）

而若`RequiredAcks`配置为1或者-1，则不论使用同步还是异步，都会”等待“server端的返回。区别是，`同步发送`真的会同步等待返回，而`异步发送`则是给你一个选择，返回是在一个单独的channel里，你感兴趣的话，可以自己读取

#### 可以借鉴学习的代码

从以下代码可以学习它的`重试`机制的实现，在函数内部实现了一个`retry`函数，在发生不需要retry的 error时函数直接return，当遇到需要重试的error时，直接调用`retry`，`retry`里封装了重试的策略以及次数等：

```go
// client.go的tryRefreshMetadata方法
// 作用是刷新 Metadata
func (client *client) tryRefreshMetadata(topics []string, attemptsRemaining int, deadline time.Time) error {
    pastDeadline := func(backoff time.Duration) bool {
        if !deadline.IsZero() && time.Now().Add(backoff).After(deadline) {
            // we are past the deadline
            return true
        }
        return false
    }
    retry := func(err error) error {
        if attemptsRemaining > 0 {
            backoff := client.computeBackoff(attemptsRemaining)
            if pastDeadline(backoff) {
                Logger.Println("client/metadata skipping last retries as we would go past the metadata timeout")
                return err
            }
            Logger.Printf("client/metadata retrying after %dms... (%d attempts remaining)\n", client.conf.Metadata.Retry.Backoff/time.Millisecond, attemptsRemaining)
            if backoff > 0 {
                time.Sleep(backoff)
            }
            return client.tryRefreshMetadata(topics, attemptsRemaining-1, deadline)
        }
        return err
    }

    broker := client.any()
    for ; broker != nil && !pastDeadline(0); broker = client.any() {

        ....
        
        req := &MetadataRequest{Topics: topics, AllowAutoTopicCreation: allowAutoTopicCreation}
        ....
        response, err := broker.GetMetadata(req)
        switch err.(type) {
        case nil:
            allKnownMetaData := len(topics) == 0
            // valid response, use it
            shouldRetry, err := client.updateMetadata(response, allKnownMetaData)
            if shouldRetry {
                Logger.Println("client/metadata found some partitions to be leaderless")
                return retry(err) // note: err can be nil
            }
            return err
        ....

        case KError:
            // if SASL auth error return as this _should_ be a non retryable err for all brokers
            if err.(KError) == ErrSASLAuthenticationFailed {
                Logger.Println("client/metadata failed SASL authentication")
                return err
            }

            if err.(KError) == ErrTopicAuthorizationFailed {
                Logger.Println("client is not authorized to access this topic. The topics were: ", topics)
                return err
            }
            // else remove that broker and try again
            Logger.Printf("client/metadata got error from broker %d while fetching metadata: %v\n", broker.ID(), err)
            _ = broker.Close()
            client.deregisterBroker(broker)

        default:
            // some other error, remove that broker and try again
            Logger.Printf("client/metadata got error from broker %d while fetching metadata: %v\n", broker.ID(), err)
            _ = broker.Close()
            client.deregisterBroker(broker)
        }
    }

    if broker != nil {
        Logger.Printf("client/metadata not fetching metadata from broker %s as we would go past the metadata timeout\n", broker.addr)
        return retry(ErrOutOfBrokers)
    }

    Logger.Println("client/metadata no available broker to send metadata request to")
    client.resurrectDeadBrokers()
    return retry(ErrOutOfBrokers)
}        
```

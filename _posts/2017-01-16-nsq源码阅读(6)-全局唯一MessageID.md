---
layout : post
categories: [源码阅读]
tags : [golang, nsq, 消息队列, mq]
keywords :
excerpt:
sequence: yes
---

客户端发送消息, 进行PUB命令时, nsqd 会使用建立msgid来标识一个msg

```go
//客户端推送消息
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
	//创建 Message 对象
	msg := NewMessage(<-p.ctx.nsqd.idChan, messageBody)

	//topic 发送消息
	err = topic.PutMessage(msg)
}
```

从这看出, MessageID是从一个idChan缓存队列中取出来的; 还是利用chan的思路来减少竞争; idChan的输入端, 也就是id生成器, 是在nsqd启动的时候, 单独起了一个idPump线程进行

```go
func (n *NSQD) Main() {
	n.waitGroup.Wrap(func() {
		n.idPump()
	})
}
```

```go
// 一直生产 序列化 id
func (n *NSQD) idPump() {
	factory := &guidFactory{}
	lastError := time.Unix(0, 0)
	//因为可以做成 nsqd 集群, 所以 msgid 关联 nsq id
	workerID := n.getOpts().ID
	for {
		id, err := factory.NewGUID(workerID)
		if err != nil {
			now := time.Now()
			if now.Sub(lastError) > time.Second {
				// only print the error once/second
				n.logf("ERROR: %s", err)
				lastError = now
			}
			runtime.Gosched()
			continue
		}

		//这里利用了 golang 的chan 满了阻塞的特性,
		//如果n.idChan 这个id缓存池满了, 就阻塞不会继续生产
		select {
		case n.idChan <- id.Hex():
		case <-n.exitChan:
			goto exit
		}
	}

	exit:
	n.logf("ID: closing")
}

func (f *guidFactory) NewGUID(workerID int64) (guid, error) {
	// divide by 1048576, giving pseudo-milliseconds
	ts := time.Now().UnixNano() >> 20

	if ts < f.lastTimestamp {
		return 0, ErrTimeBackwards
	}

	if f.lastTimestamp == ts {
		f.sequence = (f.sequence + 1) & sequenceMask
		if f.sequence == 0 {
			return 0, ErrSequenceExpired
		}
	} else {
		f.sequence = 0
	}

	f.lastTimestamp = ts

	// id = [ 37位ts + ?位 workerId + 12位 sequence ]
	id := guid(((ts - twepoch) << timestampShift) |
		(workerID << workerIDShift) |
		f.sequence)

	if id <= f.lastID {
		return 0, ErrIDBackwards
	}

	f.lastID = id

	return id, nil
}
```
## workerid 的长度

光看这个代码, 就很疑惑, 既然参数 workerID 是 int64类型, 就是64位, 那岂不是在 跟 ts 拼的时候, 当workerid大于某个值的时候, 位数会"溢出", "侵入" 到 ts 的值中? 那岂不是 特定的两个workerid会在特定的 ts 时得到相同的 guid? 搜了下issues, [#592](https://github.com/nsqio/nsq/issues/592) 就有人提出来, workerid 必须 小于 1024, 也就是10位; 而这个是靠启动nsqd的时候检查配置项实现的

```go
if opts.ID < 0 || opts.ID >= 1024 {
	n.logf("FATAL: --worker-id must be [0,1024)")
	os.Exit(1)
}
```

最开始觉得, 这个从代码角度讲, 很不够处女座, guid 中workID 只有10位, 而参数 workID 是 int64 类型 , 不如干脆减少squence 或者 ts的位数, 让workerID是16位使用 int16;

后来仔细想想这个逻辑, ts 是 nano >> 20, 也就是 1 ts = 1048576 nano,  而 1 milliseconds = 1000000 nano; 这就是注释里说的 "giving pseudo-milliseconds" ,ts 近似等于 1毫秒

然后两次生成, 当ts 相同时, 靠 sequence 来区分, sequence 是 12位, 最大值 4096; 那也就是说, 这个生成算法, 同一个workerid, 在1毫秒内的最多能生成 4096个id;

所以, 假如减少 ts的位数或者 sequence 的位数, 让workerid的位数增加, 都会降低生成器的"性能"; 所以这三者的长度是一个权衡

这也是 NewGUID 错误处理中 做一次 runtime.Gosched() 的原因

```go
for {
	id, err := factory.NewGUID(workerID)
	if err != nil {
		// 生成的太快了, 一个ts 中 生成了 4096个 id(sequence)
		// ts + sequence 满了, 休息一会, 让ts 发生改变
		runtime.Gosched()
		continue
	}
}
```

---
layout : post
categories: [源码阅读]
tags : [golang, nsq, 消息队列, mq]
keywords :
excerpt:
sequence: yes
---

上文阅读了nsq是如何接收client连接, 读取请求消息, 执行并返回结果响应的流程. 看到具体的执行命令代码, 其中有一个是"PUB"命令, 也就是消息推送, 现在就来看一下, 一个消息从 被一个client 做PUB发送, 到订阅了消息的Client接收, 在nsqd里是如何流转的

## 1. 客户端 层面 的 PUB 消息

```go
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
	switch {
	case bytes.Equal(params[0], []byte("PUB")):
		return p.PUB(client, params)
}
```

看 PUB方法的具体实现:

```go
//客户端推送消息
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
	var err error

	if len(params) < 2 {
		return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "PUB insufficient number of parameters")
	}

	//请求格式: [ PUB topicName ...]
	topicName := string(params[1])
	if !protocol.IsValidTopicName(topicName) {
		return nil, protocol.NewFatalClientErr(nil, "E_BAD_TOPIC",
			fmt.Sprintf("PUB topic name %q is not valid", topicName))
	}

	//消息体, 在client 请求的下一行开始, 头4个字节是消息体长度
	bodyLen, err := readLen(client.Reader, client.lenSlice)
	if err != nil {
		return nil, protocol.NewFatalClientErr(err, "E_BAD_MESSAGE", "PUB failed to read message body size")
	}

	if bodyLen <= 0 {
		return nil, protocol.NewFatalClientErr(nil, "E_BAD_MESSAGE",
			fmt.Sprintf("PUB invalid message body size %d", bodyLen))
	}

	//长度是否过大
	if int64(bodyLen) > p.ctx.nsqd.getOpts().MaxMsgSize {
		return nil, protocol.NewFatalClientErr(nil, "E_BAD_MESSAGE",
			fmt.Sprintf("PUB message too big %d > %d", bodyLen, p.ctx.nsqd.getOpts().MaxMsgSize))
	}

	//简历缓冲区并读入
	messageBody := make([]byte, bodyLen)
	_, err = io.ReadFull(client.Reader, messageBody)
	if err != nil {
		return nil, protocol.NewFatalClientErr(err, "E_BAD_MESSAGE", "PUB failed to read message body")
	}

	//client 是否有权限对这个 topic 做 PUB 操作
	if err := p.CheckAuth(client, "PUB", topicName, ""); err != nil {
		return nil, err
	}

	topic := p.ctx.nsqd.GetTopic(topicName)

	//创建 Message 对象, TODO: nsqd 的 唯一id 发生器
	msg := NewMessage(<-p.ctx.nsqd.idChan, messageBody)

	//topic 发送消息
	err = topic.PutMessage(msg)
	if err != nil {
		return nil, protocol.NewFatalClientErr(err, "E_PUB_FAILED", "PUB failed "+err.Error())
	}

	return okBytes, nil
}
```

## 2. Topic 层的 PutMessage

PUB 方法做一系列检查, 然后调用 topic.PutMessage(msg) 做具体的发送

```go
func (t *Topic) PutMessage(m *Message) error {
	//TODO 这里是个 ReadLock, 应该是为了锁住 exitFlag ?
	t.RLock()
	defer t.RUnlock()
	//这里使用了一个 atomic Int32 类型的 exitFlag 退出标志, 如果已经退出了就不在 put 了
	if atomic.LoadInt32(&t.exitFlag) == 1 {
		return errors.New("exiting")
	}
	//发送
	err := t.put(m)
	if err != nil {
		return err
	}
	//做个计数
	atomic.AddUint64(&t.messageCount, 1)
	return nil
}
```

```go
func (t *Topic) put(m *Message) error {
	// 这里巧妙利用了 chan 的特性
	// 先写入 memoryMsgChan 这个队列,假如 memoryMsgChan 已满, 不可写入
	// golang 就会执行 default 语句,
	select {
	case t.memoryMsgChan <- m: //'变量' 不都是内存? 这个 '内存', 是相对于 下面这个 DiskQueue 来讲的)
	default:

		// 利用 sync.Pool 包 做了一个 bytes.Buffer 的缓存池, 从池中取出一个来用
		// <<go语言的官方包sync.Pool的实现原理和适用场景>> :
		// http://blog.csdn.net/yongjian_lian/article/details/42058893
		b := bufferPoolGet()

		// 这个 t.backend, 在NewTopic 函数中初始化, 它具体是一个 DiskQueue
		//TODO 先不去管他的具体实现, 暂时认识它是把消息 写入硬盘 就好
		//t.backend = newDiskQueue(topicName,
		//	ctx.nsqd.getOpts().DataPath,
		//	ctx.nsqd.getOpts().MaxBytesPerFile,
		//	int32(minValidMsgLength),
		//	int32(ctx.nsqd.getOpts().MaxMsgSize)+minValidMsgLength,
		//	ctx.nsqd.getOpts().SyncEvery,
		//	ctx.nsqd.getOpts().SyncTimeout,
		//	ctx.nsqd.getOpts().Logger)
		err := writeMessageToBackend(b, m, t.backend)

		// 放回缓存池
		bufferPoolPut(b)
		t.ctx.nsqd.SetHealth(err)
		if err != nil {
			t.ctx.nsqd.logf(
				"TOPIC(%s) ERROR: failed to write message to backend - %s",
				t.name, err)
			return err
		}
	}
	return nil
}
```

Message 的数据流看代码, 到这个地方断掉了; 不过既然使用了 chan , 很明显 chan的 推和拉不在一个线程中

用IDE 对 t.memoryMsgChan 做 'Find Usages' 操作, 很快定位到了 '<-memoryMsgChan' 操作在 Topic 类的 messagePump 函数中

```go
// 每个 topic 自己有一个 messagePump 消息投递 loop
func (t *Topic) messagePump() {

	//避免 锁竞争, 所以缓存 已存在的 channel
	t.RLock()
	for _, c := range t.channelMap {
		chans = append(chans, c)
	}
	t.RUnlock()

	for {
		select {
		case msg = <-memoryMsgChan:
		case <-t.channelUpdateChan:
			//之前 避免 锁竞争, 缓存了 已存在的 channel
			//假如更新了 channel 会发一个 消息过来, 重新读取 channel
			chans = chans[:0]
			t.RLock()
			for _, c := range t.channelMap {
				chans = append(chans, c)
			}
			t.RUnlock()
			continue
		}

		// 这里需要 遍历 t.channelMap, 普通的写法, 会导致锁竞争太大
		for i, channel := range chans {
			chanMsg := msg
			// copy the message because each channel
			// needs a unique instance but...
			// fastpath to avoid copy if its the first channel
			// (the topic already created the first copy)
			if i > 0 {
				chanMsg = NewMessage(msg.ID, msg.Body)
				chanMsg.Timestamp = msg.Timestamp
				chanMsg.deferred = msg.deferred
			}
			if chanMsg.deferred != 0 {
				channel.StartDeferredTimeout(chanMsg, chanMsg.deferred)
				continue
			}
			err := channel.PutMessage(chanMsg)
			if err != nil {
				t.ctx.nsqd.logf(
					"TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
					t.name, msg.ID, channel.name, err)
			}
		}
	}
}
```

这个 messagePump 函数是 在 NewTopic() 也就是创建 Topic 的时候启动的一个线程

也就是说, topic 是自己 负责 把自己的 memoryMsgChan 做 Pump 出来 , 再塞到 二级 channel 的 memoryMsgChan 里

```go
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
	t := &Topic{
		...
	}
	...
	t.waitGroup.Wrap(func() { t.messagePump() })
}
```

## 3. Channel 层的 PutMessage

topic 的 put 调用 topic下含有的所有 channel 的 PutMessage;

channel 跟topic 其实是一样的, 可以这么理解: channel 就是二级topic

```go
func (c *Channel) PutMessage(m *Message) error {
	c.RLock()
	defer c.RUnlock()
	if atomic.LoadInt32(&c.exitFlag) == 1 {
		return errors.New("exiting")
	}
	err := c.put(m)
	if err != nil {
		return err
	}
	atomic.AddUint64(&c.messageCount, 1)
	return nil
}

func (c *Channel) put(m *Message) error {
	select {
	case c.memoryMsgChan <- m:
	default:
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, c.backend)
		bufferPoolPut(b)
		c.ctx.nsqd.SetHealth(err)
		if err != nil {
			c.ctx.nsqd.logf("CHANNEL(%s) ERROR: failed to write message to backend - %s",
				c.name, err)
			return err
		}
	}
	return nil
}
```

## 4. Client 层的 messagePump
但是会发现, Channel 自己并没有 messagePump 函数, 再次通过Usage 查找, <-channel.memoryMsgChan 读取操作在 protocolV2.messagePump(), 这也就是上文中, 每个Client连接的处理函数IOLoop, 每个Client 都会有一个 protocolV2.messagePump() 线程

```go
func (p *protocolV2) IOLoop(conn net.Conn) error {

	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

	for {
		line, err = client.Reader.ReadSlice('\n')   // 读取
		params := bytes.Split(line, separatorBytes) // 解析
		response, err = p.Exec(client, params)		// 执行

		err = p.Send(client, frameTypeResponse, response) //发送结果
	}
}
```

客户端的其他操作我们先忽略不管, 主要看 Message 的数据路劲, 精简掉 messagePump 的代码:

```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	for {
		if subChannel == nil || !client.IsReadyForMessages() {
		} else {
			// we're buffered (if there isn't any more data we should flush)...
			// select on the flusher ticker channel, too
			memoryMsgChan = subChannel.memoryMsgChan
			backendMsgChan = subChannel.backend.ReadChan()
			flusherChan = outputBufferTicker.C
		}

		// 这里负责执行Client 的各种事件
		select {

		//Client 需要发送一个SUB 请求 来订阅Channel, 并切一个Client只能订阅一个Channel
		case subChannel = <-subEventChan:  // 做了订阅
			subEventChan = nil
		case msg := <-memoryMsgChan:
			//TODO 样本率? 做测试用的吗?
			if sampleRate > 0 && rand.Int31n(100) > sampleRate {
				continue
			}
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			// protocol 进行消息格式的打包, 再发送给Client
			// 这里, Message 就发送给了 client
			err = p.SendMessage(client, msg, &buf)
			if err != nil {
				goto exit
			}
			flushed = false
		case <-client.ExitChan:
			goto exit
		}
	}
}
```

## 5. 整体 Message 数据流 精简总结:

总体来讲, 一个Message的推送, 经过了 两个chan 队列:

* topic.memoryMsgChan
* channel.memoryMsgChan

两个队列的两头(读, 写), 分别有一个线程进行, 也就是3个线程:

* protocol.IOLoop(clientConn)
* topic.messagePump()
* protocol.messagePump()

Message 的数据流就是:

<div class="sequence">
ClientConn->topic.memoryMsgChan: protocol.IOLoop(clientConn)
topic.memoryMsgChan->channel.memoryMsgChan: topic.messagePump()
channel.memoryMsgChan-->ClientConn: protocol.messagePump()
</div>

从client连接进来, 到message 投递给Client 的所有流程代码, 精简如下:

```go
clientConn, err := listener.Accept()
// Client  ==>  Client 请求响应线程  ==> topic.memoryMsgChan
go handler.Handle(clientConn) {
	err = prot.IOLoop(clientConn){
		for {
			line, err = client.Reader.ReadSlice('\n') // <==Client
			params := bytes.Split(line, separatorBytes)
			response, err = p.Exec(client, params){
				switch {
				case bytes.Equal(params[0], []byte("PUB")):
					return p.PUB(client, params){
						topic.PutMessage(msg)	// ==>topic.memoryMsgChan
					}
			}		
		}
	}
}

// topic.memoryMsgChan  ==>  Topic.messagePump 线程  ==> echa channel.memoryMsgChan
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
	t.waitGroup.Wrap(func() {t.messagePump(){
		for {
			select {
			case msg = <-memoryMsgChan:    // <==topic.memoryMsgChan
			for i, channel := range chans {
				err := channel.PutMessage(chanMsg) // ==>channel.memoryMsgChan
			}
		}}
	})
}

// client.subChannel.memoryMsgChan ==>  Client的messagePump 线程  ==> Client
func (p *protocolV2) IOLoop(conn net.Conn) error {
	go p.messagePump(client, messagePumpStartedChan){
		for {
			select {
			case subChannel = <-subEventChan:
			case msg := <-subChannel.memoryMsgChan: // <==client.subChannel.memoryMsgChan
				err = p.SendMessage(client, msg, &buf) // ==>Client
			}
		}
	}
}

```

虽然 一个channel 会被多个Client 订阅, 从代码上来讲, 也就是 一个 channel.memoryMsgChan 会被多个 client 线程的 messagePump() 方法中进行读取

但因为chan 的特性, 一条缓存在 channel.memoryMsgChan 中的消息, 只会被一个 client 读取到

## topic 遍历 channel时 做的缓存

topic 类中 messagePump 需要遍历 所有的channel, 也就是遍历 t.channelMap; 多线程中遍历类中一个成员变量的操作, 需要给 类加锁, 普通的写法像这样:

```go
t.RLock()
for i, channel := range t.channelMaps {

}
t.RUnlock()
```

但是问题这里是在一个 for 循环中, 整个操作就相当于这样:

```go
for {
	t.RLock()
	for i, channel := range t.channelMaps {

	}
	t.RUnlock()
}
```

这个锁范围太大了, 锁竞争很大, 所以nsq 做了一个缓存, 并且接收 t.channelUpdateChan 通知来 更新 缓存, 就像这样:

```go
func (t *Topic) messagePump() {
	var chans []*Channel

	//避免 锁竞争, 所以缓存 已存在的 channel
	t.RLock()
	for _, c := range t.channelMap {
		chans = append(chans, c)
	}
	t.RUnlock()

	for {
		select {
		case <-t.channelUpdateChan:
			//接到channle 变更的消息, 刷新缓存
			chans = chans[:0]
			t.RLock()
			for _, c := range t.channelMap {
				chans = append(chans, c)
			}
			t.RUnlock()
			if len(chans) == 0 || t.IsPaused() {
				memoryMsgChan = nil
				backendChan = nil
			} else {
				memoryMsgChan = t.memoryMsgChan
				backendChan = t.backend.ReadChan()
			}
			continue
		}

		// 遍历 已经缓存的 chans
		for i, channel := range chans {
		}
	}
}
```

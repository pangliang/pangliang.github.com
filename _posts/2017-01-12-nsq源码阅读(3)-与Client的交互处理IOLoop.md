---
layout : post
categories: [源码阅读]
tags : [golang, nsq, 消息队列, mq]
keywords :
excerpt:
---

## 具体 每个'命令'处理的封装

前面看到, listener.Accept() 之后, 解析了版本号, 当前由 protocol_v2 具体实现了 IOLoop

```go
func (p *protocolV2) IOLoop(conn net.Conn) error {
	var err error
	var line []byte
	var zeroTime time.Time

	// nsqd 范围的 clientID 序列
	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	client := newClientV2(clientID, conn, p.ctx)

	// 这个messagePump 名字很形象, 把要发送给client 的messgae 从 缓存池子里Pump 抽出来 做具体的发送
	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan  //TODO messagePump 里要做初始化, 具体是什么? 为什么需要这样阻塞一下IOLoop ?

	// 底下这个地方是 处理 client 的请求的
	for {
		// 如果 client 设置需要做心跳检查, 则设置 读超时为 两倍 心跳检查间隔
		// 也就是说, 在这个时间里, 正常情况下, client肯定会在这个间隔内发一个心跳包过来
		// read 不会因为client 本来就没消息而超时,
		// 但是如果还是超时了, 那就肯定是 网络连接除了问题
		if client.HeartbeatInterval > 0 {
			client.SetReadDeadline(time.Now().Add(client.HeartbeatInterval * 2))
		} else {
			// 加入client 不设置做心跳则不设置读超时
			client.SetReadDeadline(zeroTime)
		}

		//TODO 没太懂这个什么意思.
		// ReadSlice does not allocate new space for the data each request
		// ie. the returned slice is only valid until the next call to it
		line, err = client.Reader.ReadSlice('\n')
		if err != nil {
			if err == io.EOF {
				err = nil
			} else {
				err = fmt.Errorf("failed to read command - %s", err)
			}
			break
		}

		// 这个V2 版本的协议, 一行是一个命令
		line = line[:len(line)-1]
		// optionally trim the '\r' , 处理一下\r, 有可能win版本的回车是 \r\n
		if len(line) > 0 && line[len(line)-1] == '\r' {
			line = line[:len(line)-1]
		}
		// 每个命令, 用 " "空格来划分 参数
		params := bytes.Split(line, separatorBytes)

		if p.ctx.nsqd.getOpts().Verbose {
			p.ctx.nsqd.logf("PROTOCOL(V2): [%s] %s", client, params)
		}

		// 执行命令
		var response []byte
		response, err = p.Exec(client, params)
		if err != nil {
			ctx := ""
			if parentErr := err.(protocol.ChildErr).Parent(); parentErr != nil {
				ctx = " - " + parentErr.Error()
			}
			p.ctx.nsqd.logf("ERROR: [%s] - %s%s", client, err, ctx)

			sendErr := p.Send(client, frameTypeError, []byte(err.Error()))
			if sendErr != nil {
				p.ctx.nsqd.logf("ERROR: [%s] - %s%s", client, sendErr, ctx)
				break
			}

			// errors of type FatalClientErr should forceably close the connection
			if _, ok := err.(*protocol.FatalClientErr); ok {
				break
			}
			continue
		}

		// 加入命令是有 '响应' 的, 发送响应
		if response != nil {
			err = p.Send(client, frameTypeResponse, response)
			if err != nil {
				err = fmt.Errorf("failed to send response - %s", err)
				break
			}
		}
	}

	p.ctx.nsqd.logf("PROTOCOL(V2): [%s] exiting ioloop", client)
	conn.Close()

	// 通知 messagePump 退出
	close(client.ExitChan)
	if client.Channel != nil {
		client.Channel.RemoveClient(client.ID)
	}

	return err
}

func (p *protocolV2) Send(client *clientV2, frameType int32, data []byte) error {
	// 因为client 的处理 还有一个 messagePump 线程, 所以 发送要锁
	client.writeLock.Lock()

	var zeroTime time.Time
	if client.HeartbeatInterval > 0 {
		client.SetWriteDeadline(time.Now().Add(client.HeartbeatInterval))
	} else {
		client.SetWriteDeadline(zeroTime)
	}

	//V2 协议版本 发送给client, 是 使用 [(4byte)消息长度 , (4byte)消息类型, (载体)] 的 帧格式
	//但是为什么这个 格式的封装不是 写在 protocal_v2.go 而是在 protocaol 定义上?
	//个人觉得, 具体封包格式的 '具体实现' 应该在 '协议的具体实现'里,  也就是 protocal_v2, 而不是 '协议的定义' 里
	_, err := protocol.SendFramedResponse(client.Writer, frameType, data)
	if err != nil {
		client.writeLock.Unlock()
		return err
	}

	if frameType != frameTypeMessage {
		err = client.Flush()
	}

	client.writeLock.Unlock()

	return err
}

func SendFramedResponse(w io.Writer, frameType int32, data []byte) (int, error) {
	beBuf := make([]byte, 4)
	size := uint32(len(data)) + 4

	binary.BigEndian.PutUint32(beBuf, size)
	n, err := w.Write(beBuf)
	if err != nil {
		return n, err
	}

	binary.BigEndian.PutUint32(beBuf, uint32(frameType))
	n, err = w.Write(beBuf)
	if err != nil {
		return n + 4, err
	}

	n, err = w.Write(data)
	return n + 8, err
}
```

读取并解析请求之后, 传给 p.Exec(client, params) 执行具体的请求

```go
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
	//这个命令不需要做认证
	if bytes.Equal(params[0], []byte("IDENTIFY")) {
		return p.IDENTIFY(client, params)
	}
	// client 是否做了认证
	err := enforceTLSPolicy(client, p, params[0])
	if err != nil {
		return nil, err
	}
	//一行一个请求, 一个命令中用 " " 空格分割参数, 第一个参数是 命令标志
	switch {
	case bytes.Equal(params[0], []byte("FIN")):
		return p.FIN(client, params)
	case bytes.Equal(params[0], []byte("RDY")):
		return p.RDY(client, params)
	case bytes.Equal(params[0], []byte("REQ")):
		return p.REQ(client, params)
	case bytes.Equal(params[0], []byte("PUB")):
		return p.PUB(client, params)
	case bytes.Equal(params[0], []byte("MPUB")):
		return p.MPUB(client, params)
	case bytes.Equal(params[0], []byte("DPUB")):
		return p.DPUB(client, params)
	case bytes.Equal(params[0], []byte("NOP")):
		return p.NOP(client, params)
	case bytes.Equal(params[0], []byte("TOUCH")):
		return p.TOUCH(client, params)
	case bytes.Equal(params[0], []byte("SUB")):
		return p.SUB(client, params)
	case bytes.Equal(params[0], []byte("CLS")):
		return p.CLS(client, params)
	case bytes.Equal(params[0], []byte("AUTH")):
		return p.AUTH(client, params)
	}
	return nil, protocol.NewFatalClientErr(nil, "E_INVALID", fmt.Sprintf("invalid command %s", params[0]))
}
```

### 精简总流程:

同样, 代码精简一下, IOLoop 里主要就是启动了两个线程, 一个处理订阅的消息的发送, 一个处理client 发过来的命令请求

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

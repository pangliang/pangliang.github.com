---
layout : post
categories: [源码阅读]
tags : [golang, nsq, 消息队列, mq]
keywords :
excerpt:
---

## TCPHandler

根据上文, 我们直接看 nsqd.Main()

```go
func (n *NSQD) Main() {
	...

	// 启动tcp listener
	tcpListener, err := net.Listen("tcp", n.getOpts().TCPAddress)

	//实现了具体的Handler
	tcpServer := &tcpServer{ctx: ctx}

	//上一篇说的, 用以优雅退出
	n.waitGroup.Wrap(func() {
		protocol.TCPServer(n.tcpListener, tcpServer, n.getOpts().Logger)
	})

	...
}
```

nsq 封装了一个TCPHandler, 当 listener.Accept() 接到client 的连接获取到connect 时, 交给 TCPHandler.Handle(net.Conn) 函数处理

所有的 tcp 服务都可以复用此代码, 只需要实现不同的TCPHandler即可

```go

type TCPHandler interface {
	Handle(net.Conn)
}

func TCPServer(listener net.Listener, handler TCPHandler, l app.Logger) {
	l.Output(2, fmt.Sprintf("TCP: listening on %s", listener.Addr()))

	for {
		// 有客户端连接
		clientConn, err := listener.Accept()
		if err != nil {
			if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
				l.Output(2, fmt.Sprintf("NOTICE: temporary Accept() failure - %s", err))
				runtime.Gosched()       // 是临时的错误, 暂停一下继续
				continue
			}
			// theres no direct way to detect this error because it is not exposed
			if !strings.Contains(err.Error(), "use of closed network connection") {
				l.Output(2, fmt.Sprintf("ERROR: listener.Accept() - %s", err))
			}
			break
		}
		//启动一个线程, 交给 handler 处理, 这里使用的是 one connect per thread 模式
		//因为golang的特性, one connect per thread 模式 实际上是  one connect per goroutine
		go handler.Handle(clientConn)
	}

	l.Output(2, fmt.Sprintf("TCP: closing %s", listener.Addr()))
}
```

## 协议不同版本 的 封装处理



```go
type Protocol interface {
	IOLoop(conn net.Conn) error
}
```

```go

func (p *tcpServer) Handle(clientConn net.Conn) {
	p.ctx.nsqd.logf("TCP: new client(%s)", clientConn.RemoteAddr())

	// 先读取4个字节 作为 协议版本号
	buf := make([]byte, 4)
	_, err := io.ReadFull(clientConn, buf)
	if err != nil {
		p.ctx.nsqd.logf("ERROR: failed to read protocol version - %s", err)
		return
	}
	protocolMagic := string(buf)

	p.ctx.nsqd.logf("CLIENT(%s): desired protocol magic '%s'",
		clientConn.RemoteAddr(), protocolMagic)

	var prot protocol.Protocol
	switch protocolMagic {
	case "  V2":
		// protocolV2 实现了 V2 这个版本的协议
		prot = &protocolV2{ctx: p.ctx}
	//假如有另外一个版本的协议
	//case "  V3":
	//	prot = &protocolV3{ctx: p.ctx}
	default:
		protocol.SendFramedResponse(clientConn, frameTypeError, []byte("E_BAD_PROTOCOL"))
		clientConn.Close()
		p.ctx.nsqd.logf("ERROR: client(%s) bad protocol magic '%s'",
			clientConn.RemoteAddr(), protocolMagic)
		return
	}

	//交给具体实现处理
	err = prot.IOLoop(clientConn)
	if err != nil {
		p.ctx.nsqd.logf("ERROR: client(%s) - %s", clientConn.RemoteAddr(), err)
		return
	}
}
```

### 精简总流程:

```go
func (n *NSQD) Main() {
	tcpListener, err := net.Listen("tcp", n.getOpts().TCPAddress)
	tcpServer := &tcpServer{ctx: ctx}
	n.waitGroup.Wrap(func() {
		protocol.TCPServer(n.tcpListener, tcpServer, n.getOpts().Logger)
	})
}

func TCPServer(listener net.Listener, handler TCPHandler, l app.Logger) {
	for {
		clientConn, err := listener.Accept()
		go handler.Handle(clientConn)
	}
}

func (p *tcpServer) Handle(clientConn net.Conn) {
	buf := make([]byte, 4)
	_, err := io.ReadFull(clientConn, buf)
	protocolMagic := string(buf)

	var prot protocol.Protocol
	switch protocolMagic {
	case "  V2":
		prot = &protocolV2{ctx: p.ctx}
	}

	err = prot.IOLoop(clientConn)
}
```

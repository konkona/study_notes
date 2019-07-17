### 客户端发起一个请求SayHello

```go
func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

// 看起来好像比较简单，new一个stream，然后开始发送和接收消息
func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
	cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
	if err != nil {
		return err
	}
	if err := cs.SendMsg(req); err != nil {
		return err
	}
	return cs.RecvMsg(reply)
}
```

```go
// 创建一个clientStream
func newClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, opts ...CallOption) (_ ClientStream, err error) {
	callHdr := &transport.CallHdr{
		Host:           cc.authority,
		Method:         method,
		ContentSubtype: c.contentSubtype,
	}

	cs := &clientStream{
		callHdr:      callHdr,
		ctx:          ctx,
		methodConfig: &mc,
		opts:         opts,
		callInfo:     c,
		cc:           cc,
		desc:         desc,
		codec:        c.codec,
		cp:           cp,
		comp:         comp,
		cancel:       cancel,
		beginTime:    beginTime,
		firstAttempt: true,
	}
	
	// cs 通过clientConn拿到一个底层的transport(http2client)
	if err := cs.newAttemptLocked(sh, trInfo); err != nil {
		cs.finish(err)
		return nil, err
	}
	
	// new一个transport的stream，用于发送数据
	op := func(a *csAttempt) error { return a.newStream() }
	if err := cs.withRetry(op, func() { cs.bufferForRetryLocked(0, op) }); err != nil {
		cs.finish(err)
		return nil, err
	}	
}

func (cs *clientStream) newAttemptLocked(sh stats.Handler, trInfo traceInfo) error {
	cs.attempt = &csAttempt{
		cs:           cs,
		dc:           cs.cc.dopts.dc,
		statsHandler: sh,
		trInfo:       trInfo,
	}
	t, done, err := cs.cc.getTransport(cs.ctx, cs.callInfo.failFast, cs.callHdr.Method)
	cs.attempt.t = t
	cs.attempt.done = done
	return nil
}

func (a *csAttempt) newStream() error {
	cs := a.cs
	cs.callHdr.PreviousAttempts = cs.numRetries
	s, err := a.t.NewStream(cs.ctx, cs.callHdr)
	if err != nil {
		return toRPCErr(err)
	}
	cs.attempt.s = s
	cs.attempt.p = &parser{r: s}
	return nil
}
```
newClientStream从clientConn拿了一个transport，然后new了一个transport.Stream,通过csAttempt关联起来.现在来看一下NewStream函数

```go
func (t *http2Client) NewStream(ctx context.Context, callHdr *CallHdr) (_ *Stream, err error) {
	s := t.newStream(ctx, callHdr)  // 真正的stream
	hdr := &headerFrame{        // headerFrame
		hf:        headerFields,
		endStream: false,
		initStream: ...,
		onOrphaned: cleanup,
		wq:         s.wq,
	}	
	
	for {
		// 晚点分析这个controlBuf， 现在知道的是这个hdr包被丢到了一个队列里面
		success, err := t.controlBuf.executeAndPut(func(it interface{}) bool {
			if !checkForStreamQuota(it) {
				return false
			}
			if !checkForHeaderListSize(it) {
				return false
			}
			return true
		}, hdr)
		if err != nil {
			return nil, err
		}
		if success {
			break
		}
	}
}

func (t *http2Client) newStream(ctx context.Context, callHdr *CallHdr) *Stream {
	// TODO(zhaoq): Handle uint32 overflow of Stream.id.
	s := &Stream{
		done:           make(chan struct{}),
		method:         callHdr.Method,
		sendCompress:   callHdr.SendCompress,
		buf:            newRecvBuffer(),  // 数据从transport读出来放到这里
		headerChan:     make(chan struct{}),
		contentSubtype: callHdr.ContentSubtype,
	}
	s.wq = newWriteQuota(defaultWriteQuota, s.done) // defaultWriteQuota = 64kB
	s.requestRead = func(n int) {
		t.adjustWindow(s, uint32(n))
	}
	// The client side stream context should have exactly the same life cycle with the user provided context.
	// That means, s.ctx should be read-only. And s.ctx is done iff ctx is done.
	// So we use the original context here instead of creating a copy.
	s.ctx = ctx
	s.trReader = &transportReader{
		reader: &recvBufferReader{ // 最终数据将从这里提取
			ctx:     s.ctx,
			ctxDone: s.ctx.Done(),
			recv:    s.buf,
		},
		windowHandler: func(n int) {
			t.updateWindow(s, uint32(n))
		},
	}
	return s
}
```

newClientStream做的事情比较简单，就是获取底层transport，新起一个stream，发送一个headFrame到transport的controlBuf中。接下来开始SendMsg

```go
func (cs *clientStream) SendMsg(m interface{}) (err error) {
	data, err := encode(cs.codec, m)
	hdr, payload := msgHeader(data, compData)  // headerLen  = payloadLen(1) + sizeLen(4)
	op := func(a *csAttempt) error {
		err := a.sendMsg(m, hdr, payload, data)
		// nil out the message and uncomp when replaying; they are only needed for
		// stats which is disabled for subsequent attempts.
		m, data = nil, nil
		return err
	}	
}

func (a *csAttempt) sendMsg(m interface{}, hdr, payld, data []byte) error {
	err := a.t.Write(a.s, hdr, payld, &transport.Options{Last: !cs.desc.ClientStreams})
}

func (t *http2Client) Write(s *Stream, hdr []byte, data []byte, opts *Options) error {
	df := &dataFrame{
		streamID:  s.id,
		endStream: opts.Last,
	}
	if hdr != nil || data != nil { // If it's not an empty data frame.
		// Add some data to grpc message header so that we can equally
		// distribute bytes across frames.
		emptyLen := http2MaxFrameLen - len(hdr)
		if emptyLen > len(data) {
			emptyLen = len(data)
		}
		hdr = append(hdr, data[:emptyLen]...)
		data = data[emptyLen:]
		df.h, df.d = hdr, data
		// TODO(mmukhi): The above logic in this if can be moved to loopyWriter's data handler.
		if err := s.wq.get(int32(len(hdr) + len(data))); err != nil {
			return err
		}
	}
	return t.controlBuf.put(df)  // ok, 这里又丢了一个dataFrame到controlBuf中
}
```

```go
func (cs *clientStream) RecvMsg(m interface{}) error {
	err := cs.withRetry(func(a *csAttempt) error {
		return a.recvMsg(m)
	}, cs.commitAttemptLocked)
	return err
}

func (a *csAttempt) recvMsg(m interface{}) (err error) {
	// 第一次recv获取服务器的response a.p 就是对transport的stream封装
	err = recv(a.p, cs.codec, a.s, a.dc, m, *cs.callInfo.maxReceiveMessageSize, inPayload, a.decomp)
	// 在recv拿到stream数据之后，通过cs.codec unMarshall，就得到了我们的response
	
	// 第二次recv ，必须是EOF，然后返回
	err = recv(a.p, cs.codec, a.s, a.dc, m, *cs.callInfo.maxReceiveMessageSize, nil, a.decomp)
}
```

RecvMsg函数返回后，业务层就拿到了server的response，通过上面的客户端创建stream到sendMsg到recvMsg来分析，
重要的数据就是底层的transport和stream，其中有一个controlBuf用来做流控的，稍后再看，。
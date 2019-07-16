### 从最开始的建立连接开始
```go
func Dial(target string, opts ...DialOption) (*ClientConn, error) {
	return DialContext(context.Background(), target, opts...)
}

func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target:         target,
		csMgr:          &connectivityStateManager{},
		conns:          make(map[*addrConn]struct{}),
		dopts:          defaultDialOptions(),
		blockingpicker: newPickerWrapper(),
		czData:         new(channelzData),
	}
	/* ... */
}	
```

client初始建立连接，通过DialContext函数，返回了一个ClientConn的结构，先来看一下这个结构定义:

![ClientConn](../images/ClientConn.png)

```go
	if cc.dopts.copts.Dialer == nil {
		cc.dopts.copts.Dialer = newProxyDialer(
			func(ctx context.Context, addr string) (net.Conn, error) {
				network, addr := parseDialTarget(addr)
				return dialContext(ctx, network, addr)
			},
		)
	}
````
设置transport的dialer函数，用于底层tcp发起连接。

```go
	if cc.dopts.resolverBuilder == nil {
		// Only try to parse target when resolver builder is not already set.
		cc.parsedTarget = parseTarget(cc.target)
		grpclog.Infof("parsed scheme: %q", cc.parsedTarget.Scheme)
		// 例子里我们的target是"localhost:50051"， scheme是nil
		cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		if cc.dopts.resolverBuilder == nil {
			// If resolver builder is still nil, the parse target's scheme is
			// not registered. Fallback to default resolver and set Endpoint to
			// the original unparsed target.
			grpclog.Infof("scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)
			cc.parsedTarget = resolver.Target{
				Scheme:   resolver.GetDefaultScheme(), // "passthrough"  default scheme
				Endpoint: target,
			}
			cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme) // 这里获取到的builder是passthroughBuilder
		}
	} else {
		cc.parsedTarget = resolver.Target{Endpoint: target}
	}

	// Build the resolver.  重点看一下这个ResolverWrapper
	cc.resolverWrapper, err = newCCResolverWrapper(cc)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}
	// Start the resolver wrapper goroutine after resolverWrapper is created.
	//
	// If the goroutine is started before resolverWrapper is ready, the
	// following may happen: The goroutine sends updates to cc. cc forwards
	// those to balancer. Balancer creates new addrConn. addrConn fails to
	// connect, and calls resolveNow(). resolveNow() tries to use the non-ready
	// resolverWrapper.
	cc.resolverWrapper.start()
	
```

```go
func newCCResolverWrapper(cc *ClientConn) (*ccResolverWrapper, error) {
	rb := cc.dopts.resolverBuilder // 这里就是上面默认的passthroughBuilder
	if rb == nil {
		return nil, fmt.Errorf("could not get resolver for scheme: %q", cc.parsedTarget.Scheme)
	}

	ccr := &ccResolverWrapper{
		cc:     cc,
		addrCh: make(chan []resolver.Address, 1),
		scCh:   make(chan string, 1),
		done:   make(chan struct{}),
	}

	var err error
	ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, resolver.BuildOption{DisableServiceConfig: cc.dopts.disableServiceConfig})
	if err != nil {
		return nil, err
	}
	return ccr, nil
}

// 看一下passthroughBuilder.Build的实现 
func (*passthroughBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
	r := &passthroughResolver{
		target: target,
		cc:     cc,
	}
	r.start()
	return r, nil
}

// r.cc 其实是ccResolverWrapper， 他集成自resolver.ClientConn
func (r *passthroughResolver) start() {
	r.cc.NewAddress([]resolver.Address{{Addr: r.target.Endpoint}})
}

// NewAddress is called by the resolver implemenetion to send addresses to gRPC.
// 继续跟踪下来，发现NewAddress函数其实是把连接的地址丢给了addrCh channel
func (ccr *ccResolverWrapper) NewAddress(addrs []resolver.Address) {
	select {
	case <-ccr.addrCh:
	default:
	}
	ccr.addrCh <- addrs
}
```

通过上面的分析，发现newCCResolverWrapper函数主要就是把地址丢给ccResolverWrapper的addrCh
channel.接下来继续分析，该channel什么时候被读取？
看一下cc.resolverWrapper.start() 实现

```go
func (ccr *ccResolverWrapper) start() {
	go ccr.watcher()
}

// watcher processes address updates and service config updates sequentially.
// Otherwise, we need to resolve possible races between address and service
// config (e.g. they specify different balancer types).
func (ccr *ccResolverWrapper) watcher() {
	for {
		select {
		case <-ccr.done:
			return
		default:
		}

		select {
		case addrs := <-ccr.addrCh: // addrCh channel在这被读取
			select {
			case <-ccr.done:
				return
			default:
			}
			grpclog.Infof("ccResolverWrapper: sending new addresses to cc: %v", addrs)
			// 这里调用ClientConn的handleResolvedAddrs函数，将addrs传进去.
			ccr.cc.handleResolvedAddrs(addrs, nil)
		case sc := <-ccr.scCh:
			select {
			case <-ccr.done:
				return
			default:
			}
			grpclog.Infof("ccResolverWrapper: got new service config: %v", sc)
			ccr.cc.handleServiceConfig(sc)
		case <-ccr.done:
			return
		}
	}
}

// ccResolverWrapper.watcher()函数拿到地址之后，直接传给了ClientConn，我们继续看handleResolvedAddrs函数
func (cc *ClientConn) handleResolvedAddrs(addrs []resolver.Address, err error) {
        if newBalancerName == "" {
            newBalancerName = PickFirstBalancerName
        }
        cc.switchBalancer(newBalancerName)
        cc.balancerWrapper.handleResolvedAddrs(addrs, nil)
}        
```
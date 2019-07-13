# gRPC源码学习

## 工作环境介绍： 
+ 系统：mac osx
+ golang版本：1.11.2
+ gRPC版本：[1.15.0-dev](https://github.com/grpc/grpc-go/tree/v1.15.x)
+ IDE: IntelliJ IDEA


### gRPC是一个高性能、开源和通用的RPC框架，面向移动和HTTP/2设计。

先看一个官方sample，server代码如下： 

```go
const (
	port = ":50051"
)

// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	// Register reflection service on gRPC server.
	reflection.Register(s)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

server端监听在50051端口，启动了一个greeterServer,
并注册到grpc上，提供一个SayHello的方法。

client代码：

```go
const (
	address     = "localhost:50051"
	defaultName = "world"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), 1000*time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)

	time.Sleep(1000*time.Second)
}
```

client端将服务器的ip:port提供给grpc，然后启动一个greeterClient，之后就可以直接调用SayHello与服务器通信了。

看起来调用十分简单，现在开始深入分析一下grpc框架到底做了什么？
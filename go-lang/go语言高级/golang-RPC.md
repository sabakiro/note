
# RPC版 Hello World
## Go语言的RPC规则：
1. 方法只能有两个可序列化的参数，其中第二个参数是指针类型，并且返回一个error类型
2. 同时必须是公开的方法。

##### 快速入门
```go
type HelloService struct{}

func (p *HelloService) Hello(request string, reply *string) error {
	*reply = "hello:" + request
	return nil
}

var ch chan struct{}

func main() {
	ch = make(chan struct{})
	/*---------------客户端------------------------*/
	go func() {
		<-ch
		fmt.Println("执行客户端代码")
		/*
			1.rpc.Dial拨号RPC服务，
			2.然后通过client.Call()调用具体的RPC方法。
				在调用client.Call()时:
					1.第一个参数是用点号链接的RPC服务名字和方法名字，
					2.第二个和第三个参数分别是定义RPC方法的两个参数。
		*/
		client, err := rpc.Dial("tcp", "localhost:1234")
		if err != nil {
			log.Fatal("dialing:", err)
		}
		var reply string
		err = client.Call("HelloService.Hello", "Caillo", &reply)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println(reply)
	}()
	
	/*---------------服务端------------------------*/
	//将HelloService类型的对象注册为一个RPC服务
	rpc.RegisterName("HelloService", new(HelloService))
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}
	ch <- struct{}{}
	conn, err := listener.Accept()
	if err != nil {
		log.Fatal("accept error:", err)
	}
	rpc.ServeConn(conn)
	/*
		rpc.RegisterName()函数调用会将对象类型中所有满足RPC规则
		的对象方法注册为RPC函数，
		所有注册的方法会放在HelloService服务的空间之下。
		然后建立一个唯一的TCP链接，
		并且通过rpc.ServeConn()函数在该TCP链接上为对方提供RPC服务。
	*/
}

```

#### 更安全的RPC接口

######  服务端

```go
import "net/rpc"

/*
接口规范分为3部分：
1.服务的名字，
2.服务要实现的详细方法列表，
3.注册该类型服务的函数。

为了避免名字冲突，所以在RPC服务的名字中增加了包路径前缀
（这个是RPC服务抽象的包路径，并非完全等价于Go语言的包路径）。
RegisterHelloService注册服务时，
编译器会要求传入的对象满足HelloServiceInterface接口。
*/
const HelloServiceName = "pkg/HelloService"

type HelloServiceInterface interface {
	Hello(request string, reply *string) error
}

func RegisterHelloService(svc HelloServiceInterface) error {
	return rpc.RegisterName(HelloServiceName, svc)
}

//实现
type HelloService struct{}

func (hs *HelloService) Hello(request string, reply *string) error {
	*reply = "hello:" + request
	return nil
}

```

###### 客户端
```go
import (
	"fmt"
	"log"
	"net/rpc"
)

type HelloServiceClient struct {
	*rpc.Client
}

func (hc *HelloServiceClient) Hello(request string, reply *string) error {
	err := hc.Client.Call(HelloServiceName+".Hello", request, reply)
	fmt.Println(*reply)
	return err
}

//获取客户端
func DialHelloService(network, address string) (*HelloServiceClient, error) {
	c, err := rpc.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return &HelloServiceClient{Client: c}, nil
}

//开启客户端
func Client() {
	client, err := DialHelloService("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}
	var reply string
	err = client.Hello("ciallo", &reply)
	if err != nil {
		log.Fatal(err)
	}
}

```
###### 执行服务端代码
```go
import (
	"log"
	"net"
	"net/rpc"
	"rpc1/pkg"
)

func main() {
	/*
		RegisterHelloService()函数来注册函数，
		这样不仅可以避免命名服务名称的工作，
		同时也保证了传入的服务对象满足RPC接口的定义。

		最后新的服务改为支持多个TCP链接，
		然后为每个TCP链接提供RPC服务。
	*/
	pkg.RegisterHelloService(new(pkg.HelloService))
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Fatal("Accept error:", err)
		}
		go rpc.ServeConn(conn)
	}
}

```

## 跨语言的RPC
#### 基于JSON编码重新实现RPC服务
###### 服务端
```go
type HelloService struct{}

func (hs *HelloService) Hello(request string, reply *string) error {
	*reply = "ciallo!," + request
	return nil
}

func main() {
	rpc.RegisterName("HelloService", new(HelloService))
	listen, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("listening error:", err)
	}
	for {
		conn, err := listen.Accept()
		if err != nil {
			log.Fatal("accept error:", err)
		}
		/*
		rpc.ServeCodec()函数替代了rpc.ServeConn()函数，
		传入的参数是针对服务器端的JSON编解码器。
		*/
		go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
	}
}

```
###### 客户端
```go

func main(){
	/*
	   先手工调用net.Dial()函数建立TCP链接，
	   然后基于该链接建立针对客户端的JSON编解码器。
	*/
	conn, err := net.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("net.Dial:", err)
	}
	client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
	var reply string
	//{"method":"HelloService.Hello","params":["hello"],"id":0}
	err = client.Call("HelloService.Hello", "hello", &reply)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply)
}
```
使用tcp客户端与服务端连接
```go
func main(){
	dial, err := net.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("dialing error:", err)
	}
	defer dial.Close()
	go dial.Write(
		[]byte(`{"method":"HelloService.Hello","params":["hello"],"id":1}`))
	buf := make([]byte, 1024)
	dial.Read(buf)
	fmt.Println(string(buf))//{"id":1,"result":"ciallo!,hello","error":null}
	
}
```
## HTTP上的RPC
#### 服务端
```go
type HelloService struct{}

func (hs *HelloService) Hello(request string, reply *string) error {
	*reply = "ciallo!," + request
	return nil
}

func main() {
	rpc.RegisterName("HelloService", new(HelloService))
	http.HandleFunc("/jsonrpc", func(w http.ResponseWriter, r *http.Request) {
		var conn io.ReadWriteCloser = struct {
			io.Writer
			io.ReadCloser
		}{
			ReadCloser: r.Body,
			Writer:     w,
		}
		rpc.ServeRequest(jsonrpc.NewServerCodec(conn))
	})
	http.ListenAndServe(":1234", nil)
}

```

#### 客户端
```go
	client := &http.Client{
		Timeout: 15 * time.Second,
	}
	req, err := http.NewRequest("POST", "http://localhost:1234/jsonrpc", 
		bytes.NewBuffer(
			[]byte(
				`{"method":"HelloService.Hello","params":["hello"],"id":0}`)))
	req.Header.Set("Content-Type", "application/json; charset=UTF-8")
	if err != nil {
		log.Fatal("new request err", err)
	}
	resp, err := client.Do(req)
	if err != nil {
		log.Fatal("send error:", err)
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		log.Fatal("read all error:", err)
	}
	fmt.Println(string(body))//{"id":0,"result":"ciallo!,hello","error":null}
```

# Protobuf
## 安装protobuf
```sh

#https://github.com/protocolbuffers/protobuf/releases
#下载 protocc-*.*-*.zip

apt install unzip

unzip protoc-23.2-linux-x86_64.zip -d /usr/local

protoc --version # libprotoc *.*

go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```
#### 示例1
```proto
//声明使用 proto3 版本，不写的话默认是 proto2 版本
syntax = "proto3";

//声明 proto 文件所在的包名，相当于 proto 文件的命名空间，防止不同的消息类型产生命名冲突，该字段是可选的。
package main;

//声明了要生成的 .pb.go 文件保存的路径（这个放在后面生成代码时讲）和 .pb.go 文件的包名，用分号 ; 隔开
option go_package=".;main";

message String{
  string value = 1;
}
```

```sh
protoc --go_out=. *.proto
```

```go
type HelloService struct{}
//新的类型String 代替旧的类型 string
func (*HelloService) Hello(request String, reply *String) error {
	reply.Value = "ciallo!," + request.Value
	return nil
}

```

#### 示例2
```proto

syntax = "proto3";

package main;

option go_package=".;main";

message String{
  string value = 1;
}

service HelloService{
  rpc Hello (String)returns (String);
}
```

```sh
#gRPC gen插件
go get -u google.golang.org/protobuf/cmd/protoc-gen-go
go install google.golang.org/protobuf/cmd/protoc-gen-go
go get -u google.golang.org/grpc/cmd/protoc-gen-go-grpc
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc

protoc --go-grpc_out=. hello.proto
```

## 定制代码生成的插件



# 玩转RPC
## 客户端RPC的实现原理
```go
//Call 实现
func (client *Client) Call(
		serviceMethod string,
		args interface{},
		reply interface{})error{
		/*
		通过Client.Go()方法进行一次异步调用，
		返回一个表示这次调用的Call结构体。
		然后等待Call结构体的Done通道返回调用结果
		*/
	call:=<-client.Go(
		serviceMethod,
		args,reply,
		make(chan *Call,1)).Done
	
	return call.Error
}

//Go 实现
func (client *Client) Go(
	    serviceMethod string, 
	    args interface{},
	    reply interface{},
	    done chan *Call) *Call {
	/*
		首先构造一个表示当前调用的call变量，
	*/
    call := new(Call)
    call.ServiceMethod = serviceMethod
    call.Args = args
    call.Reply = reply
    call.Done = make(chan *Call, 10) // buffered.
    /*
    	然后通过client.send()将call的完整参数发送到RPC框架。
    */
    client.send(call)
    /*
        client.send()方法调用是线程安全的，
	    因此可以从多个Goroutine同时向同一个RPC链接发送调用指令。
	    当调用完成或者发生错误时，将调用call.done()方法通知完成：
    */
    return call
}

//done 实现
func (call *Call) done() {
/*
	当调用完成或者发生错误时，将调用call.done()方法通知完成
	call.Done通道会将处理后的call返回。
*/
    select {
	    case call.Done <- call:
	        // ok
	    default:
	        // We don't want to block here. 
	        // It is the caller's responsibility to make
	        // sure the channel has enough buffer space.
	        // See comment in Go().
    }
}
```

## 基于RPC实现监视功能
```go
/*
构造一个简单的内存键值数据库
*/
type KVStoreService struct {
	//用于存储键值数据
	m map[string]string

	//对应每个Watch()调用时定义的过滤器函数列表
	filter map[string]func(key string)

	//互斥锁，用于在多个Goroutine访问或修改时对其他成员提供保护。
	mu sync.Mutex
}

func NewKVStoreService() *KVStoreService {
	return &KVStoreService{
		m:      make(map[string]string),
		filter: make(map[string]func(key string)),
	}
}
func (p *KVStoreService) Get(key string, value *string) error {
	p.mu.Lock()
	defer p.mu.Unlock()
	if v, ok := p.m[key]; ok {
		*value = v
		return nil
	}
	return fmt.Errorf("not found")
}

/*
输入参数是键和值组成的数组，
用一个匿名的空结构体表示忽略了输出参数。
*/
func (p *KVStoreService) Set(kv [2]string, reply *struct{}) error {
	p.mu.Lock()
	defer p.mu.Unlock()
	key, value := kv[0], kv[1]
	if oldValue := p.m[key]; oldValue != value {
		//当修改某个键对应的值时会调用每一个过滤器函数。
		for _, fn := range p.filter {
			fn(key)
		}
	}
	p.m[key] = value
	return nil
}

/*
过滤器列表在Watch()方法中提供
Watch()方法的输入参数是超时的秒数。
*/
func (p *KVStoreService) Watch(timeoutSecond int, keyChanged *string) error {
	//Watch()的实现中，用唯一的id表示每个Watch()调用，
	id := fmt.Sprintf("watch-%s-%03d", time.Now(), rand.Int())
	ch := make(chan string, 10) // buffered

	p.mu.Lock()

	//然后根据id将自身对应的过滤器函数注册到p.filter列表。
	p.filter[id] = func(key string) { ch <- key }
	p.mu.Unlock()

	select {
	case <-time.After(time.Duration(timeoutSecond) * time.Second):
		//如果超过时间后依然没有键被修改，则返回超时的错误。
		return fmt.Errorf("timeout")
	case key := <-ch:
		//当有键变化时将键作为返回值返回。
		*keyChanged = key
		return nil
	}
}

// 客户端使用 Watch 方法
func doClientWork(client *rpc.Client) {
	/*
		首先启动一个独立的Goroutine监控键的变化。
		同步的Watch()调用会阻塞，直到有键发生变化或者超时。
	*/
	go func() {
		var keyChanged string
		err := client.Call("KVstoreService.Watch", 30, &keyChanged)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println("watch:", keyChanged)
	}()
	/*
		然后在通过Set()方法修改键值时，
		服务器会将变化的键通过Watch()方法返回。
		这样就可以实现对某些状态的监控。
	*/
	err := client.Call(
		"KVStoreService.Set",
		[2]string{"abc", "ciallo"},
		new(struct{}))
	if err != nil {
		log.Fatal(err)
	}
	time.Sleep(time.Second * 3)
}
```

## 反向RPC
服务端主动连接客户端
#### 服务端代码
```go
type HelloService struct{}

func (*HelloService) Hello(req string, res *string) error {
	*res = "ciallo~," + req
	return nil
}
func main() {
	rpc.Register(new(HelloService))
	for {
		conn, _ := net.Dial("tcp", "localhost:1234")
		if conn == nil {
			time.Sleep(time.Second)
			continue
		}
		rpc.ServeConn(conn)
		conn.Close()
	}
}
```
#### 客户端代码
```go
func doClientWork(clientChan <-chan *rpc.Client) {
		client := <-clientChan
		defer client.Close()
		var reply string
		err = client.Call("HelloService.Hello", "hello", &reply)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println(reply)
	}
	
func main(){
	/*
	反向RPC的内网服务将不再主动提供TCP监听服务，而是首先主动链接到对方的TCP服务器。
	然后基于每个建立的TCP链接向对方提供RPC服务。
	
	RPC客户端则需要在一个公共的地址提供一个TCP服务，用于接受RPC服务器的链接请求
	*/
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}
	clientChan := make(chan *rpc.Client)
	go func() {
		for {
			conn, err := listener.Accept()
			if err != nil {
				log.Fatal("Accept error:", err)
			}
			//当每个链接建立后，基于网络链接构造RPC客户端对象并发送到clientChan通道。
			clientChan <- rpc.NewClient(conn)
		}
	}()
	//	客户端执行RPC调用的操作在doClientWork()函数完成
	doClientWork(clientChan)
}
```

## 上下文信息

基于上下文可以针对不同客户端提供定制化的RPC服务。
我们可以通过为每个链接提供独立的RPC服务来实现对上下文特性的支持。
```go
type HelloService struct {
	conn net.Conn
}

func (p *HelloService) Hello(request string, reply *string) error {
	*reply = "hello:" + request + ", from" + p.conn.RemoteAddr().String()
	return nil
}
func main() {
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Fatal("Accept error:", err)
		}
		go func() {
			defer conn.Close()
			p := rpc.NewServer()
			p.Register(&HelloService{conn: conn})
			p.ServeConn(conn)
		}()
	}
}

```
为RPC服务增加简单的登录状态的验证
```go
type HelloService struct {
    conn    net.Conn
    isLogin bool
}
func (p *HelloService) Login(request string, reply *string) error {
    if request != "user:password" {
        return fmt.Errorf("auth failed")
    }
    log.Println("login ok")
    p.isLogin = true
    return nil
}
func (p *HelloService) Hello(request string, reply *string) error {
    if !p.isLogin {
        return fmt.Errorf("please login")
    }
    *reply = "hello:" + request + ", from" + p.conn.RemoteAddr().String()
    return nil
}
```

# gRPC入门
## 入门
#### proto文件
```proto
syntax = "proto3";
package main;
option go_package=".;main";
message String {
    string value = 1;
}
service HelloService {
    rpc Hello (String) returns (String);
}
```
#### 编译proto文件
```sh
protoc --go_out=. *.proto
protoc --go-grpc_out=. *.proto
```
#### c、s 代码
```go
type HelloServiceImpl struct{}

func (*HelloServiceImpl) Hello(ctx context.Context, args *String) (*String, error) {
	reply := &String{Value: "Hello:" + args.GetValue()}
	return reply, nil
}

func (h *HelloServiceImpl) mustEmbedUnimplementedHelloServiceServer() {}

func main() {
	go func() {
		/*
		其中grpc.Dial负责和gRPC服务建立链接，

		*/
		conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
		if err != nil {
			log.Fatal(err)
		}
		defer conn.Close()
		/*
		然后NewHelloServiceClient()函数基于已经建立的链接构造HelloServiceClient对象。
		返回的client其实是一个HelloServiceClient接口对象
		*/
		client := NewHelloServiceClient(conn)
		reply, err := client.Hello(context.Background(), &String{Value: "hello"})
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println(reply.GetValue())
	}()

	//首先通过grpc.NewServer()构造一个gRPC服务对象，
	grpcServer := grpc.NewServer()

	/*
		然后通过gRPC插件生成的RegisterHelloServiceServer()函数
		注册我们实现的HelloServiceImpl服务
	*/
	RegisterHelloServiceServer(grpcServer, new(HelloServiceImpl))

	//再通过grpcServer.Serve(lis)在一个监听端口上提供gRPC服务。
	lis, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal(err)
	}
	grpcServer.Serve(lis)
}
```
#### 编译
```sh
go build 
```

#### gRPC和标准库的RPC框架有一个区别：
即gRPC生成的接口并不支持异步调用。
不过，可以在多个Goroutine之间安全地共享gRPC底层的HTTP/2链接，
因此可以通过在另一个Goroutine阻塞调用的方式模拟异步调用。

## gRpc流
## 背景
RPC是远程函数调用，因此每次调用的函数参数和返回值不能太大，
否则将严重影响每次调用的响应时间。
因此传统的RPC方法调用对上传和下载较大数据量的场景并不适合。
同时传统RPC模式也不适用于时间不确定的订阅和发布模式。
为此，gRPC框架针对服务器端和客户端分别提供了流特性。
## 用法
#### proto
```proto
syntax = "proto3";
package main;
option go_package=".;main";
message String{
  string value = 1;
}
service HelloService{
  rpc Hello(String) returns (String);
  /*
  关键字stream指定启用流特性，
    参数部分是接收客户端参数的流，
    返回值是返回给客户端的流。
  */
  rpc Channel(stream String)returns (stream String);
}
```
#### 生成代码
```go
type HelloServiceServer interface {
    Hello(context.Context, *String) (*String, error)
    Channel(HelloService_ChannelServer) error
}
type HelloServiceClient interface {
    Hello(ctx context.Context, in *String, opts ...grpc.CallOption) (
        *String, error,
    )
    Channel(ctx context.Context, opts ...grpc.CallOption) (
        HelloService_ChannelClient, error,
    )
}
/*
服务器端的Channel()方法的参数是一个新的HelloService_ChannelServer类型的参数，
可以用于和客户端双向通信
*/
type HelloService_ChannelServer interface {
    Send(*String) error
    Recv() (*String, error)
    grpc.ServerStream
}
/*
客户端的Channel()方法返回一个HelloService_ ChannelClient类型的返回值，
可以用于和服务器端进行双向通信。
*/
type HelloService_ChannelClient interface {
    Send(*String) error
    Recv() (*String, error)
    grpc.ClientStream
}
/*
服务器端和客户端的流辅助接口均定义了方法Send()和Recv()，用于流数据的双向通信。
*/
```
#### c，s的代码
```go
type HelloServiceImpl struct{}

/*
服务器端在循环中接收客户端发来的数据，

	如果遇到io.EOF表示客户端流关闭，
	如果函数退出表示服务器端流关闭。

生成返回的数据通过流发送给客户端，双向流数据的发送和接收是完全独立的行为。
需要注意的是，发送和接收的操作并不需要一一对应，
*/
func (h *HelloServiceImpl) Channel(p0 HelloService_ChannelServer) error {
	for {
		args, err := p0.Recv()
		if err != nil {
			if err == io.EOF {
				return nil
			}
			return err
		}
		reply := &String{Value: "ciallo~:" + args.GetValue()}
		err = p0.Send(reply)
		if err != nil {
			return err
		}
	}
}

func (h *HelloServiceImpl) Hello(p0 context.Context, p1 *String) (*String, error) {
	return nil, nil
}

func (h *HelloServiceImpl) mustEmbedUnimplementedHelloServiceServer() {}

func main() {
	go func() {
		//////////// client //////////
		conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
		if err != nil {
			log.Fatal(err)
		}
		defer conn.Close()
		client := NewHelloServiceClient(conn)
		//客户端需要先调用Channel()方法获取返回的流对象
		stream, err := client.Channel(context.Background())
		if err != nil {
			log.Fatal(err)
		}
		go func() {
			//在客户端我们将发送和接收操作放到两个独立的Goroutine。首先是向服务器端发送数据
			for {
				if err := stream.Send(&String{Value: "hi"}); err != nil {
					log.Fatal(err)
				}
				time.Sleep(time.Second)
			}
		}()
		//然后在循环中接收服务器端返回的数据
		for {
			reply, err := stream.Recv()
			if err != nil {
				if err == io.EOF {
					break
				}
				log.Fatal(err)
			}
			fmt.Println(reply.GetValue())
		}
	}()
	//////////// server /////////////
	grpcServer := grpc.NewServer()
	RegisterHelloServiceServer(grpcServer, new(HelloServiceImpl))
	lis, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal(err)
	}
	grpcServer.Serve(lis)
}

```

# 发布订阅模式
## 背景
基于Watch()的思路虽然也可以构造发布和订阅系统，
但是因为RPC缺乏流机制导致每次只能返回一个结果。
在发布和订阅模式中，由调用者主动发起的发布行为类似一个普通函数调用，
而被动的订阅者则类似gRPC客户端单向流中的接收者

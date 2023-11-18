
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

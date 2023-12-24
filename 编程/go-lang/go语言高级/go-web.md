# 请求路由
## 词缀树


# 中间件
## 函数适配器
```go
//适配器函数
func timeMiddleWare(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		timeStart := time.Now()
		next.ServeHTTP(w, r)
		timeElapsed := time.Since(timeStart)
		fmt.Println(timeElapsed)
	})
}
func main() {
	http.Handle("/", timeMiddleWare(
		http.HandlerFunc(
			func(w http.ResponseWriter, r *http.Request) {
				w.Write([]byte("ciallo"))
	})))
	_ = http.ListenAndServe(":8080", nil)
}

```
## http.Handler
函数定义
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
对于任何方法，只要实现了ServeHTTP，它就是一个合法的http.Handler
### http库的Handler、HandlerFunc和ServeHTTP的关系
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

//用来调用链接的处理逻辑
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
    /*执行函数里面的逻辑
    func(w http.ResponseWriter, r *http.Request) {
    ///////////     执行这一段   ///////////
		w.Write([]byte("ciallo"))
	///////////////////////////////////////
	}
    */
}
```
## 小结
中间件要做的事情就是通过一个或多个函数对handler进行包装，
返回一个包括了各个中间件逻辑的函数链

## 更优雅的中间件写法
```go
// ////////////////// 服务器 ///////////////////////
type Server struct {
	Addr   string
	Router *Router
	Port   string
}

func (s *Server) Run() {
	if s.Port == "" {
		panic("port is not nil")
	}
	err := http.ListenAndServe(s.Addr+":"+s.Port, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		s.Router.Handler(r.URL.Path, w, r)
	}))
	if err != nil {
		fmt.Println(err)
	}
}
func NewServer(addr, port string) *Server {
	return &Server{
		Addr:   addr,
		Port:   port,
		Router: NewRouter(),
	}
}
//////////////////// 服务器 /////////////////////

//////////////////// 路由 ///////////////////////
type middleware func(http.Handler) http.Handler
type Router struct {
	middlewareChain []middleware
	mux             map[string]http.Handler
}

func NewRouter() *Router {
	r := &Router{}
	r.middlewareChain = make([]middleware, 0)
	r.mux = make(map[string]http.Handler)
	return r
}

// 添加中间件
func (r *Router) Use(m middleware) {

	r.middlewareChain = append(r.middlewareChain, m)
}

func (r *Router) Add(route string, h http.Handler) {
	var mergedHandler = h
	for i := len(r.middlewareChain) - 1; i >= 0; i-- {
		mergedHandler = r.middlewareChain[i](mergedHandler)
	}
	r.mux[route] = mergedHandler
}
func (r *Router) Handler(patter string, w http.ResponseWriter, req *http.Request) {
	r.mux[patter].ServeHTTP(w, req)
}

// ////////////////// 路由 ///////////////////////
func main() {
	server := NewServer("", "8080")
	server.Router.Use(func(h http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			fmt.Println("中间件1.。。。。。。。。")
			h.ServeHTTP(w, r)
		})
	})
	server.Router.Use(func(h http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			fmt.Println("中间件2.。。。。。。。。")
			h.ServeHTTP(w, r)
		})
	})
	server.Router.Add("/", http.HandlerFunc(
		func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("ciallo"))
	}))
	server.Run()
}
```
# 请求校验
## 重构某个校验函数
待优化的代码
```go
type RegisterReq struct {
    Username       string   `json:"username"`
    PasswordNew    string   `json:"password_new"`
    PasswordRepeat string   `json:"password_repeat"`
    Email          string   `json:"email"`
}
func register(req RegisterReq) error{
    if len(req.Username) > 0 {
        if len(req.PasswordNew) > 0 && len(req.PasswordRepeat) > 0 {
            if req.PasswordNew == req.PasswordRepeat {
                if emailFormatValid(req.Email) {
                    createUser()
                    return nil
                } else {
                    return errors.New("invalid email")
                }
            } else {
                return errors.New("password and reinput must be the same")
            }
        } else {
            return errors.New("password and password reinput must be longer than 0")
        }
    } else {
        return errors.New("length of username cannot be 0")
    }
}
```
优化后
```go
func register(req RegisterReq) error{
    if len(req.Username) == 0 {
        return errors.New("length of username cannot be 0")
    }
    if len(req.PasswordNew) == 0 || len(req.PasswordRepeat) == 0 {
        return errors.New("password and password reinput must be longer than 0")
    }
    if req.PasswordNew != req.PasswordRepeat {
        return errors.New("password and reinput must be the same")
    }
    if emailFormatValid(req.Email) {
        return errors.New("invalid email")
    }
    createUser()
    return nil
}
```

## 用请求校验器解放体力劳动
### 示例
```go
import (
	"fmt"

	"gopkg.in/go-playground/validator.v9"
)

type RegisterReq struct {
	// 字符串的gt=0 表示长度必须 > 0，gt = greater than
	Username string `validate:"gt=0"`
	// 同上
	PasswordNew string `validate:"gt=0"`
	// eqfield 跨字段相等校验
	PasswordRepeat string `validate:"eqfield=PasswordNew"`
	// 合法 email 格式校验
	Email string `validate:"email"`
}

var validate = validator.New()

func validateFunc(req RegisterReq) error {
	err := validate.Struct(req)
	if err != nil {
		//...
		return err
	}
	//...
	return nil
}
func main() {
	//...
	req := RegisterReq{
		Username:       "Xargin",
		PasswordNew:    "ohno",
		PasswordRepeat: "ohn",
		Email:          "alex@abc.com",
	}
	err := validateFunc(req)
	fmt.Println(err)
	/*
		Key: 'RegisterReq.PasswordRepeat' Error:Field validation for
		'PasswordRepeat' failed on the 'eqfield' tag
	*/
}
```
### 原理
```go
type Nested struct {
    Email string `validate:"email"`
}
type T struct {
    Age    int `validate:"eq=10"`
    Nested Nested
}

```
struct T
	| Age
	| struct Nested
		|Email	
		
一个递归的深度优先搜索方式的遍历
```go
package main
import (
    "fmt"
    "reflect"
    "regexp"
    "strconv"
    "strings"
)
type Nested struct {
    Email string `validate:"email"`
}
type T struct {
    Age    int `validate:"eq=10"`
    Nested Nested
}
func validateEmail(input string) bool {
    if pass, _ := regexp.MatchString(
        `^([\w\.\_]{2,10})@(\w{1,}).([a-z]{2,4})$`, input,
    ); pass {
        return true
    }
    return false
}
func validate(v interface{}) (bool, string) {
    validateResult := true
    errmsg := "success"
    vt := reflect.TypeOf(v)
    vv := reflect.ValueOf(v)
    for i := 0; i < vv.NumField(); i++ {
        fieldVal := vv.Field(i)
        tagContent := vt.Field(i).Tag.Get("validate")
        k := fieldVal.Kind()
        switch k {
	        case reflect.Int:
	            val := fieldVal.Int()
	            tagValStr := strings.Split(tagContent, "=")
	            tagVal, _ := strconv.ParseInt(tagValStr[1], 10, 64)
	            if val != tagVal {
	                errmsg = "validate int failed, tag is: "+ strconv.FormatInt(
	                    tagVal, 10,
	                )
	                validateResult = false
	            }
        case reflect.String:
            val := fieldVal.String()
            tagValStr := tagContent
            switch tagValStr {
	            case "email":
	                nestedResult := validateEmail(val)
	                if nestedResult == false {
	                    errmsg = "validate mail failed, field val is: "+ val
	                    validateResult = false
	                }
            }
        case reflect.Struct:
            // 如果有内嵌的struct，那么深度优先遍历
            // 就是一个递归过程
            valInter := fieldVal.Interface()
            nestedResult, msg := validate(valInter)
            if nestedResult == false {
                validateResult = false
                errmsg = msg
            }
        }
    }
    return validateResult, errmsg
}
func main() {
    var a = T{
		    Age: 10, 
		    Nested: Nested{
			    Email: "abc@abc.com",
			    },
			}
    validateResult, errmsg := validate(a)
    fmt.Println(validateResult, errmsg)
}
```
### 反射的代替方案
使用Go内置的Parser对源代码进行扫描，然后根据结构体的定义生成校验代码。我们可以将所有需要校验的结构体放在单独的包内

# 服务流量限制

流量限制的手段有很多，最常见的有漏桶和令牌桶两种。
1. 漏桶是指我们有一个一直装满了水的桶，每隔固定的一段时间即向外漏一滴水。如果你接到了这滴水，那么你就可以继续服务请求，如果没有接到，那么就需要等待下一滴水。
2. 令牌桶则是指匀速向桶中添加令牌，服务请求时需要从桶中获取令牌，令牌的数目可以按照需要消耗的资源进行相应的调整。如果没有令牌，可以选择等待，或者放弃。

### 两者的区别
1. 漏桶流出的速率固定，而令牌桶只要在桶中有令牌，那就可以拿，也就是说，令牌桶是允许一定程度的并发的，例如，同一个时刻，有100个用户请求，只要令牌桶中有100个令牌，那么这100个请求就全都会放过去。
2. 令牌桶在桶中没有令牌的情况下也会退化为漏桶。
## 令牌桶的原理
#### 思路1
从功能上来看，令牌桶模型实际上就是对全局计数的加减法操作过程
```go

var capacity = 10
var tokenBucket = make(chan struct{}, capacity)

// 每过一段时间向tokenBucket中添加令牌，如果桶已经满了，那么直接放弃
func fillToken() {
	var fillInterval = time.Millisecond * 10000
	ticker := time.NewTicker(time.Duration(fillInterval))
	for {
		select {
		case <-ticker.C:
			select {
			case tokenBucket <- struct{}{}:
			default:
			}
			//fmt.Println("current token cnt:", len(tokenBucket), time.Now())
		}
	}
}

// 取一个令牌
func TakeAvailable(block bool) bool {
	var takenResult bool
	if block {
		select {
		case <-tokenBucket: //如果没有资源的话将在这里阻塞住
			takenResult = true
		// default:
		// 	takenResult = false
		}
	} else {
		select {
		case <-tokenBucket:
			takenResult = true
		default:
			takenResult = false
		}
	}
	return takenResult
}
func main() {

	go fillToken()
	go func() {
		var i = 0
		for {
			var result = false
			if len(tokenBucket) > 0 {
				// fmt.Println(len(tokenBucket))
				result = TakeAvailable(false)
			} else {
				result = TakeAvailable(true)

			}
			if result {
				i++
				fmt.Println(i)
			} else {
				fmt.Println("error")
			}
			time.Sleep(time.Millisecond * 500)
		}

	}()
	time.Sleep(time.Hour)
}

```
#### 思路二
令牌桶每隔一段固定的时间向桶中放令牌，如果我们记下上一次放令牌的时间为t1，当时的令牌数k1，放令牌的时间间隔为ti，每次向令牌桶中放x个令牌，令牌桶容量为cap。现在如果有人调用TakeAvailable来取n个令牌，我们将这个时刻记为t2。在t2时刻，令牌桶中理论上应该有多少个令牌呢？
伪代码：
```
cur = k1 + ((t2 - t1)/ti) * x
cur = cur > cap ? cap : cur
```
用两个时间点的时间差，再结合其他参数，理论上在取令牌之前就完全可以知道桶里有多少个令牌了。那劳心费力地像本小节前面向通道里填充令牌的操作，理论上是没有必要的。只要在每次Take的时候，再对令牌桶中的令牌数进行简单计算，就可以得到正确的令牌数。
得到正确的令牌数之后，再进行实际的Take操作就好，这个Take操作只需要对令牌数进行简单的减法即可，记得加锁以保证并发安全。

# 常见大型web项目分层
### MVC
1. 控制器（Controller）：负责转发请求，对请求进行处理。
2. 视图（View）：界面设计人员进行图形界面设计。
3. 模型（Model）：程序员编写程序应有的功能（实现算法等）、数据库专家进行数据管理和数据库设计（可以实现具体的功能）。
### 纯后端分层
1. Controller：与上述类似，是服务入口，负责处理路由、参数校验、请求转发。
2. Logic/Service：逻辑（服务）层，一般是业务逻辑的入口，可以认为从这里开始，所有的请求参数一定是合法的。业务逻辑和业务流程也都在这一层中。常见的设计中会将该层称为业务规则。
3. DAO/Repository：这一层主要负责和数据、存储打交道。将下层存储以更简单的函数、接口形式暴露给逻辑层来使用。负责数据的持久化工作。

### 处理HTTP的代码
```go
// 在协议层定义
type CreateOrderRequest struct {
    OrderID int64 `json:"order_id"`
    // ...
}
// 在控制器中定义
type CreateOrderParams struct {
    OrderID int64
}
func HTTPCreateOrderHandler(wr http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    var params CreateOrderParams
    ctx := context.TODO()
    // 绑定数据到 req
    bind(r, &req)
    // 将协议绑定映射到协议独立
    map(req, params)
    logicResp,err := controller.CreateOrder(ctx, &params)
    if err != nil {}
    // ...
}
```

# 表驱动开发
修改前
```go
func entry() {
    var bi BusinessInstance
    switch businessType {
    case TravelBusiness:
        bi = travelorder.New()
    case MarketBusiness:
        bi = marketorder.New()
    default:
        return errors.New("not supported business")
    }
}
```
修改后
```go
var businessInstanceMap = map[int]BusinessInstance {
    TravelBusiness : travelorder.New(),
    MarketBusiness : marketorder.New(),
}
func entry() {
    bi := businessInstanceMap[businessType]
}
/*
	表驱动也不是没有缺点，因为需要对输入键计算散列，
	在性能敏感的场合需要多加斟酌。
*/
```

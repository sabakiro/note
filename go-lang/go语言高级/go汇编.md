
# 快速入门

### 语法

```plan 9
//初始化包变量
//Data symbol+offset(SB)/width, value
/*
给Id变量初始化为十六进制的0x2537，
对应十进制的9527（常量需要以美元符号$开头表示）
*/
DATA ·Id+0(SB)/1,$0x37
DATA ·Id+1(SB)/1,$0x25

//变量定义好之后需要导出以供其他代码引用
GLOBL ·Id, $8
```
### 示例
masm
	pkg
		pkg.go
		pkg_amd64.s
	main.go
#### 整型
pkg/pkg_amd64.s
```plan9
#include"textflag.h"
GLOBL ·Id(SB),NOPTR,$8
DATA ·Id+0(SB)/1,$0x37 //最大值为255: 2^8 - 1
DATA ·Id+1(SB)/1,$0x25 //最小值为256 :2^8
DATA ·Id+2(SB)/1,$0x00 //最小值为 65536 2^16
DATA ·Id+3(SB)/1,$0x00
DATA ·Id+4(SB)/1,$0x00
DATA ·Id+5(SB)/1,$0x00
DATA ·Id+6(SB)/1,$0x00
DATA ·Id+7(SB)/1,$0x00

/*
不加 #include"textflag.h" 和 NOPTR
pkgpath.NameData: missing Go type information for global symbol: size 8
*/
```
pkg/pkg.go
```go
package pkg

var Id int
```
main.go
```go
package main
import (
	"fmt"
	pkg "masm/pkg"
)
func main(){
	fmt.Println(pkg.Id)//9527
}
```
#### 字符串
写法1：
pkg/pkg.go
```go
package pkg

var Name string
var NameData [8]byte
```
pkg/pkg_amd64.s
```plan9
/*
由于在Go汇编语言中，go.string."gopher"不是一个合法的符号，
因此无法通过手工创建它（这是给编译器保留的部分特权，
因为手工创建类似符号可能打破编译器输出代码的某些规则）。
*/
#include "textflag.h"

//因此我们新创建了一个·NameData符号表示底层的字符串数据。
GLOBL ·NameData(SB),NOPTR,$8
DATA  ·NameData(SB)/8,$"gopher"

//然后定义·Name符号内存大小为16字节，
GLOBL ·Name(SB),NOPTR,$16

其中前8字节用·NameData符号对应的地址初始化，后8字节为常量6表示字符串长度。
DATA  ·Name+0(SB)/8,$·NameData(SB)
DATA  ·Name+8(SB)/8,$6

```
写法2：
pkg/pkg.go
```go
package pkg

var Name string
//不暴露NameData，避免字符串底层被更改而报错
//var NameData [8]byte 
```
pkg/pkg_amd64.s
```plan9
#include "textflag.h"

GLOBL ·Name(SB),NOPTR,$24
DATA ·Name+0(SB)/8,$·Name+16(SB)
DATA ·Name+8(SB)/8,$6
DATA ·Name+16(SB)/8,$"gopher"
```
main.go
```go
package main

import (
  "fmt"
  "masm/pkg"
)

func main() {

  fmt.Println(pkg.Name)//gopher

}
```

#### 函数
main.go
```go
package main

import "fmt"

func neg(x uint64) int64

func main() {
	fmt.Println(neg(42))//-42
}

```
main_amd64.s
```plan9
#include "textflag.h"

TEXT ·neg(SB), NOSPLIT, $0
  MOVQ     x+0(FP), AX
  NEGQ     AX
  MOVQ     AX, ret+8(FP)
  RET
//EOF
```
ps:使用go build 命令才能自动连接 .s 文件

# 常量和全局变量
### 常量
Go汇编语言中常量以美元符号$为前缀
```plan9
$1            // 十进制
$0xf4f8fcff   // 十六进制
$1.5          // 浮点数
$'a'          // 字符
$"abcd"       // 字符串

//构造新的常量
$2+2       // 常量表达式；== $4
$3&1<<2    // == $4
$(3&1)<<2  // == $4
```
### 全局变量
```plan9
#include "textflag.h"

//声明int32的全局变量
GLOBL ·i(SB), NOPTR, $4

//逐字节赋值
DATA ·i+0(SB)/1,$0
DATA ·i+1(SB)/1,$1
DATA ·i+2(SB)/1,$0x00
DATA ·i+3(SB)/1,$0x00
//256

//一次性赋值
DATA ·i+0(SB)/4, $0x00000100 //256
//or
DATA ·i+0(SB)/4, $256 //256
```

#### 数组类型变量
```plan9
#include "textflag.h"

//使用汇编定义一个名为 num 的 [2]int
GLOBL ·num(SB), NOPTR, $16
DATA ·num+0(SB)/8, $10
DATA ·num+8(SB)/8, $0
//[10 0]

```

#### 布尔类型
```plan9
#include "textflag.h"
//初始化，默认为false
GLOBL ·boolValue(SB), NOPTR, $1
GLOBL ·trueValue(SB), NOPTR, $1
GLOBL ·falseValue(SB), NOPTR, $1
//赋值，值非0即为true
DATA ·trueValue+0(SB)/1, $1
DATA ·falseValue+0(SB)/1, $0

/*
boolValue:false
trueValue:true
falseValue:false
*/
```

#### 整数类型变量
```plan9
GLOBL ·int32Value(SB), NOPTR, $4
DATA ·int32Value+0(SB)/1,$0x04
DATA ·int32Value+1(SB)/1,$0x03
DATA ·int32Value+2(SB)/1,$0x02
DATA ·int32Value+3(SB)/1,$0x01
//16909060

GLOBL ·uint32Value(SB), NOPTR,$4
DATA ·uint32Value+0(SB)/4, $0x01020304
//16909060

/*------------------------------*/
GLOBL ·int32Value(SB), NOPTR, $4
DATA ·int32Value+0(SB)/1,$0x00
DATA ·int32Value+1(SB)/1,$0x00
DATA ·int32Value+2(SB)/2,$0x1001

GLOBL ·uint32Value(SB), NOPTR,$4
DATA ·uint32Value+0(SB)/4, $0x10010000

/*
268500992
268500992
*/
```

#### 浮点类型变量
```plan9
GLOBL ·float32Value(SB), NOPTR, $4
GLOBL ·float64Value(SB), NOPTR, $8

DATA ·float32Value+0(SB)/4, $1.5
DATA ·float64Value+0(SB)/8,$0x01020304
/*
1.5
8.3541856e-317
*/
```

#### 字符串类型变量
字符串头的结构体定义
```go
type reflect.StringHeader struct{
	Data uintptr
	Len int
}
```

```plan9
GLOBL ·helloworld(SB), NOPTR,$16

//定义私有变量text,内容为 Hello World,<>为私有修饰符
GLOBL text<>(SB), NOPTR,$16
DATA text<>+0(SB)/8, $"Hello Wo"
DATA text<>+8(SB)/8, $"rld!"

//StringHeader.Data
DATA ·helloworld+0(SB)/8, $text<>(SB)
//StringHeader.Len 
DATA ·helloworld+8(SB)/8, $12

```

#### 切片类型变量
切片头的结构体定义
```go
type reflect.SliceHeader struct{
	Data uintptr
	Len int
	Cap int
}
```

```plan9

GLOBL text<>(SB), NOPTR, $16
DATA text<>+0(SB)/16,$"Hello World!"

GLOBL ·helloworld(SB), NOPTR, $24
//StringHeader.Data
DATA ·helloworld+0(SB)/8, $text<>(SB)
//StringHeader.Len
DATA ·helloworld+8(SB)/8, $12
//StringHeader.Cap
DATA ·helloworld+16(SB)/8,$16

```

## chan 和 map

```plan9

//var m map[string]int
GLOBL ·m(SB), NOPTR, $8 
DATA ·m+0(SB)/8, $0

//var ch chan int
GLOBL ·ch(SB), NOPTR, $8
DATA ·ch+0(SB)/8, $0

```

#### 定义只读变量
```plan9
/*
RODATA表示将变量定义在只读内存段，
NOPTR则表示此变量的内部不含指针数据
*/

GLOBL ·intValue(SB), NOPTR|RODATA, $8
DATA ·intValue+0(SB)/8, $10086

```

```go
var intValue int

func fn(){
	intValue=555//error
	/*
	不能修改的限制并不是由编译器提供，
	而是因为对该变量的修改会导致对只读内存段进行写操作，从而导致异常。
	*/
}
```

# 函数
函数定义的语法
```plan9
TEXT symbol(SB), [flags,] $framesize[-argsize]
//TEXT:指令
//symbol：函数名
//flag:可选的标志位
//framesize:函数的帧大小
//argsize：可选的函数参数大小

```
示例：
```go
//go:noescape
func Swap(a, b int) (int, int)

```

```plan9
//写法1：
//func Swap(a, b int) (int, int)
TEXT ·Swap(SB), NOSPLIT, $0-32
/*
最完整的写法：函数名部分包含了当前包的路径，
同时指明了函数的参数大小为32字节（对应参数和返回值的4个int类型）。
*/


//写法2：
//func Swap(a, b int) (int, int)
TEXT ·Swap(SB), NOSPLIT, $0
/*
省略了当前包的路径和参数的大小。
如果有NOSPLIT标注，会禁止汇编器为汇编函数插入栈分裂的代码。
NOSPLIT对应Go语言中的//go:nosplit注释。
*/

```
对汇编函数来说，只要是函数的名字和参数大小一致就可以是相同的函数了。
而且在Go汇编语言中，输入参数和返回值参数是没有任何区别的。

函数签名
```go
func Swap(a,b int)(ret0,ret1 int)
```
4个int类型空间，所以是32字节
```plan9
TEXT ·Swap(SB),$0-32
```
为了引用这4个参数，Go汇编引入了一个伪寄存器 FP，
FP：表示函数当前帧的地址，也是第一个参数的地址
a：+0(FP); 
b：+0(FP);
ret0：+8(FP);
ret1：+24(FP);
```plan
// go汇编不允许直接使用 +0（FP），必须要与标识符前缀组合
//func Swap(a, b int) (int, int)
TEXT ·Swap(SB), NOSPLIT, $0-32
  MOVQ a+0(FP),AX       //AX = a
  MOVQ b+8(FP),BX       //BX = b
  MOVQ BX,ret0+16(FP)   //ret0 = BX
  MOVQ AX,ret1+24(FP)   //ret1 = AX
  RET
```

参数和返回值的内存布局
```go
func Foo(a bool,b int16)(c []byte)

type Foo_args struct {
    a bool
    b int16
}
type Foo_returns struct {
    c []byte
}

func Foo(FP *Foo_args, FP_ret *Foo_returns) {
    // a = FP + offsetof(&args.a)
    _ = unsafe.Offsetof(FP.a) + uintptr(FP) // a
    // b = FP + offsetof(&args.b)
    // argsize = sizeof(args)
    argsize = unsafe.Offsetof(FP)
    // c = FP + argsize + offsetof(&return.c)
    _ = uintptr(FP) + argsize + unsafe.Offsetof(FP_ret.c)
    // framesize = sizeof(args) + sizeof(returns)
    _ = unsafe.Offsetof(FP) + unsafe.Offsetof(FP_ret)
    return
}
```
Foo汇编函数和返回值的定位
```plan9
TEXT ·Foo(SB), NOSPLIT, $0
  MOVQ a+0(FP),AX         //a
  MOVQ b+2(FP),BX         //b
  MOVQ c_dat+8*1(FP),CX   //c.Data
  MOVQ c_len+8*2(FP),DX   //c.Len
  MOVQ c_cap+8*3(FP),DI   //c.Cap
  RET
```
a和b参数之间出现了1字节的空洞，
b和c之间出现了4字节的空洞。
出现空洞的原因是，要保证每个参数变量地址都要对齐到相应的倍数。

#### 局部变量

```plan9
/*
func Foo(){
	var c []byte
	var b int16
	var a bool
}

如果整个内存用Memory数组表示，
那么Memory[0(SP):end-0(SP)]就是对应当前栈帧的切片，
其中开头位置是真寄存器SP，结尾位置是伪寄存器SP。
真寄存器SP一般用于表示调用其他函数时的参数和返回值，
真寄存器SP对应内存较低的地址，所以被访问变量的偏移量是正数；
而伪寄存器SP对应高地址，对应的局部变量的偏移量是负数。

X86中栈是从高向低生长的，
所以最先定义的有更小地址的c变量离栈的底部伪寄存器SP更远。

*/
TEXT ·Foo(SB), NOSPLIT, $32-0
  MOVQ a-32(SP),      AX //a
  MOVQ b-30(SP),      BX //b
  MOVQ c_data-24(SP), CX //c.Data
  MOVQ c_len-16(SP),  DX //c.Len
  MOVQ c_cap-8(SP),   DI //c.Cap
  RET
```

```go
/*模拟局部变量的布局
3个局部变量挪到一个结构体中。
然后构造一个SP变量对应的伪寄存器SP，
对应局部变量结构体的顶部。
然后根据局部变量总大小和
每个变量对应成员的偏移量计算相对于伪寄存器SP的距离，
最终偏移量是一个负数。

通过这种方式可以处理复杂的局部变量的偏移，
同时也能保证每个变量地址的对齐要求。
当然，除地址对齐外，局部变量的布局并没有顺序要求。

参数和返回值是通过伪寄存器FP定位的，
寄存器FP对应第一个参数的开始地址（第一个参数地址较低），
因此每个变量的偏移量是正数。
局部变量是通过伪寄存器SP定位的，
而伪寄存器SP对应的是第一个局部变量的结束地址（第一个局部变量地址较大），
因此每个局部变量的偏移量都是负数。
*/
func Foo() {
    var local [1]struct{
        a bool
        b int16
        c []byte
    }
    var SP = &local[1];
    _ = -(unsafe.Sizeof(local)-unsafe.Offsetof(local.a)) + uintptr(&SP) // a
    _ = -(unsafe.Sizeof(local)-unsafe.Offsetof(local.b)) + uintptr(&SP) // b
    _ = -(unsafe.Sizeof(local)-unsafe.Offsetof(local.c)) + uintptr(&SP) // c
}
```

#### 宏函数
```plan9

#define SWAP(x,y,t)MOVQ x,t;MOVQ y,x;MOVQ t,y

//func Swap(a, b int) (int, int)
TEXT ·Swap(SB), NOSPLIT, $0-32
  MOVQ a+0(FP),AX       //AX = a
  MOVQ b+8(FP),BX       //BX = b
  SWAP(AX,BX,CX)        //AX,BX = b,a
  MOVQ AX,ret0+16(FP)   //return
  MOVQ BX,ret1+24(FP)   //
  RET
  
```
预处理器可以通过条件编译针对不同的平台定义宏的实现，
这样可以简化平台带来的差异。

#### 控制流
```go
	//一般思维
	var a = 10
	fmt.Println(a)
	var b = (a + a) * a
	fmt.Println(b)
	
	//汇编思维
	var a, b int
	a = 10
	fmt.Println(a)
	b = a
	b += b
	b *= a
	fmt.Println(b)
```

```plan9

```
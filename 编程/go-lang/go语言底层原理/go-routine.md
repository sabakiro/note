## goRoutine泄露
```go
func main() {
    ch := func() <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                ch <- i
            }
        } ()
        return ch
    }()
    for v := range ch {
        fmt.Println(v)
        if v == 5 {
            break
        }
    }
}
```
上面的程序中后台Goroutine向通道输入自然数序列，
main()函数中输出序列。但是当break跳出for循环的时候，
后台Goroutine就处于无法被回收的状态了。
可以通过context包来避免这个问题：
```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    ch := func(ctx context.Context) <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                select {
                case <- ctx.Done():
                    return
                case ch <- i:
                }
            }
        } ()
        return ch
    }(ctx)
    for v := range ch {
        fmt.Println(v)
        if v == 5 {
        /*
	        当main()函数在break跳出循环时，
	        通过调用cancel()来通知后台Goroutine退出，
	        这样就避免了Goroutine的泄漏。
        */
            cancel()
            break
        }
    }
}
```
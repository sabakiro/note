切片会导致整个底层数组被锁定，底层数组无法释放内存。如果底层数组较大会对内存产生很大的压力：
```go
func main() {
    headerMap := make(map[string][]byte)
    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = data[:1]
    }
}
```

解决的方法是将结果克隆一份，这样可以释放底层的数组：
```go
	headerMap := make(map[string][]byte)
	for i := 0; i < 5; i++ {
		name := "./ciallo.txt"
		data, err := ioutil.ReadFile(name)
		if err != nil {
			log.Fatal(err)
		}
		//将结果克隆一份
		headerMap[name] = append([]byte{}, data[:1]...)
	}
```
## 循环

####  循环生命标签
```rust
fn main() {
    'search: loop {
        loop {
            break 'search;
        }
        println!("退出内层循环");
        break;
        println!("推出外层循环1");
    }
    println!("推出外层循环2");
}
//推出外层循环2
```

#### break
1. break;
2. break '标签;
3. break 变量;
4. break ‘标签 变量;

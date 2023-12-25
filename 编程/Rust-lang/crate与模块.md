
# 模块
项目结构
src
	aaa
		aa.rs
		mod.rs
	bin
		bbb
			main.rs
	lib.rs
	main.rs

lib.rs
```rust
pub mod p1{
    pub struct S{}
    pub fn ciallo() {}
    fn invisible_for_another(){}
    pub mod p2{}
    
}
pub mod aaa;
```
mod.rs
```rust
pub mod aa;
```
aa.rs
```rust
pub struct A{}
```
main.rs
```rust
use ciallo::aaa::aa::A;
use ciallo::aaa::{self, aa};
use ciallo::p1::{self, ciallo};
fn main() {}
```
### bin/
bbb/main.rs
```rust
fn main(){
	println!("ciallo from bin.bbb");
}
```
shell
```shell
cargo run --bin bbb
```
# 属性
### 出现警告
出现 `#[warn(non_camel_case_types)]`
可以使用`#allow`属性禁用此警告
```rust
#[allow(non_camel_case_types)]
pub struct git_aaaa{...}
```
### 条件编译
```rust
//只有当编译Android构建时才在项目中包含此模块
#[cfg(target_os = "android")]
mod mobile;
```

### 内联
```rust
//建议编译器对此函数进行内敛
#[inline]
fn aaaa() {
	 ...   
}
```
`#[inline(always)]`:要求函数在每个调用点都展开内敛
`#[inline(never)]`:要求函数永不内敛

### 将一个属性附属到整一个封闭区 `#!`
```rust
#![allow(non_camel_case_types)]

pub fn fn1(){...}
pub fn fn2(){...}
```

# 工作空间
src
	aaa
		Cargo.toml
		Cargo.lock
			src
				...
	Cargo.toml
src/Cargo.toml
```toml
...
[workspace]
members = ["aaa"]
```
shell
```shell
#构建当前工作空间中的所有crate
cargo build(可替代为test,doc) --workspace
```

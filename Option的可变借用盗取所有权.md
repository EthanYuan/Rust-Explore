# Option的take方法：可变借用窃取所有权

Rust的std库，内置了Option枚举定义。

```rust
fn main() {
    struct People{
        name: Option<String>,
    }

    let mut ethan = People{ name: Some(String::from("Hello")) };
    let alias = &mut ethan;
	
    // case 1: 可变引用的情况下，不能移动所有权
    // let name = alias.name;  // cannot move out of `alias.name` which is behind a mutable reference

    // case 2: 可变引用的情况下，移动了所有权
    let name_stolen = alias.name.take();
    
    dbg!(&ethan.name);  // [src\main.rs:13] &ethan.name = None
    dbg!(&name_stolen); // [src\main.rs:14] &name_stolen = Some("Hello",)
```

Enum作为Rust的一等公民，低头不见抬头见，比如Result、Option，前者包办了Rust的错误处理，后者担当了Rust无`null`的承诺。
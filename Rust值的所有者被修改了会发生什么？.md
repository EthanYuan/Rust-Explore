# Rust所有者被修改了会发生什么？

写C++的时候，指针都在明面上。到了Rust，指针在很多场合都藏了起来。但遗憾的是，它们并不是真的想被遗忘掉，而是在和你躲猫猫，最终你不得不把它们揪出来，游戏才能继续。

![](img/cat-hiding-under-rug.jpg)

## 1. 所有者被修改了会发生什么？

先让下面这段**看似没有指针**代码引出问题：

```rust
fn main(){
    let mut x = Box::new("ABC");
    println!("x is {}", &x);
    x = Box::new("XYZ");
    println!("x is {}", &x);
}
```

> x is ABC
>
> x is XYZ

问题可能你也猜到了：

- 第1行代码里我们在堆上存放“ABC”的内存，到底泄露了没有？
- 如果没有，是什么时候释放的？
- 如何证明？

好吧，还记得《The Rust Programming Language》里的**Ownership Rules**是这么说的：

> - Each value in Rust has a variable that’s called its *owner*.
> - There can only be one owner at a time.
> - When the owner goes out of scope, the value will be dropped.

可是此时此刻，即便是权威描述，也难免心生困惑了。

接下来，我们做个实验，尝试回答问题。

## 2. 利用智能指针的Drop trait

证明的思路是这样：

- 假设，Rust能正常的进行内存释放；
- 已知，`std::boxed::Box`是一个智能指针（结构体）；
- 而且，`std::boxed::Box`实现了`Drop trait`；
- 那么，实现一个自定义结构体的`Drop trait`；
- 接着，观察实例的Owner被修改时`std::ops::Drop::drop`被调用的时机点；
- 推理，得出上例中的“ABC”的释放时机；
- 完毕；

代码如下：

```rust
#[derive(Debug)]
struct MyPointer{}

impl Drop for MyPointer{
    fn drop(&mut self) {
        println!("Dropping MyPointer!");
    }
}

fn main(){
	let mut x = MyPointer{};
    println!("x is {:?}", &x);
    x = MyPointer{};
    println!("Now x is new {:?}", &x);    
}
```

输出：

> x is MyPointer					// 持有的值。
>
> Dropping MyPointer!		// Owner被修改时释放值。
>
> Now x is new MyPointer   // 新持有的值。
>
> Dropping MyPointer!		// 持有到花括号结束时释放值。

输出的顺序，即是我们想要的答案：

- 观察到，drop会在Owner被修改的第一时间被调用；
- 推理出，字符串“ABC”会在Owner被修改的第一时间被释放掉；

## 3. std::boxed::Box真正的实现

顺便看下`Drop for Box`的[源码](https://doc.rust-lang.org/src/alloc/boxed.rs.html#576-580)，实现居然是空的：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<#[may_dangle] T: ?Sized> Drop for Box<T> {
    fn drop(&mut self) {
        // FIXME: Do nothing, drop is currently performed by compiler.
    }
}
```

也不难理解，内置在Rust语言中的所有权机制，的确不应该由Drop trait来操心，否则自己实现Drop的程序员还得操心内存释放的问题，也就太low了。

虽然我们没有亲眼看到Rust释放内存的底层代码，但是能看到`drop`能在合适的时机点被触发已经足够了。

## 4. 小结

再回头看**Ownership Rules，**其实说的还是很精准，可以这么理解：因为当作为Owner的变量被修改后，堆上的值就相当于没有了Owner（突然消失在作用域中），那值自然也就被释放了。

**Rust的内存回收的确不用操心，高效且精准**。

通过这个例子，再次加深了我对Rust一个不同寻常的印象，就是：**Rust变量作用域 <= 花括号作用域**。无论是借用的生命周期的检查，还是上例中被修改的所有者，Rust编译器都会对其作用域尽早的进行判定，而不是等待花括号结束。
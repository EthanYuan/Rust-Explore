# 悬挂引用是如何被Rust消灭的？

![](D:\Writing\技术专栏\Rust\img\introducing-the-rust-borrow-checker.png)

**Rust承诺：引用始终有效**。

可是，Rust引用并没有堆变量的生杀大权“Ownership”，对于堆变量，只能借来用用，充其量借来改改（再还回去），那么Rust是如何保障引用的权益呢？

在面对悬挂引用问题之前，我们先复习下Rust引用。

## 一 引用的内存模型

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

上面代码里，堆上有一个String“hello”，在栈上有对应其所有权变量s1，以及一个临时的引用借用s。代码内存模型如下：

![&String s pointing at String s1](D:\Writing\技术专栏\Rust\img\ref.svg)

s和s1，是两种不同的类型，可以用下面的代码把类型打印来看。之所以s和s1用起来没差别，是因为引用s能自动解引用。

```rust
fn print_type_of<T>(_: &T) {
    println!("{}", std::any::type_name::<T>())
}

fn main() {
    let s1 = String::from("hello");
    let s = &s1;
    
    print_type_of(& s1);
	print_type_of(& s);
}
```

![type](D:\Writing\技术专栏\Rust\img\type.png)

## 二 悬挂引用问题

在C++里，当我们说到指针带来的内存安全问题时，就会提到

- 空指针（null pointer）：指针值为Null；
- 野指针（wild pointer）：未经初始化的“垃圾值”地址；
- 悬挂指针（dangling pointer）：指向已经释放的地址；

在Rust里，由于没有空值Null，所以并没有空引用问题；编译期进行初始化检查，所以也没有野引用问题。那么再看悬挂，Rust是否存在下面这种场景：当`s1`通过赋值将所有权转移给`s2`，`s`变成悬挂引用？

![Dangling](D:\Writing\技术专栏\Rust\img\Dangling.png)

答案是：不会。

**Rust必须在编译期就能检查出来引用的有效性**。

## 三 策略1：借用检查器检查引用的生命周期

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

引用一个Rust文档[The Rust Programming Language](https://doc.rust-lang.org/book/)的case，上面代码用注释，分别给出了引用`r`和数据`x`的生命周期。编译时当借用检查器发现，数据`x`的生命周期`'b`明显比`r`的生命周期`'a`短，为避免`r`成为悬挂引用，编译就无法通过，得到错误`error[E0597]: 'x' does not live long enough`。

**引用的生命周期，不能短于所引用数据的生命周期。**

**Rust会检查所有的可能性，包括控制条件里的所有可能路径**。下面代码在编译时依然会得到`error[E0597]: 'x' does not live long enough`：

```rust
{    
	let y = 6;              // --------------------+-- 'a
    let r;                  // ------+-- 'b        |
    {                       //       |             |     
        let x = 5;          // ------|-------+--'c |      
        if x > y{           //       |       |     |
            r = &x;         //       |       |     |
        }                   //       |       |     |
        else{               //       |       |     |
            r = &y;         //       |       |     |
        }                   //       |       |     |
    }                       // ------|-------+     |
    println!("r: {}", r);   //       |             |
                            // ------+             |
}                           // --------------------+
```

**为了更方便的理解引用的生命周期，我们可以考虑Rust黑话“借用”（borrow）的反面：归还（return）。如果一个“借用”没有再次使用，即视为“归还”。**

在文章[Rust所有权，可转可借](https://zhuanlan.zhihu.com/p/159456192)中，有个体现引用“借与还”的例子，即使是连续的进行不可变借用、可变借用，只要生命周期没有重叠，也可以编译通过：

```rust
{
    let mut x = String::from("Hello");      // ----------------+-- 'a
    x.push_str(", world");				    //                 |
    let r1 = &x;                            // -----'b	       |	     
    let r2 = &mut x;                        // -----'c		   |
    let r3 = &mut x;                        // ---+-'d         |
    r3.push_str("!");                       //    |            |
    println!("r3: {}", r3);                 //    |            |                         
                                            // ---+            | 
}                                           // ----------------+
```

## 四 策略2：函数定义中，不能返回所有权属于函数的引用

我们将策略1中的第1个例子，改成函数定义的场景：

```rust
fn test(r:&i32)-> &i32{
  let x = 5;    
  println!("r is: {}", r);
  &x
}  
```

和策略1的情况类似，但这次我们没有得到`error[E0597]: 'x' does not live long enough`，而是得到`error[E0515]: cannot return reference to local variable 'x'`。

**在函数里创建的数据，不能将其引用作为返回值。**因为函数调用结束后，所有权属于函数的数据，将会自动释放，这样会违反策略1。

据此，我们得到一条推论：**凡是函数返回的引用，都是参数传进来的**。

## 五 策略3：函数签名生命周期标注

### 编译器的局限

前两种策略对应的情况，编译器可以自己计算出引用的生命周期，并与实际数据生命周期进行比较，从而判断是否存在悬挂引用。但是，编译器并不总能做出判断。

```rust
{
    let y = 6;                  //----------------------+--'a
    let r1;                     //--------+--'c         |
    let r2;                     //--------|------+--'b  |
 	{                           //        |      |      |
        let x = 5;              //--+--'d |      |      |
        r1 = bigger(&x, &y);    //  |     |      |      |
        r2 = second(&x, &y);    //  |     |      |      |
        println!("{}", r1);     //  |     |      |      | 
    }                           //--+     |      |      |
    println!("{}", r2);         //--------+      |      |
                                //---------------+      |
}                               //----------------------+  
```

上面代码中，函数bigger和函数second把对&x和&y的操作进行了封装，那么在调用的这个上下文context中，就等于切断了&x、&y与r1和r2的直接关联。

**如果不是内联函数（inline），编译器在编译时并不会展开函数定义，所以此时Rust的借用检查器，并不知道函数bigger和second到底会返回什么，进而无法进行比较。**

借用检查器的困惑：

- `r1`的生命周期`'c`是和`x`的生命周期`'d`比呢？还是和`y`的生命周期`'a`比？
- `r2`的生命周期`'b`是和`x`的生命周期`'d`比呢？还是和`y`的生命周期`'a`比？

```rust
fn bigger(s: &i32, t: &i32) -> &i32{
    if s > t{
        s
    }
    else{
        t
    }
}

fn second(s: & i32, t: & i32) -> &i32{
    t
}
```

函数定义如上。进行编译，会得到`error[E0106]: missing lifetime specifier`，意思是“缺少生命周期标注”。

### 输入和输出生命周期标注

既然Rust编译器缺少判断依据，那么我们要怎么提供给它呢？对输入和输出生命周期进行人工标注。

标注如下，再次编译，通过。

```rust
fn bigger<'a>(s: &'a i32, t: &'a i32) -> &'a i32{
    if s > t{
        s
    }
    else{
        t
    }
}

fn second<'a>(s: & i32, t: &'a i32) -> &'a i32{
    t
}
```

在bigger函数签名上的标注表示：输入s和t，必须和返回值存活相同的时长。

在second函数签名上的标注表示：只有输入t，必须和返回值存活相同的时长。

### 标注规则

- **只需在函数签名上进行标注；**
- **生命周期用`'`开头，后面跟一个全小写字符，比如`'a`；**
- **用尖括号在函数名与参数列表之间声明泛型生命周期参数，例如`<'a>`；**
- **标注`'a`并不是一段具体的存活时长，只要满足约束关系即可；**
- **泛型`'a`会被具体化为x与y两者中生命周期较短的那一个；**

**生命周期标注的本质：解决函数调用导致的输入参数与输出的生命周期关系的断开，使之重新关联上。**

### 函数实现与签名标注的兼容

此时，不知道你的心里会不会还有最后一丝迟疑：如果我在函数签名上标注了泛型生命周期，谁来保证函数体实现确实遵循了这个标注呢？

答案是：Rust编译器保证。

还是前面的例子，函数的签名上，改成输入参数s，和输出标注相同的生命周期`'a`，但是实现上却返回了参数`t`，编译器报错：`error[E0621]: explicit lifetime required in the type of t`。

```rust
fn second<'a>(s: &'a i32, t: & i32) -> &'a i32{
    t
}
```

总的来说，基于函数签名的生命周期标注，联结了函数调用方和函数实现方，就像定义了一个标准，颇有依赖反转（DIP，Dependence Inversion Principle）的意味：

- 函数实现处，必须兼容签名，由Rust编译器进行检查；
- 函数调用处，必须遵守签名，由Rust编译器进行检查；

## 六 结尾

本文主要分析Rust消灭悬挂引用的核心策略，关于引用生命周期，还有很多细节和规则，可以参考[Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)。

基于此，Rust悬挂引用，无所遁形。

## 七 相关文章

- [Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)
- [RUST语言的编程范式](https://coolshell.cn/articles/20845.html)
- [Rust：编译时内存安全](https://isister.cc/posts/Rust-Compile-Time-Memory-Safety/)
- [回顾一下什么是内存安全](https://marvinsblog.net/post/2016-07-26-review-on-memory-safty/)








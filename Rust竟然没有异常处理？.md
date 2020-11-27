# Rust竟然没有异常处理？

学习Rust最好的方法，是和其他主流语言，比如Java、Python进行对比学习。不然怎么能get到它的特别呢？

![](.\img\special.png)

## 1. 主流模式：try-catch-finally

基本上，当你学会了某种语言的**try catch**，对这套机制的理解就能够迁移到其他语言上了。除了C++没有**finally**关键字外，像C#、Python、Java都有基本一致的异常处理逻辑：

- 用try块包住可能会出现的异常；
- 用catch将之捕获；
- finally块统一处理资源的清理；

```java
// Java

try{

}catch(FileNotFoundException f){

}catch(IOException i){

}finally{

}
```

对于自定义的函数，我们可以**throw**异常。

```java
// Java

import java.io.*;
public class ClassName
{
  public void deposit(double amount) throws RemoteException
  {
    // Method implementation
    throw new RemoteException();
  }
  //Remainder of class definition
}
```

在这种异常处理系统中，对**异常**的定义是比较宽泛的：**意料之外，情理之中**。正是“异常”在语义上的模糊性，才产生了很多最佳实践来指导异常的使用。从“正常到异常的程度”上，大致上可以归为4类：

**0 正常：不要用异常来进行流程控制，异常只用来处理“意外”**。

这条教导告诉我们，如果分不清“异常”，那么至少在“正常”的、没有意外的流程里，绝对不要用“异常机制来代替”。否则，代码可读性、可维护性将是灾难。

**1 人造语义异常：如果主流程中存在一个连续的“闯关”pipeline（一组按顺序的调用，成功执行才能执行下一个，否则都算失败），那么可以使用try块来集中放置主流程代码，catch块来集中处理失败情况，避免if-else箭头形代码。**

```java
try {
    getSomeThing_1();
    getSomeThing_2();
    getSomeThing_3();
catch(Exception e) {
    // deal with it
}
```

这个技巧，和**0 正常**容易产生冲突，因为似乎有流程控制的嫌疑。但是凡事都有例外。这里的“意外”可以理解成一种语义上的**“软意外”**——即不能出错，区别于非法字符、找不到文件、连接不上等**”硬意外“**。

**2 情理中的意外，可恢复。**

前面提到的非法字符、找不到文件、连接不上，基本是公认的“意外”情况，基本都使用抛出异常的方式，但是这种情况，通常都会进行捕获，并进行恢复。

**3 无法意料的致命意外，不可恢复。**

通常这种情况是：

- Bug：逻辑错误导致的溢出、除0；
- 致命错误：比如Java的JVM产生的Error；

## 2. Rust的Panic！

Rust里没有异常。

但如果非要和异常机制进行映射，Rust可以说做的相当决绝、非黑即白。

**0 正常，以返回值的形式。**

相当于压缩了上一节中的0、1、2项。没有什么情理中的意外，网络连不上、文件找不到、非法输入，统统都用返回值的方式。

**1 致命错误，不可恢复，非崩不可。**

一旦存在不可恢复的错误，Rust使用Panic！宏来终止程序（线程）。一旦Panic！宏出手，基本没得救（panic::catch_unwind是个例外，稍后说）。执行时默认会进行stack unwind（栈反解），一层层上去，直到线程的顶端。

有些情况Panic！是你的程序所依赖的库产生的，比如数组越界访问时的实现。

另一种情况，是你自己的程序逻辑判断产生了不可恢复的错误，可以手动触发Panic！宏来终止程序。Panic！的使用与throw很类似。

我写了一个小例子：打开一个文本文件，在写入之前，把它删掉，不仅没有收到Panic！，返回值错误也没有，居然写成功了。看来，这在Rust都不算事儿。着实让我惊讶了一小会儿。

```rust
use std::io::prelude::*;
use std::thread;
use std::time;
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
    let mut f = OpenOptions::new().write(true).open("hello.txt")?;
    print!("{:?} \n", f);

	// on the moment, manually remove the file hello.txt
    let ten_millis = time::Duration::from_millis(10000);
    thread::sleep(ten_millis);

    print!("{:?} \n", f);
    
    let r = f.write_all(b"Hello, world!")?;
    print!("Result is {:?} \n", r);

    drop(f);

    Ok(())
}
```

输出如下：

看File结构，同一个句柄handle，但是path前后却发生了变化，文件都进回收站了，照样写你！

![](.\img\panic.png)

## 3. Rust的返回值Result

前面提到了，对于可恢复的错误，Rust一律使用返回值来进行检查，而且提倡采用内置枚举**Result**，还在实践层面给了一定的约束：**对于返回值为Result类型的函数，调用方如果没有进行接收，编译期会产生警告。**很多库函数都通过Result来告知调用方执行结果，让调用方来决定是否严重到了使用Panic！的程度。

Result枚举的泛型定义如下：

```rust
enum Result<T, E>{
    Ok(T),
    Err(E),
}
```

在Rust标准库中，可以找到许多以Result命名的类型，它们通常是Result泛型的特定版本，比如File::open的返回值就是把T替换成了std::fs::File，把E替换成了std::io::Error。

**枚举可以携带某个类型的数据，是Rust非常与众不同的特性**。

在上面的例子中，可能会有个疑问：并没有看到对Result的检查？

仔细看下，机关就在于最后的那个"**?**"

```rust
let mut f = OpenOptions::new().write(true).open("hello.txt")?;
```

或许是Rust对于“需要大量的返回值检查”的介意，于是有了“?”快捷运算符。

它可以避免模板代码。上面1行顶下面4行：

```rust
let f = OpenOptions::new().write(true).open("hello.txt")?;
let mut f = match f{
    Ok(file) => file,
    Err(e) => return Err(e),
};
```

## 4. panic::catch_unwind

最后，再来说个例外，panic::catch_unwind。

先看下它的用法：

```rust
use std::panic;

let result = panic::catch_unwind(|| {
    println!("hello!");
});
assert!(result.is_ok());

let result = panic::catch_unwind(|| {
    panic!("oh no!");
});
assert!(result.is_err());
```

没错，它的行为几乎就是try/catch了。panic！宏也被捕获了，程序并也没有挂，返回了Err。尽管如此，Rust的目的并不是让它成为try/catch机制的实现，而是当Rust和其他编程语言互动时，避免其他语言代码块throw出异常。所以呢，错误处理的正道还是用**Result**。

从catch_unwind的名字上，需要留意下unwind这个限定词，它意味着只有默认进行栈反解的panic可以被捕获到，如果是设为直接终止程序的panic，就逮不住了。

细节可进一步参考[Rust Documentation](https://doc.rust-lang.org/beta/std/panic/fn.catch_unwind.html)。


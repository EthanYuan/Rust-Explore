# 【译文】Rust futures: async fn中的thread::sleep和阻塞调用 

![async](.\img\async.jpeg)

> 原文：[Rust futures: thread::sleep and blocking calls inside async fn](https://blog.hwc.io/posts/rust-futures-threadsleep-and-blocking-calls-inside-async-fn/)
>
> URL:    https://blog.hwc.io/posts/rust-futures-threadsleep-and-blocking-calls-inside-async-fn/

近来，关于Rust的`futures`和`async/await`如何工作（“blockers”，哈哈），我看到存在一些普遍的误解。很多新用户为`async/await`带来的重大改进而感到兴奋，但是却被一些基本问题所困扰。即使有了`async/await`，并发依然很难。文档还在进一步充实，阻塞/非阻塞之间的交互很棘手。希望本文对你有所帮助。

（本篇主要是关于特定的痛点；有关Rust中的异步编程的概述，请转至[本书](https://rust-lang.github.io/async-book/index.html)）

TLDR（Too Long Didn't Read）：小心在`async fn`中使用昂贵的阻塞调用！如果不确定， 鉴于Rust `std`库中几乎所有都是阻塞的，所以就要注意哪些调用是耗时的！

虽然我认为任何人都可能犯这个错误（在引入足够的负载来显著地阻塞线程之前，往往察觉不到），但是初学者尤为如此。下面的场景可能有点冗长，但我认为有必要展示一下在`async fn`中实现阻塞调用是多么容易。

## 不要用 `std::thread::sleep` sleep

在研究了一个简单的示例之后，Rust异步新手可能要做的第一件事就是去验证程序真正实现了异步。因此，我们使用Rust异步书籍中的示例：

```rust
use futures::join;

async fn get_book_and_music() -> (Book, Music) {
    let book_fut = get_book();
    let music_fut = get_music();
    join!(book_fut, music_fut)
}
```

即使你在get_book和get_music内部打日志，也无法通过简单的方式来判断它们是同时运行的，因为任何一次运行都可能产生恰好与代码顺序匹配的输出。你必须多次运行该程序，才能查看日志记录顺序是否可以翻转（如果不翻转怎么办？）。

如果想看到get_book和get_music是100％同时运行，你可能会想到记录它们的开始时间，并查看开始时间是否相同。但是，等等，如果开始时间仍然是串行的，但fn运行得如此之快，看起来仍然像是并发该怎么办？

引入一个延迟！比如（清楚起见，使用伪码）：

```rust
async fn get_book() {
    println!("book start: time {}", current_time());
    std::thread::sleep(one_second);
    println!("book end: time {}", current_time());
}
```

在get_book和get_music内部延迟1秒，我们希望，如果是并发的话，则会看到以下的输出：

> book start: time 0.00 
>
> music start: time 0.00 
>
> book end: time 1.00 
>
> music start: time 1.00

如果是串行，我们预期是：

>  book start: time 0.00 
>
> book end: time 1.00 
>
> music start: time 1.00 
>
> music end: time 2.00

你认为会发生什么， 串行或并发？

你已经读了这篇文章的标题，可能会猜到get_book和get_music是按顺序执行的。但为什么！？异步fn中的所有内容不是都应该同时运行吗？

在继续解释之前，可以看个问题已经多次被问到： 

[reddit 1](https://old.reddit.com/r/rust/comments/e1gxf8/not_understanding_asyncawait_properly/) [reddit 2](https://old.reddit.com/r/rust/comments/dtp6z7/what_can_i_actually_do_with_the_new_async_fn/) [reddit 3](https://old.reddit.com/r/rust/comments/dt0ruy/how_to_await_futures_concurrently/) [stackoverflow 1](https://stackoverflow.com/questions/52313031/how-to-run-multiple-futures-that-call-threadsleep-in-parallel) 

因此，如果你也犯了这个错误，不用担心，其他许多人也有同样的经历。

## 什么搞错了？为什么async不行？

我不会在这里深入讨论`futures`和`async/await`（[本书](https://rust-lang.github.io/async-book/index.html)是一个很好的起点）。我只想指出造成困惑的两个可能的根源：

#### `std::thread::sleep` 会阻塞？

对于新手来说，`std::thread::sleep`会造成阻塞可能并不是显而易见的。尽管事后看起来很明显，但是当尝试掌握全新的程序执行范式时，却很容易忽略。 

即使你大致了解并发，也可能不知道`thread::sleep`是具体如何实现的。一些上层推理加上一些示例（例如上述）可能会帮助你理解。但是文档中并没有明说“*此调用是阻塞的，你不应该在异步上下文中使用它*”，并且非系统程序员可能不会过多地考虑“*将当前线程置于睡眠状态*”。

（具有讽刺意味的是，如果人们的异步编程的心智模型是让`Future`进入“睡眠”状态从而得以让其他工作发生，那么`thread::sleep`可能会特别令人困惑）。

####  `async` 可以做什么？

但是有些人可能会说：“如果`thread::sleep`阻塞了怎么办？不是把它放在`async fn`中就好了吗？”

为了理解那些在线讨论，（就要知道）他们的想法是以为`async`可以使代码块或函数内部的所有内容异步。

 首先，我想说这是有意义的；`async/await`存在的部分原因是它使每个人都容易进行异步操作。而且，如果你从较高的层次上理解了并发模型（事件循环，通常是尝试不阻塞线程），那么可能没有特定的理由导致`async`不能仅仅通过使事物定义为异步来起作用。那绝对是最简单，最符合人体工程学的方式。 

不幸的是，这不是Rust的`async`范式的工作方式。`async`功能很强大，但从本质上讲，它只是提供了一种更好的处理`Future`s的方法。而且`Future`不只是自动将阻塞调用移到一边以允许完成其他工作；它要结合使用具备轮询和异步运行时这种完全独立的系统，才能进行异步舞蹈。在该系统内进行的任何阻塞调用仍将处于阻塞状态。 

这可能会造成一些困惑，因为`async/await`允许我们编写看起来更像常规（阻塞）代码的代码。那就是`async/await`的`await`部分进入的地方。当你在`async`块中`await`future时，它能够将自己安排在线程外并为其他任务让路。阻塞代码可能看起来很相似，但是由于它不是future，所以无法`await`，也无法为其他任务腾出空间。

因此，下面不会阻塞，但是`await`可以让你编写看起来与阻塞调用非常相似的代码：

```rust
async {
    let f = get_file_async().await;
    let resp = fetch_api_async().await;
}
```

下面在每行调用时阻塞：

```rust
async {
    let f = get_file_blocking();
    let resp = fetch_api_blocking();
}
```

下面将不能通过编译：

```rust
async {
    let f = get_file_blocking().await;
    let resp = fetch_api_blocking().await;
}
```

在这里什么也没有发生（您必须在`async`内部`await`futures！）:

```rust
async {
    let f = get_file_async();
    let resp = fetch_api_async();
}
```

总的来说，最好将`async`视为允许在函数或块中 `await` 的东西，但实际上并不会使任何东西异步。

## 如何不阻塞

如果想要异步fn取消阻塞该怎么办？ 

你可以找到一个异步替代方案：当`thread::sleep`阻塞时，你可以使用它们（取决于你选择的运行时生态系统）：

- [async_std::task::sleep (1.0)](https://docs.rs/async-std/1.1.0/async_std/task/fn.sleep.html)
- [tokio::time::delay_for (0.2.0)](https://docs.rs/tokio/0.2.0/tokio/time/fn.delay_for.html)

`tokio`和`async_std`都为其他阻塞操作（例如文件系统和tcp流访问）提供了异步替代方法。

另一个选择是将阻塞调用移到另一个线程。

- [tokio::task::spawn_blocking (0.2.0)](https://docs.rs/tokio/0.2.0/tokio/task/fn.spawn_blocking.html)
- [async_std::task::spawn_blocking (1.0)](https://docs.rs/async-std/1.1.0/async_std/task/fn.spawn_blocking.html)

这要求你的运行时具有专用于卸载阻塞调用的机制（例如线程池）。

我还提出了一些问题，试图防止其他人陷入这个陷阱：

- [async-book](https://github.com/rust-lang/async-book/issues/64)
- [clippy](https://github.com/rust-lang/rust-clippy/issues/4377)

## 结语

希望该博客能够阐明有关阻塞调用如何与Rust的并发模型进行交互的一些信息！随时提供反馈给我。




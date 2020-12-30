# 回撸Rust China Conf 2020 之《浅谈Rust在算法题和竞赛中的应用》

很难通过某种单一的方式，就能get到所有Rust技能，学习的方式方法要多样化：

- 循序渐进的系统性学习（内存管理->类型系统->所有权）
- 主题学习（异步、宏）
- 交流学习（开发者大会、社区）
- 刻意练习（LeetCode）

刚刚结束的首届Rust China Conf 2020就是一种交流学习的方式。Rust中文社区采用直播并提供视频回放，为所有Rustacean提供了绝佳的、宝贵的学习资料。

本篇回撸一把《浅谈Rust在算法题和竞赛中的应用》，琳琅满目的特性和应用，让人爱不释手。

Speaker: Wu Aoxiang ([吴翱翔](https://github.com/pymongo))

视频：[Day2](https://live.csdn.net/room/u012067469/51UUkkjG) ，03:54:00~04:20:00

### 1 [std::iter::Iterator::peekable](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.peekable)

很实用的迭代器能力，标准库的注释如下：

Creates an iterator which can use [`peek`](https://doc.rust-lang.org/std/iter/struct.Peekable.html#method.peek) to look at the next element of the iterator without consuming it. 

`fn peekable(self) -> Peekable<Self>`

```rust
let xs = [1, 2, 3];

let mut iter = xs.iter().peekable();

// peek() lets us see into the future
assert_eq!(iter.peek(), Some(&&1));
assert_eq!(iter.next(), Some(&1));
```

### 2 ?的Option解包能力的应用：LeetCode 7 整数反转。

```rust
impl Solution {
    pub fn reverse(mut x: i32) -> i32 {
        || -> Option<i32> {
            let mut ret = 0i32;
            while x.abs() != 0 {
                ret = ret.checked_mul(10)?.checked_add(x%10)?;
                x /= 10;
            }
            Some(ret)
        }().unwrap_or(0)
    }
}
```

### 3 std::iter::Iterator::fold的应用：LeetCode 1486 数组异或操作

```rust
impl Solution {
    pub fn xor_operation(n: i32, start: i32) -> i32 {
        (1..n).fold(start, |acc, i| { acc ^ start + 2*i as i32 })
    }
}
```

### 4 std::net::IpAddr的应用：LeetCode 468 验证IP地址

IpAddr是个枚举类型，能通过String::parse直接进行解析。以下PPT上的代码并不能通过，所以我猜讲者只想给出一个算法框架方便做演示。

```rust
use std::net::IpAddr;

impl Solution {
    pub fn valid_ip_address(ip: String) -> String {
        match ip.parse::<IpAddr>() {
            Ok(IpAddr::V4(_)) => String::from("IPv4"),
            Ok(IpAddr::V6(_)) => String::from("IPv6"),
            _ => String::from("Neither"),
        }
    }
}
```

添加检查后可以通过测试，代码如下。可见标准库对IP地址的合法性还是比较宽容的。

```rust
use std::net::IpAddr;

impl Solution {
    pub fn valid_ip_address(ip: String) -> String {
        match ip.parse::<IpAddr>() {
            Ok(IpAddr::V4(x)) => {
                let array: Vec<Vec<char>> = ip.split('.').map(|x| x.chars().collect()).collect();
                for i in 0..array.len() {
                    if (array[i][0] == '0' && array[i].len() > 1) { return String::from("Neither"); }
                }
                String::from("IPv4")
            },
            Ok(IpAddr::V6(_)) => {
                let array: Vec<Vec<char>> = ip.split(':').map(|x| x.chars().collect()).collect();
                for i in 0..array.len() {
                    if array[i].len() == 0  { return String::from("Neither"); }
                }
                String::from("IPv6")
            },
            _ => String::from("Neither"),
        }
    }
}
```

### 5 Rust调用C函数

调用C函数的能力，使得Rust的能力范围又扩展了。

```rust
extern "C" {
    fn rand() -> i32;
}

fn main() {
    let rand = unsafe { rand() };
    println!("{}", rand);
}
```

### 6 String零开销提供数组访问：std::string::String::into_bytes

从源码来看，String提供字节数组访问，简直不费吹灰之力。在ASCII范围的场景（大多数LeetCode字符串题目），每个字节通常对应一个拉丁字符，CRUD都非常方便。

源码如下：

This consumes the `String`, so we do not need to copy its contents.

```rust
#[inline]
#[stable(feature = "rust1", since = "1.0.0")]
pub fn into_bytes(self) -> Vec<u8> {
    self.vec
}
```

需要注意的是，如果字符串涉及到国际化的时候，一个字节可能已经不再能映射一个字符了（比如中文字需要多字节存储），此时直接在字节层面进行CRUD都是非常危险的。

### 7 饱和运算（防溢出）

各类型的整数和浮点数都有saturating运算系列，以u8减法为例：

`pub const fn saturating_sub(self, rhs: u8) -> u8`

```rust
assert_eq!(100u8.saturating_sub(27), 73);
assert_eq!(13u8.saturating_sub(127), 0);
```

讲者在分析LeetCode 《1512 好数对的数目》一题中应用了该方法。但是就该题目来说，本文给出一种更加简单的解法，一次迭代即可。

同时讲者还使用了`cargo bench`，来得到该方法纳秒级的运行时间，比LeetCode的毫秒级要精确1000倍了。

这里需要注意的是，尽管Rust 2018已经放弃了extern crate，还是需要使用`extern crate test`来引入test "sysroot" crates。这是[例外](https://doc.rust-lang.org/nightly/edition-guide/rust-2018/module-system/path-clarity.html#an-exception)。

```rust
#![feature(test)]
extern crate test;

pub fn num_identical_pairs(nums: Vec<i32>) -> i32 {
    let mut map: [i32;101] = [0;101];
    nums.into_iter().fold(0, |mut acc, x| { 
        acc += map[x as usize];
        map[x as usize] += 1;
        acc })
}

#[cfg(test)]
mod tests {
    use test::Bencher;
    use super::*;
    #[bench]
    fn bench_func(b: &mut Bencher) {
        b.iter(|| {
            num_identical_pairs(vec![1,2,3,1,1,3,1]);
        });
    }
}
```

输出：

```powershell
PS D:\Project\rust_II\Language\r10_64_benchmark> cargo bench
    Finished bench [optimized] target(s) in 0.02s
     Running target\release\deps\r10_64_benchmark-8eb89c8ab9ad043d.exe

running 1 test
test tests::bench_func ... bench:          81 ns/iter (+/- 74)

test result: ok. 0 passed; 0 failed; 0 ignored; 1 measured; 0 filtered out
```

### 8 善用cargo clippy

使用clippy，多多益善。[cargo clippy](https://github.com/rust-lang/rust-clippy)是CI友好的，如果你是眼睛里容不下warning的人，可以设置使任何warning导致编译失败：

`cargo clippy -- -D warnings`
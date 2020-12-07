# Rust API 指南：文档

![](img/Art-shipping-crate.jpg)

> 原文：[Rust API Guidelines chapter 4](https://rust-lang.github.io/api-guidelines/documentation.html#rustdoc-does-not-show-unhelpful-implementation-details-c-hidden)

### Crate级别的文档应非常详尽，并包含示例（C-CRATE-DOC）

见[RFC1687](https://github.com/rust-lang/rfcs/pull/1687).

### 所有项目都应有一个rustdoc示例（C-EXAMPLE）

每个公共模块，特型，结构，枚举，函数，方法，宏和类型定义都应具有一个示例，用于该功能的练习。

该准则应在合理范围内适用。

有时，附上另一个条目的适用示例的链接可能就足够了。例如，如果恰好一个函数使用特定类型，则可以在该函数或类型上编写单个示例后，从另一个链接到该示例。

示例的目的并不总是显示如何使用该条目。虽然读者希望了解如何调用函数，在枚举上进行匹配，以及一些基本任务。但是，一个示例最应该表明为什么要使用这个条目。

```rust
// 这是使用clone()的不良示例。它机械地显示*如何*
// 调用clone()，但没有显示出*为什么*要这样做。
fn main() {
    let hello = "hello";

    hello.clone();
}
```



### 示例使用？，不是try !，也不是unwrap（C-QUESTION-MARK）

不管喜欢与否，用户通常逐字复制示例代码。Unwrapping an error应该是用户需要做出的决定。

下面是这种常见的方式会构建易出错的示例代码。以＃开头的行是在构建示例时通过cargo test编译的，但不会出现在用户可见的rustdoc中。

```rust
/// ```rust
/// # use std::error::Error;
/// #
/// # fn main() -> Result<(), Box<dyn Error>> {
/// your;
/// example?;
/// code;
/// #
/// #     Ok(())
/// # }
///
```

### 函数文档应包括错误，恐慌和安全注意事项（C-FAILURE）

错误条件应记录在“错误”部分中。这也适用于trait方法--实现允许或预期返回错误的trait方法应在“错误”部分进行记录。 

例如在标准库中，`std::io::Read::read trait`方法的某些实现可能返回错误。

```rust
/// 从该源提取一些字节到指定的缓冲区中，返回
/// 读取了多少字节。
///
/// ... lots more info ...
///
/// # Errors
///
/// 如果此函数遇到任何形式的I/O或其他错误，错误
/// 变体将返回。如果返回错误，则必须
/// 保证不会读取任何字节。
```

恐慌情况应记录在“恐慌情况”部分。这也适用于trait方法-实现允许或预期产生恐慌的traits方法应在“ Panics”部分记录。 

在标准库中，`Vec::insert`方法可能会出现恐慌。

```rust
/// 在向量中的索引位置处插入一个元素，将
/// 它后面的所有元素向右移位。
///
/// # Panics
///
/// 如果`index`超出范围，则产生恐慌。
```

不必记录所有可能的恐慌情况，特别是如果恐慌情况发生在调用方提供的逻辑中。例如，在以下代码中记录`Display`恐慌似乎过多。但如果不确定，也不是记录更多恐慌情况就更好。

```rust
/// # Panics
///
/// This function panics if `T`'s implementation of `Display` panics.
pub fn print<T: Display>(t: T) {
    println!("{}", t.to_string());
}
```

不安全的函数应记录在“安全性”部分，该部分说明了由调用者负责维护正确使用该函数的所有不变量。

不安全的`std::ptr::read`需要以下调用者。

```rust
/// 从`src`中读取值而不移动它。这使得
/// `src`中的内存不变。
///
/// ＃ 安全
///
///	除了接受原始指针之外，这是不安全的，因为它在语义上
///	将值移出src，而不阻止未来使用src。
///	如果`T`不是`Copy`，则必须注意确保`src`的值被
///	数据再次覆盖之前不会使用（例如，使用`write`，
///	`zero_memory`或`copy_memory`）。注意，`*src = foo`也算使用
///	，因为它将尝试把先前的值放在`*src`处。
///
///	指针必须对齐；如果不是这种情况，请使用`read_unaligned`。
```

### 包含指向相关内容的超链接（C-LINK）

链接到相同类型内的方法通常如下所示：

``` rust
[`serialize_struct`]: #method.serialize_struct
```

指向其他类型的链接通常如下所示

```rust
[`Deserialize`]: trait.Deserialize.html
```

链接也可能指向父模块或子模块：

```rust
[`Value`]: ../enum.Value.html
[`DeserializeOwned`]: de/trait.DeserializeOwned.html
```

RFC 1574在["Link all the things"](https://github.com/rust-lang/rfcs/blob/master/text/1574-more-api-documentation-conventions.md#link-all-the-things)标题下正式建议该准则。

### Cargo.toml包含所有常见的元数据（C-METADATA）

Cargo.toml的[package]部分应包含以下值：

- `authors`
- `description`
- `license`
- `repository`
- `readme`
- `keywords`
- `categories`

此外，还有两个可选的元数据字段：

- `documentation`
- `homepage`

默认情况下，crates.io会把crate的文档链接到[docs.rs](https://docs.rs/)上。仅当文档托管在docs.rs以外的其他位置时，才需要设置`documentation`元数据，例如，因为crate链接到了docs.rs构建环境中不可用的共享库。  

仅在有唯一的网站而不是代码库或API文档的情况下设置`homepage`元数据。不要使用`documentation`或`repository`值填充`homepage`。比如，serde将`homepage`设置为专用网站https://serde.rs

#### Crate设置html_root_url属性（C-HTML-ROOT）

假设crate使用docs.rs作为其主要API文档，则它应指向`"https://docs.rs/CRATE/MAJOR.MINOR.PATCH"`。 

`html_root_url`属性告诉rustdoc在编译下游crates时如何为crate中的项目创建URL。没有它，依赖于您的crate的crate文档中的链接将不正确。

```rust
#![doc(html_root_url = "https://docs.rs/log/0.3.8")]
```

由于此URL包含确切的版本号，因此必须与`Cargo.toml`中的版本号保持同步。 [`version-sync`](https://crates.io/crates/version-sync)的crate可以帮助您解决此问题，方法是让您添加一个集成测试，如果`html_root_url`版本号与crate版本不同步，则该集成测试将失败。 

如果您不喜欢这种机制，建议在Cargo.toml版本密钥中添加一条注释，提醒您自己将两者保持更新，例如：

```rust
version = "0.3.8" # remember to update html_root_url
```

对于在docs.rs外部托管的文档，如果在crate名称+ index.html后面的附加带您到crete根模块的文档，则`html_root_url`的值正确。例如，如果根模块的文档位于`"https://api.rocket.rs/rocket/index.html"`，则`html_root_url`将为`"https://api.rocket.rs"`。

### Release notes记录所有重大更改（C-RELNOTES）

crate的用户可以阅读release notes，以找到crate每个已发行版本中发生更改的摘要。crate级文档和/或Cargo.toml中链接的存储库中应包含release notes的链接或说明本身。 

release notes中应明确标识重大更改（如[RFC 1105](https://github.com/rust-lang/rfcs/blob/master/text/1105-api-evolution.md)中所定义）。 

如果使用Git跟踪crate的源代码，则发布到crates.io的每个发行版都应具有一个相应的tag，用于标识已发布的提交。非Git VCS工具也应使用类似的过程。

```rust
# Tag the current commit
GIT_COMMITTER_DATE=$(git log -n1 --pretty=%aD) git tag -a -m "Release 0.3.0" 0.3.0
git push --tags
```

 首选带注释的标签，因为如果存在任何带注释的标签，则某些Git命令会忽略未注释的标签。

#### Example

- [Serde 1.0.0 release notes](https://github.com/serde-rs/serde/releases/tag/v1.0.0)
- [Serde 0.9.8 release notes](https://github.com/serde-rs/serde/releases/tag/v0.9.8)
- [Serde 0.9.0 release notes](https://github.com/serde-rs/serde/releases/tag/v0.9.0)
- [Diesel change log](https://github.com/diesel-rs/diesel/blob/master/CHANGELOG.md)

### Rustdoc不要显示无用的实现细节（C-HIDDEN）

Rustdoc应该包括用户完全使用crate所需的一切。可以在技术文章中解释相关的实现细节，但是它们不应该是文档中的真实条目。  

尤其要选择在rustdoc可以看到哪些实现--所有用户需要使得能完全使用crate。在以下代码中，默认情况下，`PublicError`的rustdoc将显示`From <PrivateError>` impl。我们选择使用`#[doc(hidden)]`隐藏它，因为用户的代码中永远不会出现`PrivateError`，因此该隐含内容永远与他们无关。

```rust
// This error type is returned to users.
pub struct PublicError { /* ... */ }

// This error type is returned by some private helper functions.
struct PrivateError { /* ... */ }

// Enable use of `?` operator.
#[doc(hidden)]
impl From<PrivateError> for PublicError {
    fn from(err: PrivateError) -> PublicError {
        /* ... */
    }
}
```

[`pub(crate)`](https://github.com/rust-lang/rfcs/blob/master/text/1422-pub-restricted.md) 是另一个用于从公共API删除实现细节的好工具。它允许项目从其自身模块的外部使用，但不能在同一crate外部使用。
## Rust组合器

![](img\Combinator.png)

原文：[Learning Rust Error Handling Combinators](https://learning-rust.github.io/docs/e6.combinators.html)

## 什么是组合器？

- “组合器”有一种非正式的含义，指的是**组合器模式**，一种以组合事物为中心思想来组织库的方式。通常，会有个**类型T**，一些**用于构造T类型“原”值的函数**，以及一些**“组合器”**，它们可以通过各种方式**组合T类型的值**以**建立更复杂的T类型的值**。另一个定义是**没有自变量的函数**。

  __ [wiki.haskell.org](https://wiki.haskell.org/Combinator)

- 组合器是一个**从程序片段构建程序片段的函数**；从某种意义上说，使用组合器的程序员自动化地构造很多所需的程序，而不是手工编写每个细节。

  __ John Hughes—[Generalizing Monads to Arrows](http://www.cse.chalmers.se/~rjmh/Papers/arrows.pdf) via [Functional Programming Concepts](https://github.com/caiorss/Functional-Programming/blob/master/haskell/Functional_Programming_Concepts.org)

Rust生态系统中“组合器”的确切定义还不太清晰。

- `or()`, `and()`, `or_else()`, `and_then()`
  - **组合两个类型为T **的值并**返回相同的类型T **。
- `filter()` for `Option` types
  - **使用闭包作为条件函数过滤类型T **。
  - **返回相同的类型T**。
- `map()`, `map_err()`
  - **通过闭包转换类型T**.
  -  **可以更改T中值的数据类型**。
    例如 `Some<&str>` 转换为 `Some<usize>` 或者 `Err<&str>` to `Err<isize>` 等。
- `map_or()`, `map_or_else()`
  - **通过应用闭包来转换T类型**，并**返回T类型内部的值**。
  - 对于 **`None` 和 `Err`，需要一个默认值或者一个闭包**。
- `ok_or()`, `ok_or_else()` for `Option` types
  - **将 `Option` 转为 `Result` **.
- `as_ref()`, `as_mut()`
  - **将类型T转换为引用或可变引用**

## or()和and()

组合两个返回值为`Option/Result`的表达式

- `or()`：如果其中一个得到了`Some`或`Ok`，该值将立即返回。
- `and()`：如果两个都获得`Some`或`Ok`，则返回第二个表达式的值。如果其中一个为`None`或`Err`，则该值立即返回。

```rust
fn main() {
  let s1 = Some("some1");
  let s2 = Some("some2");
  let n: Option<&str> = None;

  let o1: Result<&str, &str> = Ok("ok1");
  let o2: Result<&str, &str> = Ok("ok2");
  let e1: Result<&str, &str> = Err("error1");
  let e2: Result<&str, &str> = Err("error2");

  assert_eq!(s1.or(s2), s1); // Some1 or Some2 = Some1
  assert_eq!(s1.or(n), s1);  // Some or None = Some
  assert_eq!(n.or(s1), s1);  // None or Some = Some
  assert_eq!(n.or(n), n);    // None1 or None2 = None2

  assert_eq!(o1.or(o2), o1); // Ok1 or Ok2 = Ok1
  assert_eq!(o1.or(e1), o1); // Ok or Err = Ok
  assert_eq!(e1.or(o1), o1); // Err or Ok = Ok
  assert_eq!(e1.or(e2), e2); // Err1 or Err2 = Err2

  assert_eq!(s1.and(s2), s2); // Some1 and Some2 = Some2
  assert_eq!(s1.and(n), n);   // Some and None = None
  assert_eq!(n.and(s1), n);   // None and Some = None
  assert_eq!(n.and(n), n);    // None1 and None2 = None1

  assert_eq!(o1.and(o2), o2); // Ok1 and Ok2 = Ok2
  assert_eq!(o1.and(e1), e1); // Ok and Err = Err
  assert_eq!(e1.and(o1), e1); // Err and Ok = Err
  assert_eq!(e1.and(e2), e1); // Err1 and Err2 = Err1
}
```

> ```rust
> 🔎Rust nightly支持Option类型的xor()，它仅在一个表达式获得Some时才返回Some，但两个则不然。
> ```

## or_else()

类似于`or()`。唯一的区别是，第二个表达式应是一个返回相同类型T的闭包。

```rust
fn main() {
    // or_else with Option
    let s1 = Some("some1");
    let s2 = Some("some2");
    let fn_some = || Some("some2"); // similar to: let fn_some = || -> Option<&str> { Some("some2") };

    let n: Option<&str> = None;
    let fn_none = || None;

    assert_eq!(s1.or_else(fn_some), s1);  // Some1 or_else Some2 = Some1
    assert_eq!(s1.or_else(fn_none), s1);  // Some or_else None = Some
    assert_eq!(n.or_else(fn_some), s2);   // None or_else Some = Some
    assert_eq!(n.or_else(fn_none), None); // None1 or_else None2 = None2

    // or_else with Result
    let o1: Result<&str, &str> = Ok("ok1");
    let o2: Result<&str, &str> = Ok("ok2");
    let fn_ok = |_| Ok("ok2"); // similar to: let fn_ok = |_| -> Result<&str, &str> { Ok("ok2") };

    let e1: Result<&str, &str> = Err("error1");
    let e2: Result<&str, &str> = Err("error2");
    let fn_err = |_| Err("error2");

    assert_eq!(o1.or_else(fn_ok), o1);  // Ok1 or_else Ok2 = Ok1
    assert_eq!(o1.or_else(fn_err), o1); // Ok or_else Err = Ok
    assert_eq!(e1.or_else(fn_ok), o2);  // Err or_else Ok = Ok
    assert_eq!(e1.or_else(fn_err), e2); // Err1 or_else Err2 = Err2
}
```

## and_then()

与`and()`类似。唯一的区别是，第二个表达式应是一个返回相同类型T的闭包。

```rust
fn main() {
    // and_then with Option
    let s1 = Some("some1");
    let s2 = Some("some2");
    let fn_some = |_| Some("some2"); // similar to: let fn_some = |_| -> Option<&str> { Some("some2") };

    let n: Option<&str> = None;
    let fn_none = |_| None;

    assert_eq!(s1.and_then(fn_some), s2); // Some1 and_then Some2 = Some2
    assert_eq!(s1.and_then(fn_none), n);  // Some and_then None = None
    assert_eq!(n.and_then(fn_some), n);   // None and_then Some = None
    assert_eq!(n.and_then(fn_none), n);   // None1 and_then None2 = None1

    // and_then with Result
    let o1: Result<&str, &str> = Ok("ok1");
    let o2: Result<&str, &str> = Ok("ok2");
    let fn_ok = |_| Ok("ok2"); // similar to: let fn_ok = |_| -> Result<&str, &str> { Ok("ok2") };

    let e1: Result<&str, &str> = Err("error1");
    let e2: Result<&str, &str> = Err("error2");
    let fn_err = |_| Err("error2");

    assert_eq!(o1.and_then(fn_ok), o2);  // Ok1 and_then Ok2 = Ok2
    assert_eq!(o1.and_then(fn_err), e2); // Ok and_then Err = Err
    assert_eq!(e1.and_then(fn_ok), e1);  // Err and_then Ok = Err
    assert_eq!(e1.and_then(fn_err), e1); // Err1 and_then Err2 = Err1
}
```

## filter()

>💡通常，在编程语言中，filter函数与数组或迭代器配合使用，通过在函数/闭包中过滤自身的元素来创建新的数组/迭代器。 Rust也提供了filter()作为迭代器的适配器，以便在迭代器的每个元素上应用闭包，以将其转换为另一个迭代器。但是，在这里我们讨论的是Option类型的filter()的函数。

当我们传递`Some`值并且给定的闭包基于该值返回true时，才会返回相同的`Some`类型。如果传递`None`类型或闭包返回false，则返回`None`。闭包使用`Some`中的值作为参数。 而且，Rust仅支持`Option`类型的`filter()`。

```rust
fn main() {
    let s1 = Some(3);
    let s2 = Some(6);
    let n = None;

    let fn_is_even = |x: &i8| x % 2 == 0;

    assert_eq!(s1.filter(fn_is_even), n);  // Some(3) -> 3 is not even -> None
    assert_eq!(s2.filter(fn_is_even), s2); // Some(6) -> 6 is even -> Some(6)
    assert_eq!(n.filter(fn_is_even), n);   // None -> no value -> None
}
```

## map() and map_err()

> 💡 通常，在编程语言中，map()函数与数组或迭代器配合使用，以对数组或迭代器的每个元素应用闭包。 Rust也提供了map()作为迭代器的适配器，以便在迭代器的每个元素上应用闭包，以将其转换为另一个迭代器。但是，在这里我们讨论的是Option和Result类型的map()的函数。

- `map()`：通过应用闭包转换类型T。 `Some`或`Ok`块的数据类型可以根据闭包的返回类型进行更改。将`Option<T>`转换为`Option<U>`，`Result<T, E>`转换为`Result <U, E>`

⭐ 通过`map()`，只有`Some`和`Ok`的值被改变。不会影响`Err`内部的值（`None`根本不包含任何值）。

```rust
fn main() {
    let s1 = Some("abcde");
    let s2 = Some(5);

    let n1: Option<&str> = None;
    let n2: Option<usize> = None;

    let o1: Result<&str, &str> = Ok("abcde");
    let o2: Result<usize, &str> = Ok(5);

    let e1: Result<&str, &str> = Err("abcde");
    let e2: Result<usize, &str> = Err("abcde");

    let fn_character_count = |s: &str| s.chars().count();

    assert_eq!(s1.map(fn_character_count), s2); // Some1 map = Some2
    assert_eq!(n1.map(fn_character_count), n2); // None1 map = None2

    assert_eq!(o1.map(fn_character_count), o2); // Ok1 map = Ok2
    assert_eq!(e1.map(fn_character_count), e2); // Err1 map = Err2
}
```

- `Result`类型的`map_err()`：可以根据闭包的返回类型来更改`Err`块的数据类型。将`Result <T, E>`转换为`Result <T, F>`。

⭐ 通过`map_err()`，只有`Err`值被改变。不会影响`Ok`内部的值。

```rust
fn main() {
    let o1: Result<&str, &str> = Ok("abcde");
    let o2: Result<&str, isize> = Ok("abcde");

    let e1: Result<&str, &str> = Err("404");
    let e2: Result<&str, isize> = Err(404);

    let fn_character_count = |s: &str| -> isize { s.parse().unwrap() }; // convert str to isize

    assert_eq!(o1.map_err(fn_character_count), o2); // Ok1 map = Ok2
    assert_eq!(e1.map_err(fn_character_count), e2); // Err1 map = Err2
}
```

## map_or() and map_or_else()

希望您还记得`unwrap_or()`和`unwrap_or_else()`函数的功能。这些函数有相似之处。但是`map_or()`和`map_or_else()`对`Some`和`Ok`值应用闭包，并返回类型T中的值。

- `map_or()`：仅支持`Option`类型（不支持`Result`）。将闭包应用于`Some`中的值，然后根据闭包返回输出。对于`None`将返回给定的默认值。

```rust
fn main() {
    const V_DEFAULT: i8 = 1;

    let s = Some(10);
    let n: Option<i8> = None;
    let fn_closure = |v: i8| v + 2;

    assert_eq!(s.map_or(V_DEFAULT, fn_closure), 12);
    assert_eq!(n.map_or(V_DEFAULT, fn_closure), V_DEFAULT);
}
```

- `map_or_else()`：支持`Option`类型和`Results`类型（`Result`还在nightly）。与`map_or()`类似，但要对于第一个参数，要提供另一个闭包，而不是默认值。

⭐`None`不包含任何值。因此，对于`Option`类型，无需输入参数传递给闭包。但是`Err`类型中包含值。因此，在使用过程中，对于`Result`类型，默认闭包应该能读取输入。

```rust
#![feature(result_map_or_else)] // enable unstable library feature 'result_map_or_else' on nightly
fn main() {
    let s = Some(10);
    let n: Option<i8> = None;

    let fn_closure = |v: i8| v + 2;
    let fn_default = || 1; // None doesn't contain any value. So no need to pass anything to closure as input.

    assert_eq!(s.map_or_else(fn_default, fn_closure), 12);
    assert_eq!(n.map_or_else(fn_default, fn_closure), 1);

    let o = Ok(10);
    let e = Err(5);
    let fn_default_for_result = |v: i8| v + 1; // Err contain some value inside it. So default closure should able to read it as input

    assert_eq!(o.map_or_else(fn_default_for_result, fn_closure), 12);
    assert_eq!(e.map_or_else(fn_default_for_result, fn_closure), 6);
}
```

## ok_or() and ok_or_else()

如前所述，`ok_or()`，`ok_or_else()`将`Option`类型转换为`Result`类型。`Some`转成`Ok`，`None`转成`Err`。

- `ok_or()` ：默认的`Err`消息应作为参数传入。

```rust
fn main() {
    const ERR_DEFAULT: &str = "error message";

    let s = Some("abcde");
    let n: Option<&str> = None;

    let o: Result<&str, &str> = Ok("abcde");
    let e: Result<&str, &str> = Err(ERR_DEFAULT);

    assert_eq!(s.ok_or(ERR_DEFAULT), o); // Some(T) -> Ok(T)
    assert_eq!(n.ok_or(ERR_DEFAULT), e); // None -> Err(default)
}
```

- `ok_or_else()` ：类似于`ok_or()`。应将闭包作为参数传入。

```rust
fn main() {
    let s = Some("abcde");
    let n: Option<&str> = None;
    let fn_err_message = || "error message";

    let o: Result<&str, &str> = Ok("abcde");
    let e: Result<&str, &str> = Err("error message");

    assert_eq!(s.ok_or_else(fn_err_message), o); // Some(T) -> Ok(T)
    assert_eq!(n.ok_or_else(fn_err_message), e); // None -> Err(default)
}
```

## as_ref() and as_mut()

🔎 如前所述，这些函数用于**借用类型T作为引用或可变引用**。

- `as_ref()` ：从 `Option<T>` 到 `Option<&T>` ，从`Result<T, E>` 到`Result<&T, &E>`
- `as_mut()` ：从 `Option<T>` 到 `Option<&mut T>` ，从 `Result<T, E>` 到 `Result<&mut T, &mut E>`
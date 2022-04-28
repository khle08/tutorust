# 概述

`trait` 可以用来告诉 `Rust` 编译器 某种类型具有哪些可以与其他类型共享的功能，是一种抽象的定义共享行为的做法，还可以通过范型参数设定，指定为实现了特定行为的类型，有点类似上节描述的范型在方法中的定义。`trait` 与其他语言中的 `interface` 有点类似，但还有些许区别。



# `Trait` 定义

它可以把方法签名放在一起，来定义实现某种目的所必需的一组行为。

+ 使用关键词：`trait`
+ 只有方法签名，没有具体实现（也可以有，那就是默认功能）
+ `trait` 可以有多个方法，每个方法签名占一行，以 `;` 结尾
+ 实现 `trait` 的类型必须提供具体的方法实现

```rust
pub trait Summary {
  fn sum(&self) -> String;
  fn add(&self) -> String;
  // ... more trait
}
```

如果 `trait` 在方法中实现，则需要定义清楚要做哪些事情，并且用 `<trait_name> for` 来区别一般方法：

```rust
pub trait Summary {
  fn sumarize(&self) -> String;
}

pub struct NewsArticle {
  pub headline: String,
  pub location: String,
  pub author: String,
  pub content: String,
}

impl Summary for NewsArticle {
  fn summarize(&self) -> String {
    // 定义方法具体要实现什么内容
    format!("{} by {} ({})", self.headline, self.author, self.location)
  }
}
```

同一个 `trait` 可以在不同方法中定义出不同的功能：

```rust
pub struct Tweet {
  pub username: String,
  pub content: String,
  pub reply: bool,
  pub retweet: bool,
}

impl Summary for Tweet {
  fn summarize(&self) -> String {
    format!("{}: {}", self.username, self.content)
  }
}
```

并且，需要注意的是，在调用方法下的 `trait` 的时候，两者必须都在同一个作用域下才可以

```toml
[package]
name = "demo"
...
```

```rust
// main.rs

use demo::{Tweet, Summary};  // 注意作用域

let tweet = Tweet {
  username: String::from("myName"),
  content: String::from("Let me introduce myself"),
  reply: false,
  retweet: false,
};

println!("new tweet: {}", tweet.summarize());
```



# `Trait` 的约束

某个类型之所以能够实现某个 `trait` 的前提条件如下：

> 这个类型 or `trait` 是在本地 crate 定义

如果是外部类型实现外部的 `trait` 则无法实现，确保程序属性的一致性，避免 `孤儿原则`（不存在父类型。另一个好处是，这个规则确保了其他人的代码不能破坏我们写的代码模块，反之亦然。



## 默认功能

上面的例子在 `trait` 里面只有定义方法名，但没有具体实现内容，如果加了内容进去的话，则表示一个默认的实现，可以不必在所有套用 `trait` 的类型中重新定义各自的内容。

```rust
pub trait Summary {
  // fn sumarize(&self) -> String;
  fn sumarize(&self) -> String {
    String::from("(reading more ...)")
  }
}

pub struct NewsArticle {
  pub headline: String,
  pub location: String,
  pub author: String,
  pub content: String,
}

impl Summary for NewsArticle {
  // 空的表示直接使用默认方法
}
```

在 `trait` 里面，有默认实现和没有默认实现的函数可以放在一起。



# `Trait` 参数化

现在有一个情况，一个新的函数 `notify` 的输入希望它既能是 `NewsArticle` 也能是 `Tweet`，而这几个 struct 都有同样的 `trait` 可以使用同一个方法。

```rust
pub trait Summary { ... }

pub struct NewsArticle { ... }
impl Summary for NewsArticle { ... }

pub struct Tweet { ... }
impl Summary for Tweet { ... }

pub fn notify(item: impl Summary) {         // 重点
  println!("breaking news! {}", item.summarize());
}
```

也就是说我们要求函数里的 item 参数类型同样实现了 `Summary` trait，这样才可以调用方法，如果其他函数也实现了同样的 `trait` 那也可以作为 item 参数传入，不过这种情况比较简单，trait bound 方法是一种进阶的做法，可以这么改写：

```rust
pub fn notify<T: Summary>(item: T) {
  println!("breaking news! {}", item.summarize());
}
```

如果有多个 `trait` 需要指定使用，就用 `+` 即可：

```rust
use std::fmt::Display

pub fn notify(item: impl Summary + Display) { ... }   // 方法 1
pub fn notify<T: Summary + Display>(item: T) { ... }  // 方法 2
```

不过这种做法如果东西一多就容易变得很乱，例如多个 `trait` 多个参数，可以进一步使用 `where` 来让代码更简洁：

```rust
pub fn notify<T: Summary + Display, U: Clone + Debug>(a: T, b: U) -> String {
  format!("breaking news! {}", a.summarize())
}

pub fn notify<T, U>(a: T, b: U) -> String
where T: Summary + Display,
      U: Clone + Debug,
{
  format!("breaking news! {}", a.summarize())
}
```



# `Trait` 作为返回类型

可以这么设定返回值必须是一个实现了某个 `trait` 的类型：

```rust
pub fn notify(s: &str) -> impl Summary {
  NewsArticle {
    headline: String::from("this is a headline"),
    content: String::from("this is the content"),
    author: String::from("username"),
    location: String::from("china"),
  }
}
```

但是这种写法必须注意，返回的具体类型只能是 "某一个" 与该 `trait` 相关的类型，如果返回的东西有可能不止一个，即便所有可能都是带了 `trait` 的类型，还是会报错，如下：

```rust
pub fn notify(flag: bool) -> impl Summary {
  if flag {
    NewsArticle { ... }
  } else {
    Tweet { ... }
  }
}
```

这是由于 `impl trait` 在工作上的限制，导致产生的错误，应尽量避免。



# `Trait` Bound 有条件的实现方法

使用范型参数的 `impl` 块上加上 `trait` bound，可以有条件地，为实现了特定 `trait` 的类型来实现方法，例如：

```rust
use std::fmt::Display;

struct Pair<T> {
  x: T,
  y: T,
}

impl<T> Pair <T> {
  fn new_method(x: T, y: T) -> Self {
    Self{x, y}
  }
}

impl<T: Display + PartialOrd> Pair <T> {
  fn cmp_display(&self) {
    if self.x > self.y {
      println!("x:{} is larger than y", self.x);
    } else {
      println!("y: {} is larger than x", self.y);
    }
  }
}
```

上面的例子表明了，不论 `Pair` 结构体传入了什么类型的数据，都会有一个方法 `new_method` 可以使用，不过只有传入的参数实现了 `Display + PartialOrd` 两个 `trait` 的时候，`cmp_display` 方法才能使用。

---

`Rust` 可以更进一步地为实现了其他 `trait` 的任意类型有条件的实现某个 `trait`，为满足 `trait` bound 的所有类型上实现 `trait` 叫做覆盖实现（blanket implementations）。

```rust
impl<T: fmt::Display> ToString for T {
  // 对所有满足 `Display trait` 这个约束的类型 `T` 都实现了 "ToString" 这个 trait
}
```


# 概述

代码组织主要包含并控制代码细节细节的公开与私有属性，还有作用域内哪些名称有效，模块系统包含以下几个部分：

+ Package (包)：Cargo 特性，让我们构建，测试，共享 crate
+ Crate (单元包)：一个模块树，它可以产生一个 library 或可执行文件
+ Module (模块)：通过 `use` 控制代码的组织，作用域，私有路径等设定
+ Path (路径)：为 `struct`, `function`, `module` 等命名的方式

这几部分的层级是由上而下的排列



# Package 和 Crate

一个 Package 包含了一个 `Cargo.toml` 配置文件，用来描述如何构建这些包含的 Crates，而 Crate 的类型分成 `binary` 与 `library` 两种形式，Crate Root 是源代码文件，rust 编译器从这里开始组成 Crate 的根 Module。一个 Package 相当于一个项目的概念，拥有以下特点：

1. 只能包含不超过 1 个 library crate
2. 可以有任意数量的 binary crate
3. 必须至少包含一个 crate (是 `library` 或 `binary` 的话不限)

创建新的 Package 的方法通过下面命令在终端执行

```bash
cargo new package-name
```

文件路径包含了以下几个部分，依照惯例有以下特性

```
package-name
│   Cargo.toml
│   Cargo.lock
│
└───src
│   │   main.rs （    default -  binary crate 的 crate root ）
│   │   lib.rs  （ additional - library crate 的 crate root ）
```

Cargo 把 crate root 文件交给 `rustc` 来构建 library 或 binary，如果要在一个 Package 中有多个 binary crate，可以在 `src` 文件夹同级的目录下再创建一个 `bin` 文件夹。

```
└───bin
│   │   first.rs
│   │   second.rs
```

Crate 的好处是可以把相关功能组合到同一个作用域下，便于项目共享，也避免冲突，例如 `rand` crate 访问他的功能需要通过名字：`rand` 来完成。



# 定义 Module

Module 的作用在于将代码在一个 crate 内分组，增加可读性，易于重复使用，并且可以控制项目的私有性，`pub` 与默认 private。使用 `mod` 关键字建立一个 module，类似建立函数 `fn` 的过程，并且，`mod` 是可以嵌套的，不限嵌套的次数

```rust
mod house {
  mod room {
    fn sleep() {}
    fn study() {}
  }
}
```

`main.rs` 与  `lib.rs` 这两个文件不论哪一个的内容形成了一个 `crate` 的模块，位于整个模块树的根部

```
crate
└───house
│   └───room
│   │   |   sleep
|   |   |   study
```



# 路径 Path

为了在 `rust`  的模块中找到某个条目，路径是一个重要的途径，主要分为两种，用 `::` 分隔

+ 绝对路径：从 crate root 开始，使用 `crate` 名，或字面值 `crate`
+ 相对路径：从当前模块开始，使用 `self`, `super` 或当前模块的标识符

```rust
// package-name
// └───src
// │   │   lib.rs  此文件下的代码

mod house {
  mod room {
    fn sleep() {}
  }
}

pub fn walk_in() {
  crate::house::room::sleep();  // 绝对路径
  house::room::sleep();         // 相对路径
}
```

由于定义和调用的位置属于同级，相对路径可以直接调用，但其实这个代码没考虑到私有性的问题，因此是会报错的。



# 私有边界 Privacy Boundary

模块的好处不止有组织代码，还可以定义私有边界，如果想把 `函数` 或`结构体` 等物件设定成私有的，可以将其放入某个模块中，一般默认没有标注的模块全是私有的模块，因此外部代码都没有办法调用，也没办法了解内部结构。

+ 父级模块没办法访问子模块的私有条目
+ 子模块里可以使用所有祖先模块的条目

因此上面错误的代码需要添加 `pub` 解决 bug 问题。

```rust
// package-name
// └───src
// │   │   lib.rs  此文件下的代码

mod house {
  pub mod room {
    pub fn sleep() {}
  }
}

pub fn walk_in() {
  crate::house::room::sleep();  // 绝对路径
  house::room::sleep();         // 相对路径
}
```

由于 `house` 和 `walk_in` 函数都是同一级的函数，因此没有公私区分，即便没有 `pub` 标注也不会报错。



## 上级调用 super

函数如果在不同的模块中被调用，下一级或者平级的函数可以用上面的方法，不过如果是上一级的函数调用就需要 `super` 协助。

```rust
// package-name
// └───src
// │   │   lib.rs  此文件下的代码

fn build() {}

mod house {
  pub fn room() {
    super::build();  // 上级路径
    crate::build();  // 绝对路径
  }
}
```



# 结构体公有化

同样是通过 `pub` 放在 `struct` 前面来实现，但是要注意的是，`struct` 里面的数据默认全部都是私有的，如果里面的字段想要公有化，需要单独设置 `pub` 来实现。

```rust
// package-name
// └───src
// │   │   lib.rs  此文件下的代码

mod festival {
  pub struct Meal {
    pub breakfast: String,  // 单独设置
    lunch: String,
    dinner: String,
  }
  
  impl Meal {
    pub fn moon(m1: &str, m2: &str) -> Meal {
      Meal {
        breakfast: String::from(m1);
        lunch: String::from(m2);
        dinner: String::from("soup");
      }
    }
  }
}

pub fn eat() {
  let mut meal = festival::Meal:moon("sandwich", "hamberger", "soup");
  meal.breakfast = String::from("wheat");
  println!("breakfast: {:?}", meal.breakfast);
  meal.lunch = String::from("beaf");  // Wrong
}
```



# 枚举的公有化

整体而言枚举的公有化做法和结构体一摸一样，唯一的区别在于枚举一旦公有化之后，内部定义的字段也都全是公有的，不像结构体还得一个一个设定部分字段的公有属性。



# Use 关键字

可以使用 `use` 关键字将路径导入到作用域内，并且，这个部分仍然遵守私有性原则，只有公共的部分才可以被引用，导入进来后作用域内的任意地方都可以使用这个字段

```rust
// package-name
// └───src
// │   │   lib.rs  此文件下的代码

mod house {
  pub mod room {
    pub fn sleep() {}
    fn study() {}  // 私有函数
  }
}

// 惯用做法是引入我们需要使用函数的上一级模块即可
use crate::house::room;

pub fn walk_in() {
  room::sleep();  // 作用域内使用字段
  room::study();  // Wrong
}
```

常用的习惯规则：

1. 函数：将函数的父级模块引入作用域
2. `struct`, `enum` 指定到完整路径
3. 同名条目指定到父级

另外有一些等同的做法可以让代码写起来更简洁：

```rust
use std::io;
use std::fmt;

use std::{io, fmt};  // 一次导入两函数
use std::*;          // 导入全部的函数
```

```rust
use std::io;
use std::io::Write;

use std::io::{self, Write}
```

还可以使用 `as` 来重新命名导入函数在作用域的名称

```rust
use std::io::Result as IoResult;
```



# Use 的公有化

在上面使用 `use` 引入的模块默认是私有的，凡事引入的模块外部是无法访问的，如果要开放外部代码访问这个变量后面代表的含义或功能，同样使用 `pub` 来完成。

```rust
pub use crate::house::room;
```

这么一来模块就可以重新被导出。



# 外部 Package 使用

只需要在 `Cargo.toml` 文件里面添加需要的包名称和版本号即可，在编译的时候就会自动下载。

```bash
[package]
name = "proj_name"
version = "0.1.0"
authors = ["your-name <your-name@mail.com>"]
edition = "2018"

[dependencies]
rand = "0.5.5"
```

`rand` 就是外部会自动下载的包，在内部使用的时候通过 `use` 引入：

```rust
use rand::Rng;
```

标准库 `std` 是一个例外，不需要包含进 `Cargo.toml` 也能在代码中使用



# 模块文件的拆分

`mod` 模块定义的时候如果模块名后面是 `;`，而不是代码块，那么 `rust` 就会从模块名的文件中加载文件内容，并且模块树并不会发生变化。

```rust
// package-name
// └───src
// |   |   lib.rs
// │   │   house.rs  此文件下的代码

pub mod room {
  pub fn sleep() {}
  fn study() {}  // 私有函数
}

// package-name
// └───src
// │   │   lib.rs  此文件下的代码
// │   │   house.rs

mod house;

// 惯用做法是引入我们需要使用函数的上一级模块即可
use crate::house::room;

pub fn walk_in() {
  room::sleep();  // 作用域内使用字段
  room::study();  // Wrong
}
```

这整个结构还可以进一步被层级化成更细小的模块

```rust
// package-name
// └───src
// |   |   lib.rs
// |
// |   └───house
// |   |   |   room.rs  此文件下的代码

pub fn sleep() {}
fn study() {}  // 私有函数

// package-name
// └───src
// │   │   lib.rs  此文件下的代码
// │   │   house.rs

mod house;

// 惯用做法是引入我们需要使用函数的上一级模块即可
use crate::house::room;

pub fn walk_in() {
  room::sleep();  // 作用域内使用字段
  room::study();  // Wrong
}
```

随着模块逐渐扩大，这个技术可以很方便的把模块的内容移动到其他文件中。
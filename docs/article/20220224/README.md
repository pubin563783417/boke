<!-- README.md -->

# Rust 初次体验

2022-02-24

## rust背景

> Rust 最早是 Mozilla 雇员 Graydon Hoare 的个人项目。从 2009 年开始，得到了 Mozilla 研究院的资助。2010 年项目对外公布，2010 ～ 2011 年间实现自举。自此以后，Rust 在设计变化 -> 崩溃的边缘反复横跳（历程极其艰辛）。终于，在 2015 年 5 月 15 日发布 1.0 版。

> 在此研发过程中，Rust 建立了一个强大且活跃社区，形成了一整套完善稳定的项目贡献机制（Rust 能够飞速发展，与这一点密不可分）。Rust 现在由 [Rust 项目开发者社区](https://github.com/rust-lang/rust) 维护。

> 大家可能疑惑 Rust 为啥用了这么久才到 1.0 版本？与之相比，Go 语言 2009 年发布，却在 2012 年仅用 3 年就发布了 1.0 版本。

> 首先，因为 Rust 语言特性较为复杂，所以需要全盘考虑的问题非常多；
其次，Rust 当时的参与者太多，七嘴八舌的声音很多，众口难调，而 Rust 开发团队又非常重视社区的意见；
最后，一旦 1.0 快速发布，那么后续大部分语言特性就无法再修改，对于有完美强迫症的 Rust 开发者团队来说，某种程度上的不完美是不可接受的。
因此，Rust 语言用了足足 6 年时间，才发布了尽善尽美的 1.0 版本。

## 优势

- 高性能
Rust 速度惊人且内存利用率极高。由于没有运行时和垃圾回收，它能够胜任对性能要求特别高的服务，可以在嵌入式设备上运行，还能轻松和其他语言集成。

- 可靠性
Rust 丰富的类型系统和所有权模型保证了内存安全和线程安全，让您在编译期就能够消除各种各样的错误。

- 生产力
Rust 拥有出色的文档、友好的编译器和清晰的错误提示信息， 还集成了一流的工具 —— 包管理器和构建工具， 智能地自动补全和类型检验的多编辑器支持， 以及自动格式化代码等等。

## 入门圣经

[https://course.rs/](https://course.rs/)

## 安装 Rust （macOS）

- 官方推荐， rustup 安装

```
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

如果安装成功，将出现下面这行：

```
Rust is installed now. Great!
```

## 安装 C 语言编译器：（非必需）

Rust 对运行环境的依赖和 Go 语言很像，几乎所有环境都可以无需安装任何依赖直接运行。但是，Rust 会依赖 libc 和链接器 linker。所以如果遇到了提示链接器无法执行的错误，你需要再手动安装一个 C 语言编译器：

```
$ xcode-select --install
```

## 使用VSCode 作为IDE 

[https://code.visualstudio.com/](https://code.visualstudio.com/)

语言支持 : rust-analyer 插件

## Cargo 包管理工具

我们无需再手动安装，之前安装 Rust 的时候，就已经一并安装了。

激活命名行

```
source $HOME/.cargo/env
```

## hello world

开始第一个项目

```
~ : cd ~/Documents 

~ : cargo new world_hello
```
现在的工程目录

```
.
├── .git
├── .gitignore
├── Cargo.toml
└── src
    └── main.rs
```

> 是的，连 git 都给你创建了，不仅令人感叹，不是女儿，胜似女儿，比小棉袄还体贴。

## 运行项目

```
cargo run
``` 

输出

```
$ cargo run
   Compiling world_hello v0.1.0 (/Users/sunfei/development/rust/world_hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running `target/debug/world_hello`
Hello, world!
```

## 前景

RUST 语言对标的是 C 和 C++，但它还具备了抗衡 GO 语言的能力，在深度学习和爬虫领域也有一定的机会。RUST 被视为 [系统级] 语言，它能够开发出稳定性超强、运行速度极快的项目。能够成为系统级编程语言，是因为它无 GC 和 Runtime，它超强的稳定性来源于所有权。

花里胡哨的语言我们见得多了，它们既不安全又慢。
多年来，没有任何语言能够撼 C/C++ 的地位，RUST 的出现使得编程界有了新的氧气。
编程语言 [快] 是趋势，静态语言是未来。






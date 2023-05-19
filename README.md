# CosmWasm book 智能合约 - 简体中文｜[English](https://github.com/CosmWasm/cosmwasm)

使用 Rust 编写 CosmWasm 智能合约

## 关于

该存储库包含了《CosmWasm Book》的源代码，可以在[这里](https://cosmwasm.github.io/book/)找到。

## 构建

该书使用[mdbook](https://github.com/rust-lang/mdBook)构建。

要构建它，首先需要安装[Rust](https://www.rust-lang.org/tools/install)。

然后使用Cargo安装`mdbook`：

```bash
$ cargo install mdbook
```

并从此目录构建该书：

```bash
mdbook build
```

有关使用mdbook的更多信息，请阅读其官方[书籍](https://rust-lang.github.io/mdBook/index.html)。
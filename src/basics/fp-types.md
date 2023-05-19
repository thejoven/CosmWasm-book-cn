# 浮点类型

现在你已经准备好自己创建智能合约了。是时候讨论一下 CosmWasm 智能合约的一个重要限制 - 浮点数类型。

故事很简单：你不能在智能合约中使用浮点数类型。绝对不行。CosmWasm 虚拟机特意没有实现浮点数的 Wasm 指令，甚至包括诸如 `F32Load` 这样的基本指令。原因很简单：在区块链世界中，它们不安全。

最大的问题是，合约会编译通过，但上传到区块链时会失败，并显示错误消息，声称合约中存在浮点数操作。用于验证合约是否有效（不包含任何浮点数操作，并且具有所有必需的入口点等）的工具称为 `cosmwasm-check` [utility](https://github.com/CosmWasm/cosmwasm/tree/main/packages/check)。

这个限制有两个影响。首先，你在合约中总是必须使用定点小数算术。这并不是问题，因为 `cosmwasm-std` 提供了 [`Decimal`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Decimal.html) 和 [Decimal256](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Decimal256.html) 类型。

另一个问题比较棘手 - 你必须谨慎使用 crate。特别是 `serde` crate 中的一个坑 - `usize` 类型的反序列化使用了浮点数操作。这意味着你不能在合约中的反序列化消息中使用 `usize`（或 `isize`）类型。

另一个与 serde 不兼容的功能是无标签枚举的反序列化。解决方法是使用 [`serde-cw-value`](https://crates.io/crates/serde-cw-value) crate 创建自定义的枚举反序列化。它是 [`serde-value`](https://crates.io/crates/serde-value) crate 的分支，避免生成浮点数指令。


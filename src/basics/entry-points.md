# 入口点

典型的 Rust 应用程序从操作系统调用的 `fn main()` 函数开始。智能合约并无显著不同。当消息发送到合约时，一个被称为"入口点"的函数被调用。与只有一个 `main` 入口点的原生应用程序不同，智能合约有一些对应于不同消息类型的入口点：`instantiate`，`execute`，`query`，`sudo`，`migrate` 等等。

首先，我们将从三个基本入口点开始：

* `instantiate` 在智能合约的生命周期中只被调用一次 - 你可以将其视为合约的构造器或初始化器。
* `execute` 用于处理能够修改合约状态的消息 - 它们被用来执行一些实际的操作。
* `query` 用于处理请求从合约获取一些信息的消息；与 `execute` 不同，它们永远不会影响任何合约状态，只是像数据库查询一样被使用。

转到你的 `src/lib.rs` 文件，并从 `instantiate` 入口点开始：

```rust,noplayground
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

#[entry_point]
pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    Ok(Response::new())
}
```

事实上，`instantiate` 是智能合约有效所必需的唯一入口点。虽然这种形式不是很有用，但它是一个开始。让我们仔细看看入口点的结构。

首先，我们开始导入一些类型以便更一致地使用。然后我们定义我们的入口点。`instantiate` 接受四个参数：

* [`deps: DepsMut`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.DepsMut.html)
  是一个与外部世界通信的实用类型 - 它允许查询和更新合约状态，查询其他合约状态，并提供访问 `Api` 对象的途径，该对象有一些用于处理 CW 地址的辅助函数。
* [`env: Env`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Env.html)
  是一个代表执行消息时的区块链状态的对象 - 链高度和id，当前时间戳，和被调用的合约地址。
* [`info: MessageInfo`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.MessageInfo.html)
  包含触发执行的消息的元信息 - 发送消息的地址，和随消息发送的链原生代币。
* [`msg: Empty`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Empty.html)
  是触发执行的消息本身 - 现在，它是代表 `{}` JSON 的 `Empty` 类型，但是这个参数的类型可以是任何可反序列化的类型，我们将来会在这里传递更复杂的类型。

如果你对区块链还不熟悉，那么这些参数可能对你来说没有太多意义，但是在学习这个指南的过程中，我将逐一解释它们的用途。

注意装饰我们入口点的重要属性 [`#[entry_point]`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/attr.entry_point.html)。它的目的是将整个入口点包装成 Wasm 运行时可以理解的形式。适当的 Wasm 入口点只能使用 Wasm 规范原生支持的基本类型，Rust 结构和枚举不在此列。使用这样的入口点会相当复杂，所以 CosmWasm 的创作者提供了 `entry_point` 宏。它创建了原始的 Wasm 入口点，内部调用装饰函数，并做所有必要的魔法以从 Wasm 运行时传递的参数构建我们的高级 Rust 参数。

下一个要看的是返回类型。我用 [`StdResult<Response>`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/type.StdResult.html) 作为这个简单例子的返回类型，它是 `Result<Response, StdError>` 的别名。返回入口点类型总会是一个 [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html) 类型，有一些实现 [`ToString`](https://doc.rust-lang.org/std/string/trait.ToString.html) trait 的错误类型和一个明确定义的成功情况类型。对于大多数入口点，"Ok" 情况将是 [`Response`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Response.html) 类型，它允许将合约适配到我们的 actor 模型，我们将很快讨论这个模型。

入口点的主体尽可能简单 - 它总是成功返回一个简单的空响应。
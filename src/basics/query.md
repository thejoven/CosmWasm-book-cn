# 创建查询

我们已经创建了一个简单的合约，对空的实例化消息作出了反应。不幸的是，它并不是很有用。让我们稍微增加一些响应性。

首先，我们需要将 [`serde`](https://crates.io/crates/serde) 库添加到我们的依赖项中。它将帮助我们进行查询消息的序列化和反序列化。更新 `Cargo.toml` 文件：

```toml
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
cosmwasm-std = { version = "1.0.0-beta8", features = ["staking"] }
serde = { version = "1.0.103", default-features = false, features = ["derive"] }

[dev-dependencies]
cw-multi-test = "0.13.4"
```

现在进入你的 `src/lib.rs` 文件，添加一个新的查询入口点：

```rust,noplayground
use cosmwasm_std::{
    entry_point, to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo,
    Response, StdResult,
};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct QueryResp {
    message: String,
}

# #[entry_point]
# pub fn instantiate(
#     _deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     _msg: Empty,
# ) -> StdResult<Response> {
#     Ok(Response::new())
# }
#
#[entry_point]
pub fn query(_deps: Deps, _env: Env, _msg: Empty) -> StdResult<Binary> {
    let resp = QueryResp {
        message: "Hello World".to_owned(),
    };

    to_binary(&resp)
}
```

请注意，为了简单起见，我省略了之前创建的实例化入口点 - 为了不给你过多的代码负担，我只会显示代码中发生变化的部分。

首先，我们需要一个结构体来作为查询的返回结果。我们希望始终返回可序列化的内容。我们只需使用 `serde` 库来派生 [`Serialize`](https://docs.serde.rs/serde/trait.Serialize.html) 和 [`Deserialize`](https://docs.serde.rs/serde/trait.Deserialize.html) traits。

然后，我们需要实现查询入口点。它与 `instantiate` 的入口点非常相似。第一个显著的区别是 `deps` 参数的类型。对于 `instantiate`，它是 `DepMut` 类型，但在这里我们使用了 [`Deps`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Deps.html) 对象。这是因为查询不能改变智能合约的内部状态，它只能读取状态。这带来了一些后果 - 例如，无法为将来的查询实现缓存（因为它需要某种数据缓存来写入）。

另一个区别是缺少 `info` 参数。这是因为执行操作的入口点（

例如实例化或执行）可以根据消息元数据的不同方式执行操作 - 例如，它们可以限制谁可以执行操作（通过检查消息的发送者）。而对于查询则不同。查询仅仅是纯粹地返回一些经过转换的合约状态。它可以基于一些链上元数据计算状态（因此状态可以在一段时间后"自动"更改），但不能基于消息信息进行计算。

请注意，我们的入口点仍然使用相同的 `Empty` 类型作为 `msg` 参数 - 这意味着我们发送给合约的查询消息仍然是一个空的 JSON 对象：`{}`

最后一个发生变化的地方是返回类型。与其返回成功时的 `Response` 类型，我们返回一个任意可序列化的对象。这是因为查询不使用典型的 Actor 模型消息流 - 它们无法触发任何操作，也无法以不同于查询的方式与其他合约进行通信（这由 `deps` 参数处理）。查询始终返回纯粹的数据，应该直接呈现给查询器。

现在看一下实现。其中没有复杂的操作 - 我们创建了要返回的对象，并使用 [`to_binary`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/fn.to_binary.html) 函数将其编码为 [`Binary`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Binary.html) 类型。

## 改进消息

我们有一个查询，但查询消息存在一个问题。它总是一个空的 JSON。这非常糟糕 - 如果我们将来想要添加另一个查询，很难在查询变体之间进行合理的区分。

在实践中，我们可以通过使用非空的查询消息类型来解决这个问题。改进我们的合约：

```rust,noplayground
# use cosmwasm_std::{
#     entry_point, to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# use serde::{Deserialize, Serialize};
# 
# #[derive(Serialize, Deserialize)]
# struct QueryResp {
#     message: String,
# }
#
#[derive(Serialize, Deserialize)]
pub enum QueryMsg {
    Greet {},
}

# #[entry_point]
# pub fn instantiate(
#     _deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     _msg: Empty,
# ) -> StdResult<Response> {
#     Ok(Response::new())
# }
#
#[entry_point]
pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => {
            let resp = QueryResp {
                message: "Hello World".to_owned(),
            };

            to_binary(&resp)
        }
    }
}
```

现在我们为查询消息引入了适当的消息类型。它是一个[枚举类型](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html)，默认情况下，它将序列化为一个具有单个字段的 JSON - 字段的名称将是枚举变体的名称（在我们的例子中始终是"greet" - 目前至少是如此），而该字段的值将是分配给此枚举变体的对象。

请注意，我们的枚举没有为唯一的 `Greet` 变体分配类型。在 Rust 中，我们通常在变体名称后不添加额外的 `{}`。但是这里的花括号是有目的的 - 如果没有它们，变体将序列化为只有字符串类型 - 因此，该变体的 JSON 表示形式将为 `"greet"`，而不是 `{ "greet": {} }`。这种行为导致了消息模式的不一致性。通常，为了更好地表示 JSON，我们总是为可序列化的空枚举变体添加 `{}`。

但是，我们还可以进一步改进代码。现在，`query` 函数有两个职责。第一个是显而易见的 - 处理查询本身。这是最初的假设，现在仍然存在。但是还有一个新的事情正在发生 - 查询消息的分发。尽管只有一个变体，但 `query` 函数很容易变成一个庞大而难以阅读的 `match` 语句。为了使代码更符合[SOLID](https://en.wikipedia.org/wiki/SOLID)原则，我们将进行重构，并将处理 `greet` 消息的部分提取到一个单独的函数中。

```rust,noplayground
# use cosmwasm_std::{
#     entry_point, to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# use serde::{Deserialize, Serialize};
# 
#[derive(Serialize, Deserialize)]
pub struct GreetResp {
    message: String,
}

# #[derive(Serialize, Deserialize)]
# pub enum QueryMsg {
#     Greet {},
# }
# 
# #[entry_point]
# pub fn instantiate(
#     _deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     _msg: Empty,
# ) -> StdResult<Response> {
#     Ok(Response::new())
# }
# 
#[entry_point]
pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => to_binary(&query::greet()?),
    }
}

mod query {
    use super::*;

    pub fn greet() -> StdResult<GreetResp> {
        let resp = GreetResp {
            message: "Hello World".to_owned(),
        };

        Ok(resp)
    }
}
```

现在看起来好多了。请注意，还有几个额外的改进。我将查询响应类型 `GreetResp` 的字段设为公共字段。因为它们现在需要从不同的模块中访问。然后，将我的新函数封装在名为 `query` 的模块中。这样可以更容易地避免名称冲突 - 将来我可以在查询和执行消息中具有相同的变体，它们的处理程序将位于不同的命名空间中。

`greet` 函数返回 `StdResult` 而不是 `GreetResp`，这可能是一个值得质疑的决定，因为它永远不会返回错误。这是一个风格问题，但我更喜欢在消息处理程序中保持一致性，其中大部分处理程序可能会有失败的情况 - 例如读取状态时。

此外，你可以将 `deps` 和 `env` 参数传递给所有的查询处理程序以保持一致性。我对此不太喜欢，因为它引入了不必要的样板代码，不易读，但我也同意保持一致性的论点 - 我将这个决定留给你自己判断。

## 结构化合约

你可以看到我们的合约现在变得稍微复杂了一些。大约 50 行可能不算太多，但在一个文件中有许多不同的实体，我认为我们可以做得更好。通常情况下，我们会将它们分为三个文件。从提取所有消息到 `src/msg.rs` 开始：

```rust,noplayground
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct GreetResp {
    pub message: String,
}

#[derive(Serialize, Deserialize)]
pub enum QueryMsg {
    Greet {},
}
```

你可能已经注意到，我将 `GreetResp` 的字段设为公共字段。这是因为它们现在需要从不同的模块中访问。现在继续前进到 `src/contract.rs` 文件：

```rust,noplayground
use crate::msg::{GreetResp, QueryMsg};
use cosmwasm_std::{
    to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    Ok(Response::new())
}

pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => to_binary(&query::greet()?),
    }
}

mod query {
    use super::*;

    pub fn greet() -> StdResult<GreetResp> {
        let resp = GreetResp {
            message: "Hello World".to_owned(),
        };

        Ok(resp)
    }
}
```

我将大部分逻辑都移到了这里，所以我的 `src/lib.rs` 只是一个非常薄的库入口，除了模块定义和入口点定义之外没有其他内容。我从 `src/contract.rs` 的 `query` 函数中移除了 `#[entry_point]` 属性。我将函数的处理部分拆分出来，现在 `contract::query` 函数是负责分发查询消息的顶层查询处理程序。顶层的 `query` 函数只是一个入口点。这是一个微妙的区别，但在将来当我们不想生成入口点而保留分发函数时，它将变得有意义。我现在引入了这个拆分，以展示典型的合约结构。

现在是最后一部分，`src/lib.rs` 文件：

```rust,noplayground
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

mod contract;
mod msg;

#[entry_point]
pub fn instantiate(deps: DepsMut, env: Env, info: MessageInfo, msg: Empty)
  -> StdResult<Response>
{
    contract::instantiate(deps, env, info, msg)
}

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: msg::QueryMsg)
  -> StdResult<Binary>
{
    contract::query(deps, env, msg)
}
```

非常简单的顶层模块。定义子模块和入口点，没有其他内容。

现在，当我们准备好测试合约时，让我们继续。
# 良好的实践

所有相关的基础知识都已经介绍完了。现在让我们讨论一些良好的实践。

## JSON 命名重命名

由于 Rust 的风格，我们所有的消息变体都使用 [驼峰命名法](https://en.wikipedia.org/wiki/CamelCase)。这是标准的做法，但它有一个缺点 - 所有的消息都通过 serde 进行序列化和反序列化，使用这些变体名称。问题是，在 JSON 世界中，更常见的是使用 [蛇形命名法](https://en.wikipedia.org/wiki/Snake_case) 来命名字段。幸运的是，有一种简单的方法告诉 serde，在序列化时更改字段的命名方式。让我们使用 `#[serde]` 属性更新我们的消息：

```rust,noplayground
use cosmwasm_std::Addr;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
#[serde(rename_all = "snake_case")]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
    pub donation_denom: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
#[serde(rename_all = "snake_case")]
pub enum ExecuteMsg {
    AddMembers { admins: Vec<String> },
    Leave {},
    Donate {},
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
#[serde(rename_all = "snake_case")]
pub struct GreetResp {
    pub message: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
#[serde(rename_all = "snake_case")]
pub struct AdminsListResp {
    pub admins: Vec<Addr>,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    Greet {},
    AdminsList {},
}
```

## JSON schema

谈到 JSON API，值得提到 JSON Schema。它是一种定义 JSON 消息结构的方式。
为合约 API 提供生成模式的方法是良好的实践。问题在于手动编写 JSON 模式很麻烦。好消息是有一个 crate 可以帮助我们。打开 `Cargo.toml` 文件：

```toml
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
cosmwasm-std = { version = "1.1.4", features = ["staking"] }
serde = { version = "1.0.103", default-features = false, features = ["derive"] }
cw-storage-plus = "0.15.1"
thiserror = "1"
schemars = "0.8.1"
cosmwasm-schema = "1.1.4"

[dev-dependencies]
cw-multi-test = "0.13.4"
```

在这个文件中还有一个额外的更改 - 在 `crate-type` 中我添加了 "rlib"。"cdylib" crate 不能像典型的 Rust 依赖一样使用。因此，无法为这样的 crate 创建示例。

现在回到 `src/msg.rs` 并为所有消息添加一个新的派生：

```rust,noplayground
# use cosmwasm_std::Addr;
use schemars::JsonSchema;
# use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
    pub donation_denom: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum ExecuteMsg {
    AddMembers { admins: Vec<String> },
    Leave {},
    Donate {},
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub struct GreetResp {
    pub message: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub struct AdminsListResp {
    pub admins: Vec<Addr>,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    Greet {},
    AdminsList {},
}
```

你可能会认为所有这些派生看起来有些笨拙，我同意这个观点。
幸运的是，[`cosmwasm-schema`](https://docs.rs/cosmwasm-schema/1.1.4/cosmwasm_schema/#) crate 提供了一个实用的 `cw_serde` 宏，我们可以使用它来减少样板代码：

```rust,noplayground
# use cosmwasm_std::Addr;
use cosmwasm_schema::cw_serde

#[cw_serde]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
    pub donation_denom: String,
}

#[cw_serde]
pub enum ExecuteMsg {
    AddMembers { admins: Vec<String> },
    Leave {},
    Donate {},
}

#[cw_serde]
pub struct GreetResp {
    pub message: String,
}

#[cw_serde]
pub struct AdminsListResp {
    pub admins: Vec<Addr>,
}

#[cw_serde]
pub enum QueryMsg {
    Greet {},
    AdminsList {},
}
```

另外，我们还必须为我们的查询消息派生额外的 `QueryResponses` trait，以将消息变体与我们为其生成的响应相关联：

```rust,noplayground
# use cosmwasm_std::Addr;
use cosmwasm_schema::{cw_serde, QueryResponses}

# #[cw_serde]
# pub struct InstantiateMsg {
#     pub admins: Vec<String>,
#     pub donation_denom: String,
# }
# 
# #[cw_serde]
# pub enum ExecuteMsg {
#     AddMembers { admins: Vec<String> },
#     Leave {},
#     Donate {},
# }
# 
# #[cw_serde]
# pub struct GreetResp {
#     pub message: String,
# }
# 
# #[cw_serde]
# pub struct AdminsListResp {
#     pub admins: Vec<Addr>,
# }
# 
#[cw_serde]
#[derive(QueryResponses)]
pub enum QueryMsg {
    #[returns(GreetResp)]
    Greet {},
    #[returns(AdminsListResp)]
    AdminsList {},
}
```

`QueryResponses` 是一个 trait，要求为所有查询变体添加 `#[returns(...)]` 属性，以生成关于查询-响应关系的额外信息。

现在，我们希望将 `msg` 模块公开并允许依赖于我们合约的 crate 访问（在本例中用于模式示例）。请更新 `src/lib.rs`：

```rust,noplayground
# use cosmwasm_std::{entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult};
# use error::ContractError;
# use msg::{ExecuteMsg, InstantiateMsg, QueryMsg};
# 
pub mod contract;
pub mod error;
pub mod msg;
pub mod state;
# 
# #[entry_point]
# pub fn instantiate(
#     deps: DepsMut,
#     env: Env,
#     info: MessageInfo,
#     msg: InstantiateMsg,
# ) -> StdResult<Response> {
#     contract::instantiate(deps, env, info, msg)
# }
# 
# #[entry_point]
# pub fn execute(
#     deps: DepsMut,
#     env: Env,
#     info: MessageInfo,
#     msg: ExecuteMsg,
# ) -> Result<Response, ContractError> {
#     contract::execute(deps, env, info, msg)
# }
# 
# #[entry_point]
# pub fn query(deps: Deps, env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     contract::query(deps, env, msg)
# }
```

我更改了所有模块的可见性 - 因为我们的 crate 现在可以作为依赖项使用。
如果有人想这样做，他可能需要访问处理程序或状态。

下一步是创建一个生成实际模式的工具。我们将通过在我们的 crate 中创建一个二进制文件来实现这个目标。创建一个新的 `bin/schema.rs` 文件：

```rust,noplayground
use contract::msg::{ExecuteMsg, InstantiateMsg, QueryMsg};
use cosmwasm_schema::write_api;

fn main() {
    write_api! {
        instantiate: InstantiateMsg,
        execute: ExecuteMsg,
        query: QueryMsg
    }
}
```

Cargo足够智能，能够识别`src/bin`目录中的文件作为该crate的实用程序二进制文件。现在我们可以生成我们的模式了：

```
$ cargo run schema
    Finished dev [unoptimized + debuginfo] target(s) in 0.52s
     Running `target/debug/schema schema`
Removing "/home/hashed/confio/git/book/examples/03-basics/schema/contract.json" …
Exported the full API as /home/hashed/confio/git/book/examples/03-basics/schema/contract.json
```

我鼓励你打开生成的文件，看看模式是什么样子的。

问题是，不幸的是，创建这个二进制文件会导致我们的项目在Wasm目标上无法编译 - 最终，这是最重要的一个目标。幸运的是，我们不需要为Wasm目标构建模式二进制文件 - 让我们调整`.cargo/config`文件：

```toml
[alias]
wasm = "build --target wasm32-unknown-unknown --release --lib"
wasm-debug = "build --target wasm32-unknown-unknown --lib"
schema = "run schema"
```

通过向`wasm`的Cargo别名添加`--lib`标志，工具链将只构建库目标 - 跳过构建任何二进制文件。此外，我添加了方便的`schema`别名，这样只需简单地调用`cargo schema`就可以生成模式。

## 禁用库的入口点

由于我们为合约添加了 "rlib" 目标，如前所述，它可以作为依赖项使用。问题是，依赖于我们的合约会在依赖项和最终合约中生成两次Wasm入口点。为了解决这个问题，我们可以通过在作为依赖项使用时禁用生成Wasm入口点来解决。我们可以使用[feature标志](https://doc.rust-lang.org/cargo/reference/features.html)来实现这一点。

首先更新 `Cargo.toml` 文件：

```toml
[features]
library = []
```

通过这种方式，我们为我们的 crate 创建了一个新的 feature。现在我们想要在入口点上禁用 `entry_point` 属性 - 我们将通过对 `src/lib.rs` 进行轻微的更新来实现：

```rust,noplayground
# use cosmwasm_std::{entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult};
# use error::ContractError;
# use msg::{ExecuteMsg, InstantiateMsg, QueryMsg};
# 
# pub mod contract;
# pub mod error;
# pub mod msg;
# pub mod state;
# 
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    contract::instantiate(deps, env, info, msg)
}

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    contract::execute(deps, env, info, msg)
}

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, env: Env, msg: QueryMsg) -> StdResult<Binary> {
    contract::query(deps, env, msg)
}
```

[`cfg_attr`](https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg_attr-attribute) 属性是一个条件编译属性，类似于我们之前用于测试的 `cfg`。如果条件为 true，则它会扩展为给定的属性。在我们的情况下 - 如果启用了 "library" 特性，它将扩展为空，否则就只会扩展为 `#[entry_point]`。

由于现在将此合约作为依赖项添加，不要忘记像这样启用特性：

```toml
[dependencies]
my_contract = { version = "0.1", features = ["library"] }
```

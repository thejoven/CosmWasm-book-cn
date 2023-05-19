# 合同状态

我们正在进行的合同已经有一些能够回答查询的行为。不幸的是，它的回答非常可预测，而且没有任何东西可以改变它们。在这一章，我引入了状态的概念，这将使我们能够真正地给智能合同带来生命。

目前的状态仍然是静态的 - 它将在合同实例化时初始化。状态将包含一份管理员列表，这些管理员在未来有资格执行消息。

首先要做的是更新`Cargo.toml`，增加另一个依赖项 - [`storage-plus`](https://crates.io/crates/cw-storage-plus) crate，它为CosmWasm智能合同状态管理提供了高级绑定：

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
cw-storage-plus = "0.13.4"

[dev-dependencies]
cw-multi-test = "0.13.4"
```

现在创建一个新文件，用来保存合同的状态 - 我们通常称之为`src/state.rs`：

```rust,noplayground
use cosmwasm_std::Addr;
use cw_storage_plus::Item;

pub const ADMINS: Item<Vec<Addr>> = Item::new("admins");
```

并确保在`src/lib.rs`中声明模块：

```rust,noplayground
# use cosmwasm_std::{
#     entry_point, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# mod contract;
# mod msg;
mod state;
#
# #[entry_point]
# pub fn instantiate(deps: DepsMut, env: Env, info: MessageInfo, msg: Empty)
#   -> StdResult<Response>
# {
#     contract::instantiate(deps, env, info, msg)
# }
#
# #[entry_point]
# pub fn query(deps: Deps, env: Env, msg: msg::QueryMsg)
#   -> StdResult<Binary>
# {
#     contract::query(deps, env, msg)
# }
```

我们在这里的新东西是`ADMINS`常量，类型是`Item<Vec<Addr>>`。你可以在这里问一个很好的问题 - 状态是如何保持恒定的？如果它是一个常数值，我如何修改它？

答案有点复杂 - 这个常数并不是保持状态本身的。状态是存储在区块链中的，通过传递给入口点的`deps`参数进行访问。storage-plus常量只是访问器工具，帮助我们以结构化的方式访问这个状态。

在CosmWasm中，区块链状态只是一个巨大的键值存储。键以元信息为前缀，指向

拥有它们的合同（所以没有其他合同可以以任何方式改变它们），但即使去掉前缀，单个合同状态也是一个较小的键值对。

`storage-plus`通过智能地为项目键额外添加前缀来处理更复杂的状态结构。现在，我们只是使用了最简单的存储实体 - 一个[`Item<_>`](https://docs.rs/cw-storage-plus/0.13.4/cw_storage_plus/struct.Item.html)，它保存了一个给定类型的单个可选值 - 在这种情况下是`Vec<Addr>`。那么在存储中这个项目的键是什么呢？这对我们来说无关紧要 - 它会被发现是唯一的，基于传递给[`new`](https://docs.rs/cw-storage-plus/0.13.4/cw_storage_plus/struct.Item.html#method.new)函数的唯一字符串。

在我们开始初始化我们的状态之前，我们需要一些更好的实例化消息。转到`src/msg.rs`并创建一个：

```rust,noplayground
# use cosmwasm_std::Addr;
# use serde::{Deserialize, Serialize};
# 
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
}
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub struct GreetResp {
#     pub message: String,
# }
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub enum QueryMsg {
#     Greet {},
# }
```

现在前进到`src/contract.rs`中的实例化入口点，并将我们的状态初始化为我们在实例化消息中得到的任何内容：

```rust,noplayground
# use crate::msg::{GreetResp, InstantiateMsg, QueryMsg};
use crate::state::ADMINS;
// --snip--
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    let admins: StdResult<Vec<_>> = msg
        .admins
        .into_iter()
        .map(|addr| deps.api.addr_validate(&addr))
        .collect();
    ADMINS.save(deps.storage, &admins?)?;

    Ok(Response::new())
}
# 
# pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     use QueryMsg::*;
# 
#     match msg {
#         Greet {} => to_binary(&query::greet()?),
#     }
# }
# 
# #[allow(dead_code)]
# pub fn execute(_deps: DepsMut, _env: Env, _info: MessageInfo, _msg: Empty) -> StdResult<Response> {
#     unimplemented!()
# }
# 
# mod query {
#     use super::*;
# 
#     pub fn greet() -> StdResult<GreetResp> {
#         let resp = GreetResp {
#             message: "Hello World".to_owned(),
#         };
# 
#         Ok(resp)
#     }
# }
# 
# #[cfg(test)]
# mod tests {
#     use cosmwasm_std::Addr;
#     use cw_multi_test::{App, ContractWrapper, Executor};
# 
#     use super::*;
# 
#     #[test]
#     fn greet_query() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &Empty {},
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: GreetResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::Greet {})
#             .unwrap();
# 
#         assert_eq!(
#             resp,
#             GreetResp {
#                 message: "Hello World".to_owned()
#             }
#         );
#     }
# }
```

我们还需要在`src/lib.rs`中的入口点更新消息类型：


```rust,noplayground
# use cosmwasm_std::{entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult};
use msg::InstantiateMsg;
// --snip--
# 
# mod contract;
# mod msg;
# mod state;
# 
#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    contract::instantiate(deps, env, info, msg)
}
# 
# #[entry_point]
# pub fn query(deps: Deps, env: Env, msg: msg::QueryMsg) -> StdResult<Binary> {
#     contract::query(deps, env, msg)
# }
```

瞧，这就是更新状态所需要的全部内容！

首先，我们需要将字符串的向量转换为要存储的地址的向量。我们不能把地址作为消息参数，因为并非每个字符串都是一个有效的地址。当我们在测试上工作时，这可能有点令人困惑。任何字符串都可以在地址的位置使用。让我解释一下。

从技术上讲，每个字符串都可以被认为是一个地址。然而，并非每个字符串都是实际存在的区块链地址。当我们在合同中保存任何类型的`Addr`时，我们假设它是区块链中的一个合适的地址。这就是为什么[`addr_validate`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/trait.Api.html#tymethod.addr_validate)函数存在的原因 - 用来检查这个前提条件。

有了要存储的数据，我们使用[`save`](https://docs.rs/cw-storage-plus/0.13.4/cw_storage_plus/struct.Item.html#method.save)函数将其写入合同状态。注意`save`的第一个参数是[`&mut Storage`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/trait.Storage.html)，这是实际的区块链存储。正如强调的那样，`Item`对象不存储任何东西，只是一个访问器。它确定如何在给定的存储中存储数据。第二个参数是要存储的可序列化数据。

现在是检查我们通过的回归测试的好时机 - 尝试运行我们的测试：

```
> cargo test

...

running 1 test
test contract::tests::greet_query ... FAILED

failures:

---- contract::tests::greet_query stdout ----
thread 'contract::tests::greet_query' panicked at 'called `Result::unwrap()` on an `Err` value: error executing WasmMsg:
sender: owner
Instantiate { admin: None, code_id: 1, msg: Binary(7b7d), funds: [], label: "Contract" }

Caused by:
    Error parsing into type contract::msg::InstantiateMsg: missing field `admins`', src/contract.rs:80:14
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    contract::tests::greet_query

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

哎呀，我们把什么东西弄坏了！但请保持冷静。让我们从仔细阅读错误消息开始：

> Error parsing into type contract::msg::InstantiateMsg: missing field `admins`', src/contract.rs:80:14

问题是，在测试中，我们在测试中发送了一个空的实例化消息，但现在，我们的端点期望有一个`admin`字段。多测试框架从入口点到结果测试合同，所以使用MT函数发送消息首先会对它们进行序列化。然后合同在入口处对它们进行反序列化。但现在它试图把空的JSON反序列化为一些非空的消息！我们可以通过更新测试快速修复它：

```rust,noplayground
# use crate::msg::{GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::ADMINS;
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# pub fn instantiate(
#     deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     msg: InstantiateMsg,
# ) -> StdResult<Response> {
#     let admins: StdResult<Vec<_>> = msg
#         .admins
#         .into_iter()
#         .map(|addr| deps.api.addr_validate(&addr))
#         .collect();
#     ADMINS.save(deps.storage, &admins?)?;
# 
#     Ok(Response::new())
# }
# 
# pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     use QueryMsg::*;
# 
#     match msg {
#         Greet {} => to_binary(&query::greet()?),
#         AdminsList {} => to_binary(&query::admins_list(deps)?),
#     }
# }
# 
# #[allow(dead_code)]
# pub fn execute(_deps: DepsMut, _env: Env, _info: MessageInfo, _msg: Empty) -> StdResult<Response> {
#     unimplemented!()
# }
# 
# mod query {
#     use crate::msg::AdminsListResp;
# 
#     use super::*;
# 
#     pub fn greet() -> StdResult<GreetResp> {
#         let resp = GreetResp {
#             message: "Hello World".to_owned(),
#         };
# 
#         Ok(resp)
#     }
# 
#     pub fn admins_list(deps: Deps) -> StdResult<AdminsListResp> {
#         let admins = ADMINS.load(deps.storage)?;
#         let resp = AdminsListResp { admins };
#         Ok(resp)
#     }
# }
# 
# #[cfg(test)]
# mod tests {
#     use cosmwasm_std::Addr;
#     use cw_multi_test::{App, ContractWrapper, Executor};
# 
#     use super::*;
# 
    #[test]
    fn greet_query() {
        let mut app = App::default();

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg { admins: vec![] },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        let resp: GreetResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::Greet {})
            .unwrap();

        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }
# }
```

## 测试状态

当状态被初始化时，我们希望有一种方式来测试它。我们希望提供一个查询来检查实例化是否影响了状态。只需创建一个简单的查询来列出所有的管理员。首先在`src/msg.rs`中为查询消息和相应的响应消息添加一个变量。我们将变量命名为`AdminsList`，响应为`AdminsListResp`，并让它返回一个`cosmwasm_std::Addr`的向量：

```rust,noplayground
# use cosmwasm_std::Addr;
# use serde::{Deserialize, Serialize};
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub struct InstantiateMsg {
#     pub admins: Vec<Addr>,
# }
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub struct GreetResp {
#     pub message: String,
# }
# 
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct AdminsListResp  {
    pub admins: Vec<Addr>,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub enum QueryMsg {
    Greet {},
    AdminsList {},
}
```

并在其中实施 `src/contract.rs`:

```rust,noplayground
use crate::msg::{AdminsListResp, GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::ADMINS;
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# pub fn instantiate(
#     deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     msg: InstantiateMsg,
# ) -> StdResult<Response> {
#     let admins: StdResult<Vec<_>> = msg
#         .admins
#         .into_iter()
#         .map(|addr| deps.api.addr_validate(&addr))
#         .collect();
#     ADMINS.save(deps.storage, &admins?)?;
# 
#     Ok(Response::new())
# }
# 
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => to_binary(&query::greet()?),
        AdminsList {} => to_binary(&query::admins_list(deps)?),
    }
}
 
# #[allow(dead_code)]
# pub fn execute(_deps: DepsMut, _env: Env, _info: MessageInfo, _msg: Empty) -> StdResult<Response> {
#     unimplemented!()
# }
#
mod query {
#    use super::*;
#
#    pub fn greet() -> StdResult<GreetResp> {
#        let resp = GreetResp {
#            message: "Hello World".to_owned(),
#        };
#
#        Ok(resp)
#    }
#
    pub fn admins_list(deps: Deps) -> StdResult<AdminsListResp> {
        let admins = ADMINS.load(deps.storage)?;
        let resp = AdminsListResp { admins };
        Ok(resp)
    }
}

# #[cfg(test)]
# mod tests {
#     use cosmwasm_std::Addr;
#     use cw_multi_test::{App, ContractWrapper, Executor};
# 
#     use super::*;
# 
#     #[test]
#     fn greet_query() {
#        let mut app = App::default();
#
#        let code = ContractWrapper::new(execute, instantiate, query);
#        let code_id = app.store_code(Box::new(code));
#
#        let addr = app
#            .instantiate_contract(
#                code_id,
#                Addr::unchecked("owner"),
#                &InstantiateMsg { admins: vec![] },
#                &[],
#                "Contract",
#                None,
#            )
#            .unwrap();
#
#        let resp: GreetResp = app
#            .wrap()
#            .query_wasm_smart(addr, &QueryMsg::Greet {})
#            .unwrap();
#
#        assert_eq!(
#            resp,
#            GreetResp {
#                message: "Hello World".to_owned()
#            }
#        );
#     }
# }
```

现在我们有了测试实例的工具，让我们编写一个测试用例：

```rust,noplayground
use crate::msg::{AdminsListResp, GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::ADMINS;
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# pub fn instantiate(
#     deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     msg: InstantiateMsg,
# ) -> StdResult<Response> {
#     let admins: StdResult<Vec<_>> = msg
#         .admins
#         .into_iter()
#         .map(|addr| deps.api.addr_validate(&addr))
#         .collect();
#     ADMINS.save(deps.storage, &admins?)?;
# 
#     Ok(Response::new())
# }
# 
# pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     use QueryMsg::*;
# 
#     match msg {
#         Greet {} => to_binary(&query::greet()?),
#         AdminsList {} => to_binary(&query::admins_list(deps)?),
#     }
# }
# 
# #[allow(dead_code)]
# pub fn execute(_deps: DepsMut, _env: Env, _info: MessageInfo, _msg: Empty) -> StdResult<Response> {
#     unimplemented!()
# }
# 
# mod query {
#     use super::*;
# 
#     pub fn greet() -> StdResult<GreetResp> {
#         let resp = GreetResp {
#             message: "Hello World".to_owned(),
#         };
# 
#         Ok(resp)
#     }
# 
#     pub fn admins_list(deps: Deps) -> StdResult<AdminsListResp> {
#         let admins = ADMINS.load(deps.storage)?;
#         let resp = AdminsListResp { admins };
#         Ok(resp)
#     }
# }
# 
#[cfg(test)]
mod tests {
#     use cosmwasm_std::Addr;
#     use cw_multi_test::{App, ContractWrapper, Executor};
# 
#     use super::*;
# 
    #[test]
    fn instantiation() {
        let mut app = App::default();

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg { admins: vec![] },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(resp, AdminsListResp { admins: vec![] });

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["admin1".to_owned(), "admin2".to_owned()],
                },
                &[],
                "Contract 2",
                None,
            )
            .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(
            resp,
            AdminsListResp {
                admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
            }
        );
    }
# 
#     #[test]
#     fn greet_query() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg { admins: vec![] },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: GreetResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::Greet {})
#             .unwrap();
# 
#         assert_eq!(
#             resp,
#             GreetResp {
#                 message: "Hello World".to_owned()
#             }
#         );
#     }
}
```


测试很简单 - 用不同的初始管理员两次实例化合同，并确保每次的查询结果都是正确的。这通常是我们测试合同的方式 - 我们在合同上执行一堆消息，然后我们查询一些数据，验证查询响应是否符合预期。

我们在开发合同方面做得很好。现在是时候使用状态并允许一些执行了。
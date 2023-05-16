# 测试查询

上次我们创建了一个新的查询，现在是时候测试一下了。我们将从基础知识开始 - 单元测试。这种方法简单且不需要除了 Rust 之外的知识。转到 `src/contract.rs` 并在其模块中添加一个测试：

```rust,noplayground
# use crate::msg::{GreetResp, QueryMsg};
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# pub fn instantiate(
#     _deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     _msg: Empty,
# ) -> StdResult<Response> {
#     Ok(Response::new())
# }
# 
# pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     use QueryMsg::*;
# 
#     match msg {
#         Greet {} => to_binary(&query::greet()?),
#     }
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
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greet_query() {
        let resp = query::greet().unwrap();
        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }
}
```

如果你以前在 Rust 中编写过单元测试，那么这里不应该有什么意外。只是一个简单的仅用于测试的模块，包含本地函数单元测试。问题是 - 此测试尚未构建。我们需要稍微调整一下消息类型。更新 `src/msg.rs`：

```rust,noplayground
# use serde::{Deserialize, Serialize};
# 
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct GreetResp {
    pub message: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub enum QueryMsg {
    Greet {},
}
```

我为两种消息类型添加了三个新的衍生。[`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) 是必需的，以允许比较类型是否相等 - 这样我们就可以检查它们是否相等了。[`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html) 是一种生成调试打印工具的 trait。它被 [`assert_eq!`](https://doc.rust-lang.org/std/macro.assert_eq.html) 用于在断言失败时显示不匹配的信息。请注意，因为我们没有以任何方式测试 `QueryMsg`，额外的 trait 衍生是可选的。但是，为了测试性和一致性，将所有消息都设为 `PartialEq` 和 `Debug` 是一个好习惯。最后一个 [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html) 目前还不需要，但是 将消息克隆到其他地方也是一个好习惯。我们以后也会需要它，所以我已经添加了它，以免来回更改。

现在我们准备运行测试：

```
$ cargo test

...
running 1 test
test contract::tests::greet_query ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

耶！测试通过了！

## 合约作为黑盒

现在让我们再进一步。Rust 的测试工具是一个友好的工具，用于构建更高级别的测试。我们当前正在测试智能合约的内部，但是如果你考虑一下你的智能合约如何在外部世界中可见。它是一个由一些输入消息触发的单个实体。我们可以通过我们的 `query` 函数创建将整个合约视为黑盒的测试。让我们更新我们的测试：

```rust,noplayground
# use crate::msg::{GreetResp, QueryMsg};
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# pub fn instantiate(
#     _deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     _msg: Empty,
# ) -> StdResult<Response> {
#     Ok(Response::new())
# }
# 
# pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     use QueryMsg::*;
# 
#     match msg {
#         Greet {} => to_binary(&query::greet()?),
#     }
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
#[cfg(test)]
mod tests {
    use cosmwasm_std::from_binary;
    use cosmwasm_std::testing::{mock_dependencies, mock_env};

    use super::*;

    #[test]
    fn greet_query() {
        let resp = query(
            mock_dependencies().as_ref(),
            mock_env(),
            QueryMsg::Greet {}
        ).unwrap();
        let resp: GreetResp = from_binary(&resp).unwrap();

        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }
}
```

我们需要为 `query` 函数提供两个实体：`deps` 和 `env` 实例。幸运的是，`cosmwasm-std` 提供了用于测试这些实例的实用工具 - [`mock_dependencies`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/testing/fn.mock_dependencies.html) 和 [`mock_env`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/testing/fn.mock_env.html) 函数。

你可能会注意到 `deps` 的依赖项模拟类型是 [`OwnedDeps`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.OwnedDeps.html) 而不是我们这里需要的 `Deps`，这就是为什么在它上面调用 [`as_ref`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.OwnedDeps.html#method.as_ref) 函数的原因。如果我们寻找的是 `DepsMut` 对象，我们将使用 [`as_mut`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.OwnedDeps.html#method.as_mut)。

我们可以重新运行测试，应该仍然通过。但是当我们考虑该测试是否反映了实际用例时，它是不准确的。合约被查询了，但是它还没有被实例化！在软件工程中，这相当于在没有构造对象的情况下调用 getter - 从无处获取它。这是一种糟糕的测试方法。我们可以做得更好：

```rust,noplayground

# use crate::msg::{GreetResp, QueryMsg};
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
# 
# pub fn instantiate(
#     _deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     _msg: Empty,
# ) -> StdResult<Response> {
#     Ok(Response::new())
# }
# 
# pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
#     use QueryMsg::*;
# 
#     match msg {
#         Greet {} => to_binary(&query::greet()?),
#     }
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
#[cfg(test)]
mod tests {
    use cosmwasm_std::from_binary;
    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};

    use super::*;

    #[test]
    fn greet_query() {
        let mut deps = mock_dependencies();
        let env = mock_env();

        instantiate(
            deps.as_mut(),
            env.clone(),
            mock_info("sender", &[]),
            Empty {},
        )
        .unwrap();

        let resp = query(deps.as_ref(), env, QueryMsg::Greet {}).unwrap();
        let resp: GreetResp = from_binary(&resp).unwrap();
        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }
}
```

这里有一些新的东西。首先，我将 `deps` 和 `env` 提取到它们自己的变量中，并将它们传递给调用。这个想法是，这些变量表示一些区块链的持久状态，我们不想为每次调用都创建它们。我们希望在 `instantiate` 中发生的任何对合

约状态的更改都可以在 `query` 中看到。而且，我们希望能够控制查询和实例化中的环境差异。

`info` 参数是另一个问题。消息信息对于每个发送的消息都是唯一的。为了创建模拟的 `info`，我们必须将两个参数传递给 [`mock_info`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/testing/fn.mock_info.html) 函数。

第一个是执行调用的地址。将 `sender` 作为地址传递可能看起来有点奇怪，而不是一些神秘的以 `wasm` 开头的哈希，但这是一个有效的地址。对于测试目的，这样的地址通常更好，因为如果测试失败，它们更为冗长。

第二个参数是与消息一起发送的资金。现在，我们将其设置为空切片，因为我不想讨论令牌转移 - 我们稍后会介绍它。

所以现在它更像是一个真实的用例。我只看到一个问题。我说合约是一个单一的黑盒。但是这里没有任何东西将 `instantiate` 调用与相应的 `query` 关联起来。看起来我们假设存在某个全局的合约。但是看起来，如果我们在单个测试用例中希望有两个以不同方式实例化的合约，那将变得混乱。如果有一些工具可以为我们抽象这个，那将是很好的，不是吗？
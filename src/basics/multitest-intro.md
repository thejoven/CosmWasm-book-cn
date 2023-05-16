# 介绍 multitest

让我来介绍一下[`multitest`](https://crates.io/crates/cw-multi-test)——一个用于在 Rust 中为智能合约创建测试的库。

`multitest` 的核心思想是抽象出合约的实体，并模拟区块链环境以进行测试。其目的是测试智能合约之间的通信。它的功能非常出色，同时也是测试单一合约场景的绝佳工具。

首先，我们需要在`Cargo.toml`中添加 multitest 作为依赖。

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

我添加了一个新的[`[dev-dependencies]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#development-dependencies)部分，用于存放在最终二进制文件中不会使用到但在开发过程中可能被工具使用的依赖项，例如测试。

当我们准备好依赖项后，更新我们的测试以使用这个框架：

```rust,noplayground
#[allow(dead_code)]
pub fn execute(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty
) -> StdResult<Response> {
    unimplemented!()
}

#[cfg(test)]
mod tests {
    use cosmwasm_std::Addr;
    use cw_multi_test::{App, ContractWrapper, Executor};

    use super::*;

    #[test]
    fn greet_query() {
        let mut app = App::default();

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &Empty {},
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
}
```

你可能注意到我添加了一个 `execute` 入口函数。我没有添加入口点本身或函数的实现，但为了进行 multitest，合约必须包含至少 `instantiate`、`query` 和 `execute` 处理程序。我将该函数标记为[`#[allow(dead_code)]`](https://doc.rust-lang.org/reference/attributes/diagnostics.html#lint-check-attributes)，这样 `cargo` 就不会抱怨它没有在任何地方使用。只在测试中启用它，使用 `#[cfg(test)]` 也是一种方式。

然后在测试的开头，我创建了一个 [`App`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.App.html#) 对象。它是 multitest 中代表虚拟区块链的核心实体。正如你所看到的，我们可以像使用 `wasmd` 一样在其上调用函数与区块链进行交互！

在创建 `app` 后，我准备了一个代表要在区块链上 "上传" 的 `code` 表示。由于 multitest 只是原生的 Rust 测试，并不涉及任何 Wasm 二进制文件，但这个名称与真实场景非常匹配。我们使用 [`store_code`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.App.html#method.store_code) 函数将该对象存储在区块链上，并获得代码的 ID - 我们将需要它来实例化合约。

接下来是实例化的步骤。在单个[`instantiate_contract`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/trait.Executor.html#method.instantiate_contract)调用中，我们提供了与 `wasmd` 相同的一切 - 合约代码 ID、执行实例化的地址、触发实例化的消息以及随消息发送的任何资金（现在为空）。我们添加了合约的标签和迁移的管理员 - `None`，因为我们还不需要它。

合约上线后，我们可以对其进行查询。[`wrap`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.App.html?search=in#method.wrap)函数是用于查询 API 的访问器（查询与其他调用的处理方式略有不同），而[`query_wasm_smart`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.QuerierWrapper.html#method.query_wasm_smart)查询需要合约和消息作为参数。此外，我们不需要关心查询结果是否为 `Binary` - multitest 假设我们希望将其反序列化为某个响应类型，因此它利用了 Rust 的类型推断优势，为我们提供了良好的 API。

现在是重新运行测试的时候了。它应该仍然通过，但现在我们已经将整个测试合约进行了良好的抽象，而不仅仅是一些内部函数。下一步，我们可能应该让合约变得更有趣，通过添加一些状态。
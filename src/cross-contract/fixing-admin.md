# 修复管理员合约

现在我们知道了我们想要实现的目标，我们可以开始对已有的合约进行调整，使其成为管理员合约。目前它基本上还可以，但我们想要进行一些清理。

## 清理查询

首先要做的是删除`Greet`查询-它作为一个起始查询示例很好，但没有实际用途，只会产生噪音。

我们想要从查询枚举中删除不必要的变体：

```rust
# use cosmwasm_schema::{cw_serde, QueryResponses};
# use cosmwasm_std::Addr;
# 
# #[cw_serde]
# pub struct InstantiateMsg {
#     pub admins: Vec<String>,
#     pub donation_denom: String,
# }
# 
# #[cw_serde]
# pub enum ExecuteMsg {
#     Leave {},
#     Donate {},
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
    #[returns(AdminsListResp)]
    AdminsList {},
}
```

然后我们还会在查询调度程序中删除无效路径：

```rust
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        AdminsList {} => to_binary(&query::admins_list(deps)?),
    }
}
```
最后，我们从`contract::query`模块中移除了不相关的处理程序。我们还需要确保所有对它的引用都消失了（例如，如果在测试中有任何引用）。

## 生成库文件输出

在本书的开头，我们将`Cargo.toml`中的`crate-type`设置为`"cdylib"`。这是为了生成wasm输出所必需的，但它有一个缺点-动态库不能用作其他crate的依赖项。在此之前，这不是一个问题，但在实践中，我们经常希望将合约依赖于其他合约，以便获得对它们的某些类型的访问权限-例如，定义的消息。

对我们来说，修复这个问题很容易。您可能会注意到`crate-type`是一个数组，而不是一个单独的字符串。这是因为我们的项目可以生成多个目标-特别是，我们可以在其中添加默认的`"rlib"`crate类型，以生成“rust库”输出-这正是我们需要作为依赖项使用的。让我们更新我们的`Cargo.toml`文件：

```toml
[package]
name = "admin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]
# 
# [features]
# library = []
# 
# [dependencies]
# cosmwasm-std = { version = "1.1.4", features = ["staking"] }
# serde = { version = "1.0.103", default-features = false, features = ["derive"] }
# cw-storage-plus = "0.15.1"
# thiserror = "1"
# schemars = "0.8.1"
# cw-utils = "0.15.1"
# cosmwasm-schema = "1.1.4"
# 
# [dev-dependencies]
# cw-multi-test = "0.15.1"
```
此外，请注意我更改了合约的名称 - "contract"不太具有描述性，所以我将其更新为"admin"。

## 项目结构

最后但并非最不重要的是 - 我们希望更好地组织我们的项目结构。到目前为止，我们只有一个合约，所以我们将其作为整个项目来处理。现在我们希望有一个反映我们创建的合约之间关系的目录树。

首先，在项目中创建一个目录。然后我们想在其中创建一个名为"contracts"的子目录。从Rust的角度来看，这在技术上并不是必需的，但在我们的环境中有一些工具（例如工作区优化器），它们会假设它是应该查找合约的位置。这是您在CosmWasm合约存储库中常见的模式。

然后，我们将上一章中的整个项目目录复制到"contracts"目录中，并将其重命名为"admin"。

最后，我们希望将所有项目（目前只有一个，但我们知道以后会有更多）耦合在一起。为此，在顶层项目目录中创建工作区级别的`Cargo.toml`文件：

```toml
[workspace]
members = ["contracts/*"]
resolver = "2"
```
这个`Cargo.toml`与典型的项目级`Cargo.toml`略有不同 - 它定义了工作区。这里最重要的字段是`members` - 它定义了作为工作区一部分的项目。

另一个字段是`resolver`。记住要添加它 - 它指示Cargo使用依赖解析器的第2版。自Rust 2021以来，这已经是非工作区的默认设置，但由于兼容性原因，默认设置无法更改为工作区 - 但建议在每个新创建的工作区中添加它。

最后一个可能对工作区有用的字段是`exclude` - 它允许在工作区目录树中创建不属于该工作区的项目 - 我们不会使用它，但了解它是很好的。

现在只是为了清晰起见，让我们来看看顶层目录结构：

```none
.
├── Cargo.lock
├── Cargo.toml
├── contracts
│  └── admin
└── target
   ├── CACHEDIR.TAG
   └── debug
```

你可以看到目录树中存在目标目录和`Cargo.lock`文件 - 这是因为我已经为`admin`合约构建并运行了测试 - 在Rust工作区中，`cargo`知道要在顶层构建所有内容，即使是从内部目录构建也是如此。
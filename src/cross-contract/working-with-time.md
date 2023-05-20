# 使用时间

在区块链中，时间的概念是棘手的 - 就像在任何分布式系统中一样，很难同步所有节点的时钟。

然而，有一种时间概念是单调的 - 这意味着它在执行之间不应该倒退。此外，重要的是，时间在整个交易过程中始终是唯一的 - 甚至在由多个交易组成的整个区块中也是如此。

时间被编码在 [`Env`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.Env.html) 类型的 [`block`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.BlockInfo.html) 字段中，其结构如下所示：

```rust
pub struct BlockInfo {
    pub height: u64,
    pub time: Timestamp,
    pub chain_id: String,
}
```
您可以看到`time`字段，它是处理的区块的时间戳。还值得一提的是`height`字段，它包含了已处理区块的序列号。在某些情况下，它比时间更有用，因为可以保证`height`字段在区块之间递增，而两个区块可能在相同的时间执行（尽管这种情况相对较少）。

此外，一个区块中可能会执行许多交易。这意味着如果我们需要一个特定消息执行的唯一标识符，我们应该寻找其他方式。这个东西就是`Env`类型的[`transaction`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.TransactionInfo.html)字段：

```rust
pub struct TransactionInfo {
    pub index: u32,
}
```
在这里，`index`包含了区块中交易的唯一索引。这意味着要获得整个区块中交易的唯一标识符，我们可以使用`(height, transaction_index)`的组合。

## 加入时间

我们希望在我们的系统中使用时间来跟踪管理员的加入时间。虽然我们还没有向群组添加新成员，但我们已经可以设置初始管理员的加入时间。让我们开始更新我们的状态：

```rust
use cosmwasm_std::{Addr, Timestamp};
use cw_storage_plus::Map;
# use cw_storage_plus::Item;

pub const ADMINS: Map<&Addr, Timestamp> = Map::new("admins");
# pub const DONATION_DENOM: Item<String> = Item::new("donation_denom");
```

正如您所见，我们的管理员集合变成了一个合适的映射（Map） - 我们将为每个管理员分配加入时间。

现在我们需要更新如何初始化映射 - 之前我们存储了`Empty`数据，但它不再匹配我们的值类型。让我们来看看更新后的实例化函数：

您可能会提出为该映射的值创建一个单独的结构，以便将来如果需要添加其他内容，但在我看来，这可能过早 - 我们以后还可以更改整个值类型，因为这将是一个破坏性的改变。

现在我们需要更新如何初始化映射 - 之前我们存储了`Empty`数据，但它不再匹配我们的值类型。让我们来看看更新后的实例化函数：

```rust
use crate::state::{ADMINS, DONATION_DENOM};
use cosmwasm_std::{
    DepsMut, Env, MessageInfo, Response, StdResult,
};

pub fn instantiate(
    deps: DepsMut,
    env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    for addr in msg.admins {
        let admin = deps.api.addr_validate(&addr)?;
        ADMINS.save(deps.storage, &admin, &env.block.time)?;
    }
    DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;

    Ok(Response::new())
}
```
现在我们不再将`&Empty {}`作为管理员值进行存储，而是存储加入时间，我们从`&env.block.time`中读取。另外，请注意我从`env`块的名称中删除了下划线 - 它只是为了让Rust编译器知道该变量是有意未使用的，而不是某种bug。

最后，请记得在整个项目中删除任何过时的`Empty`导入 - 编译器应该会帮助您找出未使用的导入项。

## 查询和测试

关于加入时间，最后要添加的是新的查询，用于询问特定管理员的加入时间。您已经讨论过完成这个任务所需的一切内容，我将把它留给您作为练习。查询变体应该类似于：

```rust
#[returns(JoinTimeResp)]
JoinTime { admin: String },
```

并且示例响应类型：

```rust
#[cw_serde]
pub struct JoinTimeResp {
    pub joined: Timestamp,
}
```

您可能会质疑在响应类型中，我建议始终返回一个`joined`值，但是如果没有添加这样的管理员怎么办？嗯，在这种情况下，我会依赖于`load`函数返回的存储中缺少值的描述性错误 - 但是，您可以根据需要定义自己的错误类型，甚至可以使`joined`字段可选，并在请求的管理员存在时返回。

最后，建议为新功能编写一个测试 - 在实例化后立即调用新的查询，以验证初始管理员具有正确的加入时间（可以通过扩展现有的实例化测试来实现）。

在测试中您可能需要帮助的一件事是如何获取执行的时间。使用任何操作系统时间都注定会失败 - 相反，您可以调用`block_info`函数以获取在应用程序中特定时刻的块状态，该函数返回一个包含块状态的[`BlockInfo`](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/struct.BlockInfo.html)结构。在实例化之前调用它可以确保您正在处理与调用时模拟的相同状态。
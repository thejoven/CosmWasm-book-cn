# 映射存储

管理员合约中有一个需要立即改进的地方，那就是检查合约的状态：

```rust
# use cosmwasm_std::Addr;
# use cw_storage_plus::Item;
# 
pub const ADMINS: Item<Vec<Addr>> = Item::new("admins");
pub const DONATION_DENOM: Item<String> = Item::new("donation_denom");
```

请注意，我们将管理员列表保存为一个向量（vector）。然而，在整个合约中，大多数情况下，我们只访问该向量的单个元素。

这并不理想，因为现在，每当我们想要访问单个管理员条目时，我们首先必须对包含所有管理员的列表进行反序列化，然后对它们进行迭代，直到找到感兴趣的条目。这可能会消耗大量的计算资源，并且是完全不必要的开销 - 我们可以使用 [Map](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html) 存储访问器来避免这种情况。

## `Map` 存储

首先，让我们定义一个 Map - 在这个上下文中，它将是一组具有分配给它们的值的键，就像 Rust 中的 `HashMap` 或许多其他语言中的字典一样。我们将其定义为类似于 `Item`，但这次我们需要两种类型 - 键类型和值类型：

```rust
use cw_storage_plus::Map;

pub const STR_TO_INT_MAP: Map<String, u64> = Map::new("str_to_int_map");
```

要在[`Map`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html)上存储一些项，我们使用[`save`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.save)方法 - 与`Item`相同：

```rust
STR_TO_INT_MAP.save(deps.storage, "ten".to_owned(), 10);
STR_TO_INT_MAP.save(deps.storage, "one".to_owned(), 1);
```

访问地图中的条目与读取项一样简单：

```rust
let ten = STR_TO_INT_MAP.load(deps.storage, "ten".to_owned())?;
assert_eq!(ten, 10);

let two = STR_TO_INT_MAP.may_load(deps.storage, "two".to_owned())?;
assert_eq!(two, None);
```

显然，如果在地图中缺少元素，[`load`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.load) 函数将会导致错误，就像对于项一样。另一方面，[`may_load`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.may_load) 在元素存在时返回 `Some` 变体。

另一个对地图特定的非常有用的访问器是 [`has`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.has) 函数，它检查地图中是否存在指定的键：

```rust
let contains = STR_TO_INT_MAP.has(deps.storage, "three".to_owned())?;
assert!(!contains);
```

最后，我们可以遍历地图的元素 - 无论是键还是键值对：

```rust
use cosmwasm_std::Order;

for k in STR_TO_INT_MAP.keys(deps.storage, None, None, Order::Ascending) {
    let _addr = deps.api.addr_validate(k?);
}

for item in STR_TO_INT_MAP.range(deps.storage, None, None, Order::Ascending) {
    let (_key, _value) = item?;
}
```
首先，你可能会想到传递给 [`keys`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.keys) 和 [`range`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.range) 的额外值 - 这些值是按顺序排列的：迭代元素的下限和上限，以及元素的遍历顺序。

在使用典型的 Rust 迭代器时，你可能会首先创建一个包含所有元素的迭代器，然后以某种方式跳过你不感兴趣的元素。之后，你将在最后一个有趣的元素之后停止。

通常情况下，这需要访问你筛选掉的元素，而这就是问题所在 - 它需要从存储中读取元素。从存储中读取元素是处理数据的昂贵部分，我们尽量避免这样做。一种方法是指示 Map 从存储中的哪个位置开始和停止反序列化元素，以便它永远不会到达范围之外的元素。

还有一个需要注意的关键事项是，keys 和 range 函数返回的迭代器并不是元素的迭代器 - 它们是 `Result` 的迭代器。这是因为，尽管罕见，但可能存在这样一种情况：项应该存在，但在从存储中读取时出现错误 - 也许存储的值以我们未预料的方式进行了序列化，导致反序列化失败。这实际上是我过去在其中一个合约上工作时遇到的真实情况 - 我们更改了 Map 的值类型，然后忘记进行迁移，导致出现各种问题。

## 将 Map 视为集合

所以我可以想象你现在可能会认为我很疯狂 - 为什么我在谈论一个 `Map`，而我们正在处理向量？显然这两者代表了两个不同的东西！还是说它们有所关联？

让我们重新考虑一下我们在 `ADMINS` 向量中保存的内容 - 我们有一个期望是唯一的对象列表，这是数学集合的定义。因此，现在让我重新提出我对地图的初步定义：

> 首先，让我们定义一个地图 - 在这个上下文中，它将是一个*键的集合*，每个键都有对应的值，就像 Rust 中的 HashMap 或许多其他语言中的字典一样。

我故意在这里使用了"集合"这个词 - 地图内嵌了集合。它是集合的一般化或逻辑的反转 - 集合是地图的特例。如果你想象一个将每个键映射到相同值的集合，那么值变得无关紧要，这样的地图在语义上就成为一个集合。

如何创建一个将所有键映射到相同值的地图？我们选择一个只有一个值的类型。通常在 Rust 中，它可能是一个 unit 类型 (`()`)，但在 CosmWasm 中，我们倾向于使用 CW 标准库中的 [`Empty`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.Empty.html) 类型：

```rust
use cosmwasm_std::{Addr, Empty};
use cw_storage_plus::Map;

pub const ADMINS: Map<Addr, Empty> = Map::new("admins");
```

现在我们需要修复合约中对地图的使用。让我们从合约的实例化开始：

```rust
use crate::msg::InstantiateMsg;
use crate::state::{ADMINS, DONATION_DENOM};
use cosmwasm_std::{
    DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    for addr in msg.admins {
        let admin = deps.api.addr_validate(&addr)?;
        ADMINS.save(deps.storage, admin, &Empty {})?;
    }
    DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;

    Ok(Response::new())
}
```

并没有简化太多，但我们不再需要收集我们的地址。然后让我们转向离开逻辑：

```rust
use crate::state::ADMINS;
use cosmwasm_std::{DepsMut, MessageInfo};

pub fn leave(deps: DepsMut, info: MessageInfo) -> StdResult<Response> {
    ADMINS.remove(deps.storage, info.sender.clone());

    let resp = Response::new()
        .add_attribute("action", "leave")
        .add_attribute("sender", info.sender.as_str());

    Ok(resp)
}
```

在这里我们看到了一个区别 - 我们不需要加载整个向量。我们使用 [`remove`](https://docs.rs/cw-storage-plus/1.0.1/cw_storage_plus/struct.Map.html#method.remove) 函数移除单个条目。

我之前没有强调的，并且相关的是，`Map` 将每个键都作为一个单独的项存储。这样，访问单个元素的成本比使用向量更低。

然而，这也有它的缺点 - 使用 Map 访问所有元素会消耗更多的 gas！一般情况下，我们会尽量避免这种情况 - 合约的线性复杂性可能导致非常昂贵的执行（从 gas 的角度）和潜在的漏洞 - 如果用户找到一种方法在这样的向量中创建许多虚拟元素，他可以使执行成本超过任何 gas 限制。

不幸的是，我们的合约中有这样的迭代 - 分发流程如下：

```rust
use crate::error::ContractError;
use crate::state::{ADMINS, DONATION_DENOM};
use cosmwasm_std::{
    coins, BankMsg,DepsMut, MessageInfo, Order, Response
};

pub fn donate(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
    let denom = DONATION_DENOM.load(deps.storage)?;
    let admins: Result<Vec<_>, _> = ADMINS
        .keys(deps.storage, None, None, Order::Ascending)
        .collect();
    let admins = admins?;

    let donation = cw_utils::must_pay(&info, &denom)?.u128();

    let donation_per_admin = donation / (admins.len() as u128);

    let messages = admins.into_iter().map(|admin| BankMsg::Send {
        to_address: admin.to_string(),
        amount: coins(donation_per_admin, &denom),
    });

    let resp = Response::new()
        .add_messages(messages)
        .add_attribute("action", "donate")
        .add_attribute("amount", donation.to_string())
        .add_attribute("per_admin", donation_per_admin.to_string());

    Ok(resp)
}
```

如果我必须编写这样一个合约，并且这个 `donate` 是一个关键且经常调用的流程，我会建议在这里使用 `Item<Vec<Addr>>`。希望情况并非如此 - 分发不必是线性复杂度的！这可能听起来有点疯狂，因为我们必须遍历所有接收者来分发资金，但事实并非如此 - 有一种非常好的方法可以在常数时间内完成，我将在本书后面描述。目前，我们将保持不变，承认合约的缺陷，稍后再修复。

需要修复的最后一个函数是 `admins_list` 查询处理程序：

```rust
use crate::state::ADMINS;
use cosmwasm_std::{Deps, Order, StdResult};

pub fn admins_list(deps: Deps) -> StdResult<AdminsListResp> {
    let admins: Result<Vec<_>, _> = ADMINS
        .keys(deps.storage, None, None, Order::Ascending)
        .collect();
    let admins = admins?;
    let resp = AdminsListResp { admins };
    Ok(resp)
}
```
在这里，我们也存在线性复杂性的问题，但问题要小得多。

首先，查询通常用于在本地节点上调用，没有 gas 成本 - 我们可以随意查询合约。

而且，即使我们对执行时间/成本有一些限制，也没有理由每次都查询所有项！我们将稍后修复这个函数，添加分页功能 - 限制查询调用者的执行时间/成本，使其能够请求从给定项开始的有限数量的项。通过阅读本章，你可能已经能够想象出实现它的方式，但当我介绍常见的 CosmWasm 实践时，我将展示我们通常是如何做到这一点的。

## 参考键

在我们使用地图的过程中，还有一个微妙之处可以改进。

问题在于现在我们使用拥有的 Addr 键对地图进行索引。这迫使我们在想要重用键时对其进行克隆（特别是在离开实现中）。这不是一个巨大的成本，但我们可以避免它 - 我们可以将地图的键定义为引用：

```rust
use cosmwasm_std::{Addr, Empty};
use cw_storage_plus::Map;

pub const ADMINS: Map<&Addr, Empty> = Map::new("admins");
pub const DONATION_DENOM: Item<String> = Item::new("donation_denom");
```

最后，我们需要在两个地方修复地图的用法：

```rust
# use crate::state::{ADMINS, DONATION_DENOM};
# use cosmwasm_std::{
#     DepsMut, Empty, Env, MessageInfo, Response, StdResult,
# };
#
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    for addr in msg.admins {
        let admin = deps.api.addr_validate(&addr)?;
        ADMINS.save(deps.storage, &admin, &Empty {})?;
    }

    // ...

#    DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;
#
   Ok(Response::new())
}

pub fn leave(deps: DepsMut, info: MessageInfo) -> StdResult<Response> {
    ADMINS.remove(deps.storage, &info.sender);

    // ...

#    let resp = Response::new()
#        .add_attribute("action", "leave")
#        .add_attribute("sender", info.sender.as_str());
#
   Ok(resp)
}
```

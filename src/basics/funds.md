# 处理资金

当您听到智能合约时，您会想到区块链。当您听到区块链时，您通常会想到加密货币。虽然它们并不完全相同，但加密资产，或者我们通常称之为代币，与区块链非常密切相关。CosmWasm有一个本地代币的概念。本地代币是由区块链核心而不是智能合约管理的资产。通常，这种资产具有一些特殊含义，比如用于支付[燃气费](https://docs.cosmos.network/master/basics/gas-fees.html)或[权益证明](https://en.wikipedia.org/wiki/Proof_of_stake)共识算法，但也可以是任意的资产。

本地代币分配给它们的所有者，但可以根据其性质进行转移。在区块链中，每个具有地址的事物都有资格拥有其本地代币。因此 - 代币可以分配给智能合约！发送到智能合约的每个消息都可以携带一些资金。在本章中，我们将利用这一点创建一种奖励管理员辛勤工作的方式。我们将创建一条新消息 - `Donate`，任何人都可以使用它向管理员捐赠一些资金，并平均分配。

## 准备消息

传统上，我们需要准备我们的消息。我们需要创建一个新的`ExecuteMsg`变体，但我们还将稍微修改`Instantiate`消息 - 我们需要有一种定义用于捐款的本地代币名称的方式。用户可以发送任何他们想要的代币，但现在我们想简化事情。

```rust,noplayground
# use cosmwasm_std::Addr;
# use serde::{Deserialize, Serialize};
# 
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
    pub donation_denom: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub enum ExecuteMsg {
    AddMembers { admins: Vec<String> },
    Leave {},
    Donate {},
}
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub struct GreetResp {
#     pub message: String,
# }
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub struct AdminsListResp {
#     pub admins: Vec<Addr>,
# }
# 
# #[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
# pub enum QueryMsg {
#     Greet {},
#     AdminsList {},
# }
```

我们还需要添加一个新的状态部分，以保留`donation_denom`（捐赠代币名称）：

```rust,noplayground
use cosmwasm_std::Addr;
use cw_storage_plus::Item;

pub const ADMINS: Item<Vec<Addr>> = Item::new("admins");
pub const DONATION_DENOM: Item<String> = Item::new("donation_denom");
```

并适当地进行实例化：

```rust,noplayground
# use crate::error::ContractError;
# use crate::msg::{AdminsListResp, ExecuteMsg, GreetResp, InstantiateMsg, QueryMsg};
use crate::state::{ADMINS, DONATION_DENOM};
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Env, Event, MessageInfo, Response, StdResult,
# };

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
    DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;

    Ok(Response::new())
}
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
# pub fn execute(
#     deps: DepsMut,
#     _env: Env,
#     info: MessageInfo,
#     msg: ExecuteMsg,
# ) -> Result<Response, ContractError> {
#     use ExecuteMsg::*;
# 
#     match msg {
#         AddMembers { admins } => exec::add_members(deps, info, admins),
#         Leave {} => exec::leave(deps, info).map_err(Into::into),
#     }
# }
# 
# mod exec {
#     use super::*;
# 
#     pub fn add_members(
#         deps: DepsMut,
#         info: MessageInfo,
#         admins: Vec<String>,
#     ) -> Result<Response, ContractError> {
#         let mut curr_admins = ADMINS.load(deps.storage)?;
#         if !curr_admins.contains(&info.sender) {
#             return Err(ContractError::Unauthorized {
#                 sender: info.sender,
#             });
#         }
# 
#         let events = admins
#             .iter()
#             .map(|admin| Event::new("admin_added").add_attribute("addr", admin));
#         let resp = Response::new()
#             .add_events(events)
#             .add_attribute("action", "add_members")
#             .add_attribute("added_count", admins.len().to_string());
# 
#         let admins: StdResult<Vec<_>> = admins
#             .into_iter()
#             .map(|addr| deps.api.addr_validate(&addr))
#             .collect();
# 
#         curr_admins.append(&mut admins?);
#         ADMINS.save(deps.storage, &curr_admins)?;
# 
#         Ok(resp)
#     }
# 
#     pub fn leave(deps: DepsMut, info: MessageInfo) -> StdResult<Response> {
#         ADMINS.update(deps.storage, move |admins| -> StdResult<_> {
#             let admins = admins
#                 .into_iter()
#                 .filter(|admin| *admin != info.sender)
#                 .collect();
#             Ok(admins)
#         })?;
# 
#         Ok(Response::new())
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
#     use crate::msg::AdminsListResp;
# 
#     use super::*;
# 
#     #[test]
#     fn instantiation() {
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
#         let resp: AdminsListResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::AdminsList {})
#             .unwrap();
# 
#         assert_eq!(resp, AdminsListResp { admins: vec![] });
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#                 },
#                 &[],
#                 "Contract 2",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: AdminsListResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::AdminsList {})
#             .unwrap();
# 
#         assert_eq!(
#             resp,
#             AdminsListResp {
#                 admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
#             }
#         );
#     }
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
# 
#     #[test]
#     fn unauthorized() {
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
#         let err = app
#             .execute_contract(
#                 Addr::unchecked("user"),
#                 addr,
#                 &ExecuteMsg::AddMembers {
#                     admins: vec!["user".to_owned()],
#                 },
#                 &[],
#             )
#             .unwrap_err();
# 
#         assert_eq!(
#             ContractError::Unauthorized {
#                 sender: Addr::unchecked("user")
#             },
#             err.downcast().unwrap()
#         );
#     }
# 
#     #[test]
#     fn add_members() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec!["owner".to_owned()],
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp = app
#             .execute_contract(
#                 Addr::unchecked("owner"),
#                 addr,
#                 &ExecuteMsg::AddMembers {
#                     admins: vec!["user".to_owned()],
#                 },
#                 &[],
#             )
#             .unwrap();
# 
#         let wasm = resp.events.iter().find(|ev| ev.ty == "wasm").unwrap();
#         assert_eq!(
#             wasm.attributes
#                 .iter()
#                 .find(|attr| attr.key == "action")
#                 .unwrap()
#                 .value,
#             "add_members"
#         );
#         assert_eq!(
#             wasm.attributes
#                 .iter()
#                 .find(|attr| attr.key == "added_count")
#                 .unwrap()
#                 .value,
#             "1"
#         );
# 
#         let admin_added: Vec<_> = resp
#             .events
#             .iter()
#             .filter(|ev| ev.ty == "wasm-admin_added")
#             .collect();
#         assert_eq!(admin_added.len(), 1);
# 
#         assert_eq!(
#             admin_added[0]
#                 .attributes
#                 .iter()
#                 .find(|attr| attr.key == "addr")
#                 .unwrap()
#                 .value,
#             "user"
#         );
#     }
# }
```

还需要对测试进行一些修正 - 实例化消息有一个新字段。我将其留给您作为一个练习。
现在，我们已经拥有了实现向管理员捐赠资金所需的一切。首先，对`Cargo.toml`进行一个小的更新 - 我们将使用一个额外的实用包：

```toml
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[features]
library = []

[dependencies]
cosmwasm-std = { version = "1.0.0-beta8", features = ["staking"] }
serde = { version = "1.0.103", default-features = false, features = ["derive"] }
cw-storage-plus = "0.13.4"
thiserror = "1"
schemars = "0.8.1"
cw-utils = "0.13"

[dev-dependencies]
cw-multi-test = "0.13.4"
cosmwasm-schema = { version = "1.0.0" }
```

然后我们可以实现捐赠处理程序：

```rust,noplayground
# use crate::error::ContractError;
# use crate::msg::{AdminsListResp, ExecuteMsg, GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::{ADMINS, DONATION_DENOM};
use cosmwasm_std::{
    coins, to_binary, BankMsg, Binary, Deps, DepsMut, Env, Event, MessageInfo,
    Response, StdResult,
};
 
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
#     DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;
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
pub fn execute(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    use ExecuteMsg::*;

    match msg {
        AddMembers { admins } => exec::add_members(deps, info, admins),
        Leave {} => exec::leave(deps, info).map_err(Into::into),
        Donate {} => exec::donate(deps, info),
    }
}

mod exec {
#     use super::*;
# 
#     pub fn add_members(
#         deps: DepsMut,
#         info: MessageInfo,
#         admins: Vec<String>,
#     ) -> Result<Response, ContractError> {
#         let mut curr_admins = ADMINS.load(deps.storage)?;
#         if !curr_admins.contains(&info.sender) {
#             return Err(ContractError::Unauthorized {
#                 sender: info.sender,
#             });
#         }
# 
#         let events = admins
#             .iter()
#             .map(|admin| Event::new("admin_added").add_attribute("addr", admin));
#         let resp = Response::new()
#             .add_events(events)
#             .add_attribute("action", "add_members")
#             .add_attribute("added_count", admins.len().to_string());
# 
#         let admins: StdResult<Vec<_>> = admins
#             .into_iter()
#             .map(|addr| deps.api.addr_validate(&addr))
#             .collect();
# 
#         curr_admins.append(&mut admins?);
#         ADMINS.save(deps.storage, &curr_admins)?;
# 
#         Ok(resp)
#     }
# 
#     pub fn leave(deps: DepsMut, info: MessageInfo) -> StdResult<Response> {
#         ADMINS.update(deps.storage, move |admins| -> StdResult<_> {
#             let admins = admins
#                 .into_iter()
#                 .filter(|admin| *admin != info.sender)
#                 .collect();
#             Ok(admins)
#         })?;
# 
#         Ok(Response::new())
#     }
# 
    pub fn donate(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
        let denom = DONATION_DENOM.load(deps.storage)?;
        let admins = ADMINS.load(deps.storage)?;

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
}
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
# #[cfg(test)]
# mod tests {
#     use cosmwasm_std::Addr;
#     use cw_multi_test::{App, ContractWrapper, Executor};
# 
#     use crate::msg::AdminsListResp;
# 
#     use super::*;
# 
#     #[test]
#     fn instantiation() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec![],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: AdminsListResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::AdminsList {})
#             .unwrap();
# 
#         assert_eq!(resp, AdminsListResp { admins: vec![] });
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract 2",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: AdminsListResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::AdminsList {})
#             .unwrap();
# 
#         assert_eq!(
#             resp,
#             AdminsListResp {
#                 admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
#             }
#         );
#     }
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
#                 &InstantiateMsg {
#                     admins: vec![],
#                     donation_denom: "eth".to_owned(),
#                 },
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
# 
#     #[test]
#     fn unauthorized() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec![],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let err = app
#             .execute_contract(
#                 Addr::unchecked("user"),
#                 addr,
#                 &ExecuteMsg::AddMembers {
#                     admins: vec!["user".to_owned()],
#                 },
#                 &[],
#             )
#             .unwrap_err();
# 
#         assert_eq!(
#             ContractError::Unauthorized {
#                 sender: Addr::unchecked("user")
#             },
#             err.downcast().unwrap()
#         );
#     }
# 
#     #[test]
#     fn add_members() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec!["owner".to_owned()],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp = app
#             .execute_contract(
#                 Addr::unchecked("owner"),
#                 addr,
#                 &ExecuteMsg::AddMembers {
#                     admins: vec!["user".to_owned()],
#                 },
#                 &[],
#             )
#             .unwrap();
# 
#         let wasm = resp.events.iter().find(|ev| ev.ty == "wasm").unwrap();
#         assert_eq!(
#             wasm.attributes
#                 .iter()
#                 .find(|attr| attr.key == "action")
#                 .unwrap()
#                 .value,
#             "add_members"
#         );
#         assert_eq!(
#             wasm.attributes
#                 .iter()
#                 .find(|attr| attr.key == "added_count")
#                 .unwrap()
#                 .value,
#             "1"
#         );
# 
#         let admin_added: Vec<_> = resp
#             .events
#             .iter()
#             .filter(|ev| ev.ty == "wasm-admin_added")
#             .collect();
#         assert_eq!(admin_added.len(), 1);
# 
#         assert_eq!(
#             admin_added[0]
#                 .attributes
#                 .iter()
#                 .find(|attr| attr.key == "addr")
#                 .unwrap()
#                 .value,
#             "user"
#         );
#     }
# }
```
将资金发送到另一个合约是通过在响应中添加银行消息来完成的。区块链期望在合约响应中返回的任何消息都作为执行的一部分。这个设计与CosmWasm实现的actor模型相关。整个actor模型将在稍后详细描述。目前，您可以将其视为处理代币转账的一种方式。在将代币发送给管理员之前，我们必须计算每个管理员的捐款金额。这是通过搜索描述我们的捐款代币的条目，并将发送的代币数量除以管理员数量来完成的。请注意，由于整数除法总是向下取整。

因此，可能并非所有作为捐款发送的代币都会最终分配给管理员账户。任何剩余的代币都将永远留在我们的合约账户上。有很多方法可以解决这个问题 - 找到其中一种方法将是一个很好的练习。

最后一个缺失的部分是更新`ContractError` - `must_pay`调用返回了一个我们尚不能转换为我们的错误类型的`cw_utils::PaymentError`：

```rust,noplayground
use cosmwasm_std::{Addr, StdError};
use cw_utils::PaymentError;
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    StdError(#[from] StdError),
    #[error("{sender} is not contract admin")]
    Unauthorized { sender: Addr },
    #[error("Payment error: {0}")]
    Payment(#[from] PaymentError),
}
```

正如您所见，为了处理进入的资金，我使用了实用函数 - 我鼓励您查看[其实现](https://docs.rs/cw-utils/0.13.4/src/cw_utils/payment.rs.html#32-39) - 这将让您对`MessageInfo`中的进入资金的结构有很好的理解。

现在是时候检查资金是否被正确分配了。为此，我们可以编写一个测试。

```rust,noplayground
# use crate::error::ContractError;
# use crate::msg::{AdminsListResp, ExecuteMsg, GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::{ADMINS, DONATION_DENOM};
# use cosmwasm_std::{
#     coins, to_binary, BankMsg, Binary, Deps, DepsMut, Env, Event, MessageInfo, Response, StdResult,
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
#     DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;
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
# pub fn execute(
#     deps: DepsMut,
#     _env: Env,
#     info: MessageInfo,
#     msg: ExecuteMsg,
# ) -> Result<Response, ContractError> {
#     use ExecuteMsg::*;
# 
#     match msg {
#         AddMembers { admins } => exec::add_members(deps, info, admins),
#         Leave {} => exec::leave(deps, info).map_err(Into::into),
#         Donate {} => exec::donate(deps, info),
#     }
# }
# 
# mod exec {
#     use super::*;
# 
#     pub fn add_members(
#         deps: DepsMut,
#         info: MessageInfo,
#         admins: Vec<String>,
#     ) -> Result<Response, ContractError> {
#         let mut curr_admins = ADMINS.load(deps.storage)?;
#         if !curr_admins.contains(&info.sender) {
#             return Err(ContractError::Unauthorized {
#                 sender: info.sender,
#             });
#         }
# 
#         let events = admins
#             .iter()
#             .map(|admin| Event::new("admin_added").add_attribute("addr", admin));
#         let resp = Response::new()
#             .add_events(events)
#             .add_attribute("action", "add_members")
#             .add_attribute("added_count", admins.len().to_string());
# 
#         let admins: StdResult<Vec<_>> = admins
#             .into_iter()
#             .map(|addr| deps.api.addr_validate(&addr))
#             .collect();
# 
#         curr_admins.append(&mut admins?);
#         ADMINS.save(deps.storage, &curr_admins)?;
# 
#         Ok(resp)
#     }
# 
#     pub fn leave(deps: DepsMut, info: MessageInfo) -> StdResult<Response> {
#         ADMINS.update(deps.storage, move |admins| -> StdResult<_> {
#             let admins = admins
#                 .into_iter()
#                 .filter(|admin| *admin != info.sender)
#                 .collect();
#             Ok(admins)
#         })?;
# 
#         Ok(Response::new())
#     }
# 
#     pub fn donate(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
#         let denom = DONATION_DENOM.load(deps.storage)?;
#         let admins = ADMINS.load(deps.storage)?;
# 
#         let donation = cw_utils::must_pay(&info, &denom)
#             .map_err(|err| StdError::generic_err(err.to_string()))?
#             .u128();
# 
#         let donation_per_admin = donation / (admins.len() as u128);
# 
#         let messages = admins.into_iter().map(|admin| BankMsg::Send {
#             to_address: admin.to_string(),
#             amount: coins(donation_per_admin, &denom),
#         });
# 
#         let resp = Response::new()
#             .add_messages(messages)
#             .add_attribute("action", "donate")
#             .add_attribute("amount", donation.to_string())
#             .add_attribute("per_admin", donation_per_admin.to_string());
# 
#         Ok(resp)
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
#     use crate::msg::AdminsListResp;
# 
#     use super::*;
# 
#     #[test]
#     fn instantiation() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec![],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: AdminsListResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::AdminsList {})
#             .unwrap();
# 
#         assert_eq!(resp, AdminsListResp { admins: vec![] });
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract 2",
#                 None,
#             )
#             .unwrap();
# 
#         let resp: AdminsListResp = app
#             .wrap()
#             .query_wasm_smart(addr, &QueryMsg::AdminsList {})
#             .unwrap();
# 
#         assert_eq!(
#             resp,
#             AdminsListResp {
#                 admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
#             }
#         );
#     }
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
#                 &InstantiateMsg {
#                     admins: vec![],
#                     donation_denom: "eth".to_owned(),
#                 },
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
# 
#     #[test]
#     fn unauthorized() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec![],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let err = app
#             .execute_contract(
#                 Addr::unchecked("user"),
#                 addr,
#                 &ExecuteMsg::AddMembers {
#                     admins: vec!["user".to_owned()],
#                 },
#                 &[],
#             )
#             .unwrap_err();
# 
#         assert_eq!(
#             ContractError::Unauthorized {
#                 sender: Addr::unchecked("user")
#             },
#             err.downcast().unwrap()
#         );
#     }
# 
#     #[test]
#     fn add_members() {
#         let mut app = App::default();
# 
#         let code = ContractWrapper::new(execute, instantiate, query);
#         let code_id = app.store_code(Box::new(code));
# 
#         let addr = app
#             .instantiate_contract(
#                 code_id,
#                 Addr::unchecked("owner"),
#                 &InstantiateMsg {
#                     admins: vec!["owner".to_owned()],
#                     donation_denom: "eth".to_owned(),
#                 },
#                 &[],
#                 "Contract",
#                 None,
#             )
#             .unwrap();
# 
#         let resp = app
#             .execute_contract(
#                 Addr::unchecked("owner"),
#                 addr,
#                 &ExecuteMsg::AddMembers {
#                     admins: vec!["user".to_owned()],
#                 },
#                 &[],
#             )
#             .unwrap();
# 
#         let wasm = resp.events.iter().find(|ev| ev.ty == "wasm").unwrap();
#         assert_eq!(
#             wasm.attributes
#                 .iter()
#                 .find(|attr| attr.key == "action")
#                 .unwrap()
#                 .value,
#             "add_members"
#         );
#         assert_eq!(
#             wasm.attributes
#                 .iter()
#                 .find(|attr| attr.key == "added_count")
#                 .unwrap()
#                 .value,
#             "1"
#         );
# 
#         let admin_added: Vec<_> = resp
#             .events
#             .iter()
#             .filter(|ev| ev.ty == "wasm-admin_added")
#             .collect();
#         assert_eq!(admin_added.len(), 1);
# 
#         assert_eq!(
#             admin_added[0]
#                 .attributes
#                 .iter()
#                 .find(|attr| attr.key == "addr")
#                 .unwrap()
#                 .value,
#             "user"
#         );
#     }
# 
    #[test]
    fn donations() {
        let mut app = App::new(|router, _, storage| {
            router
                .bank
                .init_balance(storage, &Addr::unchecked("user"), coins(5, "eth"))
                .unwrap()
        });

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["admin1".to_owned(), "admin2".to_owned()],
                    donation_denom: "eth".to_owned(),
                },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        app.execute_contract(
            Addr::unchecked("user"),
            addr.clone(),
            &ExecuteMsg::Donate {},
            &coins(5, "eth"),
        )
        .unwrap();

        assert_eq!(
            app.wrap()
                .query_balance("user", "eth")
                .unwrap()
                .amount
                .u128(),
            0
        );

        assert_eq!(
            app.wrap()
                .query_balance(&addr, "eth")
                .unwrap()
                .amount
                .u128(),
            1
        );

        assert_eq!(
            app.wrap()
                .query_balance("admin1", "eth")
                .unwrap()
                .amount
                .u128(),
            2
        );

        assert_eq!(
            app.wrap()
                .query_balance("admin2", "eth")
                .unwrap()
                .amount
                .u128(),
            2
        );
    }
}
```
相当简单。我并不特别喜欢每个余额检查都有八行代码，但是可以通过将这个断言封装到一个单独的函数中来改进，可能需要使用[`#[track_caller]`](https://doc.rust-lang.org/reference/attributes/diagnostics.html#the-track_caller-attribute)属性。

需要讨论的关键问题是`app`创建的变化。因为我们需要在`user`账户上有一些初始代币，所以不能使用默认构造函数，而是必须提供一个初始化函数。不幸的是，[`new`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.App.html#method.new)的文档不容易理解，即使函数本身并不复杂。它作为参数接受一个闭包，有三个参数——包含multi-test支持的所有模块的[`Router`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.Router.html)、API对象和状态。这个函数在合约实例化时只被调用一次。`router`对象包含一些通用字段，我们特别关注`bank`。它的类型是[`BankKeeper`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.BankKeeper.html)，其中包含[`init_balance`](https://docs.rs/cw-multi-test/0.13.4/cw_multi_test/struct.BankKeeper.html#method.init_balance)函数。

## 惊喜！

由于我们已经介绍了构建Rust智能合约的大部分重要基础知识，我给你出了一个严肃的练习。

我们构建的合约有一个可以被利用的漏洞。所有的捐赠款项都平均分配给管理员。然而，每个管理员都有资格添加另一个管理员。而且没有任何限制阻止管理员将自己添加到列表中并获得其他人两倍的奖励！

请尝试编写一个能检测到这种漏洞的测试，并修复它，确保该漏洞不再发生。

即使管理员不能将相同的地址添加到列表中，他仍然可以创建新的账户并将它们添加进去，但这是在合约层面上无法阻止的，所以不要阻止这种情况的发生。处理这种情况是通过正确设计整个应用程序来完成的，超出了本章的范围。
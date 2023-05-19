# 事件属性和数据

目前，我们合约与外界进行通信的唯一方式是通过查询。智能合约是被动的，它们不能自行调用任何操作。它们只能作为对调用的反应而执行。但是，如果您尝试过与wasmd进行交互，您就会知道在区块链上执行可以返回一些元数据。

合约可以向调用者返回两种内容：事件和数据。几乎每个现实生活中的智能合约都会产生事件。相比之下，数据很少使用，设计用于合约之间的通信。

## 返回事件

作为示例，我们将在执行`AddMembers`时，由我们的合约发出一个名为`admin_added`的事件：

```rust,noplayground
# use crate::error::ContractError;
# use crate::msg::{AdminsListResp, ExecuteMsg, GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::ADMINS;
use cosmwasm_std::{
    to_binary, Binary, Deps, DepsMut, Env, Event, MessageInfo, Response, StdResult,
};
 
# pub fn instantiate(
#     deps: DepsMut,
#     _env: Env,
#     _info: MessageInfo,
#     msg: InstantiateMsg,
# ) -> StdResult<Response> {
#     let admins: StdResult<Vec<_>> = msg
#         .admins
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
mod exec {
#     use super::*;
# 
    pub fn add_members(
        deps: DepsMut,
        info: MessageInfo,
        admins: Vec<String>,
    ) -> Result<Response, ContractError> {
        let mut curr_admins = ADMINS.load(deps.storage)?;
        if !curr_admins.contains(&info.sender) {
            return Err(ContractError::Unauthorized {
                sender: info.sender,
            });
        }

        let events = admins
            .iter()
            .map(|admin| Event::new("admin_added").add_attribute("addr", admin));
        let resp = Response::new()
            .add_events(events)
            .add_attribute("action", "add_members")
            .add_attribute("added_count", admins.len().to_string());

        let admins: StdResult<Vec<_>> = admins
            .into_iter()
            .map(|addr| deps.api.addr_validate(&addr))
            .collect();

        curr_admins.append(&mut admins?);
        ADMINS.save(deps.storage, &curr_admins)?;

        Ok(resp)
    }
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
# }
```

事件由两部分构成：在[`new`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Event.html#method.new)函数中提供的事件类型和属性。属性可以通过[`add_attributes`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Event.html#method.add_attributes)或[`add_attribute`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Event.html#method.add_attribute)方法将其添加到事件中。属性是键值对。由于事件不能包含任何列表，为了报告多个类似的事件，我们需要发出多个小事件而不是一个集合性的事件。

通过使用[`add_event`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Response.html#method.add_event)或[`add_events`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Response.html#method.add_events)方法将事件添加到响应中来发出事件。此外，还可以直接将属性添加到响应中。这只是一种简化方式。默认情况下，每次执行都会发出一个标准的"wasm"事件。将属性添加到结果中会将它们添加到默认事件中。

我们可以检查合约是否正确发出了事件。这并不总是在测试中完成的，因为它在测试中往往是样板代码，而事件通常更像日志-并不一定被视为主要的合约逻辑。现在，让我们编写一个测试，检查执行是否发出了事件：

```rust,noplayground
# use crate::error::ContractError;
# use crate::msg::{AdminsListResp, ExecuteMsg, GreetResp, InstantiateMsg, QueryMsg};
# use crate::state::ADMINS;
# use cosmwasm_std::{
#     to_binary, Binary, Deps, DepsMut, Env, Event, MessageInfo, Response, StdResult,
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
    #[test]
    fn add_members() {
        let mut app = App::default();

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["owner".to_owned()],
                },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        let resp = app
            .execute_contract(
                Addr::unchecked("owner"),
                addr,
                &ExecuteMsg::AddMembers {
                    admins: vec!["user".to_owned()],
                },
                &[],
            )
            .unwrap();

        let wasm = resp.events.iter().find(|ev| ev.ty == "wasm").unwrap();
        assert_eq!(
            wasm.attributes
                .iter()
                .find(|attr| attr.key == "action")
                .unwrap()
                .value,
            "add_members"
        );
        assert_eq!(
            wasm.attributes
                .iter()
                .find(|attr| attr.key == "added_count")
                .unwrap()
                .value,
            "1"
        );

        let admin_added: Vec<_> = resp
            .events
            .iter()
            .filter(|ev| ev.ty == "wasm-admin_added")
            .collect();
        assert_eq!(admin_added.len(), 1);

        assert_eq!(
            admin_added[0]
                .attributes
                .iter()
                .find(|attr| attr.key == "addr")
                .unwrap()
                .value,
            "user"
        );
    }
}
```

正如您所见，在简单的测试中测试事件变得有些笨拙。首先，每个字符串都以字符串为基础-缺乏类型控制使得编写此类测试困难。而且，即使类型带有"wasm-"前缀-这可能不是一个很大的问题，但它并没有澄清验证的意义。但问题是，事件结构有多层，这使得验证它们变得棘手。而且，"wasm"事件特别棘手，因为它包含一个暗含的属性-`_contract_addr`，其中包含一个称为合约的地址。我的一般规则是-除非某些逻辑依赖于它们，否则不要测试发出的事件。

## 数据

除了事件之外，任何智能合约执行都可以产生一个`data`对象。与事件不同，`data`可以是结构化的。这使得它成为执行任何依赖于通信逻辑的更好选择。另一方面，事实证明，在合约之间的通信之外，它很少有帮助。`data`始终只是响应中的单个对象，可以使用[`set_data`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.Response.html#method.set_data)函数设置。由于在单个合约环境中它的用处较低，我们现在不会花时间详细讨论它-稍后在讨论合约之间通信时将涵盖它的示例。在那之前，了解这样的实体存在是有帮助的。
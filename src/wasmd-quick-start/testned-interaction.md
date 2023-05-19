# 与测试网络交互

区块链交互是通过[wasmd](https://github.com/CosmWasm/wasmd)命令行工具来执行的。要开始使用测试网络，我们需要上传一些智能合约代码。现在，我们将使用`cw-plus`存储库中的示例`cw4-group`。首先克隆该存储库：

```
$ git clone git@github.com:CosmWasm/cw-plus.git
```

现在转到克隆的存储库并对其运行Rust优化器：

```
$ docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/workspace-optimizer:0.12.6
```

几分钟后（第一次可能需要一些时间），您的存储库中应该有一个`artifact`目录，其中应该有一个`cw4-group.wasm`文件，这是我们要上传的合约。要执行此操作，请运行以下命令-请注意，`wallet`是您在前面章节中创建的密钥的名称：

```
$ wasmd tx wasm store ./artifacts/cw4_group.wasm --from wallet $TXFLAG -y -b block

...
logs:
- events:
  - attributes:
    - key: action
      value: /cosmwasm.wasm.v1.MsgStoreCode
    - key: module
      value: wasm
    - key: sender
      value: wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux
    type: message
  - attributes:
    - key: code_id
      value: "12"
    type: store_code
...
```
执行完毕后，您应该会得到一个非常长的输出，其中包含有关发生的情况的信息。其中大部分是一个古老的密码（也称为base64）和执行元数据，但我们要寻找的是`logs`部分。应该会有一个名为`store_code`的事件，其中只有一个属性`code_id`-其`value`字段是我们上传的合约的代码ID-在我的情况下是12。

现在，当我们上传了代码之后，我们可以继续实例化一个合约来创建它的新实例：

```
$ wasmd tx wasm instantiate 12 \
  '{ "admin": "wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux", "members": [] }' \
  --from wallet --label "Group" --no-admin $TXFLAG -y

...
logs:
- events:
  - attributes:
    - key: _contract_address
      value: wasm18yn206ypuxay79gjqv6msvd9t2y49w4fz8q7fyenx5aggj0ua37q3h7kwz
    - key: code_id
      value: "12"
    type: instantiate
  - attributes:
    - key: action
      value: /cosmwasm.wasm.v1.MsgInstantiateContract
    - key: module
      value: wasm
    - key: sender
      value: wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux
    type: message
...

```
在这个命令中，`12`是代码ID-上传代码的结果。之后，一个JSON是一个实例化消息-稍后我会详细讨论这个。只需将其视为一个需要字段来创建新合约的消息。每个合约都有其实例化消息格式。对于`cw4-group`，有两个字段：`admin`是一个地址，该地址有资格在该合约上执行消息。将其设置为您的地址非常重要，因为我们将希望学习如何执行合约。`members`是一个地址数组，是组的初始成员。我们现在将其保留为空，但您可以在其中放入任何地址。在这里，我在命令行中内联放入了有关消息的提示，但我经常将要发送的消息放入文件中，并通过`$(cat msg.json)`进行嵌入。这是fish语法，但是每个shell都提供了这样的语法。

然后，在消息之后，您需要添加几个附加标志。`--from wallet`与之前的相同-之前创建的密钥的名称。`--label "Group"`只是您的合约的任意名称。一个重要的标志是`--no-admin`标志-请记住，这是我们在实例化消息中设置的不同的"admin"。此标志仅与合约迁移相关，但我们现在不涉及它们，所以将此标志保持不变。

现在，看一下执行的结果。它与之前非常相似-有关执行过程的许多数据。同样，我们需要仔细查看响应的`logs`部分。这次我们正在查看一个类型为`instantiate`的事件，以及`_contract_address`属性-其值是新创建的合约地址-例如`wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux`。

现在让我们继续查询我们的合约：

```
$ wasmd query wasm contract-state smart \
  wasm18yn206ypuxay79gjqv6msvd9t2y49w4fz8q7fyenx5aggj0ua37q3h7kwz \
  '{ "list_members": {} }'

data:
  members: []
```

请记得将地址（紧接着`smart`后面的地址）更改为您的合约地址。之后，还有另一条消息-这次是查询消息-发送到合约中。该查询应该返回一个成员列表。实际上，它确实返回了一个单独的`data`对象，其中包含一个字段-空的成员列表。这很简单，现在让我们尝试最后一件事情：执行操作：

```
$ wasmd tx wasm execute \
  wasm18yn206ypuxay79gjqv6msvd9t2y49w4fz8q7fyenx5aggj0ua37q3h7kwz \
  '{ "update_members": { "add": [{ "addr": "wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux", "weight": 1 }], "remove": [] } }' \
  --from wallet $TXFLAG
```

正如您所看到的，执行操作与实例化非常相似。区别在于，实例化只调用一次，而执行需要一个合约地址。可以说实例化是第一次执行的特殊情况，它返回合约地址。与之前一样，我们可以看到我们得到了一些日志输出-您可以分析它以查看可能发生了什么。但要确保在区块链上有影响，最好的方法是再次查询它：

```
$ wasmd query wasm contract-state smart \
  wasm18yn206ypuxay79gjqv6msvd9t2y49w4fz8q7fyenx5aggj0ua37q3h7kwz \
  '{ "list_members": {} }'

data:
  members:
  - addr: wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux
    weight: 1
```

目前，这就是您需要了解的关于`wasmd`基础知识的全部内容，以便能够与您的简单合约进行交互。我们将重点关注在本地测试它们，但如果您想在真实环境中进行检查，您现在已经掌握了一些基本知识。稍后，当我们讨论定义智能合约之间通信的actor模型的架构时，我们将更详细地了解`wasmd`。

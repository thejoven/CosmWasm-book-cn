# 区块链中的角色

在之前的讨论中，我们主要从抽象的角度谈论了区块链中的角色。然而，在深入代码之前，我们需要建立一种共同的语言，因此我们将从外部用户的角度来看待智能合约，而不是从实现的角度来看。

在这一部分中，我将使用`wasmd`二进制文件与马拉加测试网络进行通信。要正确设置，请查看[使用`wasmd`进行快速入门](../wasmd-quick-start.md)。

## 区块链作为数据库

虽然可能有点从终点开始，但我想从角色模型中的状态部分开始。与传统系统相关联，我喜欢将区块链与数据库进行比较。

回顾之前的章节，我们了解到合约的最重要部分是它的状态。通过操作状态，我们可以持久地展示对世界所做的工作。那么，用来保持状态的是什么？它就是数据库！

所以，作为合约开发人员，我认为合约是一个分布式数据库，具有一些神奇的机制使其具有民主性。这些“神奇的机制”对于区块链的存在至关重要，它们是使用区块链的原因，但对于合约创建者而言，它们并不重要-对我们来说，唯一重要的是状态。

但是你可能会问：金融部分呢？区块链（特别是`wasmd`）不就是货币的实现吗？使用所有这些燃料成本，资金的转移似乎更像是资金转移，而不是数据库更新。是的，你有一定的道理，但我也有解决方案。想象一下，对于每种原生代币（我们指的是直接由区块链处理的代币，与cw20代币相对），有一个特殊的数据库桶（或者如果你愿意，可以称之为表），将地址映射到地址拥有的代币数量。你可以查询这个表（查询代币余额），但不能直接修改它。要对其进行修改，只需向一个特殊的内置银行合约发送消息即可。一切仍然是一个数据库。

但是，如果区块链是一个数据库，那么智能合约存储在哪里呢？显然-它们存储在数据库本身中！现在想象另一个特殊的表-它将包含一个由代码ID映射到Wasm二进制代码块的单个表。同样-要操作此表，您可以使用“特殊合约”，其他合约无法访问该合约，但可以通过`wasmd`二进制文件使用它
## 编译合约

暂时不涉及到具体的代码，但是为了开始，我们需要编译合约的二进制文件。为了简单起见，我们将使用[cw-plus](https://github.com/CosmWasm/cw-plus)中的`cw4-group`合约进行操作。首先，将该仓库克隆到本地：

```bash
$ git clone git@github.com:CosmWasm/cw-plus.git
```

然后进入`cw4-group`合约目录并进行构建：

```bash
$ cd cw-plus/contracts/cw4-group
$ docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/workspace-optimizer:0.12.6
```

当合约二进制文件构建完成后，与 CosmWasm 的第一次交互是将其上传到区块链（假设你的 wasm 二进制文件位于工作目录中）：

```bash
$ wasmd tx wasm store ./cw4-group.wasm --from wallet $TXFLAG -y -b block
```

这样的操作会得到类似以下的 JSON 输出：

```
..
logs:
..
- events:
  ..
  - attributes:
    - key: code_id
      value: "1069"
    type: store_code
```
我忽略了大部分不相关的字段，因为目前还不重要 - 我们关心的是区块链发出的包含存储合约的`code_id`信息的事件 - 在我的情况下，合约代码被存储在区块链上的`code_id`为`1069`。现在我可以通过查询来查看代码：

```bash
$ wasmd query wasm code 1069 code.wasm
```

现在，重要的是要理解合约代码并不是一个 Actor。那么，合约代码是什么？我认为最简单的方式是将其视为编程中的`class`或`type`。它定义了一些关于可以执行的操作，但在大多数情况下，该类本身并没有太多用处，除非我们创建该类型的实例，并对其调用类方法。现在让我们继续讨论这种合约类的实例。

## 合约实例

现在我们有了合约代码，但我们想要的是一个实际的合约。要创建它，我们需要进行实例化。与编程中的类比类似，实例化是调用构造函数的过程。为了做到这一点，我将发送一个实例化消息给我的合约：

```bash
$ wasmd tx wasm instantiate 1069 '{"members": []}' --from wallet --label "Group 1" --no-admin $TXFLAG -y
```
在这里，我创建了一个新的合约并立即对其调用了`Instantiate`消息。每个合约代码的消息结构都是不同的。特别是，`cw4-group`的Instantiate消息包含两个字段：

* `members`字段，它是初始组成员的列表。
* 可选的`admin`字段，它定义了谁可以添加或删除组成员的地址。

在这种情况下，我创建了一个没有管理员的空组 - 因此无法更改！这可能看起来像一个不太有用的合约，但它对我们来说作为合约示例很有用。

实例化的结果如下：

```
..
logs:
..
- events:
  ..
  - attributes:
    - key: _contract_address
      value: wasm1u0grxl65reu6spujnf20ngcpz3jvjfsp5rs7lkavud3rhppnyhmqqnkcx6
    - key: code_id
      value: "1069"
    type: instantiate
```
如您所见，我们再次查看`logs[].events[]`字段，寻找有趣的事件并从中提取信息 - 这是一种常见情况。我将在未来讨论事件及其属性，但总的来说，事件是通知世界某些事情发生的方式。还记得KFC的例子吗？如果服务员正在为我们送餐，她会在吧台上放置一个托盘，并大声喊出（或在屏幕上显示）订单号 - 这将是宣布事件，因此您知道某个操作的摘要，以便您可以采取一些有用的行动。

那么，我们可以用合约做什么呢？显然，我们可以调用它！但首先，我想告诉您一些有关地址的内容。

## CosmWasm中的地址

在CosmWasm中，地址是指向区块链中实体的一种方式。有两种类型的地址：合约地址和非合约地址。区别在于您可以向合约地址发送消息，因为它们与一些智能合约代码相关联，而非合约地址仅表示系统的用户。在Actor模型中，合约地址表示Actor，非合约地址表示系统的客户端。

在使用`wasmd`操作区块链时，您也拥有一个地址 - 当您将密钥添加到`wasmd`时会得到一个地址：


```bash
# add wallets for testing
$ wasmd keys add wallet3
- name: wallet3
  type: local
  address: wasm1dk6sq0786m6ayg9kd0ylgugykxe0n6h0ts7d8t
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"Ap5zuScYVRr5Clz7QLzu0CJNTg07+7GdAAh3uwgdig2X"}'
  mnemonic: ""
```

你可以随时检查你的地址：

```bash
$ wasmd keys show wallet
- name: wallet
  type: local
  address: wasm1um59mldkdj8ayl5gknp9pnrdlw33v40sh5l4nx
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"A5bBdhYS/4qouAfLUH9h9+ndRJKvK0co31w4lS4p5cTE"}'
  mnemonic: ""
```
拥有一个地址非常重要，因为它是能够调用任何东西的先决条件。当我们向合约发送消息时，合约始终知道发送该消息的地址，因此可以识别它 - 更不用说该发送方是一个会产生燃料费用的地址。

## 查询合约

所以，我们有了合约，让我们尝试对其进行一些操作 - 查询可能是最简单的事情。让我们来做一下：

```bash
$ wasmd query wasm contract-state smart wasm1u0grxl65reu6spujnf20ngcpz3jvjfsp5rs7lkavud3rhppnyhmqqnkcx6 '{ "list_members": {} }'
data:
  members: []
```
`wasm...`字符串是合约地址，您必须将其替换为您的合约地址。`{ "list_members": {} }`是我们发送给合约的查询消息。通常，CW智能合约的查询采用单个JSON对象的形式，其中包含一个字段：查询名称（在我们的情况下为`list_members`）。此字段的值是另一个对象，即查询参数 - 如果有的话。`list_members`查询处理两个参数：`limit`和`start_after`，它们都是可选的，并支持结果分页。但是，在我们的空组中，它们并不重要。

我们得到的查询结果以人类可读的文本形式呈现（如果我们想要获取JSON形式的结果 - 例如，为了进一步使用`jq`进行处理，只需传递`-o json`标志）。如您所见，响应包含一个字段：`members`，它是一个空数组。

那么，我们能否对这个合约做更多的事情？并不多。但是让我们尝试使用一个新合约做一些操作！

## 执行以执行一些操作

我们之前的合约的问题在于，对于`cw4-group`合约，唯一可以执行操作的人是管理员，但我们的合约没有管理员。虽然这并不适用于每个智能合约，但它是这个合约的特性。

因此，让我们创建一个新的组合约，但这次我们将自己设置为管理员。首先，请检查我们的钱包地址：

```bash
$ wasmd keys show wallet
```

实例化一个新的组合约 - 这次使用正确的管理员：

```bash
$ wasmd tx wasm instantiate 1069 '{"members": [], "admin": "wasm1um59mldkdj8ayl5gknp9pnrdlw33v40sh5l4nx"}' --from wallet --label "Group 1" --no-admin $TXFLAG -y
..
logs:
- events:
  ..
  - attributes:
    - key: _contract_address
      value: wasm1n5x8hmstlzdzy5jxd70273tuptr4zsclrwx0nsqv7qns5gm4vraqeam24u
    - key: code_id
      value: "1069"
    type: instantiate
```

您可能会问，为什么我们要传递某种`--no-admin`标志，如果我们刚刚说过，我们要将一个管理员设置给合约？答案令人沮丧和困惑，但是...这是一个不同的管理员。我们想要设置的管理员是由合约本身检查并管理的管理员。而通过`--no-admin`标志拒绝的管理员是wasmd级别的管理员，可以迁移合约。至少在您学习有关合约迁移之前，您不需要担心第二个管理员 - 在那之前，您可以始终向合约传递`--no-admin`标志。

现在让我们查询我们的新合约的成员列表：

```bash
$ wasmd query wasm contract-state smart wasm1n5x8hmstlzdzy5jxd70273tuptr4zsclrwx0nsqv7qns5gm4vraqeam24u '{ "list_members": {} }'
data:
  members: []
```

就像以前一样 - 最初没有成员。现在检查管理员：

```
$ wasmd query wasm contract-state smart wasm1n5x8hmstlzdzy5jxd70273tuptr4zsclrwx0nsqv7qns5gm4vraqeam24u '{ "admin": {} }'
data:
  admin: wasm1um59mldkdj8ayl5gknp9pnrdlw33v40sh5l4nx
```

所以，有一个管理员，看起来像我们想要的那个。现在我们会添加某人到这个组中 - 也许是我们自己？

```bash
wasmd tx wasm execute wasm1n5x8hmstlzdzy5jxd70273tuptr4zsclrwx0nsqv7qns5gm4vraqeam24u '{ "update_members": { "add": [{ "addr": "wasm1um59mldkdj8ayl5gkn
p9pnrdlw33v40sh5l4nx", "weight": 1 }], "remove": [] } }' --from wallet $TXFLAG -y
```

修改成员的消息是`update_members`，它有两个字段：要删除的成员和要添加的成员。要删除的成员只是地址。要添加的成员具有更复杂的结构：它们是带有两个字段的记录：地址和权重。权重对我们来说并不重要，它只是与每个组成员一起存储的元数据 - 对于我们来说，它始终为1。

让我们再次查询合约，检查我们的消息是否有任何更改：

```bash
$ wasmd query wasm contract-state smart wasm1n5x8hmstlzdzy5jxd70273tuptr4zsclrwx0nsqv7qns5gm4vraqeam24u '{ "list_members": {} }'
data:
  members:
  - addr: wasm1um59mldkdj8ayl5gknp9pnrdlw33v40sh5l4nx
    weight: 1
```
正如您所见，合约已更新其状态。基本上就是这样运作的 - 向合约发送消息会导致它们更新状态，并且可以随时查询状态。为了保持简单，我们现在只是直接通过`wasmd`与合约进行交互，但正如前面所描述的 - 合约可以相互通信。然而，要调查这一点，我们需要了解如何编写合约。下次我们将查看合约结构，并将其逐部分映射到我们到目前为止学到的内容中。

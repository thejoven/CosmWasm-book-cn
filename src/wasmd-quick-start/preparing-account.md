# 准备账户

与测试网络进行交互的第一件事是拥有一个有效的账户。首先，在`wasmd`配置中添加一个新的密钥：

```
$ wasmd keys add wallet
- name: wallet
  type: local
  address: wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"A8pamTZH8x8+8UAFjndrvU4x7foJbCvcz78buyQ8q7+k"}'
  mnemonic: ""
...
```

通过这个命令，您可以获得关于准备好的账户的信息。这里有两个相关的内容：
* 地址是您在区块链中的身份
* 助记词（在示例中由我省略）是由12个单词组成的，允许您重新创建一个账户，以便您可以在不同的设备上使用它

出于测试目的，存储助记词可能从来不是必要的，但在现实世界中，这是需要保持安全的关键信息。

现在，当您创建一个账户时，您需要用一些代币初始化它-您将需要这些代币来支付与区块链的任何交互-我们称之为操作的“燃料成本”。通常情况下，您需要以某种方式购买这些代币，但在测试网络中，您通常可以在您的账户上创建任意数量的代币。在malaga网络上，可以执行以下操作：

```
$ curl -X POST --header "Content-Type: application/json" \
  --data '{ "denom": "umlg", "address": "wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux" }' \
  https://faucet.malaga-420.cosmwasm.com/credit
```

这是一个简单的HTTP POST请求，目标是`https://faucet.malaga-420.cosmwasm.com/credit`端点。该请求的数据是一个JSON，包含要铸造的代币名称和应该接收新代币的地址。在这里，我们铸造的是`umlg`代币，这是在malaga测试网络中用于支付燃料费用的代币。

您现在可以通过调用以下命令（用您自己的地址替换我的地址）来验证您的账户代币余额：

```
$ wasmd query bank balances wasm1wukxp2kldxae36rgjz28umqtq792twtxdfe6ux
balances:
- amount: "100000000"
  denom: umlg
pagination:
  next_key: null
  total: "0"
```

拥有100M个代币应该足够供您玩耍了，如果您需要更多，您可以随时铸造另一批代币。
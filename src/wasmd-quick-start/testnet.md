# 测试网络设置

要与区块链测试网络交互，首先要做的是选择一个测试网络。我建议使用我们通用的CosmWasm测试网络malaga-420。由于`wasmd`是通过环境变量进行配置的，我们将开始创建一个`malaga.env`文件，并将其设置为正确的值：

```sh
export CHAIN_ID="malaga-420"
export TESTNET_NAME="malaga-420"
export FEE_DENOM="umlg"
export STAKE_DENOM="uand"
export BECH32_HRP="wasm"
export WASMD_VERSION="v0.27.0"
export CONFIG_DIR=".wasmd"
export BINARY="wasmd"

export GENESIS_URL="https://raw.githubusercontent.com/CosmWasm/testnets/master/malaga-420/config/genesis.json"

export RPC="https://rpc.malaga-420.cosmwasm.com:443"
export FAUCET="https://faucet.malaga-420.cosmwasm.com"

export COSMOVISOR_VERSION="v0.42.10"
export COSMOVISOR_HOME=/root/.wasmd
export COSMOVISOR_NAME=wasmd

export NODE=(--node $RPC)
export TXFLAG=($NODE --chain-id $CHAIN_ID --gas-prices 0.05umlg --gas auto --gas-adjustment 1.3)
```

如果您是Fish用户，这个`malaga.fish`文件可能更适合您：

```fish
set -x CHAIN_ID malaga-420
set -x TESTNET_NAME malaga-420
set -x FEE_DENOM umlg
set -x STAKE_DENOM uand
set -x BECH32_HRP wasm
set -x WASMD_VERSION v0.27.0
set -x CONFIG_DIR .wasmd
set -x BINARY wasmd

set -x GENESIS_URL https://raw.githubusercontent.com/CosmWasm/testnets/master/malaga-420/config/genesis.json

set -x RPC https://rpc.malaga-420.cosmwasm.com:443
set -x FAUCET https://faucet.malaga-420.cosmwasm.com

set -x COSMOVISOR_VERSION v0.42.10
set -x COSMOVISOR_HOME /root/.wasmd
set -x COSMOVISOR_NAME wasmd

set -x NODE $RPC
set -x TXFLAG --node $RPC --chain-id $CHAIN_ID --gas-prices 0.05umlg --gas-adjustment 1.3 --gas auto -b block
```

Now source the file to our environment (for fish use `malaga.fish` in place of `malaga.env`):
```sh
$ source ./malaga.env
```

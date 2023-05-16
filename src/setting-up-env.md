# 设置环境

要使用 CosmWasm 智能合约，你需要在你的机器上安装 rust。如果你还没有，你可以在[ Rust 网站](https://www.rust-lang.org/tools/install)上找到安装说明。

我假设你在这本书中使用的是稳定的 Rust 渠道。

此外，你需要安装 Wasm rust 编译器后端来构建 Wasm 二进制文件。安装它，请运行：

```
rustup target add wasm32-unknown-unknown
```

如果你想在测试网上试用你的合约，你将需要一个 [wasmd](https://github.com/CosmWasm/wasmd) 二进制文件。我们将专注于使用 Rust 单元测试工具测试合约，所以不需要跟随。然而，看到产品在真实环境中工作可能很好。

要安装 `wasmd`，首先安装 [golang](https://github.com/golang/go/wiki#working-with-go)。然后克隆 `wasmd` 并安装它：

```
$ git clone git@github.com:CosmWasm/wasmd.git
$ cd ./wasmd
$ make install
```

此外，为了能够将 Rust Wasm 合约上传到区块链，你需要安装 [docker](https://www.docker.com/)。为了最小化你的合约大小，你需要运行 CosmWasm Rust Optimizer；没有它，更复杂的合约可能会超过大小限制。

## cosmwasm-check 工具

构建智能合约的另一个有用工具是 `cosmwasm-check`[工具](https://github.com/CosmWasm/cosmwasm/tree/main/packages/check)。它允许你检查 wasm 二进制文件是否是一个准备好上传到区块链的合适的智能合约。你可以使用 cargo 安装它：

```
$ cargo install cosmwasm-check
```

如果安装成功，你应该能够从命令行执行该工具。

```
$ cosmwasm-check --version
Contract checking 1.2.3
```

## 验证安装

为了确保你准备好构建你的智能合约，你需要确保你能够构建示例。查看 [cw-plus](https://github.com/CosmWasm/cw-plus) 仓库，并在其文件夹中运行测试命令：

```
$ git clone git@github.com:CosmWasm/cw-plus.git
$ cd ./cw-plus
cw-plus $ cargo test
```

你应该看到仓库中的所有内容都被编译，所有的测试都通过了。

`cw-plus` 是找到示例合约的好地方 - 在 `contracts` 目录中查找它们。该仓库由 CosmWasm 创建者维护，所以其中的合约应该遵循良好的实践。

要验证 `cosmwasm-check` 工具，首先，你需要构建一个智能合约。进入某个合约目录，例如 `contracts/cw1-whitelist`，并调用 `cargo wasm`：

```
cw-plus $ cd contracts/cw1-whitelist
cw-plus/contracts/cw1-whitelist $ cargo wasm
```

你应该能够在根仓库目录的 `target/wasm32-unknown-unknown/release/` 中找到你的输出二进制文件 - 不在合约目录本身！现在你可以检查合约验证是否通过：

```
cw-plus/contracts/cw1-whitelist $ cosmwasm-check ../../target/wasm32-unknown-unknown/release/cw1_whitelist.wasm
Available capabilities: {"iterator", "cosmwasm_1_1", "cosmwasm_1_2", "stargate", "staking"}

../../target/wasm32-unknown-unknown/release/cw1_whitelist.wasm: pass

All contracts (1) passed checks!
```
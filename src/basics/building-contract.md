# 构建合约

现在是构建我们的合约的时候了。我们可以使用传统的 cargo 构建管道进行本地测试：`cargo build` 用于编译它，`cargo test` 用于运行所有测试（我们还没有，但我们很快就会进行）。

然而，我们需要创建一个 wasm 二进制文件以将合约上传到区块链。我们可以通过向构建命令传递额外的参数来实现这一点：

```
$ cargo build --target wasm32-unknown-unknown --release
```

`--target` 参数告诉 cargo 为给定目标执行交叉编译，而不是为运行它的操作系统构建原生二进制文件 - 在这种情况下，`wasm32-unknown-unknown`，这是 Wasm 目标的奇特名称。

另外，我将 `--release` 参数传递给了命令 - 这并不是必需的，但在大多数情况下，链上运行时调试信息并不是很有用。减小上传的二进制文件大小对于最小化 gas 成本是至关重要的。值得知道的是，有一个 [CosmWasm Rust 优化器](https://github.com/CosmWasm/rust-optimizer) 工具，负责构建更小的二进制文件。对于生产环境，所有的合约都应该使用这个工具进行编译，但是对于学习目的，这并不是必须做的。

## 别名构建命令

现在我可以看出你对使用一些过于复杂的命令构建你的合约感到失望，而不是简单的 `cargo build`。希望这不是事实。常见的做法是为构建命令设置别名，使其像构建原生应用一样简单。

让我们在你的合约项目目录中创建 `.cargo/config` 文件，内容如下：

```toml
[alias]
wasm = "build --target wasm32-unknown-unknown --release"
wasm-debug = "build --target wasm32-unknown-unknown"
```

现在，构建你的 Wasm 二进制文件就像执行 `cargo wasm` 一样简单。我们还添加了额外的 `wasm-debug` 命令，用于我们想要构建包含调试信息的 wasm 二进制文件的罕见情况。

## 检查合约有效性

当合约被构建时，最后一步是确保它是有效的 CosmWasm 合约，方法是对其调用 `cosmwasm-check`：

```
$ cargo wasm
...
$ cosmwasm-check ./target/wasm32-unknown-unknown/release/contract.wasm
Available capabilities: {"cosmwasm_1_1", "staking", "stargate", "iterator", "cosmwasm_1_2"}

./target/wasm32-unknown-unknown/release/contract.wasm: pass
```

# Actor模型

本节介绍了CosmWasm智能合约架构的基础，它决定了它们之间的通信方式。在逐步教授如何创建彼此相关的多个合约之前，我希望先了解这一点，以便对未来可能遇到的情况有一个初步的了解。如果第一次阅读后仍不清楚，不用担心 - 我建议现在先阅读一遍这一章节，以后在了解了实际部分后再来回顾一下。

这里描述的整个过程在 `cosmwasm` 仓库的 [SEMANTICS.md](https://github.com/CosmWasm/cosmwasm/blob/main/SEMANTICS.md) 中有正式文档记录。
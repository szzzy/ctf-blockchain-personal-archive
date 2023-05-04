# realwrap

## 源码

## 题解

主要是这一篇，来自于预编译合约的问题

[How to Steal $100M from Flawless Smart Contracts — PWNING (mirror.xyz)](https://pwning.mirror.xyz/okyEG4lahAuR81IMabYL5aUdvAsZ8cRCbYBXh8RHFuE)

执行委托调用`delegatecall` 预编译的合约时，调用上下文将从前一个上下文继承，包括 `msg.sender` 和 `msg.value`

这样可以通过 approve 或者 transfer 的检测

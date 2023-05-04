# Transfer

## 源码

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.7;

contract Transfer{
    constructor() {}

    function isSolved() public view returns(bool) {
        return address(this).balance >= 0.5 ether; 
    }
}
```

## 题解

一眼自毁强转

题目环境改了，可以remix交互了，那直接remix秒了，没什么好说的
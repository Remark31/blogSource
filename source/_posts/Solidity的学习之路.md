---
title: Solidity的学习之路
date: 2018-05-25 19:39:37
tags: [区块链,智能合约,以太坊]
toc: true
---

# 合约

## 基本结构与节本数据类型

### 变量

- 全局变量
    - 会持久化到区块中
- 局部变量
    - 仅仅在内存中执行

### 时间变量

- now
- seconds
- minutes
- hours
- days
- weeks
- years

```
uint lastUpdated;

// 将‘上次更新时间’ 设置为 ‘现在’
function updateTimestamp() public {
  lastUpdated = now;
}

// 如果到上次`updateTimestamp` 超过5分钟，返回 'true'
// 不到5分钟返回 'false'
function fiveMinutesHavePassed() public view returns (bool) {
  return (now >= (lastUpdated + 5 minutes));
}
```

### 随机数

- 用 keccak256 来制造随机数
- 其他参考资料：https://ethereum.stackexchange.com/questions/191/how-can-i-securely-generate-a-random-number-in-my-smart-contract



```
// 生成一个0到100的随机数:
uint randNonce = 0;
uint random = uint(keccak256(now, msg.sender, randNonce)) % 100;
randNonce++;
uint random2 = uint(keccak256(now, msg.sender, randNonce)) % 100;
```

### maps

和普通代码编程的map并无区别，存储键值对

```
mapping (address => uint) public accountBalance;
```

### 数学运算

- 加法: `x + y`
- 减法: `x - y`
- 乘法: `x * y`
- 除法: `x / y`
- 取模: `x % y`
- 乘方: `x ** y`


### 数组

- 公共数组：Solidity 会自动创建 getter 方法
```
Person[] public people;
```

### 类型转换
- keccak256：内部SHA3散列函数
- uint8(a)

### Storage与Memory
> Storage 永久存储在区块链中的变量    
> Memory 存储在内存中的变量

 - 状态变量（在函数之外声明的变量）默认为“存储”形式，并永久写入区块链；
 - 而在函数内部声明的变量是“内存”型的，它们函数调用结束后消失。


```
contract SandwichFactory {
  struct Sandwich {
    string name;
    string status;
  }

  Sandwich[] sandwiches;

  function eatSandwich(uint _index) public {
    // Sandwich mySandwich = sandwiches[_index];

    // ^ 看上去很直接，不过 Solidity 将会给出警告
    // 告诉你应该明确在这里定义 `storage` 或者 `memory`。

    // 所以你应该明确定义 `storage`:
    Sandwich storage mySandwich = sandwiches[_index];
    // ...这样 `mySandwich` 是指向 `sandwiches[_index]`的指针
    // 在存储里，另外...
    mySandwich.status = "Eaten!";
    // ...这将永久把 `sandwiches[_index]` 变为区块链上的存储

    // 如果你只想要一个副本，可以使用`memory`:
    Sandwich memory anotherSandwich = sandwiches[_index + 1];
    // ...这样 `anotherSandwich` 就仅仅是一个内存里的副本了
    // 另外
    anotherSandwich.status = "Eaten!";
    // ...将仅仅修改临时变量，对 `sandwiches[_index + 1]` 没有任何影响
    // 不过你可以这样做:
    sandwiches[_index + 1] = anotherSandwich;
    // ...如果你想把副本的改动保存回区块链存储
  }
}
```

## 控制结构

- if 
- for 


```
function getEvens() pure external returns(uint[]) {
  uint[] memory evens = new uint[](5);
  // 在新数组中记录序列号
  uint counter = 0;
  // 在循环从1迭代到10：
  for (uint i = 1; i <= 10; i++) {
    // 如果 `i` 是偶数...
    if (i % 2 == 0) {
      // 把它加入偶数数组
      evens[counter] = i;
      //索引加一， 指向下一个空的‘even’
      counter++;
    }
  }
  return evens;
}
```

## 函数

### 属性

- 私有:private
- 公共:public
- 返回值
    
    

- 内部：internal：
    - internal 和 private 类似，不过， 如果某个合约继承自其父合约，这个合约即可以访问父合约中定义的“内部”函数。
- 外部：external
    - external与public 类似，只不过这些函数只能在合约之外调用。


- 多返回值

```
function multipleReturns() internal returns(uint a, uint b, uint c) {
  return (1, 2, 3);
}

function processMultipleReturns() external {
  uint a;
  uint b;
  uint c;
  // 这样来做批量赋值:
  (a, b, c) = multipleReturns();
}

// 或者如果我们只想返回其中一个变量:
function getLastReturnValue() external {
  uint c;
  // 可以对其他字段留空:
  (,,c) = multipleReturns();
}
```

- 结构体传参

```
function _doStuff(Zombie storage _zombie) internal {
  // do stuff with _zombie
}
```

### 修饰符


- pure：不读取应用里的状态
- view：只读取数据，不更改数据
```
function _generateRandomDna(string _str) private view returns (uint) {
        uint rand = uint(keccak256(_str));
        return rand % dnaModulus;
    }

```
- 可以带参数


```
// 存储用户年龄的映射
mapping (uint => uint) public age;

// 限定用户年龄的修饰符
modifier olderThan(uint _age, uint _userId) {
  require(age[_userId] >= _age);
  _;
}

// 必须年满16周岁才允许开车 (至少在美国是这样的).
// 我们可以用如下参数调用`olderThan` 修饰符:
function driveCar(uint _userId) public olderThan(16, _userId) {
  // 其余的程序逻辑
}
```

- payable

msg.value表示查看向合约中发送了多少以太的方法。

```
contract OnlineStore {
  function buySomething() external payable {
    // 检查以确定0.001以太发送出去来运行函数:
    require(msg.value == 0.001 ether);
    // 如果为真，一些用来向函数调用者发送数字内容的逻辑
    transferThing(msg.sender);
  }
}
```

- 提现

this.balance 存储了现有的eth
```
contract GetPaid is Ownable {
  function withdraw() external onlyOwner {
    owner.transfer(this.balance);
  }
}

```



> TIPS.     
> 1. 不能有相同名称的修饰符和函数。

## 包的导入

> 引入文件

```
import "./someothercontract.sol";

contract newContract is SomeOtherContract {

}
```

## 结构体

与普通的结构体并无区别

```
struct Person {
    uint age;
    string name;
}
```

## 接口与事件

### 事件
> 事件 是合约和区块链通讯的一种机制。你的前端应用“监听”某些事件，并做出反应。


- 事件创建
    
- 事件通知

```
pragma solidity ^0.4.19;

contract ZombieFactory {

    event NewZombie(uint zombieId, string name, uint dna);

    uint dnaDigits = 16;
    uint dnaModulus = 10 ** dnaDigits;

    struct Zombie {
        string name;
        uint dna;
    }

    Zombie[] public zombies;

    function _createZombie(string _name, uint _dna) private {
        uint id = zombies.push(Zombie(_name, _dna)) - 1;
        NewZombie(id, _name, _dna);
    }

    function _generateRandomDna(string _str) private view returns (uint) {
        uint rand = uint(keccak256(_str));
        return rand % dnaModulus;
    }

    function createRandomZombie(string _name) public {
        uint randDna = _generateRandomDna(_name);
        _createZombie(_name, randDna);
    }

}


```

### 接口

- 定义接口



- 使用接口


```
假设原合约为：
contract LuckyNumber {
  mapping(address => uint) numbers;

  function setNum(uint _num) public {
    numbers[msg.sender] = _num;
  }

  function getNum(address _myAddress) public view returns (uint) {
    return numbers[_myAddress];
  }
}

在新合约中定义接口为：
contract NumberInterface {
  function getNum(address _myAddress) public view returns (uint);
}

在新合约中使用接口为：
contract MyContract {
  address NumberInterfaceAddress = 0xab38...;
  // ^ 这是FavoriteNumber合约在以太坊上的地址
  NumberInterface numberContract = NumberInterface(NumberInterfaceAddress);
  // 现在变量 `numberContract` 指向另一个合约对象

  function someFunction() public {
    // 现在我们可以调用在那个合约中声明的 `getNum`函数:
    uint num = numberContract.getNum(msg.sender);
    // ...在这儿使用 `num`变量做些什么
  }
}
```


## 合约交互


### msg.sender
> 指的是当前调用者（或智能合约）的 address。

### Require
> 使得函数在执行过程中，当不满足某些条件时抛出错误，并停止执行

```
require(keccak256(_name) == keccak256("Vitalik"));
```






## 继承

### 继承
> 与普通面向对象语言的继承一样

```
ontract Doge {
  function catchphrase() public returns (string) {
    return "So Wow CryptoDoge";
  }
}

contract BabyDoge is Doge {
  function anotherCatchphrase() public returns (string) {
    return "Such Moon BabyDoge";
  }
}
```

### 多重继承
> 使用多重继承的时候，你只需要用逗号 , 来隔开几个你想要继承的合约

```
contract SatoshiNakamoto is NickSzabo, HalFinney {
  // 啧啧啧，宇宙的奥秘泄露了
}
```






## 合约库

### Ownable 合约

```
/**
 * @title Ownable
 * @dev The Ownable contract has an owner address, and provides basic authorization control
 * functions, this simplifies the implementation of "user permissions".
 */
contract Ownable {
  address public owner;
  event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

  /**
   * @dev The Ownable constructor sets the original `owner` of the contract to the sender
   * account.
   */
  function Ownable() public {
    owner = msg.sender;
  }

  /**
   * @dev Throws if called by any account other than the owner.
   */
  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }

  /**
   * @dev Allows the current owner to transfer control of the contract to a newOwner.
   * @param newOwner The address to transfer ownership to.
   */
  function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != address(0));
    OwnershipTransferred(owner, newOwner);
    owner = newOwner;
  }
}
```

- 构造函数：function Ownable()，初始化函数
- 函数修饰符：modifier onlyOwner()，用来修饰其它已有函数
    - 执行时先执行修饰符中的内容，到`_`部分后开始执行原有函数的内容

```
contract MyContract is Ownable {
  event LaughManiacally(string laughter);

  //注意！ `onlyOwner`上场 :
  function likeABoss() external onlyOwner {
    LaughManiacally("Muahahahaha");
  }
}
```


### ERC721 标准

```
contract ERC721 {
  event Transfer(address indexed _from, address indexed _to, uint256 _tokenId);
  event Approval(address indexed _owner, address indexed _approved, uint256 _tokenId);

  function balanceOf(address _owner) public view returns (uint256 _balance);
  function ownerOf(uint256 _tokenId) public view returns (address _owner);
  function transfer(address _to, uint256 _tokenId) public;  // 转移想移到的地址和代币
  function approve(address _to, uint256 _tokenId) public;   // 存储谁被允许提取代币
  function takeOwnership(uint256 _tokenId) public;          // 调用此处时检查后转移代币
}
```

### safeMath

```
library SafeMath {

  function mul(uint256 a, uint256 b) internal pure returns (uint256) {
    if (a == 0) {
      return 0;
    }
    uint256 c = a * b;
    assert(c / a == b);
    return c;
  }

  function div(uint256 a, uint256 b) internal pure returns (uint256) {
    // assert(b > 0); // Solidity automatically throws when dividing by 0
    uint256 c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold
    return c;
  }

  function sub(uint256 a, uint256 b) internal pure returns (uint256) {
    assert(b <= a);
    return a - b;
  }

  function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    assert(c >= a);
    return c;
  }
}
```


```
using SafeMath for uint256;

uint256 a = 5;
uint256 b = a.add(3); // 5 + 3 = 8
uint256 c = a.mul(2); // 5 * 2 = 10
```



## Gas的节省套路
- 如果一个 struct 中有多个 uint，则尽可能使用较小的 uint, Solidity 会将这些 uint 打包在一起，从而占用较少的存储空间

```
struct NormalStruct {
  uint a;
  uint b;
  uint c;
}

struct MiniMe {
  uint32 a;
  uint32 b;
  uint c;
}
```
- view 函数不花费gas
- 使用内存数组解决gas


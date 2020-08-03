# 面向合约编程 - 第一部分

这个系列是关于[面向合约编程](https://en.wikipedia.org/wiki/Design_by_contract)和 Solidity 的。“合约”在这个语境中其实不是个以太坊合约，或者一个智能合约；它是一个用来描述函数或功能单元的（目前非正式的）定义。其基本思想就是一个函数必须被拆分成前置条件、功能体和后置条件，不论是在（合约的）描述说明中还是在代码里。

显然这个东西还有更多内容，不过我们之后再深究，现在我们要开始看看 Solidity 函数和接口的一些基本功能，以及怎样应用面向合约的技巧。

尽管 Solidity 非常简单，它也不是一种非常好写代码的语言。它有合约、依赖库、函数、定制修饰器和事件。函数有很多需要正确选择的标准修饰器，并且有些修饰器实际上干些啥并不是很明显（例如`external`和`internal`）。特定的修饰器（比如`constant`）至今仍未强制使用，甚至就算你把所有修饰器都用对了，他们也会有问题，比如说在某些类型的函数中，某些 Solidity 类型不能被用作输入或输出数据。

话虽如此，习惯了也没那么糟糕，并且会越来越好。我在这里提到的情况并不是由于过差的语言设计，而仅仅因为缺少特定功能（语言的和 EVM 的）。这门编程语言还很年轻。但我还是觉得在进入合约部分前，有些东西必须要搞清楚。

（请注意，这些内容和观点的发布基础建立在此之上：实际的、正式的面向合约编程并不是标准；它没有全面的语言支持，也只有很少的信息和样例代码。因此，这些内容有一定程度上的探索和实验，请小心不要直接应用这些技巧，不论正确与否，也不要假定这会让合约100%安全。）

### 函数及标准<ruby>修饰器<rp>(</rp><rt>Modifier</rt><rp>)</rp></ruby>

修饰器被添加到变量和函数上以影响它们的行为。所有的基本修饰器都可以在[这里](https://en.wikipedia.org/wiki/Design_by_contract)找到。绝大多数常见的修饰器（譬如 public 和 private）都可用，但有些表现得和预期有一点点不同。

在描述所有修饰器之前我必须要重新过一下`call`和`transaction`机制：

一个`transaction`是一个由用户发向合约账户或外部账户、改变（或至少尝试改变）世界状态的已签名交易。交易被放进 tx 序列中并且不被认为是有效的，直到最终被挖进区块里。在发送以太币或执行任何**写入**操作时，必须使用交易。

一个`call`被用来从链上**读取**数据，或者做不改变世界状态的运算，因此不需要一个有效签名或者网络上其他用户的共识。一个使用`call`的例子应该是通过其公共访问方法检查合约的一个字段值。

#### `constant`

`constant`修饰器尚未被 Solidity 编译器要求强制使用，但其目的是向编译器和调用者发出消息：该函数并不改变世界状态。`constant`会出现在 JSON ABI 文件说明里，被`web3.js`——官方 JavaScript API——用来决定是用`transaction`还是`call`来调用函数。

#### `external`和`public`

从可见性的角度，`external`本质上与`public`是相同的。在部署一个有`public`或`external`函数的合约被后，其他合约、`call`和`transaction`都可以调用这个函数。

不要将`external`修饰器与白皮书中使用的“外部”混淆，后者指”外部账户。即不是合约账户的账户。`external`方法可以被其他合约调用，以及通过`transaction`或是`call`。

`external`和`call`修饰的函数间最大的区别，主要在于含有它们的合约调用它们的方式，以及处理输入参数的方法。如果你从同一合约内的另一个函数调用一个`public`函数，代码会通过使用一个`JUMP`来执行，有点像私有和内部函数，而`external`函数必须使用`CALL`指令来调用。另外，`external`函数不会从只读 calldata 数组复制输入数据到内存和/或栈变量里，这点可以用来优化。

最后，`public`是默认的可见度，意味着如果没有特别指出，函数都将是公开的。对于变量也有类似的机制，文档里可以找到更多信息。

#### `internal`

`internal`本质上跟`protected`是一样的。函数不能被其他合约调用，也不能被交易触发，抑或是调用合约，但可以被同一合约和继承的任一合约中的其他函数调用。

#### `private`

`private`函数只能被同一合约中的其他函数调用。

#### 例子

这是个简单的带有 public 和 private 函数的合约。可以复制粘贴进在线编译器里。

```Solidity
contract HelloVisibility {


    function hello() constant returns (string) {
        return "hello";
    }


    function helloLazy() constant returns (string) {
        return hello();
    }



    function helloAgain() constant returns (string) {
        return helloQuiet();
    }


    function helloQuiet() constant private returns (string) {
        return "hello";
    }
}
```

`hello`函数可以被其他合约调用，也能被自己的合约调用，就像`helloLazy`展示的那样。`helloQuiet`可以被其他函数调用，像`helloAgain`这样，但不能被其他合约调用，抑或是通过外部交易/调用。

请注意所有函数都被标记为`constant`，因为它们都不会改变世界状态。

请试着给`hello`函数添加`external`，然后你就会发现合约编译失败。如果由于某种原因`hello`必须要是外部的，我们可以试试修改代码，把`helloLazy`里的调用改为`this.hello()`，但这么做仍然无济于事！这就是我之前提到的另一个事情，即某些类型（具体说是动态长度数组）在特定函数里不能被用作输入或是输出值。

接下来两个合约展示了私有和内部的区别。

```Solidity
contract HelloGenerator {


    function helloQuiet() constant internal returns (string) {
        return "hello";
    }
}


contract Hello is HelloGenerator {


    function hello() constant external returns (string) {
        return helloQuiet();
    }
}
```

`Hello`继承了`HelloGenerator`且使用了它的内部函数以生成字符串。`HelloGenerator
`有一个空的 JSON ABI，因为它没有任何公开函数。试试把`helloQuiet`里的`internal`改成`private`这样你就可以收获一个编译器错误^_^。

### 自定义修饰器

在[这里](https://solidity-cn.readthedocs.io/zh/develop/contracts.html#modifier)可以找到自定义修饰器的文档。这里有个怎样使用修饰器的例子，和实现同一功能的三种不同方式：

```Solidity
contract GuardedFunctionExample1 {

    uint public data = 1;


    function guardedFunction(uint _data) {
        if(_data == 0)
            throw;
        data = _data;
    }
}


contract GuardedFunctionExample2 {

    uint public data = 1;


    function guardedFunction(uint _data) {
        check(_data);
        data = _data;
    }


    function check(uint _data) private {
        if (_data == 0)
            throw;
    }
}


contract GuardedFunctionExample3 {

    uint public data = 1;


    modifier checked(uint _data) {
        if (_data == 0)
            throw;
        _
    }


    function guardedFunction(uint _data) checked(_data) {
        data = _data;
    }
}
```

请注意：修饰器并不会在 JSON ABI 中出现。

#### 面向条件编程

看着前面的部分，可能会有人问“为啥要用修饰器把问题搞复杂呢”？如果我想要在其他函数复用这个检查，普通的解构就够了（例子2）。如果我想内联它以提高效率，我也就直接那么做了（例子1）。确实这样，但修饰器的语义更适合一个叫面向条件/条件导向编程的东西。COP 背后的基础思想被 Dr.Gavin Wood 在一篇[博客](https://medium.com/@gavofyork/condition-orientated-programming-969f6ba0161a#.8dw7jp1gq)里提出。这篇博客强调了一系列将（前置）条件和“商业逻辑”混合在一个函数里带来的丑陋副作用，并展示了怎么使用 COP 去避免这样。

>当程序员认为一个条件（和它投射到的状态）是一回事但实际上它是有着微妙区别的另一回事时，要遭大重。

一个这个的例子是让“the DAO”失败的代码。这已经被人总结过了，就是“the DAO”里的有些函数易受重放攻击，但这不是 EVM 的漏洞或是 Solidity 语言本身的问题，这是一个非常易犯的编程错误。这个漏洞的更多信息可以在[这篇](https://eng.erisindustries.com/programming/2016/06/18/lessons-learned-dao/) Solidity 核心开发者 RJ Catalano 的博文里找到。

Gavin 解释说，*本质上讲，COP 在编程中把前置条件当作一等公民*——本质上这也就是 Solidity 中的自定义修饰器——然后继续给出了一些用它们写出 COP 风格 Solidity 代码的简单例子。

##### 应用 COP

这部分包含了使用博客所描述的技巧构建的一个简单应用。下面的合约是一个 token 合约的变体，我们从没有修饰器开始。

```Solidity
contract Token {

    address public owner;

    // 每个人的余额
    mapping(address => uint) public balances;

    mapping(address => bool) public blacklisted;

    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        owner = msg.sender;
        balances[msg.sender] = 1000000;
    }


    function blacklist(address _addr) {
        if (msg.sender != owner)
            return;
        blacklisted[_addr] = true;
    }


    function transfer(uint _amount, address _dest) {
        if (blacklisted[msg.sender])
            return;
        if (balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
    }
}
```

我们要做的第一件事就是把`blacklist`函数的检查分离成一个修饰器然后把这个修饰器添加到函数上，就像这样：

```Solidity
modifier is_owner {
    if (msg.sender != owner)
        return;
    _;
}


function blacklist(address _addr) is_owner {
    blacklisted[_addr] = true;
}
```

注意修饰器如果不接受参数就不需要`()`。

搞好转账函数并不难。用在`is_owner`上的步骤可以同时用在两个检查上。难点在于我们需要为函数添加两个修饰器。

```Solidity
modifier not_blacklisted {
    if (blacklisted[msg.sender])
        return;
    _;
}


modifier at_least(uint x) {
    if (balances[msg.sender] < x)
        return;
    _;
}


function transfer(uint _amount, address _dest) not_blacklisted at_least(_amount) {
    balances[msg.sender] -= _amount;
    balances[_dest] += _amount;
}
```

请注意修饰器的顺序是从左到右的，因此为了保持顺序，黑名单检查应该先写。并且余额检查稍微改了改，虽然也可以这样写：

```Solidity
modifier at_least(uint x) {
    if (balances[msg.sender] >= x)
        _;
}
```

这个是改过的合约。肥肠干净。

```Solidity
contract COPToken {
    address public owner;

    // 每个人的余额
    mapping(address => uint) public balances;

    mapping(address => bool) public blacklisted;

    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        owner = msg.sender;
        balances[msg.sender] = 1000000;
    }


    modifier is_owner {
        if (msg.sender != owner)
            return;
        _;
    }


    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            return;
        _;
    }


    modifier at_least(uint x) {
        if (balances[msg.sender] < x)
            return;
        _;
    }


    function blacklist(address _addr) is_owner {
        blacklisted[_addr] = true;
    }


    function transfer(uint _amount, address _dest) not_blacklisted at_least(_amount) {
        balances[msg.sender] -= _amount;
        balances[_dest] += _amount;
    }
}
```

##### 单元测试

现在到了有意思的地方：我们怎么样保证所有修饰器都正常工作呢？到这个时候，我们需要做一些基本的单元测试。我们可以直接用不同的参数调用函数，然后检查结果，但这不够干净。我们要用继承来创建一个带有测试修饰器的函数的合约，和另外的函数用来测试被修饰的函数。被测试的合约甚至比之前的更简单。

```Solidity
contract Token {
    // 每个人的余额
    mapping(address => uint) public balances;

    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        balances[msg.sender] = 1000000;
    }


    modifier at_least(uint x) {
        if (balances[msg.sender] < x)
            return;
        _;
    }


    function transfer(uint _amount, address _dest) at_least(_amount) {
        balances[msg.sender] -= _amount;
        balances[_dest] += _amount;
    }
}


contract TokenTest is Token {
    address constant EMPTY_ACCOUNT = 0xDEADBEA7;


    function atLeastTester(uint _amount) at_least(_amount) constant private returns (bool) {
        return true;
    }


    function testAtLeastSuccess() returns (bool) {
        balances[msg.sender] = 1000;
        return atLeastTester(1000);
    }


    function testAtLeastFailBalanceTooLow() returns (bool) {
        balances[msg.sender] = 999;
        return !atLeastTester(1000);
    }


    // 测试：向账户转账0，然后检查余额
    function testTransfer() returns (bool) {
        balances[msg.sender] = 500;
        balances[EMPTY_ACCOUNT] = 0;
        transfer(500, EMPTY_ACCOUNT);
        return balances[msg.sender] == 0 && balances[EMPTY_ACCOUNT] == 500;
    }


    // 测试：向账户转账0，然后检查余额
    function testTransferFailBalanceTooLow() returns (bool) {
        balances[msg.sender] = 500;
        balances[EMPTY_ACCOUNT] = 0;
        transfer(600, EMPTY_ACCOUNT);
        return balances[msg.sender] == 500 && balances[EMPTY_ACCOUNT] == 0;
    }
}
```

测试合约里的第一个函数只是简单地跑了一下修饰器并在通过后返回 true。这就意味着当调用者的余额与提供的值相等或更高时，函数返回`true`，其他情况返回 false。接下来有两个函数检查修饰器是否按照设计工作了。最后有两个函数来检查转账函数的函数体是否按照期望工作。

这样会让转账函数的测试变得更容易。如果修饰器通过了测试但转账函数没有，很明显转账函数有点问题。如果修饰器测试出了问题，在修复修饰器前我们就不能期望转账函数正常工作。

### 复杂化

COP 式修饰器的方法并不是没有代价的。

在使用没有修饰器的普通函数时，是可以在出错时返回一个错误代码的。在示例合约中，我们也许会想要根据转账是否成功、是由于调用者在黑名单里还是余额不足返回不同的错误代码。这个在函数被其他合约调用时会很有用。修饰器在这个场景里有点“重”了，因为它们能做的就是 throw，即终止虚拟机并重置所有改动，或者 return，返回一个空值。

另一个问题就是当“事情正式了”之后的经典问题——代码会变得更加难写。举个例子，一个具有多重嵌套条件的函数，而且每个部分都有一大堆逻辑。Gavin 的例子用了个转帐逻辑（和它的检查）运行后调用的投票回绝函数，但事情可能会复杂很多；投票函数是默认调用的，它上面的自定义修饰器不接收参数，但如果有另外的函数应当根据交易结果被调用，并且那些函数反过来也有相似的条件，这样正确使用 COP 会变得非常困难。

### 总结

这是 Solidity 智能合约编写中的一个非常有趣的方法。和正式的验证、新的单元测试、静态分析工具、新的计划功能诸如 lambda 函数，以及坚持使用已有的安全语言功能，我们有望避免“the DAO”事件再次发生。

还有一个需要提到的是按合约设计并不是什么只适用于特殊语言（原文niche languages，非常精妙）的晦涩理论架构，它是一个很完善的编程范式，也有很多个能让它在主流语言比如 C++ 和 Java （某种程度上）中成立的库。某些语言（例如 C++）甚至有完整的语言支持。

最后，我十分不愿意在这里加入个人意见，但由于我本质上是在使用 Slock.it 的合约作为一个反面教材，我想指出的是我并不想责怪他们的程序员。没人能写出100%没有漏洞的完美代码，这也基本是他们应该做的。至少他们有足够的远见，搞了个安全锁来防止以太币直接被吸出合同。（Slock.it 的联合创始人 Christoph Jentzsch 发布了 DAO 提案框架的首个版本）

下个部分将会包含更多高阶合约和功能。会更贴合 Gavin 在他的教程里发布的第二部分。

Happy (and safe) smart contract writing!

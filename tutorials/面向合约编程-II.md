# 面向合约编程 - 第II部分

这是面向合约编程教程的第二部分。在这里我们要研究研究后置条件，和一些更复杂的函数。

请注意：就像在第一部分的一样，我们只用几个示例来展示怎么样在 Solidity 中使用自定义修饰器来应用一些简单的面向合约技巧。我们会在之后的内容里介绍更多细节，现在主要是为了实验和探索。

### 后置条件

后置条件可以用来确保跑了函数之后设想的事情确实发生了。通常这些事情会是某些状态的声明，譬如调用者的余额或者某个合约字段。我们在这里不限制可能要（或需要）检查的状态的类型。

一个很简单的后置条件的例子：

```Solidity
contract PostCheck {

    uint public data = 0;


    // 检查‘data’字段确实被改变成了‘_data’的值
    modifier data_is_valid(uint _data) {
        _;
        if (_data != data)
            throw;
    }


    function setData(uint _data) data_is_valid(_data) {
        data = _data;
    }
}
```

请注意`_`在后置条件修饰器里的位置；这样会在*检查之前*执行被修饰的函数，与前置条件修饰器里先检查再执行相反。

可以结合使用前置和后置条件。例子：

```Solidity
contract PrePostCheck {

    uint public data = 0;


    // 检查输入的‘_data’的值不等于'data'里已保存的值
    modifier data_is_valid(uint _data) {
        if (_data == data)
           throw;
        _；
    }


    // 检查‘data’字段确实被改变成了‘_data’的值
    modifier data_was_updated(uint _data) {
        _；
        if (_data != data)
            throw;
    }


    function setData(uint _data) data_is_valid(_data) data_was_updated(_data) {
        data = _data;
    }
}
```

这就是个非常安全的函数。它不仅检查了输入数据的有效性（在这个案例里是不与已存储的数据相同），还在返回前检查了 data 变量确实被改变了。因为单元测试本质上跟第一部分里一样，我们就不搞了。

### 修饰器的顺序

在给函数添加修饰器的时候，顺序非常重要。修饰器应当从左到右添加，因为它们从左到右被执行。意思就是说如果前后置都有，我们应该把前置放在前面。在语义上前后置条件的修饰器没有任何区别，所以最好（我觉得）认认真真命名。一个思路就是把所有的修饰器名字前面都挂上`pre`或者`post`，比如`pre_data_is_zero`。在 Solidity 风格指南里没有任何推荐。

### `return` vs `throw`

如果前置和后置条件对不上，怎么样跳出函数其实会很难抉择。鉴于 Solidity 在这个事情上的工作方式，问题就归结于是在修饰器里允许`return`还是直接`throw`。

个人而言，我不知道最佳解决方式是什么。我倾向于每次都抛出。这样的方式把修饰器当作传统的断言处理，且如果断言失败了，就意味着（理论上）我们能保证这次代码执行没有意料之外的后果。我说“理论上”，意思是只要修饰器是好好写的并且添加也正确、执行也正常（比如没有用奇怪的 EVM 漏洞），那就是真的。这是看起来最面向合约的做法了。副作用是`throw`不允许调用者进行任何形式的恢复，因为没有`catch`，但就算你可以，也没有错误类型。但就像我第一部分指出的，在使用修饰器的时候也没办法返回错误代码，所以这个问题也不重要了。

### 分离函数逻辑与条件

有些时候去判断一个条件应该放在在修饰器里（前置或者后置条件）还是应该作为函数体的一部分会比较难。这里有个合约，带着比第一部分的那些更棘手一点的函数：

```Solidity
contract Token {

    // 每个人的余额
    mapping(address => uint) public balances;

    mapping(address => bool) public blacklisted;


    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        balances[msg.sender] = 1000000;
    }


    /*
    转账
    如果调用者在黑名单里，失败。
    如果接收方没在黑名单里且调用者有足够资金，
    资金就会被转到接收方账户里，否则调用方会被放进黑名单里，且账户清零
    */
    function transfer(uint _amount, address _dest) {
        if (blacklisted[msg.sender])
            return;
        if (!blacklisted[_dest] && balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
        else {
            balances[msg.sender] = 0;
            blacklisted[msg.sender] = true;
        }
    }
}
```

在这里用了一大堆条件判断，但这跟之前看到的转账不同，这里的`transfer`可以基于合约状态做两件完全不同的事：一是把资金从一个账户转到另一个，二是清空调用者的账户并拉黑。所以，我们应该怎样解构这个呢？

我们从条件判断来入手。我们有以下三个：

1. `blacklisted[msg.sender] == true`

2. `blacklisted[_dest] == false`

3. `balances[msg.sender] >= _amount`

其中我会认为只有`1`是前置条件。它断言调用者没有被拉黑，且如果被拉黑了那么余下的函数体都不会被执行。另外两个条件不应该被独立出来，因为它们是函数逻辑的一部分。如果它们之一或是两者都失败了，函数也会继续按照期望工作下去。

我们现在就有了一个前置条件和零个后置条件。用上修饰器就会变成这样：

```Solidity
contract Token {

    // 每个人的余额
    mapping(address => uint) public balances;

    mapping(address => bool) public blacklisted;


    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        balances[msg.sender] = 1000000;
    }


    // msg.sender 不能是被拉黑的
    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            return;
        _
    }


    /*
    转账
    如果调用者在黑名单里，失败。
    如果接收方没在黑名单里且调用者有足够资金，
    资金就会被转到接收方账户里，否则调用方会被放进黑名单里，且账户清零
    */
    function transfer(uint _amount, address _dest) not_blacklisted {
        if (!blacklisted[_dest] && balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
        else {
            balances[msg.sender] = 0;
            blacklisted[msg.sender] = true;
        }
    }
}
```

现在假设我们改一改程序，余额不足不会导致调用者被拉黑，只是阻止他们交易（实话说这样更理智一点）。这样我们就该把余额检查也改成前置条件。

（注意文档说明我们也更新一下）

```Solidity
contract Token {

    // 每个人的余额
    mapping(address => uint) public balances;

    mapping(address => bool) public blacklisted;


    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        balances[msg.sender] = 1000000;
    }


    // msg.sender 不能是被拉黑的
    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            return;
        _
    }


    // msg.sender 必须有高于‘x’的余额
    modifier at_least(uint x) {
        if (balances[msg.sender] < x)
            throw;
        _
    }


    /*
    转账
    如果调用者在黑名单里，失败。
    如果接收方没在黑名单里且调用者有足够资金，
    资金就会被转到接收方账户里，否则调用方会被放进黑名单里，且账户清零
    */
    function transfer(uint _amount, address _dest) not_blacklisted at_least(_amount) {
        if (!blacklisted[_dest] && balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
        else {
            balances[msg.sender] = 0;
            blacklisted[msg.sender] = true;
        }
    }
}
```

这个看起来干净多了，但我们还能做更多。我们其实已经分离了调整余额的逻辑与拉黑逻辑，意思就是我们可以把代码放进另一个函数里，再把`at_least`修饰器添加到新函数上。余额调整函数将会是私有的，这样只有`transfer`函数才能接触到它。并且，为了做到更干净，我们也可以分离拉黑代码（`else块里的代码）。

```Solidity
contract Token {

    // 每个人的余额
    mapping (address => uint) public balances;

    mapping (address => bool) public blacklisted;


    // 构造函数 - I wanna be a billionaire so fucking bad~
    function Token() {
        balances[msg.sender] = 1000000;
    }


    // msg.sender 不能是被拉黑的
    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            throw;
        _
    }


    // msg.sender 必须有高于‘x’的余额
    modifier at_least(uint x) {
        if (balances[msg.sender] < x)
            throw;
        _
    }


    // 转账，当调用者余额不足时会失败
    function __transfer(uint _amount, address _dest) private at_least(_amount) {
        balances[msg.sender] -= _amount;
        balances[_dest] += _amount;
    }


    // 拉黑一个账户
    function __blacklist() private {
        balances[msg.sender] = 0;
        blacklisted[msg.sender] = true;
    }


    /*
    转账
    如果调用者在黑名单里，失败。
    如果接收方没在黑名单里且调用者有足够资金，
    资金就会被转到接收方账户里，否则调用方会被放进黑名单里，且账户清零
    */
    function transfer(uint _amount, address _dest) not_blacklisted {
        if(!blacklisted[_dest])
            __transfer(_amount, _dest);
        else
            __blacklist();
    }

}
```

注意这里其实可以直接用三元运算符，但为了可读性我们不用。

而且，注意这里我们摘出了接口部分。`transfer`函数不再有余额检查；余额检查被移到了私有`__transfer`函数上。如果我们仔细点看，我们会发现这样其实更合理了，因为如果目标账户是被拉黑的那么`transfer`函数根本就不会运行转账逻辑。这是个很重要的细节。

### 合约与智能合约

在这个教程里，我们会同时讨论两种不同的“合约”，应当说清楚这两者的区别。

- 这里的`contract`——就面向合约编程而言——是对功能的定义，比如一个函数做什么，和为使函数正常工作必须满足的条件。这是非正式的说法，意思是没有完整的语言支持。

- 在这种情况下，余额标准是（非正式的）交易合约定义的前置条件。

- `at_least`修饰器被用来完成代码中的余额检查。

- 合约中描述的功能在`transfer`函数处执行。这是`smart-contract`的一部分，也就是 EVM (以太坊虚拟机)里跑着的代码。

- `transfer`函数是入口，但它会把一些工作顺延给其他函数。

注意`contract`（在这个语境里）不是智能合约的 JSON ABI，也不是`transfer`函数的 JSON ABI，即使 JSON ABI 属于合约定义的一部分，因为它包含着输入输出数据和一些其他东西。在 Solidity 中没有办法去正式区分这些类型。绝大多数合约涉及的验证工作必须手动完成，但是这样工作还是很好的，就像 Gavin 指出的那样，这样会使代码更安全，更容易验证与测试。

### 结论

这一部分展示了怎么用自定义修饰器创建后置条件，以及怎么样构造更高级的代码。例子的意义是说明面向合约编程不能简单地把条件移到自定义修饰器里，而是需要一定的思考。

如果有人想深入钻研面向合约编程，在[这里](http://se.ethz.ch/~meyer/publications/computer/contract.pdf)可以找到一个挺好的文档。它解释了一些概念，以及应用方式。

下一篇或许会写关于文档和更多的构建代码的东西。这由 Gavin 更新的时间决定。

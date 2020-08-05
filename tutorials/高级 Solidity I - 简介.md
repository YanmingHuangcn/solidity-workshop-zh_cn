# 高级 Solidity I - 简介

这一系列会讲高级 Solidity。关注点会在幕后的事情上；memory 怎么工作，storage 怎么工作，EVM 汇编，然后就是“正式 Solidity”。

这第一部分是一个 Solidity 的高级简介，和 Solidity 代码是怎样在 EVM 里执行的。总体来说这些东西知道了有好处，尤其是对于那些想要在合约里使用内联汇编的人。

读者需要知道以太坊和 Solidity 的基本工作方式。

在以太坊创始人V神 Vitalik Buterin 的白皮书里可以找到一个很好的以太坊协议介绍。

一个合约及账户的工作方式的基本知识框架可以在[以太坊官方开发文档](https://solidity-cn.readthedocs.io/zh/develop/index.html)里找到。作者的 [Solidity 系统教程](https://github.com/androlo/solidity-workshop/blob/master/tutorials/2016-02-17-solidity-systems-I.md#accounts-code-and-storage)里也简单地解释了账户的概念。

记住有些基础原理会随着 Serenity 的发布而改变——尤其是交易和账户——但绝大多数这里讲到的都将会适用。

### 账户，交易，和代码执行

以太坊合约（智能合约）被存储在特殊合约账户里，且被一个（成功的）以该账户为目标的交易触发执行。如果我作为一个用户想要执行一个合约代码，我首先需要确认存储该合约的账户，然后发送一笔交易。交易的参数有：

- 我想要支付的 gas price(`gasPrice`)。

- 可能会消耗的最大 gas 数量(`gas`)。

- 我的账户地址(`from`)。

- 目标账户地址(`to`)。

- nonce(`nonce`)。

- 交易数据(`data`)。

或者，也可以在调用者这边创建交易对象本身，本地签名，再传入已签名字节里。上面所使用的参数是在进行 RPC 调用时使用的，然后客户端就会为你创建（并签名）一个交易对象。

不论如何，如果我的交易是有效的，以太坊客户端就会把交易数据和当前区块信息以及一些其他的东西放进一个 [context](https://solidity-cn.readthedocs.io/zh/develop/units-and-global-variables.html#id3) 里。接下来它会把合约账户的代码和这个 context 传入一个新的 EVM 实例，执行它。

**例子**

找一个[在线 Solidity 编译器](https://ethereum.github.io/browser-solidity/)，清理所有东西（或者开个新 tab)，然后把这个代码贴上去：

```Solidity
contract C {


    function add(int a, int b) constant returns (int sum) {
        sum = a + b;
    }
}
```

*提示：如果你得出了不同的结果，把 solc 版本改为0.2.2-2016-03-02-32f3a653*

贴上代码之后，你应该可以看见右边列出的合约。

点击`toggle details`会展示出更多更复杂的合约数据，比如运行时字节码，函数签名和 EVM 汇编。在关注不同部分的细节之前，我们先来看看以太坊合约的解剖学知识。

### 一个以太坊合约

以太坊合约主要包含两部分——初始和主体部分。

**初始**

先是初始部分。它包含构造器和其他初始化逻辑（比如字段初始化）。在合约生命周期里初始部分只执行一次，就是在合约被上传到链上时，比如当一个`create`指令成功创建了一个合约的新实例。

**主体**

第二部分是`runtime`部分，或者说`body`。这是合约账户里实际存储的代码。

**创建合约**

有两种方法可以创建新合约，或者从外部账户发送一个新合约交易到链上，或者用`new`从一个已存在的合约创建。

创建合约的交易由向数据字段传入编译后的字节码（所有）并置空接收者地址完成。你可能用过 web3 创建合约，也见过了你得调用合约工厂对象的`new`来创建一个新合约，并在交易对象里传入字节码。这就是原因。

这个初始/主体系统背后的机制很简单。初始只是一些以 RETURN 结尾的 VM 指令。返回的值就是运行时部分的字节码。这个可以用古老的 LLL 语言来说明。这里有个 LLL 写的基础合约：

```LLL
{
    [[0x0]] (caller)

    (return 0x0 (lll {
        (when (= (caller) @@0) (suicide (caller)))
    } 0x0))
}
```

这合约本质上就是 LLL 写的`init`部分。它使用`[[A]] B`（`(SSTORE A B)`的缩写）来把调用者的地址写入到存储地址`0x0`。然后通过返回运行时部分结束。

这个返回看起来很怪，但分解它我们就能发现它是`(return 0x0 x)`这个结构，意思就是“从内存返回`x`字节，从`0x0`开始”。

# Solidity 系统 I - 基础

这是一个关于怎样创建模块化智能合约系统、怎样以可靠的方式替换合约的新系列教程。这些，再加上一些其他的生命周期管理，就是这个DAO框架的主要任务/内容。

读者应当具有对以太坊和 Solidity 的基本认知。

### 简介

一个DApp里的绝大多数合约会在某个点变得过时，并会需要更新。正如其他应用程序。可能是因为新功能必须被添加，找到了一个bug，或者是一个更好的、优化后的版本被做出来了。更新当然可能会造成问题，所以需要小心处理。必须确保的某些事情是：

    + 新合约按照期望工作。
    + 替换期间发起的请求被成功处理。
    + 替换合约对系统的其他部分没有负面影响。

第一点是必须检查的，毕竟这关系到这次更新究竟是否可行。但由于以太坊账户的工作原理，并非总是这样。

### 账户，代码和存储

以太坊合约的一个非常重要的性质是，在一个合约被上传到了链上之后，代码再也无法改变。合约被存储在特殊账户对象里，账户对象还包含了合同（字节）码和数据库之类东西的引用。这个数据库是键-值存储，也被叫做 “storage” ，并且这就是诸如合同字段值存储的地方。

![原文Github：androlo/solidity-workshop](https://github.com/androlo/solidity-workshop/blob/master/images/ext-vs-contract-account.png)

当合约被发布到链上时，发生的第一件事就是一个新账户被创建。之后合约代码被加载到一个虚拟机中，该 VM 运行构造函数部分，初始化字段，然后将合约的运行时部分（代码体）添加到账户。在合约被创建后没有任何办法再去修改代码，也没有代码之外的方式去更新数据库。

我们来看看这个非常简单的合约例子：

```Solidity
contract Data {

    uint public data;

    function addData(uint data_) {
        if (msg.sender == 0x692a70d2e424a56d2c6c27aa97d1a86395877b3a)
            data = data_;
    }
}
```

这个合约允许用户添加和读取一个无符号整数。唯一被允许添加数据的是地址为`0x692a...`的账户。这个地址是一个十六进制的字面值，编译合约时会被添加到字节码中。

但要是我们之后想更换这个地址呢？

事实上这就不可能。如果我们想要实现这样的功能，我们得提前准备好合约。一个简单的、使其更有弹性的方法，就是把当前所有者的地址放在一个storage变量里，并提供一个改变它的可能。

```Solidity
contract DataOwnerSettabl {

    uint public data;

    address public owner = msg.sender;


    function addData(uint data_) {
        if (msg.sender == owner)
            data = data_;
    }


    function setOwner(address owner_) {
        if (msg.sender == owner)
            owner = owner_;
    }
}
```

这个合约有一个`owner`字段（映射到存储）。这个字段被创建合约的账户地址初始化，并且之后能被当前所有者通过`setOwner`修改。`addData`里的防护还是一样的；唯一不同的是所有者地址没有被硬编码。

### 委托

但如果可修改的所有者还不够咋办？如果我们想不止能更新所有者地址，还要能升级整个验证过程？

确实有方法来替换代码，就是通过连接数个合约调用来形成一个调用链。一个合约`C`能调用另一个合约`D`，意味着一个到`C`的交易不仅能执行`C`的代码，还能执行`D`的。此外，我们还能让`D`的地址可以修改，意味着可以修改`C`指向的`D`的实例。举个银行合约调用不同合约进行认证的栗子。

![原文Github：androlo/solidity-workshop](https://raw.githubusercontent.com/androlo/solidity-workshop/master/images/bank-auth-sequence.png)

在这个例子里，每次`deposit`被`bank`调用，就去调用`Authenticator`，以调用者的地址作为参数，并检查返回值，以决定是否完成存款。`Authenticator`只是另一个可替换的合约，意思是这个调用可以被指向其他合约。

我们现在来更新一下`data`合约使其这样工作，从把账户验证移到另一个合约开始。

```Solidity
contract AccountValidator {

    address public owner = msg.sender;


    function validate(address addr) constant returns (bool) {
        return addr == owner;
    }


    function setOwner(address owner_) {
        if (msg.sender == owner)
            owner = owner_
    }
}


contract DataExtrernalValidation {
    uint public data;

    AccountValidator _validator;


    function DataExtrernalValidation(address validator) {
        _validator = AccountValidator(validator);
    }


    function addData(uint data_) {
        if (_validator.validate(msg.sender))
            data = data_;
    }


    function setValidator(address validator) {
        if (_validator.validate(msg.sender))
            _validator = AccountValidator(validator);
    }
}
```

想要在链上使用这个合约，我们先得创建一个`AccountValidator`合约，再创建一个`DataExtrernalValidation`合约并通过构造函数注入 validator 的地址。当有人想要写入数据到`data`它会调用当前 validator 的`validate`方法以实现所有者检查。

这挺好的，因为现在有了替换所有者检查合约的可能性。并且由于`AccountValidator`是个独立的合约，我们可以使用这个实例去完成更多其他合约的身份验证。

不过还剩下一件事。我们仍然不能替换代码。我们做的所有只是将验证代码移到了合约之外。除 data 合约外，`AccountValidator`合约的代码再无法更改。幸运的是，Solidity 提供了一个非常简单且强大的解决方案——抽象。

### 抽象方法

使用抽象方法，validator 合约可以被改成这样：

```Solidity
contract AccountValidator {
    function validate(address addr) constant returns (bool);
}


contract SingleAccountValidator is AccountValidator {

    address public owner = msg.sender;


    function validate(address addr) constant returns (bool) {
        return addr == owner;
    }


    function setOwner(address owner_) {
        if (msg.sender == owner)
            owner = owner_;
    }
}
```

按照这些合约，data 合约不再使用具体的验证器合约，而是使用一个抽象（接口）表示，意味着我们可以选择要提供的实现。这点与 Java 和 C++ 之类的语言类似。假设我们想要允许更多的所有者账户，我们可以用这个合约来提供这个功能：

```Solidity
contract MultiAccountValidator is AccountValidator {

    mapping(address => bool) public owners;


    function MultiAccountValidator() {
        owners[msg.sender] = true;
    }


    function validate(address addr) constant returns (bool) {
        return owners[addr];
    }


    function addOwner(address addr) {
        if (owners[msg.sender])
            owners[addr] = true;
    }
}
```

最后，值得注意的是，你可以传入一个不是`AccountValidator`的合约。在你将地址转化为合约对象时没有任何类型检查，所以只有当合约被调用时才会显示。实际上只要合约有必需的方法，调用就能成功——即使它并未继承`AccountValidator`。

当然，不推荐这样使用合约。

### 总结

合适的委托/委派/授权/代理是智能合约系统的重要组成部分，因为这样合约可以被替换。但有些东西需要注意：

    + 委托需要的是类型不安全的转换，意思是在设置、更改合约时要分外小心。
    + 一个系统里合约越多越难管理，并且小型系统的策略不一定适用于中型大型系统。
    + 模块化是有代价的，因为这需要更多的代码、变量和调用。在 gas 限制非常严格的公链上，即使是小型模块化系统的部署和运行都可能会很艰难。通常来说，涉及可扩展性和效率时，我会倾向于选择可扩展性。一个过度模块化的系统里，大且昂贵的合约毕竟是可以改进和替换的，而一个写死的合约则不然。

Happy smart-contracting!

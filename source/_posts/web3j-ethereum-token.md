title: 使用 Web3J 发个以太坊 ERC20 Token
tags:
  - Ethereum
  - 以太坊
  - 区块链
  - blockchain
  - keystore
  - web3j
  - solidity
categories:
  - Ethereum(以太坊)
  - blockchain(区块链)
  - Smart Contract(智能合约)
author: Steven Hu
date: 2018-04-06 22:39:00
---
## 关于 ERC20 规范

ERC20 为 Ethereum 上的 Token 合约标准规范，遵守该规范的 Token 合约可以被各种以太坊钱包、以及相关的平台和项目支持，如在 etherscan 上可以查看遵守 ERC20 规范的 Token 信息和交易记录。

<!--more-->

如下为 ERC20 Token 标准接口：

```solidity
 // ----------------------------------------------------------------------------
 // ERC20 Token Standard Interface
 // https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20-token-standard.md
 // ----------------------------------------------------------------------------
contract ERC20 {
      function name() constant returns (string name)
      function symbol() constant returns (string symbol)
      function decimals() constant returns (uint8 decimals)
      function totalSupply() constant returns (uint totalSupply);
      function balanceOf(address _owner) constant returns (uint balance);
      function transfer(address _to, uint _value) returns (bool success);
      function transferFrom(address _from, address _to, uint _value) returns (bool success);
      function approve(address _spender, uint _value) returns (bool success);
      function allowance(address _owner, address _spender) constant returns (uint remaining);
      event Transfer(address indexed _from, address indexed _to, uint _value);
      event Approval(address indexed _owner, address indexed _spender, uint _value);
}
```

其中各个接口方法：

- **name**：返回 ERC20 token 的名称，如：Steven Ocean Token；
- **symbol**：返回 ERC20 token 的简称，如：SOT；
- **decimals**：返回 token 使用小数点的后几位，如设置为3，表示支持 0.001；
- **totalSupply**：总的 token 供应量；
- **balanceOf**：获取指定账户(address)的余额（token 拥有量）；
- **transfer**：token 转账，从 token 合约的调用者账户(address)上转移（输入参数）_value 的 token 数到指定的账户(address) _to 上，并且触发 Transfer 事件；
- **transferFrom**：从账户地址 _from 发送数量为 _value 的 token 到账户地址 _to，且触发Transfer 事件；transferFrom 方法用于允许 Contract 代理某个账号来转移 token，条件是from 账户必须经过了 approve；
- **approve**：允许 _spender 多次提取发起账户中最多为 _value 个 token，如果再次调用此函数，它将以本次 _value 覆盖当前的余量；
- **allowance**：返回仍然允许 _spender 从 _owner 账户中提取的 token 余量；

如下两个为 event，会产生对应的 event log：

- **Transfer**：该事件是在 token 发生转移时触发的，包括 0 个 token 转移也会触发；
- **Approve**：该事件是在成功调用 approve 方法后触发；

## 使用 Solidity 编写 ERC20 Token

支持在以太坊上编写智能合约的语言主要有 Solidity、Serpent、LLL 和 Mutan，其中 Solidity 语法类似 Javascript，也是目前最主要的语言，Solidity 是静态类型的面向对象的编程语言，用其编写的智能合约需要通过 solc 编译器或者 IDE 环境编译成 EVM 字节码格式才能在 EVM 中执行。当编译好的合约发送到以太坊网络之后，就可以通过 web3.js 或者 web3.js API 来调用了，从而构建一个与之交互的应用。

我们下面使用在线 IDE Remix 来编写我们的智能合约：简单的发行 31415926 个 SOT tokens，支持简单的合约转账功能。

> Remix 是以太坊官方推荐和维护的 IDE 环境，支持[浏览器在线](https://remix.ethereum.org)开发、调试、编译，也支持本地部署该 IDE 环境。

下图为使用 Remix 开发 SOT 合约的环境：

{% asset_img remix-contract-1.jpg %}

在 Remix 的 Editor 编辑器主要（通过 tab）显示了正在打开的一个或多个合约源码文件，Remix 也会自动编译合约代码，如果有编译错误，会在左边的 **compile** tab 页面显示。

> [remix 的官方文档](http://remix.readthedocs.io/en/latest/) 详细介绍了 remix 的使用。

具体的合约代码如下：

```solidity
pragma solidity 0.4.21;

contract Token {
    uint256 public totalSupply;
    function balanceOf(address _owner) public constant returns (uint256 balance);
    function transfer(address _to, uint256 _value) public returns (bool success);
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success);
    function approve(address _spender, uint256 _value) public returns (bool success);
    function allowance(address _owner, address _spender) public constant returns (uint256 remaining);
    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}

contract StandardToken is Token {
    function transfer(address _to, uint256 _value) public returns (bool success) {
        if (balances[msg.sender] >= _value && _value > 0) {
            balances[msg.sender] -= _value;
            balances[_to] += _value;
            emit Transfer(msg.sender, _to, _value);
            return true;
        } else {
            return false;
        }
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        if (balances[_to] + _value < balances[_to]) revert(); // Check for overflows
        if (balances[_from] >= _value && allowed[_from][msg.sender] >= _value && _value > 0) {
            balances[_to] += _value;
            balances[_from] -= _value;
            allowed[_from][msg.sender] -= _value;
            emit Transfer(_from, _to, _value);
            return true;
        } else {
            return false;
        }
    }

    function balanceOf(address _owner) public constant returns (uint256 balance) {
        return balances[_owner];
    }

    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowed[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function allowance(address _owner, address _spender) public constant returns (uint256 remaining) {
        return allowed[_owner][_spender];
    }

    mapping (address => uint256) balances;
    mapping (address => mapping (address => uint256)) allowed;
}

contract SOT is StandardToken {
    string public constant name = "Steven Ocean Token";
    string public constant symbol = "SOT";
    uint256 public constant decimals = 18;
    uint256 public constant total = 31415926 * 10**decimals;

    function SOT() public {
        totalSupply = total;
        balances[msg.sender] = total;
    }

    function () public payable {
        revert();
    }
}
```

上述代码支持基本的合约转账和合约代理转账功能，主要围绕两个数据成员 **balances** 和 **allowed** 来实现，其中：

- **balance**：主要使用 map 结构缓存账号的 token 余额；
- **allowed**：采用二级 map 结构缓存 owner（一级 map key）授权给哪些 spender（二级 map key）可提取的 token 数 value；

## 编译和编码合约为 Java 代码

因为我们希望能够通过在 Java 代码中实现合约的部署和转账等功能，因为我们需要将合约转换成 Java 代码，这里采用 [web3j](https://github.com/web3j/web3j) 来转换。

web3j 转换需要提供合约的 .abi 和 .bin 文件，我们先用 solc 编译器来生成 sot.abi 和 sot.bin 文件，如下命令：

```shell
➜  sot solc sot.sol --abi --bin --optimize -o ./
➜  sot ll
total 48
-rw-r--r--  1 steven  staff   2.4K  4  7 16:30 SOT.abi
-rw-r--r--  1 steven  staff   2.9K  4  7 16:30 SOT.bin
-rw-r--r--  1 steven  staff   1.7K  4  7 16:30 StandardToken.abi
-rw-r--r--  1 steven  staff   2.1K  4  7 16:30 StandardToken.bin
-rw-r--r--  1 steven  staff   1.7K  4  7 16:30 Token.abi
-rw-r--r--  1 steven  staff     0B  4  7 16:30 Token.bin
-rw-r--r--@ 1 steven  staff   2.6K  4  7 16:29 sot.sol
➜  sot
```

可以看到针对每个 contract 类都生成了对应的 .abi 和 .bin 文件。

Remix IDE 中也可以通过查看 **details** 来获取对应的 abi 和 bin 文件。如下图：

{% asset_img remix-contract-2.jpg %}

接下来使用 web3j 将合约转换为 Java 文件：

```shell
➜  sot web3j solidity generate --javaTypes SOT.bin SOT.abi -o ./src/main/java -p io.github.stevenocean.contracts

              _      _____ _     _
             | |    |____ (_)   (_)
__      _____| |__      / /_     _   ___
\ \ /\ / / _ \ '_ \     \ \ |   | | / _ \
 \ V  V /  __/ |_) |.___/ / | _ | || (_) |
  \_/\_/ \___|_.__/ \____/| |(_)|_| \___/
                         _/ |
                        |__/

Generating io.github.stevenocean.contracts.SOT ... File written to ./src/main/java

➜  sot tree src/main/java
src/main/java
└── io
    └── github
        └── stevenocean
            └── contracts
                └── SOT.java

4 directories, 1 file
➜  sot
```

生成的 SOT.java 文件中生成了一个派生于 **Contract** 类的 **SOT** 合约类，该类中实现了 ERC20 token 规范的那些方法，代码摘略如下：

```java
public class SOT extends Contract {
    private static final String BINARY = "606060......10029";

    protected SOT(String contractAddress, Web3j web3j, Credentials credentials, BigInteger gasPrice, BigInteger gasLimit) {
        super(BINARY, contractAddress, web3j, credentials, gasPrice, gasLimit);
    }

    public RemoteCall<BigInteger> totalSupply() {
        final Function function = new Function("totalSupply", 
                Arrays.<Type>asList(), 
                Arrays.<TypeReference<?>>asList(new TypeReference<Uint256>() {}));
        return executeRemoteCallSingleValueReturn(function, BigInteger.class);
    }

    public RemoteCall<BigInteger> total() {
        final Function function = new Function("total", 
                Arrays.<Type>asList(), 
                Arrays.<TypeReference<?>>asList(new TypeReference<Uint256>() {}));
        return executeRemoteCallSingleValueReturn(function, BigInteger.class);
    }

    public RemoteCall<BigInteger> balanceOf(String _owner) {
        final Function function = new Function("balanceOf", 
                Arrays.<Type>asList(new org.web3j.abi.datatypes.Address(_owner)), 
                Arrays.<TypeReference<?>>asList(new TypeReference<Uint256>() {}));
        return executeRemoteCallSingleValueReturn(function, BigInteger.class);
    }

    public RemoteCall<TransactionReceipt> transfer(String _to, BigInteger _value) {
        final Function function = new Function(
                "transfer", 
                Arrays.<Type>asList(new org.web3j.abi.datatypes.Address(_to), 
                new org.web3j.abi.datatypes.generated.Uint256(_value)), 
                Collections.<TypeReference<?>>emptyList());
        return executeRemoteCallTransaction(function);
    }

    public static RemoteCall<SOT> deploy(Web3j web3j, Credentials credentials, BigInteger gasPrice, BigInteger gasLimit) {
        return deployRemoteCall(SOT.class, web3j, credentials, gasPrice, gasLimit, BINARY, "");
    }

    public static SOT load(String contractAddress, Web3j web3j, Credentials credentials, BigInteger gasPrice, BigInteger gasLimit) {
        return new SOT(contractAddress, web3j, credentials, gasPrice, gasLimit);
    }
        
    // ...

    public static class TransferEventResponse {
        public Log log;
        public String _from;
        public String _to;
        public BigInteger _value;
    }

    public static class ApprovalEventResponse {
        public Log log;
        public String _owner;
        public String _spender;
        public BigInteger _value;
    }
}
```

后续可以将生成的 SOT.java 文件导入至 Java 项目中。

> - [solidity 编译器的安装参考](http://solidity.readthedocs.io/en/v0.4.21/installing-solidity.html)
> - [web3j 安装参考](https://web3j.readthedocs.io/en/latest/command_line.html)

## 使用 Web3j 部署合约

部署合约之前先要有个自己的钱包账号的，这个账号可以用 web3j 的 WalletUtils.generateLightNewWalletFile 来创建，如下：

```java
/// password: 提供一个生成 keystore 的密码
/// destinationDirectory: 存放 keystore 文件的路径
public static String generateLightNewWalletFile(String password, File destinationDirectory)
            throws NoSuchAlgorithmException, NoSuchProviderException,
            InvalidAlgorithmParameterException, CipherException, IOException {
        return generateNewWalletFile(password, destinationDirectory, false);
    }
```

刚创建好的钱包账号中的余额为 0，而部署合约是需要消耗一定的 ether 的，因此我们得先申请一点 ether，当然我们只能在测试环境下申请，在 rinkeby testnet 中，因为采用的是 PoA(clique) 共识机制，可以通过 [**faucet**](https://faucet.rinkeby.io/) 提交如下三个支持的社交媒体的帖子URL，而对应的帖子内容中包括你需要申请 ether 的账号地址：

- A public tweet on Twitter
- A public Facebook post
- A public Google+ link

具体的申请内容和方式请参考 [How to get on Rinkeby Testnet in less than 10 minutes](https://gist.github.com/cryptogoth/10a98e8078cfd69f7ca892ddbdcf26bc) 的 Step 4。

申请成功之后，我们的钱包账号中就可以查到余额了，如下为在 [etherscan rinkeby testnet](https://rinkeby.etherscan.io/address/0x7bdf7b545532026aa0aed61391eef6d3b9f429a2) 中查看到的账号信息：

{% asset_img address-1.jpg %}

其中 **Transactions** Tab 中第一条交易记录就是在 **faucet** 申请的 ether 的账号。

好了，ether 来了，开始使用 web3j 部署 SOT 合约，如下代码：

```java
SOT contract = SOT.deploy(
        web3, finalCredentials,
        ManagedTransaction.GAS_PRICE,
        Contract.GAS_LIMIT).send();
String contractAddress = contract.getContractAddress();
```

其中 **web3** 为使用 Web3jFactory.build 构建的实例，如下代码为连接到 rinkeby 测试网络：

```java
final Web3j web3 = Web3jFactory.build(
        new HttpService("https://rinkeby.infura.io/xxxxsyt4bzKIGsctxxxx"));
```

**finalCredentials** 为通过 WalletUtils.loadCredentials 从本地 keystore 加载的凭证，如下代码：

```java
/// password: 为 keystore 密码
/// source: keystore 文件路径
public static Credentials loadCredentials(String password, String source)
        throws IOException, CipherException {
    return loadCredentials(password, new File(source));
}
```

**contract.getContractAddress()** 在部署成功 SOT 合约之后返回对应的合约地址。

部署成功之后，我们可以查看到对应的合约信息，如下图：

{% asset_img contract-1.jpg %}

该合约信息页面中显示了 合约创建者(Contract Creator)，ERC20 Token Contract 名称为 **Steven Ocean Token(SOT)**，以及交易列表中显示了关联的首笔交易（**To** 显示的为 **Contract Creation**），[交易信息](https://rinkeby.etherscan.io/tx/0x01a0bc34a021cf18299796550b7633afb49d2bc0c6a9c3fa62d74921e772aaaf) 如下图：

{% asset_img contract-2.jpg %}

在交易信息的 Input Data 中其实承载的是 SOT 合约的 BIN 代码。

另外，可以查看 [SOT token 页面](https://rinkeby.etherscan.io/token/0xb43149e6bfef554b282671cf249e2d43e57eb4a3#balances)，如下图：

{% asset_img contract-3.jpg %}

其中显示了 SOT token 的很多信息，包括如下几个关键信息：

- **Total Supply**：token 总的供应量，这里为我们发行了 31415926 个 SOT；
- **ERC20 Contract**：展示了 SOT token 对应的合约地址，即我们在上面看到的那个截图；
- **Token Holders**：Token 持有者，下面的列表展示了持有 SOT 的账户地址列表；

> 注：上图中的 Token Transfers 中的记录是在下一步（合约转账）中完成之后出现的。

## 合约转账 - 给好基友转点币

我是 SOT token 的创建者，我给自己发行了 31415926 个 token，下面给好基友转点过去。继续使用 web3j 如下代码：

```java
// 调用 SOT.transfer 方法
Function function = new Function(
        "transfer",
        Arrays.<Type>asList(new Address(friendAddress),
                new Uint256(new BigInteger("128000000000000000000"))),
        Collections.<TypeReference<?>>emptyList());
String encodedFunction = FunctionEncoder.encode(function);

// 创建 tx 管理器，并通过 txManager 来发起合约转账
RawTransactionManager txManager = new RawTransactionManager(web3, finalCredentials);
EthSendTransaction transactionResponse = txManager.sendTransaction(
        ManagedTransaction.GAS_PRICE, Contract.GAS_LIMIT, 
        contractAddress, encodedFunction, BigInteger.ZERO);

// 获取 TxHash
String transactionHash = transactionResponse.getTransactionHash();
```

调用成功后，会提交到以太坊网络中，在交易被确认之前，为 pending 状态，如下图：

{% asset_img contract-transfer-1.jpg %}

在交易最终被确认，并被区块打包之后，如下图：

{%asset_img contract-transfer-2.jpg %}

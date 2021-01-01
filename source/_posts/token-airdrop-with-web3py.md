title: 使用 web3.py 实现 ERC20 Token 空投
tags:
  - Ethereum
  - 以太坊
  - 区块链
  - blockchain
  - airdrop
  - 合约账户
  - Contract
  - ERC20
categories:
  - Smart Contract(智能合约)
author: Steven Hu
date: 2018-07-08 14:58:00
---

## 前言

随着一些交易所搞的上币排名的活动开展，也带来了空投的火爆，当然也给以太坊整个网络带来了很大的考验（极其拥堵、交易费水涨船高），本篇文章还是从技术上来看看实现空投的一种通用方案。

关于空投的实现需要提前了解一下 ERC20 标准中的下面几个函数：

- **approve(_spender, value)**：该函数表示允许 _spender 多次取回调用此函数的的账户中的 token，最高为 value 个 token，后面再次调用此函数会覆盖之前设置的余额；
- **allowance(_owner, _spender)**：返回 _spender 仍然可以从 _owner 中提取的 token 余额；
- **transferFrom(_from, _to, _value)**: 该方法主要用于允许 contract 来自动执行转账流程，并代表所有者发送 _value 个 token；

> - ERC20 标准的基础知识，可以看下这篇 **[Understanding ERC-20 token contracts](https://medium.com/@jgm.orinoco/understanding-erc-20-token-contracts-a809a7310aa5)**
> - 实现和部署自己的符合 ERC20 Token 标准的文章，可以参考 **[使用 Web3J 发个以太坊 ERC20 Token](https://stevenocean.github.io/2018/04/06/web3j-ethereum-token.html)**

## 实现流程

{% asset_img token-airdrop-with-web3py-1.png %}

<!--more-->

流程说明：

1. 账户 **A** 创建和部署 ERC20 Token **SOT**；
2. 账户 **B** 创建和部署空投合约 **Airdrop**；
3. 需要空投的账户 **C**（如运营人员账户）授信合约账户 **Airdrop** 使用其账户上拥有的一定数额的 **SOT** token；
4. 授信成功后，通过调用 **Airdrop** 的空投方法 **drop** 实现给账户 **a,b,c** 发送 **SOT** token；
5. 在 **Airdrop** 的方法 **drop** 内部通过调用 **SOT** 的 **transferFrom** 方法实现转账；

## 环境准备

- **python3**
- **[web3.py](https://github.com/ethereum/web3.py)**
- **[solc](https://github.com/ethereum/solidity)**
- 测试网络 Ropsten（Rinkeby 也可以）已同步

## 用于部署合约的程序

如下为使用 web3.py 部署合约（deploy_contract.py）的代码：

```python
#!/usr/bin/python3
# -*- coding:utf-8 -*-

import sys
import json

from web3 import Web3
from web3 import HTTPProvider
from solc import compile_source


def load_contract_source_code(source_code_file):
    contract_source_code = ''
    with open(source_code_file) as f:
        for line in f:
            contract_source_code += line + '\n'
    return contract_source_code


def main(argv):
    """deploy_contract <contract name> <contract code path> <account address> <gas price, unit: Gwei>
    """
    code = load_contract_source_code(argv[2])
    compiled_sol = compile_source(code)
    contract_interface = compiled_sol['<stdin>:' + argv[1]]
    account_address = argv[3]
    gas_price = int(argv[4]) * 10 ** 9

    w3 = Web3(HTTPProvider("http://127.0.0.1:8545"))
    contract = w3.eth.contract(abi=contract_interface['abi'], bytecode=contract_interface['bin'])
    tx_param = {
        'from': account_address,
        'gasPrice': gas_price,
    }
    tx_hash = contract.deploy(transaction=tx_param)
    print("deploying, tx_hash {}".format(Web3.toHex(tx_hash)))

    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)
    print("receipt: ", tx_receipt)
```

## 编写合约 **Airdrop**

这里的合约代码实现逻辑是先将

```solidity
pragma solidity ^0.4.22;

contract ERC20Token {
    uint256 public totalSupply;
    function balanceOf(address _owner) public constant returns (uint256 balance);
    function transfer(address _to, uint256 _value) public returns (bool success);
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success);
    function approve(address _spender, uint256 _value) public returns (bool success);
    function allowance(address _owner, address _spender) public constant returns (uint256 remaining);
    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}

contract FutureEdgeAirdrop {
    function drop(address tokenAddr, address[] dests, uint256 value) public payable {
        uint256 valuePerCount = value / dests.length；
        for (uint i = 0; i < dests.length; i++) {
            ERC20Token(tokenAddr).transferFrom(msg.sender, dests[i], valuePerCount);
        }
    }
}
```

## 部署合约 **Airdrop**

使用之前的 deploy_contract 来部署合约，提供合约名称、合约代码文件、以及部署合约的账号名和 GasPrice，如下：

```shell
$ deploy FutureEdgeAirdrop FutureEdgeAirdrop.sol 0xEcD3E082Df97E509D83c71e660b9Fd5EfCB92345 120
account: 0xEcD3E082Df97E509D83c71e660b9Fd5EfCB92345
deploying, tx_hash is 0x3852944e009b1acbe556b78516dbfdadd1080429cf723f6f71c78e669b299d42
tx_receipt: {"blockHash": "0x9e2c321dbfaa1236adbd0b5eef2ee14577c951573c6ca665758ab7f1dabe7ea7", "contractAddress": "0xd3D476033145a976fF0304de7CAD767Ba2DBa1F4", "cumulativeGasUsed": 209681, "from": "0xecd3e082df97e509d83c71e660b9fd5efcb92345", "status": 1, "transactionHash": "0x5387e031881b2f3a49b917ed3bdeb76691c9d845d9f3f72ad00a1530b3a4347e", "to": null}
```

按照其中的 transaction hash 查询 etherscan 如下：

{% asset_img etherscan-airdrop-tx.jpg %}

新创建的 Airdrop 合约地址为: 0xd3D476033145a976fF0304de7CAD767Ba2DBa1F4

## 授信合约账号

账号 **C**（我们测试环境使用的账号是 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC）授权给 **Airdrop** 合约账号可以提取其中的 6666 个 **SOT** token（地址为: 0x814f06FFaD575cb9F13A62C835bc505394CB4678），代码实现如下：

```python
def approve(account_addr, token_addr, spender, token_count, gas_price):
    w3 = Web3(HTTPProvider("http://127.0.0.1:8545"))
    w3.defaultAccount = account_addr
    token = w3.eth.contract(address=token_addr, abi=erc20_abi)
    tx_hash = token.functions.approve(spender, token_count).transact({
        "from": account_addr,
        "gasPrice": gas_price * 10 ** 9
    })
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)
    print("receipt: ", tx_receipt)
```

针对上面的需求，相应的参数分别为：

- **account_addr**: 这里为 **C**；
- **token_addr**： **SOT** 的合约地址，其中用于生成相应的 contract 类实例；
- **spender**：这里被授权的是 **Airdrop** 合约地址；
- **token_count**：被授权使用的 token 数；

调用执行后如下：

```shell
$ acctexplorer approve 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC 0x814f06FFaD575cb9F13A62C835bc505394CB4678 0xd3D476033145a976fF0304de7CAD767Ba2DBa1F4 6666000000000000000000 30
Commit approve, txhash 0x5800e276f887c7a0f5d73fb9db31f95be7bb609270ba49c29eb5473a51fe61ce, waiting receipt ...
------ approve ------
account: 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC
token: 0x814f06FFaD575cb9F13A62C835bc505394CB4678(StevenOceanToken) SOT
spender: 0xd3D476033145a976fF0304de7CAD767Ba2DBa1F4
count: 6666000000000000000000
----- receipt -----
{
  "blockHash": "0xe9f6f93965cc7640e92e9fc83a81e446e53ba89219f2b4bdf6c749a212179e8e",
  "contractAddress": null,
  "cumulativeGasUsed": 120369,
  "from": "0xefd0367d44bf23159599fc525e797c9b79b6deec",
  "status": 1,
  "transactionHash": "0xccdc1f637e92d34e39282a0358cdea4810c15f5036ea2cd72f749a3279c6cfc3",
  "to": "0x814f06ffad575cb9f13a62c835bc505394cb4678"
}
--------------------
```

通过 allowance 来查看被授权的信息，代码实现如下：

```python
def allowance(account_addr, token_addr, spender):
    w3 = Web3(HTTPProvider("http://127.0.0.1:8545"))
    token_instance = w3.eth.contract(address=token_addr, abi=erc20_abi, ContractFactoryClass=ConciseContract)
    allow_token_count = token_instance.allowance(account_addr, spender)
    print("------ allowance ------")
    print("token: {}({}) {}".format(token_addr, token_instance.name(), token_instance.symbol()))
    print("owner: {}".format(account_addr))
    print("spender: {}".format(spender))
    print("allowence count: {}".format(allow_token_count))
    print("--------------------")
```

这里传入 **C** 的账户地址(**account_addr**)，**SOT** 的合约地址(**token_addr**)，以及被授信的 **Airdrop** 合约地址(**spender**) 之后，结果如下：

```python
$ acctexplorer allowance 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC 0x814f06FFaD575cb9F13A62C835bc505394CB4678 0xd3D476033145a976fF0304de7CAD767Ba2DBa1F4
------ allowance ------
token: 0x814f06FFaD575cb9F13A62C835bc505394CB4678(StevenOceanToken) SOT
owner: 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC
spender: 0xd3D476033145a976fF0304de7CAD767Ba2DBa1F4
allowence count: 6666000000000000000000
--------------------
```

## 执行空投

上面账号 **C** 授信成功之后，可以通过 **C** 来执行 **Airdrop** 合约的 **drop** 方法来实现给账号 **a, b, c** 发送 **SOT** token，如下代码：

```python
def airdrop(account_addr, token_addr, token_count, gas_price, dest_account_addrs):
    w3 = Web3(HTTPProvider("http://127.0.0.1:8545"))
    contract_instance = w3.eth.contract(address=AIRDROP_ADDR, abi=AIRDROP_ABI)
    tx_hash = contract_instance.functions.drop(token_addr, dest_account_addrs, token_count).transact({
        "from": account_addr,
        "gasPrice": gas_price * 10 ** 9
    })
    print("Commit airdrop, txhash {}, waiting receipt ...".format(Web3.toHex(tx_hash)))
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)
    print("receipt: ", tx_receipt)
```

> 代码中的 **AIRDROP_ADDR** 为 Airdrop 的合约地址，**AIRDROP_ABI** 为 Airdrop 合约的 ABI（具体生成方法可以参考之前的文章）；

这里通过 contract_instance 中的 functions.drop 来访问相应合约中的 drop 方法，这里演示了同时给两个账号各空投 256 个 SOT token（0x19f87671E1BFd859B203816e54fA19F464c9A155, 0x37a6e30c37fbe6af220a99a029a60e5848745AE9）。

```shell
$ acctexplorer airdrop 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC 0x814f06FFaD575cb9F13A62C835bc505394CB4678 789000000000000000000 30 0x19f87671E1BFd859B203816e54fA19F464c9A155 0x37a6e30c37fbe6af220a99a029a60e5848745AE9
Commit airdrop, txhash 0x178eb4ad81d3160433b36cfc1357d50f2ef0ce758b4a7026d3cc72f42dd60dee, waiting receipt ...
------ airdrop ------
account: 0xEfd0367D44bf23159599Fc525e797c9b79B6dEEC
token: 0x814f06FFaD575cb9F13A62C835bc505394CB4678(StevenOceanToken) SOT
dest account: 0x37a6e30c37fbe6af220a99a029a60e5848745AE9
count: 512000000000000000000
----- receipt -----
{
  "blockHash": "0x6e452edabcaf7e2e33433e4e243922fcb59dda5e6caa2852de0a2f3547ce4981",
  "contractAddress": null,
  "cumulativeGasUsed": 200618,
  "from": "0xefd0367d44bf23159599fc525e797c9b79b6deec",
  "status": 1,
  "transactionHash": "0x3f371728058e3a1ca32408c6c9613801ba0559ade91b6b63a3a389f985b5b658",
  "to": "0xd3d476033145a976ff0304de7cad767ba2dba1f4"
}
--------------------
```

我们可以查一下对应的 transaction 记录，如下：

{% asset_img etherscan-airdrop-drop-tx.jpg %}

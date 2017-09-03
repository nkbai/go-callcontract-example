# 以太坊系列之十六: 使用golang与智能合约进行交互

- [以太坊系列之十六: 使用golang与智能合约进行交互](#以太坊系列之十六-使用golang与智能合约进行交互)
    - [此例子的目录结构](#此例子的目录结构)
    - [token contract](#token-contract)
    - [智能合约的golang wrapper](#智能合约的golang-wrapper)
    - [部署合约](#部署合约)
        - [1.账户问题](#1账户问题)
        - [2. 连接到geth](#2-连接到geth)
        - [3. 部署合约](#3-部署合约)
        - [4. 测试部署结果](#4-测试部署结果)
    - [golang 查询合约](#golang-查询合约)

官方提供的使用web3来进行智能合约的部署,调用等,实际上使用go也是可以的,这样更接近geth源码,更多的库可以使用.

## 此例子的目录结构
方便大家对照使用
![目录结构](http://images2017.cnblogs.com/blog/124391/201709/124391-20170903105245171-1523829762.jpg)
我是在windows下进行的,在linux以及mac下都差不多,只需要更改里面的ipc地址即可

## token contract

这是官方提供的一个智能合约的例子,比较简单,是一个典型的基于智能合约的代币.代码位于:
[token源码](https://gist.github.com/karalabe/08f4b780e01c8452d989).

## 智能合约的golang wrapper

go直接和智能合约交互,有很多琐碎的细节需要照顾到,比较麻烦.以太坊专门为我们提供了一个abigen的工具,他可以根据sol或者abi文件生成
特定语言的封装,方便进行交互,支持golang,objc,java三种语言.
`abigen --sol token.sol --pkg mytoken --out token.go`
 [token.go地址](https://github.com/nkbai/go-callcontract-example/blob/master/mytoken/token.go)
可以看到里面把合约里面所有导出的函数都进行了封装,方便调用.

## 部署合约

使用go进行部署合约思路上和web3都差不多,首先需要启动geth,然后通过我们的程序通过ipc连接到geth进行操作.

直接上代码,然后解释
``` go
package main

import (
	"fmt"
	"log"
	"math/big"
	"strings"
	"time"

	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/ethclient"
	"token-contract/mytoken"
)

const key = `
{
  "address": "1a9ec3b0b807464e6d3398a59d6b0a369bf422fa",
  "crypto": {
    "cipher": "aes-128-ctr",
    "ciphertext": "a471054846fb03e3e271339204420806334d1f09d6da40605a1a152e0d8e35f3",
    "cipherparams": {
      "iv": "44c5095dc698392c55a65aae46e0b5d9"
    },
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "n": 262144,
      "p": 1,
      "r": 8,
      "salt": "e0a5fbaecaa3e75e20bccf61ee175141f3597d3b1bae6a28fe09f3507e63545e"
    },
    "mac": "cb3f62975cf6e7dfb454c2973bdd4a59f87262956d5534cdc87fb35703364043"
  },
  "id": "e08301fb-a263-4643-9c2b-d28959f66d6a",
  "version": 3
}
`

func main() {
	// Create an IPC based RPC connection to a remote node and an authorized transactor
	conn, err := ethclient.Dial("\\\\.\\pipe\\geth.ipc")
	if err != nil {
		log.Fatalf("Failed to connect to the Ethereum client: %v", err)
	}
	auth, err := bind.NewTransactor(strings.NewReader(key), "123")
	if err != nil {
		log.Fatalf("Failed to create authorized transactor: %v", err)
	}
	// Deploy a new awesome contract for the binding demo
	address, tx, token, err := mytoken.DeployMyToken(auth, conn, big.NewInt(9651), "Contracts in Go!!!", 0, "Go!")
	if err != nil {
		log.Fatalf("Failed to deploy new token contract: %v", err)
	}
	fmt.Printf("Contract pending deploy: 0x%x\n", address)
	fmt.Printf("Transaction waiting to be mined: 0x%x\n\n", tx.Hash())

	// Don't even wait, check its presence in the local pending state
	time.Sleep(1250 * time.Millisecond) // Allow it to be processed by the local node :P

	name, err := token.Name(&bind.CallOpts{Pending: true})
	if err != nil {
		log.Fatalf("Failed to retrieve pending name: %v", err)
	}
	fmt.Println("Pending name:", name)
}
```

### 1.账户问题
    部署合约是需要有以太坊账户的,账户一般位于/home/xxx/.eth/geth/keystore 目录里面,找到一个账户,然后把内容直接粘贴到key里面即可.
    因为部署合约是要消耗以太币的,所以必须保证里面有以太币,并且在`bind.NewTransactor(strings.NewReader(key), "123")`时,还需提供密码.
### 2. 连接到geth
    `ethclient.Dial("\\\\.\\pipe\\geth.ipc")`就是连接到本地的geth,你可以通过http等通道,要是使用http通道,记得geth启动的时候要加上
    `--rpcapi "eth,admin,web3,net,debug" `,否则很多rpc api是无法使用的
### 3. 部署合约
    真正的部署合约反而是比较简单,因为有了token的golang封装,就像直接调用构造函数一样. 只不过多了两个参数,第一个是auth,也就是账户的封装;
    第二个是ethclient的连接.
### 4. 测试部署结果
    在testrpc或者其他模拟区块链上,因为合约部署不需要花时间,所以`name, err := token.Name(&bind.CallOpts{Pending: true})`是可以获取到name的,
    但是在真实的区块链上有一个比较大的延时,所以运行结果会是:
```
    Contract pending deploy: 0x5e300171d7dc10e43f959877dba98a44df5d1466
Transaction waiting to be mined: 0xd49257ac92cff3c68f7e62604cb92edd585030f2973f6b8e389319aa98540f31

2017/09/03 10:12:27 Failed to retrieve pending name: no contract code at given address
exit status 1
```
    可以看到获取不到名字,但是我们是可以获取到合约的地址以及整个tx的地址的,我们可以利用这些信息进行其他操作.比如查询tx的状态来知道合约最终创建完成没有.

    ## 调用合约

    这里说调用实际上指的是要发生tx,这里举的例子就是token中进行转账操作,因为这个操作修改了合约的状态,所以它必须是一个tx(事务).

    先看完整的例子
```go
const key = `
{
  "address": "1a9ec3b0b807464e6d3398a59d6b0a369bf422fa",
  "crypto": {
    "cipher": "aes-128-ctr",
    "ciphertext": "a471054846fb03e3e271339204420806334d1f09d6da40605a1a152e0d8e35f3",
    "cipherparams": {
      "iv": "44c5095dc698392c55a65aae46e0b5d9"
    },
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "n": 262144,
      "p": 1,
      "r": 8,
      "salt": "e0a5fbaecaa3e75e20bccf61ee175141f3597d3b1bae6a28fe09f3507e63545e"
    },
    "mac": "cb3f62975cf6e7dfb454c2973bdd4a59f87262956d5534cdc87fb35703364043"
  },
  "id": "e08301fb-a263-4643-9c2b-d28959f66d6a",
  "version": 3
}`

func main() {
	// Create an IPC based RPC connection to a remote node and instantiate a contract binding
	conn, err := ethclient.Dial("\\\\.\\pipe\\geth.ipc")
	if err != nil {
		log.Fatalf("Failed to connect to the Ethereum client: %v", err)
	}
	token, err := mytoken.NewMyToken(common.HexToAddress("0x5e300171d7dc10e43f959877dba98a44df5d1466"), conn)
	if err != nil {
		log.Fatalf("Failed to instantiate a Token contract: %v", err)
	}
	// Create an authorized transactor and spend 1 unicorn
	auth, err := bind.NewTransactor(strings.NewReader(key), "123")
	if err != nil {
		log.Fatalf("Failed to create authorized transactor: %v", err)
	}
	tx, err := token.Transfer(auth, common.HexToAddress("0x8c1b2e9e838e2bf510ec7ff49cc607b718ce8401"), big.NewInt(387))
	if err != nil {
		log.Fatalf("Failed to request token transfer: %v", err)
	}
	fmt.Printf("Transfer pending: 0x%x\n", tx.Hash())
}

```

    首先同样要连接到geth,然后才能进行后续操作.
    ### 1. 直接构造合约
    因为合约已经部署到区块链上了,我们直接基于地址构造合约就可以了.
    `token, err := mytoken.NewMyToken(common.HexToAddress("0x5e300171d7dc10e43f959877dba98a44df5d1466"), conn)`
    这次我们不需要auth,因为这个操作实际上是仅读取区块链上的内容.

    ### 2. 创建账户
    进行转账修改了合约的状态,必须需要auth,和上次一样创建即可.
    ### 3.进行转账(函数调用)
    转账操作`token.Transfer(auth, common.HexToAddress("0x8c1b2e9e838e2bf510ec7ff49cc607b718ce8401"), big.NewInt(387))`,是要给账户
    0x8c1b2e9e838e2bf510ec7ff49cc607b718ce8401转387个代币,这实际上是调用了sol中的
```sol
    /* Send coins */
    function transfer(address _to, uint256 _value) {
        if (balanceOf[msg.sender] < _value) throw;           // Check if the sender has enough
        if (balanceOf[_to] + _value < balanceOf[_to]) throw; // Check for overflows
        balanceOf[msg.sender] -= _value;                     // Subtract from the sender
        balanceOf[_to] += _value;                            // Add the same to the recipient
        Transfer(msg.sender, _to, _value);                   // Notify anyone listening that this transfer took place
    }
```
    可以看出,和在sol中是差不多的,只不过多了一个auth,他实际上起到的作用就是要对这个事务进行签名.

    ### 4.结果
    转账是一个tx,必须等待矿工挖矿,提交到区块链中以后才能查询到结果,也可以在程序中等待一段时间进行查询,我们这里就直接在remix中进行查询了.

        #### (1) 打开http://ethereum.github.io/browser-solidity/#version=soljson-v0.4.16+commit.d7661dd9.js
        #### (2) 粘贴token.sol的内容
        #### (3) 切换到Web3 Provider
        #### (4) 使用At Address创建合约
            这是因为我们合约已经创建完毕了,通过指定地址就可以直接与我们的合约进行交互了
            然后可以调用balanceof来查询已经到账了.截图如下:
            ![转账结果](http://images2017.cnblogs.com/blog/124391/201709/124391-20170903111550312-1449512959.jpg)


## golang 查询合约

前一个例子中我们借助remix查询到已经到账了,实际上golang完全可以做到,并且做起来也很简单.
先看代码,再做解释.
```go
func main() {
	// Create an IPC based RPC connection to a remote node and instantiate a contract binding
	conn, err := ethclient.Dial("\\\\.\\pipe\\geth.ipc")
	if err != nil {
		log.Fatalf("Failed to connect to the Ethereum client: %v", err)
	}
	token, err := mytoken.NewMyToken(common.HexToAddress("0x5e300171d7dc10e43f959877dba98a44df5d1466"), conn)
	if err != nil {
		log.Fatalf("Failed to instantiate a Token contract: %v", err)
	}

	contractName, err := token.Name(nil)
	if err != nil {
		log.Fatalf("query name err:%v", err)
	}
	fmt.Printf("MyToken Name is:%s\n", contractName)
	balance, err := token.BalanceOf(nil, common.HexToAddress("0x8c1b2e9e838e2bf510ec7ff49cc607b718ce8401"))
	if err != nil {
		log.Fatalf("query balance error:%v", err)
	}
	fmt.Printf("0x8c1b2e9e838e2bf510ec7ff49cc607b718ce8401's balance is %s\n", balance)
}
```

运行结果:
```
MyToken Name is:Contracts in Go!!!
0x8c1b2e9e838e2bf510ec7ff49cc607b718ce8401's balance is 387
```
token.Name,token.BalanceOf就是读取合约上的数据,因为这些操作并不会修改合约的状态,所以不会发起tx,也不需要auth.
读取合约的第一个参数是bind.CallOpts,定义如下:
```go
// CallOpts is the collection of options to fine tune a contract call request.
type CallOpts struct {
	Pending bool           // Whether to operate on the pending state or the last known one
	From    common.Address // Optional the sender address, otherwise the first account is used

	Context context.Context // Network context to support cancellation and timeouts (nil = no timeout)
}
```
大多数时候直接使用nil即可,我们不需要特殊指定什么.
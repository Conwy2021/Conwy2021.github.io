# Lendfme事件分析


## 事件相关信息

公链 以太坊链

黑客地址
0xa9bf70a420d364e923c74448d9d817d3f2a77822

攻击合约
0x538359785a8d5ab1a741a0ba94f26a800759d91d

imBTC 地址
0x3212b29e33587a00fb1c83346f5dbfa69a458923 精度位 8 
也就是为什么我们在交易记录中看到他的数字都是小数点后8位的原因
另外这个代币是ERC777
*imBTC* (ERC-777) 是Tokenlon 在以太坊上发行的BTC 锚定币

借贷合约lendf.me
0x0eee3e3828a45f7601d5f54bf49bb01d1a9df5ea

interestRateModel合约的实现地址 利率算法合约 
0x9a18c4d9587344f2b15686aa67ee7e5c4b00d549 （这里并没有用到）

预言机合约 
0xb620707637c5b2cc49843a03d90e28d9abbda149 （这里并没有用到）
lendf.me 调用oracle函数返回价格用的

攻击者先给0x538359785a8d5ab1a741a0ba94f26a800759d91d 打了 21595 imBTC
交易hash 0x88fa4e8609baac44189a58faf7cb740cf35308957832ffd6656999229fea689f
(使用链必追查询，也可以进入合约地址的ERC-20 Token Txns 查到)

## 攻击流程

先上结论：

攻击者的思路是我提取我在借贷合约lendf.me（后续简称lendf.me）中的余额后，lendf.me中我的余额没有减少，却增加了。换句话说我提款后我在lendf.me的余额应该是减少的，实际却没有减少。注意是借贷合约lendf.me中的余额，用户在lendf.me中存的钱会保存在balance.principal中（结构体中的一个参数 我叫他本金）也就这个参数保存了用户在lendf.me合约中余额。

我们先分析他的第一笔攻击交易
0xe49304cd3edccf32069dcbbb5df7ac3b8678daad34d0ad1927aa725a8966d52a

地址：https://etherscan.io/tx/0xe49304cd3edccf32069dcbbb5df7ac3b8678daad34d0ad1927aa725a8966d52a

![/image-20221228215914137](lendfme事件分析.assets/image-20221228215914137.png)

请注意，这是攻击的第一笔交易，攻击合约目前有21595个imBTC代币。攻击合约之前是没有在lendf.me存过款的，所以我在单看这笔交易时，其实没发现什么问题。

下面我们看下代码调用过程的，通过分析代码的调用，看看能发现什么新线索。
使用工具blocksec。

红框部分是这笔交易的第一次supply（存款操作）函数的调用

![/image-20221228220253003](lendfme事件分析.assets/image-20221228220253003.png)

注意
EVENT中的amount=21,594, startingBalance=0, newBalance=21,594

说的意思是转入了21,594，刚开始本金是0，更新后的本金是21,594.也证实了这是攻击合约第一次在存款。

我们来看下这笔交易的第二次 supply

![/image-20221228220330992](lendfme事件分析.assets/image-20221228220330992.png)

EVENT中amount=1, startingBalance=0, newBalance=21,595
说的意思是转入了1，刚开始本金是0，更新后的本金是21956。
请注意我们在看ETH浏览器的交易中是，他是有withdraw（提款）的。我们看代码第二次调用时supply 中的imBTC.transferFrom 方法是与第一次不同的 其中多了第4层call，其中就含有Lendf.Me.withdraw方法的调用。
我们也在函数调用中看到攻击合约提取了21594个imBTC。这里发现两个问题，
1、	第二次transferFrom 中多了个withdraw方法。
2、	第二次supply更新后的本金是21955。提取21594 后本金本应该减少的。
先上答案。
1、	imBTC.transferFrom 方法存在重入
2、	重入导致攻击合约在lendf.me中余额变化异常，重点是攻击合约在lendf.me的余额记录。
第一个问题的解释，哪里重入了。
ImBTC是ERC777合约 这个合约会在转账时通知目标合约谁给你转钱了。也就是在通知这里有重入。（这个地方需要自己补充下相关知识，ERC777和ERC1820）
LendfMe.sol

![/image-20221228221026020](lendfme事件分析.assets/image-20221228221026020.png)

![/image-20221228221033373](lendfme事件分析.assets/image-20221228221033373.png)

imBTC.sol

​	![/image-20221228221044504](lendfme事件分析.assets/image-20221228221044504.png)

![/image-20221228221048895](lendfme事件分析.assets/image-20221228221048895.png)

![/image-20221228221055457](lendfme事件分析.assets/image-20221228221055457.png)

攻击合约在tokenToSend中重入withdraw函数。
第二个问题的解释，重入导致了什么问题。
LendfMe.sol

![/image-20221228221119637](lendfme事件分析.assets/image-20221228221119637.png)

Balance.principal 保存的是用户在lendf.me金额，在第二次supply中，攻击合约同过重入提取了之前存入的21594后，重入函数中withdraw函数执行完成。
Withdraw函数中是有正确的更改用户在合约的余额的。
此时
balance.principal 其实是0 的。也是正确的数值。
重入函数中withdraw函数执行完成后，转账函数进而执行完成。（也就是交易详情为什么withdraw在第二次supply前面的原因）
继续进行1590行后的操作，localResults.userSupplyUpdated存的是21594+1。再更新为用户的余额就是21595。也就是supply的余额覆盖了正确的数值。
正常用户的余额应该是1。

![/image-20221228221158953](lendfme事件分析.assets/image-20221228221158953.png)

localResults.userSupplyUpdated存的是21594+1的由来，是第一次supply的21594，加上第二次supply中的1。
补充说下：calculateBalance函数是计算你存入期间本金+利息。我们这一笔交易没有利息。
以上就是这次攻击的分析，总结下：

重点就是用户第一次存入了21594，合约中本金是21594.第二次存入1，提取21594，合约中本金却是21595。
下面按数据变化分析整个流程：
第一笔
攻击交易：0xe49304cd3edccf32069dcbbb5df7ac3b8678daad34d0ad1927aa725a8966d52a
时间 Apr-19-2020 12:58:43 AM +UTC
交易过程

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 21595
Lendf.Me                                      imBTC 29134710218

交易动作 Lendf.Me.supply(asset=imBTC, amount=21594)（第一次正常访问）
转账交易
0x538359785a8d5ab1a741a0ba94f26a800759d91d -> Lendf.Me   imBTC 21594

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 1
Lendf.Me                                      imBTC 29134731812

交易动作 Lendf.Me.supply(asset=imBTC, amount=1)（第二次supply 重入withdraw）
转账交易
0x538359785a8d5ab1a741a0ba94f26a800759d91d -> Lendf.Me   imBTC 1

**正常交易的账号余额应为
账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 0
Lendf.Me                                      imBTC 29134731813

withdraw
Lendf.Me  -> 0x538359785a8d5ab1a741a0ba94f26a800759d91d imBTC 21594

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 21594
Lendf.Me                                      imBTC 29134710219

Lendf.Me合约用户的余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 21595

重点就是这个Lendf.Me合约用户的余额。这个数据看第二笔交易查到的。

这边附上 查看攻击合约的代码。使用工具https://library.dedaub.com/decompile

![/image-20221228221404349](lendfme事件分析.assets/image-20221228221404349.png)

_tokensToSend = 1; 表示第一次正常supply
_tokensToSend = 0; 表示第二次重入supply
重入代码为

![/image-20221228221420382](lendfme事件分析.assets/image-20221228221420382.png)

附上第二，三次交易数据交易流程
第二笔
攻击交易：0xae7d664bdfcc54220df4f18d339005c6faf6e62c9ca79c56387bc0389274363b
时间 Apr-19-2020 12:58:55 AM +UTC

交易过程

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 21594
Lendf.Me                                      imBTC 29134710219

交易动作 Lendf.Me.supply(asset=imBTC, amount=21593)
转账交易
0x538359785a8d5ab1a741a0ba94f26a800759d91d -> Lendf.Me   imBTC 21593

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 1
Lendf.Me                                      imBTC 29134731812

交易动作 Lendf.Me.supply(asset=imBTC, amount=1)
转账交易
0x538359785a8d5ab1a741a0ba94f26a800759d91d -> Lendf.Me   imBTC 1

**正常交易的账号余额应为
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 0
Lendf.Me                                      imBTC 29134731813

withdraw
Lendf.Me  -> 0x538359785a8d5ab1a741a0ba94f26a800759d91d imBTC 43188

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 43188
Lendf.Me                                      imBTC 29134688625

Lendf.Me合约用户余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 43189
第三笔
攻击交易：0xa0e7c8e933be65854bee69df3816a07be37e108c43bbe7c7f2c3da01b79ad95e
时间 Apr-19-2020 12:59:19 AM +UTC

交易过程

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 43188
Lendf.Me                                      imBTC 29134688625

交易动作 Lendf.Me.supply(asset=imBTC, amount=43187)
转账交易
0x538359785a8d5ab1a741a0ba94f26a800759d91d -> Lendf.Me   imBTC 43187

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 1
Lendf.Me                                      imBTC 29134731812

交易动作 Lendf.Me.supply(asset=imBTC, amount=1)
转账交易
0x538359785a8d5ab1a741a0ba94f26a800759d91d -> Lendf.Me   imBTC 1

**正常交易的账号余额应为
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 0
Lendf.Me                                      imBTC 29134731813

withdraw
Lendf.Me  -> 0x538359785a8d5ab1a741a0ba94f26a800759d91d imBTC 86376

账号余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 86376
Lendf.Me                                      imBTC 29134645437

Lendf.Me合约用户余额
0x538359785a8d5ab1a741a0ba94f26a800759d91d    imBTC 86377


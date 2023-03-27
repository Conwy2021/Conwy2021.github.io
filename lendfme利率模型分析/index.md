# lendfme利率存储分析


## Lendfme 借贷合约存储利率分析

Lendfme是模仿compound的，所有很多资料 我这看的是compound的。

Compound 18年9月发布v1。Lendfme模仿的也是这个版本。   
<br/>

首先我们确定下Lendfme的利率算法(**核心问题**)：根据白皮书（https://compound.finance/documents/Compound.Whitepaper.v04.pdf）和实际代码写法。Lendfme的利率计算方式和传统的存在不同。

传统的利率算法：(1+10%)<sup>N</sup>     &emsp; N表示周期

Lendfme利率算法：（1+10% \*N）

也就是说 在一段多周期的计算中，Lendfme是没有按每个周期的结果进行累加计算的。

 <br/>

存款利率变化时

传统的利率算法：(1+10%)<sup>N-1</sup>\*(1+20%)  &emsp;N表示周期

Lendfme利率算法：（1+10% \*(N-1))\*(1+20%)

 <br/>

 <br/>

举个例子：存款利率10%每天（每个周期），存5天（多周期）

传统我们存款利息的算法: 100*(1+10%)<sup>5</sup>

Lendfme存款利息的算法: 100 \*(1+10% \*5)

存款利率变化时

传统我们存款利息的算法: 100 \*(1+10%)<sup>4</sup> \*(1+20%)

Lendfme存款利息的算法: 100 \*(1+10%\*4) \*(1+20%)

 <br/>

这里我把这个SupplyIndex 称为本息率。如何实现计算的存款本金和利息？

假设 &emsp;我存款时 记录的SupplyIndex  是  &emsp; \(1+10%\*4) \*(1+20%) &emsp;在经历了几个区块后 变为\(1+10%\*4) \*(1+20%)* <span style="color:red">(1+10%\*2)\*(1+25%) </span>

那么此时我经历这几个区块后 我的本息率是 &nbsp; (1+10%\*2)\*(1+25%)  &emsp; 这个本息率再乘以本金 就是自己的本金和利息的总和了。

计算公示为&emsp;  

<br/>
$$
\frac {NewSupplyIndex}{SupplyIndex}=(1+10\%*2)(1+25\%)
$$
<br/>

分析第一种场景：我们在区块1 小A 存 ，区块2 小B借，区块4 小B借， 区块5 小A提 

其中，第一次借款后，存款利率变为10% ，第二次借款后，存款利率变为20%。以区块为周期。

当前场景下的数据为（借款后才会有存款利率，借1表示第一个借操作）

![](/lendfme利率模型分析.assets/image-20221228232025449.png)

![](/lendfme利率模型分析.assets/image-20221228232043738.png)

补充说明下：存的是当动作发生时newSupplyIndex，也是动作发生后的SupplyIndex。因为newSupplyIndex会在函数最后更新supplyIndex。

 <br/>

这边假设提款为0，让本金和利息都在用户账本中保存。方便我们理解，也就是说，当提款为0时，表示利息结算操作。

![](/lendfme利率模型分析.assets/image-20221228232103766.png)

说下这个市场账本（market）和用户账本(balance)。

市场账本market是个全局变量， 在每次存，借，提 会更新整体的总存款，存款利率，存款利率系数，总借款，借款利率，借款系数，并且更新发生这些动作时的\*区块号（区块号更新也是重点）\*。这里我们只看存款相关的数据。

用户账本balance 也是全局变量记录用户的本金和当时存款时或者提款时的存款利率系数SupplyIndex。

 <br/>

其中 old 和new 表示 SupplyIndex变更标记。每次使用时用old SupplyIndex算出new SupplyIndex。

这个SupplyIndex 会在每次存，借，提 都会更新的。（这个点也是重点）

 <br/>

再加个说明：SupplyIndex 这个值会更新到市场账本market的supplyIndex和用户账本balance的 interestIndex的。以当前场景 ，在小A存在存和提操作时，会把new supplyIndex 更新到自己账本上。

重点：市场账本的SupplyIndex和用户账本的interestIndex。

 <br/>

以当前场景进行计算：

用户提款的金额是多少

100 \*1.44/1=144 （这一步是calculateBalance里写的代码计算过程）

等同于 100 \*0.1+100\*0.1 +120\*0.2=144

 <br/>

calculateInterestIndex函数

oldSupplyIndex \*(1+old利率\*周期)=newSupplyIndex

 <br/>

calculateBalance 函数可以理解为结算收益（本金+利息）

本金 \* newSupplyIndex/ oldSupplyIndex=新本金

 <br/>

我们细分下每步操作：

存 

newSupplyIndex= oldSupplyIndex \*(1+old利率\*周期)=1\*(1+0)=1

SupplyIndex初始默认值是10 <sup>18</sup> 这里简化为1，这里的1 其实和以前奥数中表示1份或者1个单位类似 因为这个值会做除数，最后都会被约分掉。

新本金=本金\* newSupplyIndex/ oldSupplyIndex=0\*1/1=0 

后面有个add 操作会加上你传入的值 我们假设存100

userSupplyUpdated=0+100=100 这个值会在后面更新到用户账本

更新区块号为当前区块为 1

更新newSupplyIndex至市场账本和用户账本

 <br/>

借1（第一个借操作）

借款时只更新SupplyIndex

oldSupplyIndex\*(1+old利率\*周期)=newSupplyIndex（计算公式）

1\*（1+0\*1）=1 注意此时的old利率是0 

更新至市场账本

借2

oldSupplyIndex\*(1+old利率\*周期)=newSupplyIndex（计算公式）

1\*(1+10%\*2)=1.2 注意此时的old利率是10%,过了两个周期（以块为周期）

更新至市场账本

 <br/>

提

oldSupplyIndex\*(1+old利率\*周期)=newSupplyIndex（计算公式）

1.2\*(1+20%\*1)=1.44

calculateBalance(balance.principal, balance.interestIndex, localResults.newSupplyIndex);

计算用户的总金额 

注意这里的计算公式是 用户账本的本金\*市场账本算出新的存款利率系数/**用户账本存**的存款利率系数  

calculateBalance 函数的计算公式 ： 本金\* newSupplyIndex/ oldSupplyIndex=新本金

所以当前用户余额 为 100\*1.44/1 =144

更新newSupplyIndex 1.44 至市场账本和用户账本。

 <br/>

我们再进一步计算下 当第6块时用户的本金+利息是多少

按之前的想法 提款0 在第6块 表示 计算第6块时的本金和利息。

计算如下

calculateInterestIndex函数 公式如下

oldSupplyIndex\*(1+old利率\*周期)=newSupplyIndex

1.44\*（1+20% \*1）=1.728

 <br/>

calculateBalance函数 公式如下

本金 \* newSupplyIndex/ oldSupplyIndex=新本金

144 \*1.728/1.44 =144\*1.2=1728

 <br/>

更新 newSupplyIndex 1.728至市场账本和用户账本。

![](/lendfme利率模型分析.assets/image-20221228232133358.png)

我们再多算一步进行验证，第7块有人还钱了导致存款利率降低为10%，我们在第8块 计算下用户的总金额

 <br/>

有人还钱存款利率降低至10%

calculateInterestIndex函数 公式如下

oldSupplyIndex\*(1+old利率\*周期)=newSupplyIndex

1.728 \*（1+20%\*1）=2.0736

 <br/>

更新newSupplyIndex至市场账本。注意这里是没有更新用户账本的

 <br/>

第8区块 计算

calculateInterestIndex函数 公式如下

oldSupplyIndex\*(1+old利率 \*周期)=newSupplyIndex 这里周期为1的原因是之前区块7的时候我们更新了区块号。

2.0736 \*（1+10% \*1）=2.28096 -> 1.728\*(1+20%)(1+10%)

 <br/>

calculateBalance函数 公式如下

本金 \* newSupplyIndex/ oldSupplyIndex=新本金 （oldSupplyIndex是用户账本中的值）

172.8 \*2.28096/1.728 =228.096

 <br/>

等同于：172.8 \*(1+20%)\*(1+10%)=228.096 

其中乘以1+20%是第7块的总金额

乘以1+10%是第8块的总金额 同时我们也看出newSupplyIndex/ oldSupplyIndex 就是存款

利率的累乘值。

![](/lendfme利率模型分析.assets/image-20221228232201245.png)

![](/lendfme利率模型分析.assets/image-20221228232213106.png)

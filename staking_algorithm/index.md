#  uniswap v2 抵押挖矿 算法分析




初学uniswap v2 抵押挖矿 不明白其中算法

```bash
 rewardPerTokenStored.add(
               lastTimeRewardApplicable().sub(lastUpdateTime).mul(rewardRate).mul(1e18).div(_totalSupply)
            );
```

是如何来的。核心实现函数为modifier updateReward()。  
网上的资料也比较少。所以自己把学习过程记录下（我会尽量让自己的分析正确无误，不误导大家）。  
首先我还是找到了些资料的。感谢这些优秀的博主分析。  
视频分析
Synthetix Staking Rewards Contract Explained: 

Part 0 - [https://youtu.be/6ZO5aYg1GI8](https://www.youtube.com/watch?v=6ZO5aYg1GI8) 

Part 1 -  [https://youtu.be/LWWsjw3cgDk](https://www.youtube.com/watch?v=LWWsjw3cgDk) 

Part 2 - [https://youtu.be/YqpRwJDz3xg](https://www.youtube.com/watch?v=YqpRwJDz3xg) 

Part 3 - [https://youtu.be/pFX1-kNrJFU](https://www.youtube.com/watch?v=pFX1-kNrJFU)  
大致说下 视频从数学公式上  现从传统思路上分析  
r(u,a,b) 意思是 代币u 在时间a 到b的 收益  
l(u,t)  意思是 用户在t时间点时 代币的数量   

![](/image1/image-20220824202418199.png)

对应的代码表示

![](/image1/image-20220824202509957.png)

白话分析：  
保存好每个时间点应的自己代币的数量(balanceAt(t))，和这个时间点对应当代币总量（totalSupplyAt(t)）,然后计算每个阶段自己的收益（reward）累加进行计算。   
举例分析：（视频里的图，图示化的讲解真的太好了）

![](/image1/image-20220824202849620.png) 

先明确下已知信息  
**大黄**的抵押代币是100 **小熊**的抵押代币是100  
按图这个逻辑，  
大黄在7秒时进行抵押，14秒时取出。  
小熊在9秒时进行抵押，18秒时取出。  
我们来计算下**大黄**的收益 按上面的代码计算（它按1秒 1秒 计算收益）    
第1到7秒都是没有收益的 reward=0  
第8秒  REWARD_RATE\*100/100 = 1R  reward=1R  
第9秒  REWARD_RATE\*100/100 = 1R  reward=1R+1R  
第10秒  REWARD_RATE\*100/200 = 0.5R reward=1R+1R+0.5R  
第11秒  REWARD_RATE\*100/200 = 0.5R reward=1R+1R+0.5R+0.5R  
第12秒  REWARD_RATE\*100/200 = 0.5R reward=1R+1R+0.5R+0.5R+0.5R  
第13秒  REWARD_RATE\*100/200 = 0.5R reward=1R+1R+0.5R+0.5R+0.5R+0.5R  
第14秒  REWARD_RATE\*100/200 = 0.5R reward=1R+1R+0.5R+0.5R+0.5R+0.5R+0.5R   
最后的reward=4.5R  
我们可以分析出按这算法来搞 执行是要消耗gas的 是不行的  
那么有人会提出 按每一秒算 不行 咱们按时间差算行不行 7到9秒 产出为1R 9到14秒产出为0.5R reward=1R\*2+0.5R\*5=4.5  
这种情景下 我们需要保存每次变化的totalSupply 并且记录变化时的时间戳 按timetamps[]=[0,7,9,14,18]   
伪代码为 

```bash
for(uint i=0;i<timestamps.length;i++){
	uint t=timestamps[i];
	uint t1=timestamps[i+1];
	reward+=R*(t1-t)*balanceAt(t)[msg.sender]/totalSupplyAt(t)
}
```
验证下这种算法：
 t=0 t1=7  reward=0  
 t=7 t1=9 reward=R\*2\*100/100=2R  
 t=9 t1=14 reward=2R+R\*5\*100/200=4.5R  
 t=14 t1=18 reward=4.5R+R\*4\*100/100=8.5R        

很显然 虽然步骤少了 但是这只是4个时间点的变化 实际时间变化点是很多的 所以这个方案 是不行的
继续视频中的推导

![](/image1/image-20220824203017999.png) 

r(u,a,b) 意思是 代币u 在时间a 到b的 收益  
l(u,t)  意思是 用户在t时间点时 代币的数量  
首先k 是 8到14秒时 **小黄**的代币数量 是个常数 我们给提取出来  
为什么a=8 开始呢 以为第8秒时才有收益  
公式现在推到为这个样子  

![](/image1/image-20220824203055607.png) 

![](/image1/image-20220824203213865.png) 

这一步 补充下说明 我们要记住 这个是按每秒进行计算的 每秒表示变化   
其中 1/L(0)+1/L(1)+...+1/L(b)  多写几项方便理解  1/L(0)+1/L(1)+..+1/L(a-1)+1/L(a)+1/L(a+1)+.......+1/L(b)

![](/image1/image-20220824203236135.png)

最后公式为

![image-20220824210156872](/image1/image-20220824210156872.png)

![image-20220824210305585](/image1/image-20220824210305585.png)



根据这个公式推演
**大黄**的收益

![](/image1/image-20220824203328916.png)

**小熊**的收益

![](/image1/image-20220824203348136.png)

总结下：
   当前时刻的s的累加 - 前一个时刻s的累加[account] = account增加的s  
rewardPerTokenStored 其实是个公共的数据 所有人共享的   
它的含义 应该是一定时间段<sub>这个时间段代币总量不变</sub>的每个代币可以提供多少个奖励代币的累加集合   
每当代币的数量有变化时 或者时间存在增量（可以理解成代币的数量变化为0）时 就会进行累加计算 。  
    当前时刻的s的累加 每次计算完reward后 会更新userRewardPerTokenPaid[account]  
    就会变成未来一个时刻的减号后面的数字   

最后演算下 这个场景

![](/image1/image-20220824203433388.png)

![](/image1/image-20220824203446447.png)

我们根据上面那个有横轴纵轴的图，从图像上分析他的算法。

假设用L(t) 是总收益，一定时间段<sub>这个时间段代币总量不变</sub>的每个代币可以兑换的奖励币为m<sub>n </sub> ,m<sub>n </sub>会根据代币总量的变化而变化 

我们每秒的计算 下 L(t)=m<sub>1 </sub>\*2 + m<sub>2 </sub>\*3 + m<sub>3 </sub>\*2 + m<sub>4 </sub>\*4

计算下 **大熊**的收益   他抵押代币时的 L(t)为m<sub>1 </sub>\*2  他提取代币时的L(t)为m<sub>1 </sub>\*2 + m<sub>2 </sub>\*3 + m<sub>3 </sub>\*2 + m<sub>4 </sub>\*4    

那么收益为  m<sub>1 </sub>\*2 + m<sub>2 </sub>\*3 + m<sub>3 </sub>\*2 + m<sub>4 </sub>\*4 - m<sub>1 </sub>\*2 = m<sub>2 </sub>\*3 + m<sub>3 </sub>\*2 + m<sub>4 </sub>\*4 再乘以他的代币数量100  

我们把100 放到里面去  m<sub>2 </sub>\*3\*100 + m<sub>3 </sub>\*2*100 + m<sub>4 </sub>\*4\*100  可以看出 这个其实就时每个阶段的收益之和  

当**大熊**得到收益后 会把记录下自己领过奖励时的 L(t)=m<sub>1 </sub>\*2 + m<sub>2 </sub>\*3 + m<sub>3 </sub>\*2 + m<sub>4 </sub>\*4  下次再计算时 就用最新的L(t)减去之前的（这里假               设**大熊**还有代币在池子里）

PS：这里**大熊**实际是把代币都领出去了。下次质押代币时，会先出发modifier updateReward()函数 记录抵押时的L(t)。就像他第一次抵  押一样。

***

这边再大致说下MasterChef的计算思路 

代码 在https://etherscan.io/address/0xc2EdaD668740f1aA35E4D8f227fB8E17dcA888Cd#code 我在他git上没找到  

MasterChef是每个区块产生100个奖励代币 SushiToken （他设计在多少区块前奖励是10倍，我们先按没有这个设计理解）

记录 一定区块段<sub>这个区块段代币总量不变</sub>的每个代币可以兑换的奖励币累加值m<sub>n </sub>  和自己抵押时对应的m<sub>n </sub>

然后就是累加m<sub>n </sub>-自己抵押时对应的m<sub>n </sub>

 核心函数为updatePool()  



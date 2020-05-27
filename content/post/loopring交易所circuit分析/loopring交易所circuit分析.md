---
title: "Loopring交易所circuit分析"
date: 2020-04-06T10:52:22+08:00
draft: false
summary: "Loopring交易所circuit分析，loopring是以太坊上基于zkrollup的去中心化交易所，能够保证交易所作恶但不能获益。其零知识部分基于libsnark和ethsnark"
tags: [loopring,zkrollup,zksnark]
---

### 电路列表

1. Ring Settlement(环结算)：主要是通过撮合两种不同代币的完成各种`token`之间的相互转化。环结算的协议费和操作费只能用单一的币种进行结算。
2. Deposit：主要是用户向链上存储`token`，并且改变链上的`merkleroot`的状态  。`updateBalanceGadget`和`updateAccountGadget`一般有两个功能，一种是校验当前计算的`merkleroot`是否和保存的`merkleroot`相互匹配。一种是计算更新之后新的`merkleroot`的值，并且进行保存。
3. off-chain withdraw: 优点是比较快，直接发到中继进行处理，缺点是中继可以选择性不处理。这个操作需要支付交易费用，并且交易费用可以使用和交易币种不同的token。
4. on-chain withdraw: 有关闭模式，当`bshutdownMode`模式开启后进入交易所关闭模式，所有的`on-chain`请求都会被禁止，未处理完的`on-chain deposit`请求会继续被处理。只有特殊的`on-chain withdraw`请求会被commit，这些请求会重置merkletree上的相应内容，当整个merkletree恢复到最初状态，交易所能够拿出它抵押的金额。
5. Order Cancellation：取消当前要撮合的订单。

### 注意点

1. pb_variable 类型中的`pb_variable`是`unsigned long`类型的变量。
2. 每个电路的`generate_r1cs_constraints`函数中会存在num变量，**这个值是固化在电路中的？**，主要是用来记录一个block中含有多少个该类的交易。
3. `label`主要是用来保存一些交易信息，是将一些复杂的功能简化，比如在`deposit`电路中`label`可以用来表示这个订单是从哪个钱包来的单子，在`offchainwithdraw`中`operator`需要收取一定的手续费，`label`可以用来标记operator后续手续费的具体分配，因为在链上存储了每一个block的`labelhash`，`label`的内容就不能被更改。因此也是一种使用简单的方法将各种复杂问题一起处理。
4. `DualVariableGadget`组件并不简单，因为各个`gadget`是通过`generate_r1cs_witness`函数来进行传入输入值——相当于一些`gadget`的见证，通过`generate_r1cs_constraints`来进行电路约束，但是在逻辑上后者是在前者之前被执行，但是在`constraints`中添加约束的变量值又是在`witness`函数中进行传入的——这里是因为`constraints`里面仅仅是对于电路的逻辑约束，具体的值的校验要在`witness`函数执行之后再进行。如果前者的`witness`过程出现错误，应该返回`false`。
5. `onchainwithdrawal`电路和`deposit`电路因为是on-chain操作，在eth中生成的交易中含有nonce，因此Account中的`nonce`随机数值不会进行加1。
6. 注意每条电路中的`publicData`值是每条电路的输入值和输出值组合在一起，并且计算结果hash，并且校验两者hash是否相同。注意hash结果的值是在`constraints`里面进行计算的。
7. `/blocks`下面json文件格式为`"./blocks/block_" + exchangeID + "_" + nextBlockIdx + "_info.json"`，`/states`下面的json文件格式为`"./states/state_" + str(exchangeID) + "_" + str(previousBlockIdx) + ".json"`，`exchangeID`为交易所ID，后面是块的ID。
8. 关于类型`pb.val(VariableT) = Field`
9. 使用的数字签名函数和hash函数有创新，使用的hash函数是`poseidon`哈希函数，其使用的曲线是`jubjub`曲线，通过博客说是有较好的性能。使用的数字签名函数是基于`poseidon`哈希函数的EdDSA数字签名函数。
10. `protocolFeeBips`在`ringsettlementblock`，这个值随着抵押金额的增加费用逐渐降低。`protocolFee  = amountB * protocolFeeBips / 100000`，这个费用是operator给`protocol Account`。fee 和 rebate 在 order 中。`fee = amountB * feeBips / 10000`。`rebate = amount * rebateBips / 10000`。fee是用户支付给operator，rebate是operator支付给(钱包用户)用户的回扣。
11. bshutdown模式为交易所关门模式，交易所会继续处理用户的OnchainWithdraw，即count不为0，然后withdraws里面的数据是用户提交要处理的出局。Operator也可以自己填充单子，具体的做法是将count置为0，然后自己填充withdraws，operator自己生成证明将用户的token退还给用户。
12. 当交易所跑路后，用户需要使用`dataavailable`模式获取链上所有的数据，然后通过`dataavailable`的数据证明自己有一些余额，链上合约会返回用户的token，链上的merkleroot和calldata，保证了数据链下数据不被篡改。calldata保证了根据链上数据一步一步算的时候每一步输入的正确性，最后结果会生成一个merkletree。通过验证本地的merkleroot和链上merkleroot是否相同，从而证明本地计算数据的正确性和真实性。
13. 交易所对于是否开启`onChainDataAvailable`有两种玩法，一种是开启`onChainDataAvailable`，用户可以监管和交易所跑路，用户可以证明自己有这么多钱。一种不开启，虽然吞吐量高，但是用户需要承担交易所跑路并且销毁数据，用户就不能证明自己有那么多钱。
14. 对于on-chain的`deposit`和`onchainWithdraw`两个操作计算每个子操作计算publicData的hash的时候是计算`accumulated hash`。
15. merkletree当中的account下有nonce变量，该变量防止重发off-chain操作，因此每一次off-chain操作都应该使得nonce的值加1。将该nonce包含进用户offchainWithdraw和orderCancellation，operator的RingSettlement的签名中，防止off-chain操作被重放。对于bshutdown的OnchainWithdraw操作，account的nonce值要被回执为0。
16. 在每条电路运行过程中的hash(publicData)的前23位，就是生成证明中的input的值。这样做是为了固定电路的输入值，当链上使用生成的proof验证时通过该值可以确保电路的输入是链上的输入。
17. 对于链上存储的`depositBlockHashStart`是一个bigint类型，该类型相当于是小端正常hash值的大端存储。因此在bigint转换成`DualVariableGadget`的过程中会将bigint转化成小端存储的hash值。**因此链上都是存储的反向的hash值，因此每次导出块就是反向hash值的**。对于大端和小端的理解：存储方式不一样，小端数据翻转后传输到大端上两者的值是相同的。
18. 对于DualVariableGadget相当于是一个动态数组，里面的packed相当于值，bits相当于这个gadgets所占的位数。
19. onchain操作通过accumlatehash值来确保用来prove的值不被更改。首先用户提交数据到eth，在eth链上算accumlatehash，operator打包block去生成证明。publicData中的accumlatehash可以和链上计算的accumlatehash相互比较，确保prove的输入的数据正确。如果operator输入prove的数据正确，但是上链的publicData被修改，那么链上使用publicData计算publicInput，然后用该值verifiy的时候会错误。
20. 在链下保存merkletree的状态，merkletree的状态每次在链下更新只会计算一遍，在计算的过程中相应的程序会生成block.json，该json文件包含merkletree状态更新的一些中间过程。然后该json文件会输入到prove中生成证明，如果operator作恶，那么证明的生成会错误。相当于整体上计算了两遍，中继一边，电路会计算一边使用前一遍执行的中间结果计算一边然后进行验证。
21. 数据一致性保证：

- operator不能更改用户的数据，用户对应状态树中的私钥签名保证。
- operator不能直接改merkletree上的状态，merklerootbefore和merklerootafter保证。
- operator不能改链下计算状态树后的数据，block.json是block_info.json经过链下程序计算得到。block.json是用来生成证明的数据，也就是block_info.json程序在计算过程中的一些中间过程保存在block.json中。改block_info.json，对于Onchain行为，使用accumulateHash进行校验，对于offchain行为，使用loopring的公钥验证数字签名会进行校验。改block.json中的数据，生成证明会失败。

22. merkletree使用稀疏merkletree。
23. signature分析，一共使用了两种公私密钥，eth公私钥，loopring公私钥。
24. order取消后cancell会被变为1，在ringsettlement的时候MatchingGadget将fill为0，在orderMatching检查订单CheckValidGadget，会在这里检查出fill为0。isNonZeroFillAmountS。
25. **publicDatahash在电路中计算一次确保了你在电路中计算所用的输入是和你commit到链上的block相同。然后存在一种情况就是commit到链上的block和输入到电路中额值都被中继改了，其实有两种方式校验，第一对应offchain的操作，用户会用loopring的私钥给其签名。对于onchain操作（在链下计算开始时改数据，以太坊公钥对订单验签不会通过）。2. 通过merkleroot校验，一个账户是5，你改成10，明显merkleroot的校验不会通过。（计算完之后改数据，merkleRoot校验不会通过）**

 **用户使用loopring私钥签名的场景**，下面都是offchain操作，为了保证数据不被更改，需要用户进行签名。onchain操作通过链上的accumulateHash和以太坊的私钥的签名来保证。

- offchainwithdraw request单笔交易的数据签名。
- Ordercancellation request签名
- OrderGadget 用户对自己订单进行签名

**operator使用loopring私钥签名的场景**

- operator 对ringsettlement的publicData.publicInput和accountBefore_O.nonce两者进行验证签名，因为operator能操作fee和rebate的值，它需要使用自己的私钥来授权支付的rebate和protocol fee。

24. onchain和offchain的withdraw都会在publicData里面记录`getApprovedWithdrawalData`数据，这样做主要是为了得出withdrawamount的实际金额。因为订单请求withdraw的金额和实际withdrawamount的金额要根据balance的值来得出。

### 可能问题

1. `offchainWithdrawalCircuit.h`对账户费用余额和账户余额两个账户的处理有略微不同。`feePayment(pb, NUM_BITS_AMOUNT, balanceFBefore.balance, balanceBefore_O.balance, fFee.value(), FMT(prefix, ".feePayment"))`对于费用账户的处理，有`overflow`的校验，但是对`balance_after(pb, balanceBefore.balance, amountWithdrawn.value(), FMT(prefix, ".balance_after"))`的处理是`UnsafeSubGadget`，没有进行溢出检测的校验。因此第107行左右的`balance_after`的初始化有疑问。

- **解决**：这里的`amountWithdrawn.generate_r1cs_witness(toFloat(pb.val(amountToWithdraw.result()), Float28Encoding));``amountWithdrawn`的值在withness里面使用 `amountRequested.packed`和`balanceBefore.balance`两者中的最小值进行了`witness`步骤，因此在`balance_after`进行初始化的时候不会初始化成负值。

2. `onchainWithdrawalCircuit` ，`balance_after`的初始化也有同样问题。

- **解决**：这里的解决方案和上面也是一样。

3. `MatchingGadgets.h`中的`CheckFillRateGadget`使用了`UnsafeMulGadget`不安全的乘法，因此在比较两个结果的大小时可能会`(fillAmountS * amountB * 100) < (fillAmountB * amountS * 101)`溢出。因为产生溢出正常校验可能会异常。

- **解决**：因为这里amount的值被限制在了`2^96-1`，因此这里的相乘并不会溢出。

4. RingSettlementCircuit.h`电路中的`TransformRingSettlementDataGadget`的`generate_r1cs_constraints`函数能够进一步优化为以下代码。

```c
struct Range
{
    unsigned int offset;
    unsigned int length;
};
std::vector<Range> ranges;
ranges.push_back({0, 40});   // orderA.orderID + orderB.orderID
ranges.push_back({40, 40});  // orderA.accountID + orderB.accountID
ranges.push_back({80, 16});  // orderA.tokenS + orderB.tokenS
ranges.push_back({96, 48});  // orderA.fillS + orderB.fillS
ranges.push_back({144, 8});  // orderA.data
ranges.push_back({152, 8});  // orderB.data
for(const Range& subRange : ranges)
{
    for (unsigned int i = 0; i < numRings; i++)
    {
        transformedData.add(subArray(data, i * ringSize + subRange.offset, subRange.length));
    }
}
```

5. 关于`MatchingGadgets`这个组件分析

|          caseC          | Taker     | Maker     |
| :---------------------: | --------- | --------- |
|         tokenS          | GTO       | WETH      |
|         tokenB          | WETH      | GTO       |
|         amountS         | 101       | 101       |
|         amountB         | 100       | 100       |
|          isBuy          | Yes       | no        |
|         fill.S          | 99        | 100       |
|         fill.B          | 100       | 99        |
| target price (WETH/GTO) | 1.01      | 100/101   |
| settle price (WETH/GTO) | 0.99      | 0.99      |
|     balance tokenS      | 2         | 1         |
|     balance tokenB      | 100       | 99        |
|    order sale price     | 100/101 W | 100/101 G |
|      settle price       | 100/99 W  | 99/100 G  |

对于taker：`100/99 > 100/101`，因此成交卖出价格高于订单卖出价格。

对于maker: `100/101 > 99/100`因此实际卖出价格是低于订单卖出价格。但是其依旧是能够成交，依赖于`(fillAmountS * amountB * 100) < (fillAmountB * amountS * 101)`

总结：订单匹配有益于taker。其余情况参考。 [link](https://github.com/Loopring/protocols/issues/623)

6. MatchingGadget中注释写错。`        takerFillB_lt_makerFillS(pb, takerFill.B, makerFill.S, NUM_BITS_AMOUNT, FMT(prefix, ".takerFill.B < makerFill.B")),`应该为`".takerFill.B < makerFill.S"`
7. 对于main.cpp中的`prove`和`generatekeys`两个动作同时进行可能存在问题。有两种逻辑：首先pk和vk不存在，在生成证明的时候首先generatorkeys，然后再proof，这种情况逻辑上说得通，但是不符合常理。当pk和vk不存在时，不允许执行prove操作。因为generatorkeys是trust setup，因此这一步骤需要多人参与并且只产生一次。因此这里更好采取后面一种使用方法。
8. 对于Data.h文件中，offchainWithdrawBlock类中，`startIndex`和`startHash`没有使用，貌似直接从`onchainWithdrawBlock`中复制过来😂。
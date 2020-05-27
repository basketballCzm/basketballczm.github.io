---
title: "Loopring交易所gadget分析"
date: 2020-04-06T10:51:50+08:00
draft: false
summary: "Loopring交易所gadget分析，loopring是以太坊上基于zkrollup的去中心化交易所，能够保证交易所作恶但不能获益。其零知识部分基于libsnark和ethsnark"
tags: [loopring,zkrollup,zksnark]
---

# loopringCircuit解读

## 基本Gadgets

### MathGadgets

在MathGadgets文件中各个组件Gadget类通过继承父类GadgetT来完成一些符合组件的编写工作。`generate_r1cs_constraints`理解是在电路板上添加约束，每一个转换的中间过程都应该将约束添加到原型板上。`generate_r1cs_witness`是为了生成一个证明，通过在原型板上设置公共变量的值，并且为私有变量设置见证值，这里体现出了各个变量之间的行为。一共有以下组件：

注意`ConstrainT`调用的是`r1cs_constrain`

| **Constants**                                                | **Constants stored in a VariableT for ease of use**          | **member variable**                                          | **generate_r1cs_witness**                                    | **constructor**                                              | **generate_r1cs_constraints**                                |
| ------------------------------------------------------------ | :----------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **DynamicVariableGadget**                                    | **Helper function that contains the history of all the values of a variable** | **std::vector<VariableT> variables;<br>bool allowGeneratingWitness;** | **assert(allowGeneratingWitness);<br>pb.val(variables.front()) = value;** | **variables.push_back(new variable);**                       | **none**                                                     |
| **TransferGadget**                                           | **Helper function for subadd to do transfers**               | **subadd_gadget subadd;**                                    | **subadd.generate_r1cs_witness();**                          | **from.add(subadd.X);<br>to.add(subadd.Y);**                 | **subadd.generate_r1cs_constraints();**                      |
| **UnsafeSubGadget**                                          | **A - B**                                                    | **VariableT value;<br>sub;<br>sum;**                         | **sum = value-sub;**                                         | **init VariableT**                                           | **ConstraintT(value - sub, FieldT::one(), sum)**             |
| **UnsafeAddGadget**                                          | **A + B**                                                    | **VariableT value;<br>add;<br>sum**                          | **sum = value+add;**                                         | **init VariableT**                                           | **ConstraintT(value + add, FieldT::one(), sum)**             |
| **UnsafeMulGadget**                                          | **A * B**                                                    | **VariableT valueA;<br>valueB;<br>product;**                 | **product = valueA*valueB**                                  | **init VariableT**                                           | **ConstraintT(valueA, valueB, product)**                     |
| **AddGadget**                                                | **A + B = sum with A, B and sum < 2^n**                      | **UnsafeAddGadget unsafeAdd;<br>libsnark::Undual_variable_gadget<FieldT> rangeCheck;(libsnark gadgets)** | **unsafeAdd.generate_r1cs_witness();<br>rangeCheck.generate_r1cs_witness_from_packed();** | **init unsafeAdd, rangeCheck<br>assert(n+1 <= NUM_BITS_FIELD_CAPACITY(253))** | **unsafeAdd.generate_r1cs_constraints();<br>rangeCheck.generate_r1cs_constraints(true);** |
| **TernaryGadget**                                            | **b ? A : B**                                                | **VariableT b;<br>x;<br>y;<br>selected;**                    | **select = (b == 1)?x:y;**                                   | **init VariableT**                                           | **param:bool enforceBitness = true<br>if(enforceBitness){generate_boolean_r1cs_constraint\<ethsnarks::FieldT>(pb, b, FMT(annotation_prefix, ".bitness"));}<br>ConstraintT(b, y - x, y - selected);** |
| **LeqGadget**                                                | **(A <(=) B)**  只能保证在n <= 253-1的情况下成立             | **VariableT _It;<br>__leq;<br>libsnark::comparison_gadget comparison;** | **comparison.generate_r1cs_witness();**                      | **init VariableT<br>assert(n <= NUM_BITS_FIELD_CAPACITY - 1);** | **comparison.generate_r1cs_constraints();**                  |
| **AndGadget**                                                | **(input[0] && input[1] && ...) (all inputs need to be boolean)** | **vector<VariableT> inputs;<br>vector<VariableT> results;**  | **results[0] = inputs[0]*inputs[1];<br>for(i = 2; i < input.size; i++ ){results[i-1] = results[i-2]\*inputs[i];}** | **init inputs;<br>assert(inputs.size > 1);<br>for(i = 1; i < inputs.size; i++){results.push(VariableT);}** | **ConstraintT(inputs[0], inputs[1], results[0]);<br>for(i = 2; i < inputs.size; i++){ConstraintT(inputs[i], results[i-2], results[i-1]);}** |
| **OrGadget**                                                 | **(input[0] \|\| input[1] \|\| ...) (all inputs need to be boolean)** | **vector<VariableT> inputs;<br/>vector<VariableT> results;** | **results[0] =1-(1-inputs[0])*(1-inputs[1]);<br/>for(i = 2; i < inputs.size; i++ ){results[i-1] = 1-(1-results[i-2])\*(1-inputs[i]);}** | **init inputs;<br/>assert(inputs.size > 1);<br/>for(i = 1; i < inputs.size; i++){results.push(VariableT);}** | **ConstraintT(1-inputs[0], 1-inputs[1], 1-results[0]);<br/>for(i = 2; i < inputs.size; i++){ConstraintT(1-inputs[i], 1-results[i-2], 1-results[i-1]);}** |
| **NotGadget**                                                | **!A (A needs to be boolean)**                               | **VariableT A;<br>VariableT _not;**                          | **_not = 1- A**                                              | **init VariableT;**                                          | **ConstraintT(1 - A, 1, _not)**                              |
| **XorArrayGadget**                                           | **A[i] ^ B[i]**                                              | **VariableArrayT A;<br>VariableArrayT B;<br>VariableArrayT C;** | **for(i = 0; i < C.size; i++){C[i] = A[i] +B[i] - ((A[i] == 1 && B[i] == 1)? 2:0);}** | **assert(A.size == B.size);**                                | **for(i = 0; i < C.size(); i++){ConstraintT(2*A[i], B[i], A[i]+B[i]-C[i]);}** |
| **EqualGadget**                                              | **(A == B)**                                                 | **UnsafeSubGadget difference;<br>IsNonZero isNonZeroDifference;<br>NotGadget isZeroDifference;** | **difference.generate_r1cs_witness();<br>difference.generate_r1cs_witness();<br>isZeroDifference.generate_r1cs_witness();** | **init difference, isNonZeroDifference<br>difference(pb, A, B, FMT(prefix, ".difference"));<br>isNonZeroDifference(pb, difference.result(), FMT(prefix, ".isNonZeroDifference"))<br>isZeroDifference(pb, isNonZeroDifference.result(), FMT(prefix, ".isZeroDifference"))<br>** | **difference.generate_r1cs_constraints();<br>isNonZeroDifference.generate_r1cs_constraints();<br>isZeroDifference.generate_r1cs_constraints();<br>** |
| **RequireEqualGadget**                                       | **require(A == B)**                                          | **VariableT A;<br>VariableT B;**                             | **none**                                                     | **init VariableT;**                                          | **ConstraintT(A, FieldT::one(), B);**                        |
| **RequireZeroAorBGadget**                                    | **require(A == 0 \|\| B == 0)**                              | **VariableT A;<br/>VariableT B;**                            | **none**                                                     | **init VariableT;**                                          | **ConstraintT(A, B, FieldT::zero());**                       |
| **RequireNotZeroGadget**                                     | **require(A != 0)**                                          | **VariableT A;<br/>VariableT A_inv;**                        | **A_inv = !A;**                                              | **init VariableT;**                                          | **ConstraintT(A, A_inv, FieldT::one());**                    |
| **RequireNotEqualGadget**                                    | **require(A != B)**                                          | **VariableT A;<br>VariableT B;<br>VariableT difference;<br>RequireNotZeroGadget notZero;** | **difference = A-B;<br>notZero.generate_r1cs_witness();**    | **init VariableT;**                                          | **ConstraintT(A - B, FieldT::one(), difference);<br>notZero.generate_r1cs_constraints();** |
| **MinGadget**                                                | **min(A, B)**                                                | **LeqGadget A_lt_B;<br>TernaryGadget minimum;**              | **A_lt_B.generate_r1cs_witness();<br>minimum.generate_r1cs_witness();** | **init VariableT;**                                          | **A_lt_B.generate_r1cs_constraints();<br>minimum.generate_r1cs_constraints();** |
| **RequireLeqGadget**                                         | **require(A <= B)**                                          | **LeqGadget leqGadget;**                                     | **leqGadget.generate_r1cs_witness();**                       | **init VariableT;**                                          | **leqGadget.generate_r1cs_constraints();<br>ConstraintT(leqGadget.leq(), FieldT::one(), FieldT::one());** |
| **RequireLtGadget**                                          | **require(A < B)**                                           | **LeqGadget leqGadget;**                                     | **leqGadget.generate_r1cs_witness();**                       | **init VariableT;**                                          | **leqGadget.generate_r1cs_constraints();<br>ConstraintT(leqGadget.lt(), FieldT::one(), FieldT::one());** |
| **MulDivGadget**                                             | **(value * numerator) = product**                            |                                                              | **denominator_notZero.generate_r1cs_witness();<br>product.generate_r1cs_witness();<br>if (denominator != Field::zero()){quotient = product/denominator;}else{quotient=Field::zero();}<br>remainder.packed = product.result - denominator*quotient;<br>remainder.generate_r1cs_witness_from_packed();<br>remainder_lt_denominator.generate_r1cs_witness();** | **init VariableT;<br>assert(numBitsValue + numBitsNumerator <= NUM_BITS_FIELD_CAPACITY);** | **denominator_notZero.generate_r1cs_constraints();<br>product.generate_r1cs_constraints();<br>ConstraintT(denominator, quotient, product.result() - remainder.packed)<br>remainder.generate_r1cs_constraints(true);<br>remainder_lt_denominator.generate_r1cs_constraints();** |
|                                                              | **product / denominator = quotient**                         |                                                              |                                                              |                                                              |                                                              |
|                                                              | **product % denominator = remainder**                        |                                                              |                                                              |                                                              |                                                              |
| **RequireAccuracyGadget**                                    | **_accuracy.numerator / _accuracy.denominator <=  value / original<br>original _accuracy.numerator <= value * _accuracy.denominator<br>We have to make sure there are no overflows and the value is <= the original value (so a user never spends more) so we also check <br>value <= original<br>value < 2^maxNumBits** | 用来校验float类型在电路编码的精度损失，精确度损失在不同的场景有不同的需求 [link](https://github.com/Loopring/protocols/pull/156)。要求给定数的精确度大于`_accuracy`的值 |                                                              |                                                              |                                                              |
| **EdDSA_HashRAM_Poseidon_gadget**                            | **Poseidon 哈希函数，ethsnark已经进行了相应的封装，传入相应参数生成Poseidon 哈希** | **Poseidon_gadget_T<6, 1, 6, 52, 5, 1> m_hash_RAM;<br>libsnark::dual_variable_gadget<FieldT> hash;** | **m_hash_RAM.generate_r1cs_witness();<br>hash.bits.fill_with_bits_of_field_element(pb, pb.val(m_hash_RAM.result()));<br>hash.generate_r1cs_witness_from_bits();** | **init VariableT;**                                          | **m_hash_RAM.generate_r1cs_constraints();<br>hash.generate_r1cs_constraints(true);** |
| **EdDSA_Poseidon**                                           | **使用poseidon哈希函数的EdDSA数字签名函数**                  |                                                              | **generate_r1cs_witness....<br>**                            | **init VariableT;**                                          | **generate_r1cs_constraints...<br>ConstraintT(m_lhs.result_x(), FieldT::one(), m_rhs.result_x());<br>ConstraintT(m_lhs.result_y(), FieldT::one(), m_rhs.result_y());** |
| **SignatureVerifier**                                        | **Verifies a signature hashed with Poseidon**                | **const jubjub::VariablePointT sig_R;<br>const VariableArrayT sig_s;<br>const VariableT sig_m;<br>EdDSA_Poseidon signatureVerifier;** | **pb.val(sig_R.x) = sig.R.x;<br>pb.val(sig_R.y) = sig.R.y;<br>sig_s.fill_with_bits_of_field_element(pb, sig.s);<br>signatureVerifier.generate_r1cs_witness();** | **init VariableT;**                                          | **signatureVerifier.generate_r1cs_constraints();**           |
| **PublicDataGadget**                                         | **Public data helper class.<br>将所有链上的数据(public data) 哈希成一个固定值，因此固定了每个电路证明的输入** | **const VariableT publicInput;<br>std::vector<VariableArrayT> publicDataBits;<br>std::unique_ptr<sha256_many> hasher;<br>std::unique_ptr\<libsnark::dual_variable_gadget<FieldT>> calculatedHash;<br>** | **hasher->generate_r1cs_witness();<br>calculatedHash->generate_r1cs_witness_from_bits();<br>pb.val(publicInput) = pb.val(calculatedHash->packed);** | **init VariableT;**                                          | **hasher->generate_r1cs_constraints();<br>calculatedHash->generate_r1cs_constraints(false);<br>ConstraintT(calculatedHash->packed, FieldT::one(), publicInput);** |
| **FloatGadget**                                              | **在电路中使用一种明确的编码格式来编码float类型**            |                                                              |                                                              |                                                              |                                                              |
| **LabelHasher**                                              | **用于固定label标签中的内容**                                |                                                              |                                                              |                                                              |                                                              |
| **DualVariableGadget : public libsnark::dual_variable_gadget<FieldT>** | **相当于是一个动态数组，里面有单个，变量类型是VariableT**    |                                                              |                                                              |                                                              |                                                              |

### MerkleTree

下面gadget主要实现了MerkleTree根节点的计算 ，以及查看根节点的值是否和预期结果相互匹配。

| Merkle_path_select_4            | 通过bit位选择四个叶子节点的顺序，然后按照指定顺序进行hash | OrGadget bit0_or_bit1;<br>AndGadget bit0_and_bit1;<br>TernaryGadget child0-3 | generate_r1cs_witness                                        | init bit0_or_bit1, bit0_and_bit1;<br>init child0-3;          | generate_r1cs_constraints                                    |
| ------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **merkle_path_compute_4**       | **按照顺序从底向上计算hash，该merkletree是一颗四叉树**    | **const size_t m_depth;<br>const VariableArrayT m_address_bits;<br>const VariableT m_leaf;<br>const VariableArrayT m_path;<br>std::vector<merkle_path_selector_4> m_selectors;<br>std::vector<HashT> m_hashers;** | **for(i = 0; i < m_hashers.size; i++){m_selectors[i].generate_r1cs_witness();<br>m_hashers[i].generate_r1cs_witness();}** | **assert(in_depth > 0);<br/>assert(in_address_bits.size == in_depth*2);<br/>for(i = 0; i < m_depth; i++){merkle_path_selector_4();}<br/>m_hashers.push_back(HashT(var_array()))** | **for(i = 0; i < m_hashers.size; i++){m_selectors[i].generate_r1cs_constraints();<br/>m_hashers[i].generate_r1cs_constraints();}** |
| **merkle_path_authenticator_4** | **MerkleTree验证器，验证根节点和最后的结果值**            | **const VariableT m_expected_root;**                         | **none**                                                     | **merkle_path_compute_4<HashT>::meukle_path_compute_4(in_pb, in_depth, in_address_bits, in_leaf, in_path, in_annotation_prefix);** | **merkle_path_compute_4<HashT>::generate_r1cs_constraints();<br>ConstraintT(this->result(), 1, m_expected_root);//match the root_node and m_expected_root** |





下面分析有关业务的gadgets：

### AccountGadgets

**AccountGadgets**主要有四个gadgets，`UpdateAccountGadget`依赖于`AccountGadget`，`UpdateBalanceGadget`依赖于`BalanceGadget`。主要用于更新`merkletree`上的余额，更新了余额后，`balanceroot`的值会发生变化，也会导致`Accountroot`发生变化。因此需要上述`gadgets`来更新`merkletree`。

**UpdateAccountGadget**首先分析下这个`gadgets`，首先更新叶子节点前后的变化，初始化`proof`证明计算`merkleroot`过程中的路径。该值是由`witness`传入，`proofVerifierBefore`计算之前的`merkleroot`，并且进行验证。`rootCalculatorAfter`计算之后的`merkleroot`。

**UpdateBalanceGadget**也是和上面同理。

**witness**

```c
const Proof& _proof
```

### orderGagets

**orderGagets**主要进行一些订单有效的校验，比如`feeBips`和`rebateBips`两者中有一个要为0；`feeBips`要小于或等于最大费用；订单中转换的两种币种的`tokenID`要不相同。并且校验了之前`tradeHistroyBefore`的`orderID`和当前的`orderID`之间的不同，然后做出不同的判断，如果`order.orderID > tradeHistory.orderID`订单会覆盖历史订单；如果`order.orderID == tradeHistory.orderID`使用历史订单，这里的一种情况可以用于当历史订单的时间戳失效后，通过相同的`orderID`来增加订单的时间；如果`order.orderID < tradeHistory.orderID`则订单会被取消。最后就是验证订单拥有者的有效性，即对订单的输入进行hash，然后对hash结果进行签名。最后使用公钥验证签名结果的有效性。

**witness**

```c
const Order& order, 
const Account& account, 
const BalanceLeaf& balanceLeafS, 
const BalanceLeaf& balanceLeafB, 
const TradeHistoryLeaf& tradeHistoryLeaf
```

### MatchingGadgets

**CheckFillRateGadget**这个组件主要是用来检测订单的填充率，当订单的填充率达到一定的比例时候就默认订单已经填充满了。计算公式为`(fillAmountS/fillAmountB) * 100 < (amountS/amountB) * 101`，化简之后为`(fillAmountS * amountB * 100) < (fillAmountB * amountS * 101)`，如果`fillAmount`的值满足这个等式就说明订单填充率已经完成。

**这里修改了测试，发现当amount的最大值为`296`，当超过这个值是，电路的`UnsafeMulGadget`计算就会发生溢出，导致订单填充率校验结果为无效结果。**

**CheckValidGadget**依赖于`CheckFillRateGadget`的组件，主要用于检测订单是否被正确的填满。主要的检测有`fillAmounts`小于`Amounts`； 检查`allOrNone`标识为0，应为有该标识表示订单可以被部分填充。检查时间戳是否在有效区间，检查填充率是否满足，检查`FillAmountS`和`FillAmountB`同时不能为0，这里的`FillAmount`主要是本次订单匹配需要`fill`的值，因为是已经匹配过的值，所以这里的值不能为0。

**FeeCalculatorGadget**(**不依赖别的gadgets**)计算协议费，费用，回扣。在不同的电路中，订单需要支付不同项的费用。费用都是按照`protocolFeeBips`，`feeBips`，`rebateBips`的值来进行计算。

**MaxFillAmountsGadget**(**不依赖别的gadgets**)计算订单的`remainingS`和`remainingB`的最大填充数量。首先计算买入币种还有多少没有买，然后根据`amountS/amountB`的比例算出还有多少币没有卖，然后分别得出填充数量`fillAmountS`和`fillAmountB`的值。显示根据剩余买入的数目计算剩余卖出的数目，校验剩余卖出的数目是否小于账户余额，然后根据其最小值计算其实际剩余买入数目的最大值。

**TakerMakerMatchingGadget**(**不依赖别的gadgets**)计算两个订单结算后的填充数量。首先比较toker要买的币数量和maker要卖的币数量的大小，以多的token为基础计算token少的一方需要卖出和买入的token数目。最后得出`makerFillS`，`makerFillB`，`takerFillS`，`takerFillB`和订单是否匹配的结果`bMatchable`。

**MatchingGadget ** **(依赖于`TakerMakerMatchingGadget`的组件)**以上面的`gadgets`为基础，查看两个订单是否匹配，并且通过`buy`标识来指定谁是`taker`，谁是`maker`，修改`taker`的`fillA.S`变量。最后得出此次匹配的最终匹配结果和`fillA_S`，`fillA_B`，`fillB_S`，`fillB_B`的最终结果。**在这个计算的过程中因为整除之后存在余数，taker卖出tokenS的价格高于自己定的最低价格。maker买入的价格低于自己定的最高价格，因此taker会获益，但是toker的手续费根据DESIGN.md也会高于maker。出现上面情况的主要原因是MulDivGadget除法去掉余数后产生的误差**

**OrderMatchingGadget**  **(依赖于`MatchingGadget` `MaxFillAmountsGadget` `CheckValidGadgets`的组件)**这个`gadgets`基于上面的`gadgets`，在匹配之前检查了两个订单的买入和卖出tokenID的值是否相互匹配，这个相等的判断是写入到电路中的，`MatchingGadgets`匹配两个订单后，`checkValid`需要获取每个订单的匹配结果的填充值`matchingGadget.getFillA_S()`，`matchingGadget.getFillA_B()`，从而检查出两个订单匹配的有效性。

------

## Circuit

### DepositCircuit

#### 1. 流程

1. 用户发送Deposit请求到ETH主链的XXX合约。
2. 中继监听XXX合约，收到用户的Deposit请求。
3. 中继组装Deposit请求生成Deposit请求的block进行提交，调用XXX合约的commitBlock。
4. 中继调用电路部分的block的prove。
5. 中继调用XXX合约的VerifiyBlock。

#### 2 Prove Input

Prove的input包含block的信息和deposit的信息：

**Depositblock的信息**

```c++
class DepositBlock
{
public:
    ethsnarks::FieldT exchangeID;

    ethsnarks::FieldT merkleRootBefore;
    ethsnarks::FieldT merkleRootAfter;
    
    // # Accumulated hash of starting onchain request
    libff::bigint<libff::alt_bn128_r_limbs> startHash;
    
    // # Index of onchain request
    ethsnarks::FieldT startIndex;
    // # Number of onchain requests processed in this block
    ethsnarks::FieldT count;

    std::vector<Loopring::Deposit> deposits;
};
```

每个**Deposit**的信息：

```c++
class Deposit
{
public:
    ethsnarks::FieldT amount;
    BalanceUpdate balanceUpdate;
    AccountUpdate accountUpdate;
};

class BalanceUpdate
{
public:
    ethsnarks::FieldT tokenID;
    Proof proof;
    ethsnarks::FieldT rootBefore;
    ethsnarks::FieldT rootAfter;
    BalanceLeaf before;
    BalanceLeaf after;
};

class AccountUpdate
{
public:
    ethsnarks::FieldT accountID;
    Proof proof;
    ethsnarks::FieldT rootBefore;
    ethsnarks::FieldT rootAfter;
    Account before;
    Account after;
};

class Account
{
public:
    ethsnarks::jubjub::EdwardsPoint publicKey;
    ethsnarks::FieldT nonce;
    ethsnarks::FieldT balancesRoot;
};

class BalanceUpdate
{
public:
    ethsnarks::FieldT tokenID;
    Proof proof;
    ethsnarks::FieldT rootBefore;
    ethsnarks::FieldT rootAfter;
    BalanceLeaf before;
    BalanceLeaf after;
};

class BalanceLeaf
{
public:
    ethsnarks::FieldT balance;
    ethsnarks::FieldT tradingHistoryRoot;
};
```

![1572250383687](C:\Users\czm18\AppData\Roaming\Typora\typora-user-images\1572250383687.png)

对于上面的图只要稍加修改，修改最左侧输入为`DepositBlock`数据，修改`onchainwithdraw`为`Deposit`中的数据即可。

#### 3 Circuit

电路会计算一个如下的sha256 hash，然后verifyBlock的时候输入hash会与这个hash进行验证，一致则表示成功。

```c++
// Public data
publicData.add(exchangeID.bits);
publicData.add(merkleRootBefore.bits);
publicData.add(merkleRootAfter.bits);
// 起始desposit的hash值
publicData.add(reverse(depositBlockHashStart.bits));
// 每个despoit操作的hash叠加，因为是以太坊的大端存储格式，因此要翻转
publicData.add(reverse(hashers.back().result().bits));
publicData.add(startIndex.bits);
publicData.add(count.bits);
```

**电路运行流程**

```c++
1. 计算存储金额的上限，如果存储金额后总金额大于最大金额容量2**96-1，则存储后金额为最大金额。
2. updateBalance
3. updateAccount
```

### OrderCancellationCircuit

#### 1 流程

1. 用户发送Cancel请求到中继。
2. 中继接收用户Cancel请求，组装成deposit请求生成deposit的block提交到ETH主链，调用XXX合约的commitBlock。
3. 中继调用电路部分生成block的prove。
4. 中继调用XXX合约的verifyBlock。

#### 2 Prove input

**block的信息**

```c++
CancellationBlock
{
    exchangeID: number,

    merkleRootBefore: string,
    merkleRootAfter: string,

    # Operator:
    # Account update data
    operatorAccountID: number,
    accountUpdate_O: AccountUpdate,

    cancels: Cancellation[],
}
```

**每个OrderCancellation的信息**

```c++
Cancellation
{
    # Offchain request data
    fee: string,
    label: string,
    signature: Signature,

    # User:
    # Trade history update data
    # Balance update data for tokenF and token withdrawn
    # Account update data
    tradeHistoryUpdate_A: TradeHistoryUpdate;
    balanceUpdateT_A: BalanceUpdate,
    balanceUpdateF_A: BalanceUpdate;
    accountUpdate_A: AccountUpdate;

    # Operator:
    # Balance update data for tokenF
    balanceUpdateF_O: BalanceUpdate,
}
```

数据状态更新参考上面格式，替换成相应的`Cancellation`和`OrderCancellationBlock`中的信息即可。

#### 3 Circuit

#### 3.1  计算publicDataHash

```c++
// Public data
publicData.add(exchangeID.bits);
publicData.add(merkleRootBefore.bits);
publicData.add(merkleRootAfter.bits);
publicData.add(labelHasher->result()->bits);
if (onchainDataAvailability)
{
    publicData.add(constants.padding_0000);
    publicData.add(operatorAccountID.bits);
    for (const OrderCancellationGadget& cancel : cancels)
    {
        publicData.add(cancel.getPublicData());
    }
}
```

```python
if onchainDataAvailability == True:
    for i in range(0, blocksize):
        orderCancellations += orderCancelltions.getpublicData();
    publicDataHash = sha_256(exchangeID + merkleRootBefore + merkleRootAfter + constant.padding_0000 + operatorAccountID + orderCancellations);
else:
    publicDataHash = sha_256(exchangeID + merkleRootBefore + merkleRootAfter + labelHasher;
```





上面的hash值就是`CancellationBlock`中的每个值，加上填充，加上每个`cancel.getPublicData()`的值。

#### 3.2 OrderCancellationGadget

1. 校验`oldOrderId`小于等于`newOrderId`。
2. 确保`fFee`和`fee`在精度范围内相等。`Float16Encoding`。
3. 将订单的填充数目`filled_after`置为0。
4. 用户向operator支付取消交易费用。
5. 将用户的随机数加1。
6. 更新账户A`TradeHistory`算出新的merkleroot。
7. 账户A`TradeHistroy`的merkleroot改变之后，需要更新`updateBalanceT_A`和`updateBalanceF_A`。
8. 账户A`balance`的merkleroot发生变化，A的`Accountroot`也需要更新。`updateAccount_A`。
9. 更新操作员的余额信息。`updateBalanceF_O`。
10. 更新操作员的账户信息。`updateAccount_O`。

### RingSettlement(off-chain)

#### 1 流程

1. 用户发送match order请求到中继。
2. 中继收到用户的match order请求，在线下匹配订单，并将订单两两组成`RingSettlement`，然后组成`RingSettlementBlock`提交到ETH主链，调用XXX合约commitblock。
3. 中继调用电路部分生成block的prove。
4. 中继调用XXX合约验证verifyBlock。

#### 2 Prove Input

Prove的Input包含block的信息和每个`RingSettlement`的信息。

**block的信息**

```c++
class RingSettlementBlock
{
public:
    // 交易所的ID
    ethsnarks::FieldT exchangeID;

    ethsnarks::FieldT merkleRootBefore;
    ethsnarks::FieldT merkleRootAfter;
    // timestamp used in this block
    ethsnarks::FieldT timestamp;
    
    // Protocol fees used in this block
    ethsnarks::FieldT protocolTakerFeeBips;
    ethsnarks::FieldT protocolMakerFeeBips;
    
    // operator sign
    Signature signature;
    // protocol fee account update data 
    AccountUpdate accountUpdate_P;
    
    ethsnarks::FieldT operatorAccountID;
    // operator account update
    AccountUpdate accountUpdate_O;

    std::vector<Loopring::RingSettlement> ringSettlements;
};
```

**RingSettlement的信息**

```c++
class RingSettlement
{
public:
    Ring ring;
    
    // the starting merkleroot
    ethsnarks::FieldT accountsMerkleRoot;
    
    // update tradeHistroy
    TradeHistoryUpdate tradeHistoryUpdate_A;
    TradeHistoryUpdate tradeHistoryUpdate_B;
    
    // update accountA information
    BalanceUpdate balanceUpdateS_A;
    BalanceUpdate balanceUpdateB_A;
    AccountUpdate accountUpdate_A;
    
    // update accountB information
    BalanceUpdate balanceUpdateS_B;
    BalanceUpdate balanceUpdateB_B;
    AccountUpdate accountUpdate_B;
    
    // balance update data for protocol fee payment
    BalanceUpdate balanceUpdateA_P;
    BalanceUpdate balanceUpdateB_P;
    
    // balance update data for the operator(fee/rebate/protocol fee)
    BalanceUpdate balanceUpdateA_O;
    BalanceUpdate balanceUpdateB_O;
};

Order
{
    exchangeID: number,
    orderID: number,
    accountID: number,
    tokenS: number,
    tokenB: number,
    amountS: string,
    amountB: string,
    allOrNone: number,
    validSince: number,
    validUntil: number,
    maxFeeBips: number,
    buy: number,
    label: string,

    feeBips: number,
    rebateBips: number,

    signature: Signature,
}

class Ring
{
    orderA: Order,
    orderB: Order,
}
```

#### 3 Circuit

#### 3.1 计算publicDataHash

```python
if onchainDataAvailability == True:
    for i in range(0, blocksize):
        transformaData += ringSettlement.getpublicData();
    publicDataHash = sha_256(exchangeID + merkleRootBefore + merkleRootAfter + timestamp + protocolTakerFeeBips + protocolMakerFeeBips + constant.padding_0000 + operatorAccountID + transformData);
else:
    publicDataHash = sha_256(exchangeID + merkleRootBefore + merkleRootAfter + timestamp + protocolTakerFeeBips + protocolMakerFeeBips);
```

#### 3.2RingSettlementGadget

1. 使用`OrderMatchingGadget`匹配两个订单，获得订单匹配结果，如果订单能够匹配上，获取本次匹配的填充量。
2. 校验填充值`fillS_A`使用`floatGadget`编码的精度损失，精度损失需要在`Float24Accuracy`范围内。
3. 根据`buy`标识识别谁是`maker`和谁是`taker`，然后计算实际`taker`的买入值和`maker`的卖出值。
4. 计算A和B订单填充之后的值。
5. 计算订单A和B的协议费用。
6. 订单A和B都减少卖出的`fillS_value`。
7. 用户给`operator`支付总费用。
8. `operator`分别给订单AB回扣和支付`protocol fee`。
9. 更新A,B,protocol account, operator的merkletree状态。

**onchainWithdraw和offchainWithdraw在**



现在withdraw有两种⽅式：onchain withdraw和offchain withdraw。
两个各有优劣：
1 onchain withdraw缺点是⽐较慢，要先经过以太主⽹确认才会到中继进⾏处理；优点是可以确保被处
理（合约设置超过15分钟就要被优先处理）
2 offchain withdraw的优点是快，直接发到中继进⾏处理；缺点是中继可以选择性的不处理；

### onchainWithdraw

1 ⽤户发送withdraw请求到ETH主链的Exchange合约
2 中继监听Exchange合约收到⽤户的withdraw请求
3 中继组装withdraw请求⽣成withdraw的block提交，调⽤Exchange的commitBlock
4 中继调⽤电路部分⽣成block的prove
5 中继调⽤Exchange的verifyBlock

### offchainWithdraw

1 ⽤户发送withdraw请求到中继
2 中继收到⽤户withdraw请求。组装withdraw请求⽣成withdraw的block提交到ETH主链，调⽤Exchange的commitBlock
3 中继调⽤电路部分⽣成block的prove
4 中继调⽤Exchange合约的verifyBlock
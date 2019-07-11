---
title: evm源码分析第三篇
date: 2019-07-11 09:35:08
categories:
	- ethereum
	- evm
tags:
	- ethereum
	- evm
author: ClimbYang
---

上一篇我们介绍了创建合约的交易是如何执行的，这一篇我们分析普通交易是如何在evm里面被执行的

<!-- more -->

```go
if contractCreation {
   ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
} else {
   // Increment the nonce for the next transaction
   st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
   ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
}
```

如果是普通的交易，则会把调用方nonce+1 然后 执行evm.call。

evm.call方法和evm.create方法大致相同，我们来说说不一样的地方。
call方法调用的是一个存在的合约地址的合约，所以不用创建合约账户。如果call方法发现本地也没有合约接收方的账户，则需要创建一个接收方的账户，并更新本地状态数据库。

call 方法或者create方法执行完毕，则会调用st.refundGas()方法计算需要退还的gas.

在拜占庭版本的黄皮书规范中，只有`SELFDESTRUCT`和`SSTORE`(当将一个非0位置上的数置为0时候）这两个`opcode`会触发refund。单个交易执行结束时，累计refund的gas量不能超过当前交易执行上下文的消耗的`gasUsed`的一半。

```go
func (st *StateTransition) refundGas() {
   // Apply refund counter, capped to half of the used gas.
   refund := st.gasUsed() / 2
   if refund > st.state.GetRefund() {
      refund = st.state.GetRefund()
   }
   //st.gs 代表交易完成还剩下的gas值
   st.gas += refund

   // Return ETH for remaining gas, exchanged at the original rate.
   remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
   st.state.AddBalance(st.msg.From(), remaining)
	
   //gas pool 默认初始值为区块gas limit, 每执行一次交易，将会从gas pool中减去st.msg.Gas(), 
   //既然产生了remaining， 那肯定也需要将这些值加回到原来的gas pool.
   // Also return remaining gas to the block gas counter so it is
   // available for the next transaction.
   st.gp.AddGas(st.gas)
}
```

最后实际退回给交易发起者的gas为：remaining = gas执行交易剩余 + 累计refund  

这里引用一篇博客的介绍这个[gas refund 机制](https://blog.csdn.net/KeenCryp/article/details/86586132)

 最后计算合约产生的gas总数，加入到矿工账户，作为矿工收入。

最后我们回到第一节的ApplyTransaction（）函数上，这时候还有最后一步就是更新state root了。

接下来填充receipt的各项值，然后返回。

总结一下 交易流程到EVM执行大致流程如下

```
/*
The State Transitioning Model

A state transition is a change made when a transaction is applied to the current world state
The state transitioning model does all the necessary work to work out a valid new state root.

1) Nonce handling
2) Pre pay gas
3) Create a new state object if the recipient is \0*32
4) Value transfer
== If contract creation ==
  4a) Attempt to run transaction data
  4b) If valid, use result as code for the new state object
== end ==
5) Run Script section
6) Derive new state root
*/
```






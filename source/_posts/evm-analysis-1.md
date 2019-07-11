---
title: evm源码分析第一篇
date: 2019-07-11 09:35:08
categories:
	- ethereum
	- evm
tags:
	- ethereum
	- evm
author: ClimbYang
---



evm源码分析分为3篇去讲解，所有的代码解析基于以太坊`go-ethereum-1.8.23-release`

<!-- more -->

### 源码结构

runtime 包下的文件在实际运行的geth客户端中并没有被调用到，只是作为开发人员测试使用。

- `core/vm/runtime/env.go`        设置evm运行环境，并返回新的evm对象  
- `core/vm/runtime/fuzz.go`       fuzz使得开发者可以随机测试evm代码，详见`go-fuzz`工具
- `core/vm/runtime/runtime.go`    设置evm运行环境，并执行相应的evm代码

下面的文件为实际的使用到的evm的代码文件 

- `core/vm/analysis.go`                                      分析指令跳转目标 
- `core/vm/common.go`                                          存放常用工具方法
- `core/vm/contract.go`                                       合约数据结构
- `core/vm/contracts.go`                                     存放预编译好的合约
- `core/vm/errors.go `                                           定义一些常见错误               
- `core/vm/evm.go`                                                  evm对于解释器提供的一些操作接口
- `core/vm/gas.go`                                                  计算一级指令耗费gas
- `core/vm/gas_table.go`                                      各种操作的gas消耗计算表
- `core/vm/gen_structlog.go`                             生成structlog的 序列化json和反序列化方法
- `core/vm/instructions.go`                               所有指令集的实现函数        
- `core/vm/interface.go`                                      定义常用操作接口
- `core/vm/interpreter.go`                                  evm 指令解释器  
- `core/vm/intpool.go`                                          常量池
- `core/vm/jump_table.go`                                    指令跳转表
- `core/vm/logger.go`                                             状态日志
- `core/vm/logger_json.go`                                  json形式日志
- `core/vm/memory.go`                                             evm 可操作内存
- `core/vm/memory_table.go`                                 evm内存操作表，衡量一些操作耗费内存大小
- `core/vm/opcodes.go`                                           定义操作码的名称和编号
- `core/vm/stacks.go`                                             evm栈操作
- `core/vm/stack_table.go`                                   evm栈验证函数 

上述对要介绍的evm代码进行了一个简单的介绍，接下来将详细的分析其中的代码。

### 从交易提交说起

我们将会从一个交易的提交开始讲起，当一个geth客户端接收到其他节点提交的交易后，它会首先将这笔交易提交给evm进行处理。

```go
func (w *worker) commitTransaction(tx *types.Transaction, coinbase common.Address) ([]*types.Log, error) {
   snap := w.current.state.Snapshot()

   receipt, _, err := core.ApplyTransaction(w.config, w.chain, &coinbase, w.current.gasPool, w.current.state, w.current.header, tx, &w.current.header.GasUsed, *w.chain.GetVMConfig())
   if err != nil {
      w.current.state.RevertToSnapshot(snap)
      return nil, err
   }
   w.current.txs = append(w.current.txs, tx)
   w.current.receipts = append(w.current.receipts, receipt)

   return receipt.Logs, nil
}
```

一笔交易提交到EVM前的主要过程就是上述代码所描述的

1. 创建当前stateDB的snapshot， 创建snapshot其实就是将leveldb的revisionId自增1，然后将这个revisionId加入到revisionId列表里，然后返回创建的id。
2. 将交易发送到evm，执行交易， 这步骤后面会重点分析，这个就是我们这次文章主要分析的重点EVM的执行交易过程。
3. 判断执行结果是否出错，如果出错，则回滚snapshot。 首先找到在revisionId列表里面找到需要回滚的revisionId， 然后将此revisionId里面的所有snapshot依次回滚。
4. 将当前交易加入到交易列表
5. 将交易收据加入到交易收据列表

接下来我们主要分析ApplyTransaction函数

```go
// ApplyTransaction 尝试将一次交易的执行写入数据库，并且为执行环境准备输入参数。它返回交易的收据。
// 如果gasg 使用完，或者交易执行出现error，则表示该块无效。
func ApplyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, uint64, error) {
   msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
   if err != nil {
      return nil, 0, err
   }
   // 创建一个新的evm执行上下文环境
   context := NewEVMContext(msg, header, bc, author)
   // 创建一个包含所有相关信息的新环境， 包括事务和调用机制
   vmenv := vm.NewEVM(context, statedb, config, cfg)
   // 将事务应用于当前状态(包含在env中)
   _, gas, failed, err := ApplyMessage(vmenv, msg, gp)
   if err != nil {
      return nil, 0, err
   }
   // Update the state with pending changes
   var root []byte
   if config.IsByzantium(header.Number) {
      statedb.Finalise(true)
   } else {
      root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
   }
   *usedGas += gas

   //创建一个新的收据为这笔交易，存储中间状态根和gas使用情况
   // based on the eip phase, we're passing whether the root touch-delete accounts.
   receipt := types.NewReceipt(root, failed, *usedGas)
   receipt.TxHash = tx.Hash()
   receipt.GasUsed = gas
   //如果这笔交易是创建一个合约， 存储合约地址在收据里面
   if msg.To() == nil {
      receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
   }
   // 设置收据日志和创建布隆过滤器
   receipt.Logs = statedb.GetLogs(tx.Hash())
   receipt.Bloom = types.CreateBloom(types.Receipts{receipt})

   return receipt, gas, err
}
```

### AsMessage 函数

```go
func (tx *Transaction) AsMessage(s Signer) (Message, error) {
   msg := Message{
      nonce:      tx.data.AccountNonce,
      gasLimit:   tx.data.GasLimit,
      gasPrice:   new(big.Int).Set(tx.data.Price),
      to:         tx.data.Recipient,
      amount:     tx.data.Amount,
      data:       tx.data.Payload,
      checkNonce: true,
   }

   var err error
   msg.from, err = Sender(s, tx)
   return msg, err
}
```

将tx 里面的数据填充到msg里面, 这个过程主要是将交易里面的form address 利用   ecrevoer函数恢复出来。

### NewEVMContext  函数

```go
// NewEVMContext creates a new context for use in the EVM.
func NewEVMContext(msg Message, header *types.Header, chain ChainContext, author *common.Address) vm.Context {
   //如果不能得到一个明确的author,那就从区块头里面解析author
   var beneficiary common.Address
   //如果函数参数里面的author为nil,则从区块头里面解析author， 这里不叫coinbase 主要是为了区别ehthash与clique引擎
    if author == nil {
      beneficiary, _ = chain.Engine().Author(header) // Ignore error, we're past header validation
   } else {
      beneficiary = *author
   }
   return vm.Context{
      CanTransfer: CanTransfer,
      Transfer:    Transfer,
      GetHash:     GetHashFn(header, chain),
      Origin:      msg.From(),
      Coinbase:    beneficiary,
      BlockNumber: new(big.Int).Set(header.Number),
      Time:        new(big.Int).Set(header.Time),
      Difficulty:  new(big.Int).Set(header.Difficulty),
      GasLimit:    header.GasLimit,
      GasPrice:    new(big.Int).Set(msg.GasPrice()),
   }
}
```

填充vm.Context的各项内容，并返回一个Context对象

### NewEVM函数

```go
func NewEVM(ctx Context, statedb StateDB, chainConfig *params.ChainConfig, vmConfig Config) *EVM {
   evm := &EVM{
      Context:      ctx,
      StateDB:      statedb,
      vmConfig:     vmConfig,
      chainConfig:  chainConfig,
      chainRules:   chainConfig.Rules(ctx.BlockNumber),
      interpreters: make([]Interpreter, 0, 1),
   }

   if chainConfig.IsEWASM(ctx.BlockNumber) {
      // to be implemented by EVM-C and Wagon PRs.
      // if vmConfig.EWASMInterpreter != "" {
      //  extIntOpts := strings.Split(vmConfig.EWASMInterpreter, ":")
      //  path := extIntOpts[0]
      //  options := []string{}
      //  if len(extIntOpts) > 1 {
      //    options = extIntOpts[1..]
      //  }
      //  evm.interpreters = append(evm.interpreters, NewEVMVCInterpreter(evm, vmConfig, options))
      // } else {
      //     evm.interpreters = append(evm.interpreters, NewEWASMInterpreter(evm, vmConfig))
      // }
      panic("No supported ewasm interpreter yet.")
   }

   //vmConfig.EVMInterpreter 将会被使用在EVM-C， 这里不会使用。
   //因为我们希望内置的EVM作为出错转移的备用选项 
   evm.interpreters = append(evm.interpreters, NewEVMInterpreter(evm, vmConfig))
   evm.interpreter = evm.interpreters[0]

   return evm
}
```

这个函数主要是根据当前的区块号以及相关配置，设置EVM的解释器.这里可以看到以太坊已经在为后面EWASM 虚拟机做准备了。

### ApplyMessage函数

```go
// ApplyMessage 通过给定的message计算新的DB状态，继而改变旧的DB状态
// ApplyMessage 返回EVM执行的返回结果和gas使用情况（包括gas refunds）和error(如果有错误出现)。
// 如果一个错误总是作为一个core error 出现，那么这个message将永远不会被这个区块所接受。
func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, uint64, bool, error) {
   return NewStateTransition(evm, msg, gp).TransitionDb()
}
```

这个函数的分为两个函数执行一个是NewStateTransition 函数，这个函数主要是设置一些交易执行的必要参数。

TransitionDb 这个函数则是主要负责执行交易，影响Db状态。

### TransitionDb 函数

```go
// TransitionDB 函数通过 apply message 将会改变state 并且返回 包含gas使用情况的结果。
// 如果执行失败，将会返回一个error, 这个error代表一个共识错误。
func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
    //进行最初的检查，

   if err = st.preCheck(); err != nil {
      return
   }
   msg := st.msg
   sender := vm.AccountRef(msg.From())
   homestead := st.evm.ChainConfig().IsHomestead(st.evm.BlockNumber)
   contractCreation := msg.To() == nil

   // Pay intrinsic gas
   gas, err := IntrinsicGas(st.data, contractCreation, homestead)
   if err != nil {
      return nil, 0, false, err
   }
   if err = st.useGas(gas); err != nil {
      return nil, 0, false, err
   }

   var (
      evm = st.evm
      // vm errors do not effect consensus and are therefor
      // not assigned to err, except for insufficient balance
      // error.
      vmerr error
   )
   if contractCreation {
      ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
   } else {
      // Increment the nonce for the next transaction
      st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
      ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
   }
   if vmerr != nil {
      log.Debug("VM returned with error", "err", vmerr)
      // The only possible consensus-error would be if there wasn't
      // sufficient balance to make the transfer happen. The first
      // balance transfer may never fail.
      if vmerr == vm.ErrInsufficientBalance {
         return nil, 0, false, vmerr
      }
   }
   st.refundGas()
   st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))

   return ret, st.gasUsed(), vmerr != nil, err
}
```

1. preCheck 函数主要进行执行交易前的检查，目前包含下面两个步骤

   1.1  检查msg 里面的nonce值与db里面存储的账户的nonce值是否一致。

   1.2  buyGas方法主要是判断交易账户是否可以支付足够的gas执行交易，如果可以支付，则设置stateTransaction 的gas值 和 initialGas 值。并且从交易执行账户扣除相应的gas值。

2. 先获取固定交易的基础费用，根据当前分叉版本和交易类型来决定基础费用，如果是创建合约则是至少是53000gas,如果是普通交易则至少是21000gas ，如果data部分不为空，则具体来说是按字节收费：字节为0的收4gas，字节不为0收68gas，所以你会看到很多做合约优化的，目的就是减少数据中不为0的字节数量，从而降低油费消耗。具体代码如下

```go
//IntrinsicGas 计算给定数据的固定gas消耗
func IntrinsicGas(data []byte, contractCreation, homestead bool) (uint64, error) {
   // Set the starting gas for the raw transaction
   var gas uint64
   if contractCreation && homestead {
      gas = params.TxGasContractCreation
   } else {
      gas = params.TxGas
   }
   // 通过事务数据量增加所需的气体
   if len(data) > 0 {
      // 零字节和非零字节的定价不同
      var nz uint64
      //获取非零字节的个数
      for _, byt := range data {
         if byt != 0 {
            nz++
         }
      }
      // 防止所需gas超过最大限制
      if (math.MaxUint64-gas)/params.TxDataNonZeroGas < nz {
         return 0, vm.ErrOutOfGas
      }
      //计算非0字节的gas消耗
      gas += nz * params.TxDataNonZeroGas

      z := uint64(len(data)) - nz
      if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
         return 0, vm.ErrOutOfGas
      }
      gas += z * params.TxDataZeroGas
   }
   return gas, nil
}
```

3. 根据contractCreation这个变量判断这是一个普通交易还是一个合约创建交易，如果是合约创建交易，则会进入下面的代码

```go
// Create creates a new contract using code as deployment code.
func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
   contractAddr = crypto.CreateAddress(caller.Address(), evm.StateDB.GetNonce(caller.Address()))
   return evm.create(caller, &codeAndHash{code: code}, gas, value, contractAddr)
}
```

crypto.CreateAddress 主要是利用账户地址和nonce进行rlp编码后取后20字节得到一个新的合约地址，因此合约地址其实是可以提前计算出来的，这也是很多合约地址是靓号的原因。

这一篇主要分析了交易执行前的一些准备工作，包括创建EVM，计算交易金额，设置交易对象，计算交易gas花销；下一篇主要是分析evm虚拟机解析器通过合约指令，执行智能合约代码的过程。




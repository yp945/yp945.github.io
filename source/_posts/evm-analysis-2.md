---
title: evm源码分析第二篇
date: 2019-07-11 10:35:08
categories:
	- ethereum
	- evm
tags:
	- ethereum
	- evm
author: ClimbYang
---

这一篇我们主要分析交易是如何被evm解释器解释执行的。

<!-- more -->

首先我们接着上一篇进入create函数

```go
// create creates a new contract using code as deployment code.
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address) ([]byte, common.Address, uint64, error) {
   // Depth check execution. Fail if we're trying to execute above the
   // limit.
   if evm.depth > int(params.CallCreateDepth) {
      return nil, common.Address{}, gas, ErrDepth
   }
   if !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
      return nil, common.Address{}, gas, ErrInsufficientBalance
   }
   nonce := evm.StateDB.GetNonce(caller.Address())
   evm.StateDB.SetNonce(caller.Address(), nonce+1)

   // Ensure there's no existing contract already at the designated address
   contractHash := evm.StateDB.GetCodeHash(address)
   if evm.StateDB.GetNonce(address) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) {
      return nil, common.Address{}, 0, ErrContractAddressCollision
   }
   // Create a new account on the state
   snapshot := evm.StateDB.Snapshot()
   evm.StateDB.CreateAccount(address)
   if evm.ChainConfig().IsEIP158(evm.BlockNumber) {
      evm.StateDB.SetNonce(address, 1)
   }
   evm.Transfer(evm.StateDB, caller.Address(), address, value)

   // Initialise a new contract and set the code that is to be used by the EVM.
   // The contract is a scoped environment for this execution context only.
   contract := NewContract(caller, AccountRef(address), value, gas)
   contract.SetCodeOptionalHash(&address, codeAndHash)

   if evm.vmConfig.NoRecursion && evm.depth > 0 {
      return nil, address, gas, nil
   }

   if evm.vmConfig.Debug && evm.depth == 0 {
      evm.vmConfig.Tracer.CaptureStart(caller.Address(), address, true, codeAndHash.code, gas, value)
   }
   start := time.Now()

   ret, err := run(evm, contract, nil, false)

   // check whether the max code size has been exceeded
   maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) && len(ret) > params.MaxCodeSize
   // if the contract creation ran successfully and no errors were returned
   // calculate the gas required to store the code. If the code could not
   // be stored due to not enough gas set an error and let it be handled
   // by the error checking condition below.
   if err == nil && !maxCodeSizeExceeded {
      createDataGas := uint64(len(ret)) * params.CreateDataGas
      if contract.UseGas(createDataGas) {
         evm.StateDB.SetCode(address, ret)
      } else {
         err = ErrCodeStoreOutOfGas
      }
   }
```

1.1 判断evm执行栈深度不能超过1024，

1.2 发送方持有的以太坊数量大于此次合约交易金额。

1.3 获取合约调用者账户nonce，然后将nonce+1存入stateDB

1.4 根据合约地址获取合约hash值

1.5 记录一个状态快照，用来失败回滚。

1.6 为这个合约地址创建一个合约账户，并为这个合约账户设置nonce值为1

1.5 产生以太坊资产转移，发送方地址账户金额减value值，合约账户的金额加value值。

1.6 根据发送方地址和合约地址，以及金额value 值和gas，合约代码和代码hash值，创建一个合约对象

1.7 设置合约对象的bytecode, bytecodehash和 合约地址

1.8 run方法来执行合约，内部调用evm的解析器来执行合约指令，如果是预编译好的合约，则预编译执行合约就行

接下来主要看run方法是如何执行合约的

```go
// run runs the given contract and takes care of running precompiles with a fallback to the byte code interpreter.
func run(evm *EVM, contract *Contract, input []byte, readOnly bool) ([]byte, error) {
   if contract.CodeAddr != nil {
      precompiles := PrecompiledContractsHomestead
      if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
         precompiles = PrecompiledContractsByzantium
      }
      if p := precompiles[*contract.CodeAddr]; p != nil {
         return RunPrecompiledContract(p, input, contract)
      }
   }
   for _, interpreter := range evm.interpreters {
      if interpreter.CanRun(contract.Code) {
         if evm.interpreter != interpreter {
            // Ensure that the interpreter pointer is set back
            // to its current value upon return.
            defer func(i Interpreter) {
               evm.interpreter = i
            }(evm.interpreter)
            evm.interpreter = interpreter
         }
         return interpreter.Run(contract, input, readOnly)
      }
   }
   return nil, ErrNoCompatibleInterpreter
}
```

run方法来执行合约，内部调用evm的解析器来执行合约指令，如果是预编译好的合约，则执行预编译合约就行。

这一部分我们看可以看到evm里面是用一个解释器数组合单独的解释器来存储解释器对象的，这可能是为了后面方便扩展多个解释器，因为从目前的代码来看，就使用了一个解释器。

选择好解释器后接下来就是具体解释字节码的过程了

```go
// 循环解释执行合约代码的inpput data, 如果执行没有错误产生，则返回执行结果，
// 有一点非常重要，那就是如果执行过程中有任何错误，那将会被认为是一个revert并且消耗所有gas的操作，除非是一个errExecutionReverted，那将以为这是一个revert并且保持所有gas退回操作。
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
   if in.intPool == nil {
      in.intPool = poolOfIntPools.get()
      defer func() {
         poolOfIntPools.put(in.intPool)
         in.intPool = nil
      }()
   }

   // Increment the call depth which is restricted to 1024
   in.evm.depth++
   defer func() { in.evm.depth-- }()

   // Make sure the readOnly is only set if we aren't in readOnly yet.
   // This makes also sure that the readOnly flag isn't removed for child calls.
   if readOnly && !in.readOnly {
      in.readOnly = true
      defer func() { in.readOnly = false }()
   }

   // Reset the previous call's return data. It's unimportant to preserve the old buffer
   // as every returning call will return new data anyway.
   in.returnData = nil

   // Don't bother with the execution if there's no code.
   if len(contract.Code) == 0 {
      return nil, nil
   }

   var (
      op    OpCode        // current opcode
      mem   = NewMemory() // bound memory
      stack = newstack()  // local stack
      // For optimisation reason we're using uint64 as the program counter.
      // It's theoretically possible to go above 2^64. The YP defines the PC
      // to be uint256. Practically much less so feasible.
      pc   = uint64(0) // program counter
      cost uint64
      // copies used by tracer
      pcCopy  uint64 // needed for the deferred Tracer
      gasCopy uint64 // for Tracer to log gas remaining before execution
      logged  bool   // deferred Tracer should ignore already logged steps
      res     []byte // result of the opcode execution function
   )
   contract.Input = input

   // Reclaim the stack as an int pool when the execution stops
   defer func() { in.intPool.put(stack.data...) }()

   if in.cfg.Debug {
      defer func() {
         if err != nil {
            if !logged {
               in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
            } else {
               in.cfg.Tracer.CaptureFault(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
            }
         }
      }()
   }
   // 解释器循环执行指定，当遇到STOP RETURN 或者合约自毁 或者 当执行过程中遇到错误的时候，
   // 直到所有指令被执行结束，循环才会停止
   for atomic.LoadInt32(&in.evm.abort) == 0 {
      if in.cfg.Debug {
         // Capture pre-execution values for tracing.
         logged, pcCopy, gasCopy = false, pc, contract.Gas
      }

      // Get the operation from the jump table and validate the stack to ensure there are
      // enough stack items available to perform the operation.
      op = contract.GetOp(pc)
      operation := in.cfg.JumpTable[op]
      if !operation.valid {
         return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
      }
      if err = operation.validateStack(stack); err != nil {
         return nil, err
      }
      // If the operation is valid, enforce and write restrictions
      if err = in.EnforceRestrictions(op, operation, stack); err != nil {
         return nil, err
      }

      var memorySize uint64
      // calculate the new memory size and expand the memory to fit
      // the operation
      if operation.memorySize != nil {
         memSize, overflow := bigUint64(operation.memorySize(stack))
         if overflow {
            return nil, errGasUintOverflow
         }
         // memory is expanded in words of 32 bytes. Gas
         // is also calculated in words.
         if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
            return nil, errGasUintOverflow
         }
      }
      // consume the gas and return an error if not enough gas is available.
      // cost is explicitly set so that the capture state defer method can get the proper cost
      cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
      if err != nil || !contract.UseGas(cost) {
         return nil, ErrOutOfGas
      }
      if memorySize > 0 {
         mem.Resize(memorySize)
      }

      if in.cfg.Debug {
         in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
         logged = true
      }

      // execute the operation
      res, err = operation.execute(&pc, in, contract, mem, stack)
      // verifyPool is a build flag. Pool verification makes sure the integrity
      // of the integer pool by comparing values to a default value.
      if verifyPool {
         verifyIntegerPool(in.intPool)
      }
      // if the operation clears the return data (e.g. it has returning data)
      // set the last return to the result of the operation.
      if operation.returns {
         in.returnData = res
      }

      switch {
      case err != nil:
         return nil, err
      case operation.reverts:
         return res, errExecutionReverted
      case operation.halts:
         return res, nil
      case !operation.jumps:
         pc++
      }
   }
   return nil, nil
}
```

这是执行的主要过程代码，接收三个参数，contract对象， input（因为是一个合约调用的交易，所以这里为nil）readonly 标志位，这个标志位只有在staticcall的时候是true的，其他的调用都是false;

接下来我们逐步分析执行过程

1. 判断解释器的intPool为nil，则先创建一个intPool,然后将新创建的intPool放入poolOfIntPools

2. evm 调用堆栈+1

3. 解释器循环执行指定，当遇到STOP RETURN 或者合约自毁 或者 当执行过程中遇到错误的时候， 直到所有指令被执行结束，循环才会停止。

   3.1 循环首先获取当前pc 计数器， 然后从jumptable里面拿出要执行的opcode, opcode是以太坊虚拟机指令，一共不超过256个，正好一个byte大小能装下。下面展示了一个 sha3的opteration

 ![](http://ww1.sinaimg.cn/large/c26c1fe3gy1g26y17xrepj20f804adfv.jpg)

​    execute表示指令对应的执行方法
​    gasCost表示执行这个指令需要消耗的gas
​    validateStack计算是不是解析器栈溢出
​    memorySize用于计算operation的占用内存大小

   3.2 根据不同的指令，指令的memorysize等，调用operation.gasCost()方法计算执行operation指令需要消耗的gas。

   3.3 调用operation.execute(&pc, in.evm, contract, mem, stack)执行指令对应的方法。

   3.4 operation.reverts值是true或者operation.halts值是true的指令，会跳出主循环，否则继续遍历下个op。

   3.5 operation指令集里面有4个特殊的指令LOG0，LOG1，LOG2，LOG3，它们的指令执行方法makeLog()会产生日志数据，日志内容包括EVM解析栈内容，指令内存数据，区块信息，合约信息等。这些日志数据会写入到tx的Receipt的logs里面，并存入本地levleldb数据库。

至此创建一个合约的交易大致分析完成。具体的每个opcode的执行以及每个opcode消耗gas都需要对应不同的执行再去详细的了解，这一点可以通过跑测试用例，然后一步一步debug下去，就可以大致了解整个流程了。

下一篇我们会分析一个普通的交易是如何执行的，也就是toAddress不为0的情况

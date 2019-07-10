---
title: libr-moveIR-exampleCode
date: 2019-07-10 16:07:54
categories:
	- libra
	- move语言
tags:
	- libra
	- move语言
author: ClimbYang
---

<!-- more -->

### Move IR 示例代码解读

本节描述如何在Move中间表示(IR)中编写事务脚本和模块。IR是即将到来的Move源代码语言的早期(且不稳定)先驱。Move IR是一个覆盖于Move字节码之上的薄薄的语法层，用于测试字节码验证器和虚拟机，它对开发人员不是特别友好。它的级别足够高，可以编写人类可读的代码，但又足够低，可以直接编译以移动字节码。

接下来，我将展示一些Move IR片段。关于如何在本地编译、运行和修改这些示例来学习。`libra/language/README.md` and `libra/language/compiler/README.md`解释了如何做到这一点。

#### 编写事务脚本

正如我们在Move事务脚本中解释的那样，用户编写事务脚本来请求对Libra区块链的全局存储的更新。几乎在任何事务脚本中都会出现两个重要的构建块:`LibraAccount.T`和`LibraCoin.T`。LibraAccount是模块的名称，T是该模块声明的资源的名称。这是一个通用的命名约定;模块声明的“main”类型通常命名为T。

当我们说一个用户“在Libra区块链上的地址0xff处有一个帐户”时，我们的意思是地址0xff持有`LibraAccount.T`的资源。每个非空地址都有一个`LibraAccount.T`资源。此资源存储帐户数据，如序列号、身份验证密钥和余额。Libra系统的任何部分想要与一个帐户进行交互，都必须从`LibraAccount.T`资源中读取数据或者调用`LibraAccount`的模块的过程函数。

帐户余额是`LibraCoin.T`类型的资源，就像其他移动资源一样。LibraCoin类型的资源T可以存储在程序变量中，在过程之间传递，等等。

下面这些例子展示了如何与模块和资源交互。

```
// 使用已经发布到链上的在一个地址上的Libra 模块
// 0x0...0 (with 64 zeroes). 0x0其实是一个缩写，其中包含64个0
import 0x0.LibraAccount;
import 0x0.LibraCoin;
main(payee: address, amount: u64) {
  //IR代码的局部变量范围在整个函数过程，所有的局部变量声明都必须
  //在程序的开始。声明和初始化变量是独立的操作，但是字节码验证器会阻止
  //任何使用未初始化变量的尝试。
  let coin: R#LibraCoin.T;
  //上面类型的R#部分是两个*类型注解* R#和V#之一(“Resource”和“unrestricted Value”的缩写)。这些注解
  //必须匹配类型声明的类型(例如LibraCoin模块声明“resource T”或“struct T”)

  //获取sender的数量为amount的 LibraCoin.T资源，如果sender的余额不足balance，则会失败，这里amount   //	是一个 unrestricted值，所以可以copy也可以move，但是因为后续不再使用， 所以最好还是move掉(move   //  白皮书是里面写的是copy) 
  coin = LibraAccount.withdraw_from_sender(move(amount));
  // Move LibraCoin.T 资源进入payee账户，如果账户不存在，则步骤失败
  LibraAccount.deposit(move(payee), move(coin));

  //每一个函数过程必须要有个return, IR 编译器不会在没有写return的情况下，自己加上return，如果有返回     // 值，还要写上相应的返回值 
  return;
}
```

这个事务脚本有一个不幸的问题——如果收款人地址下没有帐户，它将失败。我们将通过修改脚本来解决这个问题，以便在收款人帐户不存在的情况下为其创建帐户。

```
// 一个点对点支付的例子变体，即当账户不存在的时候，创建账户

import 0x0.LibraAccount;
import 0x0.LibraCoin;
main(payee: address, amount: u64) {
  let coin: R#LibraCoin.T;
  let account_exists: bool;

  // Acquire a LibraCoin.T resource with value `amount` from the sender's
  // account.  This will fail if the sender's balance is less than `amount`.
  coin = LibraAccount.withdraw_from_sender(move(amount));

  //这里调用LibraAccount的内置函数判断账户是否存在
  account_exists = LibraAccount.exists(copy(payee));

  if (!move(account_exists)) {
    //通过发布一个LibraAccount.T 资源在接收方地址下的方式为接受者创建一个账户，如果
    //账户资源已经存在，则失败
    create_account(copy(payee));
  }

  LibraAccount.deposit(move(payee), move(coin));
  return;
}
```

让我们看一个更复杂的例子。在本例中，我们将使用事务脚本向多个收件人付款，而不是只向一个收件人付款。

```
//多个收款人例子，这个例子将amount1+ amount2数量的资金转移到 收款1账户和收款2账户上

import 0x0.LibraAccount;
import 0x0.LibraCoin;
main(payee1: address, amount1: u64, payee2: address, amount2: u64) {
  let coin1: R#LibraCoin.T;
  let coin2: R#LibraCoin.T;
  let total: u64;

  total = move(amount1) + copy(amount2);
  coin1 = LibraAccount.withdraw_from_sender(move(total));
  // This mutates `coin1`, which now has value `amount1`.
  // `coin2` has value `amount2`.
  //下面代码执行完后，coin1资源将拥有数量为amount1的Coin
  //coin2是一个有amount2数量coin的资源，withdraw是一个内置函数
  coin2 = LibraCoin.withdraw(&mut coin1, move(amount2));

  // 执行支付
  LibraAccount.deposit(move(payee1), move(coin1));
  LibraAccount.deposit(move(payee2), move(coin2));
  return;
}
```

#### 编写模块

现在，我们将把注意力转向编写自己的Move模块，而不是重用现有的LibraAccount和LibraCoin模块。考虑这种情况:Bob将在将来的某个时候在地址a创建一个帐户。Alice想为Bob“指定”一些资金，这样一旦Bob的账户创建好了，他就可以把这些资金转到他的账户中。但她也希望，如果鲍勃从未创建过账户，她也能收回自己的资金。

为了给Alice解决这个问题，我们将编写一个模块EarmarkedLibraCoin:

- 声明一个新的资源类型EarmarkedLibraCoin。这里面有Libra coin和接收方地址
- 允许Alice创建这样的类型并将其发布到她的帐户下(`create`过程)。
- 允许Bob claim资源(`claim_for_receiver`过程)
- 允许任何拥有EarmarkedLibraCoin.T资源的人销毁它，并获得底层的LibraCoin(`unwarp`的过程)

```
//一个用于收款地址指定coin的模块
module EarmarkedLibraCoin {
  import 0x0.LibraCoin;

  //T 定义了一个资源， 包含了Libra coin 和一个收款地址
  resource T {
    coin: R#LibraCoin.T,
    recipient: address
  }

  // 为指定收款地址创建EarmarkedCoin 资源
  // 在此交易发送方账户发布该资源
  public create(coin: R#LibraCoin.T, recipient: address) {
    let t: R#Self.T;

    //构造或者叫做打包一个类型为T的资源，只有EarmarkedCoin 模块的函数可以创建 EarmarkedCoin.T
    t = T {
      coin: move(coin),
      recipient: move(recipient),
    };

    // 在此交易发送者账户下发布 earmarked coin 
    //每个账户可以包含一个指定类型的资源，如果有相同类型的资源存在，此过程会失败
    //move_to_sender是一个内置函数，作用是将资源发布到sender账户，支持泛型
    move_to_sender<T>(move(t));
    return;
  }

  //允许交易发送者认领一枚指定给他的硬币。
  public claim_for_recipient(earmarked_coin_address: address): R#Self.T {
    let t: R#Self.T;
    //这里创建T资源的引用是为了防止资源只能被move一次，assert之后而不能进行return
    let t_ref: &R#Self.T;
    let sender: address;

    //移除指定地址T类型的资源，如果指定地址没有类型T的资源，则失败
    t = move_from<T>(move(earmarked_coin_address));
    //拿到资源引用，方便后续assert
    t_ref = &t;
    //获取此次交易发送者账户
    sender = get_txn_sender();
    //确保收款账户是交易发送者，如果断言失败，则此次交易不会造成任何影响， 原子操作。
    //99是一个内部断言使用的error code,便于交易失败打印事件
    assert(*(&move(t_ref).recipient) == move(sender), 99);
	//返回获取到的资源，接下来应该使用unwarp获取资源里面的Libra coin了
    return move(t);
  }

  // 如果Bob的地址没有被创建，则Alice将会回收掉该资源
  public claim_for_creator(): R#Self.T {
    let t: R#Self.T;
    let coin: R#LibraCoin.T;
    let recipient: address;
    let sender: address;

    sender = get_txn_sender();
    // This will fail if no resource of type T under the sender's address.
    t = move_from<T>(move(sender));
    return move(t);
  }

  // 从包装资源里面获取到Libra coin,并将它返回给调用者
  public unwrap(t: R#Self.T): R#LibraCoin.T {
    let coin: R#LibraCoin.T;
    let recipient: address;

    // 将t 资源进行解包，用T类型去接收，只有声明了T类型的模块才可以接收这个资源
    T { coin, recipient } = move(t);
    //返回libra coin资源
    return move(coin);
  }

}
```

上述过程可以综述为：Alice可以创建一个事务脚本，调用`create`方法，参数为Bob的地址和libra coin资源，从而为Bob创建一个指定的coin在她的地址上。一旦创建了这个资源, Bob就可以通过从发送事务来声明coin。这将调用`claim_for_receiver`，然后将结果传递给`unwrap`，并将返回的LibraCoin存储在他希望的任何地方。如果Bob花了太长时间在创建一个帐户上，而Alice想要收回她的资金，她可以使用`claim_for_creator`和`unwrap`。将资金回收。

#### rust引用说明

这里针对上述例子有个关于rust引用或者借用的说明，这也是针对白皮书里面的一些引用形式给一个解释

```
let b = a;　　 a绑定的资源A转移给b，b拥有这个资源A

let b = &a;　　a绑定的资源A借给b使用，b只有资源A的读权限

let b =  &mut a;　　a绑定的资源A借给b使用，b有资源A的读写权限

let mut b = &mut a;　　a绑定的资源A借给b使用，b有资源A的读写权限。同时，b可绑定到新的资源上面去(更新绑定的能力)
let &mut b = &mut a; a 绑定的资源A借给b的引用使用，b的引用有资源A的读写权限。同时b的引用可以绑定到新的资源上面去(更新绑定能力)
如果一个值有了可变引用，则不能再有不变引用，这部分内容可以仔细参考rust的引用与借用关系
```

#### 例子讲解

下面我会抽取libra github仓库里面的一些典型的例子来对move进行更加全面的解释。

下面这个示例是一个module

```
module Signature {
    native public ed25519_verify(signature: bytearray, public_key: bytearray, message: bytearray): bool;
}
这个模块是官方的算法验签名模块，正如白皮书里面说得到，move语言可以直接调用一些rust的函数实现，这个过程称之为navite,用navite标识。
```

下面这个示例是一个script，由一个main过程构成

```
import 0x0.LibraAccount;
main (new_key: bytearray) {
  LibraAccount.rotate_authentication_key(move(new_key));
  return;
}
它引入了0x0地址(缩写)上的LibraAccount模块，并调用rotate_authentication_key过程，替换账户身份验证密钥，这个与账户创建密钥不一样，这个密钥是用来创建交易签名的
```

下面这个示例是一个module + script，它以`modules:`开始，表明module的开始段落，以`script`开始事务脚本段落，事务脚本可以使用上面定义的module。

```
modules:

module B {
    struct T {g: u64}

    public new(g: u64): V#Self.T {
        return T{g: move(g)};
    }

    public t(this: &V#Self.T) {
        let g: &u64;
        let y: u64;
        g = &copy(this).g;
        y = *move(g);
        //这里因为this资源没有再次被使用，所以使用release减少一次引用计数
        release(move(this));
        return;
    }
}

script:

import Transaction.B;

main() {
    let x: V#B.T;
    let y: &V#B.T;
    x = B.new(5);
    y = &x;
    B.t(move(y));
    return;
}
```

接下来主要分析一下 `LibraCoin` 和`LibraAccount`这两个内置模块

#### LibraCoin 模块分析

```
module LibraCoin {
    // A resource representing the Libra coin
    resource T {
        // The value of the coin. May be zero
        value: u64,
    }

    // A resource that grants access to `LibraCoin.mint`. Only the Association account has one.
    resource MintCapability {}

    // Return a reference to the MintCapability published under the sender's account. Fails if thesender does not have a MintCapability.
    // Since only the Association account has a mint capability, this will only succeed if it is invoked by a transaction sent by that account.
    public borrow_sender_mint_capability(): &R#Self.MintCapability {
        let sender: address;
        //可变引用
        let capability_ref: &mut R#Self.MintCapability;
        //不可变引用
        let capability_immut_ref: &R#Self.MintCapability;

        sender = get_txn_sender();
        capability_ref = borrow_global<MintCapability>(move(sender));
        //borrow_global默认返回可变引用，这里使用freeze需要去除可变性 
        capability_immut_ref = freeze(move(capability_ref));
        return move(capability_immut_ref);
    }

	// mint 一个新的LibraCoin.T资源价值 value, 调用者必须有一个MintCapability的引用
	//只有联盟成员账户可以获取这样的引用，而且只能通过 borrow_sender_mint_capbaility方法获取
    public mint(value: u64, capability: &R#Self.MintCapability): R#Self.T {
        release(move(capability));
        return T{value: move(value)};
    }
	
	//这个过程是私有的，因为只能被虚拟机内部调用它只被使用
   //在创世纪创建writeset时，为联盟帐户提供单个mintability
    grant_mint_capability() {
        move_to_sender<MintCapability>(MintCapability{});
        return;
    }

    // Create a new LibraCoin.T with a value of 0
    public zero(): R#Self.T {
        return T{value: 0};
    }

	//公开coin数值的访问方法
    public value(coin_ref: &R#Self.T): u64 {
        return *&move(coin_ref).value;
    }

    // 将 coin分为两半，一半是给定数值，一般是剩余数值
    public split(coin: R#Self.T, amount: u64): R#Self.T * R#Self.T {
        let other: R#Self.T;
        other = Self.withdraw(&mut coin, move(amount));
        return move(coin), move(other);
    }

    //从给定coin资源中划分出amount数量的资源，并将新创建的资源返回
    public withdraw(coin_ref: &mut R#Self.T, amount: u64): R#Self.T {
        let value: u64;

        // Check that `amount` is less than the coin's value
        value = *(&mut copy(coin_ref).value);
        assert(copy(value) >= copy(amount), 10);

        // 直接在原来的值上面分割
        *(&mut move(coin_ref).value) = move(value) - copy(amount);
        return T{value: move(amount)};
    }

    // 合并两个同样的coin资源，并将新的资源返回
    public join(coin1: R#Self.T, coin2: R#Self.T): R#Self.T  {
        Self.deposit(&mut coin1, move(coin2));
        return move(coin1);
    }

    // "Merges" the two coins check 是需要合并在coin_ref上的资源
    public deposit(coin_ref: &mut R#Self.T, check: R#Self.T) {
        let value: u64;
        let check_value: u64;

        value = *(&mut copy(coin_ref).value);
        T { value: check_value } = move(check);
        *(&mut move(coin_ref).value)= move(value) + move(check_value);
        return;
    }

    //销毁一个值为0的资源，不能销毁一个非0值的资源
    public destroy_zero(coin: R#Self.T) {
        let value: u64;
        T { value } = move(coin);
        assert(move(value) == 0, 11);
        return;
    }

	//临时的程序，将来将会被取消，此程序主要是用来从账户消耗一定的手续费
    public TODO_REMOVE_burn_gas_fee(coin: R#Self.T) {
        let value: u64;
        T { value } = move(coin);
        return;
    }
}
```

#### LibraAccount模块分析

```
//这个模块主要是很对账户资源的，用于管理每个Libra账户
//可以看到他引用了
module LibraAccount {
    import 0x0.LibraCoin;
    import 0x00.Hash;

    //每个Libra账户都有一个LibraAccount.T资源
    // Every Libra account has a LibraLibraAccount.T resource
    resource T {
        // The coins stored in this account
        balance: R#LibraCoin.T,
        //当前的身份认证key
        //这个key可以与创建账户的key不同
        authentication_key: bytearray,
		//账户nonce
        sequence_number: u64,
        //临时的发送交易事件计数器，等到后面事件系统被完善后，这个值应该会被代替
        sent_events_count: u64,
  		//临时的交易接收事件计数器，等到后面事件系统被完善后，这个值应该会被代替
        received_events_count: u64
    }

    // Message for sent events
    struct SentPaymentEvent {
        // The address that was paid
        payee: address,
        // The amount of LibraCoin.T sent
        amount: u64,
    }

    // Message for received events
    struct ReceivedPaymentEvent {
        // The address that sent the coin
        payer: address,
        // The amount of LibraCoin.T received
        amount: u64,
    }

 	//创建一个新的LibraAccount.T类型的资源
 	//这个过程被内置模块create_account调用
    make(auth_key: bytearray): R#Self.T {
        let zero_balance: R#LibraCoin.T;
        zero_balance = LibraCoin.zero();
        return T {
            balance: move(zero_balance),
            authentication_key: move(auth_key),
            sequence_number: 0,
            sent_events_count: 0,
            received_events_count: 0,
        };
    }

    // 向payee账户存入to_deposit资源
    public deposit(payee: address, to_deposit: R#LibraCoin.T) {
        let deposit_value: u64;
        let payee_account_ref: &mut R#Self.T;
        let sender: address;
        let sender_account_ref: &mut R#Self.T;
        let sent_event: V#Self.SentPaymentEvent;
        let received_event: V#Self.ReceivedPaymentEvent;

        // Check that the `to_deposit` coin is non-zero
        deposit_value = LibraCoin.value(&to_deposit);
        assert(copy(deposit_value) > 0, 7);

        // Load the sender's account
        sender = get_txn_sender();
        sender_account_ref = borrow_global<T>(copy(sender));
        // Log a send event
        sent_event = SentPaymentEvent { payee: copy(payee), amount: copy(deposit_value) };
        // 目前的打印事件只是临时之举，未来应该会变得更加有条理
        emit_event(&mut move(sender_account_ref).sent_events_count, b"73656E745F6576656E74735F636F756E74", move(sent_event));

        // Load the payee's account
        payee_account_ref = borrow_global<T>(move(payee));
        // Deposit the `to_deposit` coin
        LibraCoin.deposit(&mut copy(payee_account_ref).balance, move(to_deposit));
        // Log a received event
        received_event = ReceivedPaymentEvent { payer: move(sender), amount: move(deposit_value) };
        // TEMPORARY The events system is being overhauled and this will be replaced by something
        // more principled in the future
        emit_event(&mut move(payee_account_ref).received_events_count, b"72656365697665645F6576656E74735F636F756E74", move(received_event));
        return;
    }

    //mint_to-address 只能被又有mint能力的的账户使用(参照LibraCoin模块)
    //这些帐户将被收取gas费用。如果这些帐户没有足够的gas支付,则mint失败
    // 那些账户也可以mint给自己coin
    public mint_to_address(payee: address, amount: u64) {
        let mint_capability_ref: &R#LibraCoin.MintCapability;
        let coin: R#LibraCoin.T;
        let payee_account_ref: &mut R#Self.T;
        let payee_exists: bool;
        let sender: address;

        // Mint the coin
        mint_capability_ref = LibraCoin.borrow_sender_mint_capability();
        coin = LibraCoin.mint(copy(amount), move(mint_capability_ref));

        // Create an account if it does not exist
        payee_exists = exists<T>(copy(payee));
        if (!move(payee_exists)) {
        	//这一行感觉没必要
            sender = get_txn_sender();
            Self.create_new_account(copy(payee), 0);
        }

        // Deposit the minted `coin`
        Self.deposit(move(payee), move(coin));
        return;
    }

    // Helper to withdraw `amount` from the given `account` and return the resulting LibraCoin.T
    withdraw_from_account(account: &mut R#Self.T, amount: u64): R#LibraCoin.T {
        let to_withdraw: R#LibraCoin.T;
        to_withdraw = LibraCoin.withdraw(&mut move(account).balance, copy(amount));
        return move(to_withdraw);
    }

    // Withdraw `amount` LibraCoin.T from the transaction sender's account
    public withdraw_from_sender(amount: u64): R#LibraCoin.T {
        let sender: address;
        let sender_account: &mut R#Self.T;
        let to_withdraw: R#LibraCoin.T;

        // Load the sender
        sender = get_txn_sender();
        sender_account = borrow_global<T>(move(sender));

        // Withdraw the coin
        to_withdraw = Self.withdraw_from_account(move(sender_account), move(amount));
        return move(to_withdraw);
    }

    //从发送者账户转移资金到接受者，如果发送者账户不存在，先创建账户，然后再调用此过程
    public pay_from_sender(payee: address, amount: u64) {
        let to_pay: R#LibraCoin.T;
        let payee_exists: bool;
        payee_exists = exists<T>(copy(payee));
        if (move(payee_exists)) {
            to_pay = Self.withdraw_from_sender(move(amount));
            Self.deposit(move(payee), move(to_pay));
        } else {
            Self.create_new_account(move(payee), move(amount));
        }
        return;
    }

	//更新交易发送者身份认证key
	//新的key将会被用于交易签名
    public rotate_authentication_key(new_authentication_key: bytearray) {
        let sender: address;
        let sender_account: &mut R#Self.T;
        sender = get_txn_sender();
        sender_account = borrow_global<T>(move(sender));
        *(&mut move(sender_account).authentication_key) = move(new_authentication_key);
        return;
    }

    //创建一个新的账户，如果初始资金>0，则从交易发送者账户转移资金到新账户
    public create_new_account(fresh_address: address, initial_balance: u64) {
        create_account(copy(fresh_address));
        if (copy(initial_balance) > 0) {
            Self.pay_from_sender(move(fresh_address), move(initial_balance));
        }
        return;
    }

    // Helper to return u64 value of the `balance` field for given `account`
    balance_for_account(account: &R#Self.T): u64 {
        let balance_value: u64;
        balance_value = LibraCoin.value(&move(account).balance);
        return move(balance_value);
    }

    // Return the current balance of the LibraCoin.T in LibraLibraAccount.T at `addr`
    public balance(addr: address): u64 {
        let payee_account: &mut R#Self.T;
        let imm_payee_account: &R#Self.T;
        let balance_amount: u64;
        //从账户地址，拿到LibraAccount类型的资源，此时拿到的资源是可变的
        payee_account = borrow_global<T>(move(addr));
        //将资源转为不可变资源，防止误操作
        imm_payee_account = freeze(move(payee_account));
        balance_amount = Self.balance_for_account(move(imm_payee_account));
        return move(balance_amount);
    }

    // Helper to return the sequence number field for given `account`
    sequence_number_for_account(account: &R#Self.T): u64 {
        return *(&move(account).sequence_number);
    }

    // Return the current sequence number at `addr`
    public sequence_number(addr: address): u64 {
        let account_ref: &mut R#Self.T;
        let imm_ref: &R#Self.T;
        let sequence_number_value: u64;
        account_ref = borrow_global<T>(move(addr));
        imm_ref = freeze(move(account_ref));
        sequence_number_value = Self.sequence_number_for_account(move(imm_ref));
        return move(sequence_number_value);
    }

    // Checks if an account exists at `check_addr`
    public exists(check_addr: address): bool {
        let is_present: bool;
        is_present = exists<T>(move(check_addr));
        return move(is_present);
    }
	
	// 序言过程在每笔交易开始的时候被调用，它要做以下验证：
	//账户的身份key是否匹配交易的公钥
	//账户是否有足够的金额支付所有的gas
	//sequence number是否匹配当前账户sequence key
    prologue() {
        let transaction_sender: address;
        let transaction_sender_exists: bool;
        let sender_account: &mut R#Self.T;
        let imm_sender_account: &R#Self.T;
        let sender_public_key: bytearray;
        let public_key_hash: bytearray;
        let gas_price: u64;
        let gas_units: u64;
        let gas_fee: u64;
        let balance_amount: u64;
        let sequence_number_value: u64;
        let transaction_sequence_number_value: u64;

        transaction_sender = get_txn_sender();

        // 现在这些error code还是很不友好的，后续应该会变得更好
        transaction_sender_exists = exists<T>(copy(transaction_sender));
        assert(move(transaction_sender_exists), 5);

        // Load the transaction sender's account
        sender_account = borrow_global<T>(copy(transaction_sender));

		//检查交易的公钥是否与当前账户的身份key相匹配
        sender_public_key = get_txn_public_key();
        public_key_hash = Hash.sha3_256(move(sender_public_key));
        assert(move(public_key_hash) == *(&copy(sender_account).authentication_key), 2);

        // 检查是否有足够的余额支付交易手续费
        gas_price = get_txn_gas_unit_price();
        gas_units = get_txn_max_gas_units();
        //先计算最大所需的手续费,在交易结束后进行扣除真实消耗的gas手续费
        gas_fee = move(gas_price) * move(gas_units);
        imm_sender_account = freeze(copy(sender_account));
        balance_amount = Self.balance_for_account(move(imm_sender_account));
        assert(move(balance_amount) >= move(gas_fee), 6);

        // 检查交易的sequence number是否与账户保存的sequence number匹配
        sequence_number_value = *(&mut move(sender_account).sequence_number);
        transaction_sequence_number_value = get_txn_sequence_number();
        //这里多判断一次可能是未来防止并行交易的时候较大sequence number可以被接受
        assert(copy(transaction_sequence_number_value) >= copy(sequence_number_value), 3);
        assert(move(transaction_sequence_number_value) == move(sequence_number_value), 4);
        return;
    }

	// 收尾主要是被用于在交易结束后进行一些处理
	//主要是计算交易真实消耗gas的手续费和调整账户sequence number
    // The epilogue is invoked at the end of transactions.
    // It collects gas and bumps the sequence number
    epilogue() {
        let transaction_sender: address;
        let sender_account: &mut R#Self.T;
        let imm_sender_account: &R#Self.T;
        let gas_price: u64;
        let gas_units_remaining: u64;
        let starting_gas_units: u64;
        let gas_fee_amount: u64;
        let balance_amount: u64;
        let gas_fee: R#LibraCoin.T;
        let transaction_sequence_number_value: u64;

        transaction_sender = get_txn_sender();

        // Load the transaction sender's account
        sender_account = borrow_global<T>(copy(transaction_sender));

        //收取真实消耗的gas的手续费
        gas_price = get_txn_gas_unit_price();
        starting_gas_units = get_txn_max_gas_units();
        gas_units_remaining = get_gas_remaining();
        gas_fee_amount = move(gas_price) * (move(starting_gas_units) - move(gas_units_remaining));
        imm_sender_account = freeze(copy(sender_account));
        balance_amount = Self.balance_for_account(move(imm_sender_account));
        assert(move(balance_amount) >= copy(gas_fee_amount), 6);

        gas_fee = Self.withdraw_from_account(copy(sender_account), move(gas_fee_amount));
       //销毁掉相应的手续费资源
       LibraCoin.TODO_REMOVE_burn_gas_fee(move(gas_fee));

        // 账户sequence number + 1
        transaction_sequence_number_value = get_txn_sequence_number();
        *(&mut move(sender_account).sequence_number) = move(transaction_sequence_number_value) + 1;
        return;
    }

}
```
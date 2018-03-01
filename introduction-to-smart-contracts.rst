###############################
智能合约概述
###############################

.. _simple-smart-contract:

***********************
一个简单的智能合约
***********************

让我们从最基本的例子开始。如果你现在不了解所有的事情，这很好，我们将在稍后详细介绍。

存储
=======

::

    pragma solidity ^0.4.0;

    contract SimpleStorage {
        uint storedData;

        function set(uint x) public {
            storedData = x;
        }

        function get() public constant returns (uint) {
            return storedData;
        }
    }

第一行简单地告诉我们，源代码是用Solidity版本0.4.0编写的， 或者是任何不会破坏功能（最高但不包括0.5.0版本）的新版本。 这是为了确保合约不会突然与新的编译器版本冲突。 关键字``pragma`` 是告诉编译器开始处理源代码的指令 (例如 `pragma once <https://en.wikipedia.org/wiki/Pragma_once>`_).

在Solidity中合约实际上是以太坊区块链上的特定地址的代码（*functions*）和数据（*state*）的集合。``uint storedData;``这行代码声明了一个``storedData`` state变量（256位无符整型）.您可以将其视为数据库中的单个槽（solt）,通过调用管理数据库的代码的函数可以查询和修改它。以太坊合约的拥有者可以通过 ``set`` 和 ``get``函数修改或查询该变量的值。

要访问state变量，不需要像其他语言中常见的前缀 ``this.``。

基于以太坊构建的基础架构，这个合约只干了一件事：允许任何人存储一个number，并且任何人都无法阻止你发表这个number。当然，现在任何人都可以 ``set``不同的值覆盖您的number，但number仍将存储在区块链的历史记录中。稍后，我们将看到如何强制实施访问限制，以便只有您可以更改这个number。

.. note::
    所有标识符（合约名，函数名和变量名）都限制为ASCII字符集。可以将UTF-8编码的数据存储在字符串变量中。

.. warning::
    使用Unicode文本时要小心，因为看起来类似（甚至相同）的字符可以具有不同的代码点(code points)，并且因此将被编码为不同的字节数组。

.. index:: ! subcurrency

代币发行例子
===================

以下合约是最简单的加密货币。创建合约者可以凭空产生代币（以此发行是没有价值的）。此外，只要有一个以太坊密钥对（Ethereum keypair），任何人都可以互相发送代币，而无需使用用户名和密码进行注册。

::

    pragma solidity ^0.4.20; //实际上应该是0.4.21 

    contract Coin {
        // 关键字“public”使这些变量可以从外部读取
        address public minter;
        mapping (address => uint) public balances;

        // event允许轻客户端有效地对变化做出反应
        event Sent(address from, address to, uint amount);

        // 这是构造函数的代码只有在创建合约时才能运行
        function Coin() public {
            minter = msg.sender;
        }

        function mint(address receiver, uint amount) public {
            if (msg.sender != minter) return;
            balances[receiver] += amount;
        }

        function send(address receiver, uint amount) public {
            if (balances[msg.sender] < amount) return;
            balances[msg.sender] -= amount;
            balances[receiver] += amount;
            emit Sent(msg.sender, receiver, amount);
        }
    }

这份合约引入了一些新的概念，让我们一一浏览。

``address public minter;``这行声明一个可公开访问的地址类型的state变量。``address``类型是一个不允许任何算术运算的160位值。它适用于存储属于外部人员的合约地址或密钥对（keypairs）地址。关键字``public``自动生成一个函数，允许您从合约外部访问state变量的当前值。没有这个关键字，其他合约就无法访问该变量。编译器生成的函数的代码大致等同于以下内容::

    function minter() returns (address) { return minter; }

当然，像这样添加一个函数是行不通的，因为我们有一个名字相同的函数和state变量，但是希望您能够明白 - 编译器会为您解决这个问题

.. index:: mapping

下一行``mapping (address => uint) public balances;`` 也创建了一个公共state变量，但它是一个更复杂的数据类型。该类型将地址映射为无符整型。映射可以看作 `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_，它被虚拟初始化，这样每个可能的键都存在并映射到一个字节表示全为零的值。这种类比并不过分，因为它既不可能获得映射的所有键的list，也不可能获得所有值的list。因此，要么记住添加到映射中的内容（做的更好点，保留list或使用更高级的数据类型），要么在非必需环境中使用，就像由``public``关键字创建的:ref:`getter function<getter-functions>` 这样，有点复杂。它大致如下所示：
::

    function balances(address _account) public view returns (uint) {
        return balances[_account];
    }

如您所见，您可以使用此功能轻松查询单个帐户的余额。

.. index:: event

``event Sent(address from, address to, uint amount);``这行声明了在``send``函数最后一行的“事件”（event） 。UI（当然也包括服务器应用程序）可以监听区块链上正在交易（being fired）的事件，而无需花费太多成本。一旦发生交易了，监听者也将收到 ``from``, ``to``和 ``amount``参数，并且，这使得跟踪交易变得容易。为了监听这个事件，你可以使用
::

    Coin.Sent().watch({}, '', function(error, result) {
        if (!error) {
            console.log("Coin transfer: " + result.args.amount +
                " coins were sent from " + result.args.from +
                " to " + result.args.to + ".");
            console.log("Balances now:\n" +
                "Sender: " + Coin.balances.call(result.args.from) +
                "Receiver: " + Coin.balances.call(result.args.to));
        }
    })

注意在UI如何调用自动生成的``balances``函数。

.. index:: coin

特殊函数``Coin``是在创建合约期间运行的构造函数，不能在事后调用。它永久存储合约创建者的地址：``msg``（ 与``tx``和``block``一起）是一个神奇的全局变量，其中包含一些允许访问区块链的属性。``msg.sender``始终是当前（外部）函数调用的来源地址。

最后，实际可以被用户和合约调用的函数是``mint``和``send``。如果``mint``被合约创建者以外的任何人调用，则什么都不会发生。另一方面，任何人都可以使用（已经有一些这个代币的人）``send``将代币发送给其他人。请注意，如果您使用此合约将代币发送到某个地址，则当您在区块链浏览器中查看该地址时，您将看不到任何内容，因为您发送代币和已更改的余额仅存储在此数据存储中特定的代币合约。通过使用事件，创建追踪新代币交易和余额的“区块链浏览器”（"blockchain explorer"）相对容易。

.. _blockchain-basics:

*****************
区块链基础
*****************

Blockchains as a concept are not too hard to understand for programmers. The reason is that
most of the complications (mining, `hashing <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_, `elliptic-curve cryptography <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_, `peer-to-peer networks <https://en.wikipedia.org/wiki/Peer-to-peer>`_, etc.)
are just there to provide a certain set of features and promises. Once you accept these
features as given, you do not have to worry about the underlying technology - or do you have
to know how Amazon's AWS works internally in order to use it?

.. index:: transaction

Transactions
============

A blockchain is a globally shared, transactional database.
This means that everyone can read entries in the database just by participating in the network.
If you want to change something in the database, you have to create a so-called transaction
which has to be accepted by all others.
The word transaction implies that the change you want to make (assume you want to change
two values at the same time) is either not done at all or completely applied. Furthermore,
while your transaction is applied to the database, no other transaction can alter it.

As an example, imagine a table that lists the balances of all accounts in an
electronic currency. If a transfer from one account to another is requested,
the transactional nature of the database ensures that if the amount is
subtracted from one account, it is always added to the other account. If due
to whatever reason, adding the amount to the target account is not possible,
the source account is also not modified.

Furthermore, a transaction is always cryptographically signed by the sender (creator).
This makes it straightforward to guard access to specific modifications of the
database. In the example of the electronic currency, a simple check ensures that
only the person holding the keys to the account can transfer money from it.

.. index:: ! block

Blocks
======

One major obstacle to overcome is what, in Bitcoin terms, is called a "double-spend attack":
What happens if two transactions exist in the network that both want to empty an account,
a so-called conflict?

The abstract answer to this is that you do not have to care. An order of the transactions
will be selected for you, the transactions will be bundled into what is called a "block"
and then they will be executed and distributed among all participating nodes.
If two transactions contradict each other, the one that ends up being second will
be rejected and not become part of the block.

These blocks form a linear sequence in time and that is where the word "blockchain"
derives from. Blocks are added to the chain in rather regular intervals - for
Ethereum this is roughly every 17 seconds.

As part of the "order selection mechanism" (which is called "mining") it may happen that
blocks are reverted from time to time, but only at the "tip" of the chain. The more
blocks that are added on top, the less likely it is. So it might be that your transactions
are reverted and even removed from the blockchain, but the longer you wait, the less
likely it will be.


.. _the-ethereum-virtual-machine:

.. index:: !evm, ! ethereum virtual machine

****************************
以太坊虚拟机
****************************

Overview
========

The Ethereum Virtual Machine or EVM is the runtime environment
for smart contracts in Ethereum. It is not only sandboxed but
actually completely isolated, which means that code running
inside the EVM has no access to network, filesystem or other processes.
Smart contracts even have limited access to other smart contracts.

.. index:: ! account, address, storage, balance

Accounts
========

There are two kinds of accounts in Ethereum which share the same
address space: **External accounts** that are controlled by
public-private key pairs (i.e. humans) and **contract accounts** which are
controlled by the code stored together with the account.

The address of an external account is determined from
the public key while the address of a contract is
determined at the time the contract is created
(it is derived from the creator address and the number
of transactions sent from that address, the so-called "nonce").

Regardless of whether or not the account stores code, the two types are
treated equally by the EVM.

Every account has a persistent key-value store mapping 256-bit words to 256-bit
words called **storage**.

Furthermore, every account has a **balance** in
Ether (in "Wei" to be exact) which can be modified by sending transactions that
include Ether.

.. index:: ! transaction

Transactions
============

A transaction is a message that is sent from one account to another
account (which might be the same or the special zero-account, see below).
It can include binary data (its payload) and Ether.

If the target account contains code, that code is executed and
the payload is provided as input data.

If the target account is the zero-account (the account with the
address ``0``), the transaction creates a **new contract**.
As already mentioned, the address of that contract is not
the zero address but an address derived from the sender and
its number of transactions sent (the "nonce"). The payload
of such a contract creation transaction is taken to be
EVM bytecode and executed. The output of this execution is
permanently stored as the code of the contract.
This means that in order to create a contract, you do not
send the actual code of the contract, but in fact code that
returns that code.

.. index:: ! gas, ! gas price

Gas
===

Upon creation, each transaction is charged with a certain amount of **gas**,
whose purpose is to limit the amount of work that is needed to execute
the transaction and to pay for this execution. While the EVM executes the
transaction, the gas is gradually depleted according to specific rules.

The **gas price** is a value set by the creator of the transaction, who
has to pay ``gas_price * gas`` up front from the sending account.
If some gas is left after the execution, it is refunded in the same way.

If the gas is used up at any point (i.e. it is negative),
an out-of-gas exception is triggered, which reverts all modifications
made to the state in the current call frame.

.. index:: ! storage, ! memory, ! stack

Storage, Memory and the Stack
=============================

Each account has a persistent memory area which is called **storage**.
Storage is a key-value store that maps 256-bit words to 256-bit words.
It is not possible to enumerate storage from within a contract
and it is comparatively costly to read and even more so, to modify
storage. A contract can neither read nor write to any storage apart
from its own.

The second memory area is called **memory**, of which a contract obtains
a freshly cleared instance for each message call. Memory is linear and can be
addressed at byte level, but reads are limited to a width of 256 bits, while writes
can be either 8 bits or 256 bits wide. Memory is expanded by a word (256-bit), when
accessing (either reading or writing) a previously untouched memory word (ie. any offset
within a word). At the time of expansion, the cost in gas must be paid. Memory is more
costly the larger it grows (it scales quadratically).

The EVM is not a register machine but a stack machine, so all
computations are performed on an area called the **stack**. It has a maximum size of
1024 elements and contains words of 256 bits. Access to the stack is
limited to the top end in the following way:
It is possible to copy one of
the topmost 16 elements to the top of the stack or swap the
topmost element with one of the 16 elements below it.
All other operations take the topmost two (or one, or more, depending on
the operation) elements from the stack and push the result onto the stack.
Of course it is possible to move stack elements to storage or memory,
but it is not possible to just access arbitrary elements deeper in the stack
without first removing the top of the stack.

.. index:: ! instruction

Instruction Set
===============

The instruction set of the EVM is kept minimal in order to avoid
incorrect implementations which could cause consensus problems.
All instructions operate on the basic data type, 256-bit words.
The usual arithmetic, bit, logical and comparison operations are present.
Conditional and unconditional jumps are possible. Furthermore,
contracts can access relevant properties of the current block
like its number and timestamp.

.. index:: ! message call, function;call

Message Calls
=============

Contracts can call other contracts or send Ether to non-contract
accounts by the means of message calls. Message calls are similar
to transactions, in that they have a source, a target, data payload,
Ether, gas and return data. In fact, every transaction consists of
a top-level message call which in turn can create further message calls.

A contract can decide how much of its remaining **gas** should be sent
with the inner message call and how much it wants to retain.
If an out-of-gas exception happens in the inner call (or any
other exception), this will be signalled by an error value put onto the stack.
In this case, only the gas sent together with the call is used up.
In Solidity, the calling contract causes a manual exception by default in
such situations, so that exceptions "bubble up" the call stack.

As already said, the called contract (which can be the same as the caller)
will receive a freshly cleared instance of memory and has access to the
call payload - which will be provided in a separate area called the **calldata**.
After it has finished execution, it can return data which will be stored at
a location in the caller's memory preallocated by the caller.

Calls are **limited** to a depth of 1024, which means that for more complex
operations, loops should be preferred over recursive calls.

.. index:: delegatecall, callcode, library

Delegatecall / Callcode and Libraries
=====================================

There exists a special variant of a message call, named **delegatecall**
which is identical to a message call apart from the fact that
the code at the target address is executed in the context of the calling
contract and ``msg.sender`` and ``msg.value`` do not change their values.

This means that a contract can dynamically load code from a different
address at runtime. Storage, current address and balance still
refer to the calling contract, only the code is taken from the called address.

This makes it possible to implement the "library" feature in Solidity:
Reusable library code that can be applied to a contract's storage, e.g. in
order to  implement a complex data structure.

.. index:: log

Logs
====

It is possible to store data in a specially indexed data structure
that maps all the way up to the block level. This feature called **logs**
is used by Solidity in order to implement **events**.
Contracts cannot access log data after it has been created, but they
can be efficiently accessed from outside the blockchain.
Since some part of the log data is stored in `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_, it is
possible to search for this data in an efficient and cryptographically
secure way, so network peers that do not download the whole blockchain
("light clients") can still find these logs.

.. index:: contract creation

Create
======

Contracts can even create other contracts using a special opcode (i.e.
they do not simply call the zero address). The only difference between
these **create calls** and normal message calls is that the payload data is
executed and the result stored as code and the caller / creator
receives the address of the new contract on the stack.

.. index:: selfdestruct

Self-destruct
=============

The only possibility that code is removed from the blockchain is
when a contract at that address performs the ``selfdestruct`` operation.
The remaining Ether stored at that address is sent to a designated
target and then the storage and code is removed from the state.

.. warning:: Even if a contract's code does not contain a call to ``selfdestruct``,
  it can still perform that operation using ``delegatecall`` or ``callcode``.

.. note:: The pruning of old contracts may or may not be implemented by Ethereum
  clients. Additionally, archive nodes could choose to keep the contract storage
  and code indefinitely.

.. note:: Currently **external accounts** cannot be removed from the state.

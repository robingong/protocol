# State Channels 状态通道

State channels allow entities to communicate with each other with the goal of
collectively computing some function `f`.状态通道允许实体相互通讯，来完成某项功能f。
This `f` can be as simple as "send 0.1 coins every minute" or it could represent
a decentralised exchange. These functions are, in our case, represented by smart
contracts and just like any legal contract, we need an arbiter in case one party
tries to act maliciously. This arbiter is the blockchain.这个功能f可以是发送coin，或去中心化交易。我们这里，f是智能合约，像法律合同，我们需要仲裁者（arbiter） 来处理恶意操作。

- trustless 去信任
- off chain vs on chain 链下 vs 链上
- same guarantees as blockchain 担保


## Table of Contents

- [Goals](#goals)
	- [Privacy](#privacy)
	- [Security](#security)
	- [Speed](#speed)
	- [Cost](#cost)
- [Terms](#terms)
- [High level overview](#high-level-overview)
- [Channel types](#channel-types)
- [Topology](#topology)
- [Incentives](#incentives)
- [Channel types](#channel-types)
- [Artefacts](#artefacts)
- [Protocol](#protocol)
- [Communication](#communication)
	- [Overview](#overview)
- Messages
	- [Off-chain](./OFF-CHAIN.md)
	- [On-chain](./ON-CHAIN.md)
- [Contract execution in channels](#contract-execution)
- [Light node requirements](#light-node-requirements)
- [Examples](#examples)
- [Future work](#future-work)
- [References](#references)


## Goals


- generic solution that supports many (all?) smart contracts 支持很多种智能合约
	- even better if one state channel is generic, s.t. it can be instantiated and
	  then be used for many different contracts如果状态通道是通用的，也可以被实例化用在不同的合约（场景）里。
- composability of channels, i.e. an application works for the
  A <-> B case should also work in the A <-B-> C (A to C via B) case  通道能组合
- upgradeable without having to close/re-open the channel 不关/重开 通道，就能升级通道
- the previous should enable state channels to be long lived 通道可以长期存在
	- this is at odds with privacy 上面这点是和隐私性矛盾的
- on chain operations should be kept to a minimum （主？）链上操作应该尽量少
- participants state should also be kept to a minimum, i.e. O(log n) or better
  constant multiplier 参与者的状态应该尽量少上链
- trust-less to a degree with blockchain as the arbiter 去信任 是某种程度的“区块链作为仲裁者”
- any party involved can close a channel, but it should be discouraged 参加通道的任意方读可以关闭通道，但是不鼓励

Generally, an ideal state channel design should be a strict improvement over
handling interactions on chain. The dimensions we are going to use to measure
improvements are as follows:
一般地，理想的状态通道设计对链上交易是一个明确的提升。从下面4个维度来衡量
- Privacy 隐私
- Security 安全
- Speed 速度
- Cost 成本


### Privacy 隐私

On-chain interactions offer little to no privacy, since we currently make no
efforts to hide interacting parties or the nature of their interactions.链上建议没有隐私

State channels, at the least, reveal that two parties establish a channel and
the amount of coins they commit to the channel, which is no improvement.
In the case that neither party tries to cheat, the exact nature of their
interactions stays unrevealed, since a mutual closing of a channel does not
require publishing of any state.状态通道 不对外披露状态？

With that in mind, state channels offer a slight improvement in privacy.状态通道在隐私方面有少量改进。


### Security 安全

State channels offer almost the same security guarantees as normal on-chain
transactions. 状态通道能提供几乎和链上交易一样的安全

- ***Liveness***: both peers can independently initiate closing of a channel and
  that operation is then processed by the blockchain with the usual assumptions
  of liveness.节点可以独立初始化“通道的关闭”，区块链一般假设通道还活着来处理操作
- ***Trust-less***: since operations need to be signed by both peers and they
  sign operations based on their view of the state, both parties will only
  sign operations they agree on and don't need to trust the other peer. 
操作需要双方签名，他们基于自己的状态视图，进行签名；双方仅仅对他们同意的操作签名，不用信任对方。


Thus security is on par.


### Speed 速度

Opening a state channel incurs遭受 the normal on-chain latency but after the channel
has been established, operations can be executed as fast as both peers can
process them, which should be a major improvement over on-chain interactions.
开启通道要承受链上延迟，建立后，就很快

### Cost 成本

Using state channels requires at least two on-chain transactions, for opening
and closing them. Once a channel is established, no further on-chain
transactions are needed unless a peer wants to withdraw or deposit funds. Even
in the case of an one off transaction, it still might be worth opening a channel
if one already has other open channels and thus might stand to gain fees by
relaying messages.


## Terms术语

We try to follow the naming conventions used by the lightning network, wherever
it makes sense, in hope to be able to make it easier for others to adopt,
although we try to enforce a `who - what - how` rule for naming, e.g.
`channel_close_solo` as opposed to `solo_close_channel` or `channel_solo_close`.

- ***node***: client connected to the blockchain, that can be addressed via an
  IP address and port
- ***peer***: participant in a channel
- ***channel***: an off-chain method for two peers to exchange state updates,
  each node can have multiple channels and a pair of nodes can also have
  multiple channels between each other, which should be multiplexed over one
  connection


## Notation

All objects on the blockchain have a type and are uniquely addressable. We will
denote this by `Type(Id)`, e.g. `Account(A)` is the account at address `A`. If
we then want to get the balance of that account we use `Account(A).balance`


## Channel types 通道类型

The most generic kind of channel would be one that lets peers instantiate any
arbitrary smart contract within the channel and does not restrict the number of
peers that can participate in such a channel. 最常见的通道可以让双方随着通道初始化智能合约，并不限制通道参与方的数量

Our construction will try to meet the former property, allowing any number of
arbitrary smart contracts to be executed in the channel, but restrict the latter
to two peers per channel.允许通道里任意数量任何合约执行，但是通讯只发生在每个通道里的2个参与方

With that in mind, the two peers in a channel will most likely not have the
same roles but instead end up in a client-server arrangement for the majority of
channels, where a client is using a service offered by the server, which is
highly available and probably also some well known entity, mirroring the status
quo of the current web. 通道里的2个参与方不太会有相同的权限，有点像client-server这种协议
A popular example would be an exchange, where users connect to an exchange via
state channels. This would process would be trustless, since exchanges can not
lose funds, that haven't been signed over to them
比如 一个人通过状态通道连到交易所。

- offer (verified) library for standard functionality, e.g. simple payments


## Topology 拓补

It is still very much unclear, what the topology of a widely used channel
network would look like but it seems that a hub and spoke model would be the
most probable. This model seems more likely than a decentralised one because the
majority of users would not be able to offer reliable services, longer paths
through the network would lock up significantly more funds and ostensibly also
incur a higher forwarding fee.

In the hub and spoke model, we would have a number of big hubs, which would be
involved in most routes through the network. These hubs would be tightly
connected and offer highly available and short paths for most users. In turn, this would lead to a loss of privacy and making the network more fragile, since
the disappearance of one of these hubs would have a big impact.


## Incentives 激励

Operating a channel should be considered collaborative game with incentive for cooperation

Operating a channel takes at least two on-chain operations and therefore has a base amount of fees is required and this fact could be abused by a malicious peer.操作通道至少2个链上操作，需要一些基础费用，可能会被恶意用户滥用。

(***TODO***: To discourage malicious behaviour, a successful slashing of a
channel closing, forfeits the malicious party's funds to the slasher but how can
we make that work without opening attack vectors?)

If a malicious peer doesn't want to risk losing all its channel balance, it
still has other options to disrupt or annoy others. Since closing a channel
incurs a fee, they could open a channel with someone and subsequently refuse to
cooperate, locking up coins and making the other party pay the fees required to
close the channel again.

With that in mind, it is advised for users to be cautious with whom they open
channels if they want to avoid the above issues.

On the other hand, attributing every failure in a channel to malice would also
be detrimental and cause both a loss of trust in the system and cause users to
spend more on fees than they should.


## Artefacts

What should the outcome of a state channel be? The simplest answer would be a
change in on-chain balances of participants but it could also be desirable to
use state channels as a poor man's MPC and have a contract with a non-empty
state as the result on chain. (***TODO:*** would there be any way to verify that
a state was actually produced by the given smart contract without having all the
inputs?)


## Fees

If we consider state channels to be long lived objects, then problems can arise
around the handling of fees, that need to be paid for on chain transactions.

Given that all parties need to sign of everything, a malicious party could
potentially black mail a peer under some circumstances.
If fees come from the channel balance, then one might end up in situations,
where the channel balance is lower than the fee required for miners to pick up
the transaction. The upshot here is, that the black mailed peer would lose less
coins than the cost of an on chain transaction, which should not be a lot.
If the fee instead comes from the account sending the transaction, then that
account might end up without enough coins and thus possibly end up with
insufficient funds to close the channel. This could potentially be worse since
the channel might hold significant funds.

Under these circumstances it seems deducting fees directly from the channel balance as part of its closing seems like the most sensible approach.

To avoid this situation, a peer should consider halting any interactions if on-chain fees reach a point, where the fees required to close a channel in a timely
manner approach the balance of the channel.


## High level overview

Participating in a state channel requires two nodes to be able to communicate
with each other. Throughout this document we are going to assume that this
happens via TCP/IP.
The process of discovering the `IP_ADDR:PORT` pair of a remote node is not
covered in this document and assumed to happen out of band, unless both of them are already part of the state channel network, where nodes can announce their identities `(IP_ADDR, PORT, ID)`.

To open a channel, two nodes first establish a connection. If they already have
an open channel, then creating a new one can be done via the same connection,
using a new `temporary_channel_id` and multiplexing the messages.


## Protocol

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).


## Communication

Communication between participants of a channel is peer to peer and SHOULD
happen via a reliable and ordered protocol, e.g. TCP. Peers should expect to be
running their own node, in order to be able to catch transactions relevant to
the channels they are involved in, although outsourcing that job to a third party, while still being trustless, might be possible in the future.

Messages will be both on- and off-chain.

Off-chain communication MUST be encrypted. To satisfy this, we use the same
transport protocol used for sync, which offers encryption and authentication.

Each pair of nodes SHOULD have at most one open connection. Channels can be
multiplexed easily, given that each channel has a unique id, so re-using
connection does not pose any problems.


### Overview 概览

The following diagram should give an overview of the off-chain state machine.
The starting point for any channel is the `closed` state.

```
+---+ channel_open                                                         +---+
|   | -------------> +-------------+                 error                 |   |
|   |                | initialized | ------------------------------------> |   |
|   | <------------- +-------------+                                       |   |
|   |   [timeout]/          | channel_accept                               |   |
|   |   [disconnect]        v                                              |   |
|   |                +-------------+                 error                 |   |
|   | <------------- |  accepted   | ------------------------------------> |   |
| c |   [timeout]/   +-------------+                                       | e |
| l |   [disconnect]        | funding_created                              | r |
| o |                       v                                              | r |
| s |                +-------------+                 error                 | o |
| e | <------------- | half_signed | ------------------------------------> | r |
| d |   [timeout]/   +-------------+                                       |   |
|   |   [disconnect]        | funding_signed                               |   |
|   |                       v                                              |   |
|   |                +-------------+  [disconnect]                         |   |
|   | <------------- |   signed    | -------------+  error                 |   |
|   |   [timeout]    +-------------+ -------------|----------------------> |   |
|   |                       |    |  shutdown      |                        |   |
|   |        funding_locked |    +------------+   |                        |   |
|   |                       |             |   |                        |   |
|   |                       |                 |   v                        |   |
|   |                       v        [disconnect] +--------------+  error  |   |
|   |                +-------------+ ---------|-> | disconnected | ------> |   |
|   | <------------- |    open     |          |   +--------------+         |   |
|   |   [timeout]    +-------------+ <--------|------------+               |   |
|   |                 ^ |      |       channel_reestablish                 |   |
|   |       update_*/ | |      |              |                            |   |
|   |       ping/pong +-+      | shutdown     v                            |   |
|   |                          |    +-------------+ [disconnect]           |   |
|   |                          +--> |   closing   | ------------+          |   |
|   |                               +-------------+             |          |   |
|   |                                |  ^                       |          |   |
|   |                                |  | channel_reestablish   |          |   |
|   | <------------------------------+  |                       v          |   |
|   |       closing_signed              |         +--------------+  error  |   |
|   |                                   +-------- | disconnected | ------> |   |
+---+                                             +--------------+         +---+
  ^                                                                          |
  |                                                                          |
  +--------------------------------------------------------------------------+
```

***Note***: Terms in square brackets `[]` are not explicit messages defined by
the protocol but part of the underlying transport protocol.


On-chain

```
                   +--------------+
               +-> |    closed    | <----+
               |   +--------------+      |
               |      | channel_create   |
               |      v                  | channel_close_mutual
               |   +--------------+      |
channel_settle |   |     open     | -----+
               |   +--------------+
               |    ^ |    |
               |    | | channel_withdraw/
               |    | | channel_deposit
               |    +-+    |
               |           | channel_close_solo
               |           v
               |   +--------------+
               +-- |    closing   | ----+
                   +--------------+     | channel_slash
                                 ^      |
                                 +------+
```


## Light node requirements


## Future Work


## References

[1]: Lightning Network RFC: https://github.com/lightningnetwork/lightning-rfc

[2]: Miller, Andrew, et al. "Sprites and State Channels: Payment Networks that Go Faster than Lightning."

[3]: Malavolta, Giulio, et al. "Concurrency and privacy with payment-channel networks." Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2017.

[4]: Dziembowski, Stefan, et al. PERUN: Virtual Payment Channels over Cryptographic Currencies⋆. IACR Cryptology ePrint Archive, 2017: 635, 2017.

[5]: Roos, Stefanie, et al. "Settling Payments Fast and Private: Efficient Decentralized Routing for Path-Based Transactions." arXiv preprint arXiv:1709.05748 (2017).

[6]: Tremback, Jehan, and Zack Hess. "Universal payment channels." (2015).

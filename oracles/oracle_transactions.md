 [back](./oracles.md)
## Oracle transactions 预言机事务类型

Oracle transactions are of four types:4种
- Register 注册
- Extend 扩展
- Query 查询
- Response 响应

### Oracle register transaction 注册

An oracle operator can register an existing account as an oracle. 预言机操作器可以注册一个存在的帐号为预言机。

The transaction contains: 包括：
- The address that should be registered as an oracle (oracle_owner) + nonce 地址应该注册成 预言机+nonce
- Query format definition
- Response format definition
- Query fee (that should be paid for posting a query to the oracle). 查询费（向一个预言机发起查询 应该支付查询费）
- A TTL (relative in number of blocks, or absolute block height) TTL (和块数相关，或和绝对块高度相关）
- Transaction fee (must be larger than a minimum proportional to the TTL)交易（事务）费（应该大于一个比例）

Questions/Later:
- Exact format for queries (API).
  - Protobuf or something similar?
- Exact format for responses.
- Fee for posting a query. The fee can be 0.
  - Fees are flat to begin with.
  - We could imagine fees to be proportional to something.

```
{ oracle_owner    :: public_key()
, nonce           :: nonce()
, query_format    :: format_definition()
, response_format :: format_definition()
, query_fee       :: amount()
, fee             :: amount()
, ttl             :: ttl()
}
```

#### TODO
- In the future we could imagine an oracle register transaction that 
  creates a new account by double signing the request with the source
  account and the new account.将来我们想在注册预言机事务的时候，新生成一个账户，再用原来账户和这个新账户 双重签名 这个（创建）请求
- Decide on the minimum fee calculation. 确定最小费用的计算公式。

### Oracle extend transaction 扩展事务

An oracle operator can extend the TTL of an existing oracle.预言机操作器扩展 现存预言机的TTL

The transaction contains:
- The address/oracle that should be extended (and a nonce)
- An extension to the TTL (relative to current expiry in number of blocks)
- Transaction fee (must be larger than a minimum proportional to the TTL)

Questions/Later:
- Fee for posting a query. The fee can be 0.
  - Fees are flat to begin with.
  - We could imagine fees to be proportional to something.

```
{ oracle :: public_key()
, nonce  :: nonce()
, ttl    :: ttl()
, fee    :: amount()
}
```

#### TODO
- In the future we could imagine an oracle register transaction that
  creates a new account by double signing the request with the source
  account and the new account.
- Decide on the minimum fee calculation.

### Oracle query transaction 预言机查询事务
- Contains:
  - The sender (address) + nonce
  - The oracle (address)
  - The query in binary format
  - The query fee - locked up until either:
    - The oracle answers and receive the fee
    - The TTL expire and the sender gets a refund
  - Query TTL
  - Response TTL
  - The transaction fee

The transaction creates an oracle interaction object in the oracle
state tree.本事务在预言机状态树上创建 预言机交互对象。 The id of this object is constructed from the query
transaction as the hash of {sender_address, nonce, oracle_address} 该对象的ID是查询事务的 {sender_address, nonce, oracle_address} 的hash。

The query TTL decides how long the query is open for response from the
oracle.查询的TTL 决定了这个查询（对响应）开放多长时间。

The query TTL can be either absolute (in block height) or relative
(also in block height) to the block the query was included in. 查询的TTL既可以是绝对的（区块绝对高度），也可以是相对的（查询被包裹的那个区块的相对高度）

The response TTL decides how long the response is available when given
from the oracle. The response TTL is always relative. This is to not
give incentive to the oracle to post the answer late, since the oracle
is paying the fee for the response.响应的TTL决定了该响应 从发出后多长时间是可获得的。响应的TTL是相对的

### Questions/Later
- Should the query format be checked by the miner?  矿工是否要检查查询的格式
- The size of this TX is variable (the Query), so fee will vary - also the
sender will pay for storing the query, thus the TTL will also affect the
transaction fee.
- We could include a staged fee that pays more to the oracle if it
responds earlier.
- We could include an earliest time that the oracle can answer to
protect against malicious oracles answering early and collect the fee.


```
{ sender_address  :: public_key()
, nonce           :: nonce()
, oracle_address  :: public_key()
, query           :: binary()
, query_fee       :: amount()
, query_ttl       :: ttl()
, response_ttl    :: relative_ttl()
, fee             :: amount()
}
```

### Oracle response 预言机响应

The oracle operator responds to a query by posting an oracle response
transaction, signing it with the oracle account's private key. 预言机操作器发布预言机响应事务，并对它进行私钥签名。

The response transaction is invalid if the TTL from the query has
expired.如果查询的TTL过期，那么响应事务就是失效的。（？）

The oracle pays the fee of the response transaction. The mininimum fee
is determined by the response TTL from the query and the size of the
response.预言机支付响应事务的费用。费用的最小值 被决定于 查询发起-响应返回的TTL 和 响应的大小。

Note that there is an incentive（动机，激励） to keep the response precise (and
small) since the oracle pays for the response transaction.保障响应精确（并且足够小），是靠预言机要为响应事务付费。

The transaction contains
- The oracle interaction ID (derived from the query)
- The oracle (address) + nonce
- The response in binary format
- The transaction fee

```
{ interaction_id  :: tx_id()
, oracle_address  :: public_key()
, nonce           :: nonce()
, response        :: binary()
, fee             :: amount()
}
```

### Questions/Later:

- Should we have an automatic callback defined in the query?
  - Any callback is paid by the oracle.
- Should we be able to return parts of the fee if the oracle for some
    reason could not provide an answer.
- Should there be a generic _DATA TX_ and should the Oracle answer be a
    special instance of this transaction.

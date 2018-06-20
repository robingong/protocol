[back](./oracles.md)
## Oracle state tree

The oracle state tree contain oracle objects and oracle query objects. The
existence of an oracle and the existence of query/response will have to be
proven when two parties contract with each other. Oracles and queries are
stored in the same tree; where a query has an id that is the concatenation of
the oracle id and a query id. Thus the queries in effect form a subtree of the
oracle and can be iterated over, etc.
预言机状态树包含 预言机对象 和 预言机查询对象。两两关联。预言机和查询 都存在相同的树上（多个树？）；查询有一个id，这个id和预言机id相关联。查询会形成预言机子树，并可以完全迭代。

### Oracle state tree objects 预言机状态树对象

- The oracle state tree contains包含
  - Oracle objects 预言机对象
  - Query objects 查询对象

#### The oracle object 预言机对象

- Created by an oracle register transaction. 预言机注册事务 生成
- Deleted when its TTL expires. TTL过了后，删除
- Updated with a new (longer) TTL on oracle extend transaction. 预言机拓展事务 能延长TTL

```
{ owner           :: pubkey()
, query_format    :: type_spec()
, response_format :: type_spec()
, query_fee       :: amount()
, expires         :: block_height()
}
```

#### The oracle query object 预言机查询对象

- Created by an oracle query transaction. 预言机查询事务 生成
- Closed by an oracle response transaction. 预言机响应事务 关闭
- Immutable once it is closed. 一旦关闭，就不可修改
- Deleted when it expires (When the query expire for an open query, and when
the response expire for a closed query) 过期则删除

The expiry is determined at creation time by the query TTL, and the response
TTL in the oracle query transaction. If/When an oracle response transaction is
accepted on the chain, the expiry is updated according to the response TTL.
*Note:* If the maximal TTL of the query (query TTL + response TTL) is longer
than the TTL of the Oracle, then the query is *rejected*. 
预言机查询对象生命周期决定于 创建时间和查询TTL，以及查询事务里的响应TTL。当预言机响应事务被链接受后，过期时间就决定于响应TTL了。
注意：预言机查询对象最大的TTL（查询TTL+响应TTL）比预言机的TTL还要长，那么这个查询会被拒绝。

```
{ query_id       :: id()
, oracle_address :: pubkey()
, query          :: query()
, response       :: oracle_response()
, expires        :: block_height()
, response_ttl   :: relative_ttl()
, fee            :: integer()
}
```

### Oracle state tree update 预言机状态树更新

The oracle state tree is pruned when the TTL of an object is reached in the
height of the chain. We define the operation order as:状态树被删减，当对象的TTL达到链的那个高度时。

1. Delete the expired objects. Object should be deleted in ascending order of their IDs.
2. Insert new object in the transaction order of the block.

Note that the sorted order of the IDs is the same as the in-order
traversal of the tree.

### Handling of TTL of objects 维护 对象的TTL

We will keep a cache of the objects sorted by TTL and ID. Such cache
has the benefits:
- The next object to delete is the first object in the cache.
  - The TTL is lower than the block height -> We are done.
  - The TTL is equal to the block height -> Delete the object.
- The cache can be reconstructed by doing an in-order traversal of the
  oracle state tree.

### Pruning of oracle query objects 删减 预言机查询对象

If the oracle query has not been given a response, the poster of
the query should be refunded the oracle query fee. If the oracle has
responded, the oracle was already given the funds at response time.
如果预言机查询没有得到一个响应，那么查询应该被退还 预言机查询费。如果预言机得到响应，预言机已经被支付费用了。

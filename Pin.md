

# DAG Replication & Publishing Protocol

Protocol for representing, transporting and updating arbitrary [IPLD][] DAGs over time.

### Abstract

In decentralized applications **source of truth** is captured in the data itself, as opposed to a row in some database. This often leads to a less common architectures, where databases is mere index. Good litmus test is are you able to drop existing database and recreate exact replica from the data itself.

With above design goal following specification describes [IPLD][] DAG replication protocol designed for constrained environments, where peer-to-peer replication is impractical. It aims to provide following functionality:


1. Allow transfer of large DAGs in shards _(of desired size)_ across multiple network request and/or sessions.
1. Allow transient DAG representations, that is partially replicated DAGs or revisions of one with a traversable root. 
1. Allow for an uncoordinated multiplayer DAG creation/transfer with specific convergence properties.

### Motivation

All content in [IPFS][] is represented by interlinked [blocks][IPLD Block] which form hash-linked DAGs. _(Every file in [IPFS][] is an [IPLD][] DAG under the hood.)_

Many applications in the ecosystem have adopted [Content Addressable Archives (CAR)][CAR] as a transport format for (Sub)DAGs in settings where peer-to-peer replication is impractical due to network, device or other constraints. This approach proved effective in settings where CAR size limit is not a concern, however there are still many constrained environments (e.g. serverless stacks) where transferring large DAGs in single CAR is impractical or plain impossible.

Here we propose a DAG replication protocol that overcomes above limitations by transporting large DAGs in multiple causally ordered network requests and/or sessions by:

1. Encoding sub-DAGs in desired sized packets - shards.
3. Wrapping shards in casually ordered operations (which can be transported out of order).
4. Define casually ordered _publish_ operations that can be used to bind DAGs states to a globally unique identifier.
 
### Replication Protocol

Our replication protocol is defined in terms of atomic, immutable, content addressed "operations" which are wrapped in a container structure that adds casual ordering through hash-links. _(We define this container structure in the _Replica_ section below)_

#### Replica

Semantically replica represents a state (been replicated) at a specific node. It is defined in terms of an atomic **change** _(describe by enclosed operation)_ to the **prior** state. At the same time it is also a log of operations, execution of which will produce a state they describe.

It is described by a following [IPLD Schema]

```ipldsch
type Replica = {
  prior optional &Replica
  change Change
}

-- Due to lack of generics we define Instruction as Any
-- In practice there will be Instruction set specific replica
-- types
type Change = Any
```

Semantics can more accuratly be captured with a help of generics, for that reason we present typescript definition below

```ts
type Replica<Change> {
  prior?: Link<Replica<Change>>
  change: Change
}
```

#### DAG State

Arbitrary DAG can be described as a set of shards _(subset of DAG block)_ it is comprised of.

##### Append

With that insight we represent arbitrary DAG, even transient _(one that is in the middle of been transported across nodes)_ in terms of `Append` operations _(which add more shards to a prior state)_ described by following [IPLD Schema][]:

```ipldsch
type Append {
  type "append"
  -- MUST contain only unique &Shards alphabetically ordered
  -- by their base32 encoding
  shards [&Shard]
}
```

Protocol imposes additional constraint that `shards` list **MUST** contain only unique CIDs and they must be alphabetically ordered by their base32 string encoding. This constraint gurantees that same append operation will be addressed by the same CID.

##### Examples

###### Empty DAG

According to this definitions empty DAG can be represented by a following replica. _(It has no `prior` field because it is a first operation)_

```js
{ "change": { "type": "append", "shards": [] } }
```

Which in [DAG-CBOR][] encoding will be addressed by a following CID

```
bafyreihaskmlkagl5wmhocs5lhu2cbbdmym5wknaiwywnvnokkswppcmiy
```

###### Basic DAG

DAG representing [DAG-CBOR][] encoded `{ hello: "world" }` block can be encoded by:

1. Encoding `{ hello: "world" }` in [DAG-CBOR]
1. Encoding that block into shard in [CAR][] format
   ```
   bagbaierauhgb4pxfuejvgufxxczjn2o7foetzrlxvnnvjm5pdas2y27v3cua
   ```
1. Encoding replica with above change is in other example
   ```js
   Block.encode({
     value: {
       change: {
         type: "append",
         shards: [
           CID.parse('bagbaierauhgb4pxfuejvgufxxczjn2o7foetzrlxvnnvjm5pdas2y27v3cua')
         ]
       }
     },
     codec: CBOR,
     hasher: sha256
   })
   ```
 
   

#### Join

Applications with concurrent `Append`s may result in diverged replicas illustrated by this digram

```
 a1
 | 
 a2---+
 |    |
 |    |
 a3   b3---+
 |    |    |
 a4   b4  c4
```

In order to support uncoordinated `Append`s we will use additional `Join` operation in our state representation. Join represents a `DAG` consisting of shards from all the linked replicas and is defined by a following [IPLD Schema]

```ipldsch
type Join = {
  type "join"
  forks [&Replica]
}
```

Protocol imposes following additional constraints

1. List `forks` **MUST** contain only unique CIDs and they must be alphabetically ordered by their base32 string encoding.
2. `Replica` enclosing `Join` **MUST** have `prior` linked to a `replica` with a CID that comes out top in an alphabetical sort (in base32 encoding) and enclosed `Join` **MUST** omit that CID from `forks`.


Acconding to this definition all divergent replicas will converge to the same one if synchronized before next append


```
 a1
 | 
 a2---+
 |    |
 |    |
 a3   b3---+
 |    |    |
 a4   b4  c4
 |    |    |
 j5---+----+
```

> Note: New `j5` replica points to `a4` as it is sorts ahead of `b4` and `c4`, while enclosed `Join` operation links to `b4` and `c4`. Produced `Replica` will have same `CID` regardless of which out of three nodes create it.


### Shards

We will use term shard to describe set of [IPLD block][]s that are part of some [IPLD][] DAG. Shards may represent connected or disconnected set of blocks. It is defined by a following [IPLD Schema][]

```ipldsch
type Shard = {
  blocks [Any]
  roots optional [&Any]
}
```

Protocol implementation MAY choose desired [IPLD codec][](s) for shard encoding. Given the system constraints we are trying to address, we RECOMMEND [CAR] format as a baseline.

Shards according to this definition CAN be content addressed by [CID][], which is what we will exploit later.

> Arbitrary CAR files can be viewed as shards and MAY be addressed by CID with the `0x0202` multicodec code.
>
> E.g CID of the empty shard in CAR format comes out as
> `bagbaierawa335d45pwohko5s4fbut7nlfjavq2kan7z3gbzvm2k3zutifv5q`


 

### Publishing Protocol

Publishing protocol allows representing DAGs over time by allowing authorized peers to change state associated with a unique identifier.

Just like DAG state we represent it's state in terms of casually ordered operations - Replica of `Publish` operations. 

`Publish` operation associates DAG _(as defined by our protocol)_ with a specific "root" with a unique identifier, represented by [ed25519][] public key. It is defined by a following [IPLD Schema][]


```ipldsch
type Publish {
  type "publish"
  id ID
  -- Entry of the DAG (Must be contained by origin)
  link &Any
  -- DAG representation
  origin &Replica
  -- Shard containing root block (Must be contained by origin)
  shard optional &Shard
  -- UCAN with publish capability to this id
  -- (Root issuer must be same as id)
  proof UCAN 
}

-- Binary representation of the ed25519 public key
type ID = Bytes
```

##### Convergence

Concurrent publish operations would lead to multiple forks _(as with `Append`)_ which MUST be reconciled by establishing total order among `Publish` operations as follows:

1. Given replicas `Pn` and `Pm`, if all operations of `Pn` are included in `Pm` we say `Pn <= Pm` (`Pn` predates `Pm`).
1. Given replicas `Pn` and `Pm` where neither `Pn` nor `Pm` includes all operations of the other we establish total order by:
   1. Finding divergence point, common replica `Po`.
   2. Compare CID _(in base32 string encoding)_ of each `Px` `Po...Pn` with `Py` from `Po...Pm`. If `Px < Py` then `Px < Py` and we compare `Px+1` with `Py` otherwise `Py < Px` and we compare `Py+1` with `Px` etc.


##### Illustrations

Below we have peer `A` associating `a1`, `a2` and then `a3` records. Peer `B` publishes conflicting record `e1` concurrently with `k3`.

   
```
 A   B 
 .   .
g1.........1
 |   .
g2---+.....2
 |   |
g3   k1....3
```

According to our convergence algoritm order of operations can be interpolated as follows _(because `CIDof(g3) < CIDof(k1)`)_

```
A   B 
 .   .
g1.........1
 |   .    
g2---+.....2
 |   |
g3...|.....3
     |
     k1....4
```

That also implies that if `A` has become aware of `k1` it's next record `g4` will link to `k1` and not `g3`.

```
A    B 
 .   .
g1........1
 |   .
g2---+....2
 |   |
g3...|....3
     |
+----k1...4
|
g4........5
     
```

If `B` has published next record instead, event after becoming aware of `g3` it would still link to `k1` (as it sorts lower.

```
A    B 
 .   .
g1........1
 |   .
g2---+....2
 |   |
g3...|....3
     |
    k1....4
     |
    k2....5

```

In scenario where operation chains diverge further things are more complicated


```
 A      B
 .      .
 g1-----+
 |      |
 g2     e1
 |      |
 g3     k2
 |      |
 g4     e3
```

Inferred order projects as follows

```
 A      B
 .      .
 g1-----+......1 
 |      |
 |      e1.....2   (g2 > e1)
 |      |
 g2............3   (g2 < k2)
 |      |
 g3.....|......4   (g3 < k2)
 |      |
 g4.....|......5   (g4 < k2)
        |
        k2.....6
        |
        e3.....7
```

It is worth noting that while `g4` and `e3` were concurrent and `g4 > e3` we still end up with `e3` after `g4`. That is to stress that comparing just last updates alone is not enough for establishing an order because at `k2` order would have been `g3 < k2` while at `e3` it would have been `g3 > e3`. By comparing all the concurrent operations we can establish deterministic order.


[ed25519]:https://ed25519.cr.yp.to/
[UCAN]:https://whitepaper.fission.codes/access-control/ucan
[did:key]:https://w3c-ccg.github.io/did-method-key/
[IPLD Schema]:https://ipld.io/docs/schemas/
[IPNS]:https://github.com/ipfs/specs/blob/master/IPNS.md
[CAR]:https://ipld.io/specs/transport/car/carv1/
[Merkle CRDT]:https://research.protocol.ai/blog/2019/a-new-lab-for-resilient-networks-research/PL-TechRep-merkleCRDT-v0.1-Dec30.pdf
[CID]:https://docs.ipfs.io/concepts/content-addressing/
[ZDAG hearder compression]:https://github.com/mikeal/ZDAG/blob/master/SPEC.md#links_header_compression

[commutative]:https://en.wikipedia.org/wiki/Commutative_property
[idempotence]:https://en.wikipedia.org/wiki/Idempotence
[DAG-CBOR]:https://ipld.io/specs/codecs/dag-cbor/spec/
[IPLD]:https://ipld.io/specs/
[IPFS]:https://ipfs.io/
[IPLD Block]:https://ipld.io/glossary/#block
[IPLD codec]:https://ipld.io/specs/codecs/
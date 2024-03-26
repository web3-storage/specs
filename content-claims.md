# Content Claims Protocol

![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)

## Editors

- [Alan Shaw](https://github.com/alanshaw), [DAG House](https://dag.house/)

## Authors

- [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)
- [Mikeal Rogers](https://github.com/mikeal), [DAG House](https://dag.house/)

# Abstract

UCAN based protocol allowing actors to share information about specific content (identified by CID).

> We base the protocol on top of [UCAN invocation specification 0.2](https://github.com/ucan-wg/invocation/blob/v0.2/README.md#23-ipld-schema)
> [Original proposal](https://hackmd.io/IiKMDqoaSM61TjybSxwHog?view) (includes implementation notes)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Introduction

## Motivation

## NextGen IPFS Content Discovery

Content discovery in IPFS today is "peer based", meaning that an IPFS client first performs a "content discovery" step and then a "peer discovery" step to read data in the following way:

1. Client finds peerids that have announced they have a CID
2. Client finds transport protocols for the given peerid that it can use to read IPFS data from that peer.
3. Client retreives data from that peer.

![](https://hackmd.io/_uploads/r1pmgLuVh.png)

File reads in this system requires that two entities, the client and the 
peer serving the data, both support an IPFS aware transport protocol. 
The peer serving a given CID is required to serve data at a specific 
endpoint and keep that endpoint operational for data to be available.

When a publisher wants to hire a remote storage device this protocol design presents a challenge since these remote storage devices don’t speak native IPFS protocols, therefor IPFS clients can’t read content from them directly. We end up having to place a node between these remote storage devices and the client, proxying the data and incurring unnecessary egress.

![](https://i.imgur.com/Jan0Opg.png)

Content Claims change this.

Publishers post **verifiable claims** about IPFS data they publish in remote storage devices into Content Discovery networks and services.

From these claims, IPFS clients can request data directly from remote storage devices over standard transports like HTTP.

![](https://i.imgur.com/4iutSwA.png)


With content claims, any actor can provide data into the network without making the data available over an IPFS aware transport protocol. Rather than "serving" the data, content providers post verifiable claims about the data and its location.

Clients can use these claims to read directly, over any existing transport (mostly HTTP), and in the act of reading the data will verify the related claims.

This removes a **substantial** source of cost from providing content into the network. Content can be published into the network "at rest" on any permanent or temporary storage device that supports reading over HTTP.

# Protocol

## Claim Types

The requirements for content claims break down into a few isolated components, each representing a specific claim. These claims are assembled together to represent the proof information necessary for retrieving content.

All claim types map a **single** CID to "claim information."

While you can derive block indexes from these claims (see: Inclusion Claims), each individual claim is indexed by a single **significant** CID (file root, dir root, etc) referrenced by `content` field.

These claims include examples of a unixfs file  encoded into CAR files, but the protocol itself makes heavy use of CIDs in order to support a variety of future protocols and other use cases.

Since this protocol builds upon the UCAN Invocation Specification, all of these claims are contained within a message from an *issuer* to a *destination*. This means it can be used to send specific actors unique route information apart from public content discovery networks, and can also be used to send messages to public content discovery networks by simply addressing them to a DID representing the discovery network.

### Location Claims

* Claims that a CID is available at a URL.
* Block Level Interface, 
  * GET Body MUST match multihash digest in CID.
* Using CAR CID, or any future BlockSet() identifier, allows this to be a multi-block interface.

```javascript
{
  "op": "assert/location",
  "rsc": "https://web3.storage",
  "input": {
    "content" : CID /* CAR CID */, 
    "location": "https://r2.cf/bag...car",
    "range"   : [ start, end ] /* Optional: Byte Range in URL */
  }
}
```

From this, we can derive a verifiable Block interface over HTTP or any other URL based address. This could even be used with `BitSwap` using `bitswap://`.

Filecoin Storage Providers (running boost) would be addressable via the [lowest cost read method available (HTTP GET Range in Piece)](https://boost.filecoin.io/http-retrieval#retrieving-a-full-piece).

And you can provide multiple locations for the same content using a list.

```javascript
{
  "op": "assert/location",
  "rsc": "https://web3.storage",
  "input": {
    "content": CID /* CAR CID */, 
    "location": [ "https://r2.cf/bag...car", "s3://bucket/bag...car" ],
    "range": [ start, end ] /* Optional: Byte Range in URL */
  }
}
```

### Equivalency Claims

We also have cases in which the same data is referred to by another CID and/or multihash. Equivalency claims represent this association as a verifiable claim.

```javascript
{
  "op": "assert/equals",
  "rsc": "https://web3.storage",
  "input": {
    "content": CID /* CAR CID */, 
    "equals":  CID /* Commp CID */
  }
}
```

We should expect content discovery services to index these claims by `"content"` CID, since that is standard across all claims, but since this is a cryptographic equivalency, equivalency claim aware systems are encouraged to index both.

### Inclusion Claims

* Claims that a CID includes the contents claimed in another CID.
* Multi-Block Level Interface, 
  * One CID is included in another CID.
      * When that's a CARv2 CID, the CID also provides a block level index of the referenced CAR CIDs contents.
* Using CAR CIDs and CARv2 Indexes, get a multi-block interface.
* Combined with HTTP location information for the CAR CID, this means we can read individual block sections using HTTP Ranges.


```javascript
{
  "op": "assert/inclusion",
  "rsc": "https://web3.storage",
  "input": {
    "content":  CID /* CAR CID */,
    "includes": CID /* CARv2 Index CID */,
    "proof":    CID /* Optional: zero-knowledge proof */
  }
}
```

This can also be used to provide verifable claims for sub-deal inclusions in Filecoin.

```javascript
{
  "op": "assert/inclusion",
  "rsc": "https://web3.storage",
  "input": {
    "content":  CID /* PieceCID (CommP) */,
    "includes": CID /* Sub-Deal CID (CommP) */,
    "proof":    CID /* Sub-Deal Inclusion Proof */
  }
}
```

### Partition Claims

* Claims that a CID’s graph can be read from the blocks found in parts, 
  * `content` (Root CID)
  * `blocks` (List of ordered CIDs)
  * `parts` (List of archives [CAR CIDs] containing the blocks)

```javascript
{
  "op": "assert/partition",
  "rsc": "https://web3.storage",
  "input": {
    "content": CID /* Content Root CID */,
    "blocks": CID, /* CIDs CID */
    "parts": [
      CID /* CAR CID */,
      CID /* CAR CID */,
      ...
    ]
  }
}
```

# Consuming Claims

An IPFS client wishing to perform a verifiable `read()` of IPFS data can construct one from verifiable claims. Given the amount of cryptography and protocol expertise necessary to perform these operations a few examples are detailed below.

## IPFS File Publishing

An IPFS File is a contiuous set of bytes (source_file) that has been encoded into a merkle-dag. The resulting tree is referenced by CID using the `dag-pb` codec.

When files are encoded and published into the network, they are often packed into CAR files. One CAR file might contain **many** IPFS Files, and one large file could be encoded into **many** CAR Files. CAR files can be referenced by CID (digest of the CAR) and are typically exchanged transactionally (block level interface). This can get confusing, as transaction (block level) interfaces now ***contain*** multi-block interfaces.

### Large File

Large file uploads can be problematic when performed as a single pass/fail transaction, so it has become routine to break large upload transactions into smaller chunks. The default encoder for web3.storage (w3up) is configurable but defaults to ~100MB, so as the file is encoded into the unixfs block structure it is streamed to CAR encodings of the configured size. Each of these is uploaded transactionally, and if the operation were aborted and started again it would effectively resume as long as the file and settings hadn't changed.

When the encode is complete the root CID of the unixfs merkle tree will be available as a `dag-pb` CID. This means we need a claim structure that can cryptographically route from this `dag-pb` CID to the data ***inside*** the resulting CAR files.

Content Claims allow us to expose cryptography in the IPFS encoding to client such that clients can read **directly from the encoded CAR**, rather than requiring an intermediary to *assemble* the unixfs merkle tree for them.

So, after we stream encode a bunch of CARs and upload them, we encode and publish:
* A **Partition Claim** for the ***Content Root CID***, which includes:
    * An ordered list of every CID we encoded.
    * A list of CAR CIDs where those CIDs can be found.
* **Inclusion Claims** for every ***CAR CID***, which includes:
    * A CARv2 index of the CAR file.

Now, if we publish these CAR files to Filecoin, we're going to want to capture another address (CommP) for the CAR file. We then include an

* **Equivalency Claim** for every ***CAR CID***, which claims this CAR CID is equivalent to CommP for that CAR file.

Once that CAR data is aggregated into Filecoin deals, the list of CommP addresses included in each deal can be used to compute a sub-inclusion proof. At this time you may also publish:

* **Inclusion Claims** for every ***Sub-Deal CID*** (CommP) which claims each Sub-Deal CID is "included" in each Piece CID (also CommP), which includes:
    * The referenced sub-deal inclusion proof.

From this point forward, you can lookup Filecoin Storage Providers onchain 😁

The preferred (lowest cost) method of reading data from Storage Providers is through an HTTP interface that requests data by Piece CID. You could describe these locations, perhaps playing the role of chain oracle, as:

* **Location Claims** for every ***CAR CID*** in every Storage Provider, which includes:
    * The offsets in the Piece addressable by HTTP Range Header.

And of course, you can also use **Location Claims** for any other HTTP accessible storage system you ever decide to put a CAR into.

### *`read(DagPBCID, offset, length)`*

Now that we have a better idea of what sorts of claims are being published, let's construct a read operation from the claims.

It's out of scope for this specification to define the exact means by which you **discover** claims, but content discovery systems are expected to index these claims by `content` CID (in theory, you could index every block in the CARv2 indexes, but we should assume many actors don't want to pay for that, so everything in this specification works with only requires indexing the `content` CID in each claim). However, once a publisher sends these claims in public networks, they should presume all exposed addresses in the claims are "discoverable" from a security and privacy perspective.

Once you have the claims for a given `dag-pb` CID, like the claim examples above, you can:

* Build a Set() of CAR CIDs claimed to be holding relevant blocks and
* Build a Map() of inclusions indexed by CAR CID and
    * depending on your CARv2 library you may want to parse the CARv2 indexes into something you can read quickly from, and
* Build a Map() of locations indexed by CAR CID,
* and you may even be able to build those concurrently.

Take the list of blocks from the partition claim and find the smallest number of CAR CIDs containing a complete Set.

Using the CARv2 indexes, you can now determine the byte offsets within every CAR for every block. You should also take the time to improve the read performance by closing the distance between many of those offsets so that you have fewer individual reads for contiuous sections of data, that will also reduce the burden your client will put on data providers since the same data will be fetched in fewer requests.

You're now free to request this data from all CARs concurrently in a single round-trip, wherever they are.

#### From Remote HTTP Endpoint

You can use HTTP Range Headers to request the required block sections over HTTP. Keep in mind that CAR CID claims have support for offsets that would need to be included in the offset calculation you'd then need to perform *within* the CAR.

#### From Filecoin Sub-Inclusion Proofs

Pretty simple really:

* The **Equivalency Claims** tells you the sub-deal CID.
* The **Inclusion Claims** for those sub-deal CIDs give you the Piece CIDs that Storage Providers commit to on chain.

#### From BitSwap 🤯

One amazing thing about these protocols is that by simply using a BitSwap URL, we have BitSwap peers as "locations" for CAR CIDs in the same **Location Claim** protocol.

One could build a system/protocol for "loading" CAR files into BitSwap peers and then publish Location Claims.

Even over BitSwap, this would represent a performance improvement over the current client protocols as you avoid round-trips traversing the graph.

## IPLD Schema

```ipldsch

# For UCAN IPLD Schema see
# https://github.com/ucan-wg/ucan-ipld

type URL string
type URLs [URL]
type CIDs [&Any]

type Assertion union {
  | AssertLocation   "assert/location"
  | AssertInclusion  "assert/inclusion"
  | AssertPartition  "assert/partition"
} representation inline {
  discriminantKey "op"
}

type AssertLocation {
  on      URL
  input   ContentLocation
}
type AssertLocation {
  on      URL
  input   ContentInclusion
}
type AssertLocation {
  on      URL
  input   ContentPartition
}

type Range {
  start   Int
  end     Int
} representation tuple

type Locations union {
  | URL  string
  | URLs list
} representation kinded

type ContentLocation {
  content    &Any
  location   Locations
  range optional Range
}

type ContentInclusion {
  content        &Any
  includes       &Any
  proof optional &Any
}

type ContentPartition {
  # Content that is partitioned
  content    &Any
  # Links to the CARs in which content is contained
  parts      [&ContentArchive]
  # Block addresses in read order
  blocks     &CIDs
}
```

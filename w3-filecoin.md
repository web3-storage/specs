# W3 Filecoin Protocol

![status:reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Editors

- [Vasco Santos], [Protocol Labs]
- [Irakli Gozalishvili], [Protocol Labs]
- [Alan Shaw], [Protocol Labs]

## Authors

- [Vasco Santos], [Protocol Labs]

# Abstract

This spec describes a [UCAN] protocol allowing an implementer to receive an aggregate of CAR files for inclusion in a Filecoin deal.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Table of Contents

- [Terminology](#terminology)
  - [Roles](#roles)
    - [Storefront]
    - [Aggregator]
    - [Dealer]
    - [Deal Tracker]
- [Protocol](#protocol)
  - [Overview](#overview)
  - [Authorization](#authorization)
  - [Capabilities](#capabilities)
    - [Storefront Capabilities](#storefront-capabilities)
      - [`filecoin/offer`]
      - [`filecoin/submit`]
      - [`filecoin/accept`]
      - [`filecoin/info`]
    - [Aggregator Capabilities](#aggregator-capabilities)
      - [`piece/offer`]
      - [`piece/accept`]
    - [Dealer Capabilities](#dealer-capabilities)
      - [`aggregate/offer`]
      - [`aggregate/accept`]
    - [Deal Tracker Capabilities](#deal-tracker-capabilities)
      - [`deal/info`]
  - [Schema](#schema)
    - [Base types](#base-types)
    - [`filecoin/offer` schema](#filecoinoffer-schema)
    - [`filecoin/submit` schema](#filecoinsubmit-schema)
    - [`filecoin/accept` schema](#filecoinaccept-schema)
    - [`filecoin/info` schema](#filecoininfo-schema)
    - [`piece/offer` schema](#pieceoffer-schema)
    - [`piece/accept` schema](#pieceaccept-schema)
    - [`aggregate/offer` schema](#aggregateoffer-schema)
    - [`aggregate/accept` schema](#aggregateaccept-schema)
    - [`deal/info` schema](#dealinfo-schema)

# Terminology

## Roles

There are several roles in the authorization flow:

| Name         | Description                                                                                       |
| ------------ | ------------------------------------------------------------------------------------------------- |
| [Storefront]   | [Principal] identified by a DID, representing a storage API like web3.storage.                    |
| [Aggregator]   | [Principal] identified by a DID, representing a storage aggregator like w3filecoin.               |
| [Dealer]       | [Principal] identified by a DID, that arranges filecoin deals with storage providers. e.g. Spade. |
| [Deal Tracker] | [Principal] identified by a DID, that tracks deals made by the [Dealer].                            |

### Storefront

A _Storefront_ is a type of [principal] identified by a DID (typically a [`did:web`] identifier).

A _Storefront_ facilitates data storage services to applications and users, getting the requested data stored into Filecoin deals asynchronously.

### Aggregator

An _Aggregator_ is a type of [principal] identified by a DID. It is RECOMMENDED to use [`did:key`] identifier due to their stateless nature.

An _Aggregator_ facilitates data storage into Filecoin deals by aggregating smaller data (Filecoin Pieces) into a larger piece that can effectively be stored with a Filecoin Storage Provider using [Verifiable Data Aggregation](https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0058.md).

### Dealer

A _Dealer_ is a type of [principal] identified by a DID (typically a `did:key` identifier) that arranges deals for the aggregates submitted by [Storefront].

### Deal Tracker

A _Deal Tracker_ is a type of [principal] identified by a DID (typically a `did:key` identifier) that follows the filecoin chain to keep track of successful deals.

# Protocol

## Overview

A [Storefront] is a service that ensures content addressed user/application data is stored perpetually on the decentralized web. A [Storefront] ingests user/application data and replicates it across various storage systems, including Filecoin Storage Providers. Content supplied to a [Storefront] can be of arbitrary size, while (Filecoin) Storage Providers demand large (>= 16GiB) content pieces. To align supply and demand requirements, the [Aggregator] accumulates supplied content pieces into a larger verifiable aggregate pieces per [FRC-0058](https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0058.md) that can be stored by Storage Providers.

### Authorization

[Storefront]s MUST use UCAN based authorization mechanisms to interact with [Aggregator]s, [Dealer]s and [Deal Tracker]s. The way in which [Storefront]s are registered to use [Aggregator]s, [Dealer]s and [Deal Tracker]s is out of scope of this specification.

For example, an [Aggregator] can authorize invocations from `did:web:web3.storage` by validating the signature is from the DID. This way, it allows web3.storage to rotate keys and/or re-delegate access without having to coordinate with the [Aggregator].

### _Storefront_ receives a Filecoin piece

A [Storefront] MUST submit content for aggregation by its piece CID. It MAY be computed from content by a trusted actor or it MAY be computed by the [Storefront] itself. A [Storefront] MUST provide a capability that can be used to submit a piece to be replicated by (Filecoin) Storage Providers. It may be invoked by a [Storefront] client or delegated to a hired third party, ether way a [Storefront] MUST acknowledge request by issuing a signed receipt. A [Storefront] MAY decide to verify submitted piece prior to aggregation. A [Storefront] MAY also operate trusted actor that computes and submits pieces on content upload.

Once a [Storefront] receives the offer for a piece, it is pending for verification. The [Storefront] MUST issue a receipt proving that request state has transitioned from `uninitialized` to `pending` if result was `ok`, or to `failed` if result was `error`. The [Storefront] MAY fail invocation if piece `content` has not been provided.

#### `filecoin/accept` effect

A successful invocation receipt MUST have `fx.join` [effect] that links to the terminating task of the workflow. It allows the observer to lookup whether the offered piece has landed on filecoin or failed.

#### `filecoin/submit` effect

Successful invocation receipt MUST have a `fx.fork` [effect] that links to the next task of the workflow. It allows the observer to follow progress of the execution.

The [Storefront] MUST issue a receipt for the linked [`filecoin/submit`] task after it verifies the offered piece and queues it for aggregation. This receipt MUST have a `fx.join` [effect] that links to a [`piece/offer`] task that forwards the submitted piece to the [Aggregator].

![storefront flow](./w3-filecoin/storefront.svg)

### _Storefront_ offers a piece to aggregate

A [Storefront] SHOULD propagate offered pieces to Filecoin Storage Providers by forwarding them to an [Aggregator].

The [Aggregator] MUST queue offered pieces for an aggregation and issue a signed receipt proving that the piece is being `pending` to be added. The issued receipt MUST have a `fx.join` [effect] that links to a [`piece/accept`] task, which either succeeds with an (aggregate) inclusion proof or fails.

If the [Storefront] offers a piece multiple times, the [Aggregator] MUST respond with a receipt that contains the _same_ result and effect(s).

> ℹ️ An invocation nonce MAY be used to force a piece to be included in another aggregate.

The same Piece submitted by different [Storefront]s SHOULD NOT be considered a duplicate.

After an [Aggregator] includes a piece in an aggregate it MUST issue a [`piece/accept`] receipt with a piece inclusion proof as the result. The receipt MUST have a `fx.join` [effect] that links to an [`aggregate/offer`] task for the aggregate where piece was included.

![aggregator flow](./w3-filecoin/aggregator.svg)

### _Aggregator_ offers dealer an aggregate

When the [Aggregator] has enough content pieces to build a qualified aggregate ([Dealer]s MAY impose different requirements), it MUST offer an aggregate to the [Dealer]. The [Dealer] MUST issue a signed receipt acknowledging an offer, and then deal negotiation with Filecoin Storage Providers MAY be carried out of band.

If a [Dealer] receives a request with an aggregate multiple times it MUST (re)issue a receipt with the _same_ result and effects.

> ℹ️ An invocation nonce MAY be used to force an aggregate to be reprocessed.

The issued receipt MUST have a `fx.join` [effect] linking to an [`aggregate/accept`] task which either succeeds with filecoin [`DataAggregationProof`] result or fails (e.g. if a Storage Provider failed to replicate and reported an error).

The [Dealer] MUST broker deal(s) with Filecoin Storage Providers (out of band). It MUST issue a receipt for the [`aggregate/accept`] task with a successful or failed result depending on the availability of Storage Providers and their ability to replicate content pieces in the aggregate. A successful task MUST have a [`DataAggregationProof`] as it's result and contain no [effect]s.
A failed task MUST provide an error reason. When pieces of the aggregate can be retried, the issued receipt MUST contain `fx.fork` [effect]s with [`piece/offer`] tasks per piece.

> Note: The [Dealer] MAY have several intermediate steps and states it transitions through, however those are _not_ captured by this protocol intentionally, because the other actor take no action until a success / failure condition is met.

![dealer flow](./w3-filecoin/dealer.svg)

### _Deal Tracker_ can be queried for the aggregate status

[Storefront] users MAY want to check status of the deals for their content. Deals change over time as they get renewed. Therefore, the [Storefront] MAY invoke [`deal/info`] capability to gather information about an aggregate. The [Storefront] SHOULD be able to look up an aggregate from received inclusion proofs and use them to look up deal status information.

The [Dealer] MAY also use a [Deal Tracker] to poll for status of the aggregates to obtain proof that deals have made it onto a chain and to issue [`aggregate/accept`] receipts when they do.

![dealer tracker flow](./w3-filecoin/deal-tracker.svg)

## Capabilities

This section describes the set of capabilities that form the w3 filecoin protocol, along with the details relevant for invoking them with a service provider.

In this document, we describe capabilities provided by [Storefront] `web3.storage`, [Aggregator] `aggregator.web3.storage`, [Dealer] `dealer.web3.storage` and [Deal Tracker] `tracker.web3.storage`.

### _Storefront_ Capabilities

#### `filecoin/offer`

An agent MAY invoke the `filecoin/offer` capability to request storing a content piece in Filecoin. See [schema](#filecoinoffer-schema).

> `did:key:zAliceAgent` invokes `filecoin/offer` capability provided by `did:web:web3.storage`

```json
{
  "iss": "did:key:zAliceAgent",
  "aud": "did:web:web3.storage",
  "att": [
    {
      "with": "did:key:zAlice",
      "can": "filecoin/offer",
      "nb": {
        /* CID of the uploaded content */
        "content": { "/": "bag...car" },
        /* Commitment proof for piece */
        "piece": { "/": "bafk...commp" }
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

The [Storefront] MAY fail the invocation if the linked `content` has not yet been stored in the given space.

```json
{
  "ran": "bafy...filOffer",
  "out": {
    "error": {
      "name": "ContentNotFoundError",
      "content": { "/": "bag...car" }
    }
  }
}
```

Alternatively, the [Storefront] MAY choose to queue request until linked `content` has been uploaded.

[Storefront] MUST issue a signed receipt for a successful invocation acknowledging the request (regardless if it already has a `content` or if it chose to wait for an upload).

##### Effects

The issued receipt MUST have a `fx.join` [effect] that links to the [`filecoin/accept`] task. The [Storefront] MUST issue the receipt for this task once the content piece is aggregated and the deal is published to the filecoin chain.

> This allows an agent to get a result without having to follow progress across the entire invocation chain.

The issued receipt MUST have a `fx.fork` [effect] that links to the [`filecoin/submit`] task. The [Storefront] MUST issue a receipt for this task once it has processed the request and queued it for aggregation, or failed with an error (implying a problem with the piece or content).

> This allows an agent to follow progress across the entire invocation chain.

```json
{
  "ran": "bafy...filOffer",
  "out": {
    /* commitment proof for piece */
    "ok": { "piece": { "/": "bafk...commp" } }
  },
  "fx": {
    "join": { "/": "bafy...filAccept" },
    "fork": [{ "/": "bafy...filSubmit" }]
  },
  "meta": {},
  "iss": "did:web:web3.storage",
  "prf": []
}
```

##### Implementations

- `@web3-storage/capabilities` [`filecoin/offer` capability validator](https://github.com/web3-storage/w3up/blob/adb24424d1faf50daf2339b77c22fdd44faa236a/packages/capabilities/src/filecoin/storefront.js#L20)
- `@web3-storage/filecoin-api` [storefront `filecoin/offer` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/storefront/service.js#L20)

#### `filecoin/accept`

This task is effectively a shortcut allowing an observer to find out the result of the [`filecoin/offer`] task chain without having to follow each step. The [Storefront] MUST issue a signed receipt with a [`DataAggregationProof`] result or an error.

##### Filecoin Accept Failure

```json
{
  "ran": "bafy...filAccept",
  "out": {
    "error": {
      "name": "InvalidContentPiece",
      "content": { "/": "bafk...commp" }
    }
  }
}
```

##### Filecoin Accept Success

```json
{
  "ran": "bafy...filAccept",
  "out": {
    "ok": {
      /* commitment proof for piece */
      "piece": { "/": "commitment...car" },
      /* commitment proof for aggregate */
      "aggregate": { "/": "commitment...aggregate" },
      /** inclusion proof for piece */
      "inclusion": {
        "tree": {
          "path": [
            "bafk...root",
            "bafk...parent",
            "bafk...child",
            "bag...car"
          ],
          "at": 1
        },
        "index": {
          "path": [/** ... */],
          "at": 7
        }
      },
      /** deal type */
      "aux": {
        "dataType": 0,
        "dataSource": {
          "dealID": 1245
        }
      }
    }
  }
}
```

##### Implementations

- `@web3-storage/capabilities` [`filecoin/accept` capability validator](https://github.com/web3-storage/w3up/blob/adb24424d1faf50daf2339b77c22fdd44faa236a/packages/capabilities/src/filecoin/storefront.js#L82)
- `@web3-storage/filecoin-api` [storefront `filecoin/accept` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/storefront/service.js#L123)

#### `filecoin/submit`

The task MUST be invoked by the [Storefront] which MAY be used to verify the offered content piece before propagating it through the pipeline.
> `did:web:web3.storage` invokes capability from `did:web:web3.storage`

```json
{
  "iss": "did:web:web3.storage",
  "aud": "did:web:web3.storage",
  "att": [
    {
      "with": "did:web:web3.storage",
      "can": "filecoin/submit",
      "nb": {
        /* CID of uploaded content */
        "content": { "/": "bag...car" },
        /* commitment proof for piece */
        "piece": { "/": "bafk...commp" }
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

A [Storefront] MUST issue a signed receipt that either succeeds and links to the [`piece/offer`] task via a `fx.join` [effect] or fails with specified reason (e.g. the `content` does not correspond to the provided `piece`).

```json
{
  "ran": "bafy...filSubmit",
  "out": {
    "ok": {
      /* commitment proof for piece */
      "piece": { "/": "bafk...commp" } 
    }
  },
  "fx": {
    "join": { "/": "bafy...pieceOffer" }
  },
  "meta": {},
  "iss": "did:web:web3.storage",
  "prf": []
}
```

See the [`piece/offer`] section for the subsequent task.
If the added piece is invalid, the reason for the failure is also reported:

```json
{
  "ran": "bafy...filSubmit",
  "out": {
    "error": {
      "name": "InvalidPieceCID",
      "message": "...."
    }
  },
  "fx": {
    "fork": []
  },
  "meta": {},
  "iss": "did:web:web3.storage",
  "prf": []
}
```

##### Implementations

- `@web3-storage/capabilities` [`filecoin/submit` capability validator](https://github.com/web3-storage/w3up/blob/adb24424d1faf50daf2339b77c22fdd44faa236a/packages/capabilities/src/filecoin/storefront.js#L50)
- `@web3-storage/filecoin-api` [storefront `filecoin/submit` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/storefront/service.js#L82)

#### `filecoin/info`

An agent MAY invoke the `filecoin/info` capability to request known information about content piece in Filecoin. See [schema](#filecoininfo-schema).

> `did:key:zAliceAgent` invokes `filecoin/info` capability provided by `did:web:web3.storage`

```json
{
  "iss": "did:key:zAliceAgent",
  "aud": "did:web:web3.storage",
  "att": [
    {
      "with": "did:key:zAlice",
      "can": "filecoin/info",
      "nb": {
        /* Commitment proof for piece */
        "piece": { "/": "bafk...commp" }
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

##### Filecoin Info Failure

```json
{
  "ran": "bafy...filInfo",
  "out": {
    "error": {
      "name": "InvalidContentPiece",
      "content": { "/": "bafk...commp" }
    }
  }
}
```

##### Filecoin Info Success

```json
{
  "ran": "bafy...filInfo",
  "out": {
    "ok": {
      /* commitment proof for piece */
      "piece": { "/": "commitment...car" },
      /* aggregates where this piece was included */
      "aggregates": [
        {
          /* commitment proof for aggregate */
          "aggregate": { "/": "commitment...aggregate" },
          /** inclusion proof for piece */
          "inclusion": {
            "tree": {
              "path": [
                "bafk...root",
                "bafk...parent",
                "bafk...child",
                "bag...car"
              ],
              "at": 1
            },
            "index": {
              "path": [/** ... */],
              "at": 7
            }
          }
        }
      ],
      /* filecoin deals where this piece was included */
      "deals": [
        {
          /* commitment proof for aggregate */
          "aggregate": { "/": "commitment...aggregate" },
          /** deal type */
          "aux": {
            "dataType": 0,
            "dataSource": {
              "dealID": 1245
            }
          }
        }
      ]
    }
  }
}
```

##### Implementations

- `@web3-storage/capabilities` [`filecoin/info` capability validator](https://github.com/web3-storage/w3up/blob/adb24424d1faf50daf2339b77c22fdd44faa236a/packages/capabilities/src/filecoin/storefront.js#L114)
- `@web3-storage/filecoin-api` [storefront `filecoin/info` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/storefront/service.js#L239)

### _Aggregator_ Capabilities

#### `piece/offer`

A [Storefront] can invoke a capability to offer a piece to be aggregated for upcoming Filecoin deal(s). See [schema](#pieceoffer-schema).

> `did:web:web3.storage` invokes capability from `did:web:aggregator.web3.storage`

```json
{
  "iss": "did:web:web3.storage",
  "aud": "did:web:aggregator.web3.storage",
  "att": [
    {
      "with": "did:web:web3.storage",
      "can": "piece/offer",
      "nb": {
        /* commitment proof for piece */
        "piece": { "/": "bafk...commp" },
        /* grouping of joining segments into an aggregate */
        "group": "did:web:free.web3.storage"
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

An [Aggregator] MUST issue a signed receipt to acknowledge the received request. The receipt MUST contain an `fx.join` [effect] linking to [`piece/accept`] task that MUST either succeed with an [`InclusionProof`] or fail with an error describing the reason.

```json
{
  "ran": "bafy...pieceOffer",
  "out": {
    "ok": {
      /* commitment proof for piece */
      "piece": { "/": "bafk...commp" },
    }
  },
  "fx": {
    "join": { "/": "bafy...pieceAccept" }
  },
  "meta": {},
  "iss": "did:web:aggregator.web3.storage",
  "prf": []
}
```

See the [`piece/accept`] section for the subsequent task.

##### Implementations

- `@web3-storage/capabilities` [`piece/offer` capability validator](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-client/src/aggregator.js#L22)
- `@web3-storage/filecoin-api` [aggregator `piece/offer` method](https://github.com/web3-storage/w3up/blob/main/packages/filecoin-api/src/aggregator/service.js#L19)

#### `piece/accept`

An [Aggregator] MUST issue a receipt for the [`piece/accept`] task for the offered piece that was included in an aggregate. The receipt MUST contain an [`InclusionProof`] in the result and an `fx.join` [effect] linking to [`aggregate/offer`] task, or an error detailing the reason.

> It is RECOMMENDED to never fail [`piece/accept`] as piece inclusion is a deterministic computation occurring on validated data.

```json
{
  "ran": "bafy...pieceAccept",
  "out": {
    "ok": {
      /* commitment proof for piece */
      "piece": { "/": "commitment...car" },
      /* commitment proof for aggregate */
      "aggregate": { "/": "commitment...aggregate" },
      /** inclusion proof */
      "inclusion": {
        "tree": {
          "path": [/** ... */],
          "at": 4
        },
        "index": {
          "path": [/** ... */],
          "at": 7
        }
      }
    }
  "fx": {
    "join": { "/": "bafy...aggregateOffer" }
  },
  "meta": {},
  "iss": "did:web:aggregator.web3.storage",
  "prf": []
}
```

##### Implementations

- `@web3-storage/capabilities` [`piece/accept` capability validator](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-client/src/aggregator.js#L66)
- `@web3-storage/filecoin-api` [aggregator `piece/accept` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/aggregator/service.js#L62)

### _Dealer_ Capabilities

#### `aggregate/offer`

An [Aggregator] can offer an aggregate for Filecoin deal inclusion by invoking an [`aggregate/offer`] capability. See [schema](#aggregateoffer-schema).

> `did:web:aggregator.web3.storage` invokes capability from `did:web:dealer.web3.storage`

```json
{
  "iss": "did:web:aggregator.web3.storage",
  "aud": "did:web:dealer.web3.storage",
  "att": [
    {
      "can": "aggregate/offer",
      /* storefront responsible for invocation */
      "with": "did:web:web3.storage",
      "nb": {
        /* commitment proof for aggregate */
        "aggregate": { "/": "bafk...aggregate-proof" },
        /* dag-cbor CID with content pieces */
        "pieces": { "/": "bafy...many-cars" },
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

Invoking the [`aggregate/offer`] capability is a request to arrange Filecoin deals for the aggregate.

The `nb.aggregate` field represents a commitment proof for the `aggregate` to arrange a deal(s) for.

The `nb.pieces` field represents a link to a DAG-CBOR encoded list of pieces of an `aggregate`. The elements of the `nb.pieces` field MUST be sorted in the _same_ order as they were used to compute the aggregate piece CID. This block MUST be included with the invocation. Its format is:

```json
/* offers block as an array of piece CIDs, encoded as DAG-JSON (for readability) */
[
  { "/": "commitment...car0" } /* COMMP CID */,
  { "/": "commitment...car1" } /* COMMP CID */
  /* ... */
]
```

Each entry of the decoded offers block has all the necessary information for a Storage Provider to fetch and store a CAR file.

The [Dealer] MUST issue a signed receipt to acknowledge the request. The issued receipt MUST have an `fx.join` [effect] linking to the `deal/accept` task which MUST succeed with the [`DataAggregationProof`] after deals are live on the Filecoin chain or fail (with an `error` describing the problem with the `aggregate`).

```json
{
  "ran": "bafy...aggregateOffer",
  "out": {
    "ok": {
      /* commitment proof for aggregate */
        "aggregate": { "/": "bafk...aggregate-proof" },
    }
  },
  "fx": {
    "join": { "/": "bafy...aggregateAccept" }
  },
  "meta": {},
  "iss": "did:web:dealer.web3.storage",
  "prf": []
}
```

See the [`aggregate/accept`](#aggregateaccept) section for the subsequent task.

##### Implementations

- `@web3-storage/capabilities` `aggregate/offer` [capability validator](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/capabilities/src/filecoin/dealer.js#L20)
- `@web3-storage/filecoin-api` [`piece/accept` method creates `aggregate/offer` delegation](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/aggregator/service.js#L91)
- `@web3-storage/filecoin-api` [dealer service `aggregate/offer` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/dealer/service.js#L14)

#### `aggregate/accept`

The [Dealer] MUST issue a receipt for the [`aggregate/accept`] task once it arranges deals with Storage Providers and they are live on the Filecoin chain. The receipt MUST either succeed with the [`DataAggregationProof`] or fail (with an `error` describing the problem with the `aggregate`).

```json
{
  "ran": "bafy...aggregateAccept",
  "out": {
    "ok": {
      /* commitment proof for aggregate */
      "aggregate": { "/": "bafk...aggregate-proof" },
      "dataType": 0,
      "dataSource": {
        "dealID": 1245
      }
    }
  },
  "fx": {},
  "iss": "did:web:dealer.web3.storage",
  "meta": {},
  "prf": []
}
```

If a deal fails due to an invalid piece, the issued receipt MUST contain `fx.fork` [effect]s that retry valid pieces.

> ℹ️ This allows an observer to follow the new execution chain even if the original piece inclusion failed.

```json
{
  "ran": "bafy...aggregateAccept",
  "out": {
    "error": {
      "name": "InvalidPiece",
      "message": "....",
      /* commitment proof */
      "aggregate": { "/": "bafk...aggregate-proof" },
      "cause": [
        {
          "name": "InvalidPieceCID",
          "message": "....",
          "piece": { "/": "bafk...car0" }
        }
      ]
    }
  },
  "fx": {
    "fork": [
      { "/": "bafy...piece1Offer" },
      { "/": "bafy...piece2Offer" },
      /** ... */
    ]
  },
  "iss": "did:web:dealer.web3.storage",
  "meta": {},
  "prf": []
}
```

##### Implementations

- `@web3-storage/capabilities` [`aggregate/accept` capability validator](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/capabilities/src/filecoin/dealer.js#L48)
- `@web3-storage/filecoin-api` [dealer `aggregate/offer` method ➡ `aggregate/accept` effect](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/dealer/service.js#L69)
- `@web3-storage/filecoin-api` [dealer `aggregate/accept` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/dealer/service.js#L92)

### _Deal Tracker_ Capabilities

#### `deal/info`

A [Storefront] and an [Aggregator] MAY invoke a [`deal/info`] capability to request the current state of the aggregate.

> `did:web:web3.storage` invokes capability from `did:web:tracker.web3.storage`

```json
{
  "iss": "did:web:web3.storage",
  "aud": "did:web:tracker.web3.storage",
  "att": [
    {
      "with": "did:web:web3.storage",
      "can": "deal/info",
      "nb": {
        /* commitment proof */
        "piece": { "/": "commitment...aggregate-proof" }
      }
    }
  ],
  "nnc": "lmpxywjr",
  "prf": [],
  "sig": "..."
}
```

> ⚠️ The invoker SHOULD utilize a nonce on subsequent calls to avoid receiving a response for the prior invocation.

The invocation to the [Deal Tracker] MUST succeed and return deal information for the aggregate if it is on chain.

```json
{
  "ran": "bafy...dealInfo",
  "out": {
    "ok": {
      "deals": {
        "111": {
          "storageProvider": "f07...",
          "status": "Active",
          "pieceCid": "bag...",
          "dataCid": "bafy...",
          "dataModelSelector": "Links/...",
          "activation": "2023-04-13T01:58:00+00:00",
          "expiration": "2024-09-05T01:58:00+00:00",
          "created": "2023-04-11T17:57:30.522198+00:00",
          "updated": "2024-04-12T03:42:26.928993+00:00"
        }
      }
    }
  },
  "fx": {
    "fork": []
  },
  "meta": {},
  "iss": "did:web:spade.storage",
  "prf": []
}
```

The invocation to the [Deal Tracker] MUST fail if the deal information for the aggregate is _not_ on chain.

```json
{
  "ran": "bafy...dealInfo",
  "out": {
    "error": {
      "name": "DealNotFound"
      /* ... */
    }
  },
  "fx": {
    "fork": []
  },
  "meta": {},
  "iss": "did:web:spade.storage",
  "prf": []
}
```

##### Implementations

- `@web3-storage/capabilities` `deal/info` [capability validator](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/capabilities/src/filecoin/deal-tracker.js#L17)
- `@web3-storage/filecoin-api` [deal-tracker service `deal/info` method](https://github.com/web3-storage/w3up/blob/e34eed1fa3d6ef24ce2c01982764f2012dbf30d8/packages/filecoin-api/src/deal-tracker/service.js#L15)

## Schema

### Base types

```ipldsch
type FilecoinCapability union {
  | FilecoinOffer "filecoin/offer"
  | FilecoinSubmit "filecoin/submit"
  | FilecoinAccept "filecoin/accept"
} representation inline {
  discriminantKey "can"
}

type PieceCapability union {
  | PieceOffer "piece/offer"
  | PieceAccept "piece/accept"
} representation inline {
  discriminantKey "can"
}

type AggregateCapability union {
  | AggregateOffer "aggregate/offer"
  | AggregateAccept "aggregate/accept"
} representation inline {
  discriminantKey "can"
}

type DealCapability union {
  | DealInfo "deal/info",
} representation inline {
  discriminantKey "can"
}

type PieceRef struct {
  piece PieceLink
}

type AgentDID string
type StorefrontDID string
type AggregatorDID string
type DealerDID string
type DealTrackerDID string

# from a fr32-sha2-256-trunc254-padded-binary-tree multihash
type PieceLink Link
type Content Any

type AggregatePieces [PieceCid]
```

### `filecoin/offer` schema

```ipldsch
type FilecoinOffer struct {
  with AgentDID
  nb FilecoinOfferDetail
}

type FilecoinOfferDetail struct {
  # CID of file previously added to resource space
  content &Content
  # Piece as Filecoin Piece with padding
  piece PieceLink
}
```

### `filecoin/submit` schema

```ipldsch
type FilecoinSubmit struct {
  with AgentDID
  nb FilecoinOfferDetail
}

type FilecoinSubmitDetail = FilecoinOfferDetail
```

### `filecoin/accept` schema

```ipldsch
type FilecoinAccept struct {
  with AgentDID
  nb FilecoinAcceptDetail
}

type FilecoinAcceptDetail = FilecoinOfferDetail
```

### `filecoin/info` schema

```ipldsch
type FilecoinInfo struct {
  with AgentDID
  nb FilecoinInfoDetail
}

type FilecoinInfoDetail struct {
  # Piece as Filecoin Piece with padding
  piece PieceLink
}
```

### `piece/offer` schema

```ipldsch
type PieceOffer struct {
  with AgentDID
  nb PieceOfferDetail
}

type PieceOfferDetail struct {
  # Piece as Filecoin Piece with padding
  piece PieceLink
  # grouping for joining segments into an aggregate (subset of space)
  group string
}
```

### `piece/accept` schema

```ipldsch
type PieceAccept struct {
  with AgentDID
  nb PieceAcceptDetail
}

type PieceAcceptDetail = PieceOfferDetail
```

### `aggregate/offer` schema

```ipldsch
type AggregateOffer struct {
  with StorefrontDID
  nb AggregateOfferDetail
}

type AggregateOfferDetail struct {
  # Contains each individual piece within Aggregate piece
  pieces &AggregatePieces
  # Piece as Aggregate of CARs with padding
  aggregate PieceLink
}
```

### `aggregate/accept` schema

```ipldsch
type AggregateAccept struct {
  with StorefrontDID
  nb AggregateAcceptDetail
}

type AggregateAcceptDetail = AggregateOfferDetail
```

### `deal/info` schema

```ipldsch
type DealInfo struct {
  with StorefrontDID
  nb DealInfoDetail
}

type DealInfoDetail struct {
  # Piece as Aggregate of CARs with padding
  aggregate PieceLink
}
```

[`did:web`]: https://w3c-ccg.github.io/did-method-web/
[`did:key`]:https://w3c-ccg.github.io/did-method-key/
[UCAN]: https://github.com/ucan-wg/spec/
[principal]: https://github.com/ucan-wg/spec/#321-principals
[Protocol Labs]: https://protocol.ai/
[Vasco Santos]: https://github.com/vasco-santos
[Irakli Gozalishvili]: https://github.com/Gozala
[Alan Shaw]: https://github.com/alanshaw
[effect]:https://github.com/ucan-wg/invocation/#7-effect
[`DataAggregationProof`]:https://github.com/filecoin-project/go-data-segment/blob/e3257b64fa2c84e0df95df35de409cfed7a38438/datasegment/verifier.go#L8-L14
[`InclusionProof`]:https://github.com/filecoin-project/go-data-segment/blob/e3257b64fa2c84e0df95df35de409cfed7a38438/datasegment/inclusion.go#L30-L39
[Storefront]:#storefront
[Aggregator]:#aggregator
[Dealer]:#dealer
[Deal Tracker]:#deal-tracker

[`filecoin/offer`]:#filecoinoffer
[`filecoin/submit`]:#filecoinsubmit
[`filecoin/accept`]:#filecoinaccept
[`filecoin/info`]:#filecoininfo
[`piece/offer`]:#pieceoffer
[`piece/accept`]:#pieceaccept
[`aggregate/offer`]:#aggregateoffer
[`aggregate/accept`]:#aggregateaccept
[`deal/info`]:#dealinfo

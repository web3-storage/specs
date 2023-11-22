# Filecoin Deal Signing Protocol

![status:wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Protocol Labs]

## Authors

- [Irakli Gozalishvili], [Protocol Labs]

# Abstract

This specification describes a [UCAN] protocol allowing a broker like [spade] to request signing a storage deal.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Terminology

## Roles

There are several roles that actors can have in this protocol:

| Name        | Description |
| ----------- | ----------- |
| Storefront | [Principal] identified by [`did:web`] identifier, representing a storage aggregator like w3up |
| Broker   | [Principal] that arranges filecoin deals with storage providers like [spade] |
| Agency | [Principal] submitting pieces to the _Broker_ like spade-proxy |

### Storefront

A _Storefront_ is a type of [principal] identified by a [`did:web`] that aggregates user data into [aggregate][] pieces and submits those to the broker to arrange deals with storage providers.

### Broker

A _Broker_ is a type of [principal] identified that arranges deals for the aggregates submitted by _Storefront_.

# Protocol

## Overview

All the filecoin deals need to be signed by a Fil wallet, in order to avoid passing private keys to wallet _Storefront_ could delegate a capability to a sign a deal to a _Broker_ instead.

Here we propose set of UCAN capabilities that can be invoked by authorized actors (like _Agency_ or a _Broker_) to sign deals on behalf of the delegate (_Storefront_).

## IPLD Schema

```ipldsch
type Deal union {
  | Sign "deal/sign"
} representation inline {
  discriminantKey "can"
}

type Sign struct {
  with    StorefrontDID
  nb      DealProposal
}

# @see https://github.com/filecoin-project/go-state-types/blob/ff2ed169ff566458f2acd8b135d62e8ca27e7d0c/builtin/v9/market/deal.go#L201-L221
# We use capital case letters because that the way go likes them
type DealProposal struct {
  PieceCID              &Piece
  PieceSize             PaddedPieceSize
  VerifiedDeal          bool
  # Signer wallet (f0) address
  Client                Address
  # Storage provider wallet (f0) address for whom the contract is made
  Provider              Address
  # Label is an arbitrary client chosen label to apply to the deal
  Label                 DealLabel

  # Nominal start epoch. Deal payment is linear between StartEpoch and EndEpoch,
  # with total amount StoragePricePerEpoch * (EndEpoch - StartEpoch).
  # Storage deal must appear in a sealed (proven) sector no later than StartEpoch,
  # otherwise it is invalid.
  StartEpoch            ChainEpoch
  EndEpoch              ChainEpoch
  StoragePricePerEpoch  TokenAmount

  ProviderCollateral    TokenAmount
  ClientCollateral      TokenAmount
}

type StorefrontDID string

# Piece CID is Piece / Aggregate merkle root encoded as CID 
type Piece = Any

# @see https://github.com/filecoin-project/go-state-types/blob/ff2ed169ff566458f2acd8b135d62e8ca27e7d0c/abi/piece.go#L12
type PaddedPieceSize = Uint64

# @see https://github.com/filecoin-project/go-address/blob/master/address.go#L39-L40
type Address struct { addr: string }


# The DealLabel is a kinded union of string or byte slice.
# It serializes to a CBOR string or CBOR byte string depending on which form it
# takes.
# The zero value is serialized as an empty CBOR string (maj type 3).
# @see https://github.com/filecoin-project/go-state-types/blob/ff2ed169ff566458f2acd8b135d62e8ca27e7d0c/builtin/v9/market/deal.go#L37C1-L44C1
type DealLabel struct {
 bs        Bytes
 notString bool
}

# @see https://github.com/filecoin-project/go-state-types/blob/ff2ed169ff566458f2acd8b135d62e8ca27e7d0c/abi/chain.go#L9C1-L10
type ChainEpoc = Int64

# @see https://github.com/filecoin-project/go-state-types/blob/ff2ed169ff566458f2acd8b135d62e8ca27e7d0c/abi/chain.go#L16-L22
type TokenAmount = SerializedBigInt

# In principal IPLD Int range is unlimited, but in practice it is
# something implementations need to worry about so it's best to
# specify
# @see https://ipld.io/design/tricky-choices/numeric-domain/#integers
type Uint64 = Int
type Int64 = Int
# Looks like an Int serialized into a string if I'm not mistaken
# @see https://github.com/filecoin-project/go-state-types/blob/master/big/int.go#L294-L303
type SerializedBigInt = string
```

## Ucanto Interface

```ts
import { Piece, PaddedPieceSize, Uint68 } from "@web3-storage/data-segment"
import { Invocation, ToString, InvocationError } from "@ucanto/interface"

export interface Deal {
  (invocation: Invocation<{ can: "deal/sign", with: StorefrontDID, nb: DealProposal }>): Result<Signature, SignError>
}

// Note that we use capital case field names for compatibility with go
export interface DealProposal {
  Piece: Piece.Link
  Size: Piece.PaddedSize

  VerifiedDeal: boolean

  Client: Address
  Provider: Address
  Label: DealLabel

  StartEpoch: ChainEpoch
  EndEpoch: ChainEpoch
  StoragePricePerEpoch: TokenAmount

  ProviderCollateral: TokenAmount
  ClientCollateral: TokenAmount
}

export interface Address {
  addr: string
}

export interface DealLabel {
 bs: Uint8Array
 notString: boolean
}

export type ChainEpoch = Uint68
export type TokenAmount = ToString<bigint>

export type Signature = Uint8Array

export type SignError =
  | InvocationError
  | InvalidDeal // ProposalRemarshalMismatch
  | SigningError // WalletSignError
```

## Capabilities

### `deal/sign`

Broker can invoke `deal/sign` capability with `DealProposal` in (`nb` field). Storefront MUST encode supplied `DealProposal` (`nb` field) into a CBOR block and then sign it with a wallet private key.

Provider MUST respond with raw bytes of the signature.

## HTTP Interface

Given that some actors (e.g. Spade) do not support UCANs natively they are not able to send signed invocations. As compromise protocol implementer is RECOMMENDED to expose plain HTTP API that trade-offs some security for convenience of interop.

Implementation MUST expose HTTP POST endpoint that accepts `application/vnd.ucan.cbor` requests with CBOR encoded as payload. These requests MUST provide `Authorization: Bearer` header with a UCAN delegation authorizing a request. Receiving principal MUST derive invocation from the provided `Authorization` and set invoked capability `nb` field to decoded CBOR block of the payload. Receiving principal MUST execute received capability and encode result of the invocation as an HTTP response.

> Above HTTP interface could be utilized by spade to obtain signatures from w3up without having to proxy them through UCAN proxy.

## Interaction Flow

### Authorization per aggregate

_Storefront_ (w3up) MAY delegate `deal/sign` UCAN capability to the _Agency_ (spade-proxy) and specify `Piece` and `Size` fields of the submitted aggregate.

_Agency_ (spade-proxy) could also re-delegate that capability to the _Broker_ (spade) allowing it to request signature directly from _Storefront_ (w3up).

> ℹ️ Since ♠️ Spade does not support UCANs (yet), _Agency_ could instead create an invocation UCAN and pass it on to Spade so it could be used by spade as a plain, but short lived, JWT token for signing that specific deal.

### Long term authorization

_Storefront_ (w3up) MAY delegate unconstrained `deal/sign` UCAN capability to the _Agency_ (spade-proxy). By leaving out `Piece` and `Size` fields it will authorize it to sign any deals.

_Agency_ (spade-proxy) could also re-delegate same unconstrained `deal/sign` UCAN capability to the _Broker_ (spade) allowing it to sign any deals.

This trade-offs increased security for convenience.

[spade]:https://github.com/ribasushi/spade
[`did:web`]: https://w3c-ccg.github.io/did-method-web/
[UCAN]: https://github.com/ucan-wg/spec/
[principal]: https://github.com/ucan-wg/spec/#321-principals

[Protocol Labs]:https://protocol.ai/
[Irakli Gozalishvili]:https://github.com/Gozala
[aggregate]:https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0058.md#specification

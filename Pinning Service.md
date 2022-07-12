# Web3 Pinning Service

We reimagine IPFS [pinning service][] as web3 service, where:

- Users delegate neccessary capabilities to the pinning service as opposed to getting [access token][]s from the service which must be kept secret.
- As an API layer on top of core upload v2 protocol.


Below table maps [IPFS pinning service API][pinning service] operations to capabilities necessary to execute them

| Operation | Capabilities |
|-----------|--------------|
|[add](https://ipfs.github.io/pinning-services-api-spec/#tag/pins/paths/~1pins/post)|[`store/add`][]|
|[remove](https://ipfs.github.io/pinning-services-api-spec/#tag/pins/paths/~1pins~1{requestid}/delete)|[`store/remove`][]|
|[list](https://ipfs.github.io/pinning-services-api-spec/#tag/pins/paths/~1pins/get)|[`store/list`][]|
|[replace](https://ipfs.github.io/pinning-services-api-spec/#tag/pins/paths/~1pins~1{requestid}/post)|[`store/add`][] [`store/remove`][]|
|[get](https://ipfs.github.io/pinning-services-api-spec/#tag/pins/paths/~1pins~1{requestid}/get)|[`store/list`][]|

Our pinning service implementation will accept [access tokens][] that are valid JWT formatted [UCAN][]s where:

1. Root issuer is the same DID as one in `with` field of the delegated capabilities.
2. DID in `with` field is associated with some account.
1. Where all the provided [proofs are embedded](https://github.com/ipld/js-dag-ucan#embedding-proofs) inline.

:::danger
It is worth calling out that just like typical [access token][]s these do not need to be kept secret as well, since if compromised they could be used to pin arbitrary data. 
:::


This offers following advantages to typical bearer tokens:

1. Users are able to delegate access to [pinning service] to others.
2. Users could tokens with subset of capabilities and consequently restrining access to desired operations.
3. Users could revoke and rotate tokens as they wish.

:::info
This also creates an oportunity for implementing better [pinning service][] clients issue short lived tokens per operation and remove the need for keeping tokens secret. 
:::

> [@ipld/dag-ucan](https://www.npmjs.com/package/@ipld/dag-ucan) library could be used to issue such tokens


### Request ID

Pinning service uses `requestid` field to uniquely identify pinning requests. We extend IPFS [pinning service][] specification with additional constraint. 

Pin request MUST be identified with a `requestid` derived from the pin request. More specifically it shoud be a CID of the CBOR encoded `Pin` object with a `sha256` (multi)hash. In typescript this can be encoded as follows:

```ts
import type { Link, DID } from "@ipld/dag-ucan"
import * as CBOR from "@ipld/dag-cbor"
import { sha256 } from "multiformats/hashes/sha2"

type RequestID = Link<Pin, typeof CBOR.code, typeof sha256.code> {}

interface Pin {
  cid: CID
  // Optional name for pinned data (can be used for lookups)
  name?: string
  // Optional list of multiaddrs known to provide the data
  origins?: string[]
  // Optional metadata for pin object
  meta: { group: DID } & Record<string, string>
}


```

> Please note [`Link`][link-type] is a type for CID destination type hint and is equivalent of [Link](https://ipld.io/docs/schemas/features/links/#link-destination-type-hinting) IPLD Schema.

### Implementation Notes

This design frees [pinning service][] from doing any kind of access control or token validation. It simply needs to

1. Parse [access token][] as a UCAN.
2. Ensure that `audience` corresponds to own DID.
3. Verify that UCAN is valid (has right signature, has not expired etc)
4. Create UCAN invocation (corresponding to received request) and embed UCAN (from [access token][]) as a proof.

While UCAN validation is not strictly necessary (as they get verified downstream anyway) it might be a good idea to avoid unecessary work downstream.


[string literal pattern]:https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html
[pinning service]:https://ipfs.github.io/pinning-services-api-spec/
[link-type]:https://github.com/ipld/js-dag-ucan/blob/364379b54cae383198fcf6a9c0016b497e62d422/src/ucan.ts#L227-L242
[access token]:https://ipfs.github.io/pinning-services-api-spec/#section/Authentication/accessToken
[`store/add`]:https://github.com/web3-storage/ucanto/blob/w3/w3/store/src/type/store.ts#L76-L78
[`store/remove`]:https://github.com/web3-storage/ucanto/blob/w3/w3/store/src/type/store.ts#L80-L82
[`store/list`]:https://github.com/web3-storage/ucanto/blob/w3/w3/store/src/type/store.ts#L84
[UCAN]:https://github.com/ucan-wg/spec/
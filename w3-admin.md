# Admin Protocol

![status:reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square)

## Editors

- [Travis Vachon](https://github.com/travis), [Protocol Labs](https://protocol.ai/)

## Authors

- [Travis Vachon](https://github.com/travis), [Protocol Labs](https://protocol.ai/)

## Abstract

Storage providers in the w3 family of protocols need to be able to get information about the customers, subscriptions and "consumers" (i.e., spaces)
they work with. The capabilities described in this document all act on the "service" resource (i.e., `did:web:web3.storage`) and can be delegated
to administrative users by creating delegations signed with the service signer's private key.

- [Capabilities](#capabilities)
  - [`consumer/get`](#consumerget)
  - [`customer/get`](#customerget)
  - [`subscription/get`](#subscriptionget)
  - [`admin/upload/inspect`](#adminuploadinspect)
  - [`admin/store/inspect`](#adminstoreinspect)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Capabilities

### `consumer/get`

Get information about a consumer (i.e., a space).

#### inputs

`consumer: SpaceDID`

#### returns

```typescript
{
  did: SpaceDID
  allocated: number
  limit: number
  subscription: string
}
```

#### errors

`ConsumerNotFound`

#### capability definition

```javascript
export const get = capability({
  can: 'consumer/get',
  with: ProviderDID,
  nb: struct({
    consumer: SpaceDID,
  }),
  derives: (child, parent) => {
    return (
      and(equalWith(child, parent)) ||
      and(equal(child.nb.consumer, parent.nb.consumer, 'consumer')) ||
      ok({})
    )
  },
})
```

#### Implementations

- @web3-storage/capabilities: [capability in consumer.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/capabilities/src/consumer.js#L29C1-L29C1)
- @web3-storage/upload-api: [invocation handler in consumer/get.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/upload-api/src/consumer/get.js)

### `customer/get`

Get information about a customer.

#### inputs

`customer: DID<mailto>`

#### returns

```typescript
{
  did: AccountDID
  subscriptions: string[]
}
```

#### errors

`CustomerNotFound`

#### capability definition

```javascript
export const get = capability({
  can: 'customer/get',
  with: ProviderDID,
  nb: struct({
    customer: AccountDID,
  }),
  derives: (child, parent) => {
    return (
      and(equalWith(child, parent)) ||
      and(equal(child.nb.customer, parent.nb.customer, 'customer')) ||
      ok({})
    )
  },
})
```

#### Implementations

- @web3-storage/capabilities: [capability in customer.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/capabilities/src/customer.js#L12)
- @web3-storage/upload-api: [invocation handler in customer/get.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/upload-api/src/customer/get.js)

### `subscription/get`

Get information about a subscription.

#### inputs

`subscription: string`

#### returns

```typescript
{
  customer: DID<mailto>
  consumer?: SpaceDID
}
```

#### errors

`SubscriptionNotFound`

#### capability definition

```javascript
export const get = capability({
  can: 'subscription/get',
  with: ProviderDID,
  nb: struct({
    subscription: Schema.string(),
  }),
  derives: (child, parent) => {
    return (
      and(equalWith(child, parent)) ||
      and(equal(child.nb.subscription, parent.nb.subscription, 'consumer')) ||
      ok({})
    )
  },
})
```

#### Implementations

- @web3-storage/capabilities: [capability in subscription.js](https://github.com/web3-storage/w3up/blob/main/packages/capabilities/src/subscription.js)
- @web3-storage/upload-api: [invocation handler in subscription/get.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/upload-api/src/subscription/get.js)

### `admin/upload/inspect`

Get information about a content CID. This does not return the actual data identified by the CID, just metadata our
system tracks, e.g. the spaces the content identified by a given CID has been uploaded to and the dates the uploads happened.

#### inputs

`root: CID`

#### returns

The `spaces` property will be a list of spaces the given root CID's content has been uploaded to, along
with the date it was uploaded.

```typescript
{
  spaces: Array<{space: SpaceDID, insertedAt: Date}>
}
```

#### Implementations

- @web3-storage/capabilities: [capability in admin.js](https://github.com/web3-storage/w3up/blob/main/packages/capabilities/src/admin.js)
- @web3-storage/upload-api: [invocation handler in admin/upload/inspect.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/upload-api/src/admin/upload/inspect.js)

### `admin/store/inspect`

Get information about a shard (i.e., a CAR that contains part of an upload) CID. This
does not return the actual data identified by the CID, just metadata our system tracks,
e.g. the spaces the CAR identified by a given CID has been stored in and the date it was stored.

#### inputs

`link: CID`

#### returns

The `spaces` property will be a list of spaces the specified shard was stored in, along with the date on
which it was stored.

```typescript
{
  spaces: Array<{space: SpaceDID, insertedAt: Date}>
}
```

### `admin/block/inspect`

Get information about a block by CID. This does not return the actual data identified by the CID, just metadata our
system tracks. Specifically, when invoked this capability should return information about the CAR the block
has been packed into in our system.

### inputs

`link: CID`

### returns

The `carpath` property will be a string representing the location the block's parent CAR is stored in our system.

```typescript
{
  carpath: string,
  length: number,
  offset: number
}
```

#### Implementations

- @web3-storage/capabilities: [capability in admin.js](https://github.com/web3-storage/w3up/blob/main/packages/capabilities/src/admin.js)
- @web3-storage/upload-api: [invocation handler in admin/store/inspect.js](https://github.com/web3-storage/w3up/blob/231cf1f863f4e9a96c92d9ef5001617ba928028d/packages/upload-api/src/admin/store/inspect.js)

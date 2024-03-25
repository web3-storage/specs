# UCAN did:web did:dns delegation spec

![status:wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)
[![hackmd-github-sync-badge](https://hackmd.io/PmNipF1ERBKJKa7FyWBvUQ/badge)](https://hackmd.io/PmNipF1ERBKJKa7FyWBvUQ)


## Editors

- [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Authors

- [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

The [UCAN specification][ucan] describes token principals via [`did:key`][] method and allows other DID methods to be used. This specification defines extension for use of [`did:dns`][] and [`did:web`] methods.

## Motivation

While [`did:key`][] method is excellent for cryptographic verifiability, it does however suffer in contexts where key rotation is desired in way that would not invalidating all delegatinon chains it is part of. Following scenario describes these limitions:

Alice with DID `did:key:zAlice` wants to delegate some capability to a `ucan.store` service, so it can redelegate that capability to another agent `did:key:zAli`. If `ucan.store` service rotates a key it will no longer be able to issue redelagation.

<!--
 with DID `did:key:zPreRotation`. User Aice requests access to delegated capability on another device with DID `did:key:zAli` from the `ucan.store` _(after going through an out of bound verification)_. Unfortunatelly `ucan.store` had to rotate key (because old on was compromised) and now it has new DID `did:key:zPostRotation`. It is impossible for `ucan.store` to delegate capbility back to the user. Delegation can only be arranged if user can issue new delegation from `did:key:zAlice`, but user no longer has access to the that key delegation is no longer possible.

> There are other ways Alice could arrange delegation from `did:key:zAlice` to `did:key:zAli` with different tradeoffs. Evaluating them would be a good idea for a system designer, here we defining solution when no other option offers desired tradeoffs. -->

To address described limitation we propose use of [`did:dns`] or [`did:web`] principals so that delegation from `did:key:zAlice` no loner is tied to a specific key. This would get us step further, but still we run into a problem, after key rotation delegation to `did:key:zAli` is no longer valid as the key that signed it is no longer the one did document resolves to.

This specification describes solution to the second order problem by requiring that [`did:key`] that [`did:dns`] and [`did:web`] resolve to MUST assume an ambient authority over (pre-resolution) DID, which it MAY delegate to other principals through standard UCAN delegation.

> Expample below illustrates `did:dns:w3.storage` delegating own resource to `did:key:zService`, which in turn redelegates it to `did:key:zRotation`.
>
> This setup allows primary key (one that DID document resolves to) to be kept very safe e.g. on a piece of paper in safe deposit box, is it is only needed to delegate capabilities to `did:key:zService`. That key also is only used for rating keys and therefor can be stored securily e.g. in hardware key.

```ts
{
  aud: "did:key:zRotation",
  iss: "did:key:zService",
  exp: null,
  att: [],
  prf: [{
    iss: "did:dns:w3.storage",
    aud: "did:key:zService",
    exp: null
    att: []
  }]
}
```

Above delegations MAY be embedded inside a relevant UCAN tokens, so that key in rotation at the moment of delegation MAY assume full authority over corresponding [`did:dns`][] or [`did:web`][] resource.

> Example below illustrates `did:key:zRotation` delegating `did:key:zAli` capabilities derived from `did:dns:w3.storage` through `did:key:zService`. It embeds adove described delegation chain inside fact to provide a verifiable evidence that it can redelegate capbilities on `did:dns:w3.storage` bahalf

```ts
{
  iss: "did:key:zRotation",
  aud: "did:key:zAli",
  exp: null,
  att: [
    {
      with: "did:key:zAlice",
      can: "*"
    }
  ],
  prf: [
    // Proof that did:key:zRotation has authority over
    // did:dns:w3.storage delegated to it by did:key:zService
    {
      iss: "did:key:zService",
      aud: "did:key:zRotation",
      exp: null,
      att: [],
      // Proof that did:key:zService has been delegated
      // authority over did:dns:w3.storage as long as
      // did:key it resolves to is still the one that signed
      prf: [{
        iss: "did:dns:w3.storage",
        aud: "did:key:zService",
        exp: null
        att: []
      }]
    }
  ],
  fct: [
    // Evidence that "did:dns:w3.storage" has been delegated
    // full authority over did:key:zAlice by the owner.
    {
      iss: "did:key:zAlice",
      aud: "did:dns:w3.storage",
      exp: null,
      att: [
        {
          with: "did:key:zAlice",
          can: "*"
        }
      ]
    },
  ]
}
```

## Delegation of complete authority

[UCAN][] specification does not describe an ability to delegate authority over the resources delegating prinipcal MAY hold in the future. This makes it impossible for `did:dns:w3.storage` key to delegate capability that `did:key:zAlice` will delegate to it in the future.

To overcome this limitation here we propose delegation with `att: []` and `exp: null` to be treated as delegation of complete authority:

```ts
{
  iss: "did:dns:w3.storage",
  aud: "did:key:zService",
  exp: null
  att: []
}
```

It can also be interpreted as `did:dns:w3.storage` stating to be "also known as"
`did:key:zService` allowing it to delegate whatever `did:dns:w3.storage` CAN.

## Pipelining dellegations

[UCAN][] specification does not offer a way for two delegation chains to be pipelined into one. In our described scenario we have two delegation chains:

1. `did:key:zAlice -> did:dns:w3.storage`
2. `did:dns:w3.storage -> did:key:zService -> did:key:zRotation`

From which we would like to construct a delegation of the capability delegated in (1) issued by the the (rightmost) audience in (2).

```
did:key:zAlice -> did:dns:w3.storage -> did:key:zService -> did:key:zRotation -> did:key:zAli
```

To accomplish this we propose that:

- To issue delegation from `did:key:zRotation` with a valid proof chain (2) proving that `did:key:zRotation` has been delegated all capabilities from `did:dns:w3.storage`.
- Embed proof chain (1) in `fct` as a verifable evidence that `did:dns:w3.storage` has been delegated capabilities for `did:key:zAlice`.

```ts
{
  iss: "did:key:zRotation",
  aud: "did:key:zAli",
  exp: null,
  att: [
    {
      with: "did:key:zAlice",
      can: "*"
    }
  ],
  prf: [
    // Proof that did:key:zRotation has authority over
    // did:dns:w3.storage delegated to it by did:key:zService
    {
      iss: "did:key:zService",
      aud: "did:key:zRotation",
      exp: null,
      att: [],
      // Proof that did:key:zService has been delegated
      // authority over did:dns:w3.storage as long as
      // did:key it resolves to is still the one that signed
      prf: [{
        iss: "did:dns:w3.storage",
        aud: "did:key:zService",
        exp: null
        att: []
      }]
    }
  ],
  fct: [
    // Evidence that "did:dns:w3.storage" has been delegated
    // full authority over did:key:zAlice by the owner.
    {
      iss: "did:key:zAlice",
      aud: "did:dns:w3.storage",
      exp: null,
      att: [
        {
          with: "did:key:zAlice",
          can: "*"
        }
      ]
    },
  ]
}
```

## Extended principals

Above extensions could be applied to [`did:key`][] principals allowing desired delegation to be arranged just the same. Still we feel that use of [`did:dns`][] and / or [`did:web`][] identifiers with principals that regularily rotate keys, offers additional benefits:

1. It offers convinience of recognizable identifiers.
2. It allows rotating top level key without coordinating a change with all the deployd software.

Please note that top key rotation would inevitably invalidate all of the delegations given [did:dns][] / [did:web][] identifiers were part of, however that is desired behaviour as such rotation is only expected in extreme scenario likely due to system compromise.

[`did:key`]: https://w3c-ccg.github.io/did-method-key/
[`did:dns`]: https://danubetech.github.io/did-method-dns/
[`did:web`]: https://w3c-ccg.github.io/did-method-web/
[dkim]: https://www.rfc-editor.org/rfc/rfc6376.html
[rfc6376]: https://www.rfc-editor.org/rfc/rfc6376.html
[ucan]: https://github.com/ucan-wg/spec
[ucan revocation]: https://github.com/ucan-wg/spec#66-revocation
[capability invocation]: https://www.w3.org/TR/did-core/#capability-invocation

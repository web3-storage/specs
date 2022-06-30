# UCAN keypair management


### Problem statement

Building services that use [UCAN][]s for authorization require that each service manage own keypair so it can interact with other service(s). Running many microservices implies having _many keypairs that need to be kept safe and secret_.

Additionally cross service interaction e.g. service `did:key:zUpload` invoking `access/resolve` capability on service `did:key:zAccess` implies that:

1. `did:key:zAccess` needs to issue UCAN that delagets "access/resolve" to `did:key:zUpload`.
2. `did:key:zUpload` need to keep delegated UCAN around in order to invoke `access/resolve`.
3. When `did:key:zAccess` rotates keys it's DID change needs to propagate through dependencies and all the capabilities need to be re-issued.


It is easy to imagine this becoming a logistical nightmare with many interdependent services.

:::info
In summary we want to keep service keypair very secure so we don't have to deal with logistics of updating DIDs and UCANs across all the services that interact with it.
:::

### Thread model

> It is important to do some [thread modeling](https://en.wikipedia.org/wiki/Threat_model) in deciding what solution would provide effective safeguard against these threats.

#### Threat: Service key compromise

Compromise service keypair would enable attacker to invoke capabilities on other services that were delegated to it before compromise is discovered.

Our services heavily lean into content addressing and write only semantics. If all interations are also UCAN authenticated all the harm caused by compromised key could actually be undone on discovery.

For that reason forced key rotation is arguably the most painful fallout of such an attack as it would require updating DIDs, issued UCANs across services and potentially in our user applications.

#### Mitigation: Key custody


We could mitigate outlined thread by using a key custody service like [AWS Key Management Service](https://docs.aws.amazon.com/kms/latest/cryptographic-details/key-hierarchy.html) or [HashiCorp Vault](https://www.vaultproject.io/api-docs/secret/transit#hash-data).

It is however important to consider tradeoffs:

1. Signing invocations would require uploading UCAN invocations into key custody introducing extra latency.
2. Troubles in key custody service would have direct effect on our service operations.
3. All of our cross service interactions become observale by a key custody service (even when encrypted if they manage our keys).

#### Mitigation: Secure supreme key

Instead of services delegating capabilties to each other we could model our system differently in which we have a "supreme authority" that owns all the capabilities and delegates subset to a specific services that execute them.

If designed this way service key rotation would barely affect others since UCANs would have been issued by "supreme authority" and not the service providing it.

Only time "supreme authority" private key would be required is when we need to authorize new service or revoke capabilities of the compromised one. Given the low frequency of use such "supreme key" could live in secure hardware from which it can't be extracted. Furtherome we could adopt hardware key rotation.

:::success
We could also consdier [BLS](https://en.wikipedia.org/wiki/BLS_digital_signature) and/or [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing) schemes to account for hardware failures and resilience in cas of partial compromise.
:::



[UCAN]:https://github.com/ucan-wg/spec/
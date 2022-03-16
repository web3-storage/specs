User "bucket" identifier selection for .storage products
========================================================

Background
----------

In a UCAN, both the iss and aud fields contain a “DID”, which is essentially the public key of a public/private key-pair. This means that each user who uses UCANs with nft.storage or web3.storage must have a public/private key pair that they use.

Separately to that, the plan for `.storage` products is to give each user their own “bucket” (similar in concept to a bucket in S3). Each user will then be able to delegate permissions to that bucket and its subfolders to other users/clients via UCANs.

This raises the question of what identifier should be used for each user’s “bucket”. That is the topic of this document.


Possible Options
----------------

### Option 1 - User ID from database

One option is to use the primary key of the user from the database.

This works well because it’s
1. Short
2. Unique (at least for each service)
3. Permanent

The one disadvantage is:
* It results in having service specific data in the UCAN, which means it’s not unique across both web3.storage and nft.storage and it means it’s not reusable across different services	


### Option 2 - User’s public key

Another option is to use the user’s public key (or the first public key which they use with the service).

This works well because it’s:
1. Unique globally (across both services)

Although it’s only unique across both services if the user uses the same key-pair for both services.

A minor downside of tying the “bucket” ID to the user’s key-pair is that we’re creating a possibly confusing situation where a cryptographic identifier is being used for something which is really just a file path. Conceptually this might just be slightly odd to the user.

There are two possible ways we could implement this:

#### Option 2.1 - not stored in DB

Use the user’s public key as the bucket name, but don’t store it in the DB.

This is pretty much un-viable because:
* If we don’t store it in the DB then we can’t make it unique to the user. So each time a user created a UCAN they could do so with a different public key, giving them (yet another) new bucket.
* We have no way of knowing which bucket belongs to which user; we only know which bucket belongs to which key. This creates a situation with several distinct downsides:
    - a. If the user loses their private key, they lose access to their bucket; we can’t implement any form of recovery.
    - b. If the user’s private key is leaked, then they have no way of revoking it, so access to their bucket is permanently breached.
    - c. The user could potentially create “backup” keys by creating UCANs with unlimited expiry times which delegate full privileges to another key. This helps mitigate problem (a), but makes problem (b) more likely.
    - d. With no possibility of revocation or recovery, it conceptually breaks the idea that, like passwords, key-pairs can be rotated.

#### Option 2.2 - stored in DB

In this implementation (which is what’s currently implemented), when a user uploads the root DID the first time, we store the relationship between the user adding the root UCAN and the the DID.

This solves all the problems of option 2.1, or at least leaves open the possibility of solving them by adding revocation/recovery functionality. (But still shares the overall downside of option 2 mentioned at the top.)


### Option 3 - A UUID

Another option is to generate a UUID for each user to use as their bucket name.

The advantages of this are that it’s:
1. Short
2. Globally unique across services
3. Removes the link between key-pairs and the bucket name, allowing key-pairs to be rotated.

This UUID could be synced between web3.storage and nft.storage to allow each user to have a single bucket identifier which is global. But unlike with option 2, the user doesn’t sync this value for us.


Discussion
----------

It seems that the one advantage of using the DID as the bucket name is essentially this: it gets the user to do the work of setting a common value between nft.storage and web3.storage for us. As far as I (Adam) can see (but I might be wrong!) there is no other advantage to this over the UUID solution, it just gives us that syncing for free. The trade-off is the blurring of parameters and potential confusion that it may cause.

Given that we already have the https://api.nft.storage/user/did endpoint to allow the user to specify their DID. Would it make sense to allow them to specify their bucket name in a similar way? This would allow a conceptual separation between buckets and authentication keys while still allowing them to have the same bucket name across all storage services.


### Revocation / Recovery

If a user loses their private key, then we want to provide a way for them to still control their bucket. This could be done in a couple of ways:
1. Allow them to “rename” (i.e. move) their bucket.
    - If so, we need to handle the mapping between old and new from a path perspective.
2. Allow them to change their authorised keys.
    - Control a list of keys in their account which are allowed to sign UCANs?
    - Revoke UCANs which have already been issued? Would this be done automatically if the keys which signed the UCANs have been revoked?

Revocation/recovery probably needs some more thought.


Scenarios to consider
---------------------

### Recovery scenario

This scenario applies if we are using the user’s public key/DID as their bucket name and we’re implementing an ability for a user to recover/reallocate their bucket if they lose their private key.

1. User A uploads their public key; we don’t validate that they own it when registering (POST user/did).
2. User B uploads User A’s public key as their own; again, we don’t validate that we own it when registering.
3. User B then tells us that they’ve lost their private key, so they ask for their bucket to be mapped to a new public key and they provide us with a new public key.
4. We transfer ownership of the bucket belonging to User B’s public key to the new key that they provided. We’ve now accidentally transferred control of User A’s bucket to User B.

Possible solutions:
1. Make the `did` column on the `user` table unique (which it already is).
    - This prevents step 2 in the scenario. Does this also solve the problem across the different storage services as well? **What if I take the public key that someone else has used on nft.storage and register it as mine on web3.storage? The key is public, so this is perfectly possible.**
2. When we first store a user’s public key (DID) against their account, we validate that they actually own it.
    - This requires a bit more work but might be a more robust solution.


### Scenario - Multiple buckets
#### Problem (TL;DR)
With the implementation described in this document we're tying  `UserA` to a `didX`, and currently we allow for 1 root bucket.
We're strongly coupling `UserA` to `didX`(bucket).


Conceptually, `userA` (through a did, keypair) is at the moment accountable for "ticket printing" via UCANs. 
Which I suspect might be a problem when we scale to a more complex architecure: multiple buckets, payments, etc.


## Scenario description

In this scenario we want to have multiple projects, ie. `myApp1`, `myApp2` projects. With the current architecture we could achive this by having `UserA` create a `didX/myApp1` and build a UCAN like `{with: 'storage://didX/myApp1', can: "*" }`. While this works it still keeps a strong relationship between those buckets and the original user. 

If we think about it, assume `userA` has created a UCAN than grants `can: "*"` to `userB` for their entire bucket. Pratically, `userB` has the same level of auth as the original user, but from our DB perspective we're treating them "differently". There's no relationship between that userB and that bucket. At the moment the DID is column on the `User` table, so we are not linking to it.


I wonder if we should be thinking about Buckets as their own entities, that are accountable for signing UCANs (printing the tickets), and that ability can be delegated to multiple `did`s... But there's no hard 1:1 link between a single User and a single bucket. This would then leave open the possibility for things such as:
* One user having more than one bucket.
* A user being able to delete a bucket and start again through the web UI.
* Multiple users sharing _equal_ access to a bucket.
* Having "organisations" where multiple users share _equal_ top-level access to a bucket.

While these things (or similar setups) are theoretically possible by getting users to manage things through UCANs, we should probably ask whether that is the most convenient thing for the users.


I still struggle to envision how much duplication between ucan and service data structure is required to have this model working in practice. I suspect that the only way to undertand that is to start building an mvp


#### Questions
1. Who creates the keypair for the bucket in the first place? The user generating the bucket in the first place? The service?
2. I feel a relationship between the bucket creator should exist, but I reckon it should be an external key to user table.
3. While UCAN would work for auth, I suspect the service (ie to hadle the UI), should still store information about delegation etc? Or am I missing something?
4. Ultimately, do we want to build the thing which is the simplest to build (current architecture), or is there an architecture that would create a better experience for the users, and should we build that?

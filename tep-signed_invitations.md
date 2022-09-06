|TEP: ????|Unilateral Self-Generated Client Tokens|
|-:|:---|
|__Layer:__|Index|
|__Authors:__|Cameron Garnham|
|__Status:__|Draft|
|__Discussion:__||
|__Type:__|Standards Track|
|__Created:__|2022-08-31|
|__Updated:__||
|__Resolution:__||

# Introduction

## Abstract
This document proposes a format for unilateral self-generated client tokens.

## Copyright
This document is licensed under the 2-clause BSD licence.

## Motivation
Traditionally, it is the server that generates tokens and send them to the user:
1. The server chooses unique nonce.
2. The server hashes this nonce with a pepper.
3. Indexed by the nonce, the server stores the hash, its expiry, its privileges, and restrictions, in a database.
4. The nonce and pepper are concatenated and encoded into the 'token'.
5. The server gives the token and its explanation (expiry, and privileges, restrictions) to the user.

The user may transfer this token out-of-band to another.

5. Upon presenting the token to the server looks up the nonce, and computes the hash with the given pepper.
6. If matching, and not expired, and the client meets the restrictions, the privileges are bestowed onto the request.

This protocol is simple and reliable and doesn't store any private data on the server.

### Unilateral Self-Generated Client Tokens
Clients generates their own tokens according to a template and an insignia (a public key) registered with the server. This has a number of advantages over traditional server-generated tokens:

- Server does not need to include self-generated tokens in their token database.

_The server still needs to make sure that single-use self-generated tokens are not re-used. Typically, by maintaining a hash-map: mapping the hash of the token and the expiry of the token; this set can be periodically pruned for tokens that have expired, as they are no-longer a threat to being replayed._

- Clients can specialise tokens to the requirements of every request.
- Clients can just-in-time generate one-time-use tokens.
- Clients can generate and send tokens while offline or otherwise disconnected from the server.
- Clients can allocate a role to a second party without needing to also distribute a secret token.

_The client can register an insignia of the second party with the server, no secret data needs to be shared. Either from the server to the second party, or from the client to the second party._

There are some disadvantages:
- They are slower. Use of asymmetric cryptography is far slower than a simple hash.
- They are more complex.

_The asymmetric cryptography requires creating a signature on the clients side, and verifying a signature on the server side, this is very slow compared to a simple cryptographic hash. The primary mitigation is for the client to use multi-use-tokens for requests that are very common._

# Design
We first describe the database structure, then we define a token template, then we define the signed-tokens, and finally we describe the token verification process.

## Overview
Unilateral self-generated client tokens, are tokens that are made accoring to a tempalte, and signed by the client. The token-template and the insignia, the public key for making these tokens are stored in the server database.

There are two general forms of these tokens, single use, and multi-use.

## Server Storage
Usint the same proticl 


1. The 

A username is the unique key in the database retaining to the actions of a user. Public keys are associated with users, for a particular purpose. Every public-key-purpose combination has a set of timed status.

```
[user]
(PK. String) username

[purposed_public_key]
(PK. public_key || purpose)
(Enum.) purpose
(Binary.) 32-byte-public_key

[status_timed]
(PK. Unique Hashed)
(Enum.) status
(Date-Time. Optional.) beginning
(Date-Time. Optional.) end

[user] many-many [purposed_public_key]

[user][purposed_public_key] many [status_timed]
```

_"Users and Purposed-Public-Keys have a many-many association, each association may associate many timed statuses.":_
- Each _user_ may look up what _purposed public keys_ it is associated with.</br>
Likewise, Each _purposed public key_ may look up what _users_ it is associated with.
- Each _(user <> purposed public key)_ link can look up it's associated _timed statuses_.

### Purposes
In this document we only user the `A` for "Authentication".</br>In the future we may propose additional purposes.
```
Purposes (enum):

A: Authentication
```

### Status Validity
If revocation does not specify a beginning, it is for all time periods.</br>
`Active` has the lowest priority: The key must be `created`, and not `expired`, and not `revoked`.
```
Status (enum):

A: Active
C: Created
E: Expired
R: Revoked
```
#### Status Validity Table:
|status|beginning|end|
|:-|:-|:-|
|active|must|must|
|created|must|none|
|expired|must|none|
|revoked|optional|none|

# Specification
The server maintains a list of public keys that may be used for authentication for each user. A user may propose a new public key be added to their list of authentication keys, or may revoke or expire an existing key.

* To add, or expire, or increase the active period a public key, the user must first authenticate with this new key to prove ownership.
* To revoke, the user may authenticate with the key they wish to revoke.</br>
Or some other form of authentication should be used in the case that the user no-longer has access to their secret key, such as verification by a moderator.

Users who do not have any public key associated, for example new users, or when transitioning from password based authentication should first authenticate with their public key, then preform the association with the user account.

## Mnemonic Codes
Clients generate a mnemonic code according to: [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki).

The seed words are prefixed with an additional `"torrust"` word, and displayed to the user. The 'torrust' seed word is not included in the checksum calculation, however must be present when the user enters their seed in recovery.

Clients must provide an option for the user to give a seed-password in generation and recovery.

### Storage of Mnemonic Code
The client should delete from memory the recovery seed once the user has confirmed that they have backed up the seed, and their key-pair(s) have been generated.

This gives the user some forward security. In the future the user may wish to use the same seed to rotate their keys (for example, when logging in on a new computer). Because the seed is not saved in the client, if the users' client is compromised at a later date, the seed itself remains secret.

## Public and Private Key Derivation
Deriving the keys according to: [BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

We use the following hardened derivation path:</br>The index is increased in the case the user has rotated their keys, for the moment we only use zero:
```
m / purpose' / version' / torrust-purpose' / index'

m / "torrust" / 0 / "authentication" / 0
```
The resulting node is a private-public secp256k1 key-pair. (the same elliptical-curve as is used in bitcoin).

___Note:__ as an extension, the future client may choose to derive and register a series of public keys with the server, i.e. indexes: 0, 1, 2, 3, etc, but delete all except the current key from memory.</br>
This will allow for easier key-rotation in the future, as additional keys cannot be predicted because we use hardened derivation._

### Storage of Derived Keys
In general, it is _not recommended_ that a general user would interact with any derived key material, such as the encoded login credentials. It is recommended that clients store this private data in their (long-term) _localStorage_.

- The client should provide an option to not use any long-term storage, but instead only use their (temporary) _sessionStorage_.
- The client should provide an option to view, backup and manually manage their derived keys.

There are three main reasons why we wish to provide these possibilities to the user:
1. _Public or Shared Computer Login:_ At internet café's or other places with shared computers.
2. _Multi-User-Account:_ Some users may wish to switch accounts on the same computer.
3. _Password-Manager Support:_ Some users prefer to use a password-manager.


## Login Credential Encoding
For encoding the secp256k1, we use [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) encoding, 32-bytes for public keys.</br>
For displaying the login credentials, we use [BIP-350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki), 'Bech32m' encoding.

We use the follwoing "Human Readable Parts", or "hrp":

|hrp|description|
|:-|:-|
|"torrust"|for public keys|
|"torrust_secret"|for secret keys|

Upon generation the public and secret keys are stored in the application (either the temporary _sessionStorage_ or the long-term _localStorage_) and may optionally be displayed to the user.

_If displayed to the user, the user should be encouraged to save them in their password manager._

```
Login ID:   bech32m("torrust",32-byte-public-key)
Login Code: bech32m("torauth",32-byte-private-key)
```

While technically only the private key is needed, we require both as the public key acts as checksum for the private key. This makes it more comfortable for users that are used to traditional password-username based authentication.

## Authentication
The server and client preform Elliptical Curve Diffie-Hellman over the secp256k1 curve to authenticate:

_We use the `secp256k1_ecdh` as defined in [secp256k1_ecdh.h](https://github.com/bitcoin-core/secp256k1/blob/master/include/secp256k1_ecdh.h), see example: [examples/ecdh.c](https://github.com/bitcoin-core/secp256k1/blob/master/examples/ecdh.c)._

___Note:__ This scheme assumes that there a secure client connection to the server, for example, by using TLS._

### Prerequisites
The server and client generate temporary ephemeral key pairs for every connection attempt. They should not be reused.
```
Ephemeral Keys:
(E_server, e_server)
(E_client, e_client)

Client User Key:
(U_client, u_server)
```

### Setup
The server transfers their ephemeral public key to the client.</br>
The client transfers their ephemeral public key and user public key to the server.
```
Server tx to Client:
(E_server)

Client tx to Server:
(E_client, U_client)
```

### Key Derive
The server and client both derive common secret keys.
```
Server:
ECDH(E_client, e_server) -> c_ephemeral
ECDH(U_client, e_server) -> c_user

Client:
ECDH(E_server, e_client) -> c_ephemeral
ECDH(E_server, u_client) -> c_user

Note: The resulting keys should be identical.
```

### Client Auth Token
The client includes their URI to help guard against man-in-the-middle attacks.
```
Shared RFC 3986 URI: format: "proto://[user@]host[:port][/path]"

HASH(uri_client || c_ephemeral || c_user) -> token_auth
```

### Authentication
The URI should match the expected domain of the server.
```
Client tx to Server:
(token_auth, uri_client)

Server Verify:
uri_client
token_auth

If matching, authorise user according to their user public key. (U_client).
```

_The server now can issue a session token to the client accordingly._

## Security Analysis

This scheme is very simple, it would be very good there was some formal security analysis.

The basic protocol follows:

1. Both derive a common ephemeral (temporary) key:</br>
`client_ephemeral (DH) server_ephemeral -> ephemeral_key`

_The ephemeral key is used as a nonce to protect against replay attacks._

2. Both derive a common authentication key:</br>
`client_public (DH) server_ephemeral -> authentication_key`

_Client knowledge of the authentication key implicitly shows ownership of the client public key._

3. Client derives the common authentication token:</br>
`server_uri + ephemeral_key + authentication_key -> authentication_token`

_Mixing in the server's uri helps protect the user from a bad-server impersonating another server._

4. Client sends URI and authentication token to server:</br>
`client: server_uri, authentication_token => server`

_The server does not learn of the ephemeral key and authentication key from the user. The server only learns the server uri and authentication token from the client._

_If the server was impersonating another server, the uri would not match._

5. Server checks `server_uri` matches expected value.

_This protects the client from the wrong server attack: if the user connects to `a.tld`, the uri will not match for `b.tdl`._

_Man in the middle attackers are unable to use an alternative uri._

6. Server also derives the common authentication token:</br>
`server_uri + ephemeral_key + authentication_key -> authentication_token`
 `authentication_token`

_This token includes the uri, nonce, and client public key._

7. Server checks the received `authentication_token` matches self-derived value.

_If the keys match, the client is authenticated for their client public key._

# Compatibility

The updated client will support both login in with the new authentication scheme, and transferring existing users with password authentication to the new scheme.

# Reference Implementations

## Similar Implementations
This scheme is so simple that I guess there should be!

However, the common ones that I have found are far-more complex than what we use here, (such as getting a CA involved with the authentication).
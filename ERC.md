# ERC-XXXX A Schnorr Signature Scheme for Ethereum

This document specifies an EVM-efficient Schnorr signature scheme over the secp256k1 elliptic curve in combination with the keccak256 hash function.


## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Conventions and Definitions](#conventions-and-definitions)
- [Signature Creation](#signature-creation)
- [Signature Verification](#signature-verification)
- [Signature Encoding](#signature-encoding)
- [Security Considerations](#security-considerations)
- [Ethereum Schnorr Signed Message Digest](#ethereum-schnorr-signed-message-digest)
- [Nonce Generation](#nonce-generation)
- [Notes on `ecrecover` usage](#notes-on-ecrecover-usage)
- [Reference Implementation](#reference-implementation)


## Abstract

Ethereum's EOA addresses are based on ECDSA signatures. However, on the application level it is totally fine to use different cryptographic systems, such as different elliptic curves or signature schemes.

Examples:
- secp256r1 precompile
- BLS12-381 precompiles and BLS signature algorithm

However, each have issues:
- ECDSA -> many... (nonce creation, malleability, non-provable)
- BLS -> expensive, pairing assumption


## Motivation

Schnorr signatures provide a number of advantages compared to ECDSA signatures:

- **Provable secure**: Schnorr signatures are provable secure
- **Non-malleability**: Schnorr signatures are non-malleable
- **Linearity**: Schnorr signatures have multi-signature support, ie they provide a mechanism for collaborating parties to produce a signature that is valid over the sum of their public keys

Also they are heavily used by Bitcoin, see [BIP-340](https://github.com/bitcoin/bips/blob/ad1d3bc2a7b0d84247c29f847e85c35283094e2f/bip-0340.mediawiki).


## Conventions and Definitions

**Types**:

- `PublicKey(uint,uint)` - A secp256k1 public key (x, y), ie a point on the secp256k1 curve in Affine coordinations
- `SecretKey(uint)` - A secp256k1 secret key, ie a secp256k1 field element

**Functions**:

- `H() :: bytes -> bytes32` - Keccak256 Function
- `()ₓ :: PublicKey -> uint` - Function returning the x coordinate of given public key
- `()ₚ :: PublicKey -> uint ∊ {0, 1}` - Function returning the parity of the y coordinate of given public key
- `()ₑ :: PublicKey -> address` - Function returning the Ethereum address of given public key

**Operators**:

- `‖` - Concatenation operator defined as `abi.encodePacked()`

**Constants**:

- `G :: PublicKey` - Generator of secp256k1
- `Q :: uint` - Order of secp256k1

**Variables**:

- `sk :: SecretKey` - The signer's secret key
- `Pk :: PublicKey` - The signer's public key, ie `[sk]G`
- `message :: bytes` - The message to be signed as bytes blob


## Prefix Tag

Schnorr signatures are tagged as _Ethereum Schnorr Signed Messages_:
`keccak256("\x19Ethereum Schnorr Signed Message:\n32", m)`


## Signature Creation

1. Derive the `message`'s digest `m` as specified in [Ethereum Schnorr Signed Message Digest](#ethereum-schnorr-signed-message-digest).

2. Select a cryptographically secure, uniformly random nonce `k` as specified in [Nonce Generation](#notes-on-nonce-derivation).

3. Compute the nonce's public key `R = [k]G`.

4. Construct challenge `e = H(Pkₓ ‖ Pkₚ ‖ m ‖ Rₑ) % Q`.

> NOTE
>
> Modulo bias is ok, see BIP-340.
> Note that the probability of `keccak256(sk ‖ m) ∊ {0, Q}` is negligible.

5. Compute `s = k + (e * sk) (mod Q)`.

6. Let tuple `(s, R)` be the Schnorr signature


## Signature Verification

* **Input**: `(Pk, m, s, R)`
* **Output**: `True` if signature verification succeeds, `False` otherwise

1. Construct challenge `e`

```
e = H(Pkₓ ‖ Pkₚ ‖ m ‖ r)
```

2. Compute Ethereum address of nonce's public key

```
  ([s]G - [e]Pk)ₑ                | s = k + (e * sk)
= ([k + (e * sk)]G - [e]Pk)ₑ     | Pk = [sk]G
= ([k + (e * sk)]G - [e * sk]G)ₑ | Distributive Law
= ([k + (e * sk) - (e * sk)]G)ₑ  | (e * sk) - (e * sk) = 0
= ([k]G)ₑ                        | R = [k]G
= Rₑ
```

3. Return `True` if `([signature]G - [e]P)ₑ == Rₑ`, `False` otherwise


## Signature Encoding

The Schnorr signatures are encoded as `abi.encodePacked(s, R.x, R.y)` leading to `32 + 20 = 52` bytes length.

This is not a word boundary. Should have prefix byte?

> NOTE
>
> The verification process needs the signer's (may be aggregated) public key. It could come from storage, calldata... whatever.
>
> The encoding SHOULD be via SEC's uncompressed (65 bytes) or compressed (33 bytes) standard.

For both encoding `crysol` has an implementation. TODO: Make benchmarks.


## Security Considerations

Note that `crysol`'s Schnorr scheme deviates slightly from the classical Schnorr signature scheme.

Instead of using the secp256k1 point `R = [k]G` directly, this scheme uses the Ethereum address of the point `R` which decreases the difficulty of brute-forcing the signature
from 256 bits (trying random secp256k1 points) to 160 bits (trying random Ethereum addresses).

However, the difficulty of cracking a secp256k1 public key using the baby-step giant-step algorithm is `O(√Q)`[^baby-step-giant-step-wikipedia]. Note that `√Q ~ 3.4e38 < 128 bit`.

Therefore, this signing scheme does not weaken the overall security.


## Ethereum Schnorr Signed Message Digest

Need domain separator to prevent ...


## Nonce Generation

> WARNING
>
> A secret key may sign the same message via two different schemes, eg Schnorr and ECDSA. If both nonce generations use same algorithm the secret key may leak due to nonce reuse!
>
> Therefore always use the domain separator

BIP-340: "For example, if the rand value was computed as per RFC-6979 and the same secret key is used in deterministic ECDSA with RFC-6979, the signatures can leak the secret key through nonce reuse."

TODO: Do we also need curve inside of domain hash? What if Schnorr used on r1?
```
EIP_XXX_DOMAIN_HASH_SECP256K1;
EIP_XXX_DOMAIN_HASH_SECP256R1;
```

Why nonce deterministic still? Because implementation would use deterministic anyway, leading to same issues in real life!

Note that deterministic nonce derivation is defined as part of the scheme (?) This is to prevent randomness issues.

> WARNING
>
> A nonce may be biased from modulo operation if only from a 256 bit hash.
> For k1 its ok (see BIP-340) and (I guess :D) for r1 too.

Ethereum does not provide easy access to >256 bit hashes. Could rehash?


## Notes on `ecrecover` Usage

This implementation uses the ecrecover precompile to perform the necessary elliptic curve multiplication in secp256k1 during the verification process.

The ecrecover precompile can roughly be implemented in python via[^vitalik-ethresearch-post]:
```python
def ecdsa_raw_recover(msghash, vrs):
   v, r, s = vrs
   y = # (get y coordinate for EC point with x=r, with same parity as v)
   Gz = jacobian_multiply((Gx, Gy, 1), (Q - hash_to_int(msghash)) % Q)
   XY = jacobian_multiply((r, y, 1), s)
   Qr = jacobian_add(Gz, XY)
   N = jacobian_multiply(Qr, inv(r, Q))
   return from_jacobian(N)
```

A single ecrecover call can compute `([signature]G - [e]Pk)ₑ = ([k]G)ₑ = Rₑ = commitment` via the following inputs:
```
msghash = -signature * Pkₓ
v       = Pkₚ + 27
r       = Pkₓ
s       = Q - (e * Pkₓ)
```

Note that ecrecover returns the Ethereum address of `R` and not `R` itself.

The ecrecover call then digests to:
```
Gz = [Q - (-signature * Pkₓ)]G  | Double negation
   = [Q + (signature * Pkₓ)]G   | Addition with Q can be removed in (mod Q)
   = [signature * Pkₓ]G         | sig = k + (e * sk)
   = [(k + (e * sk)) * Pkₓ]G

XY = [Q - (e * Pkₓ)]Pk        | Pk = [sk]G
   = [(Q - (e * Pkₓ)) * sk]G

Qr = Gz + XY                                            | Gz = [(k + (e * sk)) * Pkₓ]G
   = [(k + (e * sk)) * Pkₓ]G + XY                       | XY = [(Q - (e * Pkₓ)) * sk]G
   = [(k + (e * sk)) * Pkₓ]G + [(Q - (e * Pkₓ)) * sk]G

N  = Qr * Pkₓ⁻¹                                                         | Qr = [(k + (e * sk)) * Pkₓ]G + [(Q - (e * Pkₓ)) * sk]G
   = [(k + (e * sk)) * Pkₓ]G + [(Q - (e * Pkₓ)) * sk]G * Pkₓ⁻¹          | Distributive law
   = [(k + (e * sk)) * Pkₓ * Pkₓ⁻¹]G + [(Q - (e * Pkₓ)) * sk * Pkₓ⁻¹]G  | Pkₓ * Pkₓ⁻¹ = 1
   = [(k + (e * sk))]G + [Q - e * sk]G                                  | signature = k + (e * sk)
   = [signature]G + [Q - e * x]G                                        | Q - (e * sk) = -(e * sk) in (mod Q)
   = [signature]G - [e * sk]G                                           | Pk = [sk]G
   = [signature]G - [e]Pk
```


## Reference Implementation

A reference implementation is provided in [verklegarden/crysol](https://github.com/verklegarden/crysol/pull/26)


<!--- References --->
[^baby-step-giant-step-wikipedia]:[Wikipedia: Baby-step giant-step Algorithm](https://en.wikipedia.org/wiki/Baby-step_giant-step)
[^vitalik-ethresearch-post]:[ethresear.ch: You can kinda abuse ecrecover to do ecmul in secp256k1 today](https://ethresear.ch/t/you-can-kinda-abuse-ecrecover-to-do-ecmul-in-secp256k1-today/2384)

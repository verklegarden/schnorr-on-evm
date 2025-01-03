# ERC-XXX - A Schnorr Signature Scheme for EVM Applications

## Abstract

This document specifies a Schnorr signature scheme over the elliptic curve secp256k1 for EVM applications.

## Motivation

EVM applications have traditionally used ECDSA signatures over secp256k1 in combination with the keccak256 hash function. However, Schnorr signatures have a number of advantages compared to ECDSA signatures:

* **Linearity**: Schnorr signatures are linear which allows them to be easily aggregated, i.e. it enables multiple collaborating parties to produce a signature that is valid over the sum of their public keys. This building block allows for higher level constructions such as multisignatures and threshold signatures.

* **Provable security**: Schnorr signatures are provable secure with weaker assumptions than the best known security proofs for ECDSA. More specifically, Schnorr signatures are strongly unforgeable under chosen message attack (_SUF-CMA_) in the random oracle model (_ROM_) assuming the hardness of the elliptic curve discrete logarithm problem (_ECDLP_)[^1].

* **Non-Malleability**: Schnorr signatures are non-malleable. Note that on the other hand ECDSA signatures are malleable which has lead to numerous security issues.

In contrast to BLS (multi-) signatures, Schnorr signatures are far more efficient to verify inside the EVM and do not require additional precompiles. Note that this is even true for already existing Schnorr multisignature and threshold signature schemes!

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC-2119 and RFC-8174 when, and only when, they appear in all capitals, as shown here.

Let `G` be secp256k1’s generator and `Q` secp256k1’s order. Let `sk` be an secp256k1 secret key and `Pk = [sk]G` be the secret key’s public key in affine coordinates. Let `Pkₑ` be a public key’s Ethereum address, `Pkₓ` a public key’s `x` coordinate, and `Pkₚ` a public key’s `y` coordinate’s parity with `0` if `y` is even and `1` if `y` is odd. Let `‖` be the concatenation operator performing a byte-wise concatenation.

## Hash Functions

We define the ASCII context string `ctx` containing our scheme and ciphersuite as `ETHEREUM-SCHNORR-SECP256K1-KECCAK256`. This enables domain separating our hash functions and ensures a signed message intended for one context is never deemed valid in a different context. It is RECOMMENDED that future signature schemes define a context string in the same pattern.

With the help of `ctx` we define our context aware, cryptographically secure, hash function `H` as `H(x) = keccak256(ctx ‖ x)`.

Using the hash function `H`, we define the following additional domain separated hash functions:

* `H₁(x) = H("message" ‖ x) (mod Q)`
* `H₂(x) = H("challenge" ‖ x) (mod Q)`
* `H₃(x) = H("nonce" ‖ x) (mod Q)`

Note that all hash functions derived from `H` are defined to return elements in `[1, Q)` via modular reduction with `Q`. While this generally introduces a bias leading to non-uniformly random output, secp256k’1 order `Q` is sufficiently close to $2^{256}$ that the modulo bias is acceptable[^2]. Note that the probability of any in this document defined hash function’s output being `0` is deemed negligible.

## Signature Creation

A Schnorr signature is generated over a byte string `message`, under secret key `sk` and public key `Pk = [sk]G` by the following steps:

1.  Derive the `message`'s domain separated hash digest `m` as specified in [_Message Hash Construction_](#message-hash-construction)
2.  Select a cryptographically secure, uniformly random nonce `k ∊ [1, Q)` as specified in [_Nonce Generation_](#nonce-generation) and compute its public key `R = [k]G`. Let `Rₑ` be the commitment
3.  Compute the challenge `e = H₂(Rₑ ‖ Pkₓ ‖ Pkₚ ‖ m) (mod Q)`
4.  Using secret key `sk`, compute the Fiat-Shamir response `s = k + (e * sk) (mod Q)`
5.  Define the signature over `m` to be `sig = (s, R)`

## Signature Verification

Validating the integrity of `m` using the public key `Pk` and the signature `sig` is performed as:

1. Parse `sig` as `(s, R)` if default encoded and `(s, Rₑ)` if compressed encoded
2. Verify `s ∊ [1, Q)`, otherwise output `0`
3. Compute challenge `e = H₂(Rₑ ‖ Pkₓ ‖ Pkₚ ‖ m) (mod Q)`
4. Compute `R'ₑ = ([s]G - [e]Pk)ₑ`
5. Output `1` if `Rₑ == R'ₑ` to indicate success, otherwise output `0`

Note that the verification is based on `R`'s Ethereum address and not on the public key itself. In order to perform the verification’s `mulmuladd` operation efficiently the `ecrecover` precompile can be abused for secp256k1. For more info, see [_Implementation Notes_](#implementation-notes).

## Message Hash Construction

1.  Compute message digest `d = keccak256(message)`
2.  Compute the message hash `m = H₁(d)`

In order to guarantee constant calldata size this Schnorr signature scheme does not accept arbitrary length messages. A context aware hash function is used to ensure the message hash cannot be reinterpreted in a different context.

## Nonce Generation

The nonce MUST be computed with a 32-byte randomness value `rand` and the secret key `sk` via `H₃(rand ‖ sk)`.

Note that by combining the randomness value `rand` with the secret key the nonce generation is hedged against a bad RNG possibly used to source `rand`.

Note that domain separating the nonce via the `H₃` function is necessary to prevent secret key leakage due to nonce reuse when the same `rand` value is sourced for different signature schemes. The same `rand` value may be sourced when using deterministic methods such as RFC-6979.

### Sourcing `rand`

The 32 byte `rand` value may be sourced via different methods, e.g. using a CSPRNG or computing it via RFC-6979. However, it MUST be guaranteed that `rand` is not reused for signatures signing different messages.

Furthermore, it is NOT RECOMMENDED to source `rand` deterministically. Note that this Schnorr scheme is compatible with multisignature and threshold signature schemes where a signer’s secret key does not match the public key the signature is created for. This property makes such schemes in general **insecure with deterministic nonce generation**. Additionally, creating nonces independently of signed messages may allow for a preprocessing stage in higher-level protocols where participants generate and publish nonce commitments prior to signing.

## Encoding

A Schnorr signature `sig = (s, R) = (s, (x, y))` is encoded in 96 bytes via:

```
[s 32 bytes][x 32 bytes][y 32 bytes]
```

### Compressed Encoding

A Schnorr signature can be compressed encoded to 52 bytes via compressing the public key `R` to its Ethereum address:

```
[s 32 bytes][Rₑ 20 bytes]
```

## Security Considerations

Note that this Schnorr scheme uses `R`'s Ethereum address instead of the public key itself, thereby decreasing the security of brute-forcing the signature from 256 bits (trying random secp256k1 points) to 160 bits (trying random Ethereum addresses). However, the difficulty of cracking a secp256k1 public key using the baby-step giant-step algorithm is `O(√Q)`[^3]. Note that `√Q ~= 3.4e38 < 128 bit`. Therefore, this scheme does not weaken the overall security.

### Signature Malleability

While Schnorr signatures as specified in this document are not malleable due to the definition of `s`, great care must be taken during verification to ensure a Schnorr signatures `s ∊ [1, Q)`. Note that this issue is mentioned explicitly due to Solidity's weak type system and the expectation of most implementations using type `uint256` for the `s` variable.

### Rogue Key Attacks

Rogue key attacks are a serious concern for Schnorr multisignature and threshold signature schemes where a subset of malicious participants use public keys computed as functions of public keys from honest participants allowing them to produce forgeries for the total set of public keys[^4].

Rogue key attacks can be prevented via a knowledge of secret key (KOSK) assumption, i.e. requiring participants to prove knowledge of the secret key during public key registration, or non-trivial public key aggregation schemes[^4][^5].

### Wagner's Birthday Attack

Another serious concern for multisignature and threshold signature schemes is Wagner's birthday attack, where malicious participants can manipulate a final aggregated nonce used for signing by submitting rogue nonces[^6].

Wagner's birthday attack can be prevents via either introducing an additional _nonce commitment_ communication round in the scheme or non-trivial nonce aggregation schemes[^5][^6].

## Rationale

Schnorr signature schemes exist in many different flavors. This Schnorr signature scheme chooses the signature to be `(s, R)` instead of `(e, R)` for closer behavior to Bitcoin’s BIP-340. Note that eventhough the signature is verified via `Rₑ`, it is still defined via `R` to ensure forward compatibility with Schnorr schemes based on aggregated public keys.

Additionally this Schnorr scheme is _key prefixed_ to protect against "related-key attacks" meaning the public key is prefixed to the message in the challenge hash input[^2]. Note that instead of prefixing the key in affine coordinates, the public key’s `x` coordinate and `y` coordinate’s parity are used to not increase the verifier's resource usage by either computing or storing the full `y` coordinate.

## Test Cases

> [!NOTE]
>
> Test cases still work-in-progress

## Reference Implementation

A reference implementation is provided in [verklegarden/crysol](https://github.com/verklegarden/crysol).

## Implementation Notes

### Elliptic Curve `mulmuladd`

In order to verify Schnorr signatures a `mulmuladd` operation must be performed over secp256k1. As Vitalik notes, the `ecrecover` precompile can be abused to perform such an operation, with the caveat that the result is not returned as elliptic curve point but rather as Ethereum address[^7].

The `ecrecover` precompile can roughly be implemented in python via:

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

Note that `ecrecover` also uses `s` as variable. From this point forward, let the Schnorr signature's `s` be `sig`.

A single `ecrecover` call can compute `([sig]G - [e]Pk)ₑ = ([k]G)ₑ = Rₑ` via the following inputs:

```
msghash = -sig * Pkₓ
v       = Pkₚ + 27
r       = Pkₓ
s       = Q - (e * Pkₓ)
```

Note that `ecrecover` returns the Ethereum address `Rₑ` and not `R` itself.

The `ecrecover` call then digests to:

```
Gz = [Q - (-sig * Pkₓ)]G      | Double negation
   = [Q + (sig * Pkₓ)]G       | Addition with Q can be removed in (mod Q)
   = [sig * Pkₓ]G             | sig = k + (e * sk)
   = [(k + (e * sk)) * Pkₓ]G

XY = [Q - (e * Pkₓ)]Pk        | Pk = [sk]G
   = [(Q - (e * Pkₓ)) * sk]G

Qr = Gz + XY                                            | Gz = [(k + (e * sk)) * Pkₓ]G
   = [(k + (e * sk)) * Pkₓ]G + XY                       | XY = [(Q - (e * Pkₓ)) * sk]G
   = [(k + (e * sk)) * Pkₓ]G + [(Q - (e * Pkₓ)) * sk]G

N  = Qr * Pkₓ⁻¹                                                         | Qr = [(k + (e * sk)) * Pkₓ]G + [(Q - (e * Pkₓ)) * sk]G
   = [(k + (e * sk)) * Pkₓ]G + [(Q - (e * Pkₓ)) * sk]G * Pkₓ⁻¹          | Distributive law
   = [(k + (e * sk)) * Pkₓ * Pkₓ⁻¹]G + [(Q - (e * Pkₓ)) * sk * Pkₓ⁻¹]G  | Pkₓ * Pkₓ⁻¹ = 1
   = [(k + (e * sk))]G + [Q - e * sk]G                                  | sig = k + (e * sk)
   = [sig]G + [Q - e * x]G                                              | Q - (e * sk) = -(e * sk) in (mod Q)
   = [sig]G - [e * sk]G                                                 | Pk = [sk]G
   = [sig]G - [e]Pk
```

<!-- References -->
[^1]:[Security Arguments for Digital Signatures and Blind Signatures](https://www.di.ens.fr/david.pointcheval/Documents/Papers/2000_joc.pdf)
[^2]:[BIP-0340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki#cite_note-13)
[^3]:[Baby-step giant-step Algorithm](https://en.wikipedia.org/wiki/Baby-step_giant-step)
[^4]:[Musig](https://eprint.iacr.org/2018/068.pdf)
[^5]:[Musig2](https://eprint.iacr.org/2020/1261.pdf)
[^6]:[On the Security of Two-Round Multi-Signatures](https://eprint.iacr.org/2018/417.pdf)
[^7]:[Vitalik "You can *kinda* abuse ECRECOVER to do ECMUL in secp256k1 today"](https://ethresear.ch/t/you-can-kinda-abuse-ecrecover-to-do-ecmul-in-secp256k1-today/2384)

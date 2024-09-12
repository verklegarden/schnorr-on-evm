# FROST - Flexible Round-Optimized Schnorr Threshold Signatures

See [RFC-9591](https://www.rfc-editor.org/rfc/rfc9591.html).

Would enable _extremely_ cheap threshold signature for Ethereum: one `ecrecover` call.

Note that [FROST does not provide robustness](https://youtu.be/FVW6Hgt_meg?feature=shared&t=1260) that may be required for decentralized applications.

Bitcoin BIP being worked on right now, see [here](https://github.com/siv2r/bip-frost-signing) and [this BIP specifically for the dkg](https://github.com/BlockstreamResearch/bip-frost-dkg).

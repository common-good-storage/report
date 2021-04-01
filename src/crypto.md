# On Crypto Compatibility

## Cryptography in Filecoin

The main [cryptographic primitives in Filecoin] are listed as:

- Merkle tree/DAG
- Vector commitment scheme
- zkSNARK
- Reliable broadcast channel (libp2p)

We do not consider broadcast here.
The remaining primitives are constructed using a combination of various underlying primities.
The main asymmetric primitives are:

- Elliptic curve \\( \mathbb{G} \\), implemented with [secp256k1], for efficiency and interoperability
- Pairing curve \\( ( \mathbb{G}_1, \mathbb{G}_2, \mathbb{G}_T )\\), implemented with [BLS12-381], for pairing-based primitives

The elliptic curve \\( \mathbb{G} \\) is used for efficiency, for instance to verify block messages.
It also serves enables wider integration with existing blockchains (e.g. Bitcoin) using the same curve.
The pairing curve enable more complex primitives using the pairing operator, for instance to construct the aggregate signatures, VRF and SNARKs.
In particular signatures in Filecoin can be either

- ECDSA over \\( \mathbb{G} \\) (secp256k1), or
- BLS signatures with public-key in \\( \mathbb{G}_1 \\) and signatures in \\( \mathbb{G}_2 \\).

Finally, Filecoin use a variety of symmetric primitives and maps to curves:

- Hash: BLAKE2b, SHA256, Poseidon, ChaCha
- Maps: Standard Hash-to-Curve for BLS12

[cryptographic primitives in Filecoin]: https://spec.filecoin.io/#section-algorithms.crypto
[secp256k1]: ???

## Compatibility and integration

In Polkadot, collators maintain a full node for the relay chain and the Parachain, and accept transactions from users.
The transactions are aggregated into Parachain blocks with an associated state transition proof (STF).
This STF is defined in WebAssembly and registered on-chain, and used by validators to verify the proposed ParaBlock.

The Parachain can be efficiently maintained by collators using native integration of the functions related to the Filecoin specification.
That is, Rust crates for verifying state transition in Filecoin can be directly used by collators.
For the validators, the primitives need to be compiled into WebAssembly.
Since the Filecoin state transition can be regarded pure/total, there is no inherent theoretical problems in compiling them to WebAssembly.

### State transition verification

The main primitives required for a Filecoin STF is integrating [Proof-of-Spacetime] via:

- Proof-of-Replication (PoRep):
When data is registered in Filecoin, it needs to be sealed.
First, the data is [encoded] using a slow SDR encoding using expander graphs, followed by [replication] for redundancy.
The data is then [hashed] using a Merkle-tree with the Poseidon hashing algorithm.
Finally a zk- is used to compress the zero-knowledge proof to a succinct one.

- Proof-of-Spacetime (PoSt):
Storage miners continuously prove they store the data with proofs-of-spacetime.
This is done by answering two different challenges: WinningPoSt and WindowPoSt.
The WinningPoSt check a miner has the data at a particular time, whereas the WindowPoSt check the data was stored continuously.
The WinningPoSt consist of the storage-power lottery and a winning miner providing a proof that he has a sealed copy of the data.
The WindowPoSt consist of a succinct zero-knowledge proof for each data sector every 30 minutes.

[Proof-of-Spacetime]: https://spec.filecoin.io/#section-glossary.proof-of-spacetime-post
[as planned]: https://wiki.polkadot.network/docs/en/learn-keys
[encoded]: https://spec.filecoin.io/#section-algorithms.sdr.encoding
[replication]: https://spec.filecoin.io/#section-algorithms.sdr.replication
[hashed]: https://spec.filecoin.io/#section-algorithms.sdr.merkle-proofs

## Summary

From the above discussion, we propose the Polkadot core should be extended with native support for common cryptographic primitives.
The most general and high impact would be supporting asymmetric elliptic curves (BLS12).
More specific BLS-signatures and SNARKs would also be a huge benefit for Parachains and integrating Filecoin.

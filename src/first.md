# Storage Network on Polkadot

## Introduction

Decentralised storage is an important part of any decentralised network.
This research report analyses the feasibility of a proposal to implement the Filecoin protocol using the Cumulus framework to provide such solution on the Polkadot network.

The proposed decentralised storage solution, PolkaFS, is a Parachain on the Polkadot network.
On a high-level, Storage Miners offer storage capacity for a price and they are PolkaFS Parachain Collators.
Storage Clients requesting for storage can be any node that can communicate with the PolkaFS Parachain, including separate clients or other Parachains using cross-chain messaging (XCM).

Filecoin is a consensus protocol, a data-storage platform, and a marketplace for storing and retrieving data.
The marketplace (Storage and Retrieval markets) is mostly handled off-chain and with [`Deal`] submitted on-chain. This is not covered in this report. For an initial demonstration of the offchain clients, please refer to the [CLI repository].
We explore the first two components in this report as they are areas are technically coupled with the Filecoin implementation and the feasibility of implementing them in Substrate must be understood.

[`Deal`]: https://spec.filecoin.io/#section-glossary.deal
[CLI repository]: https://github.com/common-good-storage/CLI

### Contribution

We present the key differences between the Filecoin protocol and Substrate based Parachain, specifically on consensus mechanism and cryptographic primitives and implementations.
These findings have led to a solution designed to potentially implement the PolkaFS Storage Network as a Parachain with collator selection and using Polkadot relay chain for finality.

## Overview

- [On Consensus Compatibility](./consensus.md)
- [On Crypto Compatibility](./crypto.md)
- [Potential Solution Design](./solution.md)

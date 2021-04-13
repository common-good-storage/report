# Potential Solution Design

We propose potential solutions satisfying requirements described in [consensus] and [cryptographic] compatibility.

[consensus]: ./consensus.md#summary
[cryptographic]: ./crypto.md#summary

![node solution](dotfs_design_v1.png)

It is proposed that the PolkaFS node will consist of:

1. Cumulus Parachain development framework collator node
1. PolkaFS services providing storage and cryptographic proofs on the state of storage, this communicates with the collator via off-chain workers and posts any updates to the chain state with signed transactions.

## On Consensus

The proposed design is one of many possible solutions. This is proposed based on usability for other Parachains and following the current implementation[^1].
In PolkaFS a storage miner is also the block producer and collator. The consensus mechanism differs from Filecoin's Expected Consensus due to incompatibilities outlined in [consensus].
PolkaFS will need to implement the following components on top of the current Cumulus Parachain node:

1. Verifiable random collator selection (mostly reusable)
1. Collator reward calculation based on Storage Power and transactions inclusion

[^1]:
In the current implementation of Cumulus, the collator is assumed as the block producer i.e. producer of the ParaBlock will also produce the Proof of Validity (PoV) block to submit to the relay validator.
Blocks are assumed to not be gossiped around between collators because _a collator has to produce a block with the relay parent of the latest Relay Chain block_.
This means that there is approx. 6 seconds to produce the PoV.

### Verifiably Random Collator Selection

In order to reward block producers on PolkaFS, it is essential to verify that the collator submitting the PoV block is entitled to produce a ParaBlock. This must be done in the State Transition Function submitted to the Relay Validators.
The existing BABE consensus on Polkadot and Substrate chains already provide most of the required functions to select slot winners with a Verifiable Random Function provided by the [Schnorkel] library and wrapped in [sp-consensus-vrf].  In addition to the winning VRF value, the slot winner must also submit the [WinningPoSt], a Proof of Spacetime that the miner has stored what it claimed at the time of winning the slot.

A new `Executor` (which implements [`ExecuteBlock`]) that verifies the VRF value and the WinningPoSt needs to be used when [registering] for [`validate_block`] to verify slot winner and that they have the sealed data up to the previous epoch.

When multiple block producers are selected as winners, the winners race to get their blocks accepted by the Relay Chain validators.
Only at most one block producer will get the rewards and for other block producers there will be no penalty or gain.

#### Randomness in VRF

For each block producer to run the VRF function and produce the WinningPoSt, randomness must be used as part of the input for security.
In Polkadot, [chain randomness] is used, which is the hash of VRF values from the blocks in the epoch `N-2`; where `N` is the current epoch).
Since the Parachain follows the Relay Chain, this may also be the source of randomness in the Parachain.

#### Multiple Collator Selection

The selection of multiple collators for the same slot is possible using the VRF function.
However, there can only be 1 ParaBlock finalised at a certain height, and it will depend on which collation gets more relay validator backing first.
Since the other collators were also selected, but were not the first one to submit collation, they should not be reported and their reputation on the Relay Chain should not be affected.

#### Block Executor Logic

On a high level, the verification logic in the pallet's `Executor` will be similar to the [`BabeVerifier`] in the [`import queue`]:

1. Get header from ParaBlock
1. Get `Seal` signature from `DigestItem::Seal` in the header
1. Get `pre-digest` from `DigestItem::PreRuntime`, the set of `BABE` compatible pre-runtime `Digest`
1. Check the signer of the `Seal` is in a Authority list where they have staked
1. Verify the VRF output with the `pre-digest` items, Relay Chain randomness, other slot information in the epoch
1. Check the signer meets the threshold requirement
1. Verify the `WinningPoST` submitted for this block is by the signer of the `Seal` and uses parent block header as randomness

Currently in Cumulus, the provided consensus in the client is `relay_chain` (and `Aura` when merged).
This will be extended with verifiable random collator selection as used by PolkaFS collators, and supporting any future Parachains with this requirement.

### Collator Rewards

The reward for producing a valid block in PolkaFS will be proportional to the block producer's Storage Power.
It is also desirable to encourage as many paid extrinsics to be included as possible without overloading the block time, therefore the calculation using the `weights` system for transactions should also be used.
The block reward calculation will be done by the `PolkaFS staking` pallet, with input from values of collator Storage Power calculated and stored in `Power` pallet.

#### Storage Power Calculation

A PolkaFS Storage Power calculation follows that of [Filecoin Storage Power Consensus] subsystem.

A storage miner's Storage Power is a proportion of the total Network Storage Power.  The total Network Storage Power is the sum of Quality Adjusted Power (QAP) in the network at a certain block.

$$QAP_{miner } = \frac{\sum_{deals} (fn(Sector Spacetime) * Deal Quality Multiplier)}{Sector Spacetime}$$

**Deal Quality Multiplier**  is used to encourage Storage Miner to provide useful storage for clients with Storage Deals, as opposed to storing their own / unused data (Committed Capacity).
For example, the multiplier may be higher for Storage Deal for the same Spacetime than it is for Committed Capacity.

In Filecoin Plus (renamed from Verified Client program in Filecoin), there is a third tier for the **Deal Quality Multiplier** called [Verified Client Deal].
These deals have a significantly higher multiplier than regular storage deals and committed capacity.
The total number of Verified Client Deals are capped by [DataCap] granted by Notary accounts.
Notary accounts are selected by the community and enacted on-chain by the Root Key Holders of the network.
The rational proposed in Filecoin Plus is that this might help accelerate the attractiveness of Filecoin.

PolkaFS may support such similar governance programs in the future directly with a Distributed Autonomous Organisation (DAO) for nominating Notaries, assigning DataCap or adopting a different **Deal Quality Multiplier** tiers system.

### Block Mining Timing

It is expected that the PolkaFS block time will be longer than that of the Relay Chain since computing and verifying the `WinningPoST` requires work.
Following the [Filecoin block production timing], a Filecoin block is produced every 30 seconds.
_Note that there is no specificity on how long a WinningPoST takes to compute._

For demonstration purposes, the diagram below uses 5 relay blocks per PolkaFS ParaBlock.

![mining process](mining_winningPoST_v0.png)

Once a ParaBlock is included, the collator node extracts it as the Parachain parent for the next block.
The current slot winner will use the latest ParaBlock's BABE header as a source of randomness for selecting sectors to construct the WinningPoST.

*Note: The Relay Chain randomness is not used because it will be known [in advance].*

[Schnorkel]: https://crates.io/crates/schnorrkel
[sp-consensus-vrf]: https://crates.parity.io/sp_consensus_vrf/schnorrkel/struct.PublicKey.html
[registering]: https://github.com/paritytech/cumulus/blob/a90308b7cebdcb616d606c15dc528259bf134b55/pallets/parachain-system/src/validate_block/mod.rs#L71
[WinningPoSt]: https://spec.filecoin.io/#section-algorithms.pos.post.winningpost
[`validate_block`]: https://github.com/paritytech/cumulus/blob/a90308b7cebdcb616d606c15dc528259bf134b55/rococo-parachains/runtime/src/lib.rs#L419
[`ExecuteBlock`]: https://github.com/paritytech/substrate/blob/b0667821e61f4790da84930b7cdb80fb20b48596/frame/support/src/traits.rs#L2292
[`BabeVerifier`]: https://github.com/paritytech/substrate/blob/b0667821e61f4790da84930b7cdb80fb20b48596/client/consensus/babe/src/lib.rs#L953
[`import queue`]: https://github.com/paritytech/substrate/blob/b0667821e61f4790da84930b7cdb80fb20b48596/client/consensus/babe/src/lib.rs#L1608
[chain randomness]: https://wiki.polkadot.network/docs/en/learn-randomness#vrf
[`DigestItem::preRuntime`]: https://github.com/paritytech/substrate/blob/b0667821e61f4790da84930b7cdb80fb20b48596/client/consensus/babe/src/authorship.rs#L275
[Filecoin block production timing]: https://spec.filecoin.io/#section-systems.filecoin_mining.storage_mining.mining_cycle.epoch-timing
[in advance]: #randomness-in-vrf
[Filecoin Storage Power Consensus]: https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.on-power
[Verified Client Deal]: https://spec.filecoin.io/#section-algorithms.verified_clients
[DataCap]: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0003.md

## On the State Transition Function and Cryptography

The STF for the Parachain should support both PoRep and PoSt, that rely on heavy use of the zk-SNARK and elliptic curve pairing.
The most expensive primitives are asymmetric cryptography e.g. SNARKs and aggregate signatures.
This means PolkaFS would benefit in a very high degree of natively supporting elliptic curve pairings and signatures.

A more tight integration between WebAssembly and the Polkadot core can be beneficial for STFs using common resource-intensive cryptography.
For instance, extending the Polkadot Keys to support BLS12 [as planned] and integrating elliptic curve pairing into Polkadot STF WebAssembly would allow much faster verification time for aggregate signatures.
Beside the underlying curve, PolkaFS would also benefit from standard constructions using the pairing, for instance BLS signatures (for matching choice of groups) and SNARKs.

Finally, PolkaFS would benefit from various other symmetric primitives such as the different hash functions (SHA256).

[as planned]: https://wiki.polkadot.network/docs/en/learn-keys#session-keys

# On Consensus Compatibility

Summary on the key differences in the block production and finalisation of two consensus mechanisms.

| Systems | Filecoin | Substrate based Parachain |
| --: | --: | --: |
| Randomness | DRAND | Chain Randomness |
| Block Producer Selection | Weighted with Storage Power | Uniformly Random |
| Valid Blocks Per Block Height | Multiple per Epoch (Tipset) | Single per Slot |
| Finality | Soft Probabilistic Finality | Relay Chain Provided |

## Details

_Only the relevant information for this report are provided here._

### Randomness for block producer selection

#### Randomness in Filecoin

The Distributed Randomness [(DRAND)] protocol supplies the randomness beacon as entropy in the system.
It uses Multi-Party Computations to produce the unbiased, verifiable random value.
Each DRAND node is entered into a trusted setup with their public keys.
The known group of `n` DRAND nodes sign a message using the `t-to-n` threshold BLS signatures in a series of successive rounds.
The full signature can be reconstructed after a node obtains `t` signatures and the randomness is the blake2b hash of the signature.

#### Randomness in Substrate

All block producers with the pre-committed keys produce pseudorandom values with a Verifiable Random Function (VRF), as input there are [several variables] including a *Chain Randomness*, which is the hash of all VRF outputs revealed during block creation in a previous epoch. This means that past randomness has an effect on the future randomness.

### Block Producer Selection

#### Block Production in Filecoin Secret Leader Election

The selection for block producers is done through the [Secret Leader Election].
Block producers run the [Generate Election Proof] algorithm to see if they have won a block for the epoch (i.e. a duration for a tipset, every 30 seconds).

The algorithm includes using the DRAND randomness as an input for computing the VRF output.
The VRF output and the block producer's power[^2] from the [Power Table], are used to compute the [WinCount], _an integer to calculate block reward and block weight_.
Miners with `WinCount > 0` can produce a block.

For the produced block to be valid and fees collectable, Filecoin protocal also requires the leader to produce a [WinningPoSt], which is a proof that they are keeping a sealed copy of the data.
This is needed because the network needs to verify that the miner does indeed have the sealed copy at the epoch of their proposed block.
Successful WinningPoST submission grants the block producer the Filecoin Block Reward, as well as the transaction fees in the valid block.
If a miner misses the end of epoch deadline for the submission, then the miner misses the opportunity to mine a block and get a Block Reward. No penalty is incurred in this case.

[^2]: A value that records the producers' storage power and quality over time

#### Block Production in with Substrate's BABE Consensus

Currently, the Cumulus framework supports "First come first serve" block producer (collator) nodes, where the block finalised depends on which block the Relay Chain validators receive first.
_Aura is being implemented for collators to take turn in producing blocks but yet to be merged._ There is a lot to learn from this on collator selection that we will explore in the next section.
<!-- koivunej: I think emphasized Aura refers to https://github.com/paritytech/cumulus/commits/bkchr-aura-the-long-way but it isn't merged yet? -->
<!-- Whalelephant: keeping this comment here as it is valid until it is not -->

Assuming that other Substrate block producter selection mechanisms will be available, the decentralised PoS mechanism BABE may be adopted, such consensus mechanism has a high level of similarity with that in Filecoin's Storage Miner System.

In a Polkadot epoch there are multiple slots.
All block producers must commit a Verifiable Random Function keys for selection as slot leader to produce a block.
They run the VRF and produce pseudorandom values with [several variables] including fresh [Chain Randomness].
The produced values are then compared to a `threshold` which is specific for each block producer included in the era.
If the value for a specific slot is lower than the `threshold`, the producer can claim as slot leader.
For each slot there maybe _0..n_ valid slot leaders who can produce blocks.
Slot leaders must also be one of the `authorities` listed for a specific epoch.
The list of `authorities` is updated and maintained by the runtime.  

> The block reward and selection is related to the specific utility - Storage Power in Filecoin, BABE Consensus covers the generic case and selects producers uniformly randomly.

### Valid Blocks Per Epoch

#### Filecoin Tipset

At any given epoch (aka block height) `N`, a set of blocks with the same block height may exist if more than zero blocks are produced.
This is called a _tipset_.
Each tipset has a weight, calculated by the total storage the network provides per commitment in the blocks.
The chain selection in Filecoin selects the heaviest chain to build new blocks on.
Filecoin is a chain of tipsets.

Allowing multiple blocks per epoch ensures high throughput.
In addition, it also provides some network security where a malicious node trying to deliberately prevent valid blocks from another node will run up against heavier chains.
<!-- Whalelephant: What about transactions that uncommit storage? -->

> - Tipsets aim to allow for high throughput and waste no work done by block producers.
> - Heaviest chain selection helps network security by preventing selection of chains with omitted blocks of transactions.

#### Substrate's BABE Consensus

In a Substrate node, the client's block import pipeline may use different verifier to determine which block to import.
In the case of BABE, when a block is received, the block must be verified to have been produced by a valid slot leader.
This is done by the client component of the node, which [checks the header] (signed by right authority, VRF proof correct, etc.).
It also checks the inherent data from the runtime (timestamp set in block matches when it was sealed).

Since there can be multiple slot leaders per slot, there may also be multiple blocks produced per slot and forks in the chain are produced.
However, only 1 block will be finalised in 1 slot, uniquely for a block height.

> The verification of the block producer is not checked in the runtime but by the client, which is not part of the State Transition Function submitted to the Relay Chain.

### Finality

#### Filecoin Finality

Filecoin has soft finality where at round `N`, all blocks forked off `N - T` blocks will not be included.
This means that over time, the heaviest chain with weights representing storage power, will become the canonical chain.

#### Parachain Finality

Parachain will be finalised by the Relay Chain.

> Since each ParaBlock needs to build on a Relay Parent (the Relay Chain Block) of the latest Relay Block, it will be difficult to collate multiple ParaBlocks into 1 PoV - i.e. one PoV Block will be one block produced by one block producer.

## Summary

Learning from the previous comparison on the differences in consensus mechanisms leads to some guidelines that is required to implement a storage network in a Substrate based Parachain:

- a mechanism to reward block producers for having high quality storage (either by selection probability / block award distribution)
- allow for high throughput and minimise mining work wasted
- verify selection of block producer in the runtime as part of the STF in the Relay Chain
- incentivise block producers to include all transactions (similar to the heavy chain selection)

Specifically, the incompatibilities in using Filecoin's Expected Consensus are due to Parachain cannot have Tipset like system and that the finality provided by the Relay Chain is provable instead of probabilistic.

[Generate Election Proof]: https://spec.filecoin.io/#section-algorithms.expected_consensus.generateelectionproof
[Power Table]: https://spec.filecoin.io/#section-systems.filecoin_blockchain.storage_power_consensus.storage_power_actor.the-power-table
[WinningPoSt]: https://spec.filecoin.io/#section-algorithms.pos.post.winningpost
[several variables]: https://github.com/paritytech/substrate/blob/master/client/consensus/babe/src/authorship.rs#L232
[checks the header]: https://github.com/paritytech/substrate/blob/a107e1f0b394bc61726e73560332f7d714589e2f/client/consensus/babe/src/verification.rs#L160-L202
[Chain Randomness]: #randomness-in-substrate
[(DRAND)]: https://spec.filecoin.io/#section-libraries.drand
[Secret Leader Election]: https://spec.filecoin.io/#section-algorithms.expected_consensus.secret-leader-election
[WinCount]: https://spec.filecoin.io/#section-algorithms.expected_consensus.generateelectionproof

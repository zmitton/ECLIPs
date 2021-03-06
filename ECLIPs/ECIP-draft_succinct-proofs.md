---
ecip: 1055
title: Succinct PoW Using Merkle Mountain Ranges
author: Zac Mitton @zmitton
discussions-to: <https://github.com/ethereum/EIPs/pulls> (Pull Number TBD)
status: Draft
type: Standards Track
category (*only required for Standard Track): Core
created: 2019-03-12
replaces (*optional): 210, 1094, 641

---

## Simple Summary

The addition of a Merkle-Mountain-Range (MMR) root to blockheaders

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->

A block _chain_ could be interpreted as a linked-list using hashpointers from each block to its previous block. A Merkle-Mountain-Range is a type of tree that, will allow every block to point back to _all_ its previous blocks. This can be accomplished using only a single hashPointer in each block - the current MMR _root_. Adding this will allow proving the cummulative work of an entire chain in Olog(n) time, instead of the current O(n) time. In the case of ethereum, with small block-times, and over 7 million blocks and counting, this will open a ton of doors in the area light-clients, and data verification.



## Motivation
<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->
The recent FlyClient [paper](https://eprint.iacr.org/2019/226.pdf) and [presentation](https://www.youtube.com/watch?v=BPNs9EVxWrA?t=8400) at Scaling Bitcoin Represented a major breakthrough the ability to succinctly, and non-interactivly prove the cumulative work of a blockchain.

The current data structures in Ethereum (merkle-patricia-trees) were made with the sole purpose of allowing succinct verification of all the data. I have personally been working for some time on the missing tools to extract, send, and validate these proofs. This work is mostly done, but there is a problem: 

Without a succinct way to verify work, there is nothing to verify these merkle proofs _against_.

A merkle proof can only verify against a hash that I _already_ trust. The validation chain goes something like this for some contract data `x`:

- `x` is proven to have existed in a particular `storageRoot_x`
- `storageRoot_x` is proven to have existed in a particular `account_x`
- `acount` is proven to have existed in a particular `stateTree_x`
- `stateTree_x` is proven to have existed in a particular `block_x`

And merkle proofs end there: `block_x` unfortunately cannot be trusted.

There are 2 main strategies for verifying a `block`:
- Validate every transaction sinse genesis 
  - This  takes a week on desktop and 
  - Requires the user to have the full dataset anyway, removing the need for merkle proofs in the first place.
  - Is infeasible on mobile.
- Validating the work (of the entire chain)
  - Relies on majority honest miner assumption
  - _currently_ requires all block headers
  - _currently_ requires validating every blockHeader's PoW

<!-- at all unless you check its proof of work. Doing that would yeild a cost parameter:

- `block_x` is proven to have cost about `5 Ether` to produce.

nowing that something cost 5 Ether is not sufficient for most applications. By checking a suffix (blocks after this point in the chain) we can prove that the data cost at least `5n Ether` to produce, where n is the number of blocks checked.

With the simple assumption that the honest chain has 2/3 of the total mining power

The next thing we can do is to request the header before that and validate its PoW as well -->



The problem is that it ends there. And we still have nothing, if the person (or machine) validating cannot validate the block. The cost to produce such proofs is free; To validate blocks we cant use the same strategy. Blocks validate against a chain, which in turn validates through its proof of work. Currently the only way to do this requires 4gb of chain data. Thats way too much. We need something that can work within a few seconds. 

FlyClient is that something, and adding an `MMRroot` to each blockheader is the protocol change that will allow FlyClient to work.

On a high level, it allows the verifier to sample a subset of blocks in a tree, rather than a full amount thought a linked list, improving the time-complexity from O(n) to O(log(n)). The sample subset can be made large enough such it is cryptographically infeasible to ever produce a false proof.

With this EIP, PoW can be proven in < 3 megabytes. This will hit a sweet spot, where for instance, mobile wallets, can request the latest cummulative work every time they are opened (no need to even "sync"). Users can finally verify data in a practical way (without a full-node). The proofs are even small enough for IoT devices.

This may also open doors for side-chains, state-channels, plasma, trustless bridges, atomic swaps, and the peer-to-peer light-client-protocols being worked on by the Geth and Parity teams.

This also solves EIPs 210, 1094, 641 - it gives the EVM a way to access to historic blockhashes. A transaction sender simply provides a merkle proof relating the historic hash against any of the last 256 blocks, and the block against its blockhash (available as op `0x40`), with a size of ~864 bytes.


## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

There is a consensus-breaking rule change proposed here, and then a bit of software around the change will be needed in order to take advantage of the new features. I think it would be best to keep _this_ EIP as just detailing the rule-change, I will summeraize the other tools and link them

### Consensus Rule Change

If `block.number >= ISTANBUL_FORK_NUM` then when constructing the block, set a 16th value `blockHashesRoot` in the header to the `MMRRoot` from the previous block.

This Root is defined by the combining all previous hashes into a [Merkle Mountain Range](https://github.com/juinc/tilap/issues/244) (Peter Todd) using it's block number as leaf index.

Implementation of this tree is up to the clients but its fairly simple and requires only an additional ~500mb (on disk) to a full/archive node.

The MMR structure uses a proccess of "bagging the peaks" in order to create a single _root_ hash. However, this may be sub-optimal for our use case. I would like to ammend this to instead define each root as _the hash of the concatonation of the mountain peaks_. This will make building and verifying proofs much simpler without significantly changing the size of the proofs.

The structure can be implimented as an array. Only a log(n) number of read/writes are done for any operation. A `put` operation needs to be done durring each block validation by the client. About 600 `get`s need to be performed when one of these proofs is requested (however the client can cache the latest proof). There is exactly 1 valid FlyProof for the current tip (latest mined block).

The FlyCLient paper can be followed almost exactly. Find it [here](https://eprint.iacr.org/2019/226.pdf)for specs.

Missing from the paper is a specific way to attain the samples from the blockhash (used as random seed). For this process, I propose we use [slice-sampling](https://en.wikipedia.org/wiki/Slice_sampling). This should be a very efficient algorithm as long as our probability density function is _invertible_. We may require a hash during each iteration (~600). This process is only required for building and verifying proofs, it is not required for instance, of miners.


<!--T
A PoC JS implimentaion with the full, real dataset (7 million tree values) shows the following statistics:
- Calculating a `blockHashesRoot`: <X> seconds
- Creating a succinct Proof `blockHashesRoot`: <Y> seconds
- Size of a Proof <Z> mb
-->




## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->


## Backwards Compatibility
<!--All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->


## Test Cases
<!--Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable.-->

## Implementation
<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

The most obvious need is for a non-full-node to be able to request this Proof from a full-node. This should be done by adding an RPC method. The method will take no arguments and it should return an object with: 

- The lastest blockHeader
- A list of the randomly selected ~600 blockheaders for the proof
  - we need to create the standard deterministic way to pick the the blocks (using blockhash as seed) over the probability denisty function. I propose that we do something like [this](https://en.wikipedia.org/wiki/Slice_sampling)
  - The 64 byte dataset (each) for verifying an ethHash (~2.4 MB)
- A list of proofNodes from the MMR (+"bag proof" nodes)


## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


<!--
#### MY notes
see if there is a requirment for the cummulative work to be included in each blockheader. Hopefully we only tneed the mmr root. In either case we still will probably need to return the cummulative work (or probably the cummulative work _of each blockheader_) with the rpc request.


-->

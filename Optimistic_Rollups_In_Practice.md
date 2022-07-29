# Optimistic Rollups In Practice

*TLDR: The optimistic rollup ecosystem needs fraud prover light clients for the “1/N honest participants” assumption to be meaningful.* 

*We demonstrate that the Optimism sequencer can be circumvented on mainnet, but show that there’s currently no way for end users to conduct Fraud Proofs on either Arbitrum or Optimsim.*

## Motivation

> *The switch to layer two is one where it’s very important for the ecosystem to try and maintain and even improve its decentralization [...], and so we need light clients that can poke into optimism and poke into arbitrum and poke into starknet. And that’s something where there’s been a little bit of theoretical think-y work on, but it’s the sort of thing that I think there is a lot of room for people to slot themselves in and really try to improve that ecosystem a lot…*
> 

*-Vitalik Buterin, ‘22*

Ethereum scaling solutions have once again found themselves in the spotlight. Most of this rollup-centric-worldview type thinking is centered around computational complexity analysis, trust assumptions, and system-wide game-theoretic outcomes. 

But at the end of the day, usage isn’t driven by people reading arxiv papers and codebases for years, trying to come up with objective frameworks to think about what’s “best.”

It’s driven by users who ask:

> How good is the system? Is it easy to use? Deep liquidity? Rich Ecosystem?
> 

> Am I comfortable depositing my money? What’s the probability I get rugged?
> 

The ZK camp of people have chosen to answer these questions by building a polynomial black-box (at least for people without a high level of mathematical sophistication) which theoretically minimizes trust assumptions and gives an extraordinary level of scalability at the cost of system complexity and development time.

The optimistic camp says “if we just make a weak trust assumption (1/N honest participants), then we get all that scalability with only a fraction of the complexity!”

**This 1/N assumption warrants further investigation: what does it actually look like in practice? Is it as accessible as it’s claimed to be?**

## ORs Refresher

*You can probably skip this section if you’re intimately familiar with ORs.*

The general idea is that L2 user transactions get sent to a server somewhere (called a sequencer), which processes the transaction, updates its internal state, and adds the transaction to a batch. Once there are a number of transactions in the batch (usually takes a few minutes), they’re sent on-chain. Optimism and Arbitrum have different data structures associated with the data being posted, but the high-level idea is the same. Then, anyone can see this data sitting on the L1 and make a “fraud proof” if the data’s wrong, which ‘guarantees’ only valid transactions survive. This makes the system trustless since “anyone” can publish a fraud proof in theory. So if even 1/N participants are honest and run a validator, the system can’t rug you.

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/1.png)

In the diagram above,  you see this system mapped out. There are really two subsystems which are important; the Fraud Proof (FP) mechanism and the Sequencer mechanism. Both systems have on-chain components, i.e. L1 contracts which are read from and written to periodically. 

We delineate between these 3 pieces because they’re the principal ways something could go wrong in an optimistic system:

1. No one posts a fraud proof
    1. e.g.  an invalid state root which zeros out your balances becomes finalized.
2. The sequencer acts maliciously
    1. e.g. OffChainLabs is forced to drop Txn’s that touch something like L2 tornado cash.
    2. e.g. Optimism’s sequencer frontruns all of your trades.
3. The on-chain contracts are tampered with
    1. e.g. the bridge contract is permanently zero’d out or something, rendering the fraud proof mechanic useless.

Now that we have a solid background on what *could* go wrong, Let’s dive into Arbitrum and Optimism and see where they’re at in the current state. 

## Arbitrum One

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/2.png)

The above diagram shows what’s happening in Arbitrum under the hood in the optimistic case, i.e. the sequencer is honest and validators are doing the Fraud Proof thing by staking on state commits in the Rollup Contract (or actually submitting fraud proofs, in theory).

*One important detail to add is that in this system, if validators don’t stake on recent state commitments, the rollup chain can’t move forward and the chain is effectively halted, as even L2 updates aren’t able to be posted to the L1 chain.*

This diagram depicts what happens if the sequencer is misbehaving:

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/3.png)

In this case, the user is able to manually submit a raw transaction to a function called sendContractTransaction() on [Arbitrum One’s L1 Inbox contract](https://etherscan.io/address/0x048cc108763de75E080Ad717bD284003aa49eA15#writeContract), which is processed after a few minutes (to avoid conflicts with the internal sequencer state between posting data). This is to say that the sequencer is able to be bypassed altogether. We’ll show a live demo of a similar process of bypassing the sequencer on Optimism in the next section. But it’s safe to assume this is doable presently on mainnet.

So the sequencer is able to be bypassed, and we can mostly ignore that risk. But what about Fraud Proofs?

From [https://developer.offchainlabs.com/docs/mainnet](https://developer.offchainlabs.com/docs/mainnet):

> we currently maintain various levels of control over the system while still in this early Beta phase; this includes contract upgradability, the ability to pause the system, [and] validator whitelisting.
> 

The validator is actually the piece of software that is theoretically responsible for conducting fraud proofs, and is currently ‘whitelisted’ i.e. not available for end-users.

We tried running a node and realized a few things:

- it takes several days to get up and running- probably around 5-7 on the equivalent of an AWS t2.xlarge.
- There’s almost no documentation on what the node actually does or if you can somehow manually run fraud proofs as a non-whitelisted participant.
- it requires a running geth instance to talk to and will make >1m requests to it during the syncing process.

**To conclude with Arbitrum’s current state:** 

1. Fraud Proofs aren’t actually live for end-users, despite most of the documentation having conflicted language on this. (see below)

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/4.png)

1. The centralized sequencer is able to be bypassed.
2. The contracts are still under centralized control by OffChainLabs.

*(note: L2Beat has a great dashboard which covers these risks [https://l2beat.com/projects/arbitrum/](https://l2beat.com/projects/arbitrum/))*

## *Optimism*

### High Level View
![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/5.png)

When a block is produced by the sequencer (block producer), a new merkle hash of the most recent block in the Optimism chain will be calculated and stored in the `StateCommitmentChain` contract, which can be used to prove that a claim about the state is correct.

A decentralized network of verifiers work to keep the block producer (the sequencer) honest, by verifying the integrity of the rollup chain & serve blockchain data to users. Each verifier will reconstruct the Optimism state from the transactions stored in the `CanonicalTransactionChain` contract by downloading and executing all transactions. 

Alternatively, verifiers can also independently generate state merkle roots using the transactions stored in the `CanonicalTransactionChain` contract and compare it with the one being stored in the `StateCommitmentChain` contract.

In return for securing the chain, verifier nodes that manage to produce a valid fraud proof will be rewarded with the bond (in the form of ERC20 tokens) that the block producer deposited into the `BondManager` contract as collateral for the right to write to the state of the L2 chain.

But what happens in the case where we can’t trust the sequencer?

Imagine we have some [contract](https://optimistic.etherscan.io/address/0x028f4bf5fed3f9f39fbbfc1a73c28746923a4c6d#code) which we need to interact with directly:

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/6.png)

*this is a real contract deployed to optimistic ethereum on L2 at 0x028f4bf5fed3f9f39fbbfc1a73c28746923a4c6d*

In this case, let’s say we want to flip the *arbitrary_killswitch* from on to off (1 —> 0).

We need to send raw transaction data to the *enqueue* method on the L1 CTC contract. Fortunately we can construct this data by hand and not mess with web3js or metamask since it’s a simple contract.

```solidity
method signature: 0x5f76f6ab (set fn)
parameter in question: 0000000000000000000000000000000000000000000000000000000000000000

final payload: 0x5f76f6ab0000000000000000000000000000000000000000000000000000000000000000
```

[Then plug into the “write contract” fields of the CTC’s *enqueue* function to send via etherscan.](https://etherscan.io/address/0x5E4e65926BA27467555EB562121fac00D24E9dD2#writeContract)

Once it’s processed, we can see it in the list of latest sequencer batches on the Optimism L1 contract’s Txn history on etherscan. Clearly, we stand out like a sore thumb with the only “enqueue” call to be seen. Normally people don’t use this function because you’re paying for the entire gas cost yourself instead of amortizing it over a full sequencer batch, which gets processed faster anyways. (we paid ~0.008 ETH and waited ~5 mins for a single L2 transaction)

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/7.png)

And sure enough, [our function on L2 was called](https://optimistic.etherscan.io/tx/0xe393d5dae42783d12c76cd7fe54d655317eeccc0a8e786ad21454681dc5031ac). (i.e. our L1 → L2 msg was passed successfully without a sequencer). The switch has been successfully flipped, and imaginary crisis averted.

Here are the addresses we used in the process, if you want to verify for yourself that this is indeed functional on mainnet:

```solidity
my address:                         0x2aa951FeFf3A1C75a3c2ea3696FaFCB8B8505068 (both chains)
contract on optimism:               0x028f4bF5FEd3F9F39fbBfC1a73c28746923A4c6D (optimism scan)
CTC on ETH contract:                0x5E4e65926BA27467555EB562121fac00D24E9dD2 (etherscan)

set function signature: 0x5f76f6ab
"off" param:    0000000000000000000000000000000000000000000000000000000000000001
"on" param:     0000000000000000000000000000000000000000000000000000000000000000

all together: 0x5f76f6ab0000000000000000000000000000000000000000000000000000000000000001
    (this is the thing you submit in the _data(bytes) field)

L1 Txn Hash:    0xda35d8aac29a5fdbe00568540e6389c1d55ed952484f91374fd23b917d944659

L2 effect:      0xe393d5dae42783d12c76cd7fe54d655317eeccc0a8e786ad21454681dc5031ac
```

**Though theoretically possible, it is practically infeasible to do so for the vast majority of users as it requires some level of expertise and experience developing in the web3 space.** The easiest way (no-code) a user is able to this is by doing the following:

1. Navigate to your favourite dApp on Optimism and perform the desired action (e.g. lend assets on Aave, swap between 2 assets on uniswap)
2. On metamask, look for the raw data that will be posted to the blockchain.
    
   ![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/8.png)
    
3. Copy the HEX DATA and navigate to the etherscan page for the CTC contract.
    1. Mainnet: [https://etherscan.io/address/0x5e4e65926ba27467555eb562121fac00d24e9dd2](https://etherscan.io/address/0x5e4e65926ba27467555eb562121fac00d24e9dd2)
    2. Kovan testnet: [https://kovan.etherscan.io/address/0xf7b88a133202d41fe5e2ab22e6309a1a4d50af74#readContract](https://kovan.etherscan.io/address/0xf7b88a133202d41fe5e2ab22e6309a1a4d50af74#readContract)
4. Navigate to the “Write Contract” part of the page, and look for “enqueue”.
    
   ![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/9.png)
    
5. Under `_data`, paste the HEX DATA from step (3) and set the appropriate values for the _gasLimit and _target fields.
    
![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/10.png)
    
6. Send the TX and your TX will be appended to the queue stored in the CTC contract.

So again we see the same thing we saw on Arbitrum. People talk about the sequencer being problematically centralized. But the real issue is that on Optimism, fraud proofs aren’t live at all. If the sequencer censors transactions, you can reasonably get around it. But if the sequencer arbitrarily sets their eth balance to 1M, there’s really nothing that can be done.

![](https://github.com/sinoglobalcap/research/blob/main/static/img/1%20-%20Optimistic%20Rollups/21.png)

You cannot conduct your own fraud proofs, and neither can whitelisted validators- the mechanism still isn’t actually fully implemented. 

**In summary, the current state of Optimism is:**

1. Fraud proofs aren’t live.
2. The centralized sequencer is able to be bypassed.
3. The contracts are still controlled by the Optimism Foundation.

(Again, [L2beat has a great summary of the current risks in the Optimism system](https://l2beat.com/projects/optimism/), which we’ve found to be entirely accurate.)

## Conclusions:

- Neither Optimism nor Arbitrum have functional fraud proofs for end-users.
- We show that sequencers can be circumvented by technical users.
- Optimism and Arbitrum contracts are fully upgradable with no upgrade delay and controlled by their respective organization’s multisigs.
- Sequencer misbehavior can be checked by fraud proofs. But these are highly complex systems and the idea that “anyone” can submit a fraud proof from scratch is extremely far-fetched without pre-written software (a light client) to facilitate this process.
- **A “light client” that independently verifies state, and lets users run fraud proofs against a certain L2 state pushed to L1 should probably be prioritized over sequencer decentralization.**
    - ***without live FPs, decentralizing the sequencer really doesn’t matter that much (it’s mostly circumventable anyways) and would likely hurt performance.***
    - ***making fraud proofs available for users doesn’t hurt performance and drastically reduces trust assumptions.***

*Disclaimer: This information is timely as of July, 2022. Expect that this information will go out of date in the next 3-6 months as both arbitrum and optimism release updates to their sequencers and fraud proof mechanisms/whitelist dynamics.*

---
layout: post
title: Inclusion Lists
---

_Many thanks to Barnabé Monnot ([@barnabemonnot](https://twitter.com/barnabemonnot)) and Francesco ([@fradamt](https://twitter.com/fradamt)) for feedback._

## Outline

- [Introduction](#introduction)
- [General Construction](#general-construction)
- [Delayed Reveal + SSLE & Forward Inclusion Lists](#delayed-reveal)
- [Parallel Auctions](#parallel-auctions)
- [Future Direction](#future-direction)
- [Conclusion](#conclusion)

## Introduction

Censorship resistance is a key feature of decentralization and has been an issue that has garnered a lot of support from the original crypto advocates. However until now, other points of the blockchain trilemma have been getting more attention, especially scalability, which has resulted in the growth of rollups, and app-chains. This has since changed from the shocking details of the Tornado Cash incident. 

On 8th August 2022, the Office of Foreign Assets cracked down on the privacy mixer [Tornado Cash](https://blog.chainalysis.com/reports/tornado-cash-sanctions-challenges/), and issued 44 Ethereum addresses on their publicly available Specially Designated Nationals List (SDN List). This essentially made it illegal for centralized crypto businesses, subject to US jurisdiction, to interact with these addresses. Mining and staking pools were greatly affected - the largest Ethereum miner based in the US, Ethermine, stopped including Tornado router transactions in its blocks a day after the SDN list was announced. 

It is this exact form of censorship that we aim to prevent through technology like Inclusion Lists. When there are gas-paying eligible transactions in the mempool, the builder should not be submitting blocks less than maximum size, unless they can fill the block completely with other transactions that may arise from exclusive order flow and are not present in the mempool.

## General Construction

A basic mechanism to start thinking about Inclusions Lists is as follows (in reference to each slot):

1. Proposer observes mempool and creates a list of transactions that they think deserve to be included. Proposer then broadcasts this inclusion list before beginning the auction of the slot. 
2. Builder makes a proposed block, either with mempool transactions or EOF (exclusive orderflow). Submits this exec body along with a bid to the auction. 
3. Proposer accepts winning header that either has a full block, or includes transactions from the inclusion list (as well as a high enough bid). 

However, with this proposed design there is one obvious flaw that surrounds the auction process between proposers and builders. If builders publish a publicly available “blacklist” of transactions that they cannot include (similar to the SDN list), then proposers might be enticed to publish an empty inclusion list or an inclusion list that complies with this blacklist in order to maximize the number of bids in the auction process as well as the winning bid amount. Proposers could also do off-band communication with certain builders in order to make sure that the inclusion list is acceptable to some of the highest-performing builders. Therefore there is a strong need to balance having an honest inclusion list as well as fair participation in the auction process. 

## Delayed Reveal + SSLE & Forward Inclusion Lists {#delayed-reveal}

One way to get around the problem of no bids in the auction process due to a restrictive inclusion list is to publish the list after accepting the builder bid in auction. This locks in the builder and forces them to comply to the IL (inclusion list) or otherwise risk losing their bid. One caveat is that, builders would then be able to use history to identify proposers who make them build blocks against their internal black list and purposely not bid in their auctions. This could however be avoided using Single Secret Leader Election (SSLE). As per [this paper](https://eprint.iacr.org/2020/025.pdf) by Dan Boneh et al. introducing SSLE, its aim is to 

> ….to randomly choose exactly one leader from the group with the restriction that the identity of the leader will be known to the chosen leader and nobody else. At a later time, the elected leader should be able to publicly reveal her identity and prove that she has won the election.
> 

However, another disadvantage to this design is that it opens up the possibility of MEV extraction for the proposer. Between the time of the builder bidding in the auction, and the proposer publishing the inclusion list, new transactions could have entered the mempool, giving the proposer access to some MEV revenue. This is a problem as it could result in the proposer relying on another entity to do MEV searching and create an optimal inclusion list - reversing the original benefits of PBS. 

[Forward inclusion lists](https://notes.ethereum.org/@fradamt/H1ZqdtrBF) (FIL) were later devised to primarily solve incentive compatibility issues that may arise from IL but to also help with the above MEV extraction problem. Incentive issues arise because proposers might not want to disadvantage themselves if they are creating their own IL, and therefore release only empty IL. FIL were then created under the fair assumption that another random proposer may not necessarily have any vested interest in the next proposers auction outcome, thus allowing to circumvent this compatibility issue. This particular variation of inclusion lists assumes two-slot PBS, which is when a proposer block is first published and then a builder block. The proposer block is essentially a beacon block, therefore splitting up the main exec block from the beacon block, and the builder block is this main exec block. Forward inclusion lists can be summarized as - the proposer at slot (n), creates the inclusion list for the builder at slot (n+1), and publishes this list along with the beacon block. This has the same benefits as the delayed reveal above, but without the MEV risk as the IL is based on a snapshot of the past mempool state rather than the future state, and with incentive alignment.

Forward inclusion lists are however highly dependent on the type of PBS being implemented. For example, above we assumed two-slot PBS, but did not specify whether it would be a [block auction or slot auction](https://mirror.xyz/0x03c29504CEcCa30B93FF5774183a1358D41fbeB1/CPYI91s98cp9zKFkanKs_qotYzw09kWvouaAa9GXBrQ). The MEV resistance benefits would only be possible through slot auctions where the builder does not commit to the block contents at the time of auction. With block auctions, the only way to enforce the IL would be by releasing the IL pre-auction, which brings the old problems of empty IL from the general construction. Additionally, if we instead go the different route of traditional (one-slot) PBS where the proposer at slot (n) would be making the IL for the proposer at slot (n+1), we solve the problem of having an empty inclusion list as the proposer at slot (n) should not necessarily care about the auction of the (n+1) proposer, but the (n+1) proposer would still suffer from low / no bids, due to the list being published pre-auction. One can read more about the set of possible outcomes for traditional PBS in [this article](https://www.notion.so/Inclusion-list-economics-b0d61d0e21a74963a54448e48d9adc8a?pvs=21). 

## Parallel Auctions

[Parallel auctions](https://notes.ethereum.org/s3JToeApTx6CKLJt8AbhFQ#Solution-1-can-we-run-many-pre-confirmation-private-PBS-auctions-in-parallel) is when the slot auction for a whole block is split into multiple, parallel auctions for parts of the block (say for “one transaction” segments). The main purpose of this process is to increase the cost of censorship than in the general construction. This happens because the censor would need to outbid the censored transaction across multiple auctions. If the censored transaction wants to pay (P) in priority fees, the censor would need to pay (P+1)*N to exclude the transaction, where N is the number of parallel auctions. 

The drawback comes in the coordination between different builders in the multiple auctions. If the builder at block (n) and block (n+1) both pick the same transaction to build, the builder at (n+1) would be forced to send in an empty block and take the loss on the bid.

## Future Direction

In the distant future, we may see censorship resistance methods being a combination of either {delayed reveal, forward inclusion lists} with parallel auctions. When choosing between delayed reveal or forward inclusion lists it is important to consider factors such as size of blockchain history and strain on P2P communication. The IL would be included within the block header for delayed reveal, adding extra (unnecessary) data to the size of blockchain history, which would affect only archival nodes. Whereas for forward inclusion lists the IL would be temporary data on the P2P network, enforced by attestors. In this case, the delayed reveal would be preferred as bandwidth of P2P is a more concerning problem than the size of blockchain history. 

Parallel auctions is a new concept that is largely undeveloped. More thorough analysis is needed on its characteristics moving forward, namely in: builder coordination and auction structure. Collision errors where multiple builders in the parallel auction have chosen the same transaction to build, would result in wasted resources that should be attempted to be minimized. Additionally, parallel auction structure still needs to be optimized for highest revenue and censorship resistance. New designs have already emerged such as [secondary auctions](https://notes.ethereum.org/@fradamt/H1TsYRfJc#Secondary-auctions) that allows for a main body to be the whole block, and secondary blocks to be as small as they want. If a proposers sees a bid for a main body it can take that or multiple smaller secondary bodies (that would collectively reach a main exec body in size) - whichever has a higher profit! This allows block producers who need to have control over a large part of the block due to MEV extraction, and EOF to be able to execute without having to worry about multiple parallel auctions to guarantee it.

An important technology that has not been discussed thus far is encrypted mempools which is also relevant to the problem of censorship resistance, although mostly referenced in the context of MEV resistance. Encrypted mempools could solve censorship resistance, as the builder would not be able to discern the nature of the transaction (other than the associated bid), until the block has been committed. Therefore, there would not be enough information for censorship based on tx.sender or other attributes to be possible! 

An aspect to think about as IL research matures is how to include incentive structures within the mechanism design to ensure optimal behavior so as to not rely solely on altruism. The two main questions that come to mind when thinking about this is:

1. How do we incentivize proposers to make inclusion lists?
    1. This is a difficult problem to solve as transactions that are included in the IL are not guaranteed to make it into the block due to the possibility of EOF. 
    2. Users who want to get onto the IL could pay the proposer a fee, thereby giving proposers a reward for creating and including that transaction in the list. However, the user is not guaranteed to get in the next block, but should be guaranteed to get included eventually.
    3. Is this better than how the current base-fee and tip structure works for these transactions? In this case too, with a high enough tip, a non-censoring proposer / builder will come in the future and pick up the transaction.
2. What metric should proposers use to pick transactions for the inclusion list?
    1. Obvious answer would be that the list should be the transactions with highest-paying tips as this would satisfy the builders and encourage higher prices in the block auction. 
    2. However, do also want to optimize for other factors such as time spent in mempool, tx.sender origin etc.

Inclusion lists also raise potential griefing attacks between builders and proposers that are important to note. For example, say the builder at slot (n) has a sizable amount of potential MEV they could extract, the builder at slot (n+1) could steal it if the builder (n) is censoring. This is by submitting a transaction that violates a publicly-known blacklist (ie: a tornado cash transaction) into the mempool and thereby getting onto the IL. Since on average blocks are half-full (from EIP-1559), censoring a single transaction would cost 15 million gas, as a “full-block” would be considered a block with the maximum limit of 30 million gas. If this is more expensive than the expected revenue from the block (rewards + MEV), then the builder at slot (n) would be forced to withdraw from producing, allowing the builder at (n+1) to steal all of the MEV from that slot along with the normal MEV between the two slots. Some could see this as a feature as well, since this allows participants to push out censoring builders from the network for very cheap, dramatically increasing censorship resistance in the network. 

## Conclusion

Optimistically, censorship resistance should push regulation to stop holding consensus participating nodes responsible for enforcing sanctions and regulations, as inclusion lists would leave these parties without any other option. However, if the censoring jurisdictions were to take a harder stance and ban these validators, this could result in these nodes being pushed out. This would adversely affect the validator set, and could even result in a continuous hide and seek between nodes and censoring jurisdictions as more of them start regulating, which would be extremely detrimental to the future of the network. 

Therefore, inclusion lists raises one of the most fundamental questions of blockchain: **Should the protocol be sensitive to the demands of specific parties if it guarantees future success?**
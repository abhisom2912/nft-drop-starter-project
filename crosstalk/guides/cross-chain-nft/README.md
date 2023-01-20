---
title: Cross-chain NFT (ERC-1155)
sidebar_position: 2
description: A sample ERC-1155 contract using Router CrossTalk
---

### Overview
In this section, we will create cross-chain ERC-1155 standard NFTs. We will be using the burn-mint mechanism to transfer the NFTs to different chains where we will be burning the NFTs from user’s account on the source chain and mint those NFTs to the recipient on the destination chain.

We will be using the standard ERC-1155 contract from Openzeppelin and add some extra code to make it cross-chain compatible. Also unlike the earlier section, we will not be using the acknowledgement this time. This will show us how the contract will look if we don’t want to handle the acknowledgement. 

We will also create a mapping that will store the addresses of our contracts on different chains so that we can match whether the request on the destination was originated from our contract on the source chain.


- [Using Gateway Contract Directly](./cross-chain-nft/using-gateway-contract)
- [Using Router CrossTalk Utils Library](./cross-chain-nft/using-contract-utils-library)

---
id: marketplace
title: Marketplace
sidebar_label: Marketplace
---
import {Github} from "@site/src/components/codetabs"

In this tutorial, you'll learn the basics of an NFT marketplace contract where you can buy and sell non-fungible tokens for $NEAR. In the previous tutorials, you went through and created a fully fledged NFT contract that incorporates all the standards found in the [NFT standard](https://nomicon.io/Standards/NonFungibleToken). 

## Introduction

Throughout this tutorial, you'll learn how a marketplace contract could work on NEAR. This is meant to be an example and there is no canonical implementation. Feel free to branch off and modify this contract to meet your specific needs.

Using the same repository as the previous tutorials, if you checkout the `8.marketplace` branch, you should have the necessary files to complete the tutorial.

```bash
git checkout 8.marketplace
```

## File structure {#file-structure}

The changes made include a new root level directory called `market-contract`. This contains both the build script, dependencies as well as the actual contract code as outlined below.

```
market-contract
├── Cargo.lock
├── Cargo.toml
├── README.md
├── build.sh
└── src
    ├── external.rs
    ├── internal.rs
    ├── lib.rs
    ├── nft_callbacks.rs
    ├── sale.rs
    └── sale_views.rs
```

Usually, when doing work on multiple smart contracts that all pertain to the same repository, it's a good idea to structure them in their own folders as done in this tutorial. To make your work easier when building the smart contracts, we've also modified the repository's `package.json` file so that building both smart contracts can be easily done by running the following command.

```bash
yarn build
```
This will install the dependencies for both contracts and compile them to `wasm` files that are stored in the following directory.

```
nft-tutorial
└── out
    ├── main.wasm
    └── market.wasm
```

## Understanding the contract

At first, the contract can be quite overwhelming but if you strip away all the fluff and dig into the core functionalities, it's actually quite simple. This contract was designed for only one thing - to allow people to buy and sell NFTs for NEAR. This includes the support for paying royalties, updating the price of your sales, removing sales and paying for storage.

Let's go through the files and take note of some of the important functions and what they do.

## lib.rs {#lib-rs}

This file outlines what information is stored on the contract as well as some other crucial functions that you'll learn about below.

### Initialization function {#initialization-function}

The first function you'll look at is the initialization function. This takes an `owner_id` as the only parameter and will default all the storage collections to their default values.

<Github language="rust" start="85" end="105" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/lib.rs" />

### Storage management model {#storage-management-model}

Next, let's talk about the storage management model chosen for this contract. On the NFT contract, users attached $NEAR to the calls that needed storage paid for. For example, if someone was minting an NFT, they would need to attach `x` amount of NEAR to cover the cost of storing the data on the contract. 

On this marketplace contract, however, the storage model is a bit different. Users will need to deposit $NEAR onto the marketplace to cover the storage costs. Whenever someone puts an NFT for sale, the marketplace needs to store that information which costs $NEAR. Users can either deposit as much NEAR as they want so that they never have to worry about storage again or they can deposit the minimum amount to cover 1 sale on an as-needed basis. 

You might be thinking about the scenario when a sale is purchased. What happens to the storage that is now being released on the contract? This is why we've introduced a storage withdrawal function. This allows users to withdraw any excess storage that is not being used. Let's go through some scenarios to understand the logic. The required storage for 1 sale is 0.01 NEAR on the marketplace contract.

**Scenario A**

- Benji wants to list his NFT on the marketplace but has never paid for storage. 
- He deposits exactly 0.01 NEAR using the `storage_deposit` method. This will cover 1 sale.
- He lists his NFT on the marketplace and is now using up 1 out of his prepaid 1 sales and has no more storage left. If he were to call `storage_withdraw`, nothing would happen.
- Dorian loves his NFT and quickly purchases it before anybody else can. This means that Benji's sale has now been taken down (since it was purchased) and Benji is using up 0 out of his prepaid 1 sales. In other words, he has an excess of 1 sale or 0.01 NEAR.
- Benji can now call `storage_withdraw` and will be transferred his 0.01 NEAR back. On the contract's side, after withdrawing, he will have 0 sales paid for and will need to deposit storage before trying to list anymore NFTs.

**Scenario B**

- Dorian owns one hundred beautiful NFTs and knows that he wants to list all of them.
- To avoid having to call `storage_deposit` everytime he wants to list an NFT, he calls it once. Since Dorian is a baller, he attaches 10 NEAR which is enough to cover 1000 sales. He now has an excess of 9 NEAR or 900 sales.
- Dorian needs the 9 NEAR for something else but doesn't want to take down his 100 listings. Since he has an excess of 9 NEAR, he can easily withdraw and still have his 100 listings. After calling `storage_withdraw` and being transferred 9 NEAR, he will have an excess of 0 sales.

With this behavior in mind, the following two functions outline the logic.

<Github language="rust" start="110" end="173" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/lib.rs" />

In this contract, the storage required for each sale is 0.01 NEAR but you can query that information using the `storage_minimum_balance` function. In addition, if you wanted to check how much storage a given account has paid, you can query the `storage_balance_of` function.

With that out of the way, it's time to move onto the `nft_callbacks.rs` file where you'll look at how NFTs are put for sale.

## nft_callbacks.rs {#nft_callbacks-rs}

This file is responsible for the logic used to put NFTs for sale. If you remember from the [marketplaces section](/tutorials/nfts/approvals#marketplace-integrations) of the approvals tutorial, when users call `nft_approve` and pass in a message, it will perform a cross-contract call to the `receiver_id`'s contract and call the method `nft_on_approve`. This `nft_callbacks.rs` file will implement that function.

### Listing logic {#listing-logic}

The first important thing to note is the `SaleArgs` struct. This is what the market contract is expecting the message that the user passes into `nft_approve` on the NFT contract to be. This outlines the sale price in yoctoNEAR for the NFT that is listed.

<Github language="rust" start="5" end="10" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/nft_callbacks.rs" />

Next, we'll look at the `nft_on_approve` function which is called via a cross-contract call by the NFT contract. This will make sure that the signer has enough storage to cover adding another sale. It will then attempt to get the `SaleArgs` from the message and create the listing.

<Github language="rust" start="32" end="134" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/nft_callbacks.rs" />

## sale.rs {#sale-rs}

Now that you're familiar with the process of both adding storage and listing NFTs on the marketplace, let's go through what you can do once a sale has been listed. The `sale.rs` file outlines the functions for updating the price, removing, and purchasing NFTs.

### Sale object {#sale-object}

It's important to understand what information the contract is storing for each sale object. Since the marketplace has many NFTs listed that come from different NFT contracts, simply storing the token ID would not be enough to distinguish between different NFTs. This is why you need to keep track of both the token ID and the contract by which the NFT came from. In addition, for each listing, the contract must keep track of the approval ID it was given to transfer the NFT. Finally, the owner and sale conditions are needed.

<Github language="rust" start="7" end="18" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/sale.rs" />

### Removing sales {#removing-sales}

In order to remove a listing, the owner must call the `remove_sale` function and pass the NFT contract and token ID. Behind the scenes, this calls the `internal_remove_sale` function which you can find in the `internal.rs` file. This will assert one yoctoNEAR for security reasons.

<Github language="rust" start="23" end="34" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/sale.rs" />

### Updating price {#updating-price}

In order to update the list price of a token, the owner must call the `update_price` function and pass in the contract, token ID, and desired price. This will get the sale object, change the sale conditions, and insert it back. For security reasons, this function will assert one yoctoNEAR.

<Github language="rust" start="36" end="65" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/sale.rs" />

### Purchasing NFTs {#purchasing-nfts}

For purchasing NFTs, you must call the `offer` function. It takes an `nft_contract_id` and `token_id` as parameters. You must attach the correct amount of NEAR to the call in order to purchase. Behind the scenes, this will make sure your deposit is greater than the list price and call a private method `process_purchase` which will perform a cross-contract call to the NFT contract to invoke the `nft_transfer_payout` function. This will transfer the NFT using the [approval management](https://nomicon.io/Standards/Tokens/NonFungibleToken/ApprovalManagement) standard that you learned about and it will return the `Payout` object which includes royalties.

The marketplace will then call `resolve_purchase` where it will check for malicious payout objects and then if everything went well, it will pay the correct accounts.

<Github language="rust" start="67" end="99" url="https://github.com/near-examples/nft-tutorial/blob/8.marketplace/market-contract/src/sale.rs" />

## sale_view.rs {#sale_view-rs}

The final file we'll go through is the `sale_view.rs` file. This is where some of the enumeration methods are outlined. It allows users to query for important information regarding sales.

### Creating a sub-account

Run the following command to create a sub-account marketplace of your main account with an initial balance of 25 NEAR which will be transferred from the original to your new account.

```bash
near create-account marketplace.$NFT_CONTRACT_ID --masterAccount $NFT_CONTRACT_ID --initialBalance 25
```

Next, you'll want to export an environment variable for ease of development:

```bash
export MARKETPLACE_CONTRACT_ID=marketplace.$NFT_CONTRACT_ID
```

Using the build script, deploy the contract as you did in the previous tutorials:

```bash
near deploy --wasmFile out/market.wasm --accountId
 $MARKETPLACE_CONTRACT_ID
```

### Initialization and minting

Since this is a new contract, you'll need to initialize it. Use the following command to initialize the contract:

```bash
near call $MARKETPLACE_CONTRACT_ID new '{"owner_id": "'$MARKETPLACE_CONTRACT_ID'"}' --accountId $MARKETPLACE_CONTRACT_ID
```

### Total supply {#total-supply}

To query for the total supply of NFTs listed on the marketplace, you can call the `get_supply_sales` function. An example can be seen below.

```bash
near view $MARKETPLACE_CONTRACT_ID get_supply_sales
```

### Total supply by owner {#total-supply-by-owner}

To query for the total supply of NFTs listed by a specific owner on the marketplace, you can call the `get_supply_by_owner_id` function. An example can be seen below.

```bash
near view $MARKETPLACE_CONTRACT_ID get_supply_by_owner_id '{"account_id": "benji.testnet"}'
```

### Total supply by contract {#total-supply-by-contract}

To query for the total supply of NFTs that belong to a specific contract, you can call the `get_supply_by_nft_contract_id` function. An example can be seen below.

```bash
near view $MARKETPLACE_CONTRACT_ID get_supply_by_nft_contract_id '{"nft_contract_id": "fayyr-nft.testnet"}'
```

### Query for listing information {#query-listing-information}

To query for important information for a specific listing, you can call the `get_sale` function. This requires that you pass in the `nft_contract_token`. This is essentially the unique identifier for sales on the market contract as explained earlier. It consists of the NFT contract followed by a `DELIMITER` followed by the token ID. In this contract, the `DELIMITER` is simply a period: `.`.  An example of this query can be seen below.

```bash
near view $MARKETPLACE_CONTRACT_ID get_sale '{"nft_contract_token": "fayyr-nft.testnet.token-42"}'
```

In addition, you can query for paginated information about the listings for a given owner by calling the `get_sales_by_owner_id` function.

```bash
near view $MARKETPLACE_CONTRACT_ID get_sales_by_owner_id '{"account_id": "benji.testnet", "from_index": "5", "limit": 10}'
```

Finally, you can query for paginated information about the listings that originate from a given NFT contract by calling the `get_sales_by_nft_contract_id` function.

```bash
near view $MARKETPLACE_CONTRACT_ID get_sales_by_nft_contract_id '{"nft_contract_id": "fayyr-nft.testnet", "from_index": "5", "limit": 10}'
```

## Conclusion

In this tutorial, you learned about the basics of a marketplace contract and how it works. You went through the [lib.rs](#lib-rs) file and learned about the [initialization function](#initialization-function) in addition to the [storage management](#storage-management-model) model. 

You then went through the [nft_callbacks](#nft_callbacks-rs) file to understand how to [list NFTs](#listing-logic). In addition, you went through some important functions needed for after you've listed an NFT. This includes [removing sales](#removing-sales), [updating the price](#updating-price), and [purchasing NFTs](#purchasing-nfts).

Finally, you went through the enumeration methods found in the [`sale_view`](#sale_view-rs) file. These allow you to query for important information found on the marketplace contract. 

You should now have a solid understanding of NFTs and marketplaces on NEAR. Feel free to branch off and expand on these contracts to create whatever cool applications you'd like. In the [next tutorial](9-series.md), you'll learn how to take the existing NFT contract and optimize it to allow for:
- Lazy Minting
- Creating Collections
- Allowlisting functionalities
- Optimized Storage Models

:::note Versioning for this article

At the time of this writing, this example works with the following versions:

- near-cli: `3.0.0`
- NFT standard: [NEP171](https://nomicon.io/Standards/Tokens/NonFungibleToken/Core), version `1.0.0`

:::

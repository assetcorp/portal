# Decentralized exchange (DEX) sample

## Overview

To enable DEFI applications on the IC, canisters need to interact with token canisters and the ledger canister. This sample dapp illustrates how to facilitate these interactions. You can see a quick introduction on [YouTube](https://youtu.be/fLbaOmH24Gs).

The sample exchange is implemented in [Motoko](https://github.com/dfinity/examples/tree/master/motoko/defi) and [Rust](https://github.com/dfinity/examples/tree/master/rust/defi) and can be seen [running on the IC](https://gzz56-daaaa-aaaal-qai2a-cai.ic0.app/).

## Architecture

The design of the IC allows for more complex on-chain computation. In combination with cheap storage, it is possible to have on-chain order books. This sample code takes advantage of these features and stores user balances and orders inside the exchange canister. The sample exchange functionality can be condensed into the following steps:

-   Exchange takes custody of funds (different mechanism for tokens and ICP, see below).

-   Exchange updates internal balance book.

-   Users trade on exchange causing the exchange to update its internal balance book.

-   Withdrawing funds from the exchange gives custody back to the user.

### Interface

Request user-specific ledger account identifier from the exchange. This unique account identifier represents a user-specific subaccount in the exchange’s ledger account, allowing it to differentiate between user deposits.

    getDepositAddress: () -> (blob);

Initiate user deposit to exchange. If the user wants to deposit ICP, the exchange moves the funds from the user-specific deposit address to its default subaddress and adjusts the user’s ICP balance on the DEX. If the user wants to deposit a DIP token, the exchange tries to move the approved funds to its token account and adjusts the user’s balance.

    deposit: (Token) -> (DepositReceipt);

Withdraw request to the exchange. The exchange will send funds back to the user if the user has a sufficient balance.

    withdraw: (Token, nat, principal) -> (WithdrawReceipt);

Place new order to exchange. If the order matches an existing order, it will get executed.

    placeOrder: (Token, nat, Token, nat) -> (OrderPlacementReceipt);

Allows the user to cancel submitted orders.

    cancelOrder: (OrderId) -> (CancelOrderReceipt);

Request user’s balance on exchange for a specific token.

    getBalance: (Token) -> (nat) query;

### Fee

It is the responsibility of the exchange to subtract fees from the trades. This is important because the exchange must pay fees for withdrawals and internal transfers.

## Token exchange walkthrough

This section contains a detailed walkthrough of the core exchange functionalities. Most interactions require multiple steps and are simplified by using the provided frontend. Since the exchange canister functions are public, advanced users can use `dfx` to interact with the exchange.

### Depositing ICP

The ledger canister provides a unique interface so that interactions with ICP need to be resolved separately.

-   The user calls the `getDepositAddress` function. The response contains a unique account identifier representing a user-specific subaccount controlled by the exchange. The exchange can identify the user responsible for deposits through this address.

-   User transfers ICP to the fetched deposit address and waits for the transfer to complete.

-   To notify the exchange, the user calls `deposit` with the ICP token principal. The exchange will look into the user’s subaccount and adjust the user’s balance on the exchange. In a second step, the exchange will transfer the funds from the user subaccount to its default subaccount, where the exchange keeps all of its ICP.

### Depositing tokens

There are a number of token standards in development (e.g. IS20, DFT, and DRC20); This sample uses DIP20.

-   The user calls the `approve` function of the token canister. This gives the exchange the ability to transfer funds to itself on behalf of the user.

-   Similar to the ICP depositing, the user calls the `deposit` function of the exchange. The exchange then transfers the approved token funds to itself and adjusts the user’s exchange balance.

### Placing orders

After depositing funds to the exchange, the user can place orders. An order consists of two tuples. `from: (Token1, amount1)` and `to: (Token2, amount2)`. These orders get added to the exchange. What happens to these orders is specific to the exchange implementation. This sample provides a simple exchange that only executes exactly matching orders. Be aware this is just a toy exchange, and the exchange functionality is just for completeness. 

### Withdrawing funds

Compared to depositing funds, withdrawing funds is simpler. Since the exchange has custody of the funds, the exchange will send funds back to the user on `withdraw` requests. The internal exchange balances are adjusted accordingly.

## Common mistakes

-   Concurrent execution: If canister functions have `await` statements, it is possible that execution is interleaved. To avoid bugs, it is necessary to carefully consider the placement of data structure updates to prevent double-spend attacks.

-   Floating Points: More advanced exchanges should take care of floating points and make sure to limit decimals.

-   No panics after await: When a panic happens, the state gets rolled back. This can cause issues with the correctness of the exchange.

Curly Cider Chinchilla

high

# Callback gas usage is not checked

## Summary
In Arbitrum, a seller (auction setter) can force a buyer to purchase at stale prices

## Vulnerability Detail
I've <ins>underlined</ins> some key assumptions below.

When setting up an FPAM auction, <ins>a seller can include their own callback contract with arbitrary code</ins>. Apart from passing the `hasPermission` check and including the required callback methods, the <ins>callback contract is free to execute arbitrary code</ins> when called in `AuctionHouse#purchase`.

<ins>In Arbitrum, a transaction which exceeds the block gas limit will be sent back to the retry queue, perpetually, until it gets below the limit</ins>. A seller is able to abuse this by toggling high/low gas consumption on the callback contract. 
>Toggle high gas consumption 

`purchase` called by buyer will trigger the callback contract, which will consume more gas than the block limit. In Arbitrum this transaction will be sent to the retry queue perpetually so it is basically stuck.

>Toggle low gas consumption 

When the buyer wants to let `purchase` execute, they toggle low gas on the callback, and the transaction will go through.
## Impact
1. Arbitrage
>Stale price > current market price

Seller can buy `baseToken` with the cheaper price on the open market, and then sell to the buyer at the stale price, and the difference is now profit.

>Stale price < current market price

Seller can simply not supply `baseToken` to the callback contract and it will revert the `purchase` call.

2. DOS

Buyers cannot buy `baseToken` as advertised. Since the transaction enters the retry queue perpetually, multiple buyers can attempt to purchase it, and the first caller of `purchase` will not necessarily be first in line for the `baseToken`.


## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L201
## Tool used

Manual Review

## Recommendation

Set a gas limit for callbacks. Could refer to GMX's implementation with their `callbackGasLimit`.

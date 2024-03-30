Special Black Spider

medium

# onCurate callback can be set such that the curate call reverts, preventing curator from rightfully receiving fees

## Summary
onCurate callback can be set such that the curate call reverts, preventing curator from rightfully receiving fees


## Vulnerability Detail
Contract: AuctionHouse.sol

The curator plays an important role in the protocol and is responsible for certain activities that can boost a sale, these could be marketing interactions, listing on third parties or vouching. 

Now we need to understand on which occasions and under what conditions a curate fee is applied:

a) FeeData.curated must be true
b) FeeData.curatorFee must be set

Once these both conditions are fulfilled, a curator fee is applied upon:

a) purchase
-> [https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L247](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L247)

b) settle
-> [https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L496](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L496)


Now we need to understand that these conditions will be only fulfilled after the curate function has been called:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L651](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L651)

(So far, everything is the same as described in the “Auction creator can prevent curator fee via revoking approval or insufficient tokens in the wallet” issue)

The problem is, there is a second scenario how this can be exploited which causes a valid reason to create a second issue:

The onCurate function can be coded in such a way that it reverts, effectively preventing the curator from invoking the curate function. 

The curator could however have already provided services and will then remain unpaid.

*Please note that this issue is different from the “Auction creator can prevent curator fee via revoking approval or insufficient tokens in the wallet” issue because even if the curate tokens would be transferred in at the beginning of the auction, the callback on the “curate” call would still revert and not setting the desired fee.

## Impact
IMPACT:

a) Prevention of curator fee payout
b) DoS of important/basic functionality


## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L247
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L496
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L651

## Tool used

Manual Review

## Recommendation
Consider not executing a callback on the curate call and align with the other issue’s recommendation to transfer the curator fees in, upon auction start and then refund any unclaimed curator fees after the auction has fully settled (and the sold capacity has been determined).

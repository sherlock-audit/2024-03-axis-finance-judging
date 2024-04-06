Special Black Spider

medium

# Unsafe casting within _purchase function can result in overflow

## Summary
Unsafe casting within _purchase function can result in overflow

## Vulnerability Detail
Contract: FPAM.sol

The _purchase function is invoked whenever a user wants to buy some tokens from an FPAM auction. 

Note how the amount_ parameter is from type uint96:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L128](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L128)

The payout is then calculated as follows:

amount * 10^baseTokenDecimals / price

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L135](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L135)

The crux: The quote token can be with 6 decimals and the base token with 18 decimals.

This would then potentially result in an overflow and the payout is falsified. 

Consider the following PoC:

amount = 1_000_000_000e6 (fees can be deducted or not, this does not matter for this PoC)

baseTokenDecimals = 18

price = 1e4

This price basically means, a user will receive 1e18 BASE tokens for 1e4 (0.01) QUOTE tokens, respectively a user must provide 1e4 (0.01) QUOTE tokens to receive 1e18 BASE tokens

The calculation would be as follows:

1_000_000_000e6 * 1e18 / 1e4 = 1e29

while uint96.max = 7.922….e28

Therefore, the result will be casted to uint96 and overflow, it would effectively manipulate the auction outcome, which can result in a loss of funds for the buyer, because he will receive less BASE tokens than expected (due to the overflow).

It is clear that this calculation example can work on multiple different scenarios (even though only very limited because of the high bidding [amount] size) . However, using BASE token with 18 decimals and QUOTE token with 6 decimals will more often result in such an issue.

This issue is only rated as medium severity because the buyer can determine a minAmountOut parameter. The problem is however the auction is a fixed price auction and the buyer already knows the price and the amount he provides, which gives him exactly the fixed output amount. Therefore, there is usually absolutely no slippage necessity to be set by the buyer and lazy buyers might just set this to zero.

## Impact
IMPACT:

a) Loss of funds for buyer


## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L128
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L135


## Tool used

Manual Review

## Recommendation
Consider simply switching to a uint256 approach, this should be adapted in the overall architecture. The only important thing (as far as I have observed) is to make sure the heap mechanism does not overflow when calculating the relative values:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L114](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L114)

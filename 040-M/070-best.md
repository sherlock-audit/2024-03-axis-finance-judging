Orbiting Opaque Kestrel

medium

# Marginal price auction can be spoiled if a maliciously crafted bid is placed

## Summary
If such a bid in an auction is placed that has a price below the lot's minimum price set by the seller, the whole auction will be spoiled and all other bids will only be refundable and not filled.

## Vulnerability Detail
When the very first bid (by size where size is `amountIn` * `amountOut`) in an auction has a price below the minimum one set by the seller of the auction and is for such amount that doesn't satisfy the auction's minimum fill amount, the settlement of the auction will set the marginal price to `type(uint96).max` and will not fill any of the bids. Since bids are sorted by the aforementioned "size" (`amountIn` * `amountOut`) in the decrypted bids queue, the malicious bid can be placed at whichever point in time before the lot concludes and it'll still render the auction pointless. 

Let's suppose a seller creates an auction with the following parameters:
- base and quote tokens both have 18 decimals
- capacity          = 100_000e18
- minPrice          = 1e14 (0.0001 ether)
- minFillPercent    = 50_000 (50%)
- *minFilled*         = 50_000e18 (capacity * minFillPercent / 100_000)
- minBidPercent     = 100 (0.1%)
- *minBidSize*        = 100e18 (capacity * minBidPercent / 100_000)
- minAmount(`_bid()`) = 1e16 (minBidSize * minPrice / 10\*\*18) 

A malicious user can craft such a bid that gets decrypted and included in the `decryptedBids` queue and consecutively spoils the `settle()` function. One of the many possible bids that this user can forge can have the following parameters:
- amount    = 1e16
- amountOut = 100_000e18 (this can be bumped up to `type(uint96).max`)
- *price*     = 1e11 (user can craft an even lower price since they control `amountOut`)

If we follow the execution, starting from placing the bid:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L228-L267

Given a valid bid public key is provided, the only other check we need to satisfy in order to have the bid placed is:
```solidity
        // Amount must be at least the minimum bid size at the minimum price
        uint256 minAmount = Math.mulDivDown(
            uint256(auctionData[lotId_].minBidSize),
            uint256(auctionData[lotId_].minPrice),
            10 ** lotData[lotId_].baseTokenDecimals
        );
        if (amount_ < minAmount) revert Auction_AmountLessThanMinimum();
```
As outlined earlier, `minAmount` would evaluate to `1e16`, which is just the same as our bid `amount`, so the `if` clause above does not enter.

Other bids pile in as well. Then the auction concludes and it's time to decrypt it.
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L459-L522

There are a few checks that the malicious bid has to satisfy. First of all is the check that the `amountOut` of the bid is less than `type(uint96).max`, which is an easy task as our `amountOut` is set to 100_000e18.
```solidity
            // Decrypt the bid
            uint96 amountOut;
            {
                uint256 result = _decrypt(lotId_, bidId, lotBidData.privateKey);

                // Only set the amount out if it is less than or equal to the maximum value of a uint96
                if (result <= type(uint96).max) {
                    amountOut = uint96(result);
                }
            }
```

The next two checks we have to satisfy are that the decoded amount out is `>=` than the auction's minimum bid size and that the bid's price is < `type(uint96).max`:
```solidity
            // Only store the decrypt if the amount out is greater than or equal to the minimum bid size
            if (amountOut > 0 && amountOut >= minBidSize) {
                // Only store the decrypt if the price does not overflow
                // We don't need to check for a zero bid price, because the smallest possible bid price is 1, due to the use of mulDivUp
                // 1 * 10^6 / type(uint96).max = 1
                if (
                    Math.mulDivUp(
                        uint256(bidData.amount),
                        10 ** lotData[lotId_].baseTokenDecimals,
                        uint256(amountOut)
                    ) < type(uint96).max
                ) {
                    // Store the decrypt in the sorted bid queue and set the min amount out on the bid
                    decryptedBids[lotId_].insert(bidId, bidData.amount, amountOut);
                    bidData.minAmountOut = amountOut;
                }
            }
```

The auction's `minBidSize` is 100_000e18, so `amountOut` passes both checks in the first `if` statement. The calculation in the inner `if` statement evaluates to 1e11, as intended. The bid is then recorded in `decryptedBids` and its `minAmountOut` is set to 100_000e18.

After decrypting the bids the next step is to settle the auction. In the very beginning of `settle()`, `_getLotMarginalPrice()` is invoked to get the marginal price result from iterating over the bids.
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L595-L651

It starts iterating over the decrypted bids queue and the first in the queue is the malicious one. Since it's price is below the `minPrice` of the auction, the first `if` statement in the loop is entered:
```solidity
                if (price < lotAuctionData.minPrice) {
                    // We know that the lastPrice was not sufficient to fill capacity or the loop would have exited
                    // We check if minimum price can result in a fill. If so, find the exact marginal price between last price and minimum price
                    // If not, we set the marginal price to the minimum price. Whether the capacity filled meets the minimum filled will be checked later in the settlement process.
                    if (
                        lotAuctionData.minPrice == 0
                            || Math.mulDivDown(result.totalAmountIn, baseScale, lotAuctionData.minPrice)
                                >= capacity
                    ) {
                        result.marginalPrice =
                            uint96(Math.mulDivUp(result.totalAmountIn, baseScale, capacity));
                    } else {
                        result.marginalPrice = lotAuctionData.minPrice; // note this cannot be zero since it is checked above
                    }

                    // If the marginal price is re-calculated and is the same as the previous, we need to set the marginal bid id, otherwise the previous bid will not be able to claim.
                    if (lastPrice == result.marginalPrice) {
                        result.marginalBidId = lastBidId;
                    }

                    // Update capacity expended with the new marginal price
                    result.capacityExpended = Math.mulDivDown(
                        result.totalAmountIn, baseScale, uint256(result.marginalPrice)
                    );
                    // marginal bid id can be zero, there are no bids at the marginal price

                    // Exit the outer loop
                    break;
                }
```

Then `minPrice` of auction is checked if it's zero - it's not and then it's checked if the total amount in up until now can result in a fill at the lot minimum price (`lotAuctionData.minPrice`). `result` is the return value of the `_getLotMarginalPrice()` and when the loop spins up for the first time, it's values are zeros. This results in essentially checking `0 >= 100_000e18 (capacity)` and entering the `else` block which sets `result.marginalPrice` to `lotAuctionData.minPrice` which is `1e14`.
The next `if` statement that sets `result.marginalBidId` is skipped (and irrelevant in our case anyways) and finally `result.capacityExpanded` is set to `0` (as `result.totalAmountIn` is 0 and 0 * 1e18 / 1e14 = 0). The loop breaks and the function returns the marginal price result.

Execution is then handled back to `_settle()` which empties the `decryptedBids` queue and proceeds to check if the lot's expanded capacity covers the minimum fill amount set by the seller:
```solidity
        if (
            result.capacityExpended >= auctionData[lotId_].minFilled
                && result.marginalPrice >= lotAuctionData.minPrice
        ) {
		    // Not entered
		    // ...
		    // code skipped for brevity
		    //
        } else {
            // Auction cannot be settled if we reach this point
            // Marginal price is set as the max uint96 for the auction so the system knows all bids should be refunded
            auctionData[lotId_].marginalPrice = type(uint96).max;

            // totalIn and totalOut are not set since the auction does not clear
        }
```
which it does **not** and the `else` block is entered, setting `auctionData[lotId_].marginPrice` to `type(uint96).max` essentially signalling that all bids from this auction must be refunded.

## Impact
Potentially any auction is susceptible to malicious bids like this one as the maximum amount out value a bid can have is `type(uint96).max` which in most cases will be enough to result in a low enough `price` that's smaller than most of the reasonable `minPrice`s auctions can have. As a result any other bids in this auction following the malicious one will be pointless and are doomed to never be filled. They remain claimable but at the end of the day all that have happened is that gas was wasted money on.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L228-L267
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L459-L522
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L595-L651

## Tool used
Manual Review
Foundry Forge
## Recommendation
I do think that the best solution here would be to not immediately stop iterating over the bids queue as it's too radical of a measure. Rather iterate over the first 10-20-30% of bids, calculate the marginal price from them and then allow for such an early break. Or introduce an auction setting where the seller specifies a tolerance for a bid's price deviation from the lot's minimum price. If the bid doesn't meet the threshold - ignore it, if it does - include it.

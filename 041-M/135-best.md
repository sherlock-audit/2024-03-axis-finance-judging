Orbiting Opaque Kestrel

medium

# If a bid for the whole capacity is placed it won't be filled and auction will settle pointless

## Summary
If a bid for the whole capacity is placed in a marginal price auction, and it is the only bid the auction will be settled without filling the bid.
## Vulnerability Detail
Let's suppose a seller creates an auction with the following parameters:
- base and quote tokens both have 18 decimals
- capacity          = 100_000e18
- minPrice          = 1e14 (0.0001 ether)
- minFillPercent    = 50_000 (50%)
- *minFilled*         = 50_000e18 (capacity * minFillPercent / 100_000)
- minBidPercent     = 100 (0.1%)
- *minBidSize*        = 100e18 (capacity * minBidPercent / 100_000)
- minAmount(`_bid()`) = 1e16 (minBidSize * minPrice / 10\*\*18) 

A bidder comes and wants to buy the whole capacity at the price the Seller has specified, so they place a bid with the following specs:
- amount    = 100_000e18
- amountOut = 1e9
- *price*     = 1e14 (same as minPrice)

The lot concludes, this is the only bidder in the auction and the bids are then decrypted and `settle()` is called. In the very beginning, `_getLotMarginalPrice()` is invoked to get the marginal price result from iterating over the bids. It starts iterating over the decrypted bids queue and for the only bid in there it'd enter the second `if` statement in the loop as the expanded capacity just from that bid will be equal to the capacity of the auction itself.
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L656-L673
```solidity
                result.capacityExpended = Math.mulDivDown(result.totalAmountIn, baseScale, price);
                if (result.capacityExpended >= capacity) {
                    result.marginalPrice =
                        uint96(Math.mulDivUp(result.totalAmountIn, baseScale, capacity));

                    // If the marginal price is re-calculated and is the same as the previous, we need to set the marginal bid id, otherwise the previous bid will not be able to claim.
                    if (lastPrice == result.marginalPrice) {
                        result.marginalBidId = lastBidId;
                    } else {
                        result.marginalBidId = uint64(0); // we set this to zero so that any bids at the current price are not considered in the case that capacityExpended == capacity
                    }

                    // Calculate the capacity expended in the same way as before, instead of setting it to `capacity`
                    // This will normally equal `capacity`, except when rounding would cause the the capacity expended to be slightly less than `capacity`
                    result.capacityExpended =
                        Math.mulDivDown(result.totalAmountIn, baseScale, result.marginalPrice); // updated based on the marginal price
                    break;
                }
```

The function then proceeds to set `result.marginalBid` and `result.capacityExpanded` but `result` is the return value of the `_getLotMarginalPrice()` and when the loop spins up for the first time, it's values are zeros meaning both calculations involving `result.totalAmountIn` will result in 0 as we are multiplying by 0.

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
Honest bidders that want to buy a lot in its full capacity at a fair price will not be able to do so and they'll end up only wasting money on gas.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L656-L673

## Tool used
Manual Review
## Recommendation
In the case of calculating `capacityExpanded`, add the bid's total amount towards `result.totalAmountIn` so it doesn't calculate stuff multiplying by 0 and revise the order of actions/calculations in the `_getLotMarginalPrice()` function in general.

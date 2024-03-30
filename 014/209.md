Special Black Spider

high

# Silent overflow within _getLotMarginalPrice will falsify auction outcome

## Summary
Silent overflow within _getLotMarginalPrice will falsify auction outcome

## Vulnerability Detail
Contract: EMPAM.sol

The _getLotMarginalPrice function is responsible for calculating the price at which the auction is cleared. The marginalPrice will then later be used to determine if a bid is valid or not. This can be seen within the _claimBid function:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L348](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L348)

The problem has its root-cause within the calculation of the marginalPrice, we can see this calculation on several occasions within the _getLotMarginalPrice function and is *unsafely* casted to uint96:

1. [https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L632](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L632)

2. [https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L658](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L658)

3. [https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L708](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L708)

As we can see, the calculation will always be:

totalAmountIn * baseScale / capacity


Now we need to observe that the input amounts are always uint96, however, the totalAmountIn is uint256:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L95](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L95)

and it is increased each bid by the input amount:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L680](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L680)

This means, totalAmountIn can be very well above uint96. Therefore our calculation:

totalAmountIn * baseScale / capacity

is prone to be larger than uint96. If we now assume a baseScale of 1e18, it of course depends on the size of totalAmountIn in reference to the capacity. 

Now it is clear that this is an edge-case which was probably not sufficiently tested. For example, an overflow would happen for the following scenario:

totalAmountIn = uint96+2 (79228162514264337593543950336)
baseScale = 1e18
capacity = 1e18

The result for this calculation would be a marginalPrice of 1

Now we need to understand that there are multiple different scenarios depending on the amounts which will then influence the marginalPrice, it could also very well be that totalAmountIn = uint96 + 1e18, which would bring the marginalPrice to 1e18.

I acknowledge that this example is just one of many, at the end of the day it depends on the capacity, desired price and amounts. If for example a user tries to auction a very expensive token such as WBTC and expects as quoteToken a very cheap token, the capacity and input amounts could very well result in such a scenario. I could now spend more time on crafting a proper PoC with feasible tokens and amounts but I am of the opinion that my time is well spent on other parts of the codebase, especially because the PoC should by now be crystal clear and everyone should understand it.

So there are two scenarios which can result due to an incorrect marginalPrice:

a) marginalPrice is below minPrice:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L828](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L828)

Auction is not settled here.

b) marginalPrice is above minPrice

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L791](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L791)

Auction is settled here, this will influence bid claiming negatively by allowing the early claimers to claim unlawfully (due to the overflow of the marginalPrice, it is lower than expected and therefore invalid bids will become valid). This means that funds can be stolen by invalid bids and valid bids will eventually not receive any payout. An additional problem is the unchecked block in the funding decrease:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L437](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L437)

This can spin so far that all funds of the quoteToken in the contract are drained, even those which were allocated to other bids.


It might very well be that there are several other side-effects due to this unsafe casting. However, at this point, also considering time constraints, it is clear that this will effectively break the whole auction on multiple occasions and can result in a loss of funds, which makes it a valid high severity issue

## Impact
IMPACT: 

a) Falsification of auction outcome
b) Drainage of funds


## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L348
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L632
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L658
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L708
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L95
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L680
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L828
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L791
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L437



## Tool used

Manual Review

## Recommendation

Consider simply not casting it as uint96, rather it should be casted at uint256. Potential side-effects in the call flow must be considered as well.


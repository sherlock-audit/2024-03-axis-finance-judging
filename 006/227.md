Funny Punch Snake

high

# `curate()` calculates `curatorFeePayout` incorrectly when capacity is in quote tokens.

## Summary
An auction lot can have a curator, whose goal is to promote an auction lot or simply provide credibility. For that service, a curator is eligible for a `curatorFeePayout`. The problem is that when the capacity of the lot is in quoteTokens, the `curatorFeePayout` is calculated incorrectly.
## Vulnerability Detail
When creating an auction, the lot creator can choose to have capacity either in base or quote tokens.
[Auction.sol#L320-L321](https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L320-L321)
```solidity
lot.capacityInQuote = params_.capacityInQuote;
lot.capacity = params_.capacity;
```

When curator accepts a curation proposal, he calls `curate()` function of the AuctionHouse, where `curatorFeePayout` is calculated.
[AuctionHouse.sol#L655-L659](https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L655-L659)
```solidity
uint96 curatorFeePayout = uint96(
    _calculatePayoutFees(
        feeData.curated,
        feeData.curatorFee,
        module.remainingCapacity(lotId_)
    )
);
```
As can be seen, it depends on the capacity of the lot, which as we already know, can be in quoteTokens. Despite the fact, that capacity can be in quoteTokens, `curate()` always tries to send the `curatorFeePayout` in baseTokens.
[AuctionHouse.sol#L683-L685](https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L683-L685)
```solidity
Transfer.transferFrom(
    routing.baseToken,
    routing.seller,
    address(this),
    curatorFeePayout,
    false
);
```
## Impact
This can lead to a loss of funds for either of the parties as the underlying value of both tokens is not the same, not to mention that both tokens may have different decimals of precision.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L634-L699
## Tool used

Manual Review

## Recommendation
`curate()` should check if capacity is in quoteTokens and send quoteTokens if so.
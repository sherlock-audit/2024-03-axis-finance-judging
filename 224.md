Macho Sandstone Platypus

medium

# No setter for minAuctionDuration

## Summary
No setter for minAuctionDuration

## Vulnerability Detail

The `minAuctionDuration` parameters lacks setter functions

```solidity
    uint48 public minAuctionDuration;
```
```solidity
    constructor(address auctionHouse_) AuctionModule(auctionHouse_) {
        // Set the minimum auction duration to 1 day initially
        minAuctionDuration = 1 days;
    }
```

## Impact

minAuctionDuration cannot be updated and hence the protocol is stuck with the same duration unless an upgrade is made

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/FPAM.sol#L47-L50

## Tool used

Manual Review

## Recommendation

Add a function to set `minAuctionDuration`
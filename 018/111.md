Orbiting Opaque Kestrel

medium

# Curator can inflate fee right before accepting an invitation for an auction

## Summary
A curator can inflate their fee to a higher one than promoted to an auction's seller right before accepting the invitation to curate the auction and thus charging the seller more and incurring unexpected costs for them.
## Vulnerability Detail
> In the contest's README there is no mention of the Curator role, so it is to be considered RESTRICTED.

When a seller is creating an auction, they specify the address of the curator of their auction once and for all.

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L215C1-L220C10
```solidity
        // Store curation information
        {
            FeeData storage fees = lotFees[lotId];
            fees.curator = routing_.curator;
            fees.curated = false;
        }
```

From then on the only possible action involving the curator is for them to accept the invitation. They can neither be replaced, nor removed. The curator on the other side is allowed to accept the invitation at any point in time before the lot ends and the curator fee for the lot is set to the current fee the curator has set for themselves at the time of acceptance.

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L646C1-L652C101
```solidity
        if (feeData.curated || module.hasEnded(lotId_) == true) revert InvalidState();

        Routing storage routing = lotRouting[lotId_];

        // Set the curator as approved
        feeData.curated = true;
        feeData.curatorFee = fees[keycodeFromVeecode(routing.auctionReference)].curator[msg.sender];
```

The curator sets their fee by calling `setCuratorFee()` on the auction house:

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/FeeManager.sol#L110C1-L116C6
```solidity
    function setCuratorFee(Keycode auctionType_, uint48 fee_) external {
        // Check that the fee is less than the maximum
        if (fee_ > fees[auctionType_].maxCuratorFee) revert InvalidFee();

        // Set the fee for the sender
        fees[auctionType_].curator[msg.sender] = fee_;
    }
```

As can be seen, the only thing mitigating the damage the curator can do is the `maxCuratorFee` that the protocol itself sets, but that wouldn't do much as this is a global limit and some curators might charge more, others less and if the protocol is to welcome the ones that charge more that would mean they have to set a higher `maxCuratorFee` that curators with bad intentions can take advantage of.

## Impact
A proposed curator can set their fee to the maximum right before accepting an invitation to curate an auction and thus incur an extra expense for the lot seller. The seller themselves have almost non-existent means to tackle a malicious curator as their only option is to cancel the auction they've set this curator to and this is only an option before the lot has started. After the lot starts the curator is guaranteed to get their fees. For fixed price auctions (atomic) that is every time a buyer calls `purchase()` and for marginal price auctions (batch) that is at `settle()`ment of the auction.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L634-L699

## Tool used
Manual Review

## Recommendation
The safety measure I'd recommend for this issue are:
- Allow the curator to only accept their invitation before the lot has started
- Allow the seller of a lot to remove a curator as such from the lot
- Record the curator fee for the lot at the time of creating the lot

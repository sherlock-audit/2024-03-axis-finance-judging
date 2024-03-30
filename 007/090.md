Sharp Hazel Barracuda

medium

# If pfBidder gets blacklisted the settlement process would be broken and every other bidders and the seller would lose their funds

## Summary
During batch auction settlement, the bidder whos bid was partially filled gets the refund amount in quote tokens and his payout in base immediately. In case if quote or base is a token with blacklisted functionality (e.g. USDC) and bidder's account gets blacklisted after the bid was submitted, the settlement would be bricked and all bidders and the seller would lose their tokens/proceeds.
## Vulnerability Detail
In the `AuctionHouse.settlement()` function there is a check if the bid was partially filled, in which case the function handles refund and payout immediately:
```solidity
            // Check if there was a partial fill and handle the payout + refund
            if (settlement.pfBidder != address(0)) {
                // Allocate quote and protocol fees for bid
                _allocateQuoteFees(
                    feeData.protocolFee,
                    feeData.referrerFee,
                    settlement.pfReferrer,
                    routing.seller,
                    routing.quoteToken,
                    // Reconstruct bid amount from the settlement price and the amount out
                    uint96(
                        Math.mulDivDown(
                            settlement.pfPayout, settlement.totalIn, settlement.totalOut
                        )
                    )
                );

                // Reduce funding by the payout amount
                unchecked {
                    routing.funding -= uint96(settlement.pfPayout);
                }

                // Send refund and payout to the bidder
                //@audit if pfBidder gets blacklisted the settlement is broken
                Transfer.transfer(
                    routing.quoteToken, settlement.pfBidder, settlement.pfRefund, false
                );

                _sendPayout(settlement.pfBidder, settlement.pfPayout, routing, auctionOutput);
            }
```
If `pfBidder` gets blacklisted after he submitted his bid, the call to `settle()` would revert. There is no way for other bidders to get a refund for the auction since settlement can only happen after auction conclusion but the `refundBid()` function needs to be called before the conclusion:
```solidity
    function settle(uint96 lotId_)
        external
        virtual
        override
        onlyInternal
        returns (Settlement memory settlement, bytes memory auctionOutput)
    {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfLotActive(lotId_); //@audit
        _revertIfLotSettled(lotId_);
        
       ...
}
```
```solidity
    function refundBid(
        uint96 lotId_,
        uint64 bidId_,
        address caller_
    ) external override onlyInternal returns (uint96 refund) {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfBidInvalid(lotId_, bidId_);
        _revertIfNotBidOwner(lotId_, bidId_, caller_);
        _revertIfBidClaimed(lotId_, bidId_);
        _revertIfLotConcluded(lotId_); //@audit

        // Call implementation-specific logic
        return _refundBid(lotId_, bidId_, caller_);
    }
```
Also, the `claimBids` function would also revert since the lot wasn't settled and the seller wouldn't be able to get his prefunding back since he can neither `cancel()` the lot nor `claimProceeds()`.
## Impact
Loss of funds
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L503-L529
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L501-L516
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L589-L600
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L733-L741
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L885-L891
## Tool used

Manual Review

## Recommendation
Separate the payout and refunding logic for pfBidder from the settlement process.
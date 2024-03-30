Macho Sandstone Platypus

medium

# User's can be grieved by not submitting the private key

## Summary
User's can be grieved by not submitting the private key

## Vulnerability Detail

Bids cannot be refunded once the auction concludes. And bids cannot be claimed until the auction has been settled. Similarly a EMPAM auction cannot be cancelled once started. 

```solidity
    function claimBids(
        uint96 lotId_,
        uint64[] calldata bidIds_
    )
        external
        override
        onlyInternal
        returns (BidClaim[] memory bidClaims, bytes memory auctionOutput)
    {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotNotSettled(lotId_);
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
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _cancelAuction(uint96 lotId_) internal override {
        // Validation
        // Batch auctions cannot be cancelled once started, otherwise the seller could cancel the auction after bids have been submitted
        _revertIfLotActive(lotId_);
```

```solidity
    function cancelAuction(uint96 lotId_) external override onlyInternal {
        // Validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _settle(uint96 lotId_)
        internal
        override
        returns (Settlement memory settlement_, bytes memory auctionOutput_)
    {
        // Settle the auction
        // Check that auction is in the right state for settlement
        if (auctionData[lotId_].status != Auction.Status.Decrypted) {
            revert Auction_WrongState(lotId_);
        }
```

For EMPAM auctions, the private key associated with the auction has to be submitted before the auction can be settled. In auctions where the private key is held by the seller, they can grief the bidder's or in cases where a key management solution is used, both seller and bidder's can be griefed by not submitting the private key.

## Impact

User's will not be able to claim their assets in case the private key holder doesn't submit the key for decryption 

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L747-L756

## Tool used

Manual Review

## Recommendation

Acknowledge the risk involved for the seller and bidder
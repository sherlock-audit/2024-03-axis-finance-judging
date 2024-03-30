Zany Iron Beaver

high

# Auction creators have the ability to lock bidders' funds.

## Summary
`Auction creators` have the ability to cancel an `auction` before it starts.
However, once the `auction` begins, they should not be allowed to cancel it.
During the `auction`, `bidders` can place `bids` and send `quote` tokens to the `auction house`.
After the `auction` concludes, `bidders` can either receive `base` tokens or retrieve their `quote` tokens.
Unfortunately, `batch auction creators` can cancel an `auction` when it ends.
This means that `auction creators` can cancel their `auctions` if they anticipate `losses`.
This should not be allowed.
The significant risk is that `bidders' funds` could become locked in the `auction house`.
## Vulnerability Detail
`Auction creators` can not cancel an `auction` once it concludes.
```solidity
function cancelAuction(uint96 lotId_) external override onlyInternal {
    _revertIfLotConcluded(lotId_);
}
```
They also can not cancel it while it is active.
```solidity
function _cancelAuction(uint96 lotId_) internal override {
    _revertIfLotActive(lotId_);

    auctionData[lotId_].status = Auction.Status.Claimed;
}
```
When the `block.timestamp` aligns with the `conclusion` time of the `auction`, we can bypass these checks.
```solidity
function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
    if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
        revert Auction_MarketNotActive(lotId_);
    }

    if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
}
function _revertIfLotActive(uint96 lotId_) internal view override {
    if (
        auctionData[lotId_].status == Auction.Status.Created
            && lotData[lotId_].start <= block.timestamp
            && lotData[lotId_].conclusion > block.timestamp
    ) revert Auction_WrongState(lotId_);
}
```
So `Auction creators` can cancel an `auction` when it concludes.
Then the `capacity` becomes `0` and the `auction status` transitions to `Claimed`.

`Bidders` can not `refund` their `bids`.
```solidity
function refundBid(
    uint96 lotId_,
    uint64 bidId_,
    address caller_
) external override onlyInternal returns (uint96 refund) {
    _revertIfLotConcluded(lotId_);
}
 function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
    if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
}
```
The only way for `bidders` to reclaim their tokens is by calling the `claimBids` function.
However, `bidders` can only claim `bids` when the `auction status` is `Settled`.
```solidity
function claimBids(
    uint96 lotId_,
    uint64[] calldata bidIds_
) {
    _revertIfLotNotSettled(lotId_);
}
```
To `settle` the `auction`, the `auction status` should be `Decrypted`.
This requires submitting the `private key`.
The `auction creator` can not submit the `private key` or submit it without decrypting any `bids` by calling `submitPrivateKey(lotId, privateKey, 0)`.
Then nobody can decrypt the `bids` using the `decryptAndSortBids` function which always reverts.
```solidity
function decryptAndSortBids(uint96 lotId_, uint64 num_) external {
    if (
        auctionData[lotId_].status != Auction.Status.Created     // @audit, here
            || auctionData[lotId_].privateKey == 0
    ) {
        revert Auction_WrongState(lotId_);
    }

    _decryptAndSortBids(lotId_, num_);
}
```
As a result, the `auction status` remains unchanged, preventing it from transitioning to `Settled`.
This leaves the `bidders'` `quote` tokens locked in the `auction house`.

Please add below test to the `test/modules/Auction/cancel.t.sol`.
```solidity
function test_cancel() external whenLotIsCreated {
    Auction.Lot memory lot = _mockAuctionModule.getLot(_lotId);

    console2.log("lot.conclusion before   ==> ", lot.conclusion);
    console2.log("block.timestamp before  ==> ", block.timestamp);
    console2.log("isLive                  ==> ", _mockAuctionModule.isLive(_lotId));

    vm.warp(lot.conclusion - block.timestamp + 1);
    console2.log("lot.conclusion after    ==> ", lot.conclusion);
    console2.log("block.timestamp after   ==> ", block.timestamp);
    console2.log("isLive                  ==> ", _mockAuctionModule.isLive(_lotId));

    vm.prank(address(_auctionHouse));
    _mockAuctionModule.cancelAuction(_lotId);
}
```
The log is
```solidity
lot.conclusion before   ==>  86401
block.timestamp before  ==>  1
isLive                  ==>  true
lot.conclusion after    ==>  86401
block.timestamp after   ==>  86401
isLive                  ==>  false
```
## Impact
Users' funds can be locked.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L354
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L204
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L512
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L556
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L449
## Tool used

Manual Review

## Recommendation
```solidity
function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
-     if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
+     if (lotData[lotId_].conclusion <= uint48(block.timestamp)) {
        revert Auction_MarketNotActive(lotId_);
    }

    // Capacity is sold-out, or cancelled
    if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
}
```
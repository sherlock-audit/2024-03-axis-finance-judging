Zany Iron Beaver

high

# Bidders can not claim their bids if the auction creator claims the proceeds.

## Summary
Before the `batch auction` begins, the `auction creator` should `prefund` `base` tokens to the `auction house`.
During the `auction`, `bidders` transfer `quote` tokens to the `auction house`.
After the `auction` settles,
- `Bidders` can claim their `bids` and either to receive `base` tokens or `retrieve` their `quote` tokens.
- The `auction creator` can receive the `quote` tokens and retrieve the remaining `base` tokens.
- There is no specific order for these two operations.

However, if the `auction creator` claims the `proceeds`, `bidders` can not claim their `bids` anymore.
Consequently, their `funds` will remain locked in the `auction house`.
## Vulnerability Detail
When the `auction creator` claims `Proceeds`, the `auction status` changes to `Claimed`.
```solidity
function _claimProceeds(uint96 lotId_)
    internal
    override
    returns (uint96 purchased, uint96 sold, uint96 payoutSent)
{
    auctionData[lotId_].status = Auction.Status.Claimed;
}
```
Once the `auction status` has transitioned to `Claimed`, there is indeed no way to change it back to `Settled`.

However, `bidders` can only claim their `bids` when the `auction status` is `Settled`.
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
    _revertIfLotInvalid(lotId_);
    _revertIfLotNotSettled(lotId_);   // @audit, here

    return _claimBids(lotId_, bidIds_);
}
```

Please add below test to the `test/modules/auctions/claimBids.t.sol`.
```solidity
function test_claimProceeds_before_claimBids()
    external
    givenLotIsCreated
    givenLotHasStarted
    givenBidIsCreated(_BID_AMOUNT_UNSUCCESSFUL, _BID_AMOUNT_OUT_UNSUCCESSFUL)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenLotHasConcluded
    givenPrivateKeyIsSubmitted
    givenLotIsDecrypted
    givenLotIsSettled
{
    uint64 bidId = 1;

    uint64[] memory bidIds = new uint64[](1);
    bidIds[0] = bidId;

    // Call the function
    vm.prank(address(_auctionHouse));
    _module.claimProceeds(_lotId);


    bytes memory err = abi.encodeWithSelector(EncryptedMarginalPriceAuctionModule.Auction_WrongState.selector, _lotId);
    vm.expectRevert(err);
    vm.prank(address(_auctionHouse));
    _module.claimBids(_lotId, bidIds);
}
```
## Impact
Users' funds could be locked.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L846
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L556
## Tool used

Manual Review

## Recommendation
Allow `bidders` to claim their `bids` even when the `auction status` is `Claimed`.
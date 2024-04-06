Zany Iron Beaver

high

# Bidders' funds may become locked due to inconsistent price order checks in MaxPriorityQueue and the _claimBid function.

## Summary
In the `MaxPriorityQueue`, `bids` are ordered by decreasing `price`.
We calculate the `marginal price`, `marginal bid ID`, and determine the `auction winners`.
When a `bidder` wants to claim, we verify that the `bid price` of this `bidder` exceeds the `marginal price`.
However, there's minor inconsistency: certain `bids` may have `marginal price` and a smaller `bid ID` than `marginal bid ID` and they are not actually `winners`.
As a result, the `auction winners` and these `bidders` can receive `base` tokens.
However, there is a finite supply of `base` tokens for `auction winners`.
Early `bidders` who claim can receive `base` tokens, but the last `bidders` can not.
## Vulnerability Detail
The comparison for the order of `bids` in the `MaxPriorityQueue` is as follow:
if `q1 * b2 < q2 * b1` then `bid (q2, b2)` takes precedence over `bid (q1, b1)`.
```solidity
function _isLess(Queue storage self, uint256 i, uint256 j) private view returns (bool) {
    uint64 iId = self.bidIdList[i];
    uint64 jId = self.bidIdList[j];
    Bid memory bidI = self.idToBidMap[iId];
    Bid memory bidJ = self.idToBidMap[jId];
    uint256 relI = uint256(bidI.amountIn) * uint256(bidJ.minAmountOut);
    uint256 relJ = uint256(bidJ.amountIn) * uint256(bidI.minAmountOut);
    if (relI == relJ) {
        return iId > jId;
    }
    return relI < relJ;
}
```
And in the `_calimBid` function, the `price` is checked directly as follow:
if `q * 10 ** baseDecimal / b >= marginal price`, then this `bid` can be claimed.
```solidity
function _claimBid(
    uint96 lotId_,
    uint64 bidId_
) internal returns (BidClaim memory bidClaim, bytes memory auctionOutput_) {
    uint96 price = uint96(
        bidData.minAmountOut == 0
            ? 0 // TODO technically minAmountOut == 0 should be an infinite price, but need to check that later. Need to be careful we don't introduce a way to claim a bid when we set marginalPrice to type(uint96).max when it cannot be settled.
            : Math.mulDivUp(uint256(bidData.amount), baseScale, uint256(bidData.minAmountOut))
    );
    uint96 marginalPrice = auctionData[lotId_].marginalPrice;
    if (
        price > marginalPrice
            || (price == marginalPrice && bidId_ <= auctionData[lotId_].marginalBidId)
    ) { }
}
```
The issue is that a `bid` with the `marginal price` might being placed after `marginal bid` in the `MaxPriorityQueue` due to rounding.
```solidity
q1 * b2 < q2 * b1, but mulDivUp(q1, 10 ** baseDecimal, b1) = mulDivUp(q2, 10 ** baseDecimal, b2)
```

Let me take an example.
The `capacity` is `10e18` and there are `6 bids` (`(4e18 + 1, 2e18)` for first `bidder`, `(4e18 + 2, 2e18)` for the other `bidders`.
The order in the `MaxPriorityQueue` is `(2, 3, 4, 5, 6, 1)`.
The `marginal bid ID` is `6`.
The `marginal price` is `2e18 + 1`.
The `auction winners` are `(2, 3, 4, 5, 6)`.
However, `bidder 1` can also claim because it's `price` matches the `marginal price` and it has the smallest `bid ID`.
There are only `10e18` `base` tokens, but all `6 bidders` require `2e18` `base` tokens.
As a result, at least one `bidder` won't be able to claim `base` tokens, and his `quote` tokens will remain locked in the `auction house`.

The Log is
```solidity
marginal price     ==>   2000000000000000001
marginal bid id    ==>   6

paid to bid  1       ==>   4000000000000000001
payout to bid  1     ==>   1999999999999999999
*****
paid to bid  2       ==>   4000000000000000002
payout to bid  2     ==>   2000000000000000000
*****
paid to bid  3       ==>   4000000000000000002
payout to bid  3     ==>   2000000000000000000
*****
paid to bid  4       ==>   4000000000000000002
payout to bid  4     ==>   2000000000000000000
*****
paid to bid  5       ==>   4000000000000000002
payout to bid  5     ==>   2000000000000000000
*****
paid to bid  6       ==>   4000000000000000002
payout to bid  6     ==>   2000000000000000000
```
Please add below test to the `test/modules/auctions/EMPA/claimBids.t.sol`
```solidity
function test_claim_nonClaimable_bid()
    external
    givenLotIsCreated
    givenLotHasStarted
    givenBidIsCreated(4e18 + 1, 2e18)           // bidId = 1
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 2
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 3
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 4
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 5
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 6
    givenLotHasConcluded
    givenPrivateKeyIsSubmitted
    givenLotIsDecrypted
    givenLotIsSettled
{
    EncryptedMarginalPriceAuctionModule.AuctionData memory auctionData = _getAuctionData(_lotId);

    console2.log('marginal price     ==>  ', auctionData.marginalPrice);
    console2.log('marginal bid id    ==>  ', auctionData.marginalBidId);
    console2.log('');

    for (uint64 i; i < 6; i ++) {
        uint64[] memory bidIds = new uint64[](1);
        bidIds[0] = i + 1;
        vm.prank(address(_auctionHouse));
        (Auction.BidClaim[] memory bidClaims,) = _module.claimBids(_lotId, bidIds);
        Auction.BidClaim memory bidClaim = bidClaims[0];
        if (i > 0) {
            console2.log('*****');
        }
        console2.log('paid to bid ', i + 1, '      ==>  ', bidClaim.paid);
        console2.log('payout to bid ', i + 1, '    ==>  ', bidClaim.payout);
    }
}
```
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/lib/MaxPriorityQueue.sol#L109-L120
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L347-L350
## Tool used

Manual Review

## Recommendation
In the `MaxPriorityQueue`, we should check the `price`: `Math.mulDivUp(q, 10 ** baseDecimal, b)`.
Rural Midnight Cottonmouth

high

# On refund bids sorted incorrectly, which break settlement functionality and make some bids, expected to win, lose

## Summary
When we refund a bid, decrypted bids are sorted incorrectly, should be sorted by `bidId` but they are not.


## Vulnerability Detail
It is expected to sort decrypted bids by price (high to low) and then by `bidId` as stated in documentation and just right thing to expect.

> Sort bids by price (high to low), then by order submitted (low to high). In EMPAM, bids are pre-sorted during decryption.

However when we refund a bid, sorting become broken.

Let's reproduce the issue

To get easy access to decrypted bidIds `Queue.bidIdList` from 
```solidity
moonraker/src/lib/MaxPriorityQueue.sol
struct Queue {
    ///@notice array backing priority queue
    uint64[] bidIdList;
    ///@notice total number of bids in queue
    uint64 numBids;
    //@notice map bid ids to bids
    mapping(uint64 => Bid) idToBidMap;
}
```
I added a simple getter to contract `EncryptedMarginalPriceAuctionModule`

```solidity
moonraker/src/modules/auctions/EMPAM.sol
contract EncryptedMarginalPriceAuctionModule is AuctionModule {
    (...)
    function getBidsArr(uint96 lotId) external view returns (uint64[] memory bidIdList) {
        return decryptedBids[lotId].bidIdList;
    }
```

Then add this test next to other tests in `moonraker/test/modules/auctions/EMPA/decryptAndSortBids.t.sol`

```solidity
moonraker/test/modules/auctions/EMPA/decryptAndSortBids.t.sol
//add console2
import {console2} from "forge-std/console2.sol";
(...)
     function test_multipleBidsWithRefund()
        external
        givenMinimumPrice(1)
        givenLotIsCreated
        givenLotHasStarted
        givenBidIsCreated(800, 1e18)
        givenBidIsCreated(700, 1e18)
        givenBidIsCreated(600, 1e18)
        givenBidIsCreated(500, 1e18)
        givenBidIsCreated(400, 1e18)
        givenBidIsCreated(300, 1e18)
        givenBidIsCreated(200, 1e18)
        givenBidIsCreated(100, 1e18)
    {

        vm.prank(address(_auctionHouse));
        uint256 refundAmount = _module.refundBid(_lotId, 3, _BIDDER);//refund bid with bidId == 3

        vm.warp(_start + _DURATION + 1);//skip time to conclude the auction

        _module.submitPrivateKey(_lotId, _AUCTION_PRIVATE_KEY, 7);//decrypt all 7 bids

        uint64[] memory bidIdList = _module.getBidsArr(_lotId);
        for (uint i; i < bidIdList.length; ++i){
            console2.log("bidId: %s", bidIdList[i]);//expected:0 1 2 4 5 6 7 8   actual:0 1 2 6 4 5 8 7
        }
    }
```

In example above if we assume lot capacity is `3e18`, then only bids with `bidId` `1`,`2`,`6` will be able to claim tokens, despite the fact that bids with `bidId` `4` and `5` offer higher price.

## Impact
Can break settlement functionality in a way that marginal price can be completely wrong, and some bids even with higher price being discarded.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L53-L55

## Tool used

Manual Review

## Recommendation
Change the sorting algorithm.
For example what I did to fix it is to compare elements next to each other `arr[k]==arr[k-1]` instead of `arr[k]==arr[k/2]`, it worked. However consider making a better solution, I am no expert in algorithms.
```diff
moonraker/src/lib/MaxPriorityQueue.sol
    function _swim(Queue storage self, uint64 k) private {
-        while (k > 1 && _isLess(self, k / 2, k)) {
-            _exchange(self, k, k / 2);
-            k = k / 2;
+        while (k > 1 && _isLess(self, k - 1, k)) {
+            _exchange(self, k, k - 1);
+            k = k - 1;
        }
    }
```
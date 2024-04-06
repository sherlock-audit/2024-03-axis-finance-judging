Shiny Cinnabar Horse

high

# Upon cancelling a `Batch` Auction, Refunds are never returned to the `bidders`.

## Summary
When a seller creates a `Batch` auction, then later, the seller `cancels` the auctio. `Refunds` are only sent back to
the `Seller`. and if the bidders call `Refund()` the transaction reverts with a  `Auction_MarketNotActive` error.
## Vulnerability Detail

place the following code in moonraker/test/AuctionHouse/bid.t.sol

```javascript
    function test_shouldRefundWhenBatchGetCancelled()
        external
        whenAuctionTypeIsBatch
        whenBatchAuctionModuleIsInstalled
        givenSellerHasBaseTokenBalance(_LOT_CAPACITY)
        givenSellerHasBaseTokenAllowance(_LOT_CAPACITY)
        givenLotIsCreated
        givenLotHasStarted
        givenUserHasQuoteTokenBalance(_BID_AMOUNT)
        givenUserHasQuoteTokenAllowance(_BID_AMOUNT)
    {
        _bidAuctionData = abi.encode("auction data");

        // Check the balance before placing a bid
        uint256 balanceBeforeBid = _quoteToken.balanceOf(_bidder);
        // we create a bid with the bid amount
        _createBid(_BID_AMOUNT, _bidAuctionData);
        // the seller decides to cancel the auction

        vm.prank(_SELLER);
        _auctionHouse.cancel(_lotId, bytes(""));
        vm.stopPrank();

        uint256 balanceAfterCancel = _quoteToken.balanceOf(_bidder);
        uint256 balanceOfBase = _baseToken.balanceOf(_bidder);

        uint256 balanceOfBaseseller = _baseToken.balanceOf(_SELLER);
        // lets check if its ever refunded
       
        console.log(
            balanceBeforeBid,
            "This is The Balance before placing the bid"
        );
        console.log(
            balanceAfterCancel,
            "The money is never returned in Quote after a cancelled auction"
        );
        console.log(balanceOfBase, "or in BaseTokens");

        // The seller receives his 10 eth back
        console.log(
            balanceOfBaseseller,
            "Amount is Refunded back to the seller"
        );
        // If we try to call Refund
        // uncomment this line its going to you an error saying the market is not active when we try to call refund.

        // vm.prank(_bidder);
        // _auctionHouse.refundBid(_lotId, bidId);
        // vm.stopPrank();

    }
```

## Impact 
The `Bidder` will get rekt whenever, a batch auction is cancelled.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L301

## Tool used

Manual Review

## Recommendation
Add functionality in `Auctioneer.sol`, on cancel the amount is sent back to the `Bidders` and `Sellers`succesfully, not just the `Sellers`

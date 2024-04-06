Shiny Cinnabar Horse

high

# Arbritage oppurtunity in `FPAM.sol` because price is fixed.

## Summary
When a seller sets up an `automic` auction, the price is fixed. Meaning that when the base token is volatile and it increases in value. `bidders` can rug pull the seller by buying them at a cheaper price. 

## Vulnerability Detail
- When we run this test `function test_success()` in  test/modules/auctions/FPA/purchase.t.sol.
 we see the purchase power when price is `2e18`.
```bash
        ├─ [0] VM::assertEq(1000000000000000000 [1e18], 1000000000000000000 [1e18], "sold") 
```
- but when price changes on the upside more tokens are sold for the same amount of purchase power
- first change the price from being a constant in test/modules/auctions/FPA/FPAModuleTest.sol
```diff

    //price can never be constant
+    uint96 internal constant _PRICE = 2e18;
-    uint96 internal _PRICE = 2e18;
```
- place this code in test/FPA/purchase.t.sol
```javascript
     function test_Arbitrage_Oppurtunity_WhenHighPrice()
        public
        givenLotIsCreated
        givenLotHasStarted
    {
        _PRICE = 4e18;
        uint96 expectedHighSold = _mulDivDown(
            _scaleQuoteTokenAmount(_PURCHASE_AMOUNT),
            uint96(10) ** _baseTokenDecimals,
            _scaleQuoteTokenAmount(_PRICE)
        );

        // Call the function

        _createPurchase(
            _scaleQuoteTokenAmount(_PURCHASE_AMOUNT),
            _scaleBaseTokenAmount(_PURCHASE_AMOUNT_OUT)
        );

        // Assert the capacity, purchased and sold
        Auction.Lot memory _HighPrice = _getAuctionLot(_lotId);

        uint256 _HighInPriceBase = _HighPrice.capacity;
        uint256 _HighInPriceQuote = _HighPrice.purchased;
        console.log(_HighInPriceBase);
        console.log(_HighInPriceQuote);
        console.log(expectedHighSold, "sold");
    }

```
more coins are sold for the same amount of purchase power
```bash
     [0] console::log(500000000000000000 [5e17], "sold") 
```
## Impact
This will led to `Sellers` to always get rug pulled by `buyers` cause they will wait, for price of the basetoken to increase

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L35
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L36
## Tool used
Manual Review

## Recommendation
The price should not be Fixed. add chanlink priceFeed address or TWAP oracle to always calculate the price of tokens for seller, to remove the arbitrage oppurtunity


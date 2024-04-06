Kind Admiral Crocodile

high

# An atomic auction seller can earn more than intended in an unfair manner, which can also cause loss of funds of other auction sellers, completely wrecking the system.

## Summary
An auction seller can earn more than intended in an unfair manner by intentionally providing a misconfiguration while creating an auction, which can also cause loss of funds of other auction sellers who created auction with same baseToken.

## Vulnerability Detail
While creating an auction using `Auctioneer::auction` function , `Auction::AuctionParams` are being provided:

```javascript
struct AuctionParams {
        uint48 start;
        uint48 duration;
        bool capacityInQuote;
        uint96 capacity;
        bytes implParams;
    }
```

where you can see, `capacity` & `capacityInQuote` are being passed.

Now here a malicious seller intentionally do a misconfiguration & can set the capacity in Quote token, but can set the bool `capacityInQuote` as false. 

Now for explaining how this can benefit seller , I will provide a POC below

## POC
`The coded POC is also being provided for this same situation, and I have taken assumption of paramters on the basis of already existing testcases by protocol team for simplicity`

Lets say for example malicious seller creates an auction , with `quoteToken : ETH` & `baseToken: USDC`, decimals of `quoteToken > baseToken`.
Now while providing capacity, it sets it in ETH(quoteToken) = 10 ETH i.e 10e18.
And set the bool `capacityInQuote` as false. Where in actual he has set the capacity in Quote token only.
The price is 2e18

Now while purchasing using `AuctionHouse::purchase` function :

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L201-L329

where the Auction module's `_purchase` is being called internally :

```javascript
    function purchase(
        uint96 lotId_,
        uint96 amount_,
        bytes calldata auctionData_
    ) external override onlyInternal returns (uint96 payout, bytes memory auctionOutput) {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotInactive(lotId_);

        // Call implementation-specific logic
        (payout, auctionOutput) = _purchase(lotId_, amount_, auctionData_);

        // Update capacity
        Lot storage lot = lotData[lotId_];
        // Revert if the capacity is insufficient
        if (lot.capacityInQuote ? amount_ > lot.capacity : payout > lot.capacity) {
            revert Auction_InsufficientCapacity();
        }
        unchecked {
            lot.capacity -= lot.capacityInQuote ? amount_ : payout;
        }

        // Update the purchased and sold amounts for the lot
        lot.purchased += amount_;
        lot.sold += payout;
    }
```
where you can see , FPA module's `_purchase` function is being called, which calculates the payout in baseToken and is being stored in payout.

After that, the capacity of lot is being updated.

Now here comes the part where seller will get profited heavily.
```javascript
lot.capacity -= lot.capacityInQuote ? amount_ : payout;
```
Here you can see if capacityInQuote is true then capacity is reduced from the `amount_`, that is which was being provided by purchaser , which will be in `ETH` , which is the Quote token, else it would be reduced by payout, i.e the baseTokens the purchaser would get.
Hence the amount_ will have 18 decimals & payout amount will have 6 decimals
Now , lets say a purchaser sends amount = 2ETH, i.e. _PURCHASE_AMOUNT = 10e18.
so here payoutAmount or `expectedSold` = 5e6, i.e. (amount * baseTokenDecimals )/ price

So here the payout or the amount sold will be done correctly, but while reducing the capacity, the capacity will be reduced by the payout amount i.e
capacity = 10e18-5e6 = 9.999999999995e18
where it should have been 10e18-5e18 = 5e18.

The capacity didn't got reduced as it should have been.

Now how this will benefit seller is, in two purchases of 10e18 purchaseAmount the capacity would have reduced to zero, and further no payments would have been done, the seller would have got 20e18 and the purchasers would have got their payout amounts & the auction would have ended.

But since here the capacity is getting reduced from baseTokens decimals, here in this example it is getting reduced from e6 decimals. 

Hence till capacity gets zero, the purchasers will be to purchase the baseTokens and the quoteTokens will always be earned by the seller, where here the capacity will get zero after many many purchases, where it could have been zero just from two 5e18 purchases.

Hence the seller will be able to earn a lot of ETH(quoteToken) in this case since the ETH will always be reduced by `e6` amount.

And how can other auction seller with same baseTokens , have loss of funds is : the baseTokens are being transferred to purchasers from AuctionHouse contract, since the purchasers will always be able to purchase, when the baseTokens from this seller get being completely sent, the future purchasers will get baseToken from other auction seller provided base tokens.

Where at a point since just like in this example it would have taken a long time to capacity be zero and auction to conclude, the purchasers would have continued purchasing, and then all the baseTokens in the AuctionHouse contract will be completely used up. 

Because of which the other auction purchaser will also not be able to purchase in their auction. Leading to completely wrecking the system.

<details>
<summary>CODED POC</summary>
Run this test using 

`forge test --mt test_success_quoteTokenDecimalsLarger -vvvv`

This test is the modified version of already exisiting testcase in purchase.t.sol for FPA module.

I have used console.log since was getting error related to assertEq, hence commented it too.
Before running:
- put `_PurchaseAmount = 10e18`.
- This issue is actually because of not scaling the capacity in baseToken decimals if capacityInQuote is false, which is not done in the main contracts code, but is being implemented while writing test cases.
So also do comment out just the following way in _setBaseTokenDecimals modifier in FpaModuleTest.sol

Commented out because it is actually not implemented in main code. If this would have been implemented in main code, then this vulnerability won't exist.

```javascript
function _setBaseTokenDecimals(uint8 decimals_) internal {
        _baseTokenDecimals = decimals_;

        // if (!_auctionParams.capacityInQuote) {
        //     _auctionParams.capacity = _scaleBaseTokenAmount(_LOT_CAPACITY);
        // }
    }
```

```javascript
function test_success_quoteTokenDecimalsLarger()
        public
        givenQuoteTokenDecimals(18)
        givenBaseTokenDecimals(6)
        givenLotIsCreated
        givenLotHasStarted
    {
        // Calculate expected values
        uint96 expectedSold = _mulDivDown(
            _scaleQuoteTokenAmount(_PURCHASE_AMOUNT),
            uint96(10) ** _baseTokenDecimals,
            _scaleQuoteTokenAmount(_PRICE)
        );

        Auction.Lot memory lot = _getAuctionLot(_lotId);
        console.log("Capacity before", lot.capacity);

        // Call the function
        _createPurchase(
            _scaleQuoteTokenAmount(_PURCHASE_AMOUNT), _scaleBaseTokenAmount(_PURCHASE_AMOUNT_OUT)
        );

        // Assert the capacity, purchased and sold
        Auction.Lot memory lot1 = _getAuctionLot(_lotId);
        console.log("Boolean :", lot1.capacityInQuote);
        console.log("Capacity after", lot1.capacity);
        console.log("Purchased", lot1.purchased);
        console.log("Actual sold", lot1.sold);
        console.log("Expected sold", expectedSold);
        // assertEq(lot.capacity, _scaleBaseTokenAmount(_LOT_CAPACITY) - expectedSold, "capacity");
        // assertEq(lot.purchased, _scaleQuoteTokenAmount(_PURCHASE_AMOUNT), "purchased");
        // assertEq(lot.sold, expectedSold, "sold");
    }
```

You will get following logs.
```diff
Running 1 test for test/modules/auctions/FPA/purchase.t.sol:FpaModulePurchaseTest
[PASS] test_success_quoteTokenDecimalsLarger() (gas: 136126)
Logs:
  Capacity before 10000000000000000000
  Boolean : false
  Capacity after 9999999999995000000
  Purchased 10000000000000000000
  Actual sold 5000000
  Expected sold 5000000
```
</details> 

## Impact
1. Auction seller can earn more quoteTokens than intended in an unfair manner.
2. If the auction goes for long time, just like happening in above example, then other auction sellers with same baseTokens would be in loss of funds since all their baseTokens would be purchase by this auction purchasers, completely wrecking the system

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L201-L329

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/bases/Auctioneer.sol#L160-L285

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L388-L413

## Tool used

Manual Review

## Recommendation
As I said in the coded POC part, this vulnerability is existing just because there no scaling of capacity to baseTokenDecimals, done in the main contracts code, if `capacityInQuote` is kept as false.

But it has been implemented in test case, I commented out that part in test file, for showing the vulnerability in main code.
If those comments are removed and again if test is run, you can see in the following logs:

```diff
Running 1 test for test/modules/auctions/FPA/purchase.t.sol:FpaModulePurchaseTest
[PASS] test_success_quoteTokenDecimalsLarger() (gas: 140130)
Logs:
  Capacity before 10000000
  Boolean : false
  Capacity after 5000000
  Purchased 10000000000000000000
  Actual sold 5000000
  Expected sold 5000000
```

The capacity is scaled to `e6` and all implementation are correct now.

Hence for mitigation , implement the same scaling method in main contract codes just like it was done in test file `FPAModuleTest::_setBaseTokenDecimals` modifier:

```javascript
if (!_auctionParams.capacityInQuote) {
            _auctionParams.capacity = _scaleBaseTokenAmount(_LOT_CAPACITY);
        }

function _scaleBaseTokenAmount(uint96 amount_) internal view returns (uint96) {
        return _mulDivUp(amount_, uint96(10 ** _baseTokenDecimals), _BASE_SCALE);
    }
```
Macho Shadow Walrus

high

# Catalogue incorrectly calculates prices for auctions, leading to heavily inflated display of prices

## Summary
`Catalogue.sol` provides view functions which allows users to view details of specific auctions. This includes prices of auctions, so they can have an estimate on how much they should pay for a specific payout, which is exactly what `Catalogue::priceFor` is intended to do. However, the calculated return price for `priceFor` (and `maxAmountAccepted`) is incorrect due to a miscalculation error. The miscalculation returns a price that is almost double than the actual price the user has to pay for the payout they entered. When users determine how much to pay via the prices returned from the catalogue, they will either pay much more than the actual price of the items being sold, or decide not to participate in the auction at all because it is heavily overpriced.

## Vulnerability Detail
Lets first take a look at `FeeManager::calculateQuoteFees`, which is called throughout the `AuctionHouse.sol` contract, for example when a user wants to purchase base tokens with quote tokens for an atomic bid. These fees are intended for the protocol and a referrer (optional) for the bid. The fees will be deducted from the price the user entered for the bid, and allocated to the protocol and referrer.

`FeeManager::calculateQuoteFees`
```javascript
    /// @notice     Calculates and allocates fees that are collected in the quote token
    function calculateQuoteFees(
        uint96 protocolFee_,
        uint96 referrerFee_,
        bool hasReferrer_,
        uint96 amount_
    ) public pure returns (uint96 toReferrer, uint96 toProtocol) {
        uint96 feeDecimals = uint96(_FEE_DECIMALS);

        if (hasReferrer_) {
            // In this case we need to:
            // 1. Calculate referrer fee
            // 2. Calculate protocol fee as the total expected fee amount minus the referrer fee
            //    to avoid issues with rounding from separate fee calculations
            toReferrer = uint96(Math.mulDivDown(amount_, referrerFee_, feeDecimals));
            toProtocol = uint96(Math.mulDivDown(amount_, protocolFee_ + referrerFee_, feeDecimals))
                - toReferrer;
        } else {
            // If there is no referrer, the protocol gets the entire fee
            toProtocol = uint96(Math.mulDivDown(amount_, protocolFee_ + referrerFee_, feeDecimals));
        }
    }
```

This is how the fees for the quote tokens are calculated. `protocolFee_` and `referrerFee` are in basis points where `1000 = 1%`. This is from the docs in `FeeManager.sol`:

```javascript
    /// @notice     Fees are in basis points (3 decimals). 1% equals 1000.
    uint48 internal constant _FEE_DECIMALS = 1e5;
```

The following example will display how fees are calculated from the price the user entered. For this example lets set the following:

amount_ = 2 x 10^18 (this is the amount of quote tokens the user is paying for the base tokens)
protocolFee_ = 100 (0.1%)
referrerFee_ = 105 (0.105%)

In this example, we have a referrer, however it doesn't matter because the problem is pertinent whether or not there is a referrer.

Note that these values are not chosen arbitrarily by me, they are the same values used for testing by the Axis Finance team. Please check `test/AuctionHouse/AuctionHouseTest.sol` where you can confirm this.

From the above example, when executing `calculateQuoteFees` we have:

toRefferer = (2 x 10^18) * (105) / (10^5) = 2.1 x 10^15.
toProtocol = (2 x 10^18) * (105 + 100) / (10^5) = 4.1 x 10^15 - 2.1 x 10^15 = 2 x 10^15.

Together, the fees are: `toRefferer +  toProtocol = 4.1 x 10^15`.

When the user pays the `2 x 10^18` amount of quote tokens, `4.1 x 10^15` amount will be deducted from that amount and allocated for fees. The amount, after deducting, will be used for the actual purchase of the base tokens, which in our example is `2 x 10^18 - 4.1 x 10^15 = 1.9959 x 10^18`.  We can confirm this by checking `AuctionHouse::purchase`, where `params_.amount` is the amount the user entered and `totalFees` is the sum of the fees (in our example 2 x 10^18 and 4.1 x 10^15, respectively). Note that `_allocateQuoteFees` calculates the fees (by calling `FeeManager::calculateQuoteFees`), and proceeds to add it to the respective balances, then returns the sum. You can see that the `amountLessFees` is used for the actual purchase, which is the amount the user entered minus the allocated fees (1.9959 x 10^18 in our example, not the initial 2 x 10^18 the user entered).

`AuctionHouse::purchase`
```javascript
    uint96 amountLessFees;
    {
        Keycode auctionKeycode = keycodeFromVeecode(routing.auctionReference);
@>      uint96 totalFees = _allocateQuoteFees(
            fees[auctionKeycode].protocol,
            fees[auctionKeycode].referrer,
            params_.referrer,
            routing.seller,
            routing.quoteToken,
@>          params_.amount
        );
        unchecked {
@>          amountLessFees = params_.amount - totalFees;
        }
    }

    // Send purchase to auction house and get payout plus any extra output
    bytes memory auctionOutput;
     (payoutAmount, auctionOutput) = _getModuleForId(params_.lotId).purchase(
@>      params_.lotId, amountLessFees, params_.auctionData
    );
```

`AuctionHouse::_allocateQuoteFees`
```javascript
    function _allocateQuoteFees(
        uint96 protocolFee_,
        uint96 referrerFee_,
        address referrer_,
        address seller_,
        ERC20 quoteToken_,
        uint96 amount_
    ) internal returns (uint96 totalFees) {
        // Calculate fees for purchase
        (uint96 toReferrer, uint96 toProtocol) = calculateQuoteFees(
            protocolFee_, referrerFee_, referrer_ != address(0) && referrer_ != seller_, amount_
        );

        // Update fee balances if non-zero
        if (toReferrer > 0) rewards[referrer_][quoteToken_] += uint256(toReferrer);
        if (toProtocol > 0) rewards[_protocol][quoteToken_] += uint256(toProtocol);

        return toReferrer + toProtocol;
    }
```


As we can see, if the user decides that they want to pay the full amount of `2 x 10^18` quote tokens, they must pay `2 x 10^18 + 4 x 10^15 = 2.0041 x 10^18 (amount = 2004100000000000000)` to account for the fees. Thankfully, the user can rely on `Catalogue::priceFor` which will return the price in quote tokens the user must pay, including the fee estimate.

`Catalogue::priceFor`
```javascript
    function priceFor(uint96 lotId_, uint96 payout_) external view returns (uint256) {
        Auction module = Auctioneer(auctionHouse).getModuleForId(lotId_);
        Auctioneer.Routing memory routing = getRouting(lotId_);

        // Get price from module (in quote token units)
        uint256 price = module.priceFor(lotId_, payout_);

        // Calculate fee estimate assuming there is a referrer and add to price
        price += _calculateFeeEstimate(keycodeFromVeecode(routing.auctionReference), true, price);

        return price;
    }
```

Lets say the user calls `priceFor` for a specific `payout_`. For this example, it doesn't matter what `payout_` is, but lets assume that the price returned for that payout is `2 x 10^18`. The function proceeds to call `_calculateFeeEstimate` with the `price = 2 x 10^18` to calculate the fees the user must pay as well.

`Catalogue::_calculateFeeEstimate`
```javascript
    /// @notice Estimates fees for a `priceFor` or `maxAmountAccepted` calls
    function _calculateFeeEstimate(
        Keycode auctionType_,
        bool hasReferrer_,
        uint256 price_
    ) internal view returns (uint256 feeEstimate) {
        // In this case we have to invert the fee calculation
        // We provide a conservative estimate by assuming there is a referrer and rounding up
        (uint48 fee, uint48 referrerFee,) = FeeManager(auctionHouse).fees(auctionType_);
        if (hasReferrer_) fee += referrerFee;

        uint256 numer = price_ * _FEE_DECIMALS;
        uint256 denom = _FEE_DECIMALS - fee;

        return (numer / denom) + ((numer % denom == 0) ? 0 : 1); // round up if necessary
    }
```

We will continue to use the same values: `fee = 100` (protocol fee), `referrerFee = 105` (referrer fee). `hasRefferrer_` is true, so fee = 100 + 105 = 205. 

numer = 2x10^18 * 10^5 = 2 x 10^23
denom = 10^5 - 205 = 99795

(numer/denom) = 2004108422265644571

Then, we account for the rounding (numer%denom != 0, therefore we add 1) => our final answer is 2004108422265644572.

This value is quite exactly what we calculated above where if we want to pay 2 x 10^18 in quote tokens, we must pay 2.0041 x 10^18 to account for the fees. Therefore we now have the correct price, plus the fees.

Continuing with the `priceFor` function:

price += _calculateFeeEstimate(keycodeFromVeecode(routing.auctionReference), true, price); 

=> price = (2 x 10^18) + (2004108422265644572) = 4004108422265644572 (4.0041 x 10^18);

This price is incorrect. We know that `_calculateFeeEstimate` already correctly calculates the price + fees and confirmed it at the start of the example by checking these exact values with `AuctionHouse::purchase` and `FeeManager::calculateQuoteFees`. However, due to the `price +=` operation, instead of `price =`, we are incorrectly adding the price to the already calculated `price + fees`, which returns a price almost double than what to actually pay for the desired payout. (2.0041 x 10^18 versus 4.0041 x 10^18). 

## Impact
This is certainly a high severity issue for the following reasons:

1. Users can very well pay this double amount, thus paying far more than what is actually worth and ultimately losing massive amounts of funds
2. Realize that this is far too expensive (when in reality, it is not) and decide to not participate in the auction.
3. Due to the above two issues, many people will not resort to Axis Finance for holding auctions.  

`Catalogue::priceFor` will be relied upon heavily for price calculations, thus it is very important that it returns the correct value. This exact issue is also in `Catalogue::maxAmountAccepted`. Whether it is a batch or atomic auction, `priceFor` and `maxAmountAccepted` will return a heavily inflated price.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/Catalogue.sol#L83

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/Catalogue.sol#L105-L106

## Tool used
Manual Review

## Recommendation
Remove the `+=` operator:

`Catalogue::priceFor`
```diff
    function priceFor(uint96 lotId_, uint96 payout_) external view returns (uint256) {
        Auction module = Auctioneer(auctionHouse).getModuleForId(lotId_);
        Auctioneer.Routing memory routing = getRouting(lotId_);

        // Get price from module (in quote token units)
        uint256 price = module.priceFor(lotId_, payout_);

        // Calculate fee estimate assuming there is a referrer and add to price
-       price += _calculateFeeEstimate(keycodeFromVeecode(routing.auctionReference), true, price);
+       price = _calculateFeeEstimate(keycodeFromVeecode(routing.auctionReference), true, price);

        return price;
    }
```

`Catalogue::maxAmountAccepted`
```diff
    function maxAmountAccepted(uint96 lotId_) external view returns (uint256) {
        Auction module = Auctioneer(auctionHouse).getModuleForId(lotId_);
        Auctioneer.Routing memory routing = getRouting(lotId_);

        // Get max amount accepted from module
        uint256 maxAmount = module.maxAmountAccepted(lotId_);

        // Calculate fee estimate assuming there is a referrer and add to max amount
-       maxAmount +=
-           _calculateFeeEstimate(keycodeFromVeecode(routing.auctionReference), true, maxAmount);

+       maxAmount =
+           _calculateFeeEstimate(keycodeFromVeecode(routing.auctionReference), true, maxAmount);

        return maxAmount;
    }
```
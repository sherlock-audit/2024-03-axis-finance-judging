Macho Sandstone Platypus

medium

# Inaccurate value is used for partial fill quote amount when calculating fees

## Summary
Inaccurate value is used for partial fill quote amount when calculating fees which can cause reward claiming / payment withdrawal to revert

## Vulnerability Detail

The fees of an auction is managed as follows:

1. Whenever a bidder claims their payout, calculate the amount of quote tokens that should be collected as fees (instead of giving the entire quote amount to the seller) and add this to the protocol / referrers rewards

```solidity
    function claimBids(uint96 lotId_, uint64[] calldata bidIds_) external override nonReentrant {
        
        ....

        for (uint256 i = 0; i < bidClaimsLen; i++) {
            Auction.BidClaim memory bidClaim = bidClaims[i];

            if (bidClaim.payout > 0) {
               
=>              _allocateQuoteFees(
                    protocolFee,
                    referrerFee,
                    bidClaim.referrer,
                    routing.seller,
                    routing.quoteToken,
=>                  bidClaim.paid
                );
```

Here bidClaim.paid is the amount of quote tokens that was transferred in by the bidder for the purchase

```solidity
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

2. Whenever the seller calls claimProceeds to withdraw the amount of quote tokens received from the auction, subtract the quote fees and give out the remaining

```solidity
    function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        
        ....
        
        uint96 totalInLessFees;
        {
=>          (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
=>              totalInLessFees = purchased_ - toProtocol;
            }
        }
```

Here purchased is the total quote token amount that was collected for this auction.

In case the fees calculated in claimProceeds is less than the sum of fees allocated to the protocol / referrer via claimBids, there will be a mismatch causing the sum of (fees allocated + seller purchased quote tokens) to be greater than the total quote token amount that was transferred in for the auction. This could cause either the protocol/referrer to not obtain their rewards or the seller to not be able to claim the purchased tokens in case there are no excess quote token present in the auction house contract.

In case, totalPurchased is >= sum of all individual bid quote token amounts (as it is supposed to be), the fee allocation would be correct. But due to the inaccurate computation of the input quote token amount associated with a partial fill, it is possible for the above scenario (ie. `fees calculated in claimProceeds is less than the sum of fees allocated to the protocol / referrer via claimBids`) to occur

```solidity
    function settle(uint96 lotId_) external override nonReentrant {
        
        ....

            if (settlement.pfBidder != address(0)) {

                _allocateQuoteFees(
                    feeData.protocolFee,
                    feeData.referrerFee,
                    settlement.pfReferrer,
                    routing.seller,
                    routing.quoteToken,

                    // @audit this method of calculating the input quote token amount associated with a partial fill is not accurate
                    uint96(
=>                      Math.mulDivDown(
                            settlement.pfPayout, settlement.totalIn, settlement.totalOut
                        )
                    )
```

The above method of calculating the input token amount associated with a partial fill can cause this value to be higher than the acutal value and hence the fees allocated will be less than what the fees that will be captured from the seller will be

### POC
Apply the following diff to `test/AuctionHouse/AuctionHouseTest.sol` and run `forge test --mt testHash_SpecificPartialRounding -vv`

It is asserted that the tokens allocated as fees is greater than the tokens that will be captured from a seller for fees

```diff
diff --git a/moonraker/test/AuctionHouse/AuctionHouseTest.sol b/moonraker/test/AuctionHouse/AuctionHouseTest.sol
index 44e717d..9b32834 100644
--- a/moonraker/test/AuctionHouse/AuctionHouseTest.sol
+++ b/moonraker/test/AuctionHouse/AuctionHouseTest.sol
@@ -6,6 +6,8 @@ import {Test} from "forge-std/Test.sol";
 import {ERC20} from "solmate/tokens/ERC20.sol";
 import {Transfer} from "src/lib/Transfer.sol";
 import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";
+import {SafeCastLib} from "solmate/utils/SafeCastLib.sol";
+
 
 // Mocks
 import {MockAtomicAuctionModule} from "test/modules/Auction/MockAtomicAuctionModule.sol";
@@ -134,6 +136,158 @@ abstract contract AuctionHouseTest is Test, Permit2User {
         _bidder = vm.addr(_bidderKey);
     }
 
+        function testHash_SpecificPartialRounding() public {
+        /*
+            capacity 1056499719758481066
+            previous total amount 1000000000000000000
+            bid amount 2999999999999999999997
+            price 2556460687578254783645
+            fullFill 1173497411705521567
+            excess 117388857750942341
+            pfPayout 1056108553954579226
+            pfRefund 300100000000000000633
+            new totalAmountIn 2700899999999999999364
+            usedContributionForQuoteFees 2699900000000000000698
+            quoteTokens1 1000000
+            quoteTokens2 2699900000
+            quoteTokensAllocated 2700899999
+        */
+
+        uint bidAmount = 2999999999999999999997;
+        uint marginalPrice = 2556460687578254783645;
+        uint capacity = 1056499719758481066;
+        uint previousTotalAmount = 1000000000000000000;
+        uint baseScale = 1e18;
+
+        // hasn't reached the capacity with previousTotalAmount
+        assert(
+            FixedPointMathLib.mulDivDown(previousTotalAmount, baseScale, marginalPrice) <
+                capacity
+        );
+
+        uint capacityExpended = FixedPointMathLib.mulDivDown(
+            previousTotalAmount + bidAmount,
+            baseScale,
+            marginalPrice
+        );
+        assert(capacityExpended > capacity);
+
+        uint totalAmountIn = previousTotalAmount + bidAmount;
+
+        uint256 fullFill = FixedPointMathLib.mulDivDown(
+            uint256(bidAmount),
+            baseScale,
+            marginalPrice
+        );
+
+        uint256 excess = capacityExpended - capacity;
+
+        uint pfPayout = SafeCastLib.safeCastTo96(fullFill - excess);
+        uint pfRefund = SafeCastLib.safeCastTo96(
+            FixedPointMathLib.mulDivDown(uint256(bidAmount), excess, fullFill)
+        );
+
+        totalAmountIn -= pfRefund;
+
+        uint usedContributionForQuoteFees;
+        {
+            uint totalOut = SafeCastLib.safeCastTo96(
+                capacityExpended > capacity ? capacity : capacityExpended
+            );
+
+            usedContributionForQuoteFees = FixedPointMathLib.mulDivDown(
+                pfPayout,
+                totalAmountIn,
+                totalOut
+            );
+        }
+
+        {
+            uint actualContribution = bidAmount - pfRefund;
+
+            // acutal contribution is less than the usedContributionForQuoteFees
+            assert(actualContribution < usedContributionForQuoteFees);
+            console2.log("actual contribution", actualContribution);
+            console2.log(
+                "used contribution for fees",
+                usedContributionForQuoteFees
+            );
+        }
+
+        // calculating quote fees allocation
+        // quote fees captured from the seller
+        {
+            (, uint96 quoteTokensAllocated) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(totalAmountIn)
+            );
+
+            // quote tokens that will be allocated for the earlier bid
+            (, uint96 quoteTokens1) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(previousTotalAmount)
+            );
+
+            // quote tokens that will be allocated for the partial fill
+            (, uint96 quoteTokens2) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(usedContributionForQuoteFees)
+            );
+            
+            console2.log("quoteTokens1", quoteTokens1);
+            console2.log("quoteTokens2", quoteTokens2);
+            console2.log("quoteTokensAllocated", quoteTokensAllocated);
+
+            // quoteToken fees allocated is greater than what will be captured from seller
+            assert(quoteTokens1 + quoteTokens2 > quoteTokensAllocated);
+        }
+    }
+
+        function calculateQuoteFees(
+        uint96 protocolFee_,
+        uint96 referrerFee_,
+        bool hasReferrer_,
+        uint96 amount_
+    ) public pure returns (uint96 toReferrer, uint96 toProtocol) {
+        uint _FEE_DECIMALS = 5;
+        uint96 feeDecimals = uint96(_FEE_DECIMALS);
+
+        if (hasReferrer_) {
+            // In this case we need to:
+            // 1. Calculate referrer fee
+            // 2. Calculate protocol fee as the total expected fee amount minus the referrer fee
+            //    to avoid issues with rounding from separate fee calculations
+            toReferrer = uint96(
+                FixedPointMathLib.mulDivDown(amount_, referrerFee_, feeDecimals)
+            );
+            toProtocol =
+                uint96(
+                    FixedPointMathLib.mulDivDown(
+                        amount_,
+                        protocolFee_ + referrerFee_,
+                        feeDecimals
+                    )
+                ) -
+                toReferrer;
+        } else {
+            // If there is no referrer, the protocol gets the entire fee
+            toProtocol = uint96(
+                FixedPointMathLib.mulDivDown(
+                    amount_,
+                    protocolFee_ + referrerFee_,
+                    feeDecimals
+                )
+            );
+        }
+    }
+
+
     // ===== Helper Functions ===== //
 
     function _mulDivUp(uint96 mul1_, uint96 mul2_, uint96 div_) internal pure returns (uint96) {

```

## Impact

Rewards might not be collectible or seller might not be able to claim the proceeds due to lack of tokens

## Code Snippet

inaccurate computation of the input quote token value for allocating fees
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L512-L515

## Tool used

Manual Review

## Recommendation

Use `bidAmount - pfRefund` as the quote token input amount value instead of computing the current way
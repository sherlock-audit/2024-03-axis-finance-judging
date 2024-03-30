Macho Shadow Walrus

medium

# Bids can pass price < minimum amount check for batch auctions due to inflated bids, breaking protocol invariant

## Summary
The Axis Finance Protocol allows for sellers to create batch auctions and atomic auctions. When users submit their bids, fees are deducted from that amount and allocated for the protocol and referrer (optional). Looking at atomic auctions, the price (in quote tokens) which buyers submit is immediately used to allocate fees for the protocol/referrer. The fees are then subtracted from the price, and that price is used to purchase base tokens from the seller. The problem is that for batch auctions, the price *including fees* is used to purchase base tokens from the seller. This means that the price used to buy the amount of base tokens is inflated, because the user is not actually paying for the base tokens with the price including the fees (the seller receives the payment minus fees). Due to this inflation, bids can exceed the minimum bid size at the minimum price, breaking a protocol invariant.

## Vulnerability Detail
Lets take a look at `AuctionHouse::bid`, where buyers enter the amount they would like to pay, in quote tokens, for a batch auction.

`AuctionHouse::bid`
```javascript
    function bid(
        BidParams memory params_,
        bytes calldata callbackData_
    ) external override nonReentrant returns (uint64 bidId) {
        _isLotValid(params_.lotId);

        // Record the bid on the auction module
        // The module will determine if the bid is valid - minimum bid size, minimum price, auction status, etc
        bidId = _getModuleForId(params_.lotId).bid(
            params_.lotId, msg.sender, params_.referrer, params_.amount, params_.auctionData
        );

        // Transfer the quote token from the bidder
        _collectPayment(
            params_.amount,
            lotRouting[params_.lotId].quoteToken,
            Transfer.decodePermit2Approval(params_.permit2Data)
        );

        // Call the onBid callback
        Callbacks.onBid(
            lotRouting[params_.lotId].callbacks,
            params_.lotId,
            bidId,
            msg.sender,
            params_.amount,
            callbackData_
        );

        // Emit event
        emit Bid(params_.lotId, bidId, msg.sender, params_.amount);

        return bidId;
    
    }
```

The entire payment the buyer entered is transferred to the contract. Looking at `EMPAM.sol` (batch auction module), the bid is processed and encrypted. When the auction is finally settled, bids are decrypted in descending order (highest price to lowest) and used to determine the marginal price, which is the price (in quote tokens) to determine at what price to sell the base tokens. Once the price is determined, the seller calls `AuctionHouse::claimProceeds` to receive the payment in quote tokens.

`AuctionHouse::claimProceeds`
```javascript
        // Calculate the referrer and protocol fees for the amount in
        // Fees are not allocated until the user claims their payout so that we don't have to iterate through them here
        // If a referrer is not set, that portion of the fee defaults to the protocol
        uint96 totalInLessFees;
        {
            (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
@>              totalInLessFees = purchased_ - toProtocol;
            }
        }

        // Send payment in bulk to the address dictated by the callbacks address
        // If the callbacks contract is configured to receive quote tokens, send the quote tokens to the callbacks contract and call the onClaimProceeds callback
        // If not, send the quote tokens to the seller and call the onClaimProceeds callback
        _sendPayment(routing.seller, totalInLessFees, routing.quoteToken, routing.callbacks);
```

Where `purchased_` = total amount purchased in quote tokens, the total fees are calculated and deducted from that amount. The new amount, after deducting fees, is sent to the seller.

Finally, users can claim their bids. If their bid was filled, they claim the base tokens. If not, they receive a full refund.

`AuctionHouse::claimBids`
```javascript
            // If payout is greater than zero, then the bid was filled.
            // Otherwise, it was not and the bidder is refunded the paid amount.
            if (bidClaim.payout > 0) {
                // Allocate quote and protocol fees for bid
                _allocateQuoteFees(
                    protocolFee,
                    referrerFee,
                    bidClaim.referrer,
                    routing.seller,
                    routing.quoteToken,
                    bidClaim.paid
                );

                // Reduce funding by the payout amount
                unchecked {
                    routing.funding -= bidClaim.payout;
                }

                // Send the payout to the bidder
                _sendPayout(bidClaim.bidder, bidClaim.payout, routing, auctionOutput);
            } else {
                // Refund the paid amount to the bidder
                Transfer.transfer(routing.quoteToken, bidClaim.bidder, bidClaim.paid, false);
            }
```

The fees are allocated from each bid that was filled. This is correct, because the protocol calculated the fees to deduct from the amount that was paid to the seller (in quote tokens), and here it is actually allocated. 

Now that we have established that when buyers send their bids, their bids include fees which are deducted and allocated after all bids have been processed and settled. This means that all bids are slightly inflated by the amount of fees set by the protocol (price + fees). Although this does not cause any issues with the auction itself (since all bidders must allocate the same amount of fees, all bids are inflated by the same amount, which does not impact batch auctions), it can break a core invariant where the amount a buyer bids can exceed the minimum amount set by the seller due to inflation, when in reality a lower amount is sent to the seller for the bid.

Batch auctions allow sellers to set a minimum amount for bids. This is done by setting `minPrice` and `minBidPercent`. `minBidPercent` is then used to calculate the `minBidSize`.

`EMPAM::_auction`
```javascript
@>      data.minPrice = implParams.minPrice;

@>      data.minBidSize = uint96(
            Math.mulDivUp(
                uint256(lot_.capacity), implParams.minBidPercent, uint256(_ONE_HUNDRED_PERCENT)
            )
        );
    }
```

A buyer must bid at least the minimum amount for their bid to be processed:

`EMPAM::_bid`
```javascript
        // Amount must be at least the minimum bid size at the minimum price
        uint256 minAmount = Math.mulDivDown(
            uint256(auctionData[lotId_].minBidSize),
            uint256(auctionData[lotId_].minPrice),
            10 ** lotData[lotId_].baseTokenDecimals
        );
@>      if (amount_ < minAmount) revert Auction_AmountLessThanMinimum();
    }
```

The following example will showcase this issue. For simplicity, we will look at a batch auction with just 1 bid.

The following are set by the seller of the auction:

- lot_.capacity = 4 x 10^18 base tokens
- minPrice = 1 x 10^18 quote tokens
- minBidPercent = 50_000 (50%)

Auction.sol docs:
```javascript
    /// @notice Constant for percentages
    /// @dev    1% = 1_000 or 1e3. 100% = 100_000 or 1e5.
    uint48 internal constant _ONE_HUNDRED_PERCENT = 100_000;
```

From the above values, we have `minBidSize = 2 x 10^18`, which gives us `minAmount = 2 x 10^18 quote tokens`

Now, lets say the buyer bids with amount = 2 x 10^18 quote tokens and fills the capacity.

Lets assume the total fees = 5000 (5%). 

FeeManager.sol docs:
```javascript
    /// @notice     Fees are in basis points (3 decimals). 1% equals 1000.
    uint48 internal constant _FEE_DECIMALS = 1e5;
```

When the seller calls `AuctionHouse::claimProceeds`, the amount that is sent is: (amount purchased) - (calculated fees)

From the amount used to purchase the bid `2 x 10^18 quote tokens` and `total fees = 5000`, we get `calculated fees = 0.1 x 10^18`. The seller receives 2 x 10^18 - 0.1 x 10^18 = 1.9 x 10^18 quote tokens . This is less than the `minPrice` they set at 2 x 10^18 quote tokens, breaking the invariant of bids not being able to exceed the minimum price set by the seller.

Essentially what happened here was that the bidder actually used `1.9 x 10^18 quote tokens` for the bid, since 0.1 x 10^18 were used for fees, not for the bid. The seller set the minimum at `2 x 10^18 quote tokens`, but the bidder was able to bid less, thus breaking the invariant. To fix this, fees must be deducted from the bidder's payment initially.

## Impact
Invariant broken, bidders are able to bid price lower than minimum price set by seller. This causes a loss for sellers who desired a minimum payment for the base tokens they are auctioning.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L340

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L578-L595

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L425-L434

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L835

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/FeeManager.sol#L68

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L242-L247


## Tool used
Manual Review

## Recommendation
Deduct fees from the price entered by bidders for batch auctions at the beginning, rather than after settling. Use that price to process the bids, just how it's done for atomic auctions (please see `AuctionHouse::purchase`).
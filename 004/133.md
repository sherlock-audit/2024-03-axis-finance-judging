Macho Shadow Walrus

medium

# Users can pay less protocol fees by setting themselves as referrer

## Summary
The Axis Finance Protocol allocates fees for the protocol and referrer (optional) from bids for both atomic and batch auctions. The protocol allows bidders the option to pass in the address of a referrer, who is subsequently paid a fee from the buyer. The amount of the fees to pay are set by the `AuctionHouse` contract owner. This incentivizes participants to refer auctions to others, in hopes of receiving fees from bidders. If a referrer address is not provided, buyers still have to pay the referrer fee, but the fee will be added to the protocol fee instead. The problem is that bidders can simply avoid paying the referrer fee by setting themselves as the referrer. The protocol will not only receive less fees than expected, but this is also unfair to bidders who pay for the referrer fee.

## Vulnerability Detail
To participate in an atomic auction, the following function is called, where `params_.amount` is the amount the user bids with. Prior to processing the user's bid, fees are first allocated for the protocol and referrer (`params_.referrer`, which is optional and set by the bidder).

`AuctionHouse::purchase`
```javascript
    function purchase(
        PurchaseParams memory params_,
        bytes calldata callbackData_
    ) external override nonReentrant returns (uint96 payoutAmount) {
        _isLotValid(params_.lotId);

        Routing storage routing = lotRouting[params_.lotId];

        uint96 amountLessFees;
        {
            Keycode auctionKeycode = keycodeFromVeecode(routing.auctionReference);
@>          uint96 totalFees = _allocateQuoteFees(
                fees[auctionKeycode].protocol,
                fees[auctionKeycode].referrer,
@>              params_.referrer,
                routing.seller,
                routing.quoteToken,
@>              params_.amount
            );

            unchecked {
                amountLessFees = params_.amount - totalFees;
            }
        }

        // Send purchase to auction house and get payout plus any extra output
        bytes memory auctionOutput;
        (payoutAmount, auctionOutput) = _getModuleForId(params_.lotId).purchase(
            params_.lotId, amountLessFees, params_.auctionData
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

A subsequent call to `FeeManager::calculateQuoteFees` is made:

```javascript
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

As you can see, the referrer address passed to the function by the user is optional. If it is empty (0 address), then the protocol gets the entire fee (protocolFee + referrerFee). Once fees are allocated and deducted from the users bid, the fees can be claimed:

`FeeManager::claimRewards`
```javascript
    function claimRewards(address token_) external nonReentrant {
        ERC20 token = ERC20(token_);
        uint256 amount = rewards[msg.sender][token];
        rewards[msg.sender][token] = 0;

        Transfer.transfer(token, msg.sender, amount, false);
    }
```

Lets look at the following example:

- User bids amount 2 x 10^18 quote tokens
- protocol fee set by owner: 100 (0.1%)
- referrer fee set by owner: 105 (0.105%)

This is from the docs in `FeeManager.sol`:

```javascript
    /// @notice     Fees are in basis points (3 decimals). 1% equals 1000.
    uint48 internal constant _FEE_DECIMALS = 1e5;
```

**Scenario 1**: User does not set referrer address

    Fees calculated (referrer address is 0): 4.1 x 10^15 amount of fees, in quote tokens, are sent to the protocol from the user.

**Scenario 2**: User decides to set themselves as referrer to avoid paying the full amount of the above fee, and decides to set their own address/alt address as referrer

    Fees calculated (referrer address is non-zero):

    to referrer: 2.1 x 10^15
    to protocol: 2 x 10^15

    User calls claimRewards and receives the 2.1 x 10^15 quote tokens, thus paying a total of 2 x 10^15 fees instead of the 4.1 x 10^15 they should have paid.


Note that although the issue was showcased with atomic auctions, the exact issue also applies to batch auctions.

## Impact
By setting themselves as the referrer, users can avoid paying a large amount of fees to the protocol. In addition to the protocol receiving less fees than expected, this is unfair to users who pay the entire fee fairly, causing a loss for them, and overall loss of trust for the protocol due to this loophole.


## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L215

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L835

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/FeeManager.sol#L68

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/FeeManager.sol#L123

## Tool used
Manual Review

## Recommendation
Since it's not possible to verify if the bidder is the owner of the referrer address (because they can have many addresses), the best recommendation I can suggest is to either remove the referrer option, or to not allocate referrer fees to the protocol if there is no referrer address set. That way, if a user sets themselves as a referrer it makes no difference since they will not be paying those fees to the protocol anyways, and users who do not have a referrer will not suffer unfairly.
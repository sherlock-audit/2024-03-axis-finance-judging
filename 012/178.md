Macho Sandstone Platypus

medium

# Bidder's payout claim could fail due to validation checks in LinearVesting

## Summary
Bidder's payout claim will fail due to validation checks in LinearVesting after the expiry timestamp

## Vulnerability Detail

Bidder's payout are sent by internally calling the `_sendPayout` function. In case the payout is a derivative which has already expired, this will revert due to the validation check of `block.timestmap < expiry` present in the mint function of LinearVesting derivative

```solidity
    function _sendPayout(
        address recipient_,
        uint256 payoutAmount_,
        Routing memory routingParams_,
        bytes memory
    ) internal {
        
        if (fromVeecode(derivativeReference) == bytes7("")) {
            Transfer.transfer(baseToken, recipient_, payoutAmount_, true);
        }
        else {
            
            DerivativeModule module = DerivativeModule(_getModuleIfInstalled(derivativeReference));

            Transfer.approve(baseToken, address(module), payoutAmount_);

=>          module.mint(
                recipient_,
                address(baseToken),
                routingParams_.derivativeParams,
                payoutAmount_,
                routingParams_.wrapDerivative
            );
```

```solidity
    function mint(
        address to_,
        address underlyingToken_,
        bytes memory params_,
        uint256 amount_,
        bool wrapped_
    )
        external
        virtual
        override
        returns (uint256 tokenId_, address wrappedAddress_, uint256 amountCreated_)
    {
        if (amount_ == 0) revert InvalidParams();

        VestingParams memory params = _decodeVestingParams(params_);

        if (_validate(underlyingToken_, params) == false) {
            revert InvalidParams();
        }
```

```solidity
    function _validate(
        address underlyingToken_,
        VestingParams memory data_
    ) internal view returns (bool) {
        
        ....

=>      if (data_.expiry < block.timestamp) return false;


        // Check that the underlying token is not 0
        if (underlyingToken_ == address(0)) return false;


        return true;
    }
```

Hence the user's won't be able to claim their payouts of an auction once the derivative has expired. For EMPAM auctions, a seller can also wait till this timestmap passes before revealing their private key which will disallow bidders from claiming their rewards.

## Impact

Bidder's won't be able claim payouts from auction after the derivative expiry timestamp

## Code Snippet

_sendPayout invoking mint function on derivative to send payouts 
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L823-L829

linear vesting derivative expiry checks
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/derivatives/LinearVesting.sol#L521-L541

## Tool used

Manual Review

## Recommendation

Allow to mint tokens even after expiry of the vesting token / deploy the derivative token first itself and when making the payout, transfer the base token directly incase the expiry time is passed
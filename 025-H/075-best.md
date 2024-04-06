Dandy Chrome Tarantula

high

# User can drain the balances of `LinearVesting` for every base token

## Summary

User can set the start time of token derivative deployment far enough in the past in order to accumulate the full balance of a particular derivative token in the contract in order to immediately drain the balance


## Vulnerability Detail

Deployment of derivative token is implemented in `deploy`:

```solidity
function deploy(
        address underlyingToken_,
        bytes memory params_,
        bool wrapped_
    ) external virtual override returns (uint256 tokenId_, address wrappedAddress_) {

        // Decode parameters (start, expiry)
        VestingParams memory params = _decodeVestingParams(params_);

        // Validate parameters
        if (_validate(underlyingToken_, params) == false) {
            revert InvalidParams();
        }

        // If necessary, deploy and store the data
        (uint256 tokenId, address wrappedAddress) =
            _deployIfNeeded(underlyingToken_, params, wrapped_);

        return (tokenId, wrappedAddress);

    }
```

Firstly there is validation of token deployment params(start and expiry)

```solidity
function _validate(
        address underlyingToken_,
        VestingParams memory data_
    ) internal view returns (bool) {
        // Revert if any of the timestamps are 0
        if (data_.start == 0 || data_.expiry == 0) return false;

        // Revert if start and expiry are the same (as it would result in a divide by 0 error)
        if (data_.start == data_.expiry) return false;

        // Check that the start time is before the expiry time
        if (data_.start >= data_.expiry) return false;

        // Check that the expiry time is in the future
        if (data_.expiry < block.timestamp) return false;

        // Check that the underlying token is not 0
        if (underlyingToken_ == address(0)) return false;

        return true;
    }
```

As you can see here no validation on the `data._start` is implemented to check if it is in the past. This will allow an attacker to set it far enough in the past to immediately gain the max amount of this particular derivative token and steal the balance of the contract.

```solidity
function redeemable(
        address owner_,
        uint256 tokenId_
    ) public view virtual override onlyValidTokenId(tokenId_) returns (uint256) {

        // Get the vesting data
        Token storage token = tokenMetadata[tokenId_];

        VestingData memory data = abi.decode(token.data, (VestingData));

        // If before the start time, 0
        if (block.timestamp <= data.start) return 0;

        // Get balances
        uint256 derivativeBalance = balanceOf[owner_][tokenId_];

        uint256 wrappedBalance =
            token.wrapped == address(0) ? 0 : SoulboundCloneERC20(token.wrapped).balanceOf(owner_);

        uint256 claimedBalance = userClaimed[owner_][tokenId_];

        uint256 totalAmount = derivativeBalance + wrappedBalance + claimedBalance;

        // Determine the amount that has been vested until date, excluding what has already been claimed
        uint256 vested;

        // If after the expiry time, all tokens are redeemable
        if (block.timestamp >= data.expiry) {
            vested = totalAmount;
        }

        // If before the expiry time, calculate what has vested already

        else {
            vested = totalAmount.mulDivDown(block.timestamp - data.start, data.expiry - data.start);
        }

        // Check invariant: cannot have claimed more than vested
        if (vested < claimedBalance) {
            revert BrokenInvariant();
        }

        // Deduct already claimed tokens
        vested -= claimedBalance;

        return vested;
    }
```

We can take advantage of this right here, our data.expiry just has to be < than block.timestamp and skip the first in order to go into the else statement where this vulnerability of setting the data.start far in the past will take place. This is how we can accumulate big profits for every derivative token.

```solidity
 // If after the expiry time, all tokens are redeemable
        if (block.timestamp >= data.expiry) {
            vested = totalAmount;
        }

        // If before the expiry time, calculate what has vested already

        else {
            vested = totalAmount.mulDivDown(block.timestamp - data.start, data.expiry - data.start);
        }
```


## Impact

User can drain the balances of `LinearVesting` for every base token

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/derivatives/LinearVesting.sol#L521-L541

## Tool used

Manual Review

## Recommendation

Implement restrictions like the ones in auction creation

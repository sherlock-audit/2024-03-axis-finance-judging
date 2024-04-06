Strong Juniper Lizard

high

# Seller claim quote tokens with losses for bueyrs

## Summary
There is a possibility that allows the seller to create an `EMPAM` in such a way that they can receive quote tokens while buyers cannot receive base tokens.

## Vulnerability Detail
If the seller wishes to limit who can call, they can simply not reveal the key to anyone else.
Let's consider the following scenario:

1)The seller sets the derivative module and specifies derivative parameters when creating the `EMPAM`. Suppose `lot.start = 100`, `lot.duration = 100`, meaning `lot.conclusion = 200`, `vesting.start = 210`, `vesting.expiry = 310`.
2) After the auction has concluded, a malicious seller does not submit the private key until `vesting.expiry < block.timestamp`.
3) After this, the seller calls `submitPrivateKey`, `decryptAndSortBids`, `settle` functions. 
`Note`: `partially filled bidder` and `curator` should be zero addresses during the `settle()` call in order to create such an attack.
The seller can successfully call `claimProceeds` and get quote tokens, while buyers will not be able to receive their base tokens due to the call to the derivative module (`LinearVesting`), which will fail due to the vesting expiration:
```solidity
if (_validate(underlyingToken_, params) == false) {
            revert InvalidParams();
        }
        
function _validate(
        address underlyingToken_,
        VestingParams memory data_
    ) internal view returns (bool) {
        ///code
    
        // Check that the expiry time is in the future
-->     if (data_.expiry < block.timestamp) return false;

        // Check that the underlying token is not 0
        if (underlyingToken_ == address(0)) return false;

        return true;
    }        
```
The issue lies in the fact that the seller intentionally did not submit the private key and waited for vesting expiration, but this situation can occur even if `submitPrivateKey, decryptAndSortBids, and settle` functions are not called due to other reasons (simply no one calls them), thereby causing buyers to lose their tokens. Therefore, two factors contributed to the successful attack: the delay in submitting the private key and how `auction.start, auction.duration, vesting.start, and vesting.expiry` are set - these are all numbers that are not validated against each other, so they can intersect, and `vesting.start` can start during `auction.start`, meaning there is no validation.

This validation is insufficient when creating an auction:
```solidity
 if (
                derivativeModule.TYPE() != Module.Type.Derivative ||
                !derivativeModule.validate(
                    address(routing.baseToken),
                    routing_.derivativeParams
                )
            ) {
                revert InvalidParams();
            }
```

In general, buyers can claim their bids and transfer base tokens to the derivative module 1 second before the vesting expiry, which means that buyers, by holding their base tokens for just one second on `LinearVesting`, can receive all base tokens with redeem(), which is also illogical.

## Impact
Buyers lose their base tokens to claim, while the seller loses nothing.

## Code Snippet
[src/modules/auctions/EMPAM.sol#L400](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L400)
[src/modules/derivatives/LinearVesting.sol#L535](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/derivatives/LinearVesting.sol#L535)

## Tool used

Manual Review

## Recommendation
1)Allowing the call to `refundBid` until the private key is submitted could be a way to partly mitigate the issue. 

2)Also, you need to validate `vesting.start` and `vesting.expiry` period.
Consider setting `vesting.start` after the lot has settled: `vesting.start = block.timestamp`, and instead of `vesting.expiry`, use `vesting.duration`, which will be calculated as `vesting.expiry = vesting.start + vesting.duration` if this is acceptable for the protocol, where a buyer holding their base tokens for just one second on `LinearVesting` can receive all base tokens with redeem().
Otherwise, the logic of `LinearVesting` needs to be changed.

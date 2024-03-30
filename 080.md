Zany Iron Beaver

medium

# The auction creator has control over the auction process.

## Summary
While no one should have control over an ongoing `auction`, the `auction creator`, who possesses the `private key`, can influence the `auction` by decrypting the `bid` information.
Unfortunately, this can result in several undesirable outcomes.
## Vulnerability Detail
During the `auctions`, the `creators` can access and decrypt `bids` using the `private key`.
Consequently, they gain insight into the ongoing `auctions'` status.
The `auction creator` can have several chocies.
1. `Aggressive Bidding`.
If the `creator` believes that `bidders'` prices are insufficient, they can place a `bid` at the highest price.
While this may result in some loss, it allows them to pay `fees` in `quote` tokens while preserving `base` tokens.
`Bidders`, however, can not acquire `base` tokens.
2. `Strategic Bidding`
Alternatively, the `creator` can `bid` at an appropriate price for optimal results.
For instance, consider `bids (q1, p1), (q2, p2), (q3, p3) ...` where `p1 > p2 > p3`.
if `q1 + q2` doesn't fill the `capacity`, the `auction` involves the third `bid`, and the `marginal price` falls to `p3`.
Suppose the `creator` desires `quote` tokens at price `p2`: he can create a `bid` with price `p2` and `quote` tokens to fill the `capacity`.
Consequently, he can acquire `q1 + q2` tokens at price `p2`.
3. `Mitigating Failure`
If `quote` amounts fall short of the required amounts, the `auction` is considered unsuccessful,.
```solidity
function _settle(uint96 lotId_)
    internal
    override
    returns (Settlement memory settlement_, bytes memory auctionOutput_)
{
    if (
        result.capacityExpended >= auctionData[lotId_].minFilled
            && result.marginalPrice >= lotAuctionData.minPrice
    ) { }
}
```
To prevent this, the `auction creator` can create a `bid` with a small amount of `quote` tokens to fill the `auction`.
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L791-L794
## Tool used

Manual Review

## Recommendation
```solidity
- mapping(uint96 lotId => mapping(uint64 bidId => EncryptedBid)) public encryptedBids;
+ mapping(uint96 lotId => mapping(uint64 bidId => EncryptedBid)) private encryptedBids;
```
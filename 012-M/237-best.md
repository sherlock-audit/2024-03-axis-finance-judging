Dazzling Chocolate Reindeer

high

# Settlement of batch auction can exceed the gas limit

## Summary

Settlement of batch auction can exceed the gas limit, making it impossible to settle the auction.

## Vulnerability Detail

When a batch auction (EMPAM) is settled, to calculate the lot marginal price, the contract [iterates over all bids](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L611-L651) until the capacity is reached or a bid below the minimum price is found. 

As some of the operations performed in the loop are gas-intensive, the contract may run out of gas if the number of bids is too high.

Note that additionally, there is [another loop](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L772-L781) in the `_settle` function that iterates over all the remaining bids to delete them from the queue. While this loop consumes much less gas per iteration and would require the number of bids to be much higher to run out of gas, it adds to the problem.

## Impact

Settlement of batch auction will revert, causing sellers and bidders to lose their funds.

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L611-L651

## Proof of concept

Change the minimum bid percent to 0.1% in the `EmpaModuleTest` contract in `EMPAModuleTest.sol`.

```diff
-   uint24 internal constant _MIN_BID_PERCENT = 1000; // 1%
+   uint24 internal constant _MIN_BID_PERCENT = 100; // 0.1%
```

Add the following code to the contract `EmpaModuleSettleTest` in `settle.t.sol` and run `forge test --mt test_settleOog`.

```solidity
modifier givenBidsCreated() {
    uint96 amountOut = 0.01e18;
    uint96 amountIn = 0.01e18;
    uint256 numBids = 580;

    for (uint256 i = 0; i < numBids; i++) {
        _createBid(_BIDDER, amountIn, amountOut);
    }
    
    _;
}

function test_settleOog() external
    givenLotIsCreated
    givenLotHasStarted
    givenBidsCreated
    givenLotHasConcluded
    givenPrivateKeyIsSubmitted
    givenLotIsDecrypted
{        
    uint256 gasBefore = gasleft();
    _settle();

    assert(gasBefore - gasleft() > 30_000_000);
}
```

## Tool used

Manual Review

## Recommendation

An easy way to tackle the issue would be to change the `_MIN_BID_PERCENT` value from 10 (0.01%) to 1000 (1%) in the `EMPAM.sol` contract, which would limit the number of iterations to 100.

A more appropriate solution, if it is not acceptable to increase the min bid percent, would be to change the settlement logic so that can be handled in batches of bids to avoid running out of gas.

In both cases, it would also be recommended to limit the number of decrypted bids that can be deleted from the queue in a single transaction.
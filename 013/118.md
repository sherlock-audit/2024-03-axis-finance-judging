Rare Mercurial Sealion

medium

# Code that Written in Order to Optimize Gas, Instead Nearly Triples Gas Usage

## Summary
Gas refund mechanism misconception creates a significant amount of gas usage while trying to optimize for gas.
## Vulnerability Detail
The settlement process within the auction mechanism is executed by invoking the settle() function in the AuctionHouse. This function trace leads to the _settle() function within the EMPA module. Within this function, an iteration is conducted through all decrypted bids, deleting them individually to facilitate gas refund and prevent potential gas exhaustion during the settlement transaction. This deletion process doesn't provide any value or functionality to function except gas deletion. Here is the process:
```solidity
        // Delete the rest of the decrypted bids queue for a gas refund
        {
            Queue storage queue = decryptedBids[lotId_];
            uint256 remainingBids = queue.getNumBids();
            if (remainingBids > 0) {
                for (uint256 i = remainingBids - 1; i >= 0; i--) {
                    uint64 bidId = queue.bidIdList[i];
                    delete queue.idToBidMap[bidId];
                    queue.bidIdList.pop();

                    // Otherwise an underflow will occur
                    if (i == 0) {
                        break;
                    }
                }
                delete queue.numBids;
            }
        }
```
As we can see this is done in order to reduce gas consumption in settle() function. While intended to reduce gas consumption within the settle() function, this implementation unfortunately introduces more harm than benefit due to a probable misunderstanding of the gas refund mechanism. 

I will start by showing the harm, then will explain why.
To illustrate the adverse effects, execute the *test_largeNumberOfUnfilledBids_gasUsage* test using the command *forge test --mt test_largeNumberOfUnfilledBids_gasUsage -vv*. This protocol-implemented test evaluates the gas usage of the _settle() call when there are 1500 bids within an auction, which returns a result of 3,836,948 gas units.

Now if we modify the EMPAM.sol file and remove the part that is passed above from _settle() function and rerun the test, gas usage will dropped significantly to 1,667,850 gas units.

This shows that the mentioned deletion process consumes an excessive amount of gas, totaling approximately 2.2 million gas units if there are 1500 bids in an auction.

Now let's continue with probable gas refund misconception.
Here are some quotes about gas refund taken from [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf):
>   Ar is the refund balance, increased through using the SSTORE instruction in order to reset contract storage to zero from some non-zero value. Though not immediately refunded, it is allowed to partially offset the total execution costs.

>   Then the state is finalised by determining the amount to be refunded, g∗ from the remaining gas, g′, plus some allowance from the refund counter, to the sender at the original rate.

>    The total refundable amount is the legitimately remaining gas g′, added to Ar, with the latter component being capped up to a maximum of one fifth (rounded down) of the total amount used Tg − g′. Therefore, g∗ is the total gas that remains after the transaction has been executed.

>    The max refundable proportion of gas was reduced from one half to one fifth by EIP-3529 by Buterin and Swende [2021] in the London release.

>    If there is not enough gas remaining to pay this, i.e. g∗∗ < c, then we also declare an out-of-gas exception. The gas remaining will be zero in any such exceptional condition...... However, the value of the transaction is not transferred to the aborted contract’s address when we are out-of-gas, thus the contract’s code is not stored. If such an exception does not occur, then the remaining gas is refunded to the originator and the now-altered state is allowed to persist.

Let me elaborate what I am trying to show:
Primarily, gas refunds via this mechanism (Ar) are constrained to one-fifth of the total gas consumed. Hence in the scenario provided above, after gas refunded, 3836948 can at most drop to 3069559, which leads to nearly two times more gas usage while the goal is reducing the gas usage. We don't see the result in test because gas refunds are done after transaction execution finished, which the provided test does not calculate that (it is hard to calculate in Foundry currently, an improvement for this is proposed and will probably be added to Foundry in the near future), it just calculates gas used during the settle() process without considering refund. Which brings us to next problem.
As we can see from above explanations, gas refunds are done after transaction execution if it is succesful, not in the moment of deleting from queue. Hence if transaction reaches to block gas limit, the refund won't happen and the call will revert.

Now, if we change the number in modifier *givenLargeNumberOfUnfilledBids* from 1500 unfilled bids to 14000 unfilled bids:
```solidity
    modifier givenLargeNumberOfUnfilledBids() {
        // Create 10 bids that will fill capacity
        for (uint256 i; i < 10; i++) {
            _createBid(2e18, 1e18);
        }

        // Create more bids that will not be filled
        // Lower price, otherwise they will be filled first due to ordering
        for (uint256 i; i < 14000; i++) {
            _createBid(19e17, 1e18);
        }

        // Marginal price: 2
        _expectedMarginalPrice = _scaleQuoteTokenAmount(2 * _BASE_SCALE);
        _expectedMarginalBidId = 10;

        _expectedTotalIn = 10 * 2e18;
        _expectedTotalOut = 10 * 1e18;
        _;
    }
```
And  rerun the same test both with this gas optimization, and without that optimization part (again with deleting it), we will reach the following results:
- With optimization = 31,244,540
- Without optimization = 11,025,345

Hence, as a result of the code implemented for gas optimization, we reach to the block gas limit approximately three times faster. Moreover, due to this acceleration, refunds fail to occur because the transaction cannot succeed under these conditions. Consequently, all funds become locked permanently since the auction has concluded, decryption is complete, and the only callable function, **settle()**, cannot execute due to the gas limit.

This is however is not the main focus of this submission, revert due to block gas limit will happen even if this problem solved, but with this "gas optimization", it is reached 3 times faster which further promotes that problem's likelihood (It's submitted seperately in #3 (EMPA's Can be Locked because of Reaching Block Gas Limit)).  

From Sherlock's Criteria for Issue Validity:
>  List of Issue categories that are not considered valid:
Gas optimizations: The user/protocol ends up paying a little extra gas because of this issue.

However, the impact of this function goes beyond a mere increment in gas usage. Despite being designed for gas optimization, it at least doubles the amount of gas paid in every settle() call (nearly triples at some cases). Considering that settlement process itself is inherently gas-expensive, with typical usage costing millions of gas, doubling or tripling these amounts could easily escalate gas usage into tens of millions. Therefore, I believe this issue cannot be categorized as merely paying "a little extra gas".


## Impact
The code implemented for gas optimization results in a significant increase in gas usage, ranging from two to three times more than usual. This escalation often reaches into the millions of gas units and, in certain instances, even tens of millions.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol/#L767-L784
## Tool used

Manual Review

## Recommendation
Remove the Queue deleting part from _settle() function in EMPAM.sol
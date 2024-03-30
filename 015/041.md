Proud Ruby Loris

medium

# Attacker can forbid users to get refunded if sends enough bids on the EMPAM module

## Summary

When an auction starts an attacker can send enough encrypted bids to make future users that bid unable to be refunded.

## Vulnerability Detail

An attacker can send valid bids with amounts equal to the minimum allowed amount for a bid. If enough bids are sent, users that bid after him won't be able to get refunded if they want to. Scenario:

- Attacker sends lots of bids just after auction creation.
- User sends bids
- User wants to refund some of them: The _refundBid function on the EMPAM module loops through all the bids to find the requested one, then pops it out of the decryption array. If there are too many bids before the one we are looking fur, gas can run out.

## Impact
Breaks the refund functionality. User won't be able to refund bid. Possible loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L284-L305

Putting this test on the EMPA refund bid [tests](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/test/modules/auctions/EMPA/refundBid.t.sol) file can show how its performed.

```solidity
    function test_audit_dos_bids() public givenLotIsCreated givenLotHasStarted givenBidIsCreated(2e18, 1e18){
        uint bidNums = 60000;
        //worst case scenario for attack is: max limit for block, only tx in the block. This doesnt take in account the gas spent on entrypoint.
        uint ETH_GAS_LIMIT = 30_000_000;

        // attacker bids
        for (uint i=0;i < bidNums; i++)
            _createBid(1e18, 1e18); //amount can be as small as possible

        uint64 normalUserBid = _createBid(2e18, 1e18);
        uint256 gasBefore = gasleft();

        vm.prank(address(_auctionHouse));
        uint256 refundAmount = _module.refundBid(_lotId, normalUserBid, _BIDDER);
        uint256 gasAfter = gasleft();

        uint256 gasUsed = gasBefore - gasAfter;
        assertEq(gasUsed > ETH_GAS_LIMIT,true, "out of gas");
    }
```

## Tool used

Manual Review

## Recommendation

Instead of looping through each bidId, holding the index position of a non encrypted bid on a mapping should solve it.
The mapping should be from bidId to its position in the array. When a bid is refunded, this position should also be changed for its replacement.
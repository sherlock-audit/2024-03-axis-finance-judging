Rare Mercurial Sealion

medium

# Auction Can Be Locked with It's Funds in Decryption Phase in Mainnet Because Massive Gas Payment is Required From Users

## Summary
The decryption process in the EMPA module presents a significant issue as it demands a massive amount of gas, potentially locking funds within the contract. With the burden of gas costs falling on auction participants (they are the ones that expected to decrypt the auction), there is little to no incentive to complete the decryption process in the mainnet, leading to funds being trapped within the contract.
## Vulnerability Detail
Upon conclusion of a sealed bid batch auction, the settlement process initiates with decryption. It is expected that users will call the decrypt function within the EMPA module, and only after all decryption operations are completed can the settlement proceed.This decryption task is undertaken by users of the protocol without any incentives, except for the incentive to be able to retrieve their amount back if the auction is stuck in the decryption phase, which actually can not be counted as an "incentive".

Decryption process is very expensive in terms of gas such that it doesn't have to be done in one transaction because it can exceed gas limit of a block. This is also documented by protocol in *[design/EMPA.md](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/design/EMPA.md#auction-settlement-part-1-decryption)* as follows:
>    It may take several decryption transactions depending on the number of bids and gas limit. Once, all bids are decrypted on the contract, we can move to part 2 of the settlement process.

While such costs might be acceptable for Layer 2's, they present a serious challenge for mainnet operations. As demonstrated in the provided POC below, even with an auction which has 100 bids in total, the gas cost for decryption escalates to 7 million. Considering the mean gas price in February 2024, which is 40 gwei, and an ETH price of $3500, this translates to nearly $1,000 in gas costs for decryption.

If we consider auctions that have more bids we can reach to more catastrophic scenarios. Here is a demonstration:
Add following test to *test/modules/auctions/EMPA/settle.t.sol*. Modifier **givenLargeNumberOfUnfilledBids** is created by protocol and it has 1510 total bids in it. You can adjust the number of bids in that modifier to test different bid amounts.
```solidity
     function test_largeNumberOfUnfilledBids_gasUsageOfDecryption()
        external
        givenLotIsCreated
        givenLotHasStarted
        givenLargeNumberOfUnfilledBids
        givenLotHasConcluded
        givenPrivateKeyIsSubmitted
    {
        EncryptedMarginalPriceAuctionModule.AuctionData memory auctionData = _getAuctionData(_lotId);
        uint256 gasBefore = gasleft();
        _module.decryptAndSortBids(_lotId, auctionData.nextBidId - 1);
        uint256 gasAfter = gasleft();

        console2.log(gasBefore - gasAfter);
    }
```
Output:
>  [PASS] test_largeNumberOfUnfilledBids_gasUsageOfDecryption() (gas: 321856062)
Logs:
  101789276

In this scenario which has 1510 bids, the gas cost of decryption reached 101 million, hence it needs to be decrypted in at least 4 blocks. The cost of this decryption with 40 gwei gas price is = $14_000 (imagine it during a bullish market season). 

Failure to cover this decryption cost would result in all funds being trapped within the contract. Since there is no way to change the state of the auction once it has concluded, except through decryption, funds cannot be moved. This is an amount that cannot be reasonably expected from users to provide, thus leading to the likelihood of numerous auctions becoming stuck in the decryption phase on the mainnet.

One side note is that settle() process that comes after decryption also is a process that is costly in terms of gas. It worth nearly 1/4 of decryption process. Since decryption phase is costly enough to cause an issue, I didn't go into details of settle() process and it's gas costs in submission.
## Impact
Funds belonging to both buyers and sellers will remain trapped within the contract unless someone is willing to pay for a rescue operation. The possibility of a rescue scenario can only be anticipated within auctions where an individual has placed a high-value bid, or it may be expected from the seller. However, in both cases, these users stand to lose a significant amount of funds during the decryption process.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol/#L441
## Tool used

Manual Review

## Recommendation
It would be unfair to suggest changing the decryption process, as it is one of the main functionalities of the system and is necessary for the sealed bid batch auction to function properly. Additionally, limiting bid amounts would not resolve the issue. Therefore, the only feasible solution that comes to mind without requiring extensive changes to the codebase is to refrain from using the mainnet for the EMPA module.
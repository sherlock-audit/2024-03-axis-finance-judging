Fun Carob Whale

high

# Module's gas yield can never be claimed and all yield will be lost

## Summary
Module's gas yield can never be claimed
## Vulnerability Detail
The protocol is meant to be deployed on blast, meaning that the gas and ether balance accrue yield.

By default these yield settings for both ETH and GAS yields are set to VOID as default, meaning that unless we configure the yield mode to claimable, we will be unable to recieve the yield. The protocol never sets gas to claimable for the modules, and the governor of the contract is the auction house, the auction house also does not implement any function to set the modules gas yield to claimable.

```solidity
 constructor(address auctionHouse_) LinearVesting(auctionHouse_) BlastGas(auctionHouse_) {}
```
The constructor of  both BlastLinearVesting and BlastEMPAM set the auction house here 
`BlastGas(auctionHouse_)` if we look at this contract we can observe the above.

BlastGas.sol
```solidity
    constructor(address parent_) {
        // Configure governor to claim gas fees
        IBlast(0x4300000000000000000000000000000000000002).configureGovernor(parent_);
    }
```

As we can see above, the governor is set in constructor, but we never set gas to claimable. Gas yield mode will be in its default mode which is VOID, the modules will not accue gas yields. Since these modules never set gas yield mode to claimable, the auction house cannot claim any gas yield for either of the contracts. Additionally the auction house includes no function to configure yield mode, the auction house contract only has a function to claim the gas yield but this will revert since the yield mode for these module contracts will be VOID.
## Impact
Gas yields will never acrue and the yield will forever be lost
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/blast/modules/BlastGas.sol#L11
## Tool used

Manual Review

## Recommendation
change the following in BlastGas contract, this will set the gas yield of the modules to claimable in the constructor and allowing the auction house to claim gas yield.

```solidity
interface IBlast {
   function configureGovernor(address governor_) external;
   function configureClaimableGas() external; 
}

abstract contract BlastGas {
    // ========== CONSTRUCTOR ========== //

    constructor(address parent_) {
        // Configure governor to claim gas fees
       IBlast(0x4300000000000000000000000000000000000002).configureClaimableGas();
       IBlast(0x4300000000000000000000000000000000000002).configureGovernor(parent_);
    }
}
```
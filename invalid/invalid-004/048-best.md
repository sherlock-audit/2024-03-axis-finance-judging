Strong Juniper Lizard

medium

# WETH and USDB have Automatic yield in the BlastLinearVesting smart contract

## Summary
`Are there any REBASING tokens interacting with the smart contracts?`
Rebasing tokens with strictly increasing token balances should work in the contract, `but extra balances will be accrued by the contract`.

The `BlastLinearVesting` smart contract has automatic yield by default for `WETH` and `USDB`, which means that the token balance of `BlastLinearVesting` will increase over time.

## Vulnerability Detail
Since WETH and USDB accounts have automatic yield by default for both EOAs and smart contracts, the balance of `BlastLinearVesting` will gradually increase due to `WETH` and `USDB` yield. However, this yield cannot be withdrawn, resulting in a loss of yield.
One of these tokens may be held on the `BlastAuctionHouse` smart contract, for example, `minAuctionPeriod`, and then transferred to a derivative (`BlastLinearVesting`), where it may remain much longer, resulting in the protocol not claiming additional yield.
Protocol need to configure claimable yield for USDB and WETH in the `BlastLinearVesting` smart contract.
```solidity
 // Set the yield mode to claimable for the WETH and USDB tokens
        _WETH.configure(YieldMode.CLAIMABLE);
        _USDB.configure(YieldMode.CLAIMABLE);
```

## Impact
The protocol lose additional USDB and WETH yeild in BlastLinearVesting smart contract. 
Broken invariant(`extra balances will be accrued by the contract`) - extra USDB and WETH balances won't be accrued by the contract.
The likelihood of USDB and WETH being used as the base token is low.

## Code Snippet
[src/blast/modules/derivatives/BlastLinearVesting.sol#L10](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/blast/modules/derivatives/BlastLinearVesting.sol#L10)

## Tool used

Manual Review

## Recommendation
Consider configuring claimable yield for USDB and WETH in the `BlastLinearVesting` smart contract:
```diff
contract BlastLinearVesting is LinearVesting, BlastGas {
    // ========== CONSTRUCTOR ========== //

    constructor(address auctionHouse_) LinearVesting(auctionHouse_) BlastGas(auctionHouse_) {
+    _WETH.configure(YieldMode.CLAIMABLE);
+    _USDB.configure(YieldMode.CLAIMABLE);
    }
}
```

Also, `claimYeild` should be implemented in `BlastLinearVesting` smart contract:
```diff
+function claimYeild() external {
+      // Claim the yield for the WETH and USDB tokens and send to protocol
+       _WETH.claim(_protocol, _WETH.getClaimableAmount(address(this)));
+       _USDB.claim(_protocol, _USDB.getClaimableAmount(address(this)));   
+   }
```


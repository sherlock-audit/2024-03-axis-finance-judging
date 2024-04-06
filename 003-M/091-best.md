Sharp Hazel Barracuda

medium

# BlastAuctionHouse contract has wrong WETH and USDB addresses as immutables which leads to lost yield

## Summary
The `BlastAuctionHouse` contract has WETH and USDB addresses as constants assigned to the WETH and USDB testnet addresses. In blast developer documentation:
```solidity
contract MyContract {
  // NOTE: these addresses differ on the Blast mainnet and testnet; the lines below are the mainnet addresses
  IERC20Rebasing public constant USDB = IERC20Rebasing(0x4300000000000000000000000000000000000003);
  IERC20Rebasing public constant WETH = IERC20Rebasing(0x4300000000000000000000000000000000000004);
  // NOTE: the commented lines below are the testnet addresses
  // IERC20Rebasing public constant USDB = IERC20Rebasing(0x4200000000000000000000000000000000000022);
  // IERC20Rebasing public constant WETH = IERC20Rebasing(0x4200000000000000000000000000000000000023);
```
In the `BlastAuctionHouse` contract:
```solidity
    /// @notice    Blast contract for claiming gas fees
    IBlast internal constant _BLAST = IBlast(0x4300000000000000000000000000000000000002);

    /// @notice    Address of the WETH contract on Blast
    IERC20Rebasing internal constant _WETH =
        IERC20Rebasing(0x4200000000000000000000000000000000000023); //@audit this is a testnet addresses
        
    /// @notice    Address of the USDB contract on Blast
    IERC20Rebasing internal constant _USDB =
        IERC20Rebasing(0x4200000000000000000000000000000000000022); //@audit

```
This may result in unexpected behavior in smart contracts. If the constructor wouldn't revert, then at worst the yield for WETH and USDB would be lost.
## Vulnerability Detail
See summary
## Impact
DoS of contract creation or lost yield
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/blast/BlastAuctionHouse.sol#L32-L44
## Tool used

Manual Review

## Recommendation
Check for the correct addresses at:
https://docs.blast.io/building/guides/weth-yield
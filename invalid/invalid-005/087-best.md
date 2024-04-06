Rhythmic Velvet Mockingbird

medium

# reentrancy in `claimProceeds`

## Summary
`AuctionHouse` updates `routing.funding` after sending funds via call to `_sendPayment` in the function `claimProceeds`. The receiver may reenter the code to invoke call to `_sendPayment` multiple times.

## Vulnerability Detail
```solidity
File: src/AuctionHouse.sol

/// @audit ******************* Issue Detail *******************
Reentrancy (loss of funds) in AuctionHouse.claimProceeds(uint96,bytes) (src/AuctionHouse.sol#570-619):
	External calls:
	- (purchased_,sold_,payoutSent_) = _getModuleForId(lotId_).claimProceeds(lotId_) (src/AuctionHouse.sol#578-579)
	State variables written after the call(s):
	- routing.funding -= prefundingRefund (src/AuctionHouse.sol#606)

/// @audit ************** Possible Issue Line(s) **************
	L#578-579,  L#606,  

/// @audit ****************** Affected Code *******************
 578:         (uint96 purchased_, uint96 sold_, uint96 payoutSent_) =
 579:             _getModuleForId(lotId_).claimProceeds(lotId_);
 600:             _sendPayment(routing.seller, totalInLessFees, routing.quoteToken, routing.callbacks);
 606:             routing.funding -= prefundingRefund;
```

## Impact
Attacker may claim/ withdraw more than allowed funds.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L570-L619

## Tool used
Manual Aided Review

## Recommendation
Code should follow the best-practice of [check-effects-interaction](https://blockchain-academy.hs-mittweida.de/courses/solidity-coding-beginners-to-intermediate/lessons/solidity-11-coding-patterns/topic/checks-effects-interactions/), where state variables are updated before any external calls are made. Doing so prevents a large class of reentrancy bugs.
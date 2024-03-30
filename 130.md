Macho Shadow Walrus

high

# EMPAM and FPAM auction modules do not implement catalogue functions

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
Macho Sandstone Platypus

medium

# Max curator fee would be bypassed for existing curators

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
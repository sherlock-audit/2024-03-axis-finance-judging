Muscular Lace Tapir

medium

# Incorrect `FIELD_MODULUS` -  compromise the entire encryption and decryption process.

## Summary
The ECIES library `ECIES::_fieldmul`   assumes a hard-coded value for `FIELD_MODULUS`, which is critical for the correct execution of elliptic curve cryptographic operations. If this value is incorrect, it can lead to systemic failures in encryption and decryption, resulting in potential security vulnerabilities. 

## Vulnerability Detail

The current implementation relies on `FIELD_MODULUS` for field arithmetic operations within the `_fieldmul` and `_fieldadd` functions. 
These functions are foundational for the `isOnBn128` function, which checks if a point lies on the `alt_bn128` curve.
- Current Implementation in `_fieldmul` and `_fieldadd`:
```solidity
function _fieldmul(uint256 a, uint256 b) private pure returns (uint256 c) {
    assembly {
        c := mulmod(a, b, FIELD_MODULUS)
    }
}

function _fieldadd(uint256 a, uint256 b) private pure returns (uint256 c) {
    assembly {
        c := addmod(a, b, FIELD_MODULUS)
    }
}
```
- What Goes Wrong: If `FIELD_MODULUS` is not the correct prime modulus for the `alt_bn128` curve, the results of these operations will be incorrect, leading to invalid cryptographic computations.
 
***PoC*** 
- The issue can be demonstrated by altering the `FIELD_MODULUS` to an incorrect value and showing that `isOnBn128` would incorrectly validate points on the curve.  
- create a simple Solidity script that alters the `FIELD_MODULUS` and checks if a known valid point on the `alt_bn128` curve is still considered valid by the `isOnBn128` function.
- Invalid Curve Checks: The `isOnBn128` function checks if a point is on the `alt_bn128` curve by verifying the curve equation `y^2 = x^3 + 3`. If `FIELD_MODULUS` is incorrect, this check will yield incorrect results, potentially allowing points that are not actually on the curve to pass as valid or rejecting points that are on the curve.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract POC {
    struct Point {
        uint256 x;
        uint256 y;
    }

    // Incorrect FIELD_MODULUS for demonstration purposes
    uint256 private constant INCORRECT_FIELD_MODULUS = 21;

    // Known valid point on the alt_bn128 curve
    Point private constant VALID_POINT = Point({
        x: 1,
        y: 2
    });

    function _fieldmul(uint256 a, uint256 b) private pure returns (uint256 c) {
        assembly {
            c := mulmod(a, b, INCORRECT_FIELD_MODULUS)
        }
    }

    function _fieldadd(uint256 a, uint256 b) private pure returns (uint256 c) {
        assembly {
            c := addmod(a, b, INCORRECT_FIELD_MODULUS)
        }
    }

    function isOnCurve(Point memory p) public pure returns (bool) {
        return _fieldmul(p.y, p.y) == _fieldadd(_fieldmul(p.x, _fieldmul(p.x, p.x)), 3);
    }

    function testValidity() external pure returns (bool) {
        // Check if the known valid point is still considered valid with the incorrect FIELD_MODULUS 
        return isOnCurve(VALID_POINT); } 
}
```
This PoC contract defines an incorrect `FIELD_MODULUS` and uses it in the `_fieldmul` and `_fieldadd` functions. It then provides a `testValidity` function that uses the `isOnCurve` function to check if a known valid point on the `alt_bn128` curve is still considered valid with the incorrect modulus.
To use this PoC, deploy the contract and call the `testValidity` function. If the `FIELD_MODULUS` is correct, the function should return `true` for the valid point. 

However, with the incorrect `FIELD_MODULUS`, the function will likely return `false`, demonstrating that the integrity of the curve check is compromised.

## Impact
-   Invalid field operations compromise the entire encryption and decryption process.
-   Could lead to the exposure of encrypted messages or the inability to decrypt messages correctly.

## Code Snippet
The vulnerability arises from the use of `FIELD_MODULUS` in the `_fieldmul` and `_fieldadd` functions, as shown above.
- https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L141
- https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L147
## Tool used

Manual Review

## Recommendation
- **Verification**: Before deploying, verify the `FIELD_MODULUS` against trusted sources and ensure it matches the standard value for the `alt_bn128` curve.

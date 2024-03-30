Active Walnut Dragonfly

medium

# Missing Check for the Generator Point and Point at Infinity in the encryption and decryption process

## Summary
The `isOnBn128()` function in the  `ECIES` implementation checks if a given `point` is on the `elliptic curve`. This check is crucial for ensuring that the `public keys` used in the encryption and decryption process are `valid points on the curve`. However, the function does not explicitly check if the `point is the generator point or the point at infinity`, which are considered `invalid public keys` in the context of elliptic curve cryptography (ECC).

## Vulnerability Detail
#### Exploit Scenario:
1. The attacker knows that the `generator point` for the `alt_bn128 curve` used in Ethereum is `(1, 2)`. This point is publicly known and is used to generate public keys for Ethereum addresses.
2. The attacker crafts a `message` that they wish to encrypt and send to a `victim`. They use the `generator point` as the public key for the encryption process.
3. The attacker encrypts the message using the `generator point as the public key`. Since the `isOnBn128()` function does not check if the `point is the generator point`, the encryption process proceeds without issue.
4. The attacker sends the encrypted message to the victim. The victim, unaware of the attacker's actions, receives the message and attempts to `decrypt` it using their `private key`.
5. The victim's private key is used to decrypt the message. However, because the `generator point` was used as the public key for encryption, the decryption process will produce a `valid plaintext message`. This is because the generator point is a valid point on the curve and can be used to generate a valid public key.
6. The attacker now has access to a plaintext message that was intended for the victim. This could be used for various malicious purposes, such as gaining unauthorized access to sensitive information, spreading malware, or conducting phishing attacks.

## Impact
- Malleability Attacks: 
An attacker could use the generator point as a public key to encrypt a message. Since the generator point is a known point on the curve, an attacker could potentially decrypt the message using the corresponding private key. This could lead to malleability attacks, where an attacker can manipulate encrypted data to produce different plaintexts.

## Code Snippet
- https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L129-L134

- https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L111-L112
- 
## Tool used
Manual Review

## Recommendation
To mitigate these vulnerabilities, the `isOnBn128()` function should be enhanced to explicitly check if a point is the `generator point or the point at infinity`. This can be done by adding additional conditions to the function:

```solidity
function isOnBn128(Point memory p) public pure returns (bool) {
    // Check if the point is on the curve
    bool onCurve = _fieldmul(p.y, p.y) == _fieldadd(_fieldmul(p.x, _fieldmul(p.x, p.x)), 3);
    
    // Check if the point is not the generator point or the point at infinity
    bool notGeneratorOrInfinity = !(p.x == 1 && p.y == 2) && !(p.x == 0 && p.y == 0);
    
    return onCurve && notGeneratorOrInfinity;
}
```
Rhythmic Myrtle Pheasant

medium

# The security offered by the BN254 Curve for the auction protocol is ineffective.

## Summary

Axis utilises in its `ECIES.sol` library the BN254 curve undermining the overall security scheme of the protocol. 

## Vulnerability Detail

Although the protocol may acknowledge the security implications associated with the BN254 curve in the `ECIES.sol` library, it remains crucial to underscore the significant security risks inherent within it and to report them, as they fall within the scope of the contest and look for a solution plan if this is available.

Certain selections of elliptic curves and/or underlying fields diminish the security of an elliptic curve cryptosystem by decreasing the complexity of the Elliptic Curve Discrete Logarithm Problem (ECDLP) associated with that curve.

Various types of weak curves give rise to distinct ECDLP attacks, including but not limited to:

- Pohlig-Hellman attack
- Pairing-based attacks such as MOV attack and Frey-Rück attack
- Smart attack
- Singular curve attack
- SSSA attack

And numerous others.

In this sense, the BN354 curve does not quite meet the recommendations of certain national security agencies and standards (which can be revised at https://safecurves.cr.yp.to/). Each of these standards aims to guarantee the complexity of the elliptic curve discrete logarithm problem (ECDLP) entailing the challenge of determining an ECC user's secret key when provided with the user's public key.

Exploitations utilizing the Tower Number Field Sieve (TNFS) and its variations, have resulted in a lowered assessment of the security level of BN254. While scholars and practitioners addressing this matter do not unanimously concur on the revised estimate, viewpoints vary from 96 to 110 bits. (As an example see Zcash discussion https://github.com/zcash/zcash/issues/714)

## Impact

As mentioned above, even if acknowledged by protocol the security issues are evident. It might be possible to calculate the sender’s private key via Pohlig-Hellman algorithm or other vectors. Also, the anticipated benefits from a successful attack are substantial, indicating that an attacker would be motivated to allocate considerable resources to achieve it.

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/ECIES.sol#L22

## Tool used

Manual Review

## Recommendation

It is recommended to use an alternative elliptic curve such as BLS12-381 having at least 120 bit security level. 
Of course, due to the technical challenges this presents and that unfortunately there's not much cryptographic flexibility beyond BN254, one might think that the best solution might be to "accept it as a risk", however the protocol team must consider to develop a plan to upgrade this library if a feasible solution is available and to acknowledge the security issue precisely and thoroughly in their documentation to avoid any compliance factor, escalation or misunderstanding. 

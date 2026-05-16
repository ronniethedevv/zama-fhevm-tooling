# Anti-Patterns

The common fhEVM mistakes, each with its symptom. The symptom matters: some mistakes revert loudly, some fail silently, and some leak data while appearing to work. The dangerous ones are the silent and leaking ones.

## ACL: skipping sender authorization on encrypted inputs

**Mistake:** using a user-supplied ciphertext without first checking `FHE.isSenderAllowed(...)`.

**Symptom:** **leaks data.** An attacker can infer information from whether the call succeeds or fails. The confidential ERC20 case shows attackers inferring balances from whether a transfer succeeds. This is the most dangerous class of bug because the tool looks like it works.

**Fix:** check `FHE.isSenderAllowed(encryptedValue)` before using any caller-supplied ciphertext whose success or failure is observable.

## ACL: allowing the user but not the contract

**Mistake:** calling `FHE.allow(value, msg.sender)` without also calling `FHE.allowThis(value)`.

**Symptom:** **fails.** User decryption does not succeed unless both the user and the contract are authorized.

**Fix:** for any value a user must decrypt, grant both: `FHE.allow(value, user)` and `FHE.allowThis(value)`.

## Decryption: branching on a decrypted result without verifying signatures

**Mistake:** accepting a public-decrypted cleartext and acting on it without calling `FHE.checkSignatures(...)`.

**Symptom:** **reverts** when the proof is invalid, the cleartext mismatches, or the ciphertext/cleartext pair is wrong. (And if checks were skipped entirely, an unverified value would be trusted.)

**Fix:** always call `FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof)` before decoding or using a decrypted value. Add replay protection.

## Branching: using `if`, `while`, or `break` on an encrypted condition

**Mistake:** trying to drive Solidity control flow with an `ebool`. Breaking a loop on an encrypted condition, or using an encrypted index to pick an array element.

**Symptom:** **does not work.** Solidity control flow cannot depend on an encrypted value. Encrypted array indexing is also extremely gas-heavy.

**Fix:** use `FHE.select` for branching between encrypted values, and fixed-length loops. If public branching is genuinely required, decrypt off-chain first.

## Type choice: oversized types, or expecting overflow reverts

**Mistake:** using `euint256` for small values, or assuming encrypted arithmetic reverts on overflow.

**Symptom:** **fails silently on overflow.** Encrypted arithmetic wraps around; it does not revert. Oversized types do not break anything but waste gas.

**Fix:** pick the smallest type that fits the value range. To guard against overflow, compute an explicit encrypted check (for example, compare the post-operation value and use `FHE.select` to keep or discard the update).

## Division: `div` or `rem` with an encrypted right-hand side

**Mistake:** dividing an encrypted value by another encrypted value.

**Symptom:** **panics.** `div` and `rem` require a plaintext right-hand operand.

**Fix:** ensure the divisor is plaintext. If the divisor must be secret, redesign the computation.

## Summary table

| Anti-pattern | Symptom |
|---|---|
| No `isSenderAllowed` on user ciphertext | Leaks data via inference |
| `allow` user but not contract | User decryption fails |
| Branch on decrypted value without `checkSignatures` | Reverts / trusts unverified data |
| `if`/`while`/`break` on encrypted condition | Does not work |
| Oversized type or expecting overflow revert | Fails silently on overflow; wastes gas |
| `div`/`rem` by encrypted value | Panics |

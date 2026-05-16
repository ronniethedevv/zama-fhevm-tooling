# Revealing Results Safely

A confidential tool should reveal only its final result, never its inputs. When a public answer is genuinely needed (a clear pass/fail rather than an encrypted `ebool`), use the async public decryption flow and mark only the final handle as decryptable.

## When you need this

You do NOT need this if the tool can return an encrypted `ebool` and let the consumer handle it encrypted. Returning the `ebool` is the most private option and keeps everything synchronous.

You DO need this when the outcome must become a clear, public value: for example, a public if/else must run based on the result, or an off-chain caller needs a plain `true`/`false`.

## The flow

1. **On-chain:** mark only the final result handle as publicly decryptable.
   ```solidity
   FHE.makePubliclyDecryptable(finalBoolHandle);
   ```
2. **Off-chain:** call `publicDecrypt([finalBoolHandle])`, passing a single handle.
3. **Off-chain:** receive the returned `abiEncodedClearValues` and `decryptionProof`.
4. **On-chain:** submit those back and verify before using them.

## Decrypting just the final `ebool`

Pass a single handle to `publicDecrypt`. Then, on-chain, verify only that handle list:

```solidity
FHE.checkSignatures(handlesList, abiEncodedCleartexts, decryptionProof);
bool result = abi.decode(abiEncodedCleartexts, (bool));
```

Because only the final boolean handle was made decryptable, the inputs remain confidential. Nothing else can be decrypted through this flow.

## What the callback contract must do

The contract function that receives the decrypted result must:

1. Accept `handlesList`, `abiEncodedCleartexts`, and `decryptionProof`.
2. Call `FHE.checkSignatures(...)` to verify the proof.
3. Decode the cleartext only after verification succeeds.
4. Include replay protection. This is explicitly required. Without it, a valid decryption response could be replayed.

## Latency

The workflow is asynchronous. There is no fixed latency figure. The off-chain decrypt step must complete before the result can be resubmitted and verified on-chain. Design the tool as a request plus a callback, not a single synchronous call, and set that expectation with the user.

## Anti-pattern reminder

Never branch on a decrypted cleartext without calling `FHE.checkSignatures(...)` first. Accepting an unverified cleartext means an invalid proof, a mismatched cleartext, or a wrong ciphertext/cleartext pair goes undetected. See `anti-patterns.md`.

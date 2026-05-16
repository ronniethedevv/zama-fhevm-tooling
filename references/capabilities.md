# Capabilities and Hard Limits

What fhEVM can and cannot compute on encrypted data. Read this before designing any tool.

## Operations on `euintX` (encrypted integers)

- **Arithmetic:** `add`, `sub`, `mul`, `min`, `max`, `neg`
- **Bitwise:** `and`, `or`, `xor`, `not`, `shl`, `shr`, `rotl`, `rotr`
- **Comparisons:** `eq`, `ne`, `ge`, `gt`, `le`, `lt`
- **Ternary:** `select`
- **Random:** `randEuintX()` / `randBounded` where supported

## Operations on `ebool` (encrypted booleans)

- **Boolean-style ops:** `and`, `or`, `xor`, `not`
- **Equality:** `eq`, `ne`
- **Ternary:** `select`
- **Random:** supported for booleans

## What comparisons return

Every comparison function (`eq`, `ne`, `ge`, `gt`, `le`, `lt`) returns an **`ebool`**, an encrypted boolean. It is never a clear Solidity `bool`.

That `ebool` is consumed by `FHE.select` for encrypted branching. It cannot be read directly by Solidity control flow.

## Hard limits

These are the boundaries that constrain every tool design:

1. **`div` and `rem` require a plaintext right-hand operand.** You cannot divide by an encrypted value. Doing so panics. If a design needs division by a secret, redesign it.

2. **Comparison overloads with a plaintext left operand invert the comparison** to keep the plaintext operand on the right. Be aware of this when mixing plaintext and ciphertext in a comparison.

3. **Solidity control flow cannot depend on an encrypted condition.** There is no encrypted `if`, no encrypted `while`, no encrypted `break`. The only way to branch on a secret is `FHE.select`, which chooses between two encrypted values.

4. **To branch into non-encrypted (public) logic, you must decrypt off-chain first.** This means an async public decryption round trip. It is not a synchronous operation.

## Mixed plaintext and encrypted operands

- Some operations accept plaintext operands through overloads.
- Bitwise overloads trivially encrypt the plaintext operand first.
- Ciphertext-plaintext (scalar) operations are generally cheaper than ciphertext-ciphertext operations. Prefer scalar operands when a value can be public.

## Design implication

A confidential tool is built entirely from: convert input, compute with the operations above, compare to produce an `ebool`, optionally `select` on that `ebool`, and either return the encrypted result or decrypt one final value off-chain. If a task cannot be expressed within those moves, it is not a fit for this skill.

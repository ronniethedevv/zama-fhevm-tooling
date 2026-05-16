# ACL Rules

The Access Control List (ACL) governs who can use and decrypt each ciphertext. Getting it wrong is the most common source of fhEVM bugs: some mistakes revert, others fail silently, others leak data.

## The verification flow

For a tool where a caller submits an encrypted value to be checked:

1. The caller encrypts the input off-chain and submits it as an `externalEuintXX` plus an `inputProof`.
2. The contract calls `FHE.fromExternal(...)` to validate the proof and turn the ciphertext bytes into an on-chain handle.
3. Before using the value, the contract checks `FHE.isSenderAllowed(encryptedValue)`. This blocks unauthorized callers and prevents inference attacks.
4. The contract computes on the handle and authorizes any result handle it wants to keep or return.

## The three grant functions

- **`FHE.allow(ciphertext, address)`** — grants `address` permanent permission to decrypt that ciphertext.
- **`FHE.allowThis(ciphertext)`** — shorthand for `FHE.allow(ciphertext, address(this))`. Grants the current contract permanent permission to use that ciphertext, including reusing the handle in future transactions.
- **`FHE.allowTransient(ciphertext, address)`** — grants access for the current transaction only. It does not persist.

## Who needs which permission

- For a contract to **reuse a handle across transactions**, the handle needs `FHE.allowThis(...)`.
- For a **specific user to decrypt** a result, the ciphertext needs `FHE.allow(ciphertext, user)`.
- For **user decryption to succeed, both are required**: the ciphertext must be allowed for the user AND for the contract. Missing `allowThis` makes user decryption fail.
- For **anyone to verify or decrypt** against a ciphertext, use `FHE.makePubliclyDecryptable(ciphertext)`.

## How a contract computes on a ciphertext it did not create

The contract never sees plaintext. It works on **handles**, which are references to ciphertexts. The ACL authorizes the handle. The executor/coprocessor performs the actual FHE operations on the underlying ciphertext. The contract only orchestrates symbolic execution.

This is why a caller can submit an encrypted value and the contract can compare it without ever learning what it is.

## Handle persistence and reuse

Encrypted handles are stored as ciphertext references in contract state. A handle can stay in storage and be reused later, across transactions and across callers, as long as it has a permanent allowance.

- `FHE.allowTransient(...)` works only for the current transaction.
- `FHE.allow(...)` grants long-term access.
- The same stored ciphertext can be reused by multiple callers, but each caller that needs to verify or decrypt against it must be granted explicitly with `FHE.allow(ciphertext, caller)`.
- For a "store once, verify many times" tool, store the handle, keep it permanently allowed for the contract with `FHE.allowThis`, and grant individual callers as needed. For fully public verification, use `FHE.makePubliclyDecryptable`.

## What breaks when a permission is missing

- **No `isSenderAllowed` check** on a user-supplied ciphertext: data can leak through success/failure inference. An attacker learns information from whether the call succeeds.
- **`allow` the user but not the contract** (missing `allowThis`): user decryption fails.
- **Result handle not authorized** with `allowThis`: the contract cannot reuse the handle in a later transaction.
- **No permission for a caller** trying to verify against a stored handle: `isSenderAllowed` fails for that caller, and access attempts are rejected at the contract level.

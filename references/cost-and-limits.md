# Cost and Limits

fhEVM operations are metered in HCU (Homomorphic Complexity Units), not by a fixed count of encrypted operations per transaction. Designing a tool that fits the budget means knowing which operations are expensive.

## The per-transaction budget

- **20,000,000 HCU per transaction** (current devnet limit).
- **5,000,000 HCU depth per transaction.**

If either limit is exceeded, the transaction reverts. There is no partial execution.

## Operation cost, cheapest to most expensive

1. **Bitwise ops** — usually cheapest.
2. **Comparisons** — moderate.
3. **`select`** — moderately heavy.
4. **`mul`** — expensive.
5. **`div` and `rem`** — most expensive. (And remember: they require a plaintext divisor anyway.)

Cost also rises with bit-width: `euint8` is cheaper than `euint256` for the same operation.

## Concrete figures

`euint64`:
- `mul`: 365,000 HCU scalar, 596,000 non-scalar
- `div`: 715,000
- `rem`: 1,153,000
- `eq`: 83,000
- `select`: 55,000

`euint128`:
- `mul`: 696,000 scalar, 1,686,000 non-scalar
- `div`: 1,225,000
- `rem`: 1,943,000
- `select`: 57,000

## Design rules for tools

A tool that does several encrypted checks per call is usually feasible, as long as the checks are mostly comparisons and kept small. To stay well inside the budget:

- **Use the smallest encrypted type that fits.** A counter that never exceeds a few million fits in `euint32`. Do not reach for `euint64` or `euint256` by default.
- **Prefer comparisons over multiplication.** A Gate built on `FHE.ge` is cheap. A Scorer that multiplies several encrypted inputs is far heavier.
- **Prefer scalar (plaintext) operands.** Ciphertext-plaintext operations are cheaper than ciphertext-ciphertext. If one side of an operation can be public, make it public.
- **Count the operations per call.** A Verifier doing one `eq` is trivial. An allowlist Verifier doing fifty `eq` plus fifty `or` is fifty times heavier. Estimate before building.

## When a design does not fit

If a tool needs heavy per-call computation (many multiplications, large types, long allowlists checked exhaustively), it may not fit the HCU budget. When a design is near the edge, say so to the user and propose a lighter alternative rather than shipping something that reverts.

---
name: zama-fhevm-tooling
description: "Design and build confidential verification tools, gates, meters, and scorers on Zama's fhEVM. Use this skill when a task involves checking, gating, metering, or scoring data that should stay encrypted, and where only a small result (a yes/no or a number) should ever be revealed. Triggers include: encrypted key or credential verification, private allowlist or membership checks, confidential eligibility gating, encrypted usage metering or rate limiting, private scoring against a threshold, confidential wallet or address checks, and any tool where the caller submits a value that a contract must evaluate without learning what it is. This skill is for building tools (verifiers, gates, meters, scorers), not for confidential token transfers, payments, or auctions. If the task is a transfer or needs public if/else control flow driven by an encrypted value, see the constraints in this skill before proceeding."
---

# Zama fhEVM Tooling

This skill teaches an agent to design and build **confidential tools** on Zama's fhEVM: contracts that evaluate encrypted data and reveal only a small result. It covers four reusable patterns and points to reference files for depth.

A confidential tool takes an encrypted input, computes on it without ever decrypting it on-chain, and returns either an encrypted boolean (kept private) or, when a public answer is genuinely needed, a single decrypted result produced through a verified async flow. The caller's actual data is never exposed.

## When this skill applies

Use this skill when the task fits this shape:

1. A caller submits a value (a key, a score, an address, a count, a measurement).
2. A contract must evaluate that value against a rule or a stored reference.
3. Only the outcome of the evaluation should be visible. The input itself must stay hidden.

Typical fits: encrypted credential verification, private allowlist or membership checks, confidential eligibility gating, encrypted usage metering, private threshold scoring, confidential address or wallet checks.

## When this skill does NOT apply

Stop and reconsider if any of these is true:

- **The task is a token transfer or payment.** That is confidential-token territory, not tooling. A different design applies.
- **The task needs real Solidity `if`/`while`/`break` driven by an encrypted value.** This is impossible on fhEVM. Solidity control flow cannot depend on an encrypted condition. The only branching available on ciphertext is `FHE.select`, which chooses between two encrypted values. If the task genuinely needs public branching, that requires an off-chain public decryption step first (see `references/revealing-results.md`), which adds latency and an async round trip. Flag this to the user before building.
- **The computation needs division or remainder by an encrypted value.** `div` and `rem` require a plaintext right-hand operand. Dividing by a ciphertext is not supported and will panic. If the design needs division by a secret, redesign it.
- **The data or computation is large.** fhEVM operations are metered by HCU and there is a per-transaction budget. Tools that do a handful of comparisons per call are fine. Tools that need heavy multiplication, large bit-widths, or many operations per call may exceed the budget. See `references/cost-and-limits.md`.

If the task is borderline, explain the constraint to the user honestly rather than building something that fails silently.

## Core capability summary

fhEVM can compute on encrypted integers (`euintX`) and encrypted booleans (`ebool`) without decrypting them. Available operations include arithmetic (`add`, `sub`, `mul`, `min`, `max`, `neg`), bitwise ops, comparisons (`eq`, `ne`, `ge`, `gt`, `le`, `lt`), and the ternary `select`.

Two facts shape every tool you build:

- **Every comparison returns an `ebool`, not a clear `bool`.** You cannot read it directly in Solidity. You either keep it encrypted and return it, or you run it through the async public decryption flow to get a clear value.
- **`FHE.select(condition, ifTrue, ifFalse)` is the only branching on ciphertext.** It picks between two encrypted values based on an encrypted condition. There is no encrypted `if`.

Full detail is in `references/capabilities.md`. Read it before designing any tool.

## The four tooling patterns

Every confidential tool in scope for this skill is one of these four patterns, or a composition of them. Identify which pattern the task is, then build from the canonical shape.

### Pattern 1: Verifier

**Use when:** the tool checks whether a submitted encrypted value matches a stored encrypted reference. Encrypted key verification, credential checks, allowlist or membership checks.

**Shape:** convert the external input with `FHE.fromExternal`, compare it to the stored ciphertext with `FHE.eq`, authorize the result handle, and return the `ebool`. Both the submitted value and the stored value stay hidden.

```solidity
function verify(externalEuint64 input, bytes calldata proof) external
returns (ebool) {
    euint64 candidate = FHE.fromExternal(input, proof);
    ebool matches = FHE.eq(storedValue, candidate);
    FHE.allowThis(matches);
    return matches;
}
```

For an allowlist, compare the candidate against each stored entry with `FHE.eq`, then combine the resulting `ebool` values with encrypted logic (`FHE.or`).

Returning the `ebool` keeps both inputs confidential. Only convert it to a clear `bool` if a public answer is genuinely required, using the flow in `references/revealing-results.md`.

### Pattern 2: Gate

**Use when:** the tool decides pass or fail by comparing an encrypted value to a threshold. Eligibility gating, access control on an encrypted attribute.

**Shape:** a Gate is a Verifier whose comparison is a threshold check (`FHE.ge`, `FHE.gt`, `FHE.le`, `FHE.lt`) instead of an equality. It produces an `ebool` that downstream logic consumes through `FHE.select`. The decision stays encrypted.

```solidity
ebool passes = FHE.ge(score, threshold);
```

The `passes` handle is then used as the condition in a `FHE.select` to choose between encrypted outcomes, or returned directly.

### Pattern 3: Meter

**Use when:** the tool tracks a running encrypted count and checks it against an encrypted limit. Usage metering, encrypted rate limiting, quota enforcement.

**Shape:** keep both the counter and the limit as encrypted state. On each call, add the encrypted increment with `FHE.add`, compare the new total to the limit with `FHE.le`, and use `FHE.select` to commit the new count only if it is within budget. Return the `ebool` (allowed or denied). The running count is never revealed because no decryption permission is granted for it.

```solidity
euint32 private _count;
euint32 private _limit;

function useQuota(externalEuint32 delta, bytes calldata proof)
external
returns (ebool)
{
    euint32 inc = FHE.fromExternal(delta, proof);
    euint32 next = FHE.add(_count, inc);
    ebool allowed = FHE.le(next, _limit);
    _count = FHE.select(allowed, next, _count);
    FHE.allowThis(_count);
    return allowed;
}
```

Keep `_count` private by never granting it a decryption permission. `FHE.allowThis(_count)` lets the contract reuse the handle on the next call but does not make it readable by anyone.

### Pattern 4: Scorer

**Use when:** the tool computes a score from encrypted inputs, then checks that score against a threshold and produces an encrypted result. Private scoring, confidential risk or eligibility assessment.

**Shape:** a Scorer composes computation and a Gate. Compute the encrypted score with arithmetic ops, compare it to the threshold with `FHE.ge`, then use `FHE.select` to keep or zero the result based on the encrypted outcome. Authorize the new handle.

```solidity
function checkScore(
    externalEuint64 encryptedInput,
    bytes calldata inputProof
) external {
    euint64 score = FHE.fromExternal(encryptedInput, inputProof);
    // encrypted threshold check
    ebool passes = FHE.ge(score, threshold);
    // encrypted branch
    adjustedScore = FHE.select(passes, score, FHE.asEuint64(0));
    FHE.allowThis(adjustedScore);
}
```

`FHE.select(condition, valueIfTrue, valueIfFalse)` only branches between encrypted values. A public if/else on the outcome requires off-chain decryption.

## Build checklist

Whatever pattern you build, every confidential tool must do these things. Skipping any of them either fails or leaks data (see `references/anti-patterns.md`).

1. **Convert external input.** Any caller-supplied ciphertext arrives as `externalEuintXX` plus a `bytes` proof. Validate it with `FHE.fromExternal(...)` before use. Detail in `references/client-integration.md`.
2. **Authorize the caller where it matters.** Before using a user-supplied ciphertext in a way whose success or failure is observable, check `FHE.isSenderAllowed(...)`. Skipping this can leak data through inference. Detail in `references/acl-rules.md`.
3. **Authorize every result handle.** Any new ciphertext the contract creates and wants to keep or reuse needs `FHE.allowThis(...)`. Any ciphertext a specific user must decrypt needs `FHE.allow(..., user)`. Both are required for user decryption. Detail in `references/acl-rules.md`.
4. **Reveal only the final result, never the inputs.** If a public answer is needed, mark only the final `ebool` (or final small result) decryptable, run the async public decryption flow, and verify it on-chain with `FHE.checkSignatures(...)` plus replay protection. Detail in `references/revealing-results.md`.
5. **Pick the smallest encrypted type that fits.** Cost rises sharply with bit-width. Use `euint32` for a counter, not `euint256`. Detail in `references/cost-and-limits.md`.
6. **Stay inside the HCU budget.** Keep operations per call small, prefer comparisons over multiplication, prefer scalar (plaintext) operands where possible. Detail in `references/cost-and-limits.md`.

## Reference files

Load the reference file relevant to the step you are on:

- `references/capabilities.md` — every operation fhEVM supports on encrypted types, comparison return types, and the hard limits. Read before designing.
- `references/acl-rules.md` — `allow`, `allowThis`, `allowTransient`, `isSenderAllowed`, handle persistence and reuse, and what breaks when a permission is missing.
- `references/revealing-results.md` — the async public decryption flow for revealing a single final result safely, with signature verification and replay protection.
- `references/client-integration.md` — how an off-chain agent or CLI encrypts a value, what the ZKPoK proof is, which SDK to use, and the on-chain validation step.
- `references/cost-and-limits.md` — HCU metering, the per-transaction budget, which operations are expensive, and concrete cost figures.
- `references/anti-patterns.md` — the catalog of common fhEVM mistakes, each with its symptom (reverts, fails silently, or leaks data).
- `references/network-setup.md` — current networks, testnet, required tooling and package names, and licensing.

## Honest framing for the user

When recommending a confidential tool, be straight about three things:

- **fhEVM hides inputs from everyone, including the contract itself, but it does not hide them from whoever runs the off-chain client that encrypts them.** The encryption happens client-side; whoever controls that client sees the plaintext first.
- **Async decryption adds latency.** Any tool that needs a public result is not a single synchronous call. It is a request plus a callback. Set expectations accordingly.
- **There is a real cost ceiling.** Tools with a few encrypted comparisons per call are practical today. Tools that need heavy computation per call may not fit the HCU budget. When a design is near the edge, say so.

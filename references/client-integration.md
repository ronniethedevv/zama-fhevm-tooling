# Client Integration

How an off-chain agent or CLI tool encrypts a value and submits it to an fhEVM contract. This is the path for tooling, not for a dApp browser UI.

## Which SDK

- **For a tool, CLI, or backend agent:** use `@zama-fhe/sdk`. In Node.js, pair it with `RelayerNode`.
- **For a browser app:** the same SDK offers `RelayerWeb`. React-only hooks like `useEncrypt` are just wrappers around it.

The SDK handles client-side FHE encryption, keypair management, and the submission flow.

## Input encryption flow

1. Create an encrypted input bound to both the contract address and the sender address.
2. Add plaintext values with typed helpers: `addBool`, `add32`, `add64`, `addAddress`, and so on.
3. Call `encrypt()` locally. Encryption happens on the client.
4. Submit the returned handles plus the proof to the contract.

```javascript
const input = sdk.relayer.createEncryptedInput(contractAddress, userAddress);
input.add32(12345);
const encrypted = await input.encrypt();
await contract.foo(encrypted.handles[0], encrypted.inputProof);
```

## What the proof is

Each encrypted input is accompanied by a **ZKPoK** (zero-knowledge proof of knowledge). It proves two things:

- The caller knows the plaintext behind the ciphertext.
- The ciphertext is bound to the intended contract and sender.

When multiple values are encrypted together, they are packed into a single ciphertext with one shared proof.

## On-chain validation

The contract receives the encrypted input as `externalEuintXX`, `externalEbool`, or `externalEaddress`, plus a `bytes inputProof`. It validates and converts the input with:

```solidity
euint64 candidate = FHE.fromExternal(input, proof);
```

Only after `FHE.fromExternal` succeeds does the contract have a usable on-chain handle.

## Trust note

Encryption happens client-side, inside the tool or agent that calls the SDK. That means whoever controls that client sees the plaintext before it is encrypted. fhEVM hides the value from the contract, the chain, and every other party, but not from the encrypting client itself. State this honestly when describing a tool's privacy guarantees.

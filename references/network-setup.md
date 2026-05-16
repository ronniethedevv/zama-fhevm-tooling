# Network Setup

Current networks, tooling, and licensing for building fhEVM tools, as of 2026.

## Networks

For testing:

- **Hardhat** — mock encryption, in-memory. Fastest iteration.
- **Hardhat Node** — mock encryption, persistent.
- **Sepolia Testnet** — real encryption, full stack.

For production:

- **Ethereum Mainnet** — the core Ethereum deployment.

Network presets: `MainnetConfig` (chainId 1), `SepoliaConfig` (chainId 11155111), `HardhatConfig` (chainId 31337).

## Testnet

The testnet is **Sepolia**. The Zama Protocol is currently available on Sepolia, with additional chains planned later. Build and test there before any mainnet deployment.

## Tooling and package names

- **Hardhat plugin:** `@fhevm/hardhat-plugin`
- **Solidity package:** `@fhevm/solidity`
- **App SDK:** `@zama-fhe/sdk`
- **React SDK:** `@zama-fhe/react-sdk`

For Sepolia deployments, set `MNEMONIC` and `INFURA_API_KEY`.

For contracts, `ZamaEthereumConfig` is the configuration for Ethereum Mainnet or Ethereum Sepolia.

## Restrictions

- **Cleartext mode is local-development only.** It is blocked on Ethereum Mainnet and Sepolia. Do not design a tool that depends on it outside local testing.
- **Protocol operations can incur fees** for ZKPoK verification, ciphertext decryption, and ciphertext bridging. Fees are paid in `$ZAMA`.

## Licensing

- **Building on the Zama Protocol does not require an extra license.**
- **Forking, copying, or using Zama's technology outside the protocol does require a license.**
- Non-commercial use is free. Commercial use needs an enterprise license, or a protocol that already has one.

Check the current Zama Confidential Blockchain Protocol litepaper for the authoritative and up-to-date terms before any commercial deployment.

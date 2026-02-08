
# Token-Bound Account Project

This repository implements a token-bound account (ERC-6551 style) wallet that is also ERC-4337 compatible and supports a guardian-based social recovery module. The system is designed to be used with Foundry and is tested against a mainnet fork with live EntryPoint, ERC-6551 Registry, USDC, and Chainlink price feed addresses.

## Overview

The project delivers:

- **ERC-6551 token-bound account** behavior so that an ERC-721 token owns a smart account.
- **ERC-4337 account abstraction compatibility**, allowing UserOperations via EntryPoint.
- **Guardian-based recovery** with configurable thresholds and recovery proposals.
- **ERC-20 paymaster** to sponsor gas using USDC and a Chainlink price feed.
- **Foundry test suite** covering success paths and revert branches.

The core wallet is `TBAWallet`, which inherits ERC-4337 validation logic and guardian recovery functionality. It computes the NFT owner dynamically and uses the ERC-6551 “footer” convention to bind the wallet to an NFT. Transactions can be executed directly by the NFT owner or via EntryPoint through UserOperations.

## Contracts (src)

### `ERC4337Compatible`
- A reusable base implementing ERC-4337 `IAccount` validation flow.
- Enforces EntryPoint-only `validateUserOp` entry.
- Handles nonce validation and prefund logic.

### `GuardianCompatible`
- Guardian-based recovery module with voting and threshold enforcement.
- Supports guardian initialization, add/remove, threshold changes, and recovery proposals.
- Tracks recovery proposals per “generation” (counter) with cooldown.
- Expects inheriting contracts to implement ownership checks and NFT transfer logic.

### `TBAWallet`
- ERC-6551-compatible account tied to an ERC-721 token.
- Implements ERC-1271 signature validation and ERC-6551 account interfaces.
- Extends ERC-4337 for UserOperation validation and execution.
- Includes guardian recovery by inheriting `GuardianCompatible`.
- Supports direct owner execution and EntryPoint execution.
- Exposes `approveToken` to allow the NFT owner to approve ERC-20 spending.

### `AAWallet`
- A minimal ERC-4337 account (BaseAccount) with a single EOA owner.
- Supports `execute` and `executeBatch` for transactions.
- Uses `EntryPoint` for validation, deposit handling, and prefunding.
- Upgradeable via UUPS (`UUPSUpgradeable`) and initialized with an owner.
- This contract is purely used as a base example of how a AA Wallet works (It is not part of the main project)

### `AccountFactory`
- Factory for deploying `AAWallet` instances via `ERC1967Proxy` and `Create2`.
- Computes counterfactual addresses so EntryPoint can resolve sender addresses before deployment.

### `ERC20Paymaster`
- A paymaster that charges users in USDC for gas.
- Uses a Chainlink feed (configured in constructor) to estimate token cost.
- Collects USDC upfront in `_validatePaymasterUserOp` and refunds excess in `_postOp`.
- Supports owner withdrawals in USDC and ETH.

### `NFT`
- Simple ERC-721 with configurable base URI, minting, and burning.
- Used to create token-bound accounts in tests.

### `TestContract`
- `CantReceiveEther` helper contract that reverts on receiving ETH.
- Used for negative-path testing of execution failures.

## How It Works

![TBA Wallet Architecture](assets/TBA_Wallet_Architecture.png)

The diagram shows the architecture of the proposed TBA Wallet design. For a basic TBA that follows the ERC6551 pattern, the contract inherits both IERC6551 interfaces, IERC165 and IERC1271 interfaces. The proposed additional designs ERC4337Compatible and GuardianCompatible can also be inherited by the TBA Wallet to achieve Basic AA functions and also guardians to recover the ownership of the TBA Wallet in case of any emergencies.

A TBA Wallet can be created via the ERC-6551 Registry using `TBAWallet` as the implementation contract. The owner of the bound NFT will then have permission to execute the TBA Wallet functions. 

## Why Combine ERC6551 and ERC4337?
This project implements a Token Bound Account (ERC-6551) that inherits / integrates an ERC-4337-compatible execution module. The result is a TBA that can hold assets and represent an NFT on-chain, while also supporting AA features like gas sponsorship, batching, modular auth, and programmable security policies — effectively turning NFTs into portable smart wallets.

Lets look at some of their advantages separately

## Advantages of ERC-6551 (Token Bound Accounts / TBAs)

- **NFTs can own assets and interact on-chain**  
  An NFT can have its own account that can hold ETH, ERC-20s, ERC-721s, ERC-1155s, and can call contracts.

- **Portable identity and state bound to the token**  
  Ownership of the NFT transfers the “account + inventory + permissions” implicitly, without migrating wallets or reconfiguring addresses.

- **Composable “container” pattern**  
  A TBA can act like an on-chain backpack / loadout / vault for a token: the NFT becomes a hub that aggregates assets, roles, memberships, approvals, etc.

- **Supports richer NFT utility**  
  Enables game characters, memberships, AI agents, DAO roles, event tickets, “profile NFTs”, and other NFTs that need their own on-chain presence.

## Advantages of ERC-4337 (Account Abstraction)

- **Smart-contract wallet UX without protocol changes**  
  Works at the mempool layer using `UserOperation`s, bundlers, and an `EntryPoint` contract.

- **Gas sponsorship & flexible fee payment**  
  Pay fees with ERC-20s, sponsor gas for users, or implement custom paymaster rules (great for onboarding and consumer UX).

- **Custom authentication**  
  Plug in different signature schemes (multi-sig, passkeys/WebAuthn via aggregators, session keys, guardians, social recovery, etc.).

- **Batching and atomic multi-calls**  
  Execute multiple actions in one operation (approve + swap + stake, etc.) with a single “sign”.

- **Programmable security policies**  
  Limits, allowlists/denylists, velocity controls, per-dApp permissions, time locks, spend caps, and more.

- **Better recovery flows**  
  More user-friendly recovery than EOAs (guardians, delays, recovery modules) without requiring seed-phrase-only security.


## Advantages of combining ERC-6551 + ERC-4337 for a “TBA Wallet”

By combining ERC-6551 + ERC-4337, the NFT gets its own account (TBA) **and** that account gains AA execution + UX.

- **A token becomes a full smart wallet**  
  The TBA isn’t just an address that can hold assets — it can behave like a modern AA wallet with richer execution and security.

- **Great onboarding for NFT-centered apps**  
  Users can interact “as the NFT” with sponsored gas, simple signing flows, and fewer friction points (no need to pre-fund the TBA with ETH).

- **Session keys for NFTs**  
  Enable gameplay / agent behavior / marketplaces with scoped session permissions (e.g., “this session key can trade item X for 24h”) tied to the NFT’s account.

- **Batching for NFT workflows**  
  Common “character loadout” actions become one atomic op (equip + transfer in + approve + craft), improving UX and reducing failure cases.

- **Transferable account abstraction**  
  Since control follows NFT ownership, you effectively get an AA wallet whose “admin” transfers automatically when the NFT is sold/transferred.

- **Programmable security per NFT**  
  Different tokens can have different policies: high-value NFTs can enforce stricter rules (multisig/guardians), while low-value tokens stay lightweight.

- **Composability with existing AA infrastructure**  
  Bundlers, paymasters, and tooling built for ERC-4337 can be reused for TBAs, reducing custom infra burden.


## Tests (test)

### `AAWallet.t.sol`
- Forks mainnet and validates account creation via EntryPoint `handleOps`.
- Builds and signs a UserOperation for the factory `createAccount` flow.

### `TBAWallet.t.sol`
- Covers ERC-6551 account creation and owner resolution.
- Validates owner-driven `execute` and EntryPoint-driven `execute` flows.
- Tests paymaster usage with USDC, including approval and transfers.
- Exercises batched execution via `executeBatch`.

### `Guardian.t.sol`
- End-to-end guardian lifecycle: initialize, add/remove, threshold updates.
- Tests recovery proposal, voting, and execution to change NFT ownership.
- Verifies guardian reset behavior post-recovery.

### `GuardianBranches.t.sol`
- Extensive revert-branch coverage for guardian initialization and updates.
- Validates access control, operator checks, threshold constraints, and proposal rules.
- Checks owner mismatch paths when NFT owner diverges from recorded owner.

### `TBAWalletBranches.t.sol`
- Branch tests for signature failures and execution failures.
- Verifies `execute` and `executeBatch` guardrails.
- Validates `approveToken` access control.

## External Integrations and Mainnet Addresses

The tests expect a mainnet fork with the following deployed contracts:

- **ERC-4337 EntryPoint**: `0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789`
- **ERC-6551 Registry**: `0x000000006551c19487814612e58FE06813775758`
- **USDC**: `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`
- **Chainlink feed (USDT/ETH)**: `0xEe9F2375b4bdF6387aa8265dD4FB8F16512A1d46`

## Environment Variables

Tests rely on:

- `MAINNET_RPC_URL` for mainnet forking.
- `URI` for the NFT metadata URI used in tests.


## Security Notes

- Guardian operations require the TBA to be approved as an operator of the NFT.
- Recovery resets guardian configuration and increments the recovery counter.
- The paymaster assumes a working Chainlink feed and adequate USDC allowance.

## Project Structure

- src: core contracts
- test: Foundry tests covering functionality and revert branches
- script: Foundry scripts (e.g., NFT deployment)
- dependency/lib: external libraries (account-abstraction, OpenZeppelin, Chainlink, ERC-6551 reference)


# Building on Wormhole Settlement

Wormhole Settlement is meant to serve as the underlying chain abstraction infrastructure layer for protocols across Wormhole connected ecosystems by enabling protocols to bundle call data containing arbitrary protocol actions, which can be executed atomically alongside each transfer. This feature allows developers to create fully chain-abstracted user experiences, including the construction of natively cross-chain decentralized exchanges (DEXs), borrow-lend protocols, payment protocols, and other applications atop of this layer. The following section describes the key smart contract components for teams seeking to build atop of Wormhole Settlement. 

The EVM Token Router is a simple interface to integrate against. For an integrator, there are two main entry points with the contracts: `placeMarketOrder` and `placeFastMarketOrder`. 

### EVM placeFastMarketOrder()

The `placeFastMarketOrder` allows the caller to elect for a ***faster-than-finality*** transfer of USDC (with an arbitrary message payload) to the destination chain by setting the `maxFee` and `deadline` parameters. This interface does NOT guarantee that the caller’s transfer will be delivered faster than finality, however, any willing market participants can compete for the specified `maxFee` by participating in an auction on the Solana `MatchingEngine`. 

```solidity
function placeFastMarketOrder(
        uint128 amountIn,
        uint16 targetChain,
        bytes32 redeemer,
        bytes calldata redeemerMessage,
        uint128 maxFee,
        uint32 deadline
    ) external payable returns (uint64 sequence, uint64 fastSequence);
```

This function takes several parameters:
1. `amountIn`: the amount to transfer
2. `targetChain`: target chain ID
3. `redeemer`: redeemer contract address
4. `redeemerMessage`: an arbitrary payload for the redeemer
5. `maxFee`: the maximum fee that the user is willing to pay to execute a fast transfer
6. `deadline`:  The deadline for the fast transfer auction to start. Note: This timestamp should be for the `MatchingEngine` chain (e.g. Solana) to avoid any clock drift issues between different blockchains. Integrators can set this value to 0 to opt out of using a deadline.

It returns a sequence number for the Wormhole Fill message. This function requires the caller to provide a `msg.value` equal to the amount returned by the `messageFee()` function on the IWormhole.sol interface.

### EVM placeMarketOrder()

The `placeMarketOrder` function is a ***wait-for-full-finality*** USDC transfer (with an arbitrary message payload). The Swap Layer, built on top of the Wormhole Settlement, uses this function if the auction on the matching engine for `placeFastMarketOrder` doesn't start within a specific deadline.  

```solidity
function placeMarketOrder(
        uint128 amountIn,
        uint16 targetChain,
        bytes32 redeemer,
        bytes calldata redeemerMessage,
    ) external payable returns (uint64 sequence, uint64 protocolSequence);
```
This function takes several parameters:
1. `amountIn`: the amount to transfer
2. `targetChain`: target chain ID
3. `redeemer`: redeemer contract address
4. `redeemerMessage`: an arbitrary payload for the redeemer

It returns a sequence number for the Wormhole Fill message. This function requires the caller to provide a `msg.value` equal to the amount returned by the `messageFee()` function on the IWormhole.sol interface.


### **Solana Token Router** 

This **Solana Token Router** program manages **minting**, **burning**, and **transferring** tokens on Solana.

1. **Lock/Burn** tokens on Solana, sending a Wormhole message to another chain.
2. **Unlock/Mint** tokens on Solana in response to a Wormhole message from another chain.

**Where It Fits in the Architecture**

- **Hub Chain**: In many Wormhole Settlement designs, Solana is treated as the “hub” where liquidity sits. The Token Router on Solana acts as the local authority for tokens that are minted and burned.
- **Bridging**: When tokens move **out** (Solana → another chain), the router burns the local tokens and posts a Wormhole message. When tokens come **in** (another chain → Solana), the router receives and validates a Wormhole VAA (Verified Action Approval) and mints the tokens. This complements the **auction** or **matching engine** program, which handles bidding logic for “fast bridging.” The Token Router is more about **actual token custody** and bridging instructions.

### Solana Token Router Key Components

- `lib.rs`
    - The program's main entry point (process_instruction) and library references
- `instruction.rs`
    - Defines the **instruction enum** Instruction with variants like Initialize, SetMint, Transfer, BridgeIn, BridgeOut
    - Each variant corresponds to a specific function call
- `processor.rs`
    - Contains the **core logic** for each instruction, including:
    - process_initialize sets up program state
    - process_set_mint sets which mint the router manages
    - process_transfer performs an SPL token transfer
    - process_bridge_in handles bridging tokens **into** Solana (Wormhole redemption)
    - process_bridge_out handles bridging tokens **out** of Solana (Wormhole emission)
- `state.rs`
    - Defines the program's **PDA state** structures, authorities, and any relevant on-chain config (e.g., the token mint reference)
- `error.rs`
    - Custom error types used throughout the program

**Understanding the Solana Token Router Instructions** 

**`Initialize`**

- **Sets up** the program’s on-chain state (PDA), including any administrative keys or configuration.
- Typically called **once** after deployment to define the router’s state account.

**`SetMint`**

- **Registers or updates** the SPL token mint that the Token Router controls.
- The router program must be set as the **mint authority** on this SPL token to manage minting and burning.

**`Transfer`**

- Performs a **local** SPL token transfer between two Solana addresses.
- Often used internally or by integrators who want on-chain logic to move tokens around prior to bridging.

**`BridgeIn`**

- **Mints** tokens on Solana in response to a valid Wormhole message from another chain.

Internally, this does:

- **Verify** the Wormhole VAA via the Wormhole Core Bridge contracts.
- **Check** that the message is valid for this router (e.g. correct emitter chain, correct contract).
- **Mint** the specified SPL tokens to the recipient’s Solana account.

**`BridgeOut`**

- **Burns** tokens on Solana and publishes a Wormhole message to replicate that supply on another chain.

Internally, this does:

- **Burn** SPL tokens from the caller’s account.
- **Call** the Wormhole Post VAA instruction to create a bridging message.
- The user or a contract on the **destination chain** can redeem that message to mint tokens there.

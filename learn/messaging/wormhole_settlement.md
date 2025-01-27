# Understanding Wormhole Settlement

- Wormhole Settlement is a cross-chain transfer protocol that utilizes Solana as the central orchestration layer for all cross-chain intents, enabling solvers to hold and deploy liquidity from a single hub rather than distributing it across multiple chains.
- This hub-and-spoke model replaces fragmented, chain-by-chain liquidity with a consolidated pool on Solana, simplifying rebalancing and enhancing capital efficiency.
- By leveraging standards like Wormhole’s Native Token Transfers (NTT) and Circle’s CCTP, Wormhole Settlement ensures both established and newly launched chains gain immediate access to deep liquidity and can execute arbitrary on-chain actions atomically alongside transfers.

# **Overview**

Wormhole Settlement is a cross-chain transfer protocol designed to enable faster-than-finality transfers across the Wormhole ecosystem through a novel hub-and-spoke architecture centered on Solana. This architecture allows solvers to facilitate cross-chain transfers by fronting assets on the destination chain and assuming the finality risk of the originating source chain transaction. Solvers, however, concentrate their liquidity entirely on Solana, where they participate in permissionless on-chain English auctions to fulfill each cross-chain transfer. Upon the conclusion of each auction, the winning solver initiates a transfer from Solana to the specified destination chain. The solver rebalances inventory once the originating source chain transaction reaches finality and arrives to Solana. 

Wormhole Settlement is meant to serve as the underlying chain abstraction infrastructure layer for protocols across Wormhole connected ecosystems by enabling protocols to bundle call data containing arbitrary protocol actions, which can be executed atomically alongside each transfer. This feature allows developers to create fully chain-abstracted user experiences, including the construction of natively cross-chain decentralized exchanges (DEXs), borrow-lend protocols, payment protocols, and other applications atop of this layer.

## **Architecture Motivation**


### **Solvers and Liquidity Fragmentation** 


Traditional intent-based protocols require solvers to distribute their capital across each supported chain in the network. This fragmentation of liquidity leads to capital inefficiency and necessitates complex rebalancing to manage asymmetric flows between chains, imposing significant operational overhead on solvers. As the number of chains increases, solvers face scalability challenges, which can result in market concentration reducing competition and potentially harming user welfare.

The Wormhole Settlement addresses these challenges by consolidating solver liquidity on a single chain—Solana—using a hub-and-spoke model. This model eliminates the need for complex cross-chain rebalancing and simplifies infrastructure requirements for solvers. Solvers only need to consider the finality risk of the originating source chain transaction and the size of the payload when bidding on transfers. By concentrating liquidity on Solana, the protocol can handle large transfer volumes with a smaller capital base, enhancing capital efficiency and lowering barriers to entry for solvers. This approach promotes competition, improves overall market efficiency, and ultimately benefits users with better prices while still preserving the speed of transactions.


### **Transfer Rails: Enabling Unified Liquidity through Cross-Chain Fungibility**


The novel hub-and-spoke liquidity architecture relies on interoperable token standards that enable cross-chain token fungibility, such as Circle’s Cross-Chain Transfer Protocol (CCTP) and Wormhole’s Native Token Transfers (NTT). These protocols allow assets to move seamlessly between chains, making unified liquidity possible.

On the liquidity hub (Solana), solvers concentrate their liquidity in NTT or CCTP supported assets, such as USDC. These assets act the shuttle asset between chains but may not necessarily be the user’s original or final asset.

After the Solana auction concludes, the appropriate instructions are called on the CCTP or NTT contract initiating the transfer from Solana to the destination chain, by burning/locking the asset on Solana and sequently minting on the destination chain. Solvers rebalance their inventory on Solana using these interoperable token standards as well — once the originating source chain transaction reaches finality and arrive to Solana, the solver can redeem the NTT or CCTP message, minting the inventory for use once again.

By leveraging interoperable token standards like NTT, this model of liquidity facilitation for cross-chain intents can **arbitrarily** **scale** to any chain or ecosystem while still preserving fully unified liquidity—eliminating the need for solver "buy-in" for new chain expansion. Additionally, this means new chains, even without proven traction, can access the same amount of liquidity for cross-chain intent fulfillment from day one of mainnet launch as they would if they were long-standing ecosystems with clear evidence of adoption — commonly overlooked by solvers who must aggressively prioritize high flow chains to due high opportunity costs. Importantly, this includes new ecosystems without Centralized Exchange (CEX) enabled withdrawals. 

### **Understanding the Hub: Solana Matching Engine**

A central component of the protocol architecture is the permissionless English auction conducted on Solana, specifically the Matching Engine contract, allowing any third-party solver to interact with the matching engine to place bids or improve existing ones. The Matching Engine Contract includes four key instructions:

**`initialize_auction`** Creates a new auction account on-chain. This sets basic parameters like the auction’s token mint, the amount required, and the bidding period details. 

**`bid`** Allows a solver to place or update a bid on the active auction. The solver transfers the bid amount to the program escrow account, which ensures they actually have the liquidity. With each successful call of **`Bid`,** the program updates the auction to the new highest bidder and ****the prior bid is atomically sent back to the originating solver, allowing that solver to repurpose those funds if it so chooses to improve the bid once more. 

**`finalize_auction`** Following conclusion of the auction, this instruction completes the fast transfer by sending funds to the recipient on the target chain. This instruction may call the Circle CCTP contract or in the future release an NTT contract depending on the shuttle asset in question. Failure to execute this message within a predefined grace period may result in penalty for the winning bidder (more on this below). 

When placing a bid—whether initial or improved—the solver must deposit the required funds *plus* a **security deposit** into the matching engine contract. In this permissionless auction, the requirement of this principal amount plus the security deposit ensures a solver's credible commitment to fulfill the transfer. Without this safeguard, malicious actors could place hollow bids, undermining the auction's credibility and hindering true price discovery. 

Following the auction, the winning solver has to call an instruction on the matching engine to execute the intent. If the winning solver fails to do so, other competing solvers may permissionlessly ‘slash’ the solver by executing the instruction on their behalf, collecting a portion of the original security deposit as a reward while the remaining portion is routed to the user as compensation for the unanticipated delay. This mechanism is designed to properly incentivize timely execution through solver redundancy and competition.  

**`cancel_auction`** Cancels an open auction—intended for scenarios where the auction is no longer valid or was created by mistake. The program returns all locked funds to their respective owners.


### Summarizing the Protocol Flow: How It Works


**Initiation** — Users or protocols initiate a transfer via an interface or directly on-chain. They choose between a standard transfer (waiting for finality on the sending chain) or a fast transfer (triggering the auction process). For fast transfers, users or the protocol specify a maximum fee and an auction start deadline. **Note:** If an auction doesn't start within the set deadline, a standard transfer will proceed directly from the source to destination chain. 

**Auction** — Solvers monitor the Wormhole network for these fast transfer requests and initiate an auction on Solana by offering to fulfill the transfer at or below the user's maximum fee. To start the auction, the solver must transfer the requested funds plus a *small security deposit* to the Matching Engine contract.

**Competition** — Once initiated, other solvers can participate by submitting lower bids in a simple English auction, aiming to provide users with the best rate. If a new solver submits a better offer, the previous solver's funds and security deposit are returned, with the new offer taking precedence atomically. This competition ensures users receive the best possible transfer rate.

**Fulfillment** — After the auction concludes, the winning solver must complete the transfer within a predefined grace period to earn their fee and reclaim their security deposit. Failure to do so may result in the security deposit being slashed, with the slashed amount compensating the user for delays. This mechanism incentivizes prompt execution. Upon successful completion, the Fast Transfer hub sends the USDC to the user's destination wallet, and the solver receives their security deposit and transfer fee.

**Settlement** — Once the original source chain transaction reaches finality, the winning solver can use the finalized Wormhole message to settle the auction with the matching engine and rebalance. This allows the solver to retrieve the original transfer amount into their wallet.




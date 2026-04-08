# Review: ERC-XXXX -- Wallet Title Deeds

**Reviewer:** Hilmar Veigar Petursson perspective  
**Date:** 2026-04-08  
**Document reviewed:** ERCS/erc-XXXX.md (Draft, created 2026-04-02)

---

## 1. What I Like About It

### The Separation of Control from Custody Is the Key Insight

This is the single most important design decision in the proposal, and the authors got it right. In EVE Online we learned the same lesson over twenty years: the entity that controls an asset and the entity that custodies it do not have to be the same thing. A corporation CEO controls the corporate hangars, but the hangars exist independently of who sits in the CEO chair. When leadership changes, the assets stay put and the new leader inherits the full operational context. ERC-XXXX does exactly this for Ethereum accounts. The account is the hangar. The NFT is the CEO title. Transfer the title, transfer the command authority, assets remain in place. This is deeply correct.

### Hierarchical Account Trees Model Real Organizational Structure

The nesting capability -- where an NFT-controlled account can itself hold controller NFTs for child accounts -- is one of the most consequential features in the proposal. This maps directly to how real organizations work: holding companies own subsidiaries, subsidiaries own divisions, divisions own project accounts. In EVE, we watched players independently invent exactly this kind of structure. Alliances hold corporations, corporations hold divisions, directors manage hangars. The players needed this hierarchy and built it from whatever tools we gave them. ERC-XXXX provides the primitive natively, which means the organizational structures that emerge will be cleaner and more capable than anything built from ad hoc workarounds.

The `maxNestingDepth` with cycle detection is a pragmatic engineering constraint that prevents infinite recursion while still permitting meaningful depth. Default of 4 is well-chosen for most real organizational hierarchies.

### Transfer Lock and Mutual Exclusivity Are Load-Bearing Security

The design where "marketplace approval valid" and "execution allowed" are mutually exclusive is elegant and important. This is a sell-and-drain prevention mechanism that actually works at the protocol level rather than relying on social trust. In EVE, every time we have seen a corporate theft or scam, it happened because execution authority and transfer authority were not properly separated. The attacker could drain assets and transfer control in the same operational window. ERC-XXXX closes that window by construction. The unlock delay, the transfer approval version invalidation on `proposeUnlock`, and the execution freeze during the unlock window collectively create a real temporal separation between "operating the account" and "transferring the account." This is one of the best anti-scam mechanisms I have seen in any smart account proposal.

The asymmetric `setUnlockDelay` semantics -- increases immediate, decreases meta-timelocked -- is a subtle but critical detail. Without this, an attacker who accumulated trust with a long delay could collapse it to one second and immediately drain-and-sell. The meta-timelock means the delay is a commitment that observers can rely on.

### Control Version Invalidation Is Automatic Trust Revocation

The `controlVersionOf(tokenId)` increment on every transfer, with automatic invalidation of all prior delegated authority, solves a problem that has plagued every organization system I have seen in virtual worlds. When a director leaves a corporation in EVE, we have to manually revoke every role, every access, every delegation. ERC-XXXX makes this atomic: transfer the NFT, and every validator, every ERC-1271 signature, every delegated authority from the previous regime is dead. The new controller starts with a clean slate. This is how it should always have worked.

### The Direct-Owner Path Preserving Public Mempool Compatibility

The authors clearly understand that censorship resistance is not optional infrastructure. The direct-owner execution path stays within ordinary transaction validity, which means FOCIL inclusion lists and VOPS-style local validation continue to work. This is the kind of forward-thinking infrastructure design that matters at civilizational timescales. They are not just building for today's Ethereum -- they are building for an Ethereum where inclusion guarantees matter for account abstraction.

### Deterministic Deployment with CREATE2

The address derivation (`tokenId == uint256(uint160(account))`) being computable by anyone without querying a registry is exactly right. No central registry means no single point of failure, no governance bottleneck, no rent-seeking intermediary. Multiple implementations can coexist. The ecosystem picks winners through quality and trust, not through registry control. This is how standards should work.

### Approval Revocation During Transfer Freeze

The narrow revocation interface that works during the unlock freeze is a thoughtful detail. It threads the needle between "the seller needs to clean up the account before transfer" and "the seller must not be able to drain the account during the transfer window." Each revocation function makes exactly one external call with hardcoded zero-value calldata -- it cannot be repurposed as a general execution path. This shows careful security thinking.

---

## 2. What the Issues Are

### NFT Theft Is Total Account Takeover with No Recovery Default

The proposal correctly identifies that "theft of the controlling NFT is account takeover." Recovery is OPTIONAL. This means the default deployment of an ERC-XXXX account has zero recovery path if the controlling NFT is stolen or the owner key is lost. In EVE, we learned that even the most sophisticated players lose access to their accounts, and the consequences of permanent loss are severe enough to drive players away from the game entirely. The proposal should more strongly recommend (SHOULD-level, not just MAY) that implementations ship with a configurable recovery mechanism enabled by default, even if the user can choose to disable it.

### Surviving Token Approvals Are a Ticking Bomb for Account Buyers

The document acknowledges that ERC-20/ERC-721/ERC-1155 approvals granted by the account to third-party contracts survive the NFT transfer. The revocation interface helps, but the fundamental problem remains: there is no on-chain enumeration mechanism for outstanding approvals. A buyer of an account cannot programmatically discover all existing approvals -- they must rely on off-chain indexing of Approval events, which may be incomplete, lagged, or manipulated. This is the equivalent of buying a house where the previous owner gave copies of the keys to unknown parties and there is no locksmith who can tell you how many copies exist.

This is the biggest practical risk for account marketplace scenarios. Consider recommending or specifying a standard "approval audit" helper contract that can be deployed alongside the factory, or at minimum specify that wallets and marketplaces MUST surface a warning that approval enumeration is inherently incomplete.

### The One-Hour Default Unlock Delay May Be Too Short for High-Value Accounts

One hour is the default, and it can be reduced to one second. For accounts holding significant value -- treasury accounts, fund management accounts, DAO operational accounts -- one hour is a very short window for counterparties to observe and react. In EVE, our most important sovereignty timers run for hours to days precisely because the stakes justify giving defenders adequate reaction time. For institutional use cases, the proposal should consider whether the default should be higher (24 hours) or whether there should be a RECOMMENDED minimum for accounts above some self-declared value tier.

The minimum of one second is particularly concerning. Even with the meta-timelock on decreases, once the delay reaches one second the temporal separation between execution and transfer is effectively zero for practical purposes. One second is not enough time for any monitoring system, human or automated, to observe and react. Consider whether the floor should be higher -- perhaps 60 seconds or 300 seconds -- to maintain meaningful separation even in the most aggressive configuration.

### Cross-Chain Control Is Explicitly Out of Scope, but Users Will Need It

The document states: "Cross-chain control is out of scope." This is pragmatically correct for the initial standard, but it means that an organization operating on multiple chains will have separate, independently controlled accounts on each chain with no protocol-level mechanism to synchronize ownership. In EVE, we run a single shard precisely because fragmented sovereignty creates confusion and exploitable gaps. The authors should consider at least sketching a forward-compatible extension point -- perhaps a `crossChainControlVersion` or a bridge-aware ownership verification path -- so that future cross-chain extensions do not require breaking changes.

### The EIP-7702 Delegation Risk Section Is Alarming

The security considerations around EIP-7702 delegation are thorough and frightening. A malicious 7702 delegation can install validators, initiate unlock delay reductions, and grant token approvals -- and revoking the delegation does not undo any of these. The document correctly identifies the mitigation steps, but the fact that users must manually perform four separate recovery operations after a delegation compromise suggests the system is not fail-safe in this scenario. Consider whether `resetDelegations` should automatically trigger on detection of a delegation change, or whether the account should maintain a pinned `EXTCODEHASH` for the owner that automatically freezes execution if it changes unexpectedly.

### No Standard for Account Discovery Beyond Enumerable

ERC-721 Enumerable is SHOULD, not MUST. Without it, discovering which accounts an address controls requires off-chain event indexing. For wallet UX, organizational tooling, and estate planning, this is a serious gap. If a user holds twenty controller NFTs and their wallet does not have complete indexing, they may not even know some of their accounts exist. This should be MUST for the controller token.

### The Validator Interface Is Minimal to the Point of Under-Specification

The validator interface defines `onInstall`, `onUninstall`, and `isValidSignatureForAccount`. But there is no standard for capability scoping, spending limits, time bounds, or target restrictions on what a validator can authorize. The document says validators are "for delegated signature validation, not to define root control," but in practice, a validator that can approve arbitrary ERC-1271 signatures has significant power. Without a standard capability model, every validator implementation will define its own permission language, making cross-implementation interoperability difficult.

### Gas Costs for Nested Account Operations Could Be Prohibitive

Operating a child account through a parent account requires at minimum: the outer transaction to the parent, the parent's authorization check (ownerOf lookup on parent's controller token), the parent's execute call to the child, the child's authorization check (ownerOf lookup on child's controller token, which returns the parent), and the child's actual execution. For depth-4 hierarchies, this chain of authorization checks and delegated calls could consume substantial gas. The proposal should include gas analysis for nested operations at various depths.

### The Metadata Extension Coupling Is Surprising

The optional metadata extension where a controller token can use an NFT held by the controlled account as its avatar is clever but creates an unexpected dependency: the visual identity of the controller token depends on the assets held by the controlled account. If the account sells or transfers the NFT being used as its avatar, the controller token's metadata silently falls back to default. This is a minor issue but could create confusion in marketplace listings.

---

## 3. Most Compelling Usage Scenarios

### DAO Treasury Management Rotation

This is the killer application described in the proposal and it is entirely correct. When a DAO votes to change its treasury committee, today that requires migrating every asset, updating every protocol integration, changing every allowlist entry, and hoping nothing breaks. With ERC-XXXX, the outgoing committee transfers one NFT to the new multisig. Done. The treasury address, its Aave positions, its Uniswap LP, its ENS name, its governance participation history -- everything persists. This alone justifies the standard.

### Digital Estate Planning and Inheritance

The dead man's switch contract holding the controller NFT is a profoundly important use case. Today, crypto inheritance is a nightmare of shared seed phrases, instructions in safe deposit boxes, and centralized custodian workarounds. ERC-XXXX makes the entire on-chain estate -- every asset, every position, every membership -- transferable through a single NFT operation triggered by a time-lock or guardian multisig. This is the first proposal I have seen that makes crypto inheritance actually workable at a protocol level.

### Organizational Account Hierarchies with Approval Isolation

The combination of hierarchical accounts and per-account approval isolation solves a real problem that currently requires managing multiple seed phrases or hardware wallet slots. A fund manager can have a high-value treasury account and a DeFi interaction account, both controlled by the same key, where a compromised approval on the DeFi account cannot touch the treasury. This is how institutional crypto custody should work.

### Account Sale as a First-Class Operation

Selling a fully configured account -- with its protocol positions, allowlist memberships, reputation, and history -- through a standard NFT marketplace is a genuinely new capability. Today, selling a crypto account means either sharing keys (terrible security) or individually transferring every asset (expensive, incomplete, loses history). ERC-XXXX creates a real market for operational accounts.

### Vesting with Tradeable Claims

A vesting contract holds the controller NFT. The beneficiary has a claim on the account when vesting completes. But the claim itself -- the future right to control the account and its contents -- can be sold by transferring the vesting contract's claim. The assets remain locked; only the right to eventually control them changes hands. This creates a liquid secondary market for vesting positions without requiring the grantor to release tokens early.

---

## 4. Additional Possibilities the Authors May Not Be Thinking About

### Player-Run Organizations in On-Chain Games and Virtual Worlds

This is where I see the most transformative potential, and it is not mentioned in the proposal. ERC-XXXX provides the exact infrastructure needed for player-run organizations in on-chain games and autonomous worlds. Consider a fully on-chain game where guilds, corporations, and alliances are ERC-XXXX account hierarchies:

- The alliance is a parent account holding controller NFTs for each member corporation
- Each corporation is an account holding controller NFTs for division accounts (treasury, military, industry)
- Division accounts hold the actual game assets
- A hostile takeover is literally an NFT transfer -- the attacker acquires the corporation's controller NFT (through purchase, theft via in-game mechanics, or conquest) and instantly controls all of that corporation's division accounts and assets
- A corporate merger is two controller NFTs moving to the same parent account
- A corporate divorce is splitting controller NFTs to different parents
- Espionage means infiltrating a governance structure (multisig) that holds a controller NFT, then triggering a transfer at the right moment

This is EVE Online's corporate mechanics, but trustlessly enforced on-chain. The authors should consider on-chain games and autonomous worlds as a primary use case, not just DeFi and institutional custody.

### Reputation Markets and Account-Level Credit Scoring

Because accounts maintain persistent addresses through ownership changes, and because execution history is preserved, it becomes possible to build reputation systems that score accounts rather than keys. An account with a five-year history of prompt DeFi loan repayment, consistent governance participation, and clean protocol interactions has provable reputation. That reputation now has monetary value because the account can be sold. This creates:

- Account-level credit scoring where the account's history, not the controller's identity, determines creditworthiness
- Reputation markets where accounts with strong histories trade at premiums
- Age-weighted governance power that accumulates at the account level and persists through control transfers
- Anti-sybil mechanisms based on account age and activity density

The implications for undercollateralized lending are particularly interesting: a protocol could offer better terms to an account with a long, clean history, and the borrower's incentive to maintain that history (because it increases the account's resale value) creates a self-reinforcing credit system.

### Autonomous Agent Accounts with Bounded Blast Radius

AI agents and autonomous programs need Ethereum accounts to operate. ERC-XXXX's child account model provides the perfect container: fund a child account with exactly the assets the agent needs, install the agent's signing key as a validator on the child, and the agent operates independently within its bounded sandbox. If the agent is compromised or malfunctions, the damage is hard-capped by the child account's balance. The parent can revoke the agent's validator or reclaim the child's controller NFT at any time.

This is materially better than giving an agent a session key on a valuable account with permission scoping as the only defense. Permission policies are complex and may have gaps. Account boundaries are simple and absolute.

Taken further: an AI agent that manages multiple strategies could itself hold a parent account with child accounts for each strategy. The agent's human operator holds the parent's controller NFT. Transferring that NFT transfers the entire managed portfolio -- all strategies, all positions, all history -- to a new operator.

### Subscription and Service Access Tied to Account Control

A service provider deploys an ERC-XXXX account, configures it with a validator that gates access to their service (API keys, data feeds, compute resources), and sells the controller NFT. The buyer controls the account and its service access. When they no longer need the service, they sell the account to someone else. The service provider never has to manage customer credentials -- the controller NFT IS the credential. This creates transferable subscription NFTs with teeth: not just a claim on access, but actual operational control over the service integration.

### Trustless Escrow for Complex Multi-Asset Transactions

An escrow contract holds a controller NFT. The account behind it contains a complex portfolio: some ETH, various ERC-20 tokens, LP positions, staked assets, governance tokens. The escrow releases the controller NFT to the buyer when payment conditions are met. One NFT transfer, one atomic operation, and the buyer controls the entire portfolio. Today, escrowing a complex portfolio requires individually depositing each asset into the escrow, which is expensive, error-prone, and may be impossible for non-transferable positions. ERC-XXXX makes the escrow hold the control credential instead of the assets.

### Franchise and License Models

A parent organization deploys child accounts configured for specific business operations (a retail outlet, a franchise location, a licensed operator). Each child account has validators installed for the operator's staff, spending limits enforced by the parent, and approved contract whitelists. The parent sells or licenses operation by transferring the child's controller NFT to the franchisee. If the franchise agreement terminates, the franchisor can reclaim control through whatever recovery or governance mechanism was configured. The account's history of transactions, customer interactions, and protocol integrations persists across operators.

### Dead-Man's-Switch Cascading Failovers

For critical infrastructure accounts, a hierarchy of dead-man's-switch contracts can provide cascading failover. The primary operator has direct control. If they go silent for 30 days, control falls to a secondary operator. If the secondary is also silent, control falls to a governance contract. If governance fails to act, control falls to a hardcoded last-resort address. Each level is a contract holding the next level's controller NFT with time-locked release conditions. This is infrastructure-grade operational resilience using nothing but ERC-721 transfers and time-locks.

### On-Chain Mergers and Acquisitions

Two organizations want to merge. Today this requires months of asset migration, contract updates, and governance votes. With ERC-XXXX hierarchies, a merger can be structured as: create a new parent account, transfer both organizations' controller NFTs to the new parent. Both organizations' full account hierarchies -- treasuries, divisions, projects -- are now under unified control. A divestiture is the reverse: transfer a subsidiary's controller NFT out of the parent. The subsidiary retains its full operational context. This is real corporate M&A infrastructure on-chain.

---

## 5. Changes That Could Increase Optionality Further

### Add a Standard Event for Pre-Transfer Account State Snapshot

Before transfer, the account should emit a standardized event containing a hash of its current state -- balances, active validators, outstanding approval count (if knowable), pending operations. This gives buyers and counterparties an on-chain commitment about the account state at transfer time, which can be verified against off-chain indexing. This does not solve the approval enumeration problem completely, but it creates an auditable record.

### Define a Standard Capability Descriptor for Validators

Instead of leaving validator permissions entirely implementation-defined, specify a minimal capability descriptor: what types of signatures can this validator authorize, for what targets, up to what value, until what time. This does not need to be the full permission language -- just enough structure that wallets can display "this validator can authorize ERC-1271 signatures for Uniswap Router up to 10 ETH until April 2027" in a human-readable way. Without this, validators are opaque black boxes from the user's perspective.

### Consider a "Freeze" Operation Distinct from Unlock Mechanics

The current model overloads the transfer lock state machine for security freezes. If a user detects suspicious activity, they want to freeze the account immediately -- but `lock(tokenId)` is already the mechanism for canceling an unlock proposal. Consider a separate `freeze(tokenId)` operation that immediately halts all execution (including revocations), all validator operations, and all unlock state changes, and can only be unfrozen after a configurable delay or through a recovery mechanism. This is the "panic button" that high-value accounts need and that the current state machine does not cleanly provide.

### Specify a Standard "Account Manifest" View

Add a standardized view function that returns the account's operational summary: controller token address, tokenId, current controller, control version, active validator count, lock state, unlock delay, pending unlock delay change, whether a recovery mechanism is configured. This is one `STATICCALL` that gives any integrator the full picture. Without it, every integrator must make six or seven separate calls to reconstruct the account's status.

### Allow Optional "Transfer Hooks" for Accounts

When a controller NFT is transferred, the account may need to perform cleanup -- invalidate cached state, emit events for indexers, notify integrated protocols. The current design has no hook for this. Consider an optional `onControlTransfer(address previousOwner, address newOwner, uint256 newControlVersion)` callback on the account that the controller token invokes as part of the transfer. The callback MUST NOT be able to block or revert the transfer (to prevent denial-of-transfer attacks), but it can perform cleanup. The reference implementation guidance explicitly says "avoid transfer hooks from the controller token into the account" -- reconsider this prohibition.

### Consider an "Account Tag" or "Account Type" Field

When organizations deploy hierarchies of accounts, they need to distinguish treasury accounts from operational accounts from project accounts. A simple `bytes32 accountTag` settable by the controller would allow organizational tooling to classify accounts without relying on off-chain metadata. This is trivial to implement and dramatically improves discoverability and UX for multi-account setups.

### Specify Behavior for Receiving Controller NFTs of Other Implementations

The document says multiple independent compliant controller-token implementations may coexist. But it does not address what happens when an account deployed under Implementation A receives a controller NFT from Implementation B. The account would then control a child account under a different implementation with potentially different security properties, different transfer lock semantics, and different control version behavior. This cross-implementation nesting should be explicitly addressed -- either prohibited, warned about, or given clear semantics.

### Consider Standardizing a Minimal "Account Score" or "Account Age" View

For reputation and credit scoring use cases, a standardized `accountAge()` returning `block.timestamp - deploymentTimestamp` and a `transferCount()` returning the number of times the controller NFT has been transferred would provide baseline reputation inputs without requiring off-chain indexing. Accounts that have been deployed for years and never transferred are meaningfully different from freshly deployed accounts, and protocols should be able to distinguish them on-chain.

### Add Explicit Support for "Read-Only Delegates"

The current model has controllers (full authority) and validators (delegated signature validation). There is no standard concept of a read-only delegate who can call view functions on the account's behalf for monitoring, reporting, or audit purposes. While view functions are technically callable by anyone, a standardized `isReadDelegate(address)` would allow the account to signal to integrated protocols that a specific address is authorized to query on the account's behalf -- useful for organizational monitoring, auditor access, and regulatory reporting without granting execution authority.

### Define a Standard "Account Migration" Path

The document says ERC-6551 migration is not automatic. But what about migration between different ERC-XXXX implementations? If a user wants to move from Implementation A (which is immutable) to Implementation B (which supports a new feature), there should be a standardized migration path that preserves the account address. This likely requires upgradeability, which the proposal deliberately keeps optional, but the migration use case deserves explicit treatment.

---

## Summary Assessment

ERC-XXXX is the most thoughtful account abstraction proposal I have reviewed. It gets the fundamental architecture right: control is separable from custody, transfer is a first-class operation, organizational hierarchies emerge naturally from the nesting model, and security is built into the state machine rather than bolted on as policy. The sell-and-drain prevention via mutual exclusivity of execution and transfer authority is particularly well-designed.

The primary risks are: surviving token approvals creating hidden liabilities for account buyers, the optional status of recovery creating a default-unsafe configuration for most users, and the under-specification of validator capabilities leaving a delegation surface that is powerful but opaque.

The most exciting long-term possibility is not DeFi custody or institutional treasury management -- it is the creation of trustlessly governed on-chain organizations with real corporate structure, real M&A mechanics, real succession planning, and real accountability. This is the infrastructure that autonomous worlds, on-chain games, and digital nations need. The authors should think bigger about what organizational structures their primitive enables, because the players will build things they never imagined.

In EVE Online we spent twenty years watching players build civilizations from the primitives we gave them. ERC-XXXX provides better primitives for organizational control than anything we had. The question is not whether players will build surprising things with it -- the question is whether the standard is flexible enough to accommodate the organizations and governance structures that have not been invented yet. Based on this review, I believe it mostly is, with the caveats noted above.

*"Our job is to be the universe. The players are the content."*

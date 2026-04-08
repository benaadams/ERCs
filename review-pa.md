# Review: ERC-XXXX "Wallet Title Deeds"

**Reviewer:** Paradigm-Aware Analysis  
**Date:** 2026-04-08  
**Status:** Draft Review of Draft ERC  
**Authors under review:** Ben Adams, Tim Seaward, Artemis Black, Carlos Perez, Giulio Rebuffo

---

## 1. What I Like About It

### The Core Insight Is Genuinely Profound

This ERC does something rare: it identifies a latent duality in the Ethereum account model and reifies it into a clean primitive. The insight that **control and custody are orthogonal axes** -- and that conflating them is the root cause of most account-management pain -- is not new in the abstract, but this proposal crystallizes it into a concrete, composable mechanism with unusual elegance.

The `tokenId == uint256(uint160(account))` canonical mapping is beautiful. It is the kind of design decision that looks obvious in retrospect but required someone to actually commit to it. No registry. No lookup table. No oracle. The relationship between controller and controlled is a pure arithmetic identity, verifiable by anyone with a calculator. This is topology, not bureaucracy -- the control relationship is embedded in the structure of the identifiers themselves, not in some mutable data structure that must be consulted. It eliminates an entire class of indirection bugs and trust assumptions in one stroke.

### The Transfer Lock State Machine Is Well-Engineered

The `proposeUnlock -> completeUnlock` flow with the asymmetric meta-timelock on `setUnlockDelay` is the kind of security engineering that shows the authors have thought carefully about adversarial game theory, not just the happy path. The key properties:

- Execution and transferability are mutually exclusive (not merely "discouraged" -- enforced by the contract).
- The transfer approval version invalidates stale approvals on every state transition that matters.
- Delay decreases are themselves timelocked for the duration of the current delay, preventing a long delay from being collapsed to nothing right before an attack.
- The execution freeze covers the entire proposal window, not just the moment of transfer.

This is a closed attack surface. The authors have correctly identified that the dangerous moment is not the transfer itself but the window between "I intend to transfer" and "transfer happens," and they have sealed it. The fact that approval revocation is specifically carved out from the execution freeze -- because revoking is always safe from the buyer's perspective -- shows good practical judgment about what operations are monotonically risk-reducing.

### Hierarchical Nesting Is a Natural Consequence, Not a Bolted-On Feature

Because an account can hold ERC-721 tokens, and controller tokens are ERC-721, the nesting falls out of the existing semantics. The cycle detection on transfer (bounded chain walk up to `maxNestingDepth`) is the right way to handle this: it makes the dangerous operation (creating cycles) expensive and bounded rather than making the common operation (executing) expensive with per-call ancestry checks. The cost is borne by transfers (rare) rather than subcalls (frequent). This is the correct amortization.

### The FOCIL/VOPS Alignment Is Forward-Looking

The explicit positioning relative to EIP-7805 and VOPS shows architectural awareness of where Ethereum's infrastructure is heading. The direct-owner path remains an ordinary transaction -- no new mempool validation prefix, no special opcode restrictions for mempool admissibility. This is significant because it means the standard does not create a new class of "mempool-invalid but execution-valid" objects that censorship-resistance mechanisms cannot see. The authors are building for the Ethereum that will exist in two years, not the one that existed two years ago.

### Recovery as NFT Transfer Is Philosophically Correct

Defining recovery as culminating in an NFT transfer rather than bypassing the NFT is the right call. It preserves the core invariant (the NFT is always the root control object) even under adversarial conditions. Social recovery systems that bypass the ownership primitive are security theater -- they create a second, often less audited, control path that an attacker can target independently. Here, recovery and normal operation go through the same bottleneck. One throat to choke.

### The ERC-6551 Differentiation Is Honest and Precise

The comparison table with ERC-6551 is unusually clear for a standards document. The authors resist the temptation to claim they are "better than 6551" and instead explain the structural differences: 6551 says "an arbitrary NFT can have an account," this ERC says "a dedicated NFT is the canonical root-control object for one specific account." These solve genuinely different problems, and the authors say so plainly. This kind of intellectual honesty in a standards proposal is refreshing.

### Metadata Delegation via Held NFTs Is Clever

The optional `setTokenImage` mechanism -- where a controller token's visual identity is derived from an NFT the controlled account holds, verified dynamically on each `tokenURI` call -- is a small touch that reveals deep thinking about how these objects will be experienced in practice. It means the "wallet" can visually represent itself as its most prized holding without copying metadata, and the representation updates automatically if the underlying NFT changes. This turns a purely functional instrument into something with visual identity, which matters enormously for adoption.

---

## 2. Issues and Concerns

### 2.1 The Surviving Approvals Problem Is More Dangerous Than Acknowledged

The ERC correctly notes that token-level approvals (ERC-20 `approve`, ERC-721 `setApprovalForAll`, ERC-1155 operator approvals) survive control transfer. The revocation interface is provided, and the security considerations discuss the risk. But I think this undersells the danger in practice.

The fundamental issue: there is no on-chain enumeration of approvals. The new controller must rely on off-chain indexing (event logs) to discover what approvals exist. This creates an information asymmetry that is structurally embedded in the protocol. A seller who has granted approvals to contracts that are currently benign but upgradeable has created a latent drain vector that may not appear in any off-chain approval scanner's risk model. The buyer receives a clean-looking account with a hidden mine.

**Recommendation:** The ERC should more strongly RECOMMEND that marketplaces and wallet UIs treat account transfer as a "buyer beware" operation with mandatory approval surface display. Consider whether an `approvalSnapshotHash` mechanism -- where the account emits a hash of known approvals at the time of `proposeUnlock` -- could help buyers verify they have complete information, even if the hash cannot be trustlessly computed on-chain.

### 2.2 The One-Hour Default Unlock Delay May Be Too Short for Marketplace Contexts

One hour is enough time for a monitoring bot to notice a `proposeUnlock`, but it is arguably too short for human-scale due diligence on a high-value account sale. A buyer who sees an account listed on a marketplace needs to: enumerate the account's assets, check outstanding approvals, verify validator installations, inspect the control hierarchy, check for pending unlock-delay decreases, and evaluate any positions the account holds in DeFi protocols. One hour is tight for this, especially across time zones.

The meta-timelock on delay decreases means an owner cannot suddenly collapse the delay, which is good. But the default of one hour means a freshly deployed account can be transferred with only one hour of visibility by default.

**Recommendation:** Consider whether the default should be higher (e.g., 24 hours) with the ability to reduce via the meta-timelocked `setUnlockDelay`. Users who need faster transfers for programmatic use cases can reduce it, but the default should protect the most common case (a human buying an account and wanting time to inspect it).

### 2.3 Cross-Chain Story Is Intentionally Absent but Will Cause Pain

The ERC explicitly declares cross-chain control out of scope. This is intellectually honest but practically painful. A user who controls account `A` on mainnet via controller token `T` will naturally expect that "their wallet" exists on Arbitrum, Optimism, Base, etc. The user-salt mode provides address stability, but:

- Ownership is chain-local. The same human must independently deploy and manage controller tokens on each chain.
- A transfer on mainnet does not transfer on L2s. This is confusing and dangerous.
- Validators, recovery configurations, and unlock delays are all chain-local.

This is not a criticism of the ERC per se -- cross-chain anything is genuinely hard -- but the ecosystem around this standard will need a cross-chain coordination protocol almost immediately, and the ERC should at least sketch the design space to prevent incompatible extensions.

**Recommendation:** Add a "Future Extensions" or "Design Space" section that acknowledges the cross-chain coordination problem and sketches the shape of a solution (e.g., a bridge-mediated ownership sync protocol, or a cross-chain "home chain" designation) without standardizing it. This helps implementers build in the right direction.

### 2.4 The Execution-Active Guard Relies on Transient Storage

The `isExecutionActive()` mechanism uses `TSTORE`/`TLOAD` (transient storage) for a reference-counted execution flag. This is correct and gas-efficient, but it creates an implicit dependency on EIP-1153 (transient storage opcodes). While EIP-1153 is live on mainnet, not all EVM-compatible chains support it. The ERC should either:

- Explicitly list EIP-1153 as a dependency, or
- Define a fallback mechanism for chains without transient storage (e.g., a regular storage slot that is set/cleared, with the gas cost acknowledged).

### 2.5 The `maxNestingDepth` Default of 4 Constrains Organizational Depth

The default `maxNestingDepth` of 4 (parent -> subsidiary -> department -> project) covers most corporate hierarchies, but it constrains multi-jurisdictional structures common in international organizations. A holding company with regional subsidiaries, each with local operating entities, each with project-specific accounts, already exceeds 4 levels. The gas cost argument (4 hops x 30,000 gas = 120,000 gas) is reasonable for current gas prices, but gas economics change over time.

**Recommendation:** Make `maxNestingDepth` configurable per deployment rather than per spec. The ERC already exposes it as a view function; making it constructor-configurable (rather than spec-defaulted) gives deployers flexibility without changing the interface.

### 2.6 The Validator Interface Is Thin

`IERCXXXXValidator` defines only `onInstall`, `onUninstall`, and `isValidSignatureForAccount`. This is intentionally minimal, but it may be too minimal for the delegated-execution patterns the motivation section describes (corporate cards, departmental budgets, agent-operated child accounts). A validator that cannot express "this signer may call these functions on these contracts with values up to X" is not a validator in the organizational sense described in the motivation -- it is only a signature verifier.

The ERC acknowledges that validators are for "delegated signature validation, not to define root control," but the motivating use cases describe permission-scoped execution delegation. There is a gap between what the motivation promises and what the interface delivers.

**Recommendation:** Either scope the motivation section's organizational delegation examples more carefully (emphasizing that they require additional standards like ERC-7579 modules), or expand the validator interface to include at least a `validateExecution(address target, uint256 value, bytes calldata data)` hook that can express execution-time policy.

### 2.7 No Standard for Account Upgradeability Creates Fragmentation Risk

The ERC says upgradeability "is not core" and "adds trust assumptions." This is correct, but the lack of a standard upgrade path means every implementation will invent its own. Some will use UUPS, some will use transparent proxies, some will use diamond patterns, some will be immutable. This fragmentation is fine for the accounts themselves but becomes a problem for tooling, indexers, and wallets that need to understand what an account can do.

**Recommendation:** If upgradeability is supported, RECOMMEND a specific pattern (UUPS with an `Upgraded` event is already suggested) and define the event as REQUIRED for any implementation that supports upgrades.

### 2.8 No Explicit Gas Limits on Batch Execution

`executeBatch` is atomic and can contain an arbitrary number of calls. There is no spec-level guidance on maximum batch size or per-call gas allocation. A batch that is too large to fit in a block is useless; a batch where early calls consume all gas leaving later calls to fail is a subtle atomicity hazard (the entire batch reverts, but the user may not understand why).

**Recommendation:** Add guidance on batch size limits and gas estimation, even if these are not enforceable at the spec level.

---

## 3. Usage Scenarios I Find Exciting

### 3.1 DAO Treasury Management Without Address Migration

A DAO's treasury has accumulated years of protocol positions, allowlist memberships, governance participation history, and address-based reputation. Today, changing the treasury management committee requires either sharing keys (terrible) or migrating assets to a new multisig (expensive, disruptive, and lossy -- you cannot migrate address-based reputation). With this ERC, the DAO holds the controller token in a governance contract, and committee changes are NFT transfers. The treasury address, its Aave positions, its Uniswap LP, its ENS name, its Snapshot delegation history -- all persist at the same address under new management.

### 3.2 Institutional Custody with Auditable Control Transfer

A fund manager manages client accounts as a set of NFT-controlled wallets. Each client's portfolio is a separate account with a separate controller token. The fund manager's master account holds all the controller tokens. When the fund is sold to another manager, the entire portfolio -- every client account, every position, every approval -- transfers via a single NFT transfer of the parent controller token. The transfer is on-chain, timestamped, and auditable. This is a clean handoff that securities regulators can actually inspect.

### 3.3 Dead Man's Switch / Digital Inheritance

A user holds their controller token in a contract that implements a dead man's switch: if the user does not ping the contract within N days, a designated heir can claim the controller token. The user's entire on-chain estate -- every asset, every position, every membership -- transfers to the heir through a single NFT transfer. No seed phrases, no custodians, no probate court trying to understand what a "private key" is.

### 3.4 Vesting with Transferable Claim

A startup grants tokens to an employee by funding a child account with a vesting schedule enforced by a parent account. The employee's controller token for the child account can be sold on a secondary market (transferring the right to eventually receive the vested tokens) without the startup releasing tokens early. The assets remain locked; only the claim changes hands. This is genuinely novel -- current vesting contracts make the beneficiary non-transferable because they hard-code the recipient address.

### 3.5 Approval-Scoped DeFi Interaction

A user has a "vault" account holding their long-term ETH and a "playground" account for degen DeFi. Both are controlled by the same EOA. The playground account grants unlimited approvals to experimental protocols. If one of those protocols is exploited, the blast radius is limited to the playground account's balance. The vault account is untouched because its approvals are completely separate. This is the security model that hardware wallet users achieve by maintaining separate devices, but without the key management burden.

### 3.6 AI Agent Custody Boundaries

An AI trading agent operates a child account funded with a specific budget. The agent has a session key (validator) on the child account. If the agent is compromised or goes rogue, the maximum loss is the child account's balance. The parent account can revoke the agent's validator via `resetDelegations` or simply stop funding the child. The agent never has access to the parent's assets. This is hard-capped blast radius delegation -- the kind of safety boundary that AI agent frameworks desperately need and currently lack.

---

## 4. Additional Possibilities the Authors May Not Be Thinking Of

### 4.1 Programmable Account Ownership as a Financial Primitive

The controller token is an ERC-721, which means it can be held by any contract. This is stated in the ERC. What is not fully explored is the implication: **any contract that can hold an ERC-721 can become an account ownership policy.** This turns account control into a programmable primitive composable with the entire existing smart contract ecosystem.

- **Options on account control:** A contract holds the controller token and releases it to whichever party exercises an on-chain option by a deadline. This is a call option on an entire wallet, not on a specific asset.
- **Account control as collateral:** The controller token can be posted as collateral in a lending protocol. If the borrower defaults, the lender receives control of the entire account. This creates a new asset class: secured lending against on-chain portfolio value where the collateral is not individual tokens but the entire position.
- **Conditional ownership:** A contract holds the controller token and releases it only when an oracle condition is met. Example: an insurance contract that transfers control of a claims-payout account to the claimant only when the oracle confirms the insured event. The account is pre-funded and ready; the condition controls release.
- **Account rental:** A contract holds the controller token and temporarily "lends" control to a renter for a fixed period, automatically reclaiming it when the period expires. This enables account-as-a-service for users who need a pre-configured DeFi position temporarily (e.g., a flash loan of an entire yield-farming setup).

### 4.2 The Account Tree as an Organizational Topology

The hierarchical nesting, combined with the fact that transferring a parent's controller token transfers control of all descendants, creates a structure isomorphic to corporate ownership graphs. But it goes further than current corporate structures because:

- **Divestiture is atomic:** Selling a subsidiary (child account) is a single NFT transfer. The subsidiary retains its address, positions, history, and all downstream relationships. This is what corporate M&A would look like if corporate law were as composable as smart contracts.
- **Mergers via re-parenting:** Two organizations can merge their account trees by transferring their respective root controller tokens to a new shared parent account. The merged entity has a single root, and each legacy organization is a subtree.
- **Joint ventures as shared nesting:** Two parent accounts can each hold child-account controller tokens within a shared grandchild structure. The joint venture is a distinct subtree whose control is distributed between the parents according to whatever governance the parent accounts implement.

The second-order effect: **on-chain organizational structure becomes introspectable.** Auditors, regulators, and counterparties can walk the ownership tree on-chain and see the complete control hierarchy. This is more transparent than any existing corporate registry.

### 4.3 Account Reputation as a Transferable Asset

Because the account address persists across control transfers, the account accumulates history: transaction volume, protocol participation, governance votes, allowlist memberships, Sybil resistance scores, credit history (in on-chain lending), and more. This history is address-bound, not key-bound. When the controller token transfers, this reputation transfers with it.

This creates a secondary market for on-chain reputation -- which is simultaneously exciting and terrifying. It means:

- A new user can buy a "seasoned" account with an established credit score.
- Protocol allowlists become transferable access passes.
- Sybil resistance mechanisms that rely on address age or activity history become gameable through account purchase.

The ERC should probably acknowledge this double-edged sword more explicitly. Protocols that assign reputation to addresses will need to decide whether reputation should survive control transfers or be reset (by watching for `ControlVersionChanged` events).

### 4.4 The Controller Token as a Coordination Point for MEV Protection

Because the controller token's state machine creates a visible separation between "account can execute" and "account can transfer," and because these states are mutually exclusive, there is an interesting MEV property: during the unlock window, the account is execution-frozen. This means no front-running of the account's transactions is possible during the transfer preparation window, because the account cannot transact.

A more speculative extension: a "pre-committed execution" model where the owner commits to a set of post-transfer operations during the unlock window (via a Merkle root or similar commitment), and the new owner can execute only those pre-committed operations in the first N blocks after transfer. This would enable verifiable account-transfer escrow where the buyer can inspect exactly what the account will do after the transfer.

### 4.5 Interplay with EIP-7702: EOA-to-Smart-Account Migration Path

The ERC mentions EIP-7702 compatibility, but the full migration story is worth spelling out. A user with an existing EOA can:

1. Deploy an NFT-controlled account.
2. Move their assets from the EOA to the new account.
3. Hold the controller token in the EOA.
4. Later, use EIP-7702 to add code to the EOA itself, or transfer the controller token to a multisig, or move to a post-quantum signer via EIP-8202.

The NFT-controlled account becomes the **stable identity** while the EOA becomes a **replaceable control mechanism**. This inverts the current Ethereum identity model, where the EOA address is the identity and the signing key is the control mechanism. Here, the account address is the identity, and the controller token (which can be held by anything) is the control mechanism.

This is a genuine paradigm shift in how Ethereum identity works, and the ERC should articulate it more explicitly.

### 4.6 Composability with Account Abstraction Paymasters for Gasless Operation

The ERC mentions ERC-4337 compatibility but does not explore the full implications. With a paymaster, the NFT-controlled account can be operated without the controller ever holding ETH on the controller's own address. A sponsor deploys the account, a paymaster pays for all transactions, and the controller holds only the NFT and nothing else. This enables a "keycard" model: the controller NFT is the keycard, and the account is the vault. The keycard holder never needs native currency -- they just sign transactions that the paymaster sponsors.

This is particularly powerful for onboarding: a new user receives a controller NFT (via airdrop, QR code, or NFC tap), and the account is already deployed and funded. The user immediately has a fully operational smart account with zero friction. The controller token is literally a "key to the wallet" that can be physically handed to someone.

### 4.7 Time-Locked Governance Transitions

A governance system can hold a controller token in a timelock contract: when the governance vote passes, the controller token is transferred to the new controller after a delay. During the delay, the community can inspect and verify. This is similar to existing governance timelocks, but it applies to the entire account (all assets, positions, and relationships) rather than to individual contract function calls.

### 4.8 Account Trees as a Substrate for Programmable Organizations

The combination of hierarchical nesting, validators, and the ability for any contract to hold a controller token creates the substrate for what might be called "programmable organizations" -- entities whose internal structure, authorization rules, and ownership hierarchy are all on-chain, introspectable, and composable.

A company could be modeled as:

- Root account: controlled by a governance token or multisig (the "board")
- Treasury account: child of root, holds the operating capital
- Payroll account: child of root, with a validator for the payroll service
- Per-department accounts: children of treasury, each with validators for department heads
- Per-project accounts: children of department accounts, each with validators for project leads

This is not a theoretical exercise -- this is literally the organizational chart encoded as an account tree. Department budgets are just child account balances. Authority delegation is just validator installation. Restructuring is just NFT transfers. Auditing is just walking the tree.

---

## 5. Changes to Increase Optionality

### 5.1 Define an Event for Account-Level Approval Grants

The ERC cannot enumerate existing approvals on-chain, but it could standardize an event emitted by the account when it grants approvals through `execute` or `executeBatch`. If the account detects that the calldata matches `approve(address,uint256)` or `setApprovalForAll(address,bool)`, it could emit a standardized event:

```solidity
event ApprovalGranted(address indexed token, address indexed spender, uint256 amount);
event OperatorApprovalGranted(address indexed token, address indexed operator, bool approved);
```

This would make approval tracking trivially indexable without relying on the target token contracts' events. The cost is minimal (one additional event per approval-granting call). The benefit is enormous for account transfer safety.

### 5.2 Add a `transferWithCallback` Extension Point

The ERC should consider an optional `onControlTransfer(uint256 tokenId, address previousController, address newController)` callback on the account, invoked after a successful transfer. This would allow accounts to perform cleanup logic on transfer (e.g., auto-revoking high-risk approvals, emitting internal events, notifying dependent contracts). The callback should be optional and bounded in gas to avoid griefing.

### 5.3 Standardize a "Read-Only Viewer" Role

Many organizational use cases need a role that can inspect the account's state (balances, approvals, positions) without execution authority. This is currently not possible -- either you are the controller (full authority) or you are nobody. A standardized `viewer` role (perhaps another ERC-721 token, or a simple address mapping) would enable auditors, compliance officers, and read-only dashboards without granting execution authority.

This is less about on-chain access (view functions are public) and more about off-chain semantics: a "viewer" badge that tooling can recognize as "this address is authorized to be shown this account's full state in a wallet UI" for private or permissioned frontends.

### 5.4 Define a Canonical Account-Discovery Interface

The ERC implies that wallets should discover controlled accounts by enumerating owned controller tokens (hence the ERC-721 Enumerable recommendation). But for the hierarchical case, discovery requires recursion: for each owned controller token, check if the controlled account itself owns controller tokens, and recurse.

A standardized `discoverControlledAccounts(address root, uint256 maxDepth)` view function (either on the controller token or as a separate helper) would simplify wallet implementation and ensure consistent discovery across implementations.

### 5.5 Consider an "Emergency Drain" Mechanism

The execution freeze during unlock is a critical safety feature, but it creates a scenario where the controller cannot respond to a time-sensitive emergency (e.g., a DeFi protocol announces imminent insolvency and the controller needs to withdraw immediately). The controller must first call `lock()` to cancel the unlock, then execute the emergency withdrawal, then restart the unlock process.

This is the correct behavior from a security perspective, but it adds three transactions where one might suffice. An "emergency lock-and-execute" atomic operation that cancels the pending unlock, increments the transfer approval version, and executes a batch in one transaction would reduce the emergency response latency without compromising the security model (because the unlock is cancelled, the transfer approval version is invalidated, and the account returns to the execution-only state).

### 5.6 Explicit EIP-1153 Dependency

As noted in the concerns section, the transient storage dependency should be explicit. Add EIP-1153 to the `requires` list or define a fallback for chains without it.

### 5.7 Consider a `delegateExecute` Path for Validators

The current validator interface only covers signature validation. For the organizational use cases described in the motivation (corporate cards, departmental budgets, agent-operated accounts), validators need to be able to authorize execution, not just signatures. A `delegateExecute(address target, uint256 value, bytes calldata data, bytes calldata validatorProof)` function on the account, which checks with an installed validator before executing, would close the gap between the motivation and the interface. This should be optional but standardized if present.

### 5.8 Standardize the Relationship Between Parent and Child Execution

When a parent account executes on behalf of a child (by calling `child.execute(...)` through `parent.executeBatch(...)`), the authorization chain is: controller owns parent NFT -> parent calls child execute -> child checks `msg.sender == ownerOf(childTokenId)` which is the parent. This works, but it means the child sees the parent as the controller, not the ultimate human. For audit and attribution purposes, it would be valuable to have a standardized way to propagate the original caller through the hierarchy. Consider an optional `executeFor(address target, uint256 value, bytes calldata data, bytes calldata originProof)` that carries provenance.

### 5.9 Token-Gated Read Access for Account State

While all on-chain state is technically public, the rise of private mempools and encrypted state proposals means that "publicly readable" may not always be the case. The ERC should consider how controller tokens interact with future privacy-preserving execution environments. If account state becomes encrypted, the controller token becomes not just the control credential but the read credential. The ERC should be designed with this future in mind, even if it does not standardize encrypted state.

---

## Summary

ERC-XXXX is one of the most architecturally thoughtful Ethereum standards proposals I have reviewed. It identifies a genuine structural gap -- the conflation of control and custody in the Ethereum account model -- and closes it with a mechanism that is simultaneously simple (an NFT controls a smart account), deep (the security properties are carefully engineered), and composable (the primitive composes with the entire ERC-721 ecosystem).

The strongest aspect is the security state machine around transfers: the mutual exclusion of execution and transferability, the transfer approval versioning, the asymmetric meta-timelock on delay decreases, and the execution-active guard. These are not "nice to haves" -- they are the load-bearing walls of the security model, and they are well-designed.

The weakest aspect is the gap between the motivation's organizational delegation promises and the validator interface's actual capabilities. The motivation describes corporate cards, departmental budgets, and agent-operated accounts with spending limits and contract whitelists. The validator interface provides only signature validation. This gap will either be closed by future ERCs (ERC-7579 modules, for example) or will become a source of non-standard extensions that fragment the ecosystem.

The most exciting implication is what happens when you combine this primitive with the existing smart contract ecosystem: account control becomes a programmable, composable, tradeable primitive. Options on wallets. Collateralized lending against entire portfolios. Atomic corporate divestitures. Programmable organizations as account trees. These are not speculative -- they are direct consequences of making account control an ERC-721 token.

The proposal would benefit from stronger guidance on cross-chain coordination (even if only as a design-space sketch), explicit EIP-1153 dependency, a higher default unlock delay for human-scale due diligence, and a canonical account-discovery helper for hierarchical trees. The surviving-approvals problem deserves more prominent treatment, perhaps including the standardized approval-tracking events suggested above.

Overall: this is a well-crafted primitive that will become infrastructure. The authors should be proud of it. Ship it.

# ERC-XXXX "Wallet Title Deeds" -- Technical Review

**Reviewer:** Tomasz K. Stanczak (channeled perspective)
**Date:** 2026-04-08
**Document version:** As of commit 54365bd5 on branch `nft-aa`
**Status of ERC:** Draft

---

## 1. What I Like About It

### The core insight is architecturally sound

The separation of control from custody is not a novelty -- it is how property law has worked for centuries. Title deeds to land do not move the land; they rotate authority over it. This ERC applies that model to smart accounts and gets it right in the places that matter most.

The `tokenId == uint256(uint160(account))` mapping is the single best design decision in the entire proposal. No registry. No lookup table. No central contract dependency. Pure arithmetic derivation means any client, any indexer, any contract can resolve the relationship without touching state. Compare this to ERC-6551's registry model where discovery requires querying a singleton -- that registry is both a performance bottleneck and a systemic dependency. This ERC has neither.

### The transfer lock state machine is unusually rigorous

Most ERC authors handwave the "what happens during transfer" question. This one does not. The `proposeUnlock -> unlockDelay -> completeUnlock -> transfer -> auto-relock` state machine, combined with the mutual exclusion between execution authorization and transfer readiness, is genuinely well thought through. The key insight -- that the drain window and the sale window must be provably disjoint -- is correct, and the mechanism achieves it.

The asymmetric meta-timelock on `setUnlockDelay` decreases is a defense-in-depth measure that shows the authors have thought about adversarial timelines. An attacker who configures a long unlock delay to build trust and then collapses it to 1 second is a realistic attack. The meta-timelock makes the collapse itself take as long as the current delay, which is the right answer.

### The execution-active transient guard is the right tool for the job

Using `TSTORE`/`TLOAD` for a reference-counted execution flag that the controller token checks on transfer is elegant. It costs nothing during normal execution (no cross-contract calls on every subcall), and it costs one `STATICCALL` on transfer (which is rare). The cost is borne by the rare operation, not the frequent one. This is good engineering.

### FOCIL/VOPS alignment is a real advantage

The direct-owner execution path being an ordinary transaction is not a marketing claim; it is a genuine architectural property. The ERC does not introduce a new mempool object type. It does not require nodes to simulate validation prefixes. The controlled account is the *destination* of a normal call, not a new sender type. This matters for public mempool health and for inclusion list mechanisms. The authors understand why this matters and explain it clearly, including the honest caveat that this property does not extend to ERC-4337 UserOperations.

### The approval revocation during freeze is a pragmatic addition

The tension between "execution must be frozen during the sale window" and "the seller needs to clean up approvals before handing over" is real. Solving it with hardcoded zero-approval-only revocation functions that cannot be used as a general execution path is the right tradeoff. It is a narrow, safe escape hatch from the freeze.

### The control version mechanism solves a real problem

Incrementing `controlVersionOf(tokenId)` on every transfer, and scoping all delegated authority to that version, means that transfer is a hard boundary. Prior signatures are dead. Prior validator authority is dead. This is mandatory and the ERC correctly identifies that omitting it would be a material weakness.

### Hierarchical nesting with bounded cycle detection

Allowing controlled accounts to themselves hold controller NFTs creates genuine organizational utility. The bounded cycle check (walk the ownership chain up to `maxNestingDepth()` hops per transfer) is the correct way to prevent cycles without making transfer gas costs unbounded. The default depth of 4 is justified: parent -> subsidiary -> department -> project covers real organizational structures, and the worst-case gas (4 * ~30k = ~120k) is acceptable for a rare operation.

### No central registry, no singleton dependency

The explicit decision to avoid a singleton registry contract is correct. Multiple independent implementations can coexist. The ecosystem chooses implementations based on audit quality and trust, the same way it chooses ERC-20 implementations. No single contract is a systemic dependency of the standard.

---

## 2. Issues and Concerns

### 2.1 The `ownerOf` call on every execution is a cross-contract SLOAD

Every call to `execute` or `executeBatch` must check `msg.sender == controllerToken.ownerOf(tokenId)`. That is a cross-contract `STATICCALL` that touches at least one storage slot in the controller token contract. At current gas prices, this is roughly 2600 gas (cold) or 100 gas (warm) per execution frame. For most use cases this is fine. But for high-frequency execution patterns -- an agent performing many small operations in rapid succession -- this adds up. The ERC should acknowledge this cost explicitly and recommend that implementations warm the slot where possible (e.g., by including the controller token address in access lists).

More importantly: the account "MUST NOT rely on a stale internal cache as the authoritative controller or authoritative control version." This is correct for security but means there is no way to amortize the lookup cost. Every execution frame pays it. For hierarchical execution where a parent account executes on a child account which executes on a grandchild, the verification cost multiplies at each level.

### 2.2 The unlock delay floor of 1 second is dangerously low

The ERC sets a default `unlockDelay` of 3600 seconds (one hour) but allows the owner to reduce it to a floor of 1 second. A 1-second delay is operationally equivalent to no delay on a 12-second block time. If `proposeUnlock` and `completeUnlock` can land in the same block (which they can, since 1 second < 12 seconds), the entire sell-and-drain protection collapses.

The meta-timelock on decreases means the reduction to 1 second takes `currentDelay` seconds to take effect, which is the intended defense. But once the delay is at 1, it stays at 1 forever for that token (increasing it is the owner's choice, not a protocol requirement). Any subsequent sale of that token has effectively zero protection. Marketplaces must check `unlockDelayOf(tokenId)` and treat low values as a red flag, but the ERC does not define a RECOMMENDED minimum for marketplaces to enforce.

**Suggestion:** The floor should be at least one full block time (12 seconds on mainnet), or the ERC should define a RECOMMENDED marketplace minimum and explain why values below one block time are dangerous. Alternatively, define a chain-specific floor as a function of expected block time.

### 2.3 Surviving token-level approvals are a material risk that the ERC can only partially mitigate

The ERC is honest about this: ERC-20 allowances, ERC-721 approvals, and ERC-1155 operator approvals granted by the account survive control transfer because they are stored on the external token contracts. The revocation interface helps, but it cannot enumerate what approvals exist. Discovery requires off-chain event indexing.

This is not a flaw in the ERC -- it is a fundamental limitation of Ethereum's approval model. But it means that every account sale or transfer is an incomplete handoff unless the buyer does extensive off-chain due diligence. For a standard that wants account transfer to be "a first-class operation," the gap between the clean control-rotation story and the messy approval-survival reality is significant.

**Suggestion:** The ERC should RECOMMEND that compliant accounts emit an event on every outbound approval grant (not just the standard Approval event on the token contract, but a dedicated account-level event like `ApprovalGranted(address token, address spender, uint256 amount)`). This would make account-level approval tracking possible through the account's own event log without relying on cross-referencing every token contract the account has ever interacted with.

### 2.4 The `initCalls` in `deployAccountConfigured` run as the factory, not as the owner

The specification says: "The initCalls batch MUST be executed with the same authorization as if initialOwner had called executeBatch directly." But the factory is the entity making the calls. The factory must therefore have a privileged code path in the account that allows it to execute on behalf of the initial owner during deployment only. This creates a trust assumption: the factory implementation must be correct, because a buggy factory could execute arbitrary calls on newly deployed accounts.

The ERC should explicitly state that this one-time factory privilege must be non-reentrant and must be permanently disabled after the deployment transaction completes. Otherwise, a factory with a bug or a malicious upgrade path could execute on any account it previously deployed.

### 2.5 Cross-chain state divergence is acknowledged but unsolved

The ERC correctly states that "cross-chain control is out of scope." User-salt stabilizes the deployment address, but ownership, delegated authority, validators, and recovery configuration are all chain-local. An account deployed at the same address on two chains has two independent control graphs. A user who transfers the controller NFT on one chain but not the other has created a state divergence that no one can resolve except by manual coordination.

This is a reasonable scope boundary for a Draft, but the ERC should be more explicit about the failure mode. A SHOULD-level recommendation that wallets surface which chains an account is deployed on and whether control is consistent across them would help.

### 2.6 The EIP-7702 delegation attack surface is larger than the Security Considerations suggest

The ERC's analysis of 7702 risks is good but incomplete. The key gap: a 7702-delegated EOA that holds controller NFTs can have its delegation changed by anyone who can submit a valid 7702 authorization for that EOA's key. The ERC notes that "no control-version increment" happens on delegation change, and that malicious delegated code can install validators that persist. But it does not discuss the case where the 7702 delegation is changed *back* to benign code after installing a malicious validator. The owner might never notice the validator was installed during the brief delegation window.

**Suggestion:** The ERC should RECOMMEND that accounts emit a validator-installation event that is visible to the owner's wallet even if the installation was performed by delegated code during a 7702 window. More generally, it should recommend that wallet implementations poll or subscribe to `ValidatorInstalled` events for all controlled accounts, not just react to user-initiated installations.

### 2.7 The `EXTCODEHASH` check in the execution-active transfer guard has a subtle edge case

The controller token must check that the account has deployed code before allowing transfer. The RECOMMENDED mechanism is `EXTCODEHASH`. But `EXTCODEHASH` returns `keccak256("")` for an account that exists but has no code (e.g., an account that has received ETH but has not been deployed). It returns `0x0` for an account that does not exist at all.

If an account is deployed, receives ETH, and then somehow has its code removed (impossible today under Constantinople rules that removed `SELFDESTRUCT` code deletion in most cases, but potentially relevant under future EVM changes), the `EXTCODEHASH` check would correctly block transfer. Good. But the ERC should note that this check is dependent on the EVM's code-presence guarantees and may need revision if future EVM changes alter `SELFDESTRUCT` or code-deletion semantics. The ERC already says "MUST NOT include a reachable SELFDESTRUCT path," which is the right defense, but the dependency on EVM-level code permanence should be called out as an assumption.

### 2.8 Gas cost analysis for batch operations is missing

The ERC defines `executeBatch` as atomic, meaning all-or-nothing. For large batches (e.g., revoking 50 approvals during the freeze window, or performing a complex DeFi rebalance across 10 protocols), the gas cost can exceed block gas limits. The ERC does not discuss what happens when a batch is too large for a single block.

**Suggestion:** The ERC should note that implementations SHOULD NOT impose an artificial batch size limit but SHOULD document the practical gas ceiling. Applications building large batches should be aware that block gas limits bound the batch size. For the revocation interface specifically, if an account has hundreds of outstanding approvals, batch revocation may require multiple transactions -- which means the freeze window must be long enough to accommodate the cleanup.

### 2.9 The validator interface is underspecified for critical paths

The `IERCXXXXValidator` interface defines `onInstall`, `onUninstall`, and `isValidSignatureForAccount`. But it does not define:

- How a validator signals what *kind* of signatures it can validate (passkey? secp256r1? multisig threshold?)
- How an account chooses which validator to delegate to when multiple are installed
- What happens if two validators both claim to validate the same signature scheme
- Whether validators can have internal state that is mutable by the account controller
- Gas limits for `isValidSignatureForAccount` -- an unbounded validation call is a DoS vector for ERC-1271 callers

The ERC says validators are OPTIONAL and leaves the details to implementations. This is reasonable for a Draft, but the validator interface needs more specification before this can move beyond Draft. As written, two different implementations could install validators that are completely incompatible, which defeats the purpose of standardization.

### 2.10 The `resetDelegations` function increments controlVersion but does not invalidate the transfer approval version

`resetDelegations(tokenId)` increments `controlVersionOf(tokenId)`, which invalidates all delegated validator authority. But the specification says transfer approval version is incremented on transfer, `proposeUnlock`, and `lock` -- not on `resetDelegations`. This means a `resetDelegations` call does not invalidate outstanding ERC-721 single-token approvals on the controller token.

Is this intentional? If the owner calls `resetDelegations` because they believe their delegated authority has been compromised, they might also want to invalidate any existing transfer approvals on the controller token. The current design leaves those approvals valid, which could allow a party approved for transfer to complete the transfer even after the owner has reset delegations.

### 2.11 No account-level nonce for off-chain action ordering

The ERC explicitly states that it does not require an account-level execution nonce for direct ordinary transactions, relying instead on the owner account's own replay model. This is fine for direct execution. But for off-chain flows that go through ERC-1271 (e.g., Permit2, off-chain order signing for DEXs), there is no account-level nonce to enforce ordering or prevent replay of signed messages within the same control version.

The ERC says "Applications SHOULD include their own message nonce or deadline semantics inside the original signed payload when needed." This pushes the problem to application authors, which is honest but may lead to inconsistent replay protection across the ecosystem. A RECOMMENDED account-level nonce for ERC-1271 flows would be more robust.

---

## 3. Usage Scenarios I Find Compelling

### 3.1 Organizational treasury management

This is the killer use case. A DAO or corporate treasury controlled by a multisig holds the controller NFT. The treasury address, its Aave positions, its Uniswap LP, its ENS name, its protocol allowlists -- all survive a committee rotation. The outgoing committee transfers the NFT to the new multisig. Done. No multi-governance-vote asset migration. No updating every protocol integration that references the old address. One NFT transfer replaces weeks of coordination.

### 3.2 Approval-scoped risk isolation

The ability to have multiple accounts under one controller, where each account is a separate custody address with separate approvals, is genuinely useful today. Users currently achieve this by managing multiple EOAs with separate seed phrases. This ERC makes it possible with one key and multiple controller NFTs. The high-risk "DeFi exploration" account has unlimited approvals to experimental protocols; the "cold storage" account has none. A compromise of the experimental protocol cannot reach the cold storage account. This is the same security model, without the UX burden of multiple keys.

### 3.3 Digital inheritance

The controller NFT held by a dead man's switch or multisig with designated heirs is a genuinely practical inheritance mechanism. The entire on-chain estate -- assets, positions, memberships, and address-based identity -- transfers through a single NFT transfer. No shared seed phrases. No centralized custodian. No probate court arguing about who controls a private key. The assets remain at the same address, so every protocol integration continues to work. This is one of the few inheritance solutions that does not require the heir to know anything about the predecessor's key management.

### 3.4 Account sale with verifiable history

An account with years of protocol interaction history, a clean governance participation record, and valuable address-based reputation (protocol allowlists, early-user benefits, airdrop eligibility) can be sold as a unit. The buyer gets the entire history at the same address. This is genuinely novel -- today, selling an "account" means sharing a private key, which is both insecure and impractical. This ERC makes it a standard NFT sale with all the existing marketplace infrastructure.

### 3.5 Vesting and escrow with transferable claims

An account locked by a vesting contract (which holds the controller NFT and releases it on schedule) keeps the vesting assets at a fixed address. But the beneficiary's claim to those assets -- the right to eventually control the account -- can be sold by transferring the controller NFT from one vesting wrapper to another, or by the vesting contract implementing its own secondary market. The grantor never releases tokens early. The assets remain locked. But the economic interest in those assets is liquid. This is a meaningful improvement over current vesting models.

---

## 4. Additional Possibilities Enabled

### 4.1 Protocol-level credit scoring and reputation

Because accounts persist across control rotations, an account's entire interaction history is permanently tied to its address. This enables credible on-chain reputation: the account at address `A` has never been liquidated in 3 years of DeFi usage, has participated in 200 governance votes, and has maintained a collateralization ratio above 200% through two bear markets. That reputation belongs to the address, not to any particular controller. An under-collateralized lending protocol could use this history as a basis for credit decisions.

The second-order effect: reputation becomes *transferable* via account sale. This is both an opportunity (accounts with good history are more valuable) and a risk (reputation laundering -- buy a clean account, use it for bad things, sell it before anyone notices). Protocol designers building on this should consider whether reputation should be degraded or reset on control transfer, using `ControlVersionChanged` events as the trigger.

### 4.2 Automated agent accounts with hard spending caps

The ERC's child-account model naturally supports AI agent wallets. An agent operates a child account funded with only the assets needed for its task. The parent account (controlled by a human) holds the controller NFT for the child. The agent has a session key or validator on the child account. If the agent is compromised or misbehaves, the damage is bounded by the child's balance. The parent can revoke the agent's validator via `resetDelegations` without transferring the NFT. This is materially better than giving an agent a session key on a large account with permission scoping as the only defense.

### 4.3 Composable account modules via the hierarchy

The hierarchical nesting model enables a design pattern the ERC does not explicitly discuss: account-as-module. A parent account holds child accounts that each serve a specific purpose -- one for staking, one for lending, one for trading. Each child has its own validators, its own approval scope, its own risk profile. The parent account's `executeBatch` can orchestrate across children atomically. This is a composable account architecture that does not require a module system or plugin standard -- the module boundary is the account boundary.

### 4.4 On-chain fund structures

A fund manager controls a parent account. The parent holds controller NFTs for child accounts, each representing a strategy or asset class. Investors buy shares in the fund (via a separate mechanism) and the fund manager operates the children. The entire fund can be transferred to a new manager via a single NFT transfer of the parent's controller token. The children, their assets, their positions, and their histories all persist. This is a genuine building block for on-chain asset management.

The ERC's regulatory considerations correctly note that fractionalization of the controller token may change the legal characterization. But the fund structure described here does not require fractionalization of the controller token -- it requires a separate investment wrapper that maps investor shares to fund performance. The controller token remains held by a single entity (the fund manager or a governance contract).

### 4.5 Cross-protocol identity without ENS dependency

Today, on-chain identity is fragmented: ENS names, Lens profiles, Farcaster IDs, protocol-specific profiles. Because ERC-XXXX accounts persist across control rotations, the account address itself becomes a stable identity anchor. Protocols that reference accounts by address (governance systems, allowlists, airdrops, credit scoring) automatically reference the right entity regardless of who controls it. The controller NFT's metadata extension (name and avatar) adds a human-readable layer without requiring ENS. This does not replace ENS, but it provides an alternative identity anchor that is natively tied to the account's execution history.

### 4.6 Programmable account lifecycle contracts

Because the controller NFT is a standard ERC-721 token, it can be held by *any* contract. This enables account lifecycle patterns that the ERC does not explicitly discuss:

- **Auction accounts:** A newly deployed account is auctioned; the auction contract holds the controller NFT and transfers it to the winner. The account may be pre-funded or pre-configured with validators.
- **Rental accounts:** A rental contract holds the controller NFT and grants temporary execution authority (via validator installation) to a renter. When the rental period expires, the validator is revoked.
- **Insurance accounts:** An insurance contract holds the controller NFT as collateral. If the policy terms are violated, the insurer can seize the account.
- **Conditional release:** A controller NFT held by a contract that releases it only when an oracle condition is met (e.g., a milestone in a grant program, a vesting cliff, a regulatory approval).

### 4.7 MEV-aware account design

The hierarchical model enables a MEV mitigation pattern: a parent account submits a batch that operates on multiple child accounts atomically. Because the entire batch is one transaction, there is no MEV opportunity between the individual operations. A user who currently performs `approve on account A -> swap on account A -> deposit on account B` as separate transactions exposes each gap to MEV extraction. With this ERC, the parent account's `executeBatch` can do all of this atomically, eliminating inter-transaction MEV. Intra-transaction MEV (e.g., sandwich attacks around the swap) still exists, but the surface area is reduced.

---

## 5. Changes to Increase Optionality

### 5.1 Define an account-level event log for approval grants

As discussed in section 2.3, surviving token-level approvals are the biggest practical risk in account transfer. The ERC should define an OPTIONAL `ApprovalGranted(address indexed token, address indexed spender, uint256 amount)` event that compliant accounts emit whenever `execute` or `executeBatch` makes an outbound approval call. This is detectable by inspecting the calldata of outbound calls (the first 4 bytes of the data field for known approval function selectors). The gas cost is one additional LOG per approval, which is small compared to the approval itself.

This would make account-level approval tracking possible without off-chain cross-referencing of every token contract the account has ever touched. Marketplaces and wallets could build approval dashboards from the account's own event log.

### 5.2 Define a RECOMMENDED minimum `unlockDelay` floor tied to block time

The current floor of 1 second is too low. A floor of `2 * expectedBlockTime` (24 seconds on mainnet) would guarantee that `proposeUnlock` and `completeUnlock` cannot land in the same block under any circumstances. The meta-timelock on decreases still protects against rapid collapse to the floor, but the floor itself should be safe. The ERC should allow this floor to be chain-specific (different L2s have different block times).

### 5.3 Add an OPTIONAL account-level nonce for ERC-1271 flows

While the ERC correctly avoids requiring an execution nonce for direct transactions, off-chain signature flows through ERC-1271 would benefit from a standardized account-level nonce. Define an OPTIONAL `signatureNonce() -> uint256` that implementations MAY expose, and RECOMMEND that ERC-1271 signatures include this nonce in the signed payload. This gives application authors a standard replay-protection mechanism without requiring each application to invent its own.

### 5.4 Define validator capability advertisement

The validator interface should include a RECOMMENDED `supportsValidation(bytes4 validationType) -> bool` or equivalent capability query. Without this, an account with multiple validators has no standard way to route a validation request to the correct validator. The account must try each validator in order, which is both gas-wasteful and nondeterministic. A capability advertisement mechanism would let the account (or its ERC-1271 implementation) route directly.

### 5.5 Define an OPTIONAL `emergencyLock` callable by anyone

Currently, `lock(tokenId)` is callable only by `ownerOf(tokenId)`. If the owner's key is compromised, the attacker can `proposeUnlock`, wait the delay, `completeUnlock`, and transfer the NFT. The owner (if they still have access to the key) can call `lock` to cancel, but this is a race condition.

An OPTIONAL `emergencyLock(tokenId, proof)` callable by anyone who provides a valid proof (e.g., a signature from a pre-registered guardian) would allow third parties to cancel a malicious unlock without having the owner's key. This is complementary to the OPTIONAL recovery mechanism but operates on a faster timescale -- it only cancels the unlock, it does not transfer the NFT.

### 5.6 Consider a `transferWithCallback` pattern for atomic marketplace settlement

The current transfer flow requires: `proposeUnlock -> wait -> completeUnlock -> approve marketplace -> marketplace calls transferFrom`. For atomic marketplace settlement, the buyer wants to pay and receive the NFT in one transaction. But the seller must have pre-approved the marketplace, and the marketplace must call `transferFrom` after receiving payment. This is the standard NFT marketplace flow, but it has a timing gap between the buyer's payment and the NFT transfer.

An OPTIONAL `transferWithCallback(tokenId, recipient, callbackTarget, callbackData)` that atomically transfers the NFT and calls back to a settlement contract would enable truly atomic marketplace settlement. The callback executes after the transfer, so the settlement contract can verify that the transfer succeeded before releasing payment to the seller.

### 5.7 Make `maxNestingDepth` configurable per-deployment rather than global

The current design exposes `maxNestingDepth()` as a view on the controller token, implying it is a global setting. For some deployments, a depth of 2 is sufficient and the gas savings on transfer checks are meaningful. For others, a depth of 8 is needed for complex organizational structures. Making this a per-token or per-deployment configuration would increase flexibility without changing the cycle-detection algorithm.

### 5.8 Define explicit interaction with ERC-4337 `validateUserOp`

The ERC says accounts "MAY additionally implement ERC-4337" and that "authorization MUST be bound to the current control version." But it does not define how. Should `validateUserOp` check `controlVersionOf(tokenId)` and reject operations signed under a prior version? Should the 4337 nonce space be partitioned to include the control version? Should the `initCode` for counterfactual deployment use this ERC's factory? These are implementation-critical questions that the ERC should answer at least at the RECOMMENDED level, because ERC-4337 is the dominant smart-account execution framework and inconsistent integration will fragment the ecosystem.

### 5.9 Consider an OPTIONAL `delegateExecute` for parent-to-child without full `execute`

When a parent account operates a child account, it currently does so via `execute(childAccount, 0, abi.encodeCall(childAccount.execute, ...))`. This double-wrapping is verbose and gas-expensive. An OPTIONAL `delegateExecute(parentTokenId, childTokenId, target, value, data)` on the account or the controller token that verifies the parent-child ownership relationship and forwards the call would be more gas-efficient and less error-prone for hierarchical execution patterns.

### 5.10 Specify behavior when the controller token contract itself is upgraded

The ERC says "upgradeability is not core to this ERC" but acknowledges it as a trust assumption. If the controller token contract is behind a proxy and is upgraded, the entire control model for every account depends on the upgrade being correct. The ERC should define what invariants an upgrade MUST preserve (e.g., `ownerOf` mappings must not change, `controlVersionOf` must not decrease, locked tokens must remain locked) and RECOMMEND that controller token contracts be either non-upgradeable or governed by a timelock with a delay at least as long as the maximum `unlockDelay` of any token in the collection.

---

## Summary Assessment

This is a well-engineered ERC that solves a real problem. The core model -- NFT as title deed, account as custody address, transfer as control rotation -- is sound and architecturally clean. The transfer lock state machine, the control version mechanism, and the FOCIL/VOPS alignment show that the authors understand the infrastructure implications of their design, not just the application-layer story.

The main concerns are at the edges: surviving token-level approvals that cannot be enumerated on-chain, the dangerously low `unlockDelay` floor, underspecification of the validator interface, and the lack of explicit ERC-4337 integration guidance. None of these are fatal. All of them are addressable in subsequent revisions.

The use cases are genuine. Organizational treasury management, digital inheritance, approval-scoped risk isolation, and account sale with verifiable history are all meaningfully enabled by this standard in ways that are not achievable with current tooling. The hierarchical nesting model opens further composition possibilities (agent accounts, fund structures, account-as-module patterns) that the authors may not have fully explored.

The standard is honest about what it does not solve: cross-chain state synchronization, privacy, and the fundamental limitation of Ethereum's approval model. That honesty is a strength. The worst ERCs are the ones that claim to solve everything.

If I were advising the authors, I would say: tighten the `unlockDelay` floor, add the account-level approval event, specify the validator capability advertisement, define the ERC-4337 integration at RECOMMENDED level, and ship a reference implementation with a test suite that covers the transfer lock state machine exhaustively. The design is ready for that.

The grade is not encouragement. It is a timestamp. As of 2026-04-08, this is a strong Draft that addresses its problem space with more rigor than most ERCs at this stage. The gap between "specified" and "running in production" is where the real test happens. I would like to see a headless simulation stress test: deploy 1000 accounts in a hierarchy, perform 10,000 executions across the tree, transfer 100 controller NFTs, and measure gas costs, state growth, and edge-case behavior under load. That would convert this from a good specification into evidence.

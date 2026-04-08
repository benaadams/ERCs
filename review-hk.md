# ERC-XXXX: Wallet Title Deeds -- Design Review

**Reviewer perspective: thematic compression, system semantics, artistic stakes, interactive-medium analysis.**

---

## I. What This Design Is Actually Saying

Before discussing strengths or weaknesses, I want to name the core proposition, because the document does not state it this cleanly:

**Ownership is not possession. Ownership is the right to act.**

Every consequential design decision in this ERC flows from that single insight. The account holds the assets (possession). The NFT confers the right to operate that account (ownership). These are separated, and the separation is the entire point. This is not a technical convenience. It is a philosophical position about what "having a wallet" means, encoded into Ethereum's object model.

The closest real-world analogy is not a key to a house. It is a title deed to land. The land does not move when the deed changes hands. The buildings on the land do not move. The tenants, the easements, the zoning -- none of it moves. Only the right to act upon it transfers. The authors chose this metaphor deliberately, and it is the correct one.

---

## II. What I Like

### 1. The Separation is Load-Bearing, Not Decorative

Many smart-account proposals separate "control" from "custody" in their documentation and then fail to make that separation matter mechanically. This ERC makes it matter. The transfer lock, the control version, the execution freeze during unlock, the cycle detection, the approval revocation interface that works during the freeze -- every one of these exists because the separation between the deed and the land creates real edge cases, and the authors confront each one instead of hand-waving.

The sell-and-drain analysis is particularly rigorous. The authors identified that the dangerous window is not the moment of transfer but the entire period between "I intend to transfer" and "transfer completes." The execution freeze from `proposeUnlock` onwards, combined with the transfer-approval-version increment that invalidates marketplace approvals when the seller cancels via `lock`, creates a genuine mutual exclusion between "can drain" and "can sell." This is not just a safety feature. It is a statement about what an honest transaction looks like: you cannot simultaneously prepare to hand over control and exercise that control. The design forces the seller to choose.

### 2. Hierarchical Nesting is the Quiet Revolution

The document spends most of its motivational energy on the single-account transfer case. But the more consequential design space is the nesting. An account can hold controller NFTs for other accounts. A parent can atomically rebalance across children via `executeBatch`. Transfer of a child's controller NFT is a divestiture. Transfer of the parent's controller NFT transfers the entire tree.

This is corporate structure as a protocol primitive. Not simulated. Not approximated. Actual hierarchical control with actual isolation boundaries. The four-hop default nesting depth (parent, subsidiary, department, project) maps exactly to how organizations actually structure authority and risk.

The cycle detection on transfer -- walk the ownership chain, reject if it loops -- is the kind of invariant that seems obvious in retrospect and is brutally easy to get wrong in practice. The bounded gas stipend per hop (30k gas, max 4 hops) makes the cost predictable and the attack surface bounded.

### 3. The Address-Identity Preservation

This deserves its own section because it solves a problem that the Ethereum ecosystem has been working around for years.

When you change who controls an account today, you change the account's address. Your ENS name, your governance voting history, your Aave health factor, your protocol allowlist membership, your on-chain reputation -- all of it is address-bound. Rotating keys means abandoning identity. This ERC makes key rotation a metadata update on the control layer while the identity layer (the address) stays fixed.

"Key rotation without identity loss" is the formulation. Five words. That is what this system says to every user who has ever been afraid to change their security model because their entire on-chain life is welded to one address.

### 4. The FOCIL/VOPS Alignment is Not Just Technical Positioning

The authors go to considerable length explaining why the direct-owner execution path is an ordinary transaction rather than a new transaction type. This is not pedantry. It is a bet about what the Ethereum public mempool will look like in two years.

If VOPS narrows what state nodes must retain for local transaction validation, and if FOCIL builds inclusion lists from the public mempool, then any account-abstraction scheme that requires EVM simulation to determine mempool admissibility is swimming against the current. This ERC's direct-owner path requires no such simulation. The transaction is valid or invalid by the same rules that govern any contract call. The authors are not just building for today's infrastructure. They are building for the infrastructure that the research community is converging toward.

### 5. The Approval Revocation Interface During Freeze

This is a small piece of the design that reveals careful thinking. During the execution freeze (after `proposeUnlock`), the account cannot execute arbitrary calls. But it CAN revoke approvals -- and only revoke them, with hardcoded zero-value calldata. This means a seller preparing an account for transfer can clean up outstanding approvals without breaking the execution freeze.

The insight: "revoking an approval can only reduce risk, never create it." By restricting the interface to zero-value approval calls, the design gets the practical benefit (cleanup before handoff) without reopening the drain window. The constraint is the feature.

---

## III. Issues

### 1. The Surviving-Approvals Problem is Not Solved, Only Documented

The most dangerous sentence in this ERC is: "Token-level approvals that the account previously granted to third-party contracts survive the transfer."

The document handles this honestly. It provides the revocation interface. It recommends wallets surface outstanding approvals. But the fundamental problem remains: there is no on-chain enumeration of approvals. The new controller cannot know what they do not know. The revocation interface requires you to specify which token and which spender -- you must already know the approval exists to revoke it.

This is the single largest footgun in the entire design. Someone buys an account through an NFT marketplace. They see the ETH balance, the token balances, the NFT collection. They do not see the unlimited USDC approval to an obscure DeFi contract that the previous owner left in place. That approval is a ticking bomb, and the new owner has no on-chain way to discover it.

**Recommendation:** The ERC should RECOMMEND (or ideally REQUIRE) that wallet and marketplace implementations perform off-chain approval scanning and present a "known approvals" list as part of the transfer UX. The ERC should also explicitly note that batch-revoking ALL known approvals is the safest first action for any new controller. Consider adding a convenience method that takes a list of common token addresses and revokes all known approval patterns. The ERC cannot solve Ethereum's lack of on-chain approval enumeration, but it can be louder about the risk.

### 2. The Unlock Delay Floor of 1 Second is Effectively Zero

The spec says: "The owner MAY reduce to a floor of 1 via setUnlockDelay." One second is not a meaningful delay. It is one block on most L2s and sub-block on Ethereum post-slot auctions. A floor of 1 second, combined with the meta-timelock on decreases, means an attacker who compromises the owner can reduce the delay to 1 second, wait out the meta-timelock, and then execute a near-instantaneous drain-and-transfer.

The meta-timelock on decreases helps -- the attacker must wait the current delay before the reduction takes effect. But once the reduction is effective, the 1-second floor means `proposeUnlock` and `completeUnlock` can happen in consecutive blocks. For high-value accounts, this is uncomfortably fast.

**Recommendation:** Consider whether the floor should be higher -- perhaps 60 seconds or 300 seconds -- or whether the floor should be configurable per-deployment rather than fixed in the standard. The one-hour default is good. The one-second floor undermines it for accounts that reduce the delay and then get compromised.

### 3. EIP-7702 Is a Hole in the Control-Version Model

The Security Considerations section is admirably honest about this: "controlVersionOf(tokenId) does not change when the owner's 7702 delegation changes." Validators installed by malicious delegated code persist. Pending unlock-delay reductions persist. The control version, which is the design's primary invalidation mechanism, is blind to 7702 delegation changes.

This is not a bug in the ERC. It is a fundamental mismatch between two models of account identity. But the practical consequence is that a 7702-compromised EOA owner can install persistent backdoors in every account it controls, and revoking the delegation does not clean them up.

The ERC suggests `resetDelegations(tokenId)` as the remedy, but this requires the user to know they were compromised and to remember to call it for every account they control. In the correlated multi-account case (one EOA holding many controller NFTs), the cleanup burden scales linearly with the number of accounts.

**Recommendation:** Consider a mechanism where the account can optionally pin the owner's `EXTCODEHASH` at control-version establishment time and reject execution if it changes. This is suggested in the Security Considerations but not standardized. Standardizing it as an OPTIONAL mode would give implementations a clean way to opt into 7702 protection.

### 4. No Standard for Account Discovery Beyond Enumerable

The ERC recommends ERC-721 Enumerable but does not require it. Without Enumerable, discovering which accounts an address controls requires off-chain event indexing. For the hierarchical nesting use case -- which is one of the design's strongest features -- this means the tree structure is invisible on-chain unless you can enumerate children.

For wallets rendering the hierarchy, this is a UX problem. For smart contracts that need to reason about the tree (governance contracts, for example, that want to know if an address controls any subsidiary accounts), this is a capability gap.

**Recommendation:** Require Enumerable. The gas cost of maintaining the enumeration data structure is paid once at mint and transfer. The benefit -- on-chain discoverability of the control tree -- is essential to the hierarchical use case.

### 5. Cross-Chain Story is Explicitly Absent

The ERC says: "Cross-chain control is out of scope." This is honest but painful. The user-salt deployment mode gives you the same address on every chain, but the controller NFT is chain-local. You have the same address on Arbitrum, Optimism, and mainnet, controlled by three different NFTs that may be held by three different owners.

This is not the ERC's job to solve. But the ERC should be clearer about the mental model: "same address" does not mean "same controller" across chains. Users who deploy with user-salt and assume cross-chain control equivalence will be wrong.

### 6. The Privacy Section Understates the Metadata Leakage

The ERC correctly notes that controller-token ownership is public. But the hierarchical nesting makes this worse: the entire corporate structure is legible on-chain. Parent-child relationships, organizational depth, which accounts are siblings -- all of this is visible to anyone who reads the ownership chain.

For organizations that want the operational benefits of hierarchical accounts but do not want their corporate structure publicly legible, this is a showstopper. The ERC should acknowledge this more directly and note that privacy-sensitive deployments may need to avoid nesting or use intermediary non-controlled accounts to break the legibility chain.

### 7. No Standardized Emergency Freeze

If a controller NFT is stolen but recovery has not been configured, the attacker has full control. The unlock delay prevents immediate transfer of the NFT itself, but the attacker can execute freely on the account (drain assets) while the NFT is locked. There is no standardized "freeze execution" mechanism that a third party (guardian, DAO, protocol) could invoke.

Recovery is OPTIONAL and not standardized. This means every implementation will have a different freeze/recovery interface, or none at all. For a standard that treats the NFT as a root credential, the absence of a standardized emergency response is a gap.

**Recommendation:** Consider standardizing an OPTIONAL emergency freeze interface that can be triggered by pre-configured guardians. This would not bypass NFT ownership -- it would only pause execution until the owner or recovery mechanism resolves the situation.

---

## IV. Usage Scenarios That Excite Me

### 1. DAO Treasury Handoff as a Single Transaction

A DAO votes to change its treasury committee. Today this requires migrating every asset, updating every protocol integration, re-establishing every allowlist entry. Under this ERC, the outgoing committee transfers one NFT. The treasury address, its entire history, every protocol relationship -- all preserved. The new committee is immediately operational at the same address.

This is not just more efficient. It changes what a "DAO treasury" IS. It becomes a persistent institution rather than a temporary arrangement around a set of keys. The treasury outlives its stewards.

### 2. Fund Management With Atomic Rebalancing

A fund manager controls a parent account. The parent holds controller NFTs for child accounts representing different strategies: a DeFi yield account, a blue-chip hold account, a trading account. The parent can atomically rebalance across all children via `executeBatch` -- sweep profits from the yield account, fund the trading account, all in one transaction.

Transferring the entire fund to a new manager is one NFT transfer of the parent's controller token. The new manager inherits the full position structure, the strategy separation, and the risk isolation.

### 3. Digital Inheritance Without Shared Secrets

The controller NFT is held by a dead-man's-switch contract. If the owner does not check in for 90 days, the contract transfers the NFT to a designated heir. The heir receives the entire on-chain estate -- every token, every position, every membership, every address-based relationship -- without ever knowing a seed phrase.

This is the first credible path to digital inheritance that does not require trusted custodians or shared secrets. The dead-man's-switch contract is auditable. The transfer is atomic. The heir's control is immediate and complete.

### 4. Approval-Isolated Interaction Accounts

A user holds their life savings in Account A. They want to try a new, unaudited DeFi protocol. They deploy Account B, fund it with only what they are willing to lose, and interact with the risky protocol from Account B. Both accounts are controlled by the same EOA -- no additional keys to manage.

If Account B is compromised through an approval exploit, the damage is bounded by Account B's balance. Account A's assets are at a different address and have never granted approvals to the compromised protocol. This is the security benefit of hardware wallet separation, achieved without hardware wallet separation.

### 5. Vesting Without Key Sharing

An employee receives tokens that vest over four years. Today, the tokens are locked in a vesting contract. Under this ERC, the tokens are held by an account whose controller NFT is held by a vesting contract. The employee cannot access the account until the vesting schedule completes and the vesting contract transfers the NFT.

But here is the interesting part: the employee can SELL the vesting position by finding a buyer for the vesting contract's release rights, without the employer releasing tokens early. The assets stay locked. The right to eventually control them changes hands. This is secondary-market vesting, which does not exist today.

---

## V. Additional Possibilities the Authors May Not Be Thinking About

### 1. On-Chain Credit and Collateralized Account Lending

A lender holds the controller NFT for an account as collateral. The borrower operates the account through an installed validator with spending limits. If the borrower defaults, the lender already holds the controller NFT and can liquidate the account's assets.

This inverts the current DeFi collateral model. Instead of depositing specific tokens as collateral, you deposit CONTROL of an account. The account can hold diversified assets, LP positions, staked tokens -- anything. The collateral is the account itself, not a specific token. Liquidation does not require unwinding positions; the lender simply takes control.

This creates a new primitive: the collateralized account as a credit instrument. The unlock-delay mechanism even provides a grace period: the lender cannot immediately drain the account (it is locked), giving the borrower time to cure default.

### 2. Account-Level Governance Delegation

Today, governance delegation is per-token. Under this ERC, a user who controls multiple accounts can delegate governance rights at the account level by transferring a child account's controller NFT to a governance delegate. The delegate controls the child account and can vote with its tokens, but cannot access the parent account's other assets.

This is scoped governance delegation with hard boundaries, not permission policies. The delegate controls exactly one account and can do exactly what that account's assets allow. Revoking delegation is transferring the NFT back.

### 3. AI Agent Accounts With Hard Spending Caps

An AI agent needs to operate on-chain. You deploy a child account, fund it with a limited budget, and give the agent a validator or session key on that account. The agent can operate freely within the child account's balance. The parent account is untouched.

This is the correct architecture for autonomous agent spending. The permission boundary is not a policy that the agent's code must respect -- it is a balance cap that Ethereum enforces. The agent literally cannot spend more than the child account holds, regardless of what its code does.

### 4. Transferable Airdrop Eligibility

An account's on-chain history (transaction count, protocol interactions, governance participation) determines airdrop eligibility. Under this ERC, that history is address-bound and survives control transfer. An account with strong airdrop eligibility criteria can be transferred to a new owner who inherits the eligibility.

This creates a secondary market for on-chain reputation, which is philosophically uncomfortable but economically inevitable. The ERC should at least acknowledge this possibility, because protocol designers who use address-based eligibility criteria will need to understand that control transfer changes who benefits from that history.

### 5. Time-Locked Organizational Succession

A founder holds the controller NFT for a company's master account. The NFT is held by a contract that implements a succession schedule: after a certain date, or upon a board vote, the NFT transfers to the next generation of leadership. The entire organizational infrastructure -- treasury, subsidiary accounts, protocol positions, vendor relationships -- transfers atomically.

This is corporate succession as a protocol event. No lawyers, no probate, no asset-by-asset transfer. One NFT transfer, one block, complete handoff.

### 6. Composable Account Packages as Products

With `deployAccountConfigured`, a developer can create a pre-configured account template: specific validators, specific child-account structure, specific protocol integrations, specific approval patterns. Deploy it, hand it to a user, they have a fully operational financial structure on day one.

This means financial products can be delivered as account configurations. A "DeFi starter pack" is not a website -- it is an account with pre-installed strategy accounts, pre-approved routers, and pre-configured risk isolation. The product IS the account.

---

## VI. Changes to Increase Optionality

### 1. Standardize an Account Metadata Extension

The controller token metadata (name, image) is OPTIONAL and cosmetic. But the account itself has no standardized metadata. Consider an extension where the account can expose structured metadata: its purpose, its risk tier, its relationship to parent/child accounts, its intended use case.

This is not vanity. It is the difference between a tree of opaque addresses and a legible organizational chart. Wallets cannot present meaningful hierarchies without semantic metadata about what each account is FOR.

### 2. Add a Standardized "Account Health" View

A single view function that returns whether the account has: (a) outstanding approvals to known-dangerous contracts (requires an oracle or registry), (b) a pending unlock-delay decrease, (c) active validators from a prior control version that should have been cleaned up, (d) any other detectable anomaly.

This does not need to be comprehensive. Even a partial health check would dramatically improve the UX of account acquisition. A buyer checking `accountHealth(tokenId)` before purchase is better than a buyer checking nothing.

### 3. Allow Controller Token Contract Upgradability Path

The ERC says: "Upgradeability adds trust assumptions. Implementations SHOULD prefer immutable logic." This is correct for the account implementation. But the controller token contract itself may need to evolve. What happens when ERC-721 gets a successor? What happens when a new lock mechanism is standardized?

Consider an OPTIONAL upgrade path for the controller token contract that requires supermajority consent of current token holders (or a time-locked governance process). Without this, the controller token contract is frozen forever, which is safe but brittle.

### 4. Standardize Cross-Account Messaging

When a parent account controls child accounts, the only way to coordinate between children is through the parent's `executeBatch`. Consider standardizing a lightweight cross-account messaging interface where child accounts can emit structured messages that siblings or the parent can read.

This enables reactive child accounts: a child account that automatically rebalances when it receives a message from a sibling, or a parent account that sweeps profits when a child signals above a threshold. Without standardized messaging, all coordination must be orchestrated by the parent, which limits composability.

### 5. Define a "Read-Only Controller" Role

Currently, the only roles are: controller (full execution authority) and validator (delegated signature validation). Consider a standardized read-only role: an address that can call view functions on the account and enumerate its state, but cannot execute.

This enables auditors, compliance systems, and monitoring services to inspect accounts without requiring execution authority. The current design requires either full control or off-chain indexing. A standardized read-only role fills the gap.

### 6. Standardize Batch Transfer of Controller NFTs

When transferring an entire organizational tree, the parent transfers one NFT (its own controller). But when splitting an organization -- transferring some children but not others -- each child requires a separate unlock and transfer cycle. Consider a batch-unlock and batch-transfer mechanism on the controller token that can atomically move multiple controller NFTs to different recipients.

This is divestiture as a first-class operation, not a repeated single-transfer loop.

### 7. Optional "Transfer Hook" on the Account

When the controller NFT transfers, the account does not execute any code. The control version increments, validators are invalidated, but the account itself does not get a callback. Consider an OPTIONAL transfer hook where the account can execute pre-configured cleanup logic on control rotation: auto-revoke specific approvals, disable specific integrations, emit specific notifications.

The ERC currently avoids transfer hooks ("avoid transfer hooks from the controller token into the account" in the reference implementation guidance). This caution is warranted -- hooks during transfer are a reentrancy risk. But a post-transfer hook, executed by the NEW controller's first action, could be standardized as an OPTIONAL "onControlTransfer" initialization pattern.

---

## VII. The Highest-Stakes Question

This ERC is infrastructure. Infrastructure shapes behavior. The behavior it shapes is: how humans relate to digital ownership, control, and identity on Ethereum.

The design says: your identity (address) persists. Your control (NFT) transfers. Your assets (account) stay put. This is a healthy decomposition. It acknowledges that identity is not the same as control, and control is not the same as possession. Most of the Ethereum ecosystem conflates all three into a single private key. This ERC pulls them apart.

But pulling them apart creates a new kind of risk: the risk that people will treat accounts as commodities. An account with a rich transaction history, strong governance reputation, and deep protocol relationships becomes a tradable asset. The authors know this -- the vesting, inheritance, and sale scenarios are all in the document. But the second-order effects deserve scrutiny.

When accounts become tradable, reputation becomes tradable. When reputation becomes tradable, reputation stops meaning what it meant. Every protocol that uses address-based eligibility (airdrops, governance weight, allowlist membership) will need to reckon with the fact that the address's history may not belong to its current controller.

This is not a flaw in the ERC. It is a consequence of the ERC's core proposition: ownership is the right to act, and rights can transfer. The authors should acknowledge this consequence explicitly, not as a risk, but as a feature of the world they are building. The protocols that depend on address-based reputation will need to adapt. The ones that adapt well will build better reputation systems. The ones that do not will get farmed.

The honest thing to say is: this ERC reveals that address-based reputation was always a convenient fiction. The fiction worked when keys were non-transferable (or at least inconvenient to transfer). This ERC makes transfer frictionless, and the fiction breaks. That is not the ERC's fault. It is the ERC's gift: forcing the ecosystem to build real reputation systems instead of relying on the accident that keys are hard to share.

---

## VIII. Summary Assessment

This is a seriously considered, technically rigorous, and thematically coherent standard. The core insight -- separation of control from custody via a transferable NFT -- is sound, and the authors have followed its implications with unusual discipline. The sell-and-drain mitigation, the execution freeze, the control-version invalidation, the cycle detection, the approval-revocation-during-freeze interface -- each of these exists because the authors asked "what could go wrong?" and built the answer into the mechanism rather than documenting it as a warning.

The main weaknesses are: the surviving-approvals problem (inherent to Ethereum, mitigated but not solved), the EIP-7702 blind spot in the control-version model (honest, needs a standardized response), and the absence of standardized emergency freeze and cross-chain control (both acknowledged as out of scope but limiting).

The main opportunity the authors may be underweighting is the hierarchical nesting. The single-account transfer case is compelling. The hierarchical case is transformative. It creates organizational structure as a protocol primitive. The ERC should invest more specification energy in making the hierarchical case first-class: standardize Enumerable, standardize account metadata, standardize cross-account coordination patterns.

**One sentence:** This ERC says that a wallet is not a key -- it is a deed -- and then builds the entire conveyancing system to make that metaphor real.

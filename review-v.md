# Review: ERC-XXXX "Wallet Title Deeds"

**Reviewer perspective:** Vitalik Buterin design philosophy -- mechanism design, credible neutrality, long-term sustainability, user sovereignty, resistance to capture.

---

## 1. What I Like About This ERC

### The core abstraction is genuinely novel and well-chosen

The fundamental insight here is that account control is a transferable property right, and property rights have a natural representation: title deeds. Rather than trying to bolt transferability onto smart-account signer storage (which is implementation-specific, hard to standardize, and has no established UX), this ERC encodes control as an ERC-721 token and inherits the entire existing transfer, approval, marketplace, and wallet infrastructure that already exists for NFTs. This is good mechanism design. You are not inventing a new interaction pattern; you are mapping a new capability onto an interaction pattern that millions of users, hundreds of wallets, and dozens of marketplaces already understand.

The separation of custody from control is the key intellectual contribution. Most account abstraction proposals conflate these two things, which is fine when you never want to transfer control, but terrible when you do. By making the account address the custody object and the NFT the control object, you get something that neither EOAs nor conventional smart wallets provide: the ability to rotate root control without moving a single asset, without sharing a single key, and without updating a single protocol integration that references the account address. This is a genuinely useful primitive.

### The transfer lock mechanism is well-designed mechanism design

The sell-and-drain problem is real, and the solution here is elegant. The mutual exclusivity between "execution is permitted" and "transfer approvals are valid" is the right structural invariant. The fact that canceling an unlock (via `lock()`) invalidates all transfer approvals granted during the unlock window is the correct game-theoretic response: a seller who drains and then tries to re-list must wait the full delay again, giving buyers a visible observation window.

The asymmetric meta-timelock on `setUnlockDelay` decreases is particularly thoughtful. Without it, the unlock delay is a lie: you show buyers a 24-hour delay, then collapse it to 1 second right before the sale. The meta-timelock makes the delay a credible commitment -- it has held for at least as long as its current value says, because reducing it takes at least that long. This is the kind of detail that separates serious mechanism design from hand-waving.

### FOCIL and VOPS alignment is strategically correct

This is forward-looking in the right way. The direct-owner execution path is an ordinary transaction, which means it works with inclusion lists, local mempool validation, and the statelessness roadmap without requiring any new mempool-side validation model. This is a significant practical advantage over EIP-8141's frame-transaction approach for the baseline case. The ERC is honest about the limitation: ERC-4337 UserOperations do not get this property. That honesty is good.

### Hierarchical account nesting models real organizational structures

The ability to have parent accounts control child accounts through NFT ownership is a natural fit for organizational structures. Corporate treasury controlling departmental accounts, fund managers controlling strategy-specific accounts, DAOs controlling operational subsidiaries -- these are real use cases, and the model here is clean. Each child account is a separate custody address with separate approvals, so approval-scoped risk isolation falls out naturally without requiring complex permission systems.

The cycle detection via bounded chain walk is the right approach. It is gas-bounded, predictable, and handles the degenerate cases (self-ownership, cycles) correctly.

### Post-quantum migration path is a genuine advantage

The ability to migrate to a post-quantum signature scheme by simply transferring the controlling NFT to a new owner account that uses that scheme is a real and underappreciated benefit. Most account abstraction proposals would require either upgrading the account contract or migrating assets. Here, you transfer the NFT to a PQ-scheme owner account and you are done. Every position, every protocol integration, every address-based relationship survives intact. This is the kind of long-term thinking that matters for infrastructure that is supposed to last decades.

### Approval revocation during the transfer freeze is a good addition

The tension between "execution must be frozen during transfer preparation" and "the seller should be able to clean up approvals before handoff" is a real UX problem. The solution of allowing hardcoded zero-approval revocations during the freeze -- which can only remove drain vectors, never create them -- is correct. It makes the marketplace flow practical without reopening the attack surface.

### The ERC-6551 differentiation is clear and honest

The comparison table and the explanation of why this is not just a constrained ERC-6551 profile is well-done. The canonical one-token/one-account mapping, the standardized transfer-aware control semantics, the lack of a central registry -- these are genuine architectural differences, not marketing distinctions.

---

## 2. Issues and Concerns

### The surviving-approvals problem is more severe than the ERC acknowledges

Token-level approvals (`approve`, `setApprovalForAll`, operator approvals) survive control transfer. The ERC acknowledges this and provides a revocation interface, but the fundamental problem is that there is no on-chain way to enumerate all outstanding approvals. The new controller must rely on off-chain indexing (block explorers, event logs) to discover what they are inheriting. This is fine for sophisticated users and institutional transfers with due diligence processes. It is a trap for retail users buying accounts on an NFT marketplace.

The revocation interface helps, but only if you know what to revoke. A buyer who does not check -- or whose wallet does not surface this information -- can inherit unlimited approvals to contracts they have never heard of. The ERC should be more prescriptive here. At minimum, it should RECOMMEND that marketplaces display all known outstanding approvals before a purchase completes. Better yet, consider a SHOULD-level recommendation that wallets automatically present a "revoke all known approvals" batch on first use after control transfer, sourced from off-chain event indexing.

The deeper architectural question is whether the ERC should define a way for the account to emit events when it grants approvals through `execute` or `executeBatch`, so that approval state is at least partially discoverable without external indexing. This would add gas cost to every approval operation, but it would make the security-critical information more accessible. I understand why you did not do this -- it requires parsing calldata on-chain, which is fragile and expensive -- but the trade-off should be discussed explicitly in the rationale.

### The unlock delay default of 1 hour may be too short for high-value accounts

One hour is a reasonable default for typical usage, but for high-value accounts (large DeFi positions, corporate treasuries, DAO vaults), it is arguably too short. A sophisticated attacker who gains temporary access to the controlling key can propose unlock, wait one hour, complete unlock, and transfer the NFT -- all within a window that many monitoring systems might not catch in time, especially across time zones or during weekends.

The ERC correctly allows increasing the delay, but the default sets the floor for accounts that never customize it. Consider whether the ERC should RECOMMEND that implementations prompt users to set a longer delay for accounts above a certain value threshold, or whether the default should be higher (24 hours?). The meta-timelock on decreases means a higher default is not onerous -- users who want shorter delays can reduce it, but the reduction is observable for the duration of the current delay.

### EIP-7702 delegation creates a serious gap in the control-version model

The ERC acknowledges this in Security Considerations, but I want to elevate it: this is not a minor edge case; it is a fundamental gap. The control version increments on NFT transfer and on `resetDelegations`, but it does NOT increment when the owner's 7702 delegation changes. This means:

- Malicious delegated code can install validators that persist after delegation is revoked.
- The controlled account has no way to detect that its owner's execution environment changed.
- The user must manually call `resetDelegations` after revoking a bad delegation, which they are unlikely to know.

The ERC's suggestion of detecting 7702 delegation via `EXTCODESIZE == 23` is fragile and may not survive future protocol changes. The more robust approach would be to define a mechanism by which the account can verify the owner's delegation state at execution time, or to recommend that high-security implementations pin the owner's `EXTCODEHASH` and reject execution when it changes.

I would go further: the ERC should RECOMMEND that implementations treat 7702-delegated EOAs as a higher-risk owner type and surface this in wallet UIs. The phrase "SHOULD be treated as a mutable contract owner rather than a plain key-held EOA" is correct but should be more prominent.

### The "no burn" rule creates a permanent liability

Controller tokens MUST NOT be burnable. This means every deployed account exists forever with no way to decommission it. If the account is drained of all value and the owner wants to stop thinking about it, they cannot. The NFT persists in their wallet permanently. Over time, users will accumulate dead controller tokens that clutter their interfaces and create cognitive overhead.

More importantly, the no-burn rule means there is no clean way to handle accounts that are in an unrecoverable fault state (e.g., the account contract was somehow bricked through an upgrade gone wrong, if upgradeability is implemented). The controller token still exists, still shows up in enumerations, and still cannot be disposed of.

Consider allowing burn under narrow, well-defined conditions -- for example, only when the account holds zero ETH and zero known token balances, or only after a long delay period with the account empty. Alternatively, define a "deactivated" state that makes the token invisible to Enumerable queries without actually burning it.

### Cross-chain story is incomplete

The ERC explicitly scopes cross-chain control as out-of-scope, and user-salt mode only stabilizes the deployment address. This is honest but leaves a significant gap. In practice, users will deploy accounts on multiple chains and expect unified control. The current design means:

- The same account address on two chains may be controlled by different NFTs on different controller token contracts.
- There is no standard way to synchronize control across chains.
- A cross-chain transfer (e.g., bridging the controller NFT) is not defined and could create split-brain ownership.

The ERC does not need to solve this, but it should explicitly discuss the failure modes and RECOMMEND against naive cross-chain bridging of controller tokens. It should also discuss how the design interacts with L2s where the account address might be the same but the controller token contract is a different deployment.

### The tokenId-to-address mapping creates a discoverability concern

Because `tokenId == uint256(uint160(account))`, anyone who knows an account address can compute the tokenId and look up the controller. This is presented as a feature (no registry needed), but it also means there is no privacy between "I know this account exists" and "I know who controls it." For users who want to control accounts pseudonymously -- using the account as their public identity while keeping the controlling address private -- this mapping leaks information.

The ERC's Privacy Considerations section acknowledges this, but I think it understates the practical impact. In a world where this standard is widely adopted, knowing someone's account address immediately reveals their controller address, which may be their personal EOA or their organizational multisig. This is a meaningful de-anonymization vector.

### The execution-active transfer guard relies on transient storage

The RECOMMENDED implementation uses `TSTORE`/`TLOAD` for the execution-active flag. Transient storage was introduced in EIP-1153 (Cancun) and is not available on all EVM-compatible chains. The ERC should clarify the minimum EVM version required and discuss fallback approaches for chains that do not support transient storage. The alternative of using regular storage is more expensive but works everywhere.

### Validator interface is underspecified

The validator interface defines `onInstall`, `onUninstall`, and `isValidSignatureForAccount`, but does not specify:

- How the account selects which validator to use when multiple are installed.
- Whether validators can be ordered or prioritized.
- What happens if two validators disagree (one returns valid, one returns invalid).
- Whether a validator can be installed with restrictions (e.g., "this validator can only validate signatures for specific function calls").

This is intentional minimalism, but it means that every implementation will make different choices, and validators will not be portable across implementations. Consider whether a minimal validator-selection specification would improve interoperability.

### The ERC is very long and complex

This is not a criticism of the content quality -- the content is thorough and well-reasoned. But the ERC is extremely long for a standard. Implementation complexity correlates with bug surface. The specification section alone includes: token model, transfer approval versioning, execution-active guards, transfer locks, unlock delays, meta-timelocked delay changes, cycle detection, deployment semantics, two salt modes, execution interface, approval revocation, nonce requirements, ERC-4337 compatibility, ERC-1271 with ERC-7739, validators, recovery, privacy-pool considerations, metadata, and events.

Consider whether some of these could be factored into companion ERCs. The metadata extension is explicitly OPTIONAL and could be a separate ERC. The validator mechanism could be a separate ERC. The privacy-pool withdrawal flow section could be removed entirely (it mainly says "this ERC does not standardize privacy-pool details," which could be a one-line note). Shorter ERCs are easier to review, easier to implement correctly, and easier to adopt incrementally.

---

## 3. Most Compelling Usage Scenarios

### Digital inheritance and estate planning

This is the single most compelling use case in my view. Today, digital inheritance on Ethereum is terrible. Users write seed phrases on paper and put them in safe deposit boxes, which is insecure, fragile, and requires the heir to understand wallet recovery. Under this ERC, the controller NFT can be held by a dead man's switch contract, a multisig with designated heirs, or a time-locked release mechanism. The entire on-chain estate -- every token, every DeFi position, every ENS name, every governance stake -- transfers atomically when the trigger fires. No seed phrases, no custodians, no asset-by-asset migration.

### Organizational treasury management

A DAO or company controls a parent account. Departmental budgets are child accounts. The corporate treasury deploys a child account for each department, funds it with a budget, and the department head operates it through a validator or a delegated key. If a department head leaves, the organization revokes the validator. If a department is shut down, the parent sweeps the child's assets. If the entire treasury management committee changes, one NFT transfer rotates control of the entire hierarchy.

### Account marketplace and reputation transfer

The ability to sell an account with its full history, positions, and address-based reputation intact enables a new kind of marketplace. A DeFi power user who has built up a strong liquidation-free history, protocol allowlist positions, and governance participation can sell that account as a going concern. The buyer gets not just the assets but the address identity. This is novel and valuable.

### Key rotation and security upgrade without disruption

Moving from a hot wallet to a hardware wallet multisig, or from secp256k1 to a post-quantum scheme, currently requires migrating every asset, updating every protocol integration, and losing address-based history. Under this ERC, it is a single NFT transfer. This is transformative for security hygiene -- it makes the safe thing the easy thing, which is the hallmark of good security design.

### Vesting and escrow without custodial risk

A vesting contract holds the controller NFT and releases it on schedule completion. The vested assets remain in the account and are never moved to the vesting contract itself. The beneficiary can even sell the vesting position (the right to eventually control the account) by transferring the vesting contract's claim, without the grantor releasing tokens early. This is a cleaner vesting model than anything that exists today.

---

## 4. Additional Possibilities the Authors May Not Be Thinking Of

### Programmable account policies via controller-contract intermediaries

Because the controller NFT can be held by any contract, you can create programmable policy layers by placing a "policy contract" between the human user and the controller NFT. For example:

- **Spending limits**: A contract holds the controller NFT and only forwards execution calls that stay within daily/weekly spending limits.
- **Whitelisted interactions**: A contract that only allows execution against a curated set of contract addresses.
- **Time-of-day restrictions**: A contract that only allows execution during business hours (useful for organizational accounts).
- **Multi-party approval for high-value transactions**: A contract that requires 2-of-3 approval for transactions above a threshold, but single-signer for small ones.

These policy contracts can be stacked and composed. The user's EOA controls the policy contract, the policy contract holds the controller NFT, and the policy contract enforces whatever rules it encodes. This is a form of programmable authorization that emerges naturally from the design without any changes to the ERC.

### Agent-operated accounts with hard-bounded risk

AI agents and automated systems need accounts to operate on-chain. The current approach is to give the agent a private key, which means a compromised agent can drain everything. Under this ERC, you deploy a child account, fund it with exactly the assets the agent needs, and give the agent a delegated validator on the child. The agent can operate freely within the child account's balance, but it cannot access the parent account or any sibling accounts. The blast radius is hard-bounded by the child's balance, not by a permission policy that might have bugs.

This is particularly compelling for the emerging agent economy. You could have a portfolio of agent-operated child accounts, each with a specific task and budget, all controlled by a single parent account that the human manages directly.

### On-chain credit and collateralized lending against accounts

Because the controller NFT is a standard ERC-721 token, it can be used as collateral in lending protocols. A user can borrow against their entire account -- all assets, all positions, all reputation -- by posting the controller NFT as collateral. If they default, the lender seizes control of the account.

This creates a new form of credit that does not require individual asset-by-asset collateralization. The account itself is the collateral unit. Liquidation is a single NFT transfer, not a complex multi-asset liquidation.

The unlock delay mechanism actually helps here: the lender can monitor the unlock state and trigger liquidation if the borrower proposes an unlock (which would signal an attempt to transfer the NFT out of the lending protocol's custody).

### Cross-protocol identity persistence

Many protocols grant benefits or access based on address history: governance weight from long-term holding, allowlists based on past participation, airdrop eligibility from historical usage, Sybil-resistance scores from on-chain reputation. All of these survive a control rotation under this ERC. This means "on-chain identity" becomes a durable, transferable, and improvable asset rather than something tied to a specific key pair.

### Regulatory-compliant account structures

The hierarchical nesting model maps naturally onto regulated entity structures. A regulated fund could have a parent account (controlled by the fund's governance contract) with child accounts for different asset classes, each with validators that enforce compliance rules (e.g., no transactions with sanctioned addresses, no unregistered securities). The compliance layer is the controller-contract intermediary, not a modification to the account itself.

### Dead man's switch with graceful degradation

Rather than a binary dead-man's switch, you can create a graduated system: after 30 days of inactivity, a designated family member gains validator access (can execute limited operations). After 90 days, the controller NFT transfers to the heir. After 180 days, an emergency multisig of trustees gains control. Each tier is a different contract in the ownership chain, and the system degrades gracefully rather than failing catastrophically.

### Account-as-a-service

A service provider could deploy pre-configured accounts with specific validator setups, protocol integrations, and child-account structures, then transfer the controller NFT to the client. The client receives a ready-to-use account with all the infrastructure already in place. This is the "preconfigured account packages" flow mentioned in the deployment section, but the business model implications are broader: it enables a service economy around account setup and management.

---

## 5. Suggested Changes to Increase Optionality

### Add an optional "account freeze" mechanism independent of the transfer lock

Currently, the only way to freeze execution is to propose an unlock, which is specifically designed for the transfer flow. There is no way for an owner to freeze their own account as a panic button without entering the transfer preparation flow. Consider adding a `freezeAccount(tokenId)` that prevents all execution without starting the transfer clock. This would be useful for:

- A user who suspects compromise and wants to stop all activity immediately while they investigate.
- An organizational account that needs an emergency halt without signaling that a transfer is coming.
- Integration with external security monitoring services that detect suspicious activity.

The freeze should be instantly reversible by the current owner.

### Define a standard "transfer preparation report" view function

Add a view function like `transferReport(tokenId)` that returns structured data about the account's state relevant to a prospective buyer: the number of known active approvals (if any accounting is maintained), the number of installed validators, the current unlock delay, any pending delay changes, and the control version. This gives marketplaces and wallets a standard way to surface pre-transfer due diligence information without requiring off-chain indexing for every data point.

### Allow optional "post-transfer hooks" on the account

When the controller NFT transfers, the account currently has no way to react. Consider defining an optional callback that the controller token invokes on the account after a successful transfer. The account could use this to:

- Auto-revoke a predefined list of approvals.
- Reset internal state that should not persist across controllers.
- Emit events that help the new controller discover the account's configuration.

This should be OPTIONAL and gas-bounded to avoid griefing, but it would significantly improve the transfer UX for accounts that implement it.

### Define a minimal "account capability discovery" interface

Add a view function like `supportsFeature(bytes4 featureId)` or extend ERC-165 with feature-specific interface IDs for optional capabilities: validator support, ERC-4337 support, ERC-8211 composable batching, metadata, recovery. This helps wallets and tooling discover what a specific account implementation supports without trial and error.

### Consider a "read-only mode" for decommissioned accounts

Rather than the absolute no-burn rule, define an optional "read-only" or "decommissioned" state where the controller token still exists but the account can only receive assets and execute revocations, not make arbitrary calls. This provides a clean off-ramp for accounts the owner no longer wants to actively manage while preserving the invariant that every controller token corresponds to a valid account.

### Allow controller token implementations to define custom transfer conditions

The current specification defines the unlock delay as the only transfer gate. Consider allowing implementations to add additional transfer conditions via an extensible hook. For example:

- A compliance-aware implementation might require that the recipient is on an allowlist.
- An organizational implementation might require multi-party approval for transfer.
- A regulated implementation might require KYC verification of the recipient.

The hook should be OPTIONAL and clearly defined as an extension point, not a required feature. The base specification's unlock delay should remain the minimum required behavior.

### Standardize a minimal "account migration" flow

When a user wants to move from one ERC-XXXX implementation to another (e.g., to a newer version with additional features), there is currently no standard path. The user would need to deploy a new account, transfer all assets individually, and then decommission the old one. Consider defining a standard `migrateFrom(address oldAccount)` flow that allows the new account to atomically pull assets from an old account with the old owner's authorization. This is complex but would significantly improve long-term upgradeability.

### Add an event for validator-selection policy changes

If multiple validators are supported (which the interface implies), add an event or view function that surfaces the current validator selection policy. This helps external tooling understand which validators are active and how they interact, without requiring implementation-specific knowledge.

### Consider separating the "operational" and "emergency" control paths more explicitly

Currently, the owner of the controller NFT has full authority: they can execute, install validators, configure unlock delays, propose transfers, and initiate recovery. For high-value accounts, it may be valuable to separate these into "operational" (day-to-day execution) and "administrative" (validator management, delay changes, transfer preparation) paths, with different authorization requirements for each. This does not need to be in the base ERC, but the ERC should be designed to allow this separation in extensions without conflicting with the base specification.

---

## Summary Assessment

This is a strong, well-thought-out ERC that addresses a genuine gap in the Ethereum account model. The core abstraction -- control as a transferable title deed -- is sound, the mechanism design around transfer locks is sophisticated, and the alignment with the Ethereum roadmap (FOCIL, VOPS, post-quantum migration) is forward-looking in the right ways.

The main concerns are: (1) the surviving-approvals problem needs more aggressive mitigation at the standard level, not just documentation; (2) the EIP-7702 gap in the control-version model is a real security concern that should be addressed more substantively; (3) the ERC is very long and would benefit from factoring optional components into companion standards; and (4) the cross-chain story needs at least explicit failure-mode documentation even if the solution is out of scope.

The most exciting aspects are not the NFT mechanism itself but the possibilities it enables: digital inheritance, agent-bounded accounts, credit against account value, programmable policy intermediaries, and the general principle that making security upgrades frictionless (one NFT transfer to rotate to a post-quantum scheme) is how you actually get people to adopt better security practices.

The honest question I always ask: does the NFT serve the user, or does the NFT serve a narrative? In this case, the NFT is doing real work. It is not decoration; it is the control mechanism. The ERC-721 encoding is not speculative infrastructure -- it is the reason that transfer, custody, marketplace integration, and composability work without building anything new. This is blockchain serving users, not users serving blockchain.

I would encourage the authors to pursue this ERC seriously, address the concerns above, and consider the companion-ERC factoring to make adoption more incremental. This is the kind of infrastructure-level standard that, if done right, could meaningfully improve how Ethereum accounts work for the next decade.

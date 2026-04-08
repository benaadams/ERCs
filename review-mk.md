# Review: ERC-XXXX -- Wallet Title Deeds

**Reviewer:** Michal Kovarik (kovaex) perspective -- systems design, optimization, scalability, and emergent-complexity analysis

**Date:** 2026-04-08

**Status of ERC:** Draft

---

## 1. What I Like About It

### 1.1 The Core Abstraction Is Genuinely Elegant

The single best thing about this proposal is the mapping: `tokenId == uint256(uint160(account))`. No registry. No lookup table. No oracle. Pure arithmetic. This is the kind of design decision that looks obvious in retrospect but requires real taste to arrive at. It means anyone can derive the relationship in either direction without touching the chain. No state dependency for discovery. No central point of failure. The mapping is as permanent and trustless as the address space itself.

Compare this to ERC-6551, which requires a registry contract and allows multiple accounts per NFT. That flexibility sounds good on paper, but it creates ambiguity: which account is THE account? Here, the answer is always computable. One token, one account, one address, one derivation. This is the right tradeoff for a control standard.

### 1.2 Separation of Control and Custody Is the Right Primitive

The ERC correctly identifies that the fundamental problem with EOAs is the conflation of identity (address), custody (what holds the assets), and control (who can move them). By making the NFT the control object and the contract the custody object, you get:

- Key rotation without asset migration
- Account transfer without key sharing
- Organizational handoff without multi-step governance votes
- Security model changes (EOA to multisig to post-quantum) without address changes

This is a real problem that currently costs real gas and real operational complexity. The proposal solves it cleanly.

### 1.3 The Transfer Lock and Sell-and-Drain Mitigation Are Well-Thought-Out

The mutual exclusion between execution and transferability is the critical security property that makes the whole system viable for marketplace use. The authors clearly thought through the attack scenario carefully:

- `proposeUnlock` freezes execution immediately
- The delay window gives counterparties time to observe account state
- `lock` cancels the unlock AND invalidates all transfer approvals granted during the unlock window
- Execution and transfer are never simultaneously possible

The asymmetric meta-timelock on `unlockDelay` changes (increases immediate, decreases delayed by the current delay) is particularly well-designed. Without it, a seller could accumulate trust with a long delay, then collapse it to 1 second right before a drain-and-sell. The meta-timelock ensures the current delay has been in effect for at least that many seconds. This is the kind of defense-in-depth that shows genuine adversarial thinking.

### 1.4 The Approval Revocation Interface During Freeze Is Pragmatic

Allowing hardcoded zero-value approval revocations during the unlock freeze is a smart concession. It recognizes that the freeze creates a tension: the seller SHOULD clean up approvals before transfer, but the freeze exists to prevent the seller from draining. By restricting the interface to exact zero-value calls with hardcoded calldata, the revocation path cannot be abused as a general execution backdoor. This is good engineering: identifying where two security requirements conflict and finding the narrow safe path through.

### 1.5 Hierarchical Account Trees Are a Natural Consequence

The nesting model -- where a controlled account can itself hold controller NFTs -- creates organizational structures as an emergent property of the base primitive. Parent-child relationships, departmental budgets, fund structures, subsidiary management -- these all fall out of the composition rules without any special-cased hierarchy logic. The cycle detection via bounded ownership-chain walk (default 4 hops) is the right constraint: it caps complexity predictably and the gas cost is bounded and borne by the rare transfer operation, not by the frequent execution path.

### 1.6 No Central Registry, No Singleton Dependency

Multiple independent implementations can coexist. The standard defines interfaces and invariants, not a deployed contract address. This is architecturally sound. No single contract becomes a systemic dependency. The ecosystem can evaluate implementations based on audit quality and feature set, exactly as it does with ERC-20 implementations.

### 1.7 The FOCIL/VOPS Compatibility Is Strategically Important

Keeping the direct-owner execution path as an ordinary transaction -- rather than inventing a new transaction type or requiring mempool-side EVM validation -- is a forward-looking design choice. As Ethereum moves toward statelessness and inclusion lists, proposals that require novel mempool validation logic face adoption headwinds. This ERC sidesteps that entirely for its primary execution path. The authors are correct that this is a meaningful advantage over EIP-8141's frame-transaction model for the baseline case.

### 1.8 The ERC-7739 Integration Shows Standards Literacy

Using ERC-7739 defensive rehashing with an ERC-5267-discoverable domain that binds the controller token, tokenId, and control version into the salt is thorough. It prevents cross-account signature replay, cross-version replay, and cross-chain replay in a single domain construction. The authors clearly understood the signature-validation problem space.

---

## 2. Issues and Concerns

### 2.1 The Surviving Approvals Problem Is Undersolved

This is the single largest practical issue with the proposal. Token-level approvals (ERC-20 `approve`, ERC-721 `setApprovalForAll`, ERC-1155 operator approvals) survive controller transfer. The ERC acknowledges this repeatedly and provides a revocation interface, but the fundamental problem remains: there is no on-chain way to enumerate what approvals the account has granted.

The buyer of a controlled account has no mechanism to discover all outstanding approvals except by scraping historical event logs off-chain. The revocation interface is only useful if you know what to revoke. For a marketplace sale of an account with years of DeFi history, the approval surface could be enormous and non-obvious.

**Concrete risk scenario:** Alice uses account A for 3 years across dozens of DeFi protocols, granting approvals to routers, vaults, lending pools, and aggregators. She sells the controller NFT to Bob. Bob revokes the 10 approvals he knows about from a block explorer query. But one obscure approval to a protocol that was later compromised survives. That protocol's attacker drains Bob's account through the surviving approval.

The ERC should at minimum RECOMMEND that compliant accounts maintain an on-chain approval registry (even if approximate), or RECOMMEND that marketplace integrations require sellers to execute a batch revocation of ALL known approvals as part of the sale flow. Better yet, the standard could define an optional `revokeAllKnownApprovals` pattern that works with common token standards.

### 2.2 The EIP-7702 Delegation Gap Is a Real Vulnerability

The ERC honestly documents this, but "honestly documenting a vulnerability" is not the same as solving it. The core problem: changing the 7702 delegation on the owner EOA does NOT increment `controlVersionOf`. This means:

- Malicious delegated code can install validators that persist after delegation is revoked
- Malicious delegated code can initiate a pending `unlockDelay` reduction
- Malicious delegated code can grant token approvals from the controlled account
- None of these are reversed when the delegation is revoked

The mitigation suggestions (detect `EXTCODESIZE == 23`, pin `EXTCODEHASH`) are operational workarounds, not protocol-level solutions. Since EIP-7702 is live and adoption is growing, this gap will be actively exploited.

**Suggestion:** The ERC should either mandate that `execute` and `executeBatch` check for delegation changes since the last control-version snapshot, or define an explicit `acknowledgeOwnerChange` function that the account owner must call after any 7702 delegation change to trigger a control-version increment. The current approach of "users should call `resetDelegations` manually" will be forgotten by the majority of users.

### 2.3 The Unlock Delay Minimum of 1 Second Is Too Low

While zero is correctly forbidden, a 1-second minimum is effectively zero for any practical counterparty monitoring. No marketplace, indexer, or human can react within 1 second. The meta-timelock means reducing TO 1 second takes time, but once you are at 1 second, every subsequent unlock is effectively instant.

The concern is not about the meta-timelock working correctly -- it does. The concern is about the steady-state: an account that has been at `unlockDelay = 1` for months provides no meaningful protection to buyers. Marketplaces will need to check the current delay and warn users, but the standard itself should probably set a higher floor (60 seconds? 300 seconds?) or at least RECOMMEND that marketplaces reject listings with delays below a practical threshold.

### 2.4 Gas Cost of Nested Execution Is Unanalyzed

The ERC permits hierarchical account trees up to `maxNestingDepth` levels. But executing through a hierarchy means the parent calls the child, which calls the grandchild. Each level adds:

- One cross-contract CALL
- One `ownerOf` check on the controller token
- One `locked` check
- One `unlockReadyAt` check
- TSTORE/TLOAD for the execution-active flag

At 4 levels of nesting, a leaf-level `execute` requires 4 cross-contract calls with authorization checks at each level. This is potentially expensive and the gas cost is borne by the user on every operation, not just on rare transfers.

The ERC should include a gas analysis for nested execution at various depths, and should consider whether the account interface should support a `executeOnBehalf` pattern that flattens the call chain for common cases.

### 2.5 The `isExecutionActive` Check Creates a Cross-Contract Dependency on Transfer

The controller token must STATICCALL `isExecutionActive()` on the account during every transfer. This means:

- If the account contract is somehow bricked (bad upgrade, storage corruption), the controller NFT becomes permanently non-transferable
- The transfer gas cost depends on the account implementation, not just the token contract
- A malicious account implementation could grief transfers by consuming gas in `isExecutionActive`

The ERC should specify a gas stipend for the `isExecutionActive` call (similar to the 30,000 gas stipend for `ownerOf` in cycle detection), and should define what happens if the call reverts or exceeds the stipend.

### 2.6 No Standard for Account Discovery Beyond Enumerable

The ERC RECOMMENDS ERC-721 Enumerable but does not REQUIRE it. Without Enumerable, a wallet cannot discover which accounts a user controls without off-chain indexing. For a standard that aims to be the foundation of account management, on-chain discoverability should be mandatory, not recommended. The gas cost of maintaining Enumerable indices is paid at mint and transfer time, which are rare operations under this ERC's design. The tradeoff favors requiring it.

### 2.7 Cross-Chain Story Is Explicitly Out of Scope but Practically Critical

The ERC correctly notes that user-salt stabilizes deployment addresses but does not synchronize ownership. In practice, users will deploy the same account address on multiple chains and expect unified control. If the controller NFT is transferred on chain A but not on chain B, the account on chain B is still controlled by the old owner. This is a fundamental footgun that the ERC waves away as "out of scope."

At minimum, the ERC should include a Security Considerations subsection that explicitly warns about cross-chain control desynchronization and outlines the properties that a future cross-chain extension would need to provide.

### 2.8 Recovery Is Too Loosely Specified

Recovery is marked OPTIONAL with a list of SHOULDs. Given that NFT theft equals account takeover, recovery is not really optional for any serious deployment. The ERC should either:

1. Define a minimal mandatory recovery interface (even if implementations can choose their recovery policy), or
2. Much more strongly emphasize that deploying without recovery configured is equivalent to deploying a smart wallet with no backup key, and specify the exact properties a compliant recovery extension MUST satisfy.

### 2.9 The `setApprovalForAll` Ban Creates Marketplace Friction

Forbidding `setApprovalForAll` on the controller token is the correct security decision, but it means existing NFT marketplace flows (which universally rely on `setApprovalForAll` for the collection) will not work. Users must explicitly `approve` each individual token for each sale. The ERC acknowledges the security rationale but does not discuss the UX impact. Marketplaces will need custom integration flows for controller tokens. This is worth calling out as an adoption friction point.

### 2.10 Pending Unlock Delay Decrease Surviving Transfer Is Surprising

A pending `setUnlockDelay` decrease survives a successful transfer. This means a seller can schedule a delay reduction, then transfer the token, and the buyer inherits a ticking timebomb that will weaken their transfer protection. The buyer CAN cancel it by calling `setUnlockDelay` with a value >= current, but they need to KNOW to check for it.

The ERC should either:
- Clear pending delay decreases on transfer, or
- Emit a prominent event on transfer when a pending decrease exists, or
- REQUIRE wallets to display pending decreases on receipt

The current behavior is defensible (the new owner can always cancel) but surprising, and surprising security semantics are dangerous.

---

## 3. Usage Scenarios and Possibilities I Like

### 3.1 DAO Treasury Rotation

The DAO treasury use case is genuinely compelling. Currently, changing a DAO's treasury management committee requires:
1. Governance vote to approve new committee
2. Multi-signature ceremony to add new signers
3. Multi-signature ceremony to remove old signers
4. Potential asset migration if the multisig implementation requires it
5. Updating every protocol integration that references the treasury address

Under this ERC: transfer the controller NFT. One operation. The address, positions, integrations, and reputation all persist. This alone justifies the standard's existence.

### 3.2 Approval-Scoped Risk Isolation

The ability to hold high-value assets in one account and interact with risky protocols from another, both controlled by the same EOA, is a pattern that every sophisticated user currently achieves with multiple seed phrases. This ERC makes it trivially easy. The blast radius of a compromised approval is bounded by the account that granted it, not by the user's total holdings.

### 3.3 Digital Inheritance

The dead man's switch / designated-heir multisig pattern for the controller NFT is a clean solution to a real problem. Currently, digital inheritance requires either:
- Sharing seed phrases (security risk during life, operational complexity at death)
- Centralized custodians (counterparty risk)
- Complex smart contract vaults with bespoke logic

Under this ERC, a standard dead man's switch contract holds the NFT, and the entire on-chain estate transfers to the designated heir through a single NFT transfer. No shared secrets, no custodian, standard tooling.

### 3.4 Vesting Without Key Sharing

The vesting contract use case is subtle but powerful: a vesting contract holds the controller NFT, the account holds the actual assets. The vesting schedule controls when the beneficiary gains access. But crucially, the CLAIM to eventual control can be transferred (by transferring whatever token the vesting contract recognizes), while the underlying assets remain locked. This creates a secondary market for vested positions without the grantor releasing tokens early. This is genuinely novel and useful.

### 3.5 Account Sale as a First-Class Operation

Selling an entire DeFi portfolio -- including LP positions, lending collateral, governance voting power, ENS names, protocol allowlists, and reputation -- as a single atomic transfer is something that is currently impossible. Under this ERC, it is a standard NFT sale. The marketplace infrastructure already exists. The buyer receives a complete on-chain identity with full history.

---

## 4. Additional Possibilities the Authors May Not Be Thinking Of

### 4.1 Automated Account Lifecycle Management (The Factory Pattern)

The hierarchical account model combined with atomic deployment enables something the ERC does not explicitly discuss: automated account factories for ephemeral, purpose-specific accounts.

Consider a DeFi aggregator that, for each user strategy, deploys a child account, funds it with exactly the needed assets, executes the strategy, and sweeps profits back to the parent. The child account is a disposable execution environment with hard-bounded risk (only the funded amount is exposed). The parent never grants any approvals from its own address.

Take this further: a "strategy template" is a set of `initCalls` for `deployAccountConfigured`. Users share strategy templates the way Factorio players share blueprints. A template might encode: deploy child, approve USDC to Aave, supply USDC, borrow ETH against it, swap ETH for specific tokens. The entire strategy is a shareable, reproducible, inspectable configuration.

This is the blueprint system for DeFi. It naturally creates a library of community-audited account configurations. The authors mention "preconfigured account packages" in passing but may not have fully explored the implications of a template-sharing ecosystem.

### 4.2 Programmable Custody Schedules

The controller NFT can be held by any contract. This means custody can be programmatic in ways the ERC does not fully explore:

- **Time-locked accounts:** A contract holds the NFT and only transfers it after a date. The account exists, has assets, but nobody can execute until the time lock expires. Useful for regulatory holds, cool-down periods, or staged token launches.
- **Conditional custody:** A contract holds the NFT and transfers it only when an oracle condition is met (price threshold, governance vote, real-world event via Chainlink). The account is a conditional escrow with full smart-account capabilities.
- **Rotating custody:** A contract holds the NFT and rotates ownership on a schedule -- useful for shared infrastructure where different teams get operational control at different times.
- **Auction-determined custody:** The NFT goes to the highest bidder in a periodic auction. The account becomes a shared resource whose controller is market-determined. This is a natural model for MEV or blockspace markets.

### 4.3 Account-Level Credit and Collateralization

Because the controller NFT represents verifiable, transferable control over a specific on-chain balance sheet, it can serve as collateral in ways the ERC does not discuss:

- A lending protocol accepts a controller NFT as collateral. The NFT represents control over an account with 100 ETH. The lender can verify the account's balance on-chain. On default, the lender receives the NFT and thus control of the account's assets -- without requiring the borrower to transfer assets out of the account first. This is overcollateralized lending where the collateral object IS the balance sheet, not a proxy for it.
- A margin protocol holds the controller NFT while the user trades. The protocol can liquidate by transferring the NFT to itself, gaining control of the account and its positions. No need for individual position liquidation -- the entire account changes hands.
- Insurance products where the premium is paid from one account and the insured account's controller NFT is held in escrow for claims processing.

The locked-by-default and unlock delay would need careful interaction with these use cases, but the primitive is there.

### 4.4 Agent-Operated Accounts With Hard Spending Limits

The ERC discusses validators for delegated signing, but the combination of child accounts and validators enables a specific pattern that will become critical as AI agents proliferate on-chain:

1. User deploys a child account
2. User funds it with a specific budget (e.g., 0.5 ETH)
3. User installs a validator that authorizes a specific agent's signing key
4. The agent operates autonomously within the child account's balance
5. The blast radius is exactly 0.5 ETH, enforced by account boundary, not by permission policy

This is fundamentally more secure than session keys with spending limits because the limit is PHYSICAL (the account's balance) not LOGICAL (a policy that must be correctly enforced). A bug in the permission policy cannot drain more than the account holds. This maps directly to the game-design principle of emergent security: simple rules (account boundaries) creating robust outcomes (hard spending limits) without complex policy engines.

### 4.5 On-Chain Organizational Charts

Nested account hierarchies create something the ERC touches on but does not fully articulate: a legible, on-chain organizational structure. If a corporation deploys:

```
HoldCo (root account)
  +-- Treasury (child)
  +-- Operations (child)
  |     +-- Engineering (grandchild)
  |     +-- Marketing (grandchild)
  +-- Investments (child)
        +-- Fund A (grandchild)
        +-- Fund B (grandchild)
```

Then the ownership chain of controller NFTs IS the org chart. Transferring the Operations controller NFT to a new parent is a restructuring. Transferring HoldCo's controller NFT to a new owner is an acquisition. The corporate structure is not a metaphor layered on top of smart contracts -- it IS the smart contract ownership graph.

This has implications for regulatory compliance (auditors can verify the org structure on-chain), governance (voting power can follow the hierarchy), and reporting (aggregate balance sheets are computable by walking the tree).

### 4.6 Composable Account Templates as Protocol Primitives

Protocols could define expected account configurations as templates. For example:

- A lending protocol publishes an "approved borrower template": deploy child account, install the protocol's validator, approve the protocol's contracts
- An insurance protocol publishes a "covered account template": specific validators, specific approval revocation schedules, specific child-account structure
- A yield aggregator publishes a "strategy vault template": child account per strategy, specific rebalancing validators, specific risk parameters

These templates are inspectable, auditable, and shareable. They create a market for account configurations, not just for tokens. The composability of templates (apply template A AND template B to the same account) opens design space the ERC does not explore.

### 4.7 Post-Quantum Migration Path Without Ecosystem Coordination

The ERC mentions EIP-8202 compatibility for post-quantum signatures, but the deeper implication is this: migrating to post-quantum security does not require any protocol change, any token contract change, or any DeFi protocol change. The user:

1. Deploys a new owner account using a post-quantum signature scheme
2. Transfers the controller NFT to that new owner
3. Done

Every DeFi position, every approval, every ENS name, every governance delegation -- all of them are now controlled by a post-quantum key, and none of them needed to be aware of the migration. The address did not change. The positions did not move. Only the root control rotated. This is the least disruptive post-quantum migration path possible, and the ERC should emphasize it more strongly as a selling point.

### 4.8 Reputation Markets

Because account transfer preserves address-based history, reputation becomes a transferable asset. An account with 3 years of perfect lending history, consistent governance participation, and whitelisted access to restricted protocols has VALUE beyond its token balances. Under this ERC, that reputation can be priced and sold.

This creates a secondary market for on-chain identity that does not exist today. Whether this is desirable is debatable -- it enables Sybil attacks on reputation systems that assume addresses are non-transferable. But the possibility is real and the ERC should acknowledge it, perhaps in Security Considerations or as a note on reputation-system design.

### 4.9 Streaming Payment Accounts

Combined with streaming payment protocols (Sablier, Superfluid), child accounts become natural payment channels:

1. Parent deploys child account for each contractor/employee
2. Parent funds child account with streaming payment setup
3. Contractor's validator is installed on the child
4. Contractor can withdraw from their stream but not access the parent
5. Parent can reclaim by transferring the child's controller NFT back to itself

This is payroll infrastructure built from composable primitives rather than purpose-built payroll contracts.

---

## 5. Changes to Increase Optionality Further

### 5.1 Define an Optional Account Introspection Interface

Add an optional `IERCXXXXAccountIntrospection` interface:

```solidity
interface IERCXXXXAccountIntrospection {
    function knownApprovals() external view returns (ApprovalRecord[] memory);
    function childAccounts() external view returns (uint256[] memory tokenIds);
    function parentAccount() external view returns (uint256 tokenId);
    function accountAge() external view returns (uint256 deployTimestamp);
}
```

This enables on-chain discovery of account state without relying on off-chain indexing. It makes marketplace integrations possible without external dependencies and enables smart contracts (lending protocols, insurance products) to reason about account state. The approval tracking is approximate (the account cannot know about approvals it has forgotten), but maintaining a best-effort registry is far better than nothing.

### 5.2 Add an `executeOnBehalf` Pattern for Flattened Nested Execution

For hierarchical account trees, add an optional method that allows the root controller to execute directly on a descendant without traversing the full call chain:

```solidity
function executeOnBehalf(
    uint256[] calldata ancestorTokenIds,
    address target,
    uint256 value,
    bytes calldata data
) external payable returns (bytes memory);
```

The account verifies the caller is the root of the ancestor chain and executes directly. This saves gas proportional to the nesting depth for the common case where the root controller is operating on a leaf account.

### 5.3 Define a Standard Account Migration Interface

For accounts that need to upgrade their implementation, define a standard migration path:

```solidity
interface IERCXXXXMigration {
    function migrateToImplementation(address newImplementation, bytes calldata migrationData) external;
    function migrationDelay() external view returns (uint256);
    function proposeMigration(address newImplementation) external;
    function completeMigration() external;
}
```

Apply the same propose-delay-complete pattern used for unlock to implementation upgrades. This gives account holders a path to newer implementations without losing their address, while giving counterparties visibility into pending changes.

### 5.4 Add Configurable Execution Hooks

Allow accounts to install pre-execution and post-execution hooks:

```solidity
interface IERCXXXXExecutionHooks {
    function installPreExecutionHook(address hook) external;
    function installPostExecutionHook(address hook) external;
    function removePreExecutionHook(address hook) external;
    function removePostExecutionHook(address hook) external;
}
```

Hooks enable:
- Spending limits without modifying the core execution path
- Automated compliance checks (blocked addresses, restricted tokens)
- Audit logging to external contracts
- Rate limiting

These are currently achievable only through validators or external wrappers. First-class hook support would make the account more extensible without complicating the core authorization model.

### 5.5 Support Conditional Transfers

Extend the transfer lock to support condition-based transfers:

```solidity
function proposeConditionalUnlock(
    uint256 tokenId,
    address condition,
    bytes calldata conditionData
) external;
```

Where `condition` is a contract implementing:

```solidity
interface ITransferCondition {
    function isConditionMet(uint256 tokenId, address from, address to) external view returns (bool);
}
```

This enables:
- Transfers that complete only when a payment is confirmed (escrow)
- Transfers that complete only when an oracle reports a specific state
- Transfers that are restricted to a whitelist of recipients
- Transfers that require multi-party approval

Without this, every conditional transfer requires a separate wrapper contract. With it, the conditions are composable and inspectable.

### 5.6 Define a Standard Event for Account Balance Snapshots

```solidity
event AccountSnapshot(
    uint256 indexed tokenId,
    bytes32 indexed snapshotHash
);
```

Where `snapshotHash` is a commitment to the account's known state at a specific block. This enables off-chain verification of account state for marketplace listings without requiring on-chain enumeration of all assets. The account (or a designated viewer contract) computes the hash; verifiers can check it against the chain state.

### 5.7 Make `maxNestingDepth` Configurable Per Deployment

The current spec exposes `maxNestingDepth()` as a view but does not define whether it is configurable. Different deployments have different needs:

- A personal wallet may want depth 2 (parent + children)
- A corporate structure may want depth 6+
- A DeFi protocol may want depth 1 (no nesting)

Making this a constructor parameter or governance-adjustable value (with appropriate delay) would increase the standard's applicability without changing its semantics.

### 5.8 Add Batch Account Deployment

For organizational setup, deploying 10 child accounts requires 10 transactions. Add:

```solidity
function deployAccountBatch(
    address initialOwner,
    uint256 count
) external returns (uint256[] memory tokenIds, address[] memory accounts);
```

This reduces gas cost for organizational setup and makes the "deploy organizational chart" use case practical in a single transaction.

### 5.9 Consider an Account "Freeze" Distinct From Transfer Unlock

Currently, the only way to freeze execution is `proposeUnlock`, which is entangled with transfer preparation. There are scenarios where you want to freeze execution WITHOUT preparing for transfer:

- Emergency response to detected compromise
- Regulatory compliance (court order to freeze account)
- Self-imposed cooldown (user wants to prevent panic selling)

A separate `freezeExecution(tokenId, duration)` function would serve these cases without conflating them with transfer preparation. The freeze would block `execute` and `executeBatch` for the specified duration without affecting the lock state or transfer approval version.

### 5.10 Define an Event Standard for Cross-Implementation Interop

Different controller token implementations will coexist. For wallets and indexers that need to discover and display controller tokens from multiple implementations, define a standard interface that any controller token implementation MUST implement:

```solidity
interface IERCXXXXDiscovery {
    function isERCXXXXControllerToken() external pure returns (bool);
    function implementationVersion() external pure returns (string memory);
    function supportedFeatures() external pure returns (bytes32);
}
```

Where `supportedFeatures` is a bitmap of optional features (metadata, migration, execution hooks, conditional transfers, etc.). This enables wallets to discover and correctly interact with controller tokens from any compliant implementation.

---

## Summary Assessment

This is a strong proposal that identifies a real primitive missing from the Ethereum account model: transferable, composable control as a first-class object. The core abstraction (NFT as title deed, arithmetic address derivation, separation of control and custody) is elegant and well-grounded. The security model (transfer lock, sell-and-drain mitigation, control version, execution-active guard) demonstrates genuine adversarial thinking.

The main weaknesses are:

1. **The surviving approvals problem is underserved** -- the revocation interface is necessary but not sufficient without discovery.
2. **The EIP-7702 delegation gap** needs a protocol-level solution, not just documentation.
3. **Cross-chain control synchronization** is too important to be out of scope forever.
4. **Recovery is too loosely specified** for a standard where NFT loss means total account loss.

The emergent possibilities -- account templates as shareable blueprints, agent-operated bounded-risk accounts, on-chain organizational charts, post-quantum migration without ecosystem coordination, reputation markets -- suggest the design has the right kind of generativity. Simple rules creating complex, player-discovered patterns. That is the hallmark of a good systems design.

The factory must grow. The accounts must compose.

---

*Review written from the perspective of systems-level design analysis, focusing on composability, scalability, emergent complexity, and whether the design creates a vocabulary that users can combine in ways the designers did not anticipate.*

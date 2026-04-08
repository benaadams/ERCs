# Review: ERC-XXXX "Wallet Title Deeds"

**Reviewer:** James Niesewand  
**Date:** 2026-04-08  
**Status reviewed:** Draft (created 2026-04-02)

---

## 1. What I Like About It

### The core insight is correct and important

Separating control from custody is the right primitive. The entire history of on-chain account management has been plagued by the conflation of "the address that holds assets" with "the key that authorises operations." Every time someone wants to rotate keys, change security models, hand off a treasury, or die and leave their crypto to their children, the answer has been "move everything one by one and pray you don't miss something." This ERC makes control rotation a single atomic operation. That is genuinely novel at the standard level and genuinely useful.

### The title deed metaphor is well chosen

This is not an accident of naming. Real property title deeds work exactly this way: the deed conveys control over the property; the property stays where it is. The metaphor maps cleanly onto the technical mechanism, which makes it communicable to lawyers, regulators, and normal humans. That matters for adoption.

### The sell-and-drain protection is properly thought through

The mutual exclusion between "execution is permitted" and "marketplace approval is valid" is the most careful piece of mechanism design in the document. The state machine is:

- Locked + no proposal = execution allowed, transfer impossible
- Proposal pending = execution frozen, transfer impossible, approvals invalidated
- Unlocked = execution frozen, transfer possible

The key insight is freezing execution from `proposeUnlock()` onwards, not just at `completeUnlock()`. This closes the drain window that would otherwise exist during the unlock delay period. The transfer approval version increment on `proposeUnlock` and `lock` creates the mutual exclusion: any approval granted before the state transition is dead, and re-approving requires the owner to have been through the state transition. This is tight.

### The asymmetric meta-timelock on unlockDelay is load-bearing and correct

Increases take effect immediately (strictly safer), decreases are timelocked by the current delay. This means `unlockDelayOf(tokenId)` has held its current value for at least that many seconds at any given moment. Without this, the entire sell-and-drain protection collapses: an attacker could set a large delay to build trust, then collapse it to 1 second immediately before proposing unlock. The meta-timelock makes the delay a credible commitment.

### Forbidding setApprovalForAll on the controller token

This is the kind of decision that shows someone thought about adversarial use. A blanket operator approval on the controller token would be root access to every account the operator controls. Forbidding it and forcing per-token approvals (which are themselves versioned) is the right call.

### The cycle detection and nesting model

Hierarchical accounts with bounded depth and exhaustive cycle prevention is well specified. The gas analysis is honest: at most `maxNestingDepth()` hops x ~30,000 gas per lookup, borne only on transfers (rare), not on execution (frequent). The default depth of 4 maps to real organisational structures (parent -> subsidiary -> department -> project).

### EIP-5192 integration for lock state

Using an existing standard (Soulbound Token lock interface) for the binary lock state rather than inventing a new one is the right call. Marketplaces already understand EIP-5192. This means OpenSea and others can display lock status without custom integration.

### The approval revocation interface design

Allowing revocations during the execution freeze is clever and necessary. The constraint that revocation functions can only zero out approvals (hardcoded calldata, no arbitrary execution) maintains the sell-and-drain protection while enabling practical pre-transfer cleanup. This resolves what would otherwise be an irreconcilable tension.

### Direct-owner path and VOPS/FOCIL compatibility

The deliberate decision to keep the baseline execution path as an ordinary transaction is strategically important. By making the NFT-controlled account a *destination* rather than a *sender type*, the standard avoids the need for mempool-side EVM validation and stays compatible with inclusion lists and partial statelessness. This is forward-looking infrastructure thinking, not just feature design.

### No burn path

Forbidding controller token burning eliminates an entire class of bugs where accounts become permanently inaccessible through accidental or malicious token destruction. The account still exists, the token still exists, someone always controls it.

---

## 2. Issues

### 2.1. CRITICAL: Surviving token approvals are a ticking bomb for marketplace sales

The spec acknowledges this extensively in Security Considerations but does not solve it at the protocol level. The problem:

1. Alice controls Account A. She grants unlimited ERC-20 approval to Uniswap Router, Aave Lending Pool, and three other protocols.
2. Alice sells control to Bob by transferring the controller NFT.
3. Those five approvals survive. They are stored on the individual ERC-20 contracts and cannot be enumerated on-chain.
4. Bob has no reliable way to discover all outstanding approvals without off-chain indexing.
5. Any of those approved contracts (or, more dangerously, any contract Alice previously approved that turns malicious) can still drain assets from Account A.

The spec says "wallets SHOULD surface outstanding approvals" but this is a SHOULD on external tooling, not a MUST on the standard. For a standard that explicitly enables account *sale*, this is insufficient. A buyer paying real money for an account cannot rely on wallet UX to protect them.

**Concrete suggestion:** The standard should either (a) MUST require batch revocation of all known approvals as part of the transfer flow (discoverable via events), or (b) define a `revokeAllKnownApprovals` pattern that the account maintains as an append-only list of contracts it has ever approved, or (c) explicitly state that account sale without off-chain approval auditing is unsafe and MUST NOT be presented to users as a safe operation. Option (b) adds storage costs but makes the system self-contained. Option (c) is honest but limits the use case.

### 2.2. HIGH: The execution-active guard relies on transient storage semantics that are subtle

The recommended implementation uses `TSTORE`/`TLOAD` with reference counting, relying on EVM revert semantics to roll back transient-storage writes on failure without explicit decrement. This is correct under current EVM semantics (transient storage is rolled back on revert), but:

1. The spec says "Other mechanisms that provide the same behavioral guarantee... are compliant." This is dangerously open-ended. An implementation using persistent storage with manual decrement has a revert-path bug if the decrement is missed.
2. The reference-count approach means that if Account A calls Contract X, which calls back into Account A's `execute` (reentrant), the counter increments to 2, and the transfer guard correctly stays active. But if the reentrant call exits successfully, the counter decrements to 1, and execution is still considered active. This is correct. But what if Account A's batch calls Account A itself? The spec doesn't prohibit self-calls via `execute`, and a self-call to `execute` would increment the counter. If the outer batch reverts after the inner call succeeds, transient storage rollback handles it. If the outer batch succeeds, the counter decrements correctly. This needs test vectors in the reference implementation.

### 2.3. HIGH: Cross-chain address stability is weaker than presented

The user-salt deployment mode claims cross-chain address stability: "the same deployer, salt, factory address, and account implementation produce the same account address on every chain." But this requires the factory to be deployed at the same address on every chain AND the account implementation to be at the same address on every chain. The spec does not standardise factory deployment addresses. It mentions EIP-7997 for "keyless deployment or deterministic factory predeploy patterns" but treats this as out of scope.

In practice, this means cross-chain address stability is an implementation detail, not a standard guarantee. The spec should be more honest about this. A user who receives funds at a counterfactual address on Chain B, expecting to deploy the same account there later, is making an assumption the standard does not actually enforce.

### 2.4. HIGH: The validator interface is underspecified

The validator interface defines `onInstall`, `onUninstall`, and `isValidSignatureForAccount`. But:

1. **No standardised execution entrypoint for validators.** The spec says validators exist "for delegated signature validation, not to define root control" and "the base ERC does not define validator-driven execution entrypoints." This means validators can validate signatures but cannot actually execute transactions. Session keys, agent wallets, and automated strategies all need execution authority, not just signature validation. The spec explicitly mentions session keys and AI agents in the motivation but provides no execution path for them.

2. **No permission scoping interface.** Validators have no standard way to express "this validator can approve transactions up to X ETH" or "this validator can only call contracts in set Y." The spec mentions "spending limits and contract whitelists" in the motivation but provides no interface for them.

3. **No validator enumeration.** There is no `getActiveValidators()` view. A new controller after transfer has no on-chain way to discover what validators were previously installed (they are all inactive due to version scoping, but the new controller cannot verify this without knowing what to check).

**Concrete suggestion:** Either (a) expand the validator interface to include an execution path with permission scoping, making session keys and agents first-class, or (b) explicitly scope validators as signature-only and define a separate "operator" or "session" extension for execution delegation. The current middle ground promises more than it delivers.

### 2.5. MEDIUM: The factory initCalls authorisation model has a privilege escalation vector

`deployAccountConfigured` mints the token to `initialOwner` and then "the factory acts on behalf of the initialOwner for this one-time initialization only." But the factory is not the owner. The factory is calling `executeBatch` on the account. For the account to accept this, either:

1. The factory is hardcoded as an authorised caller during deployment (before the first `execute` check), which means the factory has a special privilege path, or
2. The account checks `msg.sender == ownerOf(tokenId)` and the factory is not the owner, so the call should revert.

The spec needs to clarify the exact authorisation mechanism for `initCalls`. If the factory has a special bypass, that bypass must be exhaustively constrained (one-time, same transaction, no persistent authority). If the factory calls through the owner somehow, the mechanism needs to be explicit.

### 2.6. MEDIUM: The 1-second minimum unlock delay is too small for meaningful protection

`unlockDelayOf(tokenId)` has a floor of 1 (second). A 1-second delay provides essentially no protection for counterparties. The sell-and-drain mitigation depends on the delay being long enough for off-chain monitoring to observe the frozen state. One second is 1/12th of a block time on mainnet. No monitoring system can reliably react in that window.

The meta-timelock means reducing from the default 3600 to 1 takes at least 3600 seconds. But once reduced, the owner has a 1-second window forever. For accounts that will be sold, this is insufficient.

**Concrete suggestion:** Either (a) set a higher minimum (60 seconds? 300 seconds?) or (b) require marketplaces to check and display the effective delay, or (c) allow the controller token to expose a `minUnlockDelay()` that implementations can set. The current floor of 1 is formally correct but practically toothless.

### 2.7. MEDIUM: No standard for account upgradeability

The spec says "Upgradeability is not core to this ERC" and notes trust assumptions, but offers no guidance on upgrade mechanisms. This is a problem because:

1. Smart contract bugs happen. An account with immutable logic and a non-burnable controller token is a permanent liability if a vulnerability is found.
2. The EVM evolves. New opcodes, new precompiles, new standards will want integration.
3. The spec says implementations "SHOULD prefer immutable logic or clearly disclosed upgrade paths" but provides no standard for the upgrade path.

If every implementation invents its own upgrade mechanism, the ecosystem fragments. If none do, the first serious bug in a widely deployed implementation creates an irrecoverable crisis.

### 2.8. MEDIUM: The ERC-7739 recursive composition limitation is acknowledged but not resolved

The spec correctly notes: "Because ERC-7739 defensive rehashing is not recursively composable across multiple signers of the same kind, a controller contract that itself only accepts ERC-7739-wrapped ERC-1271 signatures may require a validator for delegated-signature paths."

This means: if Account A is controlled by Account B (which is itself an ERC-XXXX account), and both use ERC-7739, the signature chain breaks. The workaround is installing a validator, but this means the "simple" case of nested accounts requires additional configuration for off-chain signing to work. The spec should quantify how common this case is and whether the workaround is acceptable for the hierarchical account trees it promotes.

### 2.9. LOW: Event hashing strategy for Executed and BatchExecuted may complicate indexing

`dataHash` and `resultHash` in the `Executed` event use `keccak256` of calldata and result respectively. `batchHash` in `BatchExecuted` uses `keccak256(abi.encode(calls))`. The rationale is avoiding per-byte log gas costs. But this means:

1. Indexers cannot reconstruct the actual calls or results from events alone; they need trace data.
2. The `batchHash` uses standard ABI encoding of a dynamic array, but verifying a specific call was in a batch requires the complete call array.

This is a reasonable gas optimisation, but it degrades the self-contained auditability of the event log. The tradeoff should be explicitly acknowledged: you are trading event-level auditability for gas efficiency.

### 2.10. LOW: No standard for account discovery beyond ERC-721 Enumerable

The spec recommends (SHOULD) ERC-721 Enumerable on the controller token. But for a user who holds controller NFTs across multiple independent controller token deployments, there is no standard way to discover all their accounts. Each deployment is a separate ERC-721 contract. A user needs to know which controller token contracts exist and query each one.

This is inherent in the "no central registry" design choice, which has other benefits (no singleton dependency). But it means account discovery is an off-chain indexing problem, which limits the "self-sovereign" narrative.

### 2.11. LOW: No guidance on gas limits for execute/executeBatch

The `execute` function performs "a single CALL to target with value and data." The spec does not address gas forwarding. How much gas does the account forward to the target? All remaining gas minus a reserve? A specific amount? The standard EVM behaviour for CALL forwards 63/64 of remaining gas, but implementations that want to ensure post-call bookkeeping (decrementing the execution-active counter) need to reserve gas for that.

---

## 3. Usage Scenarios I Like

### 3.1. DAO treasury handoff

This is the killer application and the spec explains it well. When a DAO votes to change its treasury management committee, the old committee transfers the controller NFT to the new multisig. The treasury address, its Aave positions, its Uniswap LP, its governance participation history, its protocol allowlists -- everything stays. No multi-governance-vote migration. No "update the address in 47 different protocols." One transfer.

This alone justifies the standard. Every DAO that has gone through a treasury migration knows the pain. The gas costs, the missed approvals, the protocols that reference the old address, the governance votes to update each integration. All eliminated.

### 3.2. Digital inheritance

The controller NFT held by a dead man's switch contract or a multisig with designated heirs. The entire on-chain estate transfers via a single NFT operation. No shared seed phrases, no custodians, no probate court trying to understand private keys. This is the kind of thing that makes crypto actually usable by normal humans.

### 3.3. Vesting with transferable beneficiary claims

The account holds vesting tokens. The controller NFT represents the right to eventually control them. The NFT itself can be sold (transferring the claim) without the grantor releasing tokens early. This creates a secondary market for vesting positions without touching the vesting logic. Clever.

### 3.4. Risk isolation with unified control

Multiple accounts under one key, each with separate approval scopes. The "DeFi degen" account has unlimited approvals to experimental protocols; the "treasury" account has no approvals at all. If the degen account gets drained via a malicious approval, the treasury is untouched. Previously this required managing multiple seed phrases. Now it is one key, many accounts, proper isolation.

### 3.5. Escrow and conditional control

The controller NFT held by an escrow contract. The account is fully operational while the escrow condition is pending, but control transfers atomically when the condition is met. No need to escrow the assets themselves; just escrow the control object. This is particularly powerful for M&A-style acquisitions of on-chain entities.

---

## 4. Additional Possibilities the Authors May Not Be Thinking Of

### 4.1. Credit delegation and on-chain lending collateralised by account control

A controller NFT could be used as collateral for a loan. The NFT goes into a lending vault. While the borrower is current, the vault delegates execution back to them (via a validator or by holding the NFT in a programmable contract that forwards calls). If they default, the vault takes control. The collateral is not "an NFT" -- it is "control over an account with its entire balance sheet." This is fundamentally different from collateralising individual assets because it captures the emergent value of the position portfolio as a whole, including LP positions, yield-bearing deposits, and governance power that cannot be individually priced.

### 4.2. On-chain corporate structures with real juridical parallel

The nesting model (parent controls child accounts via child controller NFTs) maps directly onto corporate subsidiary structures. A holding company (parent account) owns operating subsidiaries (child accounts). Transfer of a subsidiary is a single NFT transfer -- a divestiture. Auditing the corporate structure is on-chain enumeration. This is not just a metaphor; it could become the actual legal structure for on-chain entities in jurisdictions that recognise smart contract governance.

### 4.3. Composable account packages as products

The `deployAccountConfigured` path with `initCalls` enables "account templates." An organisation could publish a template: "Deploy an account pre-configured with our multisig validator, our recovery setup, our approved DeFi protocol list, and our spending limits." Users deploy the template and get a fully configured account. This is software distribution for financial infrastructure. Think of it as "install this wallet configuration" rather than "set up your wallet manually."

### 4.4. Time-locked account succession chains

Multiple controller NFTs for child accounts, each held by different time-lock contracts with staggered release dates. Account A's controller releases to Person B after 1 year. Account B's controller releases to Person C after 2 years. This creates pre-programmed succession chains for organisations, trusts, or any structure that needs planned control rotation.

### 4.5. AI agent containment via child accounts

The spec mentions session keys and agents. But the stronger pattern is: give the AI agent its own child account with a bounded balance. The agent has full execution authority within that account (via a validator or direct control). The parent account refills the child periodically. If the agent goes rogue, the loss is bounded by the child account balance, and the parent can revoke by transferring the child's controller NFT away. This is not just permission scoping; it is custody containment. The agent literally cannot access more than what is in its account.

### 4.6. Programmable account "modes" via controller contracts

The controller NFT can be held by a mode-switching contract. In "normal mode," the contract forwards all execution requests from the user. In "lockdown mode," it only allows revocations and withdrawals to a predefined safe address. In "governance mode," it requires multi-sig approval for transactions above a threshold. The user switches modes by calling the controller contract, not by reconfiguring the account. The account itself never changes; only the behaviour of its controller changes.

### 4.7. Marketplace for operating businesses

Accounts that have been operating DeFi strategies, accumulating governance power, building on-chain reputation, and maintaining protocol relationships become sellable as going concerns. This is not "sell your JPEGs" -- it is "sell your on-chain business with its full history, positions, and relationships intact." The unlock delay and execution freeze make this safe. The transfer is atomic. The buyer gets the business as-is.

### 4.8. Sovereign identity anchoring

Because the account address is permanent and control can rotate without changing it, the address becomes a stable identifier. ENS names, attestations, protocol memberships, governance history -- all anchored to an address that never changes even as the controlling key, security model, or human behind it changes. This is proper self-sovereign identity infrastructure, not because the spec claims it, but because the properties emerge from the design.

---

## 5. Changes to Increase Optionality

### 5.1. Add an optional approval registry to the account

Add an append-only set of `(token, spender)` pairs that the account populates whenever `execute` or `executeBatch` is used to call `approve`, `setApprovalForAll`, or similar approval functions (detectable by function selector in calldata). This enables on-chain enumeration of outstanding approvals and makes the "revoke all approvals before transfer" flow self-contained. Gas cost is one SSTORE per new approval target. Make it OPTIONAL but standardise the interface so tooling can rely on it when present.

```solidity
function knownApprovals() external view returns (ApprovalRecord[] memory);
function revokeAllKnownApprovals() external; // callable during freeze
```

### 5.2. Define a standard execution delegation interface alongside the validator interface

The validator interface handles signature validation. Define a parallel "operator" interface for execution delegation:

```solidity
interface IERCXXXXOperator {
    function canExecute(
        address account,
        address target,
        uint256 value,
        bytes calldata data
    ) external view returns (bool);
}
```

This gives session keys, AI agents, and automated strategies a standardised execution path with permission scoping. The account checks `msg.sender` against installed operators before falling through to the validator path. Operators are version-scoped like validators.

### 5.3. Allow a "transfer callback" hook on the account

After a transfer completes and the new controller takes over, allow the account to execute a pre-registered callback (set by the previous controller, stored on-chain). This could be used for:

- Automatic approval revocation on transfer
- Notification to dependent protocols
- State cleanup

Constrain it: the callback runs under the *new* controller's authority, is gas-bounded, and can be a no-op. But the hook point is valuable.

### 5.4. Standardise a minimum viable recovery interface

Recovery is OPTIONAL, but when it exists, it "MUST preserve the core invariant" and "SHOULD include configurable delay, cancellation path, clear guardian update rules, transparent events." This is enough guidance for implementers to diverge wildly. Define a minimal standard recovery interface:

```solidity
interface IERCXXXXRecovery {
    function initiateRecovery(uint256 tokenId, address newOwner) external;
    function cancelRecovery(uint256 tokenId) external;
    function completeRecovery(uint256 tokenId) external;
    function recoveryDelay(uint256 tokenId) external view returns (uint256);
    function pendingRecovery(uint256 tokenId) external view returns (address newOwner, uint256 readyAt);
}
```

This does not mandate recovery. It standardises the interface *when* recovery is implemented, so tooling can support it generically.

### 5.5. Consider a "read-only mode" between propose-unlock and complete-unlock

Currently, the execution freeze during the unlock window is total: no `execute`, no `executeBatch`. This is correct for safety but prevents legitimate read-only interactions. Consider allowing `STATICCALL`-only execution during the freeze:

```solidity
function staticExecute(address target, bytes calldata data) external view returns (bytes memory);
```

This allows the account to query protocol state (check positions, read balances, verify oracle prices) during the transfer preparation window without opening a drain vector. The `view` modifier ensures no state changes.

### 5.6. Expose a "controller chain" view for hierarchical accounts

For nested accounts, add a view that returns the full controller chain up to the root:

```solidity
function controllerChain() external view returns (address[] memory chain);
```

This walks the ownership hierarchy from this account up to the root controller (the first address that is not itself a controlled account). Bounded by `maxNestingDepth()`. This enables tooling to display "Account A -> controlled by Account B -> controlled by EOA 0x..." without multiple separate queries.

### 5.7. Consider a "delegate execution during freeze" mechanism for time-sensitive operations

Some accounts may have time-sensitive obligations (e.g., Aave positions approaching liquidation) that cannot wait for the freeze to end. Consider allowing the controller to designate a "freeze-safe executor" that can perform specific whitelisted operations (e.g., repay debt, add collateral) during the freeze window. This is more dangerous than pure revocation, so it needs careful scoping, but the alternative (forced liquidation during a transfer) is also undesirable.

### 5.8. Standardise the controller token contract's interface ID

The spec requires EIP-165 but does not define the `interfaceId` for the controller token extension (`IERCXXXXControllerToken`). This should be computed and specified explicitly so that any contract can query "is this an ERC-XXXX controller token?" via `supportsInterface`.

### 5.9. Add an explicit "non-transferable" mode

Some accounts should never be transferable. The current design requires `unlockDelay >= 1`, meaning every account is *theoretically* transferable. Consider allowing `unlockDelay = 0` as an explicit "permanently non-transferable" mode (where `proposeUnlock` always reverts). The spec currently forbids this, arguing it "collapses the required separation between execution and transfer." But for accounts that will never be transferred, this separation is unnecessary, and the permanently-locked state is a valid and useful configuration.

An alternative: a boolean `transferable` flag set at mint time. If `false`, the unlock flow is permanently disabled.

---

## Summary Assessment

This is a strong draft. The core mechanism -- NFT as title deed for account control -- is sound, well-motivated, and solves real problems. The sell-and-drain protection is the best-designed part and demonstrates genuine adversarial thinking. The VOPS/FOCIL alignment shows strategic foresight about where Ethereum infrastructure is heading.

The main weaknesses are:

1. **Surviving approvals** are the biggest practical risk for the headline use case (account sale) and are not solved at the protocol level.
2. **The validator interface** promises more than it delivers -- session keys and agents are motivated but not mechanised.
3. **Cross-chain stability** is claimed but not guaranteed by the standard.
4. **The factory initCalls authorisation** needs precise specification of its privilege model.

The additional optionality suggestions above would make the standard significantly more powerful without compromising its core invariants. The approval registry (5.1) and execution delegation interface (5.2) are the highest-impact additions.

The writing quality is high. The security considerations section is unusually thorough for an ERC draft. The comparison with ERC-6551 is honest and precise. The regulatory considerations section is refreshingly direct about what the standard is and is not.

Where are the numbers? The gas analysis for cycle detection is there (good). The unlock delay analysis is qualitative but correct. What is missing is a formal analysis of the state machine transitions -- ideally a state diagram with all 6 states (locked/no-proposal, proposal-pending, proposal-ready, unlocked, executing, transferring) and all valid transitions, with the mutual exclusion property proven rather than argued. The current prose is convincing but a formal model would be stronger for a standard that will hold real money.

Build the reference implementation. Run the test vectors. Ship it.

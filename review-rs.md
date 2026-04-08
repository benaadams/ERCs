# Behavioral Economics Review: ERC-XXXX (Wallet Title Deeds)

**Reviewer perspective:** Perception theory, behavioral economics, counterintuitive UX thinking

---

## 1. What Is Psychologically Clever

### The Title Deed Metaphor Is Doing More Work Than the Authors Realize

The single most important decision in this entire ERC is the word "deed." Not "key," not "token," not "controller." A deed. This word carries centuries of legal and psychological weight that no amount of technical specification could manufacture. When someone holds a deed to a house, they do not confuse the piece of paper with the house. They do not expect the furniture to travel with the deed when they sell it. They understand, intuitively, that the deed confers authority over a thing that is separate from the deed itself. The control-custody separation that takes the Specification section hundreds of lines to formalize is already understood by anyone who has ever bought property.

This is not a minor branding choice. It is the load-bearing psychological architecture of the entire proposal. The metaphor pre-solves the hardest UX problem identified in the Security Considerations section: "Users MAY transfer the NFT expecting the assets to move." A deed-holder does not expect this. A key-holder might. The metaphor choice determines which cognitive error is the default.

The authors should lean into this harder. The abstract currently says "transferable control credential." That is technically precise and psychologically inert. "Title deed" should appear in the Abstract's first sentence and never be abandoned for engineering synonyms.

### The Transfer Lock Is a Trust Machine, Not a Safety Mechanism

The `proposeUnlock` / `completeUnlock` / `lock` state machine (lines 153-191 of the spec) is presented as sell-and-drain protection. That is its engineering function. Its psychological function is more interesting: it is a visible commitment device.

The one-hour default `unlockDelay` does not primarily protect the buyer. It signals to the buyer that the seller has committed to a period of vulnerability. The seller cannot execute during the unlock window. The seller has, in effect, put their hands up and said "I am not going to touch anything for the next hour, and you can verify this on-chain." This is the Ethereum equivalent of a car dealer handing you the keys and leaving the room. The asymmetric meta-timelock on decreasing the delay (the delay reduction itself takes as long as the current delay to take effect) is even more powerful: it means the current delay has held for at least as long as itself. A 24-hour delay has been 24 hours for at least 24 hours. The delay is its own proof of age.

This is genuinely novel trust infrastructure. The spec buries it in implementation details. It should be called out as a first-class feature of the standard because its behavioral value -- "I can see this account has been frozen for 24 hours and the owner could not have reduced that freeze window in the last 24 hours" -- is the thing that makes marketplace sale of accounts psychologically viable.

### Execution Freeze Creates a Clean Perception of Handoff

The mutual exclusivity between "execution allowed" and "marketplace approval valid" (lines 191-193) is a beautiful piece of perceptual engineering. In the physical world, we understand that you cannot simultaneously live in a house and have it on the market for immediate vacant possession. This standard creates the digital equivalent. The account is either "in use" or "being transferred" -- never both. This eliminates the cognitive load of wondering "what is the seller doing to this account while I am trying to buy it?" The answer is provably "nothing." That certainty is worth more than a hundred technical safeguards that require the buyer to understand what they protect against.

### Hierarchical Accounts Map to How Organizations Actually Think

The nesting model (parent account owns child account controller tokens) is clever not because of its technical properties but because it maps to an existing mental model: the org chart. A corporate treasury controls department accounts. Department accounts control project accounts. This is how every CFO already thinks about money. The innovation is not the hierarchy -- it is that the hierarchy is transferable at any cut point. Divesting a subsidiary means transferring one NFT. The authors correctly identify this in the Motivation section, but the deeper behavioral insight is that this makes organizational restructuring feel like moving files between folders rather than like a legal proceeding.

### Approval Revocation During Freeze Is Psychologically Asymmetric in the Right Direction

The revocation interface (lines 347-363) allows the seller to clean up outstanding approvals during the transfer freeze, but not to grant new ones. This is the rare case where a constraint is doing exactly the right psychological work. From the buyer's perspective, the account can only get cleaner during the freeze, never dirtier. This creates a one-directional trust ratchet: every action the seller takes during the freeze window is legibly pro-buyer. That one-directionality is what makes marketplace interactions psychologically safe.

---

## 2. Issues: Perceptual Gaps and Behavioral Blind Spots

### The Surviving Approvals Problem Is a Perception Catastrophe Waiting to Happen

The spec acknowledges this in Security Considerations (line 979): token-level approvals survive transfer. ERC-20 allowances the previous owner granted to DeFi protocols remain active after the deed transfers. The spec's proposed solution is "wallets and marketplaces SHOULD surface outstanding approvals" and "the new controller SHOULD review and batch-revoke."

This is an engineering-culture answer to a psychological problem. The correct framing is: **the buyer has just purchased what they perceive as a clean account and has in fact purchased an account with an unknown number of open drain pipes.** There is no on-chain enumeration of outstanding approvals -- the spec says so explicitly. The buyer must rely on off-chain indexing services to discover what they have bought.

This is not a technical limitation. It is a trust-perception failure. Imagine buying a house where the previous owner had given copies of the keys to seventeen different locksmiths, and the deed transfer did not change the locks, and there is no registry of who has copies, and you have to hire a private investigator to find out. The title deed metaphor, which works so well elsewhere, makes this specific failure more jarring, not less. You expect a deed transfer to convey clean title.

**Specific recommendation:** The standard should RECOMMEND (or even REQUIRE) that `deployAccountConfigured` deployments include a batch revocation of all known approvals as part of the initialization, and that marketplace integrations surface a "clean title" certification based on off-chain approval enumeration. The concept should be named -- "encumbered transfer" vs. "clean transfer" -- so that the ecosystem develops vocabulary for the distinction. A buyer who sees "this account has 3 outstanding unlimited ERC-20 approvals" and can click "revoke all before transfer completes" is in a fundamentally different psychological position than a buyer who must discover this after the fact.

### The One-Hour Default Unlock Delay Is Probably Wrong in Both Directions

The default `unlockDelay` of 3600 seconds (one hour) is presented as a reasonable default. From a behavioral perspective, it is in the uncanny valley of waiting. One hour is too short for a marketplace buyer to perform meaningful due diligence on a valuable account (examine positions, check approvals, verify protocol relationships) and too long for a programmatic transfer between accounts the user already controls.

The deeper problem is that the default anchors expectations. Whatever the default is will become the norm because most users will never change it. The one-hour default says "one hour is enough time to verify an account before purchase." For a high-value DeFi account with dozens of positions and protocol relationships, it is not. For an account transfer between your own hot wallet and cold storage, one hour is absurd.

**Specific recommendation:** Consider two-track defaults. Programmatic same-owner transfers (detectable by examining whether the destination already controls accounts in the same hierarchy) could have a shorter default. Marketplace-visible transfers could default longer. Alternatively, abandon a single default in favor of requiring the deployer to choose during `deployAccountConfigured`, making the choice conscious rather than defaulted.

### The NFT Mental Model Will Cause Catastrophic Misunderstanding

The spec correctly identifies this (line 975): "Users MAY transfer the NFT expecting the assets to move. Users MAY approve the NFT as if it were a collectible." But the mitigation -- "Implementers SHOULD treat these as security problems" -- shifts the burden to wallet developers rather than addressing the perceptual root cause.

The problem is that the controller token is an ERC-721 token. Every existing mental model for ERC-721 tokens is wrong for this token. Users who have interacted with NFTs know that approving an NFT lets someone take it, and taking it means they have the art/membership/whatever. Users who have interacted with NFTs do NOT know that approving this particular NFT means someone can take control of a separate contract that holds all their money.

The `setApprovalForAll` prohibition is a good structural safeguard. But the single-token `approve` remains available (it must be, for marketplace listing). A user who approves their controller token for a marketplace listing is granting root account access to that marketplace contract. They may understand this intellectually. They will not feel it in their bones, because every prior NFT approval has been a transaction about an image, not about custody of a financial portfolio.

**Specific recommendation:** The standard should RECOMMEND that compliant controller tokens return metadata that makes their nature impossible to ignore. The `tokenURI` default (when no custom metadata is set) should return something that screams "THIS IS AN ACCOUNT CONTROL TOKEN" rather than anything that could be mistaken for a collectible. The optional `setTokenName` / `setTokenImage` features are nice personalization, but the default state matters more than the customized state because most tokens will never be customized.

### The Metadata Feature Buries the Most Important Signal

The optional metadata section (lines 744-765) allows users to name their accounts and assign NFT avatars. This is presented as a convenience feature. It is actually the primary legibility mechanism for the entire system, and making it OPTIONAL is a mistake.

Consider: a user holds five controller tokens. Without metadata, they see five identical tokens with hex tokenIds that encode addresses. The tokens are visually identical. The user must remember which address is their savings account and which is their DeFi interaction account. This is the same failure mode as having five unmarked keys on a keyring -- it works until it does not, and when it fails, the failure is catastrophic (transferring the wrong account's controller).

**Specific recommendation:** Make `setTokenName` RECOMMENDED rather than OPTIONAL. Better yet, require the factory to accept an initial name during deployment. The psychological cost of "name your new account" at creation time is near zero. The psychological cost of discovering you transferred the wrong unmarked account controller is potentially infinite.

### Hierarchical Nesting Creates a Principal-Agent Perception Gap

The nesting model allows account A to control account B by holding B's controller token. This means that to execute on B, the owner of A must cause A to call B. From the user's perspective, they are "doing something on account B." From the protocol's perspective, they are "executing on A, which executes on B." The gas costs, the failure modes, the event emission, and the authorization chain are all different from what the user perceives.

The four-level default nesting depth (line 197) means the worst case is a user operating on a great-grandchild account through three layers of indirection. Each layer adds gas cost, latency, and failure surface. But the wallet UI will (and should) present this as "click to execute on Project Alpha account." The gap between perceived simplicity and actual complexity is a reliability perception problem: when a nested execution fails, the user will have no intuition for why, because their mental model has no layers.

**Specific recommendation:** The standard should RECOMMEND that wallet implementations surface nesting depth as a visible property of each account ("depth: 2 of 4") and display the full control chain when executing on nested accounts. Not because users will read it every time, but because the first time a nested execution fails, they need a mental model to debug it.

### The Recovery Section Is Underspecified Where Perception Matters Most

Recovery is marked OPTIONAL and specified in broad strokes: "configurable delay, cancellation path, clear guardian update rules, transparent events." For a standard that will handle real money, the recovery UX is where users' relationship with the system is most emotionally charged. The moment someone needs recovery is the moment they are most frightened, most confused, and most susceptible to social engineering.

The spec correctly says "recovery MUST preserve the core invariant" (recovery is NFT transfer). But it says nothing about what the user should perceive during recovery. How long should the recovery delay be? What happens if the legitimate owner returns during the recovery window? What does "cancellation path" look like to someone who just realized their guardian is hostile?

**Specific recommendation:** While keeping recovery OPTIONAL at the specification level, the standard should include a dedicated "Recovery UX Guidance" subsection that addresses the psychological dimensions: what should a user see during an active recovery attempt against their account, how should they be notified, what should the cancellation flow feel like. The inheritance use case mentioned in the Motivation section is particularly underspecified -- "dead man's switch" is mentioned but the perception of that mechanism from the heir's perspective (how do they know the switch has triggered? how do they claim? what if there is a dispute?) is entirely unaddressed.

---

## 3. Usage Scenarios That Are Most Compelling

### Account Marketplaces: Selling Reputation as a First-Class Operation

The most psychologically interesting use case is not account transfer between known parties -- it is selling an account with its full history to a stranger. An Ethereum address that has been active for years, that has an Aave credit history, a governance participation record, an ENS name, protocol allowlist memberships, and a track record of profitable LP positions, is worth more than the sum of the tokens it holds. The address IS the reputation.

Today, this reputation is non-transferable. Under this standard, it becomes a tradeable asset. The unlock delay, the execution freeze, the approval revocation interface -- these are not security features in the abstract. They are the specific infrastructure that makes reputation markets psychologically viable. A buyer can inspect the account during the freeze window, verify the position history, check the approval hygiene, and purchase with confidence that the account they inspected is the account they receive.

This is the use case that will drive adoption if anything does, because it creates value that is currently trapped. Every serious DeFi user has built address-level reputation that they cannot monetize or transfer. This standard unliquidates that value.

### Digital Inheritance Without Shared Secrets

The inheritance use case is the most emotionally compelling. The current state of digital inheritance in crypto is: write your seed phrase on paper and put it in a safe deposit box, or trust a centralized custodian, or lose everything when you die. All three options are terrible for different reasons.

Under this standard, a controller token held by a dead man's switch contract or a multisig with designated heirs transfers an entire on-chain estate -- every position, every membership, every relationship -- in a single atomic operation. The heir does not need to know what the deceased held. They do not need to enumerate wallets. They do not need to import seed phrases. They receive a deed, and the deed controls everything.

The psychological power here is that it maps to how physical inheritance works. You inherit a house by receiving the deed, not by receiving a copy of every key to every room. The beneficiary's mental model is already correct.

### Organizational Treasury Handoff

The DAO committee rotation example in the Motivation section is compelling but undersold. The real insight is that this standard makes organizational continuity independent of individual continuity. When a company changes CFOs, the company's bank accounts do not change addresses. Every vendor, every automated payment, every integration continues working. This standard creates the on-chain equivalent.

The nested hierarchy makes this even more powerful for the organizational case: divesting a business unit means transferring one controller token, and the unit's entire on-chain infrastructure -- treasury, department budgets, project accounts, protocol positions, contract whitelists -- transfers as a single tree. No migration. No address updates. No broken integrations. The organizational handoff feels like moving a folder, because it IS moving a folder.

---

## 4. Additional Possibilities the Authors May Not Be Thinking Of

### Account Leasing: Temporary Control Transfer Without Permanent Sale

The standard defines transfer as permanent (until re-transferred). But the infrastructure supports something the spec never mentions: account leasing. A controller token could be transferred to a time-locked escrow contract that automatically returns it to the original owner after a period. This enables:

- **DeFi strategy rental:** A user with a valuable account (high credit score, protocol relationships, allowlist positions) leases control to a professional manager for a defined period. The manager operates the account; the owner gets it back automatically.
- **Collateralized borrowing against account value:** The controller token is escrowed as collateral. If the borrower defaults, the lender receives the account. If they repay, the token returns. The account's DeFi positions, reputation, and relationships are the collateral -- not just the token balances.
- **Trial periods for account purchases:** A buyer receives temporary control to verify the account works as expected before committing to full purchase.

The unlock delay and execution freeze already provide the trust infrastructure for this. A time-locked holder contract that automatically re-transfers is a natural extension that the standard enables without modification.

### Accounts as Composable Financial Instruments

Because the controller token is a standard ERC-721, it can be used anywhere ERC-721 tokens are accepted. This means:

- A controller token can be listed as collateral on an NFT lending platform, creating a loan backed by the entire account's contents.
- A controller token can be placed in a Dutch auction contract, creating a declining-price sale of an entire account.
- A controller token can be held by a Gnosis Safe, creating shared custody over the account's contents with arbitrary governance rules.

The standard already knows this (the Motivation section mentions vesting contracts and governance contracts as holders). What it does not explore is the second-order effect: the existence of standardized controller tokens creates a new asset class -- "account equity" -- that existing DeFi infrastructure can immediately price, trade, collateralize, and compose with.

### The "Blank Account" Economy

The deterministic deployment semantics, combined with sponsored deployment, enable a use case the spec does not mention: pre-deployed blank accounts as a service. A service could deploy thousands of accounts and sell the controller tokens as "ready-to-use wallet slots." Because deployment is the expensive step (gas cost) and the controller token is the cheap step (already minted), this creates an economy of blank accounts that users can purchase and immediately customize.

This is the Ethereum equivalent of buying a pre-registered domain name. The account address exists, the deployment is done, the controller token is in the marketplace. You buy the token, you own the account, you start using it. The address you get might even have desirable vanity properties (short hex, memorable pattern) that add non-functional value.

### Intelligence and Attestation Markets

The hierarchical nesting, combined with the fact that account addresses persist across control changes, creates something the spec does not discuss: verifiable account provenance. An account that has been controlled by a known reputable entity, that has a clean transaction history, that has never been involved in exploit-related transactions, has attestable provenance that survives transfer. 

Third-party attestation services could emerge that certify account histories: "This account has no outstanding unlimited approvals, has been active for 2 years, has never interacted with a sanctioned address, and has a clean governance voting record." These attestations attach to the account address (which is stable) rather than to the controller (which changes). This creates a market for account quality that is distinct from account contents.

### "Account Insurance" via Guardian Structures

The recovery mechanism combined with hierarchical nesting enables something never mentioned: account insurance as a service. A professional guardian service could act as recovery guardian for a fee, providing guaranteed recovery within a defined SLA. The insurance value is real because the thing being insured (the controller token) can be recovered (transferred to a new owner) through a defined on-chain mechanism. This is unlike seed phrase insurance, which requires trust in the insurer not to use the seed phrase. The guardian mechanism is structurally constrained: recovery ends in NFT transfer to the designated recipient, not in arbitrary account access for the guardian.

---

## 5. Changes That Could Increase Optionality

### Make the Unlock Delay a Range, Not a Point

The current design has a single `unlockDelay` per token. Consider instead specifying a minimum and maximum delay, where the owner can choose any delay within the range for each unlock proposal. The minimum protects buyers (no instant transfers). The maximum protects sellers (no indefinite freeze). The range is set at deployment and can only be narrowed (higher minimum or lower maximum) through the same meta-timelock mechanism.

This creates a more nuanced trust signal. An account with a minimum delay of 24 hours and a maximum of 72 hours tells the market something different from an account with a minimum of 1 second and a maximum of 1 year. The range itself becomes a legible property of the account's trust profile.

### Define a "Transfer Intent" Event Before Unlock

Currently, the first visible signal that a transfer may happen is `proposeUnlock`. But `proposeUnlock` conflates "I want to transfer" with "I am preparing to transfer." Consider adding a non-binding "transfer intent" event that the owner can emit at any time without starting the unlock clock. This would serve as a signal to the market that the account may become available, enabling price discovery before the transfer window opens.

The psychological value: buyers can begin their due diligence before the clock starts, rather than scrambling to evaluate the account within the unlock window. This transforms the unlock delay from a rushed evaluation period into a confirmation period where the buyer has already done their homework.

### Allow Controller Tokens to Carry Structured Metadata About the Account's Purpose

The current metadata spec (name and avatar) is personal. Consider allowing the controller token to carry machine-readable purpose metadata: "treasury," "DeFi interaction," "agent operating account," "escrow," "child-of-[parent-tokenId]." This would enable wallets to auto-organize accounts without manual setup and would make marketplace listings more informative.

The behavioral insight: users do not organize things voluntarily. They need defaults. If the deployment flow asks "what is this account for?" and records the answer in the controller token's metadata, wallets can auto-sort accounts into categories. Without this, most users will have five identical-looking controller tokens and will rely on memory to distinguish them.

### Specify a "Condition Report" Standard for Marketplace Listings

The biggest barrier to account marketplaces will not be technical. It will be information asymmetry. A buyer looking at a listed account does not know what they are getting. The standard should define (or at least sketch) a structured "condition report" format: outstanding approvals count, validator configuration summary, nesting depth, unlock delay history, last execution timestamp, and control version history. This is the Carfax report for on-chain accounts.

This does not need to be on-chain. An IPFS-hosted JSON document conforming to a defined schema, linked from the controller token's metadata, would suffice. The point is standardization: every marketplace listing for a controller token should present the same structured information, so buyers can compare accounts the way they compare used cars.

### Add an Optional "Viewing Key" or Read-Only Access Layer

Currently, account control is binary: you either hold the controller token (full access) or you do not (no access). For the organizational use case, there is a missing middle: read-only visibility. A board member who does not control the treasury may still need to view its contents. An auditor needs to inspect but not execute.

The standard does not need to solve this fully, but acknowledging the gap and reserving interface space for a future read-only delegation would increase optionality. Even a simple `grantViewAccess(address viewer)` that emits an event (allowing off-chain tools to know who is authorized to see detailed account information) would enable a richer organizational model.

### Consider a "Quiet Transfer" Mode for Same-Owner Reorganization

Transferring a controller token between two accounts you already control (reorganizing your own hierarchy) should not trigger the same unlock delay as a sale to a stranger. The current design treats all transfers identically. A "quiet transfer" mode, available only when the sender and recipient share a common root controller within the same hierarchy (verifiable via the cycle-detection chain walk), could use a shorter delay or no delay at all.

The behavioral argument: users who are rearranging their own account tree should not be punished with a one-hour freeze for each reorganization step. If restructuring a four-level hierarchy requires four transfers, each with a one-hour freeze, the process takes four hours minimum. This will discourage healthy organizational restructuring and create an incentive to build flat hierarchies even when depth would be more appropriate.

---

## Concluding Observation

The deepest insight in this proposal is one the authors demonstrate but never state: **the problem with crypto wallets was never the cryptography. It was the metaphor.** Keys are the wrong metaphor for account control because keys cannot be transferred without copying, cannot be held by organizations naturally, cannot be nested, and cannot be sold. Deeds can do all of these things, and humans have understood deeds for centuries.

The engineering in this ERC is thorough and careful. The behavioral architecture -- the choice to make account control a transferable physical-feeling object rather than a copyable secret -- is where the real innovation lives. The standard would benefit from treating its psychological properties with the same rigor it applies to its cryptographic ones: specifying not just what the system does, but what the user should perceive, and designing the defaults, the metadata, the marketplace signals, and the recovery flows to serve that perception.

The transfer lock is not a safety mechanism. It is a trust machine. The hierarchy is not an engineering feature. It is an org chart. The controller token is not a technical credential. It is a deed. The more the standard leans into these perceptual identities rather than treating them as marketing, the more likely it is to achieve the adoption that its engineering deserves.

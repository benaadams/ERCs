# Review of ERC-XXXX: Wallet Title Deeds

## Bottom line

I think the core idea is strong: make a smart account's root control a transferable, standardized on-chain object, and make account transfer a first-class operation instead of an awkward byproduct of signer rotation. That is a real conceptual advance. The draft has a clear thesis, and unlike many AA proposals it is trying to solve a concrete user problem: "how do I transfer control of an address, with all of its assets, positions, history, and integrations, without migrating everything or sharing keys?"

Where I think the draft gets weaker is that it does not stop at standardizing that core primitive. It also standardizes a fairly specific opinion about transfer locks, sale flows, validator shape, deployment shape, typed-signature workflow, metadata, hierarchy limits, privacy-pool framing, and mempool politics. A lot of that is thoughtful, but bundling it all into one ERC risks making adoption materially harder than the core idea deserves.

My view is: the title-deed model is good; the draft should become more modular.

## What I like

- The control/custody split is crisp. The account remains the custody address; the NFT is the control object. That is easy to explain, easy to reason about, and materially different from just "an NFT with an account".
- `controlVersion` is one of the best parts of the design. Binding delegated authority and signatures to a transfer-aware version closes a large class of stale-authorization problems that many account systems hand-wave away.
- The draft correctly treats transfer as an operationally dangerous moment. The unlock-delay plus execution-freeze model is trying to solve a real sell-and-drain problem, not inventing ceremony for its own sake.
- The draft is unusually honest about surviving token approvals. That is one of the nastiest practical issues in "sell the wallet, keep the address" systems, and the spec addresses it directly rather than pretending transfer resets everything.
- The direct-owner execution path is compelling. Preserving a plain transaction path instead of making the account fully dependent on ERC-4337-style infrastructure is a meaningful design advantage.
- Hierarchical control is more important than it first appears. "Parent account owns child controller NFTs" is a simple primitive with a lot of organizational power.
- Recovery culminating in NFT transfer is conceptually clean. It avoids the common mistake of having "root ownership" and "recovery ownership" quietly diverge into separate sovereignty models.

## Main issues

- The ERC currently standardizes too much. The real innovation is the controller NFT plus deterministic account mapping plus transfer-aware invalidation. Transfer lock policy, validator interfaces, recovery shape, metadata behavior, configured deployment, and signature workflow should mostly be extensions or profiles, not all part of the same mandatory surface.
- The draft sometimes confuses "good reference implementation policy" with "what should be universal protocol law". A one-hour default unlock delay, the exact asymmetry of delay changes, the specific freeze window, the specific metadata pattern, and the canonical ERC-7739 path all feel more like strong implementation recommendations than core interop requirements.
- The proposal is strongest as an account-control primitive, but it sometimes overstates the extent to which "transferring the account" transfers the whole real-world thing. It transfers root on-chain control. It does not automatically transfer off-chain permissions, exchange whitelists, legal rights, backend entitlements, social identity, or application-specific trust assumptions.
- The draft turns the controller NFT into a bearer instrument for total wallet sovereignty. That is powerful, but it also means theft or mistaken transfer is catastrophic. The transfer lock mitigates some paths, but it does not change the fact that this is much closer to "sell the whole account" than to "send an NFT". Wallet UX and marketplace UX become part of the security model.
- The nested-account model is good, but the spec's bounded hierarchy policy is opinionated. A mandatory bounded walk with a default depth of 4 is a reasonable implementation choice, but it should not be treated as the only sensible shape for all deployments.
- The typed-signature requirements feel too prescriptive. Requiring strong domain binding is good. Picking one canonical 1271 flow so aggressively may narrow adoption before the ecosystem has converged.
- The metadata extension is clever but too specific. Dynamic forwarding of a held NFT's metadata for the deed image is more product design than protocol standardization.
- The draft takes on several arguments around FOCIL, VOPS, 7702, 8202, privacy pools, and 8141. Those discussions are interesting, but they make the ERC feel like it is trying to win too many adjacent debates at once.

## Risks or blind spots I think deserve more emphasis

- Reputation and identity transfer are double-edged. The draft frames address continuity as a feature, and it is. But the same property enables sale or reassignment of reputational history, allowlists, governance influence, and KYC-adjacent standing in ways some protocols may explicitly not want.
- "Clean transfer" is still hard. Even with approval revocation, a buyer may inherit protocol-specific approvals, operator state, app-level sessions, signed permits, or off-chain assumptions that the account itself cannot enumerate.
- The system is safer for institutional and organizational use than for naive retail use. Sophisticated users will understand "controller deed" and "custody account". Many normal users will not.
- The design is very good at transferring full control, but weaker for partial transfer, temporary transfer, or limited-purpose delegation. Those use cases will appear quickly if the standard succeeds.

## Usage scenarios I find especially compelling

- Treasury handoff without migration. A DAO, fund, or company can replace operators while preserving the same treasury address, positions, whitelists, and historical footprint.
- Spin-outs, divestitures, and sub-entity sales. A parent account owning child controller NFTs is basically an on-chain corporate structure primitive.
- Strategy compartments for agents. A parent can own multiple child accounts, each with limited balances and separate approval surfaces, then delegate operation of each child to different automation stacks.
- Inheritance and executorship. This is one of the rare designs where "transfer the estate" actually means "transfer the live address and its positions", not "rebuild everything from keys or seed phrases".
- Escrowed control handoff. The controller NFT can be held in escrow, a court-ordered process, a vesting contract, or a settlement workflow while assets remain parked at the account address.
- Portable regulated control. An entity could move root control between different owner models, such as an EOA, Safe, custodian contract, or future scheme-agile account, without changing the account address that counterparties already know.

## Additional things this enables that may be underexplored

- Whole-account collateralization. The deed itself can become the pledged object in secured lending, margin, or structured finance arrangements. That is not just "NFT collateral"; it is collateral over an address and everything it controls.
- On-chain M&A primitives. Transfer of a subtree of controller NFTs can represent acquisition of an operating unit, not just movement of assets.
- Temporary operational leasing. With the right wrapper or escrow design, control could be rented or conditionally delegated for a period without changing custody location.
- Address continuity as a product surface. Games, SaaS-like on-chain apps, membership systems, and B2B integrations often care about a stable address more than about the human behind it. This standard makes "transfer the operating account, keep the endpoint" much more natural.
- Capability decomposition through child accounts. Even if the root deed is all-or-nothing, the ecosystem can use child deed accounts as transferable capability containers: one for trading, one for governance, one for payroll, one for IP, one for subscriptions, and so on.

## Changes I would make to increase optionality

- Split the standard into a minimal core and a set of extensions. Core should be: canonical controller NFT, `tokenId <-> account` mapping, root control = `ownerOf`, transfer-aware `controlVersion`, direct execution, and deterministic deployment discovery.
- Move transfer lock/timelock, validator model, recovery, approval revocation, metadata, configured deployment, hierarchy policy, and 1271 signature profile into extensions or named profiles.
- Make transferability policy a profile, not a universal rule. Different deployments will want delayed transfer, immediate transfer, or non-transferable/governance-gated transfer.
- Standardize capability discovery so wallets can see which transfer and authorization profile a controller token uses.
- Relax the signature prescription. Require the security properties, especially account binding, token binding, control-version binding, and replay resistance, without forcing one canonical 1271 wrapping flow too early.
- Treat hierarchy depth as implementation policy. Keep cycle prevention mandatory, but avoid making one bounded-depth traversal philosophy feel canonical for every deployment.
- Define a stronger transfer-inspection surface so buyers and marketplaces can inspect active validators, unlock state, recovery presence, and perhaps an implementation-defined configuration hash or state digest.
- Add a clearer story for partial authority. The current design is excellent for whole-account transfer and acceptable for delegated signing, but a successful ecosystem will quickly want standardized limited-purpose authority models around the root deed.
- Move product-style features such as avatar/name metadata out of the base ERC and into a separate optional extension.

## Overall assessment

I like this ERC more than most wallet proposals because it is built around a tangible primitive with real consequences: the right to operate an address becomes a transferable object. That is novel, legible, and valuable.

If I were editing it for adoption, I would protect that insight by making the base standard smaller. Right now the document reads like a strong core proposal surrounded by a full policy stack. The core should be standardized aggressively. The policy stack should mostly be optional.

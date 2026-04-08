# Codex Adversarial Review: ERC-XXXX "Wallet Title Deeds"

**Reviewer:** Codex (adversarial mode)
**Date:** 2026-04-08
**Target:** branch diff against master
**Verdict:** needs-attention

---

No-ship: the draft still opens a recipient-scoped backdoor in configured deployment, markets account transfer as safe despite non-enumerable surviving approvals, and makes a privacy-pool claim the document later disproves.

---

## Findings

### [HIGH] Configured deployment lets an untrusted sponsor seed recipient-controlled backdoors

**Location:** ERCS/erc-XXXX.md lines 256-260

`deployAccountConfigured` mints the controller token to `initialOwner` and then lets the factory execute arbitrary `initCalls` under that recipient control version, but this section never requires any authorization from `initialOwner`. A relayer, sponsor, or compromised factory can therefore install validators, approvals, or recovery state that persist as if the recipient configured them. That is a real trust-boundary break for the onboarding path, not just a UX issue.

**Recommendation:** Require `initCalls` to be authorized by `initialOwner` (for example a signature over the predicted account, salt, chain, and exact calldata), or remove recipient-scoped factory execution and require post-deployment user acceptance before those settings become active.

---

### [HIGH] The spec sells NFT transfer as full account handoff even though hidden approvals survive and cannot be audited on-chain

**Location:** ERCS/erc-XXXX.md lines 979-981

The standard repeatedly positions NFT transfer as a first-class account sale / inheritance / handoff mechanism, but this section admits that token-level approvals survive transfer and cannot be enumerated on-chain. That means a seller can leave drain rights on obscure external contracts, and the buyer has no deterministic way to prove the account is clean before taking control. The revocation interface only helps for approvals the new controller already knows about, so the advertised handoff is not actually safe under adversarial conditions.

**Recommendation:** Either narrow the standard's transfer/sale claims substantially, or add a required approval-accounting model that makes outstanding third-party rights discoverable and revocable before the handoff is treated as safe.

---

### [MEDIUM] The privacy-pool section claims configured deployment avoids ownership disclosure, but the document later says ownership and account linkage are public

**Location:** ERCS/erc-XXXX.md lines 479-483

This section says `deployAccountConfigured` satisfies the 'no on-chain controller-token ownership disclosure' requirement because deployment, minting, and the batch are atomic. Later privacy text explicitly says controller-token ownership, account address, and ownership metadata remain public. Atomicity does not fix that: the same transaction still publicly links the withdrawal flow to the minted controller token and account. Integrators following this guidance will overestimate privacy and can deanonymize users.

**Recommendation:** Remove the claim that configured deployment satisfies the disclosure requirement, or specify a different flow that actually avoids linking the withdrawal transaction to the recipient's public controller-token ownership.

---

## Next Steps

1. Require explicit `initialOwner` authorization for any configured deployment path.
2. Rework the transfer/handoff story so it does not rely on undiscoverable surviving approvals.
3. Rewrite the privacy-pool guidance to match the document's own public-ownership model.

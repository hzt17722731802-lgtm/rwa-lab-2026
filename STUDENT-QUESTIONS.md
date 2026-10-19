# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:
>
> Before introducing redemption-queue liabilities, a simplified equivalent would be:
>
> `vault.reserveBalance() + custodian.realHoldings() == vault.totalClaimValue()`
>
> The left side combines the vault’s on-chain USDC balance with the reported value of Treasury holdings. The right side values all outstanding shares at the reported NAV. Both sides use USDC’s six-decimal units; comparing directly with `totalSupply()` would mix asset values with eighteen-decimal share quantities.
>
> This equality expresses accounting consistency: recorded assets match the reported value of outstanding claims. It is a proposed reconciliation condition, not an invariant guaranteed by this implementation. Updating the NAV or the custody record independently can break it.
>
> Even when the equality holds, it does not prove that the off-chain Treasury bills exist or are available for redemption. The vault’s USDC balance is directly observable on-chain, whereas `realHoldings()` is a number that an authorized account can change. The NAV also depends on an authorized reporter. Consistent reported numbers could therefore conceal missing assets or an incorrect valuation.
>
> Compared with Lab 1’s direct comparison of on-chain collateral and token supply, this system introduces external evidence requirements. Custodian statements and independent reconciliation are needed to substantiate the reported holdings. Once redemption requests are introduced, reconciliation must also distinguish outstanding shares, locked redemption obligations, and settled cash awaiting collection, without double-counting them.
<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:
>
> A legal structure is part of the technical design because a smart contract can record and transfer tokens, but it cannot itself establish ownership of Treasury bills held outside the blockchain or compel their delivery. The SPV, custody agreements, and token terms connect the on-chain record to enforceable rights over those assets.
>
> The SPV is intended to separate the fund’s assets from the issuer’s operating business, reducing the risk that the issuer’s creditors can use those assets to satisfy unrelated debts. However, bankruptcy remoteness from the issuer does not automatically protect against every failure of the custodian.
>
> If the custodian becomes insolvent, the outcome depends on the applicable law, custody arrangements, and whether the securities are properly segregated and identifiable as assets held for the SPV. With effective segregation, the SPV or its representative should be able to seek recovery or transfer of the securities, although access and redemptions may be delayed. Missing assets or defective arrangements could instead create losses or disputed creditor claims.
>
> The token holder retains whatever fund interest or contractual claim the product documents establish, potentially including a proportional entitlement to recovered assets or proceeds through the fund. Holding the token does not automatically grant direct ownership of a particular Treasury bill, immediate USDC redemption, or guaranteed full repayment. The lab’s Solidity contracts alone cannot determine those legal entitlements.
>


<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
>
> `REPORTER_ROLE` authenticates who may publish the NAV, but it does not establish whether the published value is correct. A single trusted reporter creates several distinct failure modes.
>
> First, a stolen signing key lets an attacker submit false prices. I would mitigate this through a threshold multisignature account controlled by independent signers, protected signing keys, and a separate process for promptly revoking compromised access. Multisignature control reduces dependence on one key but does not eliminate collusion.
>
> Second, the reporter may deliberately misstate the NAV for personal gain. Independent reconciliation against custodian records and valuation sources would make manipulation easier to detect. Clear reporting duties and enforceable penalties would provide additional accountability.
>
> Third, an honest reporter may make a calculation, decimal, or data-entry error. Explicit unit validation, independent calculation checks, and review of unusually large changes could reduce this risk. An exceptional but genuine market movement should trigger investigation rather than an automatic assumption that the old price remains correct.
>
> Fourth, an outage or delayed update can leave an outdated NAV active. The contract should enforce a maximum report age, reject stale prices for new subscriptions and redemption requests, and provide a controlled backup reporting process.
>
> These controls address different risks. Neither authorized signatures nor passing queue invariants independently proves that the reported NAV matches the underlying assets.

<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
>
> A stale-but-honest NAV was accurate when published but no longer reflects the current asset value. An informed investor can exploit the difference because the contract continues using the old number to price transactions.
>
> If the true NAV rises from 1.00 to 1.02 while the on-chain NAV remains 1.00, an eligible investor can subscribe 1,000 USDC and receive 1,000 shares. At the current value, that payment should purchase approximately 980.39 shares. Issuing the extra shares transfers value from existing holders to the new subscriber. The eventual post-subscription NAV must reflect this dilution; the gain is not newly created fund wealth.
>
> Conversely, if the true NAV falls to 0.98 while the reported NAV remains 1.00, an existing holder can enqueue 1,000 shares and lock a 1,000 USDC payout despite their current asset value being only 980 USDC. The lab locks the amount at enqueue time, so a later NAV correction does not reduce that ticket. Actual payment still depends on settlement liquidity.
>
> No reporter key needs to be stolen: the advantage comes from better information and transaction timing. I would check `updatedAt` against a defined maximum age before accepting new subscriptions or redemption requests. An expired report should trigger a fresh valuation or controlled suspension, rather than continued trading at the stale price. Chainlink likewise recommends checking timestamps to determine whether an answer is recent enough for the application.


<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
>
> The queue fixes the redemption amount when shares are enqueued. If a holder submits 1,000 shares at a NAV of 1.00, the ticket promises 1,000 USDC. A subsequent increase to 1.25 does not increase that payment. The redeemer therefore gives up the appreciation available from continuing to hold the shares.
>
> Assuming the higher NAV reflects genuine appreciation in assets still held by the fund, the remaining holders benefit economically: the redemption liability stays fixed while the assets supporting the fund increase in value. Conversely, if asset values fall, remaining holders absorb the decline against an unchanged redemption obligation. The redeemer still faces payment delays or losses if the fund cannot meet that obligation.
>
> This allocation can be reasonable when the pricing cutoff is clearly disclosed, the NAV is current, and settlement is prompt. It gives the redeemer price certainty. However, long delays and stale prices can unfairly transfer risk, while asset-sale costs may burden investors who remain.
>
> I would use a published forward-pricing cutoff for each redemption batch, with a fresh valuation and disclosed adjustments for attributable liquidation costs. Requests would remain exposed to valuation changes until that cutoff, after which their payment amounts would be fixed. This would reduce timing advantages and make risk allocation clearer. Existing tickets should not be retrospectively repriced simply because settlement becomes inconvenient.
>
> The lab does not automatically reconcile NAV with actual assets, so raising the reported NAV alone does not establish that any economic gain occurred.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
>
> In September 2008, the Reserve Primary Fund’s NAV fell to $0.97 following losses on Lehman-related holdings. Redemptions and payments were suspended, and investors subsequently received distributions through liquidation rather than ordinary immediate redemption at $1.&#x20;
>
> In March 2023, USDC traded below $1 amid uncertainty over reserves at Silicon Valley Bank. Issuer redemption was constrained by banking access and operating hours, while secondary-market trading continued. Circle maintained its 1:1 redemption commitment and, after banking access resumed, reported clearing substantially all minting and redemption backlogs by March 15. This was a disruption followed by restored processing, rather than liquidation of the product.&#x20;
>
> For a tokenized T-Bill fund, closing the redemption channel should specify whether new requests, settlement, or withdrawals are suspended. Existing tickets, locked amounts, and ordering should remain recorded. During a temporary liquidity shortage, I would preserve FIFO processing, communicate delays, and resume settlement as cash arrives. Actual insolvency requires a legally authorized loss-allocation process, not preferential payments to selected users.
>
> The lab’s token-level `pause()` blocks transfers, minting, and burning. Consequently, it blocks subscriptions, redemption enqueueing, and settlement that burns shares. However, the implemented `claim()` only transfers USDC, so already-settled claims are not directly blocked by the token pause.
>
> This broad switch unnecessarily restricts share transfers during a liquidity squeeze. I would separate controls for subscriptions, redemption requests, and settlement, while normally keeping already-funded claims withdrawable. Emergency controls should have defined authority, public explanations, and review deadlines. Pausing operations buys time; it does not create liquidity or repair missing assets.


<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
>
> I would draw the admission boundary around documented investor eligibility requirements, rather than the issuer’s unrestricted discretion. Checking both transfer endpoints makes those requirements effective after issuance: an approved investor cannot bypass admission controls by transferring shares to an unapproved address.
>
> However, an address missing from the whitelist is not necessarily malicious. The owner may simply be awaiting verification or correcting an administrative error. The fund should therefore publish its eligibility criteria, provide a reasonable onboarding process, explain rejection decisions where permitted, and offer a review mechanism. Collecting identity information should not require publishing sensitive personal documents on-chain.
>
> Whitelist administration should belong to an accountable compliance function with narrowly scoped authority. Routine approvals could follow a documented verification workflow, while exceptional removals or changes to eligibility policy should require additional independent approval. Reporting NAV and administering admission should be separate responsibilities, so compromising one operational role does not automatically compromise the other.
>
> Changes should produce auditable records linking the affected address, decision, and responsible approver, while protecting confidential information. Time-sensitive restrictions may need an emergency path, but that path should require prompt review and a clear process for correcting mistakes.
>
> In this lab, the whitelist is only an on-chain representation of an off-chain decision. `isWhitelisted()` proves that the registry currently permits an address; it does not independently prove the holder’s identity or the quality of the verification process.

<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
>
> There are legitimate reasons to give a regulated fund limited intervention powers. Freezing an address may help contain a reported theft during investigation. An authorized cancellation or seizure process may also be needed to implement a valid order or correct a verified issuance error. Without intervention mechanisms, the on-chain record could become inconsistent with the rights the product represents.
>
> However, these powers conflict with decentralization’s goals of censorship resistance and minimizing trust in central intermediaries. Users can control their private keys yet still depend on an issuer’s permission to transfer or retain their shares. A compromised key, mistaken assessment, or abusive operator could freeze or destroy a balance. This does not make the blockchain itself centralized, but it makes control over this asset substantially centralized. Burning also does not automatically transfer the underlying assets to a rightful recipient.
>
> Ex4 demonstrates that the lab’s controls collide: removing Alice from the whitelist blocks both transfers and an administrator’s attempt to burn her shares, because both pass through `_update`. Temporarily re-whitelisting her would reopen ordinary transfer eligibility during that interval.
>
> I would separate routine compliance administration, emergency freezing, and exceptional seizure authority. Compliance staff would manage routine whitelist decisions; emergency freezes would require prompt independent review; seizure would require a distinct multisignature approval involving independent compliance and legal reviewers. Multisignature approval reduces unilateral control but does not eliminate centralized authority or collusion.
>
> A dedicated seizure operation should preserve the freeze on ordinary transfers while enforcing narrowly defined authorization checks. It should record the reason, amount, and corresponding asset treatment, with notice and review where appropriate. These safeguards acknowledge the centralization trade-off rather than pretending that tokenization removes it.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a | `realHoldings()` moves with no cash moving | `test_Ex2_RealHoldingsIsJustANumber` |
| b | The claim value floats while the share count stays | `test_Ex3_TheClaimFloats_TheShareCountDoesNot` |
| b | One `attest` re-prices every holder at once | `test_Ex3_OneCallMovesTheWholeBook` |
| d | A non-whitelisted address cannot receive shares, even by paying | `test_Ex4_NoKyc_NoShares_EvenIfYouPay` |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints` |
| d | Removing an address freezes that holder — and blocks a burn | `test_Ex4_FreezeBeatsConfiscation` |





Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:

I consider stale-NAV subscription a plausible exploitation scenario. Suppose the underlying assets have increased in value, but the reporter has not updated the on-chain NAV. A whitelisted investor who knows about the increase can subscribe at the outdated, lower price and receive more shares than their payment would justify at the current asset value. When the NAV is updated, those shares are valued at the higher price, diluting existing holders.

This breaks plank (b), attestation: the reported value may have been honest when published, yet no longer be suitable for pricing new transactions. The lab does not reject stale prices. A mitigation would be to enforce a maximum age for the NAV and suspend price-sensitive operations when the report expires, with a defined process for obtaining a fresh valuation.

This is a proposed additional scenario, not an additional test implemented in this submission.
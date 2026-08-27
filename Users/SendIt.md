# SendIt (Branch Agent) — Intermediary

## Role
SendIt itself, operating at the counter of a physical branch. Serves the sender on the
origin side (payment capture and KYC) and the recipient on the destination side (identity
validation and cash payout). Not a third party: it is the platform itself acting as the
end-to-end intermediary.

## A day in their work
- A sender walks up to the counter asking to send money abroad. The agent runs KYC,
  records the recipient's details, quotes the exchange rate and fee, and takes payment in
  cash or by card.
- The remittance becomes ready for collection, and the sender is given a tracking code to
  share with the recipient.
- Hours or days later, at another branch (possibly in another country), a recipient shows
  up with the tracking code to collect cash. The agent validates their identity against
  the data on the remittance before releasing the funds.
- If the channel chosen was bank deposit, the agent doesn't get involved in payout: the
  system validates the recipient's identity against the KYC provider and sends the
  instruction to the receiving bank.
- If a remittance sits unclaimed past the collection window, it expires and SendIt keeps
  the funds — nothing is paid out and nothing is refunded to the sender.

## Needs
- Needs to receive the sender's money and, later, hand it to the correct recipient.
- Needs to validate the recipient's identity against the amount and details the sender
  registered, before releasing any cash.
- Needs to validate that the bank account (when the channel is deposit) matches the
  recipient's declared details.
- Needs to be able to tell the sender immediately if something fails when creating the
  remittance (KYC rejected, payment declined, inconsistent recipient data).
- Needs a way to record/justify the source of funds the sender brings in, when the amount
  or sending pattern warrants it.
- **Needs a defined number of days (30) for the recipient to collect a remittance.** If
  it is not collected within that window, SendIt keeps the funds instead of holding them
  indefinitely or refunding the sender — this protects SendIt from carrying unclaimed
  liabilities forever.

## Current pains
- "If the system doesn't warn me right away that KYC failed, I've already taken the money
  and now I don't know what to do with it." ← payment capture and verification must be
  resolved before taking the money, not after.
- "I could hand cash to the wrong person if the system doesn't force me to validate
  identity against the exact data on the remittance." ← without that check, the branch
  counter is the weakest link in the whole flow's security.
- "If two agents at two different branches both try to pay out the same remittance, they
  can't both succeed." ← payout has to be exclusive and duplicate-proof across branches.
- "If nobody ever comes to collect, that money can't just sit open forever." ← there has
  to be a hard cutoff after which the case closes and the funds are SendIt's.

## What the system gives them
- A remittance creation flow that never lets an agent move to payment capture without the
  sender's KYC approved.
- A payout screen (destination side) that requires and records recipient identity
  validation before enabling cash release.
- Automatic locking of an already-paid remittance so it can never be paid out twice, even
  from another branch.
- Real-time visibility of the remittance status, so nothing is charged or paid against
  stale data.
- An automatic cutoff after the collection window: the remittance moves to `Expired` and
  the funds are retained by SendIt, closing the case without manual follow-up.

## Associated acceptance criteria
To be defined in `Eval-Spec.md` §11 (this persona's block).

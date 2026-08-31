# SendIt Branch Agent (Intermediary)

## Role
SendIt itself, working at the counter of a physical branch. It serves the sender on the
origin side (payment and KYC check) and the recipient on the destination side (KYC check
and cash payout). It is not a third party: it is the platform acting as the intermediary
for the whole trip. Remote transfers from the app or website are self-service and do not
involve an agent, but the branch agent handles every in-person transfer and every cash
payout, and can cancel a remittance on the sender's behalf.

## A day in their work
- A sender comes to the counter to send money abroad. The agent runs a KYC check on the
  sender's ID, records the recipient's details, shows the exchange rate and fee, and takes
  cash or a card POS payment.
- The system sets the payout amount aside in the destination-country reserve. Only then
  does the remittance become ready for collection, and SendIt sends the tracking code
  straight to the recipient.
- Hours or days later, at another branch (maybe in another country), a recipient arrives
  with the tracking code to collect cash. The agent KYC-checks their ID against the data on
  the remittance before releasing the money.
- If the channel was bank deposit, the agent is not involved in the payout: the system
  KYC-checks the recipient with the provider and sends the instruction to the receiving
  bank.
- If a remittance stays unclaimed past 30 days, it expires and SendIt keeps the funds. If
  the sender asks to cancel before payout, the agent cancels it and the sender is refunded
  in full.

## Needs
- To take the sender's money and later hand it to the correct recipient.
- To KYC-check the recipient against the details the sender registered, before releasing
  any cash.
- To check that the bank account (for the deposit channel) matches the recipient's
  registered details.
- To tell the sender right away if something fails while creating the remittance (KYC check
  rejected, payment declined, User Risk flag, recipient data does not match).
- To have a fixed number of days (30) for the recipient to collect. After that, SendIt
  keeps the funds instead of holding them forever or refunding the sender.
- To bound how much any single customer can move in a day, so SendIt's exposure to one
  customer is capped (US$ 50,000 per rolling 24 hours, across all channels).
- To have the payout amount set aside in the destination country before the recipient is
  told the money is ready.

## Current pains
- "If the system doesn't tell me right away that the KYC check failed, I've already taken
  the money and I don't know what to do with it." Payment and verification must be settled
  before the money is taken, not after.
- "I could hand cash to the wrong person if the system doesn't make me match the ID against
  the exact data on the remittance." Without that check, the counter is the weakest point
  in the flow.
- "If two agents at two branches try to pay out the same remittance, both can't succeed."
  Payout has to be exclusive and duplicate-proof across branches.
- "If nobody comes to collect, that money can't stay open forever." There has to be a hard
  cutoff after which the case closes and the funds are SendIt's.
- "If there's no money set aside in the destination country, the recipient turns up and I
  can't pay them." Readiness has to mean the money is already reserved.

## What the system gives them
- A create flow that never lets the agent move to payment without the sender's KYC check
  approved, and that holds a remittance and hands the case to a Security service if the KYC
  provider raises a User Risk flag.
- A payout screen that requires and records the recipient's KYC check before cash can be
  released.
- Automatic locking of an already-paid remittance so it can never be paid twice, even from
  another branch.
- A reservation against the destination reserve before a remittance is made ready, so
  "ready for pickup" always means the money exists in the destination country.
- Real-time status, so nothing is charged or paid against stale data.
- An automatic cutoff after 30 days: the remittance moves to `Expired`, the reservation is
  released, and the funds stay with SendIt, with no manual follow-up.
- A cancel path before payout that releases the reservation and refunds the sender in full.

## Associated requirements
`Eval-Spec.md` §11: R6, R7, R8, R9, R10, R12, R13, R16, R18, R19, R21, R22, R23. Coverage
for this persona is scored in `Eval-Results.md`.

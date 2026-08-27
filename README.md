# SendIt (Easy Peazy) — Domain Context

> This document is raw material, not the deliverable. The deliverable is `Eval-Spec.md`.
> Case study: "Design a system that lets people send remittances to people in different
> countries. Security matters a lot, and so does data consistency, since we're talking
> about money." (Design duration: 4h)

---

## 1. Context

SendIt is an international remittance platform, modeled after a physical remittance
storefront (Western Union is the reference model): it moves money from a person in one
country (the **Sender**) to another person in a different country (the **Recipient**).
SendIt itself acts as the end-to-end intermediary — it does not delegate payout to an
external partner. The money flows like this:

1. The **Sender** shows up **in person** at a SendIt branch and hands over the money in
   cash or by card (never remotely/electronically), in their home country's currency.
2. **SendIt** holds those funds, charges a **fee**, and applies an **exchange rate** to
   convert the amount into the destination currency.
3. SendIt releases the converted amount to the **Recipient** through one of two channels:
   **cash pickup at a SendIt branch** in the destination country, or **deposit into a
   bank account** held by the recipient.
4. Before releasing the money, SendIt validates the **recipient's identity** (at the
   counter for cash, or against the KYC provider for bank deposit) — this is where the
   security of the whole system is decided.

SendIt's money never "floats" without an owner: at every moment it is either in the
sender's custody (not yet handed over), in SendIt's custody (collected, pending payout),
or in the recipient's custody (paid out). There is no fourth state.

---

## 2. Purpose of the system

SendIt exists so a person can send money to a family member or acquaintance in another
country without depending on the traditional banking network (which often doesn't
connect banks across countries) and without going through a long chain of intermediaries
that make the operation slower and more expensive. The system supports **corridors
between different countries** (not a single fixed currency pair): each corridor has its
own currency pair, exchange rate, and collection rules. SendIt quotes the exchange rate
and fee upfront, takes payment from the sender at the counter, verifies the identity of
both parties, and releases the money to the recipient through whichever channel they
choose (cash pickup at a branch, or bank deposit), leaving an auditable trail of every
dollar/sol that comes in and goes out.

The design treats **security** (nobody collects a remittance that isn't theirs, no
remittance is created without verified identity) and **data consistency** (a collected
amount always corresponds to a payable amount, a remittance is never paid out twice, a
transaction's record is never lost) as first-class requirements, on the same level as the
sending functionality itself.

---

## 3. Scope

### In scope
- Creating a remittance at a SendIt branch: amount, destination country/currency, and
  the payout channel chosen for the recipient (cash pickup at a branch, or bank deposit).
- Quoting the exchange rate and fee before confirming, for any supported corridor between
  countries.
- **Full, mandatory** identity verification (KYC) of the sender, for any amount, at every
  branch visit.
- Identity verification of the recipient at the moment of payout (at the counter for
  cash; against the KYC provider for bank deposit).
- Capturing payment from the sender at the branch: cash or card (in-person payment via a
  POS terminal — never remote).
- Settling/paying out to the recipient through the two supported channels: cash pickup at
  a SendIt branch, or bank deposit.
- Tracking the remittance's status for both parties (sender and recipient).
- A receipt/proof of the transaction.
- **A collection window for the recipient**: if the remittance isn't collected within
  **30 days** of becoming ready for pickup, it expires and SendIt keeps the funds — no
  refund to the sender, no payout to the recipient.
- **Status lookup by tracking code, with no account required**: the recipient (or the
  sender) can check a remittance's status at any time using only the tracking code —
  creating an account is never a precondition for finding out where the money is.
- Handling other exceptions: a remittance rejected for failed KYC or failed payment.

### Out of scope (with reasons)
| Excluded | Reason |
|---|---|
| Remote/electronic payment by the sender (app, saved card, online transfer) | The case restricts sender payment to cash or in-person card at the branch |
| Recipient payout by card or digital wallet | The case restricts payout to exactly two channels: cash pickup at a branch and bank deposit |
| Compliance screening (sanctions/PEP lists, AML) | Out of scope for this design exercise; identity KYC is assumed to be the only admission control |
| Card payment processing (authorization, PCI) | Handled by the branch's external POS terminal/processor; SendIt is its client, not its implementer |
| Determining market exchange rates | Provided by an external FX provider; SendIt consumes the quote, it doesn't compute it |
| Biometric/document identity verification | Performed by an external KYC provider; SendIt only consumes the verdict (approved/rejected) |
| SendIt's internal accounting/tax management as a company | Corporate back-office, not part of the remittance domain |
| Banking for the sender or recipient (account opening, own cards) | SendIt is a sending intermediary, not a bank; it doesn't hold savings |
| Post-payout disputes/reversals | Out of scope for now; a paid remittance is final in this exercise |

---

## 4. Actors

### External users
| Actor | Role | Motivation |
|---|---|---|
| Sender | Person who initiates the money transfer, in person at a branch | See [Users/Aaron.md](Users/Aaron.md) |
| Recipient | Person who receives the money, in person (cash) or via deposit | See [Users/Juan.md](Users/Juan.md) |

### Internal users
| Actor | Role |
|---|---|
| SendIt Branch Agent | Person at the branch counter who serves the sender (takes payment, runs KYC, creates the remittance) and the recipient (validates identity, hands over cash). This is SendIt itself acting as intermediary — there is no delegated external third party. See [Users/SendIt.md](Users/SendIt.md) |

> No additional internal personas (compliance, support, treasury) are modeled in this
> exercise: all internal operations fall on the Branch Agent.

### Integrations (non-human actors)
| Integration | What it does for SendIt | What happens if it fails |
|---|---|---|
| Payment processor (POS terminal, in-person) | Captures the card payment when the sender pays at the branch | The remittance can't move from `Created` to `Collected` via that route; the agent offers cash as an alternative |
| FX provider | Supplies the current exchange rate for the currency corridor | Can't be quoted; the remittance can't be created |
| KYC / identity verification provider | Validates the sender's identity (always) and the recipient's (when the payout channel is bank deposit) | The remittance can't be created (sender) or the deposit can't be released (recipient) until resolved |
| Receiving bank (deposit rails, ACH/local transfer) | Credits the amount to the recipient's bank account when the chosen channel is deposit | The deposit fails and the remittance goes back to `Ready for pickup`, awaiting retry or a channel change |
| Notification service (SMS/email/push) | Notifies sender and recipient of every status change | The remittance still proceeds; the user has to check status at a branch |

---

## 5. Domain entities

- **Sender**: person who initiates the transfer. Attributes: verified identity (KYC,
  always mandatory), country, sending history.
- **Recipient**: person who receives the money. Attributes: verified identity (at the
  counter, or via remote KYC, depending on channel), country, chosen payout channel.
- **Remittance**: the core transaction. Attributes: origin amount, origin currency,
  origin country, destination amount, destination currency, destination country,
  exchange rate applied, fee, sender's payment method (cash/card), recipient's payout
  channel (cash/bank deposit), status, tracking code, creation date, collection deadline,
  settlement date.
  - States: `Created` → `Collected` → `Ready for pickup` → `Paid` | `Rejected` |
    `Expired`. (Post-payout reversals/disputes are out of scope for now.)
  - Status is queryable by tracking code alone, by anyone who has the code — no sender
    or recipient account is required to look it up.
- **Corridor**: the pair of countries/currencies SendIt allows operating between (e.g.
  Peru → United States, PEN → USD). Defines which exchange rate and rules apply.
- **Quote**: the exchange rate and fee locked in for a remittance at creation time, valid
  for a limited time before it expires and must be recalculated.
- **Identity verification (KYC)**: the record of validating a sender or recipient against
  the external provider (or against a physical document at the counter). Attributes:
  subject, result, date, verification channel.
- **Payout order**: the concrete instruction to deliver funds to the recipient — either
  the internal cash disbursement order at a branch, or the instruction sent to the
  receiving bank.
- **Receipt**: the final proof of a `Paid` remittance, with all amounts and references,
  given to both sender and recipient.

---

## 6. Processes

1. **Sender verification (KYC)** — on every branch visit, before creating a remittance,
   the agent validates the sender's identity. There is no exempt amount: every
   remittance requires full sender KYC.
2. **Quoting** — the sender states the amount, destination country/currency, and the
   payout channel they want for the recipient; the system queries the FX provider for
   the matching corridor and computes the fee, locking in the quote for a limited time.
3. **Remittance creation** — the sender confirms the recipient's details and accepts the
   quote; the remittance is created in `Created` status.
4. **Collecting payment from the sender** — the agent captures payment at the counter
   (cash or card via POS). On success, the remittance moves to `Collected`.
5. **Making it available for pickup** — once collected, the remittance moves to `Ready
   for pickup` and a tracking code is generated for the recipient. The **collection
   window** (a fixed number of days) starts counting from this point.
6. **Notifying the recipient** — SendIt informs them that a remittance is available and
   gives them the tracking code needed to collect it. From this point on, either party
   can look up the remittance's status with just the tracking code — no account or
   registration required.
7. **Recipient identity verification and settlement**:
   - If the channel is **cash**: the recipient shows up at a SendIt branch, and the agent
     validates their identity at the counter against the name registered on the
     remittance, then hands over the cash.
   - If the channel is **bank deposit**: SendIt validates the recipient's identity
     against the KYC provider and, on success, sends the crediting instruction to the
     receiving bank.
   - In both cases, once delivery is confirmed the remittance moves to `Paid` and a
     **Receipt** is issued.
8. **Expiration** — if a `Ready for pickup` remittance is not collected within the
   collection window, it automatically moves to `Expired`: SendIt retains the funds, and
   neither the sender is refunded nor the recipient is paid.
9. **Other exceptions** — a remittance whose sender KYC or payment fails never reaches
   `Collected` and is marked `Rejected` instead (money was never actually taken, so
   there's nothing to refund).

---

## 7. Business rules

- No remittance can be created without the sender having completed full KYC — there is
  no amount below which this check can be skipped.
- A remittance cannot move to `Collected` without a successful payment capture (cash
  counted at the till, or POS authorization): the internal state is never advanced ahead
  of the money actually existing at SendIt.
- The recipient cannot collect (cash or deposit channel) without their identity having
  been validated for that specific remittance — identity is validated per remittance,
  not once for a lifetime.
- A remittance can only be paid out once: once payment is confirmed, the system must
  block any second payout order against the same tracking code (settlement idempotency).
- The exchange rate and fee charged are the ones locked in by the **Quote** at the moment
  the remittance was created, not whatever is current at payout time — a quote has an
  explicit validity window and expires.
- **A `Ready for pickup` remittance that is not collected within 30 days automatically
  moves to `Expired`, and SendIt retains the funds.** There is no refund to the sender in
  this case — this is distinct from `Rejected`, where the sender's money was never
  actually taken in the first place.
- Checking a remittance's status (by tracking code) never requires an account. Account-
  free lookup only reveals status, never enough to authorize a payout — collecting still
  requires identity verification per the rules above.
- Every movement of funds (collection, settlement, expiration) must be recorded
  immutably and be auditable: the amount collected from a sender and the amount paid out
  to a recipient (or retained on expiration) must be reconcilable at all times.
- A corridor (origin/destination country-currency pair) must exist and be active before
  a remittance can be quoted or created on it.

---

## 8. Glossary

| Business term | Meaning | Name in code/system |
|---|---|---|
| Remittance | The end-to-end money transfer | `Remittance` |
| Sender | Whoever initiates the transfer, in person | `Sender` |
| Recipient / Beneficiary | Whoever receives the money | `Recipient` / `Beneficiary` |
| Corridor | Country/currency pair enabled for operation | `Corridor` |
| Quote | Exchange rate + fee locked in for a remittance | `Quote` |
| Exchange rate | Conversion rate between origin and destination currency | `FxRate` |
| Fee | The charge SendIt takes for the service | `Fee` |
| Branch Agent | SendIt person staffing the counter (collects, verifies, pays out) | `BranchAgent` |
| Branch | SendIt's physical location where money is paid in and paid out | `Branch` |
| Tracking code | Unique identifier the recipient uses to collect | `TrackingCode` |
| Collection window | Number of days the recipient has to collect before it expires (**30 days**) | `CollectionWindow` |
| Payout order | Settlement instruction (internal cash or bank deposit) | `PayoutOrder` |
| Receipt | Final proof of a paid remittance | `Receipt` |

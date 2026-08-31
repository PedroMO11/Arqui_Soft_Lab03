# SendIt: Domain Context

## 1. Context

SendIt is an international remittance platform, modeled on a real remittance service. It
moves money from a person in one country (the
**Sender**) to another person in a different country (the **Recipient**). SendIt is the
intermediary for the whole trip; it does not hand payout off to an outside partner. The
money flows like this:

1. The **Sender** starts the transfer from the mobile app, from the website, or in person
   at a SendIt branch. Remote senders pay from a linked bank account or card. Branch
   senders pay in cash or by card on the branch POS. The amount is in the sender's home
   currency.
2. **SendIt** holds those funds, charges a **fee**, and applies an **exchange rate** to
   convert the amount into the destination currency.
3. Before a remittance is made ready for pickup, SendIt sets its payout amount aside in a
   **destination-country reserve**.
4. SendIt releases the converted amount to the **Recipient** through one of two channels:
   **cash pickup at a SendIt branch** in the destination country, or **deposit into the
   recipient's bank account**.
5. Before releasing the money, SendIt runs a **KYC check** on the recipient (at the counter
   for cash, or with the KYC provider for bank deposit). This step decides the security of
   the whole system.

SendIt's money is never without an owner. At every moment it is in the sender's hands (not
yet sent), in SendIt's hands (collected, reserved, waiting for payout), in the recipient's
hands (paid out), or back in the sender's hands (refunded after a cancellation). The
sender can cancel a remittance any time before it is paid.

---

## 2. Purpose of the system

SendIt exists so a person can send money to family or an acquaintance in another country
without depending on the normal banking network (which often does not connect banks across
countries, or charges a lot) and without going through a long chain of middlemen that make
the transfer slower and more expensive. The system supports **corridors between different
countries** (not one fixed currency pair): each corridor has its own currency pair,
exchange rate, and collection rules. SendIt quotes the exchange rate and fee upfront, takes
payment from the sender, runs a KYC check on both parties, sets the payout amount aside in
the destination country, and releases the money to the recipient through the channel they
chose (cash pickup at a branch, or bank deposit), leaving an auditable trail of every unit
of money that comes in, goes out, or is refunded.

The design treats **security** (nobody collects a remittance that is not theirs, no
remittance is created without a KYC check, a flagged sender or recipient is held and handed
to a Security service) and **data consistency** (a collected amount always matches a
payable amount, a remittance is never paid out twice, the destination reserve totals add
up, a transaction's record is never lost) as core requirements, not add-ons.

---

## 3. Scope

### In scope
- Creating a remittance from the app, the website, or a branch: amount, destination
  country and currency, and the payout channel for the recipient (cash pickup at a branch,
  or bank deposit).
- Quoting the exchange rate and fee before the sender confirms, for any supported corridor.
- **Full, mandatory** KYC check of the sender, at any amount:
  - Branch senders are KYC-checked in person on every visit.
  - Remote senders pass a KYC check and prove ownership of their funding source once, at
    registration. Each transfer after that needs a logged-in session, plus an extra KYC
    check for large amounts.
- KYC check of the recipient at payout (at the counter for cash; with the KYC provider for
  bank deposit).
- Acting on the KYC provider's **User Risk flag**: hold the remittance, notify the user,
  and hand the case to an external Security service.
- Capturing payment from the sender: cash or card POS at a branch, or a debit from a
  linked bank account or an online card payment when sending remotely. Card payments go
  through a PCI payment gateway.
- **Reserving the payout amount in the destination country** before the remittance is made
  ready for pickup, topping up the reserve from SendIt's own funds, then the corporate
  account, then money borrowed from an outside bank.
- Paying out to the recipient through the two supported channels: cash pickup at a branch,
  or bank deposit.
- **Cancelling a remittance before payout** and **refunding** the sender in full, via the
  original payment method.
- Tracking the remittance's status for both parties.
- A receipt for the transaction.
- **A collection window for the recipient**: if the remittance is not collected within
  **30 days** of becoming ready for pickup, it expires and SendIt keeps the funds. No
  refund to the sender, no payout to the recipient.
- **Status lookup by tracking code, with no account required**: the recipient or the
  sender can check a remittance's status at any time with only the tracking code.
- Handling exceptions: a remittance rejected for a failed KYC check, a failed payment, or a
  confirmed User Risk case.

### Out of scope (with reasons)
| Excluded | Reason |
|---|---|
| Recipient payout by card or digital wallet | The recipient has exactly two options: cash at a branch or bank deposit |
| Building our own card processor or storing full card numbers | Card payments (branch POS and online) go through a PCI payment gateway; SendIt connects as a client of the gateway |
| Sanctions / PEP list screening and full AML transaction monitoring | SendIt acts on the KYC provider's User Risk flag, but building its own list checks and pattern monitoring is out of the current scope. A production system would need it |
| Determining market exchange rates | An external FX provider gives the rate; SendIt only uses it |
| Running the KYC checks themselves (document and biometric matching) | An external KYC provider does this; SendIt only reads the result and the User Risk flag |
| Sourcing physical cash into a branch till | The destination reserve is modelled as balances, not as cash-in-transit logistics |
| SendIt's own company accounting and taxes | Corporate back-office, not part of the remittance domain |
| Opening bank accounts or issuing cards for senders or recipients | SendIt moves money, it is not a bank |
| Reversal or dispute of an already-`Paid` remittance | A `Paid` remittance is final; cancellation and refund before payout are in scope |

---

## 4. Actors

### External users
| Actor | Role | Motivation |
|---|---|---|
| Sender | Person who starts the transfer, from the app, the website, or a branch | See [Users/Aaron.md](Users/Aaron.md) |
| Recipient | Person who receives the money, as cash at a branch or a bank deposit | See [Users/Juan.md](Users/Juan.md) |

### Internal users
| Actor | Role |
|---|---|
| SendIt Branch Agent | Person at the branch counter who serves the sender (takes payment, runs the KYC check, creates the remittance, cancels on the sender's behalf) and the recipient (KYC-checks, hands over cash). This is SendIt itself acting as intermediary; there is no outside third party. Remote transfers are self-service and do not involve an agent. See [Users/SendIt.md](Users/SendIt.md) |

> No other internal personas (compliance, support, treasury) are modeled here.
> All internal branch operations fall on the Branch Agent.

### Integrations (non-human actors)
| Integration | What it does for SendIt | What happens if it fails |
|---|---|---|
| Payment gateway (branch POS and online card) | Captures card payments, at the branch and on the app or website | The remittance cannot move from `Created` to `Collected` by card; the sender is offered another method |
| Bank debit rails | Debits a remote sender's linked bank account | The remittance cannot move to `Collected` by that method; the sender is offered another method |
| FX provider | Supplies the current exchange rate for the corridor | No quote can be made; the remittance cannot be created |
| KYC provider | Runs the KYC check on the sender (at registration for remote senders) and the recipient (for bank-deposit payout), and returns any User Risk flag | The sender cannot register or send, or the deposit cannot be released, until it is resolved |
| External Security service | Receives User Risk cases handed over by SendIt and returns a verdict (cleared or confirmed) | The held remittance stays held until a verdict arrives |
| Outside bank (borrowing) | SendIt borrows from it to top up a destination-country reserve when its own funds and corporate account do not cover a remittance. It funds the reserve; it never pays the recipient | The reservation cannot be placed; the remittance stays `Collected` until the reserve is funded |
| Receiving bank (deposit rails) | Credits the amount to the recipient's bank account for the deposit channel | The deposit fails and the remittance goes back to `Ready for pickup`, waiting for a retry or a channel change |
| Notification service (SMS, email, push) | Notifies sender and recipient of every status change | The remittance still proceeds; the user checks status by tracking code |

---

## 5. Domain entities

- **Sender**: person who starts the transfer. Attributes: KYC result (always mandatory),
  country, sending history, and (for remote senders) a sender account with one or more
  linked funding sources.
- **Recipient**: person who receives the money. Attributes: KYC result (at the counter, or
  with the KYC provider, depending on channel), country, chosen payout channel.
- **Remittance**: the core transaction. Attributes: origin amount, origin currency, origin
  country, destination amount, destination currency, destination country, exchange rate
  applied, fee, sending channel (app / web / branch), sender's payment method (cash / card
  POS / bank debit / online card), recipient's payout channel (cash / bank deposit),
  status, tracking code, creation date, collection deadline, settlement date.
  - States: `Created` -> `Collected` -> `Ready for pickup` -> `Paid` | `Rejected` |
    `Expired` | `Cancelled`. A `Paid` remittance is final; a later dispute is a new case.
  - Status can be looked up by tracking code alone, by anyone who has the code. No account
    is required.
- **Corridor**: the country and currency pair SendIt allows operating between (for example
  Peru -> United States, PEN -> USD). Defines which exchange rate and rules apply.
- **Quote**: the exchange rate and fee locked in for a remittance at creation time, valid
  for a limited time before it expires and must be recalculated.
- **KYC check**: the record of verifying a sender or recipient against the external KYC
  provider (or against a physical document at the counter). Attributes: subject, result,
  User Risk flag, date, method. Tied to one remittance, except a remote sender's
  registration-time check.
- **Destination reserve**: the pool of funds SendIt holds in a destination country to pay
  recipients. Attributes: country, currency, available amount, reserved amount.
- **Reservation**: the hold on a destination reserve for one remittance's payout amount.
  Attributes: remittance, reserve, amount, source (own funds / corporate / borrowed),
  state (held / consumed / released).
- **Payout order**: the concrete instruction to deliver funds to the recipient, either the
  internal cash disbursement order at a branch or the instruction sent to the receiving
  bank.
- **Refund**: the return of `total_charged_to_sender` to the sender via the original
  payment method, after a cancellation of a remittance that had money collected.
- **Receipt**: the final proof of a `Paid` remittance, with all amounts and references,
  given to both sender and recipient.

---

## 6. Processes

1. **Sender KYC check.** Branch: the agent KYC-checks the sender's ID on every visit,
   before creating a remittance. Remote: the sender passes a KYC check and proves
   funding-source ownership once at registration, and each later transfer needs a
   logged-in session, plus an extra KYC check for large amounts (the large-amount check).
   There is no exempt amount.
2. **Quoting.** The sender states the amount, destination country and currency, and the
   payout channel for the recipient. The system asks the FX provider for the corridor rate
   and computes the fee, then locks the quote for a limited time.
3. **Remittance creation.** The sender confirms the recipient's details and accepts the
   quote. The remittance is created in `Created` status. If the KYC provider returns a User
   Risk flag, the remittance is held and the case goes to the external Security service.
4. **Collecting payment from the sender.** Branch: the agent takes cash or a card POS
   payment. Remote: SendIt debits the linked bank account or charges the online card
   through the gateway. On success the remittance moves to `Collected`.
5. **Reserving and making it available.** The system reserves the payout amount against
   the destination-country reserve, topping the reserve up (own funds, then corporate
   account, then borrowed from an outside bank) if it is short. Once reserved, the remittance moves to
   `Ready for pickup`, a tracking code is generated, and the **collection window**
   (30 days) starts.
6. **Notifying the recipient.** SendIt tells them a remittance is available and gives them
   the tracking code. From here, either party can look up the status with just the tracking
   code, with no account.
7. **Recipient KYC check and settlement.**
   - Cash channel: the recipient goes to a SendIt branch and the agent KYC-checks their
     identity against the name on the remittance, then hands over the cash.
   - Bank deposit channel: SendIt KYC-checks the recipient with the KYC provider and, on
     success, sends the crediting instruction to the receiving bank.
   - In both cases, once delivery is confirmed the reservation is consumed, the remittance
     moves to `Paid`, and a **Receipt** is issued.
8. **Cancellation and refund.** While a remittance is `Created`, `Collected`, or `Ready
   for pickup`, the sender (in the app or website, or through an agent) can cancel it. Any
   reservation is released back to the reserve. If money had been collected, a **Refund**
   of `total_charged_to_sender` is issued to the original payment method. The remittance
   moves to `Cancelled`.
9. **Expiration.** If a `Ready for pickup` remittance is not collected within 30 days, it
   automatically moves to `Expired`, the reservation is released, SendIt retains the funds,
   and neither the sender is refunded nor the recipient is paid.
10. **User Risk hand-off.** A held remittance waits on the Security service's verdict.
    Cleared: it resumes. Confirmed: it moves to `Rejected`, with a refund if money had been
    collected.
11. **Other exceptions.** A remittance whose sender KYC check or payment fails never
    reaches `Collected` and is marked `Rejected` instead (money was never taken, so there
    is nothing to refund).

---

## 7. Business rules

- No remittance can be created without the sender having passed a KYC check for that
  transfer. There is no amount below which this is skipped. Branch: in person, every visit.
  Remote: a verified account and a valid session, plus an extra KYC check for large amounts
  (the large-amount check).
- A remittance cannot move to `Collected` without a successful payment (cash counted, POS
  authorized, bank debit cleared, or online card charged). The internal state is never
  moved ahead of the money actually being at SendIt.
- A remittance cannot move to `Ready for pickup` until its payout amount is reserved
  against the destination-country reserve. If the reserve is short it is topped up first,
  from SendIt's own funds, then the corporate account, then money borrowed from an outside
  bank, in that order.
- A reservation is consumed when the remittance is paid, and released back to the reserve
  when the remittance expires or is cancelled. The reserve's available, reserved, and
  used-up totals always add up.
- The recipient cannot collect (cash or deposit) without a KYC check for that specific
  remittance. It is done per remittance, not once for a lifetime.
- A remittance can be paid out only once. Once payment is confirmed, the system never lets
  a second payout run against the same tracking code.
- The exchange rate and fee charged are the ones locked into the **Quote** when the
  remittance was created, not whatever is current at payout time. A quote has an explicit
  validity window and expires.
- The sender can cancel a remittance while it is `Created`, `Collected`, or `Ready for
  pickup`, never once `Paid` or `Expired`. Cancellation releases any reservation and, if
  money was collected, refunds `total_charged_to_sender` in full via the original payment
  method.
- **A `Ready for pickup` remittance that is not collected within 30 days automatically
  moves to `Expired`, and SendIt retains the funds.** There is no refund to the sender.
  This is different from `Cancelled`, where the sender asked to stop and is refunded, and
  from `Rejected`, where the sender's money was never taken.
- A User Risk flag from the KYC provider stops the remittance where it is, notifies the
  user, and hands the case to the external Security service. Nothing advances until the case is
  cleared; a confirmed case ends the remittance in `Rejected` (with a refund if money had
  been collected).
- Checking a remittance's status by tracking code never requires an account. Account-free
  lookup only shows status, never enough to authorize a payout.
- Every movement of funds (collection, reservation, settlement, refund, expiration) is
  recorded immutably and is auditable. Collected = paid out + refunded + retained at all
  times.
- A corridor (origin and destination country-currency pair) must exist and be active
  before a remittance can be quoted or created on it.

---

## 8. Glossary

| Business term | Meaning | Name in code/system |
|---|---|---|
| Remittance | The end-to-end money transfer | `Remittance` |
| Sender | Whoever starts the transfer | `Sender` |
| Recipient / Beneficiary | Whoever receives the money | `Recipient` / `Beneficiary` |
| Sending channel | App, website, or branch | `SendingChannel` |
| Sender account | Registered account a remote sender needs (KYC result plus linked funding sources) | `SenderAccount` |
| Funding source | Bank account or card a remote sender pays from | `FundingSource` |
| Corridor | Country/currency pair enabled for operation | `Corridor` |
| Quote | Exchange rate plus fee locked in for a remittance | `Quote` |
| Exchange rate | Conversion rate between origin and destination currency | `FxRate` |
| Fee | The charge SendIt takes for the service | `Fee` |
| Branch Agent | SendIt person at the counter (collects, KYC-checks, pays out, cancels) | `BranchAgent` |
| Branch | SendIt's physical location where money is paid in and paid out | `Branch` |
| Tracking code | Unique identifier used to check status and to collect | `TrackingCode` |
| Collection window | Days the recipient has to collect before it expires (**30 days**) | `CollectionWindow` |
| KYC check | Record of verifying a sender or recipient, with any User Risk flag | `KycCheck` |
| User Risk flag | KYC provider signal that a party may be acting in bad faith | `UserRiskFlag` |
| Daily send limit | Max a customer can send per rolling 24h across all channels (**US$ 50,000**) | `DailySendLimit` |
| Destination reserve | Pool of funds SendIt holds in a destination country to pay recipients | `DestinationReserve` |
| Reservation | Hold on a destination reserve for one remittance's payout | `Reservation` |
| Payout order | Settlement instruction (internal cash or bank deposit) | `PayoutOrder` |
| Refund | Return of the charged amount to the sender after a cancellation | `Refund` |
| Receipt | Final proof of a paid remittance | `Receipt` |

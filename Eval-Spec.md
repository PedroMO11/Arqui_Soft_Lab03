# SendIt Eval-Spec

> Domain context: [README.md](README.md). Personas: [Users/Aaron.md](Users/Aaron.md)
> (Sender), [Users/Juan.md](Users/Juan.md) (Recipient), [Users/SendIt.md](Users/SendIt.md)
> (Branch Agent, SendIt acting as its own intermediary).

---

## 1. Summary

SendIt is an international remittance platform. A sender starts a transfer from the mobile
app, from the website, or in person at a SendIt branch. When sending remotely, the sender
pays from a linked bank account or card. When sending at a branch, the sender pays in cash
or by card on the branch card reader (POS). SendIt converts the money using a locked quote
and routes it to a recipient in another country. The recipient collects it either as cash
at a branch or as a deposit into their bank account. SendIt is the intermediary for the
whole trip; there is no outside payout partner.

Identity is checked on both sides with an **identity check (KYC)**: a check run by an
outside provider that confirms a person is who they say they are. A sender who uses the app
or website passes a KYC check and proves ownership of their bank account or card once, when
they sign up. After that, each transfer needs a logged-in session, plus an **extra check
for large amounts**. A sender who uses a branch is KYC-checked in person on every visit.
The recipient is KYC-checked every time, against the details on that specific remittance,
before any money is released. If the KYC provider warns that a person may be acting in bad
faith (a **User Risk flag**), the remittance is held and the case is handed to an outside
Security service.

Before a remittance is made ready for pickup, SendIt sets its payout amount aside in a
destination-country reserve. A remittance that is not collected within 30 days expires and
its funds stay with SendIt. The sender can cancel a remittance any time before it is paid
and get a full refund. Anyone who has the tracking code can check the status of a
remittance without an account.

---

## 2. Problem

- **Sender (Aaron)**: "I don't want to pay high fees to send money to another country." "I
  don't want my money to pass through a long chain of middlemen before it arrives." "My own
  bank either won't send to banks in other countries or charges too much when it does." "If
  I change my mind or make a mistake, I want my money back before it is handed over."
- **Recipient (Juan)**: "The money takes too long to reach me." "I'm worried someone could
  pretend to be me and collect it first."
- **Branch Agent (SendIt)**: "If the system doesn't tell me right away that the KYC check
  failed, I've already taken the cash and I don't know what to do with it." "I could hand
  cash to the wrong person if the system doesn't make me match the ID against the exact
  data on the remittance." "If two agents at two branches try to pay out the same
  remittance, both can't succeed." "If nobody comes to collect, that money can't stay open
  forever." "If there's no money set aside in the destination country, the recipient turns
  up and I can't pay them."

---

## 3. Objective

1. Let a sender finish a cross-border transfer in one session, from the app, the website,
   or a branch, with a fixed quote shown before payment.
2. Make sure no remittance is created, collected, or paid out unless the right party has
   passed a KYC check for that transfer.
3. Make sure a remittance is paid out at most once, even if two branches try at the same
   time.
4. Give the sender and the recipient a way to see the status of a remittance in real time,
   with no account, using the tracking code.
5. Close every remittance in a final state: paid, rejected, expired, or cancelled. Expired
   funds stay with SendIt after 30 days; a cancellation before payout refunds the sender in
   full.
6. Support several country and currency corridors from the start, not one fixed pair.
7. Never make a remittance ready for pickup unless its payout amount is set aside in the
   destination country.

---

## 4. Out of Scope

| Excluded | Why |
|---|---|
| Recipient payout by card or digital wallet | The recipient has exactly two options: cash at a branch or bank deposit |
| Building our own card processor or storing full card numbers | Card payments (branch card reader and online) go through a payment gateway that meets the card-industry security standard (PCI); SendIt is a client of that gateway, it does not build one |
| Checking names against sanctions and politically-exposed-person (PEP) lists, and full anti-money-laundering (AML) monitoring of spending patterns | SendIt acts on the KYC provider's User Risk flag (§5.3), but building its own list checks and pattern monitoring is out of the current scope. A production system would need it |
| Computing market exchange rates | An external FX provider gives the rate; SendIt only uses it |
| Running the KYC checks themselves (document and biometric matching) | An external KYC provider does this; SendIt only reads the result and the User Risk flag |
| Getting physical cash into a branch till | Branch cash logistics are an operations job; the destination reserve (§5) is tracked as balances, not as truck schedules |
| SendIt's own company accounting and taxes | Back-office work, not part of the remittance flow |
| Opening bank accounts or issuing cards for senders or recipients | SendIt moves money, it is not a bank |
| Reversing or disputing an already-`Paid` remittance | A `Paid` remittance is final; a later dispute is handled as a new case, not an edit to this one. Cancelling and refunding *before* payout are in scope (§5.3) |
| Recipient accounts | The recipient never needs an account. A sender needs an account only to use the app or website, never for a branch transfer and never for status lookup |

---

## 5. Key Product Concepts

- **Remittance**: the transfer that moves money from a Sender to a Recipient across a
  Corridor, converted at a locked Quote and identified by a Tracking Code.
- **Corridor**: a country pair SendIt has switched on, with its currencies, for example
  Peru to United States (PEN to USD). A remittance cannot exist outside an active corridor.
- **Quote**: the exchange rate and fee fixed when the remittance is created, valid for a
  short time.
- **Sending channel**: where the sender starts and pays for the transfer (mobile app,
  website, or branch).
- **Branch**: the physical location where cash or a card reader (POS) payment is taken and
  where cash is paid out.
- **Sender account**: the registered account a sender needs to use the app or website. It
  holds their KYC result and their linked bank accounts and cards. Branch senders do not
  have one.
- **Funding source**: the bank account or card a remote sender links and pays from.
- **KYC check**: the record of an identity check on a Sender or a Recipient, run by an
  outside provider. It is tied to one specific remittance, except a remote sender's first
  KYC check, which is done at sign-up. It stores the subject, the result, the date, the
  method, and any User Risk flag the provider returns.
- **User Risk flag**: a warning from the KYC provider that a Sender or Recipient may be
  acting in bad faith. It puts the remittance on hold and hands the case to the Security
  service.
- **Large-amount check**: an extra KYC check the system asks a remote sender for before a
  transfer over the large-amount threshold (§5.1). A logged-in session alone is not enough
  for a big transfer.
- **Tracking Code**: the one public key to a remittance. It is used to check status with no
  account, and, together with a KYC check, to collect the money.
- **Collection Window**: the 30-day period after a remittance is ready for pickup, after
  which it expires.
- **Destination Reserve**: the pool of money SendIt holds in each destination country to
  pay recipients. A remittance is not made ready for pickup until its payout amount is set
  aside against this pool.
- **Reservation**: the hold placed on the Destination Reserve for one remittance's payout
  amount. It is used up when the remittance is paid, and released back to the reserve if
  the remittance expires or is cancelled.
- **Cancellation**: the sender's decision (in the app or website, or through an agent) to
  end a remittance before it is paid. It moves the remittance to `Cancelled`.
- **Refund**: the return of `total_charged_to_sender` to the sender, by the same payment
  method they used, after a cancellation of a remittance that had money collected.

### 5.1 Parameters and Defaults

| Parameter | Value | Where it is used |
|---|---|---|
| Collection window | **30 days**, counted from the moment a remittance becomes `Ready for pickup` | Remittance state machine; expiration job |
| Sender KYC exemption threshold | **None (0)**. Every remittance needs the sender KYC-checked, at any amount | Remittance creation gate |
| Sending channels | **3**: mobile app, website, branch | Create-remittance flow |
| Sender payment methods | **4**, in two groups. In person: cash, card on the branch card reader. Remote: debit from a linked bank account, card online | Payment capture |
| Payout channels | **2**: cash pickup at a branch, deposit to the recipient's bank account | Create-remittance form; payout order |
| Remote sender verification | KYC check and bank-account / card ownership done **once at sign-up**; every transfer after that needs a logged-in session | Sender sign-up; create-remittance gate |
| Large-amount threshold | **US$ 1,000 (or corridor equivalent)** in one remittance or within 24 hours makes the system ask a remote sender for an extra KYC check. Assumed value, confirm against real policy | Create-remittance gate |
| Per-customer daily send limit | **US$ 50,000 (or corridor equivalent)** within any 24-hour window (not the calendar day), added up across app, web, and branch, per verified customer. Assumed value, confirm against real risk policy | Create-remittance gate |
| Branch sender verification | KYC-checked in person **on every branch visit**, never reused from a past visit | Branch create-remittance gate |
| User Risk action | On a User Risk flag from the KYC provider: **hold the remittance, tell the user, hand the case to the external Security service**. Nothing moves on until the case is cleared; a confirmed case ends the remittance in `Rejected` and blocks the sender's account | Create-remittance gate; Security hand-off job |
| Quote validity window | **15 minutes** from issue, then it must be recalculated | Quote; remittance creation |
| Remittance states | **7**: `Created`, `Collected`, `Ready for pickup`, `Paid`, plus final `Rejected`, `Expired`, `Cancelled` | Remittance entity |
| Destination Reserve funding order | When the reserve cannot cover a remittance: **1) SendIt's own money already collected in that country, 2) SendIt corporate account, 3) money borrowed from an outside bank (or similar source)**, in that order | Reservation step (§9 flow 4) |
| Cancellation window | A remittance can be cancelled while `Created`, `Collected`, or `Ready for pickup`. Never once `Paid` or `Expired` | Cancel flow (§9) |
| Refund on cancellation | **100% of `total_charged_to_sender`**, returned by the same payment method the sender used, target **5 business days**. Assumed value | Refund step (§9) |
| Tracking code format | **10 characters**, uppercase letters and digits, checked for repeats when issued | Remittance; status lookup |
| Account requirement for status lookup | **None**. The tracking code alone returns status only, never enough to authorize a payout | Public status endpoint |
| Recipient KYC re-check | Required on **every remittance**, never reused from a past one | Payout gate |
| Expired-funds handling | **100% kept by SendIt**, no refund to the sender, no payout to the recipient | Expiration job |
| Corridors by launch | **At least 2** country pairs active by the launch stage (§10, end of Stage 3), for example Peru–United States, Peru–Spain. The POC (Stage 0) runs one | Corridor table |

### 5.2 The Logic

**Amount the recipient can collect:**

```
destination_amount = origin_amount x exchange_rate
total_charged_to_sender = origin_amount + fee
```

`exchange_rate` and `fee` are the two values frozen into the Quote when the remittance is
created (valid for 15 minutes, see §5.1). They are not recalculated at payout time, even if
the market rate has moved since.

**Worked example:**

- Aaron sends from Peru (PEN) to Juan in the United States (USD).
- `origin_amount` = **S/ 1,000.00** (PEN)
- `exchange_rate` (PEN to USD, locked at quote time) = **0.27**
- `fee` = **S/ 35.00** (flat, set per corridor)
- `destination_amount` = 1,000.00 x 0.27 = **US$ 270.00**
- `total_charged_to_sender` = 1,000.00 + 35.00 = **S/ 1,035.00**

Aaron is charged **S/ 1,035.00**. Juan collects **US$ 270.00**. If Juan does not collect
within **30 days** of the remittance becoming `Ready for pickup`, it becomes `Expired`,
SendIt keeps the **US$ 270.00**, and Aaron is not refunded the S/ 1,035.00. If Aaron
cancels before Juan collects, the remittance becomes `Cancelled` and Aaron is refunded the
full **S/ 1,035.00** by the payment method he used.

### 5.3 Business Rules

- **Pay out once only**: a payout order for a tracking code can succeed at most once, even
  if two branches send the request at the same time. This is done as one all-or-nothing
  step (`Ready for pickup` to `Paid`): it either fully happens or not at all, so a second
  request cannot slip through.
- **Same quote, same result**: with the same locked Quote, the formulas in §5.2 always give
  the same `destination_amount` and `total_charged_to_sender`. The amount on the create
  screen and the amount on the receipt match.
- **Full history**: every state change of a remittance (created, collected, ready, paid,
  expired, cancelled) is stored as a new entry with a timestamp. Nothing is overwritten or
  deleted.
- **Status lookup stays read-only**: looking up a remittance by tracking code never needs
  more than the code, and never returns enough to authorize a payout.
- **Fast response**: issuing a quote and looking up status both return quickly enough for
  someone waiting at a counter or looking at their phone (target: under a second).
- **Expiration is automatic**: the 30-day cutoff is applied by a scheduled job, not by a
  branch action. A remittance cannot be kept open past its window by hand.
- **Per-customer daily cap**: the system refuses any remittance that would push a
  customer's sent total past US$ 50,000 (or corridor equivalent) within any 24-hour window,
  added up across app, web, and branch, per verified customer. The US$ 1,000 large-amount
  check still applies to amounts below this cap.
- **Reserve before ready**: a remittance cannot move from `Collected` to `Ready for pickup`
  until its payout amount is set aside against the destination-country reserve. If the
  reserve is short, it is topped up in the order in §5.1 before the hold is placed.
- **Reservation lifecycle**: a reservation is used up when the remittance is paid, and
  released back to the reserve when the remittance expires or is cancelled. The reserve's
  free, held, and used-up totals always add up.
- **Cancel before payout only**: the sender can cancel while `Created`, `Collected`, or
  `Ready for pickup`. Cancelling releases any reservation and, if money was collected,
  triggers a full refund of `total_charged_to_sender` by the same payment method the sender
  used. A `Paid` or `Expired` remittance cannot be cancelled.
- **User Risk stops everything**: a User Risk flag from the KYC provider stops the
  remittance where it is, tells the user, and hands the case to the external Security
  service. Nothing moves on until the case is cleared; a confirmed case ends the remittance
  in `Rejected` (with a refund if money had been collected) and blocks the sender's
  account.

---

## 6. Users and Their Needs

The model user is the **Sender**: the person who puts money into the system and decides the
transfer. The Recipient and the Branch Agent are supporting users. Most of the security
rules protect the sender's money until it reaches the right recipient.

| Persona | Role | Main needs | Profile |
|---|---|---|---|
| Aaron Camacho | Sender | One trustworthy intermediary; send from the app, the web, or a branch; know the exact amount to pay before paying; get the money back if he cancels before payout | [Users/Aaron.md](Users/Aaron.md) |
| Juan Rodríguez | Recipient | See the status with no account; collect quickly; not be impersonated; collect as cash or into his bank account | [Users/Juan.md](Users/Juan.md) |
| SendIt (Branch Agent) | Internal / intermediary | Never take money before the KYC check clears; never release cash without an ID match; never pay twice; automatic cutoff for unclaimed funds; money set aside in the destination before the recipient arrives | [Users/SendIt.md](Users/SendIt.md) |

---

## 7. Key Product Decisions

| Decision | Reason |
|---|---|
| SendIt pays out every remittance itself; when a destination reserve is short it borrows from an outside bank only to top the reserve up | The case frames SendIt as the full intermediary (README §1). The outside bank never touches the recipient or the payout; it only supplies the reserve when SendIt's own funds in a country fall short. SendIt stays the single intermediary and the destination side is still funded |
| A sender can send from the app, the website, or a branch | Matches how a real remittance service works (for example Western Union). The app and web reach senders who have a bank account or card; the branch reaches senders who want to pay cash |
| Remote payment is a debit from a linked bank account or an online card | Both are common ways to pay for a transfer. Card payments (online and on the branch card reader) go through a PCI payment gateway, so SendIt never stores card data |
| Remote senders pass a KYC check once at sign-up, plus a session per transfer and an extra check for large amounts; branch senders are KYC-checked every visit | This is how remittance apps work in practice. A logged-in session is the per-transfer control for a remote sender. A branch sender has no account, so the check is repeated in person |
| SendIt acts on the KYC provider's User Risk flag, but does not build its own AML or sanctions checks | The User Risk flag is a cheap, concrete control: hold and hand off. Its own list checks and pattern monitoring are a separate, larger effort, left for later (§4, §10) |
| A hard per-customer cap of US$ 50,000 sent within any 24 hours, across all channels | Limits SendIt's exposure to any single customer and gives data consistency a concrete control. The 50k figure is assumed and should be checked against real risk policy. Unlike the US$ 1,000 large-amount check, which only adds a step, this one blocks the transfer |
| Every remittance needs the sender KYC-checked, with no amount exemption | No "was this sender exempt?" edge cases, and it matches the rule that any amount needs verification |
| The sender can cancel and be refunded any time before payout; expiry after 30 days is not a refund | Two different endings. Cancel is a deliberate sender action, so the money goes back in full. Expiry is abandonment, so SendIt keeps it and its exposure stays limited |
| A remittance is not made ready for pickup until its payout amount is set aside in the destination country | Answers the agent's "there's no money to pay the recipient" pain: nothing is promised to a recipient unless it is already set aside |
| Status lookup needs no account | Lets Juan and Aaron follow a remittance without signing up. The tracking code is a read-only key to status |
| The recipient has exactly two payout channels: cash and bank deposit | Matches the case constraint and keeps the "exactly N" claims in §5.1 short and countable |
| Payout is one all-or-nothing step, not a check-then-act in two steps | Prevents the double-payout problem |

---

## 8. Expected User Experience

- **Aaron (Sender), app or web**: signs in, picks the destination country and currency, the
  amount, and the recipient's details. He sees the destination amount, the exchange rate,
  the fee, and the total to pay, all fixed for 15 minutes. He pays from his linked bank
  account or card. For a large amount he is asked for an extra KYC check first. After
  paying he sees a tracking code and a receipt in the app, and can check status any time.
  While the remittance is not yet paid, he can cancel it in the app and get the full amount
  back by the payment method he used.
- **Aaron (Sender), branch**: the agent KYC-checks his ID in person, takes the same
  details, shows the same fixed quote, and takes cash or a card payment on the reader. He
  leaves with a printed tracking code and receipt, and can look up status later with just
  the code. To cancel before payout he asks any branch, which does it on his behalf.
- **Juan (Recipient)**: once notified, he sees the status (`Ready for pickup`, `Paid`,
  `Expired`, or `Cancelled`), the amount in his currency, and the exact date by which he
  must collect, with no account. To collect he must show ID that matches the recipient
  details on the remittance, either to the agent (cash) or to the KYC provider (deposit).
- **SendIt (Branch Agent)**: at the counter, the create flow will not move to payment until
  the sender's KYC check comes back approved, and the payout flow will not release money
  until the recipient's identity is confirmed for that remittance. A failed KYC check, a
  declined payment, a User Risk flag, or recipient data that does not match is shown to the
  agent at that moment, with the reason, before any money is taken. The agent can see
  whether the destination reserve covered a remittance before it went ready. Any attempt to
  pay out a remittance that is already `Paid`, `Expired`, or `Cancelled` is refused, from
  any branch.

---

## 9. Main Flows

1. **Quote and create, remote (Sender, app or web).** The sender signs in and enters the
   amount, destination country and currency, payout channel, and recipient details. The
   system checks the session, asks for an extra KYC check if the amount is over the
   US$ 1,000 large-amount threshold, refuses the transfer if it would take the customer
   over the US$ 50,000 daily limit, queries the FX provider, computes the fee (§5.2), and
   locks the Quote for 15 minutes. The sender confirms. The remittance is created
   (`Created`). If the session check, the large-amount check, or the daily-limit check
   fails, the attempt is recorded and set straight to `Rejected`, the reason is shown to
   the sender at that moment, and nothing is charged. If the KYC provider returns a User
   Risk flag, the remittance is held, the sender is told, and the case is handed to the
   external Security service.
2. **Quote and create, branch (Sender + Agent).** The agent KYC-checks the sender's ID in
   person, enters the same data, the system runs the same US$ 50,000 daily-limit check,
   builds the same Quote, and the sender confirms. The remittance is created (`Created`).
   If the in-person KYC check or the daily-limit check fails, the attempt is recorded and
   set straight to `Rejected`, the agent is shown the reason at that moment, and nothing is
   taken. A User Risk flag holds and hands off the case as in flow 1.
3. **Collect payment.** Remote: SendIt charges the linked bank account or card through the
   payment gateway. Branch: the agent takes cash or a card reader payment. On success the
   remittance moves to `Collected` and the sender is told the money is in. On failure it
   moves to `Rejected`, the reason is shown to the sender or agent at that moment, and
   nothing else happens, because no money was taken.
4. **Reserve and make ready (System).** The system sets the payout amount aside against the
   destination-country reserve, topping the reserve up first (SendIt's own funds in that
   country, then the corporate account, then money borrowed from an outside bank) if it is short. Once
   the amount is set aside, the remittance moves to `Ready for pickup`, the tracking code
   is issued, the 30-day Collection Window starts, and the sender and recipient are told.
5. **Status check (Sender or Recipient, any time, no account).** Either party looks up the
   remittance by tracking code and sees the current state and, if it applies, the days
   left.
6. **Payout, cash (Recipient + Agent, at any branch).** The recipient shows the tracking
   code and ID. The agent checks the ID against the remittance data. On a match, cash is
   released, the reservation is used up, the remittance moves to `Paid`, and a receipt is
   issued.
7. **Payout, deposit (System + receiving bank).** The system KYC-checks the recipient with
   the KYC provider. On success it sends the credit instruction to the receiving bank. On
   confirmation the reservation is used up, the remittance moves to `Paid`, and a receipt
   is issued.
8. **Cancel and refund (Sender in app or web, or Agent on the sender's behalf).** Allowed
   while `Created`, `Collected`, or `Ready for pickup`. Any reservation is released back to
   the destination reserve. If money had been collected, a Refund of
   `total_charged_to_sender` is issued by the same payment method the sender used (target
   5 business days). The remittance moves to `Cancelled` and both parties are told.
9. **Expiration (System, automatic).** If a `Ready for pickup` remittance passes 30 days
   without being collected, it moves to `Expired`, the reservation is released back to the
   reserve, and SendIt keeps the funds. No refund, no payout.
10. **User Risk hand-off (System + external Security service).** A held remittance waits on
    the Security service's verdict. Cleared: it carries on from where it was. Confirmed: it
    moves to `Rejected`, with a refund if money had been collected, and the sender's
    account is blocked.

---

## 10. Scope by Stage

The public launch is the end of **Stage 3**. Stages 0 to 2 are internal steps, not public
releases.

- **Stage 0 (POC)**: one corridor (for example Peru to United States), branch channel only,
  cash payment only, cash payout only, manual FX rate entry (stub provider), status lookup
  by tracking code, and the full `Created -> Collected -> Ready for pickup -> Paid /
  Expired` state machine with the 30-day expiration job.
- **Stage 1**: add card payment on the branch card reader. Add the bank-deposit payout
  channel (receiving-bank integration and recipient KYC check with the external provider).
  Add the destination reserve and the reserve step before `Ready for pickup`. Add cancel
  and refund before payout, with the `Cancelled` state.
- **Stage 2**: add the mobile app and website channels. Add sender sign-up with a KYC check
  and bank-account / card ownership check, remote payment by bank debit and by online card
  through the gateway, the extra check for large amounts, and User Risk flag handling with
  hand-off to the external Security service.
- **Stage 3**: add the second corridor and make corridor setup general instead of one
  hardcoded pair. Add the real FX provider integration.
- **Stage N**: sanctions / PEP list checks and full AML monitoring; reversing or disputing
  an already-`Paid` remittance. Both are out of scope today (§4) and only revisited if the
  case constraints change.

---

## 11. Requirements

A flat list. Each requirement is something the system must do, tied to the persona need or
pain it resolves, with a check that can be verified against this document. **(Critical)**
marks a requirement whose failure invalidates the design. `Eval-Results.md` scores the spec
against this list.

| # | The system must... | Anchor (persona: pain / need, or case constraint) | Acceptance check |
|---|---|---|---|
| R1 | Route every remittance end to end through SendIt, with no external payout partner; any borrowing from an outside bank only tops up the destination reserve **(Critical)** | Aaron: pain "long chain of middlemen"; need "one trustworthy intermediary" | The flow in §9 has no third-party payout step; payout orders go only to a SendIt branch till or a receiving bank; borrowed money appears only in §9 flow 4 as a reserve top-up |
| R2 | Show the destination amount, exchange rate, fee, and total to pay before the sender pays, and hold them fixed for 15 minutes (§5.1, §5.2) | Aaron: need "know the exact amount before paying"; pain "high fees" (transparency) | Create a remittance: the four values show before payment and do not change for 15 min; after 15 min the quote is recalculated |
| R3 | Let a sender create and pay for a remittance from the app, the website, or a branch: remote payment by linked bank account or online card, branch payment by cash or card reader (§5.1) | Aaron: need "pay the way that suits me"; pain "my bank won't send abroad" | Each of the 3 channels can take a remittance to `Collected` with each allowed payment method |
| R4 | Let the recipient receive as cash at any branch or as a deposit to their bank account, as chosen when the remittance is created (§5.1) | Juan: need "choose cash or bank deposit" | Both channels take a remittance to `Paid` |
| R5 | Return remittance status from the tracking code alone, with no account or login, and never return data that would authorize a payout (§5.1, §5.3) **(Critical)** | Juan: need "check status without an account"; pain "takes too long" (visibility) | The status endpoint called with only a code returns state and days left, needs no login, and carries nothing that could authorize a payout |
| R6 | Block a remittance from reaching payment capture until the sender has passed a KYC check for that transfer: in person for a branch transfer, or a verified account plus a valid session for a remote transfer, with an extra KYC check above US$ 1,000 (§5.1, §9) **(Critical)** | SendIt: pain "KYC check failed and I already took the money" | A create attempt with a failed or missing KYC check is stopped before any money is taken |
| R7 | Block payout (cash or deposit) until the recipient is KYC-checked against that specific remittance's details; for a deposit, the account holder must match (§5.1, §5.3) **(Critical)** | Juan: pain "someone could pretend to be me"; SendIt: need "verify recipient before releasing cash", "account matches recipient" | A mismatched ID or account holder blocks the payout; a past remittance's KYC check is never reused |
| R8 | Pay out a remittance at most once, done as one all-or-nothing `Ready for pickup` to `Paid` step, even when two branches try at the same time (§5.3) **(Critical)** | SendIt: pain "two branches can't both succeed" | Two payout requests for one code at the same time: exactly one succeeds, the other is refused |
| R9 | On a failed sender KYC check or failed payment, never leave the remittance in `Collected`: record it and set it to `Rejected` with nothing owed (§5.3, §9) | SendIt: pain "already took the money and don't know what to do" | A failed KYC check, a declined card, a failed bank debit, or unpaid cash all end in `Rejected`, with no payout order and no refund owed |
| R10 | Move any `Ready for pickup` remittance not collected within 30 days to `Expired` through a scheduled job, release its reservation back to the reserve, and keep the funds with SendIt, with no refund and no payout (§5.1, §5.3, §9) **(Critical)** | SendIt: pain "money can't stay open forever"; need "a fixed window" | Advancing the clock 30 days on an uncollected remittance auto-moves it to `Expired`; the reservation is released; funds are kept; both parties see `Expired` |
| R11 | Show the recipient the exact collection deadline (date) and the days left, with no account (§9) | Juan: need "know exactly how many days I have"; pain "takes too long" | Status lookup and the notification both carry the deadline date |
| R12 | Report a failed KYC check, a declined payment, a User Risk flag, or wrong recipient data to the sender or agent at the moment it happens, before money is taken | SendIt: pain "if the system doesn't tell me right away" | Each failure type shows in the create flow right away, not after payment |
| R13 | Refuse any remittance that would take a customer's sent total over US$ 50,000 (or corridor equivalent) within any 24 hours, added up across all channels, per verified customer (§5.1, §5.3) | SendIt: need to limit exposure to any one customer; case: data consistency and control | Attempts that cross the 24-hour total are refused with a clear reason; the total covers any 24 hours, not the calendar day |
| R14 | Support at least two active country/currency corridors by the launch stage (§10, end of Stage 3) and refuse any remittance outside an active corridor (§5.1) | Aaron: pain "my bank won't send to other countries"; objective 6 | Two corridors are set up by launch; a remittance on an inactive corridor is refused |
| R15 | Tell the sender when the remittance is `Collected` and again when it is `Paid`, and issue a receipt | Aaron: need "confirmation once the money has been collected" | The sender gets a notification at `Collected` and at `Paid`; a receipt is available at `Paid` |
| R16 | Store every state change and every movement of money as entries that are written once with a timestamp and never changed or deleted, so that total collected = total paid out + total refunded + total kept at all times (§5.3) **(Critical)** | Case: "data consistency"; SendIt: hand the money to the correct recipient with a trail | The history has no edits or deletes; the four totals always add up for any period |
| R17 | Answer a quote request and a status lookup in under 1 second for at least 95% of requests under expected load (§5.3) | Juan: pain "takes too long" (perceived wait); Aaron: counter and app experience | Under a load test, at least 95% of both operations finish in under 1 second |
| R18 | Produce the same `destination_amount` and `total_charged_to_sender` from the same locked quote every time, so the create screen and the receipt match to the cent (§5.2, §5.3) | Aaron: need "know how much of my money reaches the recipient" | Recomputing from the stored quote equals the receipt exactly |
| R19 | Send all card payments through a PCI payment gateway and never store full card numbers (§4) | Case: "security matters a lot" | No full card number in SendIt storage; only gateway tokens are held |
| R20 | Let the sender (in the app or web) or an agent on the sender's behalf cancel a remittance any time before it is `Paid`; cancelling releases any reservation and, if money was collected, refunds 100% of `total_charged_to_sender` by the same payment method the sender used (§5.1, §5.3, §9) | Aaron: pain "if I change my mind or make a mistake, I want my money back before it is handed over" | Cancelling a `Ready for pickup` remittance sets it to `Cancelled`, releases the reservation, and issues a full refund by the same method; cancelling a `Paid` or `Expired` remittance is refused |
| R21 | Not move a remittance from `Collected` to `Ready for pickup` until its payout amount is set aside against the destination-country reserve, topping the reserve up from SendIt's own funds, then the corporate account, then money borrowed from an outside bank, in that order (§5.1, §5.3, §9) **(Critical)** | SendIt: pain "there's no money set aside in the destination country and I can't pay the recipient" | A `Collected` remittance with a short reserve does not become `Ready for pickup` until the reserve is topped up; the hold is visible against the remittance |
| R22 | On a User Risk flag from the KYC provider, hold the remittance, tell the user, and hand the case to the external Security service; the remittance does not move on until the case is cleared, and a confirmed case ends it in `Rejected` and blocks the sender's account (§5.1, §5.3, §9) | Case: "security matters a lot"; SendIt: keep bad actors out of the flow | A flagged party's remittance is held with a hand-off record and a user notification; clearing carries it on, confirming ends it in `Rejected` with a refund if money was collected and blocks the account |
| R23 | Use up a reservation only on payout and release it back to the reserve on expiry or cancellation, so the reserve's free, held, and used-up totals always add up (§5.3) | Case: "data consistency" | The reserve's numbers add up across free, held, and used-up or released for any period |

### Critical requirements

R1, R5, R6, R7, R8, R10, R16, R21. If any of these is absent from the document, or
contradicted elsewhere in it, the spec fails regardless of how the rest reads.

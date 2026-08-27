# SendIt — Eval-Spec

> Domain context: [README.md](README.md). Personas: [Users/Aaron.md](Users/Aaron.md)
> (Sender), [Users/Juan.md](Users/Juan.md) (Recipient), [Users/SendIt.md](Users/SendIt.md)
> (Branch Agent / SendIt itself as intermediary).

---

## 1. Summary

SendIt is an international remittance platform operated entirely through physical
branches. A sender walks into a branch, pays in cash or by card, and SendIt converts and
routes the money to a recipient in another country, who collects it either as cash at a
branch or as a bank deposit. SendIt is its own intermediary end to end — there is no
external payout partner. Every remittance requires full identity verification (KYC) of
the sender before a cent is taken, and of the recipient before a cent is released. A
remittance not collected within 30 days expires and its funds are retained by SendIt.
Anyone holding the tracking code can check a remittance's status without ever creating an
account.

---

## 2. Problem

- **Sender (Aaron)**: "I don't want to spend too much on fees to transfer money to other
  countries." "I don't want to go through too many intermediaries just so my money
  reaches the recipient." "My bank in the country where I live doesn't allow transfers to
  other countries' banks."
- **Recipient (Juan)**: "It takes too long for the money to reach me." "I'm afraid
  someone else could impersonate me and collect in my place."
- **Branch Agent (SendIt)**: "If the system doesn't warn me right away that KYC failed,
  I've already taken the money and now I don't know what to do with it." "I could hand
  cash to the wrong person if the system doesn't force me to validate identity against
  the exact data on the remittance." "If two agents at two different branches both try to
  pay out the same remittance, they can't both succeed." "If nobody ever comes to
  collect, that money can't just sit open forever."

---

## 3. Objective

1. Let a sender complete a cross-border remittance in a single branch visit, with a
   binding quote shown before payment (attacks Aaron's fee-opacity and multi-intermediary
   pains).
2. Guarantee that no remittance is created, collected, or paid out without the relevant
   party's identity being verified for that specific transaction (attacks the
   impersonation and wrong-payout pains of Juan and SendIt).
3. Guarantee that a remittance can be paid out exactly once, even when attempted from two
   different branches concurrently (attacks SendIt's double-payout pain).
4. Give both sender and recipient real-time, account-free visibility into a remittance's
   status via its tracking code (attacks Juan's "it takes too long, and I can't tell what
   is happening" pain).
5. Close every remittance definitively: paid, rejected, or expired — with expired funds
   automatically retained by SendIt after 30 days (attacks SendIt's unbounded-liability
   pain).
6. Support multiple country/currency corridors from day one, not a single fixed pair.

---

## 4. Out of Scope

| Excluded | Why |
|---|---|
| Remote/electronic payment by the sender (app, saved card, online transfer) | The case restricts sender payment to cash or in-person card at the branch |
| Recipient payout by card or digital wallet | The case restricts payout to exactly two channels: cash pickup and bank deposit |
| Compliance screening (sanctions/PEP lists, AML) | Out of scope for this design exercise; KYC identity check is the only admission control |
| Card payment authorization / PCI processing | Handled by the branch's external POS processor; SendIt is its client, not its implementer |
| Computing market exchange rates | Provided by an external FX provider; SendIt consumes the quote |
| Biometric/document identity verification logic | Performed by an external KYC provider; SendIt only consumes the verdict |
| SendIt's internal corporate accounting/tax management | Back-office concern, not part of the remittance domain |
| Banking for sender or recipient (accounts, cards) | SendIt is a sending intermediary, not a bank |
| Post-payout disputes/reversals | Out of scope for now; a `Paid` remittance is final in this exercise |
| Sender/recipient account creation or login | Deliberately absent — see §5.2/§8; identity is verified per transaction, not via a persistent login |

---

## 5. Key Product Concepts

- **Remittance** — the transaction moving money from a Sender to a Recipient across a
  Corridor, converted at a locked-in Quote and identified by a Tracking Code.
- **Corridor** — an enabled origin-country/destination-country currency pair. A
  remittance cannot exist outside an active corridor.
- **Quote** — the exchange rate and fee locked in at remittance creation, valid for a
  limited time.
- **Branch** — the physical location where cash/card is taken and cash is paid out.
- **Tracking Code** — the single public key to a remittance: used to check status (no
  account needed) and, together with identity verification, to collect it.
- **Collection Window** — the 30-day period after a remittance becomes ready for pickup,
  after which it expires.
- **KYC Check** — the identity-verification record for a Sender (always) or a Recipient
  (on deposit payout, or in person for cash payout), tied to one specific remittance.

### 5.1 Concrete Parameters & Defaults

| Parameter | Value | Where it is used |
|---|---|---|
| Collection window | **30 days**, counted from the moment a remittance becomes `Ready for pickup` | Remittance state machine; expiration job |
| Sender KYC exemption threshold | **None — 0.** Full KYC is mandatory for every remittance regardless of amount | Remittance creation gate |
| Payout channels | Exactly **2**: Cash pickup at a branch, Bank deposit | Remittance creation form, Payout order |
| Sender payment methods | Exactly **2**, both in-person: Cash, Card (POS) | Payment capture screen |
| Quote validity window | **15 minutes** from issuance before it must be recalculated | Quote, Remittance creation |
| Remittance states | Exactly **5**: `Created`, `Collected`, `Ready for pickup`, `Paid`, plus terminal `Rejected` / `Expired` | Remittance entity |
| Tracking code format | **10 alphanumeric characters**, uppercase, collision-checked at issuance | Remittance, status lookup |
| Account requirement for status lookup | **None.** Tracking code alone is sufficient and returns status only, never enough to authorize payout | Public status-lookup endpoint |
| Recipient identity re-verification | Required **every remittance**, never cached across transactions | Payout gate |
| Expired-funds disposition | **100% retained by SendIt**, no refund to sender, no payout to recipient | Expiration job |
| Corridors at launch | **Minimum 2** country pairs active (e.g. Peru↔United States, Peru↔Spain) to demonstrate multi-corridor support | Corridor table |

### 5.2 The Model

**Payable amount to the recipient:**

```
destination_amount = origin_amount × exchange_rate
total_charged_to_sender = origin_amount + fee
```

Where `exchange_rate` and `fee` are the two values frozen into the **Quote** at creation
time (§5.1: valid for 15 minutes) — never recomputed at payout time, even if the market
rate has since moved.

**Worked reference case:**

- Aaron sends from Peru (PEN) to Juan in the United States (USD).
- `origin_amount` = **S/ 1,000.00** (PEN)
- `exchange_rate` (PEN→USD, locked at quote time) = **0.27**
- `fee` = **S/ 35.00** (flat, corridor-specific)
- `destination_amount` = 1,000.00 × 0.27 = **US$ 270.00**
- `total_charged_to_sender` = 1,000.00 + 35.00 = **S/ 1,035.00**

Aaron pays **S/ 1,035.00** at the counter; Juan collects **US$ 270.00**. If Juan does not
collect within **30 days** of the remittance reaching `Ready for pickup`, it moves to
`Expired` and SendIt retains the **US$ 270.00** — Aaron is not refunded the S/ 1,035.00.

### 5.3 Engineering Invariants & NFRs

- **Idempotent payout**: a payout order for a given tracking code can succeed at most
  once, even under concurrent requests from two branches — enforced by an atomic
  state transition (`Ready for pickup` → `Paid`), not by application-level checks alone.
- **Determinism of the quote**: given the same locked Quote, the model in §5.2 always
  produces the same `destination_amount` and `total_charged_to_sender` — no rounding
  drift between the creation screen and the receipt.
- **Auditability**: every state transition of a Remittance (creation, collection,
  payout, expiration) is append-only and timestamped; nothing is overwritten or deleted.
- **Account-free read path**: the status-lookup capability (tracking code → status) must
  never expose or require authentication beyond the code itself, and must never return
  enough information to authorize a payout on its own.
- **Latency**: quote issuance and status lookup respond within a time short enough for a
  counter interaction (target: sub-second) — a sender or recipient standing at a counter
  or kiosk should not perceive a wait.
- **Expiration is automatic**: the 30-day cutoff is enforced by a scheduled process, not
  by manual branch action — a remittance cannot be kept artificially alive past its
  window.

---

## 6. Users and Their Needs

| Persona | Role | Key needs | Profile |
|---|---|---|---|
| Aaron Camacho | Sender | Trustworthy single intermediary; pay in cash or card; know the exact payable amount upfront | [Users/Aaron.md](Users/Aaron.md) |
| Juan Rodríguez | Recipient | Know remittance status without an account; collect quickly; not be impersonated | [Users/Juan.md](Users/Juan.md) |
| SendIt (Branch Agent) | Internal / intermediary | Never take money before KYC clears; never release cash without identity match; never double-pay; automatic cutoff on unclaimed funds | [Users/SendIt.md](Users/SendIt.md) |

---

## 7. Key Product Decisions

| Decision | Reasoning |
|---|---|
| SendIt is its own payout intermediary — no external cash network partner | The case study frames SendIt as the full intermediary (§1 of README); modeling an external payout partner would just be relabeling SendIt itself, adding a fictitious integration with no design payoff |
| Sender payment is in-person only (cash or card at a branch) | Given per the case's constraints; also sidesteps card-not-present fraud and PCI-online scope entirely, keeping the 4h design focused on branch operations |
| No AML/sanctions screening layer | Explicitly out of scope for this exercise; KYC identity verification is the sole admission control, keeping the compliance surface small enough to specify precisely in the time available |
| Full KYC on every remittance, no threshold exemption | Removes an entire class of "was this sender exempt?" edge cases and matches the instruction that any amount requires verification |
| Status lookup requires no account | Directly serves Juan's and Aaron's need to track a remittance without an onboarding step; the tracking code is the capability token, scoped to read-only status |
| Unclaimed funds are retained by SendIt after 30 days, not refunded | Bounds SendIt's liability for abandoned remittances indefinitely; 30 days is assumed as an industry-typical collection window (comparable programs use 30–45 days) and should be confirmed against real business policy |
| Exactly two payout channels (cash, deposit) and two payment methods (cash, card) | Matches the case's explicit constraint; keeps §5.1's "exactly N" claims small and fully enumerable |
| Payout is modeled as an atomic state transition, not a two-step check-then-act | Directly defends against the double-payout pain SendIt's own agent identified as a top concern |

---

## 8. Expected User Experience

- **Aaron (Sender)** sees, before paying: destination amount, exchange rate, fee, and
  total charged — all frozen for 15 minutes. After paying, he sees a tracking code and a
  receipt. He can look up status later using only the tracking code.
- **Juan (Recipient)** sees, once notified: remittance status (`Ready for pickup` /
  `Paid` / `Expired`), the amount payable in his currency, and the exact deadline (date)
  by which he must collect — all without creating an account. To actually collect, he
  must present ID that the agent (or the KYC provider, for deposit) can validate against
  the recipient details on the remittance.
- **SendIt (Branch Agent)** sees, at the counter: a creation flow that blocks progression
  to payment until KYC returns approved, and a payout flow that blocks release until
  recipient identity is validated for that specific remittance. Any attempt to pay out an
  already-`Paid` or `Expired` remittance is rejected outright, from any branch.

---

## 9. Main Flows

1. **Quote & Create** (Sender, at branch) — Sender states amount, destination
   country/currency, and payout channel → Agent runs Sender KYC → system queries FX
   provider and computes fee (§5.2) → Quote locked for 15 minutes → Sender confirms →
   Remittance created (`Created`).
2. **Collect payment** (Agent) — Agent captures cash or card payment → on success,
   Remittance moves to `Collected`; on failure, Remittance is marked `Rejected` and
   nothing further happens (no money was taken).
3. **Make ready & notify** (System) — Remittance moves to `Ready for pickup`, tracking
   code is issued, Collection Window (30 days) starts, Sender and Recipient are notified.
4. **Status check** (Sender or Recipient, anytime, no account) — either party queries
   status by tracking code and sees current state and, if applicable, days remaining.
5. **Payout — cash** (Recipient + Agent, at any branch) — Recipient presents tracking
   code and ID → Agent validates identity against Remittance data → on match, cash is
   released and Remittance moves to `Paid`, Receipt issued.
6. **Payout — deposit** (System + Receiving bank) — System validates Recipient identity
   against the KYC provider → on success, sends crediting instruction to the receiving
   bank → on confirmation, Remittance moves to `Paid`, Receipt issued.
7. **Expiration** (System, automatic) — if a `Ready for pickup` remittance passes 30 days
   uncollected, it moves to `Expired`; funds are retained by SendIt; no refund, no payout.

---

## 10. Scope by Stage

- **Stage 0 (POC)**: single corridor (e.g. Peru → United States), cash payout channel
  only, cash payment method only, manual FX rate entry (stubbed provider), tracking-code
  status lookup, the full `Created → Collected → Ready for pickup → Paid/Expired` state
  machine with the 30-day expiration job.
- **Stage 1**: add card payment (POS integration), add bank-deposit payout channel
  (receiving-bank integration, recipient remote KYC).
- **Stage 2**: add the second corridor and generalize corridor configuration (rather than
  hardcoding one pair), add real FX provider integration.
- **Stage N**: post-payout disputes/reversals, compliance screening — both explicitly
  out of scope today (§4), reconsidered only if the case's constraints change.

---

## 11. Acceptance Criteria

### 11.1 Aaron — Sender

| # | The evaluator checks that the document specifies… | Anchor |
|---|---|---|
| A1 | The exact formula and the two frozen inputs (exchange rate, fee) used to compute the total charged to the sender (§5.2), shown before payment | Pain: "I don't want to spend too much on fees to transfer money" |
| A2 | That the sender pays through exactly one intermediary (SendIt itself) with no chain of external hand-offs (§7) **(Critical)** | Pain: "I don't want to go through too many intermediaries" |
| A3 | That payment can be made in cash or card, both in person, without requiring a bank account (§5.1) | Pain: "My bank...doesn't allow transfers to other countries' banks" |

### 11.2 Juan — Recipient

| # | The evaluator checks that the document specifies… | Anchor |
|---|---|---|
| J1 | That remittance status can be queried using only the tracking code, with no account creation or login required (§5.1, §5.3) **(Critical)** | Need: "know the status... without an account" |
| J2 | The exact collection window (**30 days**) and that it is visible to the recipient before it lapses (§5.1, §9) | Pain: "It takes too long for the money to reach me" |
| J3 | That payout (cash or deposit) is blocked until the recipient's identity is validated against the specific remittance's registered details **(Critical)** | Pain: "I'm afraid someone else could impersonate me" |

### 11.3 SendIt (Branch Agent) — Internal / Intermediary

| # | The evaluator checks that the document specifies… | Anchor |
|---|---|---|
| S1 | That remittance creation cannot progress to payment capture without sender KYC returning approved (§5.1, §9) **(Critical)** | Pain: "I've already taken the money and now I don't know what to do with it" |
| S2 | That a payout on an already-`Paid` remittance is rejected atomically, including when attempted concurrently from a different branch (§5.3) **(Critical)** | Pain: "two agents at two different branches...can't both succeed" |
| S3 | The exact disposition of an expired remittance's funds (retained by SendIt, no refund) and that expiration is automatic, not manual (§5.1, §9) | Pain: "that money can't just sit open forever" |

### Scoring Guidance

- Each **(Critical)** criterion failing invalidates the design for that persona's core
  need; a document missing any of A2, J1, J3, S1, or S2 cannot score above a fail for
  that persona's block regardless of how the rest reads.
- Non-critical criteria are scored per-anchor: full credit only if the document states
  the concrete value (not just the concept) and names where it's used, per §5.1's rule.
- A criterion is graded strictly against what `Eval-Spec.md` states — not against what
  the evaluator assumes the system probably does.

---

## 12. Assumptions and Risks

| Assumption/Risk | Impact if false | Mitigation |
|---|---|---|
| 30-day collection window is acceptable to the business | If the real policy is shorter/longer, every reference to "30 days" in §5.1, §5.2, §9, and both acceptance criteria J2/S3 needs updating in lockstep | Confirm against actual SendIt (or comparable) business policy before implementation; kept as a single named parameter (`CollectionWindow`) specifically so it's a one-place change |
| 15-minute quote validity is short enough to prevent FX arbitrage but long enough for a counter transaction | Too short: senders get re-quoted mid-transaction; too long: SendIt absorbs FX risk | Treat as tunable per corridor once real FX volatility data exists |
| Full KYC with no exemption threshold is operationally feasible at every branch | If low-value remittances need a lighter path, the "no exemption" rule in §5.1 and §7 would need a tiered exception | Revisit only if the business explicitly asks for a KYC-light tier — not assumed here |
| Two payout channels and two payment methods are exhaustive for this case | If wallet or card payout is later required, §5.1's "exactly 2" claims and the Payout Order model both need extending | Payout channel is modeled as an enumerable field specifically to make this a low-cost addition later |
| Flat, corridor-specific fee (not tiered by amount) | If fee should scale with amount, §5.2's formula needs a fee function instead of a constant | No signal in the case study that fee is anything but flat; assumed for the worked example only |

---

## 13. Definition of Done

- [ ] All 5 remittance states and their transition table (§9) implemented and observable
- [ ] §5.2's model produces the exact worked reference case's numbers (S/ 1,035.00 charged,
      US$ 270.00 payable) end to end
- [ ] Sender KYC blocks payment capture; recipient identity check blocks payout — both
      demonstrable as hard gates, not soft warnings
- [ ] Tracking-code status lookup works with zero authentication and returns status only
- [ ] A payout attempted twice on the same tracking code (including from two simulated
      branches) succeeds exactly once
- [ ] A remittance past the 30-day window auto-transitions to `Expired` without manual
      intervention
- [ ] At least 2 corridors configured and operable (§10, Stage 2) if beyond POC

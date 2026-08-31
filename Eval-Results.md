# SendIt Eval-Results

How [Eval-Spec.md](Eval-Spec.md) was scored, and what is still open. This file checks the
spec; it is not part of it.

---

## 1. Method

The spec is scored on two axes plus a case-fit check, with a gate and a set of penalties.

### Axis A: requirement soundness (2 points)

For each requirement `Rn` in `Eval-Spec.md` §11, check all four points:

1. It can be verified by reading `Eval-Spec.md`, without assuming how the system behaves.
2. It is tied to a real persona pain or stated need, or to a case constraint.
3. Something else in the spec backs it with a concrete value or a named flow (§5.1, §5.2,
   §9), not only the §11 line.
4. It does not contradict any other part of the spec.

Verdict: **PASS** (all four), **WEAK** (one missing or partial), **FAIL** (two or more, or
the requirement cannot be tested).

Score: `2 * (PASS + 0.5 per WEAK) / number of requirements`.

### Axis B: needs coverage (6 points)

List every pain and every stated need of every persona in `/Users`. For each, find the
PASS requirement(s) that resolve it.

Verdict: **Resolved**, **Partial** (addressed with a real gap), **Not resolved**.

Score: `6 * (Resolved + 0.5 per Partial) / number of pains and needs`.

### Case-fit (2 points)

The case names two hard constraints, 1 point each, given only if the spec addresses the
constraint with at least one concrete mechanism.

- **Security**: identity established before money moves, on both sides.
- **Data consistency**: money is never double-counted, double-paid, or lost.

### Critical gate

The requirements marked **(Critical)** in §11 are R1, R5, R6, R7, R8, R10, R16, R21. If any
is WEAK or FAIL, or if any pain a persona calls critical is Not resolved, the spec is
capped at **5 / 10** regardless of the axis totals.

### Penalties (subtracted, capped at -2)

- -0.5 per contradiction between two sections.
- -0.5 per "exactly N" that miscounts.
- -0.5 per number stated in prose that is not in §5.1 or derived from §5.2.

### Pass bar

**8 / 10.** Below that, fix the spec, not the requirement.

---

## 2. Result

Scored against 23 requirements (R1 to R23).

### 2.1 Critical gate

| Critical req | Soundness | In gate |
|---|---|---|
| R1 single intermediary (borrowing only tops up the reserve) | PASS | ok |
| R5 account-free status | PASS | ok |
| R6 sender KYC before payment | PASS | ok |
| R7 recipient KYC per remittance | PASS | ok |
| R8 pay out once only | PASS | ok |
| R10 automatic 30-day expiry, reservation released | PASS | ok |
| R16 write-once history, totals always add up | PASS | ok |
| R21 reserve in destination before `Ready for pickup` | PASS | ok |

**Gate: PASS.** No cap applied.

### 2.2 Axis A: requirement soundness

R1 to R19 all PASS, each backed by §5.1, §5.3 or §9. R20 to R23:

| Req | Verdict | Where the spec backs it |
|---|---|---|
| R20 cancel and refund before payout | PASS | §5.1 cancellation-window and refund rows; §5.3 "Cancel before payout only"; §9 flow 8; §7 |
| R21 reserve before ready | PASS | §5.1 funding-order row; §5.3 "Reserve before ready"; §9 flow 4; §7; §3 objective 7 |
| R22 User Risk hold and hand-off | PASS | §5.1 User Risk row; §5.3 "User Risk stops everything"; §9 flows 1, 2, 10; §4 |
| R23 reservation used up on payout, released on expiry or cancel | PASS | §5.3 "Reservation lifecycle"; §9 flows 6 to 9 |

PASS 23, WEAK 0, FAIL 0. **Axis A = 2 / 2.**

### 2.3 Axis B: needs coverage

**Aaron, Sender**

| Pain or need | Requirement(s) | Verdict | Note |
|---|---|---|---|
| one trustworthy intermediary | R1 | Resolved | |
| pay the way that suits me | R3, R14 | Resolved | 3 channels, 4 methods |
| know the exact amount before paying | R2, R18 | Resolved | fixed 15 min; same quote, same numbers |
| confirmation once the money is collected | R15 | Resolved | notified at `Collected` (§9 flow 3) |
| get the money back in full if he cancels before collection | R20 | Resolved | full refund to the original method (§9 flow 8) |
| "high fees" | R2, R1 | Partial | fee is visible and flat per corridor, middlemen removed, but no low-fee guarantee |
| "long chain of middlemen" | R1 | Resolved | |
| "my bank won't send abroad" | R3, R14 | Resolved | |
| "I want my money back if I change my mind" | R20 | Resolved | cancel allowed while `Created`, `Collected`, or `Ready for pickup` |

Resolved 8, Partial 1.

**Juan, Recipient**

| Pain or need | Requirement(s) | Verdict | Note |
|---|---|---|---|
| know the status of the money | R5, R11 | Resolved | |
| check status with the code only, no account | R5 | Resolved | |
| get the money as soon as possible | R4, R10, R17 | Partial | create-to-ready time (payment clearing plus reservation) is not bounded |
| know exactly how many days I have | R11 | Resolved | |
| choose cash or bank deposit | R4 | Resolved | |
| "money takes too long" | R11, R17 | Partial | visibility and lookup speed are covered; delivery speed depends on external rails |
| "someone could pretend to be me" | R7 | Resolved | per-remittance KYC check, account-holder match, no reuse |

Resolved 5, Partial 2.

**SendIt, Branch Agent**

| Pain or need | Requirement(s) | Verdict | Note |
|---|---|---|---|
| take the money, later hand it to the right recipient | R6, R7, R8, R16 | Resolved | |
| KYC-check recipient against sender-registered details | R7 | Resolved | |
| check the bank account matches the recipient (deposit) | R7 | Resolved | |
| tell the sender right away if creation fails | R12, R22 | Resolved | §8; a User Risk flag is one of the surfaced failures |
| fixed 30-day window, then SendIt keeps the funds | R10 | Resolved | |
| bound how much one customer moves per day | R13 | Resolved | US$ 50,000 in any 24 hours |
| payout amount set aside in the destination before the recipient is told ready | R21 | Resolved | reservation gates `Ready for pickup` |
| "KYC check failed and I already took the money" | R6, R9 | Resolved | R6 blocks before money; R9 records a failed check or payment as `Rejected` |
| "could hand cash to the wrong person" | R7 | Resolved | |
| "two branches can't both succeed" | R8 | Resolved | |
| "money can't stay open forever" | R10 | Resolved | |
| "no money set aside in the destination, recipient turns up, I can't pay" | R21 | Resolved | reserve topped up from own funds, then corporate, then borrowed |

Resolved 12, Partial 0.

**Totals:** 28 rows. Resolved 25, Partial 3, Not resolved 0.
**Axis B = 6 * (25 + 1.5) / 28 = 5.68 / 6.**

### 2.4 Case-fit

| Constraint | Mechanisms in the spec | Point |
|---|---|---|
| Security | R6 (sender KYC gate), R7 (recipient KYC gate), R8 (pay out once only), R5 (read-only status), R19 (PCI gateway), R22 (User Risk hold and hand-off) | 1 / 1 |
| Data consistency | R8 (all-or-nothing payout step), R16 (write-once history: collected = paid + refunded + kept), R18 (same quote, same numbers), R21 and R23 (reserve totals add up, nothing promised without funds set aside) | 1 / 1 |

**Case-fit = 2 / 2.**

### 2.5 Penalties

No contradictions. `Eval-Spec.md` and `REDALE.md` agree on channels, payment methods,
sender accounts, the large-amount check, the daily limit, the `Cancelled` state, cancel
and refund, User Risk, and the destination reserve. The "exactly N" counts hold: channels
3, payment methods 4, payout channels 2, states 7. Every number stated in prose (15 min,
30 days, US$ 1,000, US$ 50,000, 5 business days, the funding order) is in §5.1.

**Penalties = 0.**

### 2.6 Score

| Component | Points |
|---|---|
| Axis A, requirement soundness | 2.00 / 2 |
| Axis B, needs coverage | 5.68 / 6 |
| Case-fit | 2.00 / 2 |
| Penalties | 0.00 |
| **Total** | **9.7 / 10** |

Clear pass. The three Partials in §3 are the only open items on persona coverage, and none
touches a requirement or a Critical.

---

## 3. Open gaps

The first three are Partial coverage, not missing requirements. None blocks the pass.

1. **"High fees" (Aaron).** Add a line to §7: SendIt does not promise the lowest fee, only
   a visible, flat, single-intermediary fee. Or set a fee target in §5.1.
2. **Delivery speed (Juan).** Note in §7 that end-to-end speed depends on external rails
   and the reserve step. Optionally set a target for create-to-`Ready for pickup`.
3. **Assumptions scattered inline.** The 30-day window, the US$ 50,000 cap, the US$ 1,000
   large-amount check, the 5-business-day refund target and the funding order are all
   assumed values marked in place. A short "Assumptions" section would collect them.
4. **Agent access is not a requirement.** `REDALE.md` builds agent sign-in, roles and a
   Security service, but `Eval-Spec.md` §11 has no requirement that agents authenticate or
   that cancel and refund need a supervisor role. This falls outside the three personas,
   but the case stresses security, so it is worth one requirement.

---

## 4. Iteration history

What the design gained or changed between revisions, and the score after each.

| Rev | Date | Change | Score |
|---|---|---|---|
| 1 | 2026-08-27 | First pass. Branch-only remittance service. Requirements written as one flat list, each tied to a persona pain or need. | 8.2 / 10 |
| 2 | 2026-08-27 | Closed internal contradictions: the sender is notified at `Collected`; "launch" is defined as the end of Stage 3; a failed KYC check ends the remittance in `Rejected`; failures are shown to the agent at the moment they happen. | 9.6 / 10 |
| 3 | 2026-08-27 | Scope widened to match a real remittance service: added the mobile app and website channels, remote payment (bank debit and online card), sender accounts, the large-amount check and the US$ 50,000 daily limit. Added User Risk handling, cancel and refund before payout, and the destination-country reserve. New `Cancelled` state (5 states to 7) and requirements R20 to R23. `REDALE.md` brought in line with the spec. | 9.7 / 10 |
| 4 | 2026-08-28 | Wording pass for a general reader. `Eval-Spec.md` and `REDALE.md` reconciled on every shared term and number. No change to coverage or score. | 9.7 / 10 |

# SendIt — Eval-Results

The evaluator's run against [Eval-Spec.md](Eval-Spec.md). This file holds the score and the
gaps. It is not part of the deliverable spec; it is the check on it.

---

## 1. Method

The spec is judged on two axes, a gate, and a set of penalties.

### Axis A — Requirement soundness (2 points)

For each requirement `Rn` in `Eval-Spec.md` §11, check all four:

1. It can be verified by reading `Eval-Spec.md` alone (not "the system does X" on faith).
2. It is anchored to a real persona pain or stated need, or to a case constraint.
3. It is backed by a concrete value or a named flow somewhere else in the spec (§5.1,
   §5.2, §9), not only asserted in §11.
4. It does not contradict any other part of the spec.

Verdict: **PASS** (all four), **WEAK** (one missing or partial), **FAIL** (two or more, or
the requirement is untestable).

Points = `2 × (PASS + 0.5·WEAK) / total requirements`.

### Axis B — Needs coverage (6 points)

List every pain and every stated need of every persona in `/Users`. For each, find the
`PASS` requirement(s) that resolve it.

Verdict: **Resolved** (fully), **Partial** (addressed with a real gap), **Not resolved**.

Points = `6 × (Resolved + 0.5·Partial) / total pains and needs`.

### Case-fit (2 points)

The case names two hard constraints. Each is worth 1 point, awarded only if the spec
addresses it with at least one concrete mechanism, not just a sentence.

- **Security**: identity established before money moves, on both sides.
- **Data consistency**: money is never double-counted, double-paid, or lost.

### Critical gate

The requirements marked **(Critical)** in `Eval-Spec.md` §11 are R1, R5, R6, R7, R8, R10,
R16, R21. If any of them is `WEAK` or `FAIL`, or if any pain that a persona calls critical
is `Not resolved`, the spec is capped at **5 / 10** no matter what the axes add up to.

### Penalties (subtracted, capped at −2)

- −0.5 for each contradiction between two sections.
- −0.5 for each "exactly N" that miscounts.
- −0.5 for each number promised in prose that is not in §5.1 or derived from §5.2.

### Pass bar

**8 / 10.** Below that, fix the spec, not the requirement. A requirement watered down so it
passes is exactly what this exercise is built to catch. Log every run in §4.

---

## 2. Run 5 — 2026-08-28

Run 3 scored 9.7 / 10. Two rounds of coherence and language work since:

- **Run 4** (plain language): `step-up` → "large-amount check"; `KYC` glossed once per
  document as "identity check (KYC)"; `append-only`, `atomic`, `reconcile`, `p95` replaced
  with plain words. Rule added: a confirmed User Risk case also blocks the sender's account
  (§5.3, R22).
- **Run 5** (REDALE cross-check follow-ups): the reserve top-up source is stated as
  "borrowed from an outside bank" everywhere (was "bank partner"); `REDALE.md` gained a
  Parameters table with the exact values from §5.1; `REDALE.md`'s `Transfer Service` now
  matches the spec's money model (no cross-border movement of the sender's cash);
  `Receiving Bank Service` split out from `External Bank Service`.

No coverage or soundness change in either round; the score holds at 9.7 / 10.

### 2.1 Critical gate

| Critical req | Soundness | In gate? |
|---|---|---|
| R1 single intermediary (borrowing only tops up the reserve) | PASS | ok |
| R5 account-free status | PASS | ok |
| R6 sender KYC before payment | PASS | ok |
| R7 recipient KYC per remittance | PASS | ok |
| R8 pay out once only (all-or-nothing step) | PASS | ok |
| R10 automatic 30-day expiry, reservation released | PASS | ok |
| R16 write-once history, four totals always add up | PASS | ok |
| R21 reserve in destination before `Ready for pickup` | PASS | ok |

**Gate: PASS.** No cap applied.

### 2.2 Axis A — Requirement soundness

R1–R19 all PASS (wording says "KYC check" and "large-amount check", still backed by §5.1 /
§5.3 / §9). The four newer requirements:

| Req | Verdict | Evidence in spec | Gap |
|---|---|---|---|
| R20 cancel + refund before payout | PASS | §5.1 cancellation-window and refund-on-cancellation rows; §5.3 "Cancel before payout only"; §9 flow 8; §7 decision | — |
| R21 reserve before ready | PASS | §5.1 Destination Reserve funding-order row; §5.3 "Reserve before ready"; §9 flow 4; §7 decision; §3 objective 7 | — |
| R22 User Risk hold and hand-off | PASS | §5.1 User Risk action row; §5.3 "User Risk stops everything"; §9 flows 1–2 and 10; §4 narrowed row | — |
| R23 reservation used up on payout, released on expiry / cancel | PASS | §5.3 "Reservation lifecycle"; §9 flows 6–9 | — |

PASS 23, WEAK 0, FAIL 0.
**Axis A = 2 × 23 / 23 = 2.00 / 2.**

### 2.3 Axis B — Needs coverage

**Aaron — Sender** (2 new rows: cancel/refund need and pain)

| Pain or need | Requirement(s) | Verdict | Note |
|---|---|---|---|
| Need: one trustworthy intermediary | R1 | Resolved | — |
| Need: pay the way that suits me | R3, R14 | Resolved | 3 channels, 4 methods |
| Need: know the exact amount before paying | R2, R18 | Resolved | Fixed 15 min; same quote always gives the same numbers |
| Need: confirmation once the money is collected | R15 | Resolved | §9 flow 3 notifies at `Collected` |
| Need: get the money back in full if he cancels before collect | R20 | Resolved | Full refund to the original method, §9 flow 8 |
| Pain: "high fees" | R2, R1 | Partial | Fee is visible and flat per corridor and middlemen are removed, but no low-fee guarantee |
| Pain: "long chain of middlemen" | R1 | Resolved | — |
| Pain: "my bank won't send abroad" | R3, R14 | Resolved | — |
| Pain: "if I change my mind or make a mistake, I want my money back" | R20 | Resolved | Cancel allowed while `Created` / `Collected` / `Ready for pickup` |

Resolved 8, Partial 1, Not 0.

**Juan — Recipient** (unchanged)

| Pain or need | Requirement(s) | Verdict | Note |
|---|---|---|---|
| Need: know the status of the money | R5, R11 | Resolved | — |
| Need: check status with the code only, no account | R5 | Resolved | — |
| Need: get the money as soon as possible | R4, R10, R17 | Partial | Create-to-ready time (payment clearing plus reservation) is not bounded by the spec |
| Need: know exactly how many days I have | R11 | Resolved | — |
| Need: choose cash or bank deposit | R4 | Resolved | — |
| Pain: "money takes too long" | R11, R17 | Partial | Visibility and lookup speed are covered; actual delivery speed depends on external rails |
| Pain: "someone could pretend to be me" | R7 | Resolved | Per-remittance KYC check, account-holder match, no reuse |

Resolved 5, Partial 2, Not 0.

**SendIt — Branch Agent** (2 new rows: destination-reserve need and pain)

| Pain or need | Requirement(s) | Verdict | Note |
|---|---|---|---|
| Need: take the money, later hand it to the right recipient | R6, R7, R8, R16 | Resolved | — |
| Need: KYC-check recipient against sender-registered details | R7 | Resolved | — |
| Need: check the bank account matches the recipient (deposit) | R7 | Resolved | — |
| Need: tell the sender right away if creation fails | R12, R22 | Resolved | §8; User Risk flag is one of the surfaced failures |
| Need: fixed 30-day window, then SendIt keeps funds | R10 | Resolved | — |
| Need: bound how much one customer moves per day | R13 | Resolved | US$ 50,000 in any 24 hours |
| Need: payout amount set aside in destination before recipient is told ready | R21 | Resolved | Reservation gates `Ready for pickup` |
| Pain: "KYC check failed and I already took the money" | R6, R9 | Resolved | R6 blocks before money; R9 records a failed check or payment as `Rejected` |
| Pain: "could hand cash to the wrong person" | R7 | Resolved | — |
| Pain: "two branches can't both succeed" | R8 | Resolved | — |
| Pain: "money can't stay open forever" | R10 | Resolved | — |
| Pain: "no money set aside in the destination, recipient turns up, I can't pay" | R21 | Resolved | Reserve topped up from own funds → corporate → borrowed from an outside bank |

Resolved 12, Partial 0, Not 0.

**Totals:** 28 rows. Resolved 25, Partial 3, Not 0. The 3 Partials (Aaron "high fees",
Juan "get the money ASAP", Juan "money takes too long") are unchanged; see §3.
**Axis B = 6 × (25 + 1.5) / 28 = 5.68 / 6.**

### 2.4 Case-fit

| Constraint | Mechanisms in the spec | Point |
|---|---|---|
| Security | R6 (sender KYC gate), R7 (recipient KYC gate), R8 (pay out once only), R5 (read-only status), R19 (PCI gateway), R22 (User Risk hold and hand-off) | 1 / 1 |
| Data consistency | R8 (all-or-nothing payout step), R16 (write-once history: collected = paid + refunded + kept), R18 (same quote, same numbers), R21 + R23 (reserve totals add up, no promise without funds set aside) | 1 / 1 |

**Case-fit = 2 / 2.**

### 2.5 Penalties

No new contradictions. `Eval-Spec.md` and `REDALE.md` agree on channels, payment methods,
sender accounts, the large-amount check, the daily limit, KYC wording, the `Cancelled`
state, cancel + refund, User Risk (including the sender-account block), and the destination
reserve. "Exactly N" claims recount clean: channels 3, payment methods 4, payout channels
2, states 7. Every promised number (15 min, 30 days, US$ 1,000, US$ 50,000, 5 business
days, reserve funding order) is in §5.1.

**Penalties = 0.**

### 2.6 Score

| Component | Points |
|---|---|
| Axis A — requirement soundness | 2.00 / 2 |
| Axis B — needs coverage | 5.68 / 6 |
| Case-fit | 2.00 / 2 |
| Penalties | 0.00 |
| **Total** | **9.7 / 10** |

**Result: 9.7 / 10. Pass, comfortably.** The three remaining Partials in §3 are all that is
left, and none of them touches a requirement or a Critical.

---

## 3. Gaps that remain

All three are Partial coverage, not missing requirements. None touches a Critical. Fixing
them is optional at 9.7 / 10.

1. **Aaron "high fees" (Partial).** Add a line to §7: SendIt does not promise the lowest
   fee, it promises a visible, flat, single-intermediary fee. Or set a fee target in §5.1.
2. **Delivery speed (Partial).** Acknowledge in §7 that end-to-end speed depends on
   external rails and the reserve step; optionally set a target for
   create-to-`Ready for pickup`.
3. **Assumptions not held in one place.** The 30-day window, the US$ 50,000 cap, the
   US$ 1,000 large-amount check, the 5-business-day refund target, and the reserve funding
   order are assumed values marked inline. Add an "Assumptions and Risks" section (playbook
   §12) so a strict reader sees them owned, not buried.
4. **Agent access not in §11 (from the REDALE cross-check).** `REDALE.md` builds Register /
   Login / Security services and an `Agent` entity with `role`, but `Eval-Spec.md` §11 has
   no requirement that agents sign in and act within a role. Not a coverage gap for the
   three personas, but a real hole for "security matters a lot". Suggested: add one
   requirement that agents authenticate and that money-out actions (cancel, refund) need a
   supervisor role.

---

## 4. Iteration history

| Run | Date | Score | What changed since the last run |
|---|---|---|---|
| 1 | 2026-08-27 | 8.2 / 10 | First run after §11 was rewritten as one flat requirements list |
| 2 | 2026-08-27 | 9.6 / 10 | Applied 4 fixes: §9 flow 3 now notifies the sender at `Collected` (R15); §10 defines launch as the end of Stage 3, R14 / §5.1 point to it; §9 flows 1–2 record a failed KYC check as `Rejected` (R9); §8 states failures are shown to the agent at that moment (R12). R9, R12, R14, R15 moved WEAK → PASS; both penalties cleared |
| 3 | 2026-08-27 | 9.7 / 10 | Scope grew: KYC terminology restored; User Risk, cancel + refund before payout, and the destination-country reserve pulled into scope; `Cancelled` state added (states 5 → 7); requirements R20–R23 added; REDALE.md aligned with the spec on channels, payment methods, sender accounts, the large-amount check, daily limit and `Cancelled`. All 23 requirements PASS; coverage 25/28 Resolved |
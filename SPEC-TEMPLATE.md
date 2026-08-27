# Spec Playbook — template and rules for building an Eval-Spec from scratch

A reusable guide for producing the documentation of a software-architecture case study
like UTEC's (Lea$e, Lab 02). **It does not cover the POC** — documents only.

**How to use it in a fresh chat:** paste this whole file as the first message, together
with the assignment and the professor's screenshots, and say:

> Follow this playbook. Start with `README.md`, the raw domain context, from what I'm
> giving you; then hand me the open questions before you touch `Eval-Spec.md`.

---

## 1. Which files must exist

```
root/
├── README.md              raw domain context — the raw material, not the deliverable
├── Eval-Spec.md           THE DELIVERABLE. This is what gets graded
├── Eval-Results.md        the evaluator's run against the spec: score and gaps
├── Users/
│   ├── <Persona1>.md      one per user. Role, needs, pains, criteria
│   └── ...
└── docs/
    └── ARCHITECTURE.md    drivers, style, bounded contexts, ADRs
```

The rule that governs all of it: **every fact lives in exactly one file.** If a parameter
appears in two, one of them will go stale, and the evaluator will find the contradiction
rather than you.

---

## 2. `README.md` — domain context

Purpose: the raw reference. Written **before** the spec, and not itself graded.

Sections:

1. **Context** — the three or four economic actors in the case and how money moves
   between them.
2. **Purpose of the system** — what the platform does, in two paragraphs.
3. **Scope** — in / out. Everything "out" carries its reason (a third party does it, it's
   another domain, it adds nothing to the case).
4. **Actors** — a table of internal users, external users, and **integrations** (the
   non-human actors: credit bureau, bank, e-invoicing, insurer, ERP).
5. **Domain entities** — the nouns of the business, with attributes and states.
6. **Processes** — the end-to-end lifecycle, numbered.
7. **Business rules** — everything of the form "X cannot happen without Y".
8. **Glossary** — the business term in its own language, and the name it takes in code.

Write this one in the language of the business. The spec is written in English (the
professor asks for it), but the interface and the glossary keep the real vocabulary:
*cuota*, *cronograma*, *acta de conformidad*, *mora*.

---

## 3. `Eval-Spec.md` — the deliverable

Section order. The first eleven are the professor's template; the ones marked ➕ are
additions that raise the score because they make the document **checkable**.

| # | Section | What goes in it |
|---|---|---|
| 1 | Summary | What the system is, in five to eight lines. No marketing adjectives |
| 2 | Problem | The real pains, quoting the user verbatim wherever you have the quote |
| 3 | Objective | Four to six numbered objectives, each one attacking a pain from §2 |
| 4 | Out of Scope | What's excluded **and why**. An exclusion without a reason reads as an oversight |
| 5 | Key Product Concepts | The nouns of the domain, defined |
| 5.1 ➕ | **Concrete Parameters & Defaults** | The table of pinned values. **The highest-yield section in the document** |
| 5.2 ➕ | **The Model** | The core arithmetic or logic: formula, invariants, and a worked reference case |
| 5.3 ➕ | Engineering Invariants & NFRs | Latency, determinism, audit trail, idempotency |
| 6 | Users and Their Needs | Summary table plus links into `/Users` |
| 7 | Key Product Decisions | The decisions, each with its reasoning. This is where architectural judgement shows |
| 8 | Expected User Experience | What each role sees. Written as checkable claims |
| 9 | Main Flows | The full lifecycle, numbered, naming the role responsible for each step |
| 10 | Scope by Stage | Stage 0 (POC) → Stage N. What belongs to each |
| 11 | Acceptance Criteria | One block per persona plus cross-cutting. **This is what the evaluator walks** |
| 12 ➕ | Assumptions and Risks | Assumptions that would change the design if false; risks with mitigation |
| 13 ➕ | Definition of Done | A checklist of what "finished" means (if there is a POC) |

### 3.1 The §5.1 table — concrete parameters

The single highest-scoring trick. Every vague concept gets anchored to a value:

```markdown
| Parameter | Value | Where it is used |
|---|---|---|
| VAT (IGV) rate | **18%**, on every installment and on the purchase option | Installment, receipt |
| Purchase option | **1% of asset value**; range 0–5% | Quote, contract, closing |
| Homologation checklist | Exactly the **4 named criteria**: track record, prior compliance, references, after-sales warranty — a supplier cannot go Active with any unset | Supplier record |
| Dashboard tiles | Exactly these **10**, each independently drill-down capable: 1 …, 2 …, 10 … | Management dashboard |
```

Rules:
- Every number is **bold** and carries its unit.
- If you write "exactly N", the evaluator will count. Count first.
- Every row declares **where it is used**. A parameter nothing consumes is decoration.

### 3.2 State machines

Every entity with a lifecycle carries: the list of states, the **transition table**
(from → to → trigger), and the **deliberate absences**, explained.

```markdown
States: `A`, `B`, `C`, `D`. Mutually exclusive.

| From | To | Trigger |
|---|---|---|
| A | B | … |
| B | C | … |

**Two absences are deliberate.** There is no `B → A` edge, because … . And `D` is
terminal with respect to time: … . Reversal is the single exit, and it is a new entry,
never an edit.
```

Explaining what does *not* exist is worth as much as what does: it shows the missing
state was considered and rejected, rather than forgotten.

### 3.3 Acceptance criteria (§11)

One block per persona. Format:

```markdown
### 11.x <Name> — <Role>

| # | The evaluator checks that the document specifies… | Anchor |
|---|---|---|
| D1 | The exact value of … (§5.1: **X**) | Pain: "user quote" |
| D2 | That … blocks … until … **(Critical)** | Pain: … |
```

The rules that make the difference:
- A criterion checks **the document**, not the software. Write "that the document
  specifies…", never "that the system does…".
- Every criterion anchors to a pain from §2 or to a stated need of that persona. An
  unanchored criterion is a preference.
- Mark **(Critical)** the ones whose failure invalidates the design. Be stingy: one to
  three per persona.
- The ID (`D1`, `V5`, `R3`) is referenced from the rest of the document, and later from
  code.
- Close with **Scoring Guidance**: how points are awarded and what counts as a critical
  failure.

### 3.4 The worked reference case

If the domain involves calculation — financial, pricing, inventory — publish **one
complete case with real numbers**: inputs, formula, result to the smallest unit.

It pays twice: the evaluator sees the model close, and any implementation has something
to verify itself against. In this project that case caught an arithmetic error in the
spec itself, which is the best argument for including one.

**Warning:** a number published in prose is a promise. Publish fifteen figures and all
fifteen have to agree with each other. Recompute before publishing.

---

## 4. `Users/<Persona>.md`

One per persona, same skeleton:

```markdown
# <Name> — <Job title>

## Role
What they do in the system, in three or four lines.

## A day in their work
The typical path, in steps. This is where flows the spec was missing turn up.

## Needs
- Needs … in order to …

## Current pains
- "…" ← verbatim quote where you have one; otherwise the pain phrased as they would say it

## What the system gives them
- …

## Associated acceptance criteria
D1, D4, X2 …  ← the §11 IDs, so the document navigates in both directions
```

Six to nine personas is the sensible range. Include **at least one external** (client,
supplier): it forces you to think about the portal, per-company scoping, and permissions.

---

## 5. `docs/ARCHITECTURE.md`

What an architecture course expects to see:

1. **Drivers** — a table: driver / why it hurts / which decision it forces.
2. **Style** — the decision (e.g. modular monolith, hexagonal inside) with an explicit
   **Why not X** and **Why not Y** for the alternatives you rejected.
3. **Bounded contexts** — the map and the relationships between them.
4. **The domain core** — what is pure, what touches no I/O and no clock, and why.
5. **Persistence and consistency** — what is transactional, what is eventual.
6. **Integration** — ports and adapters, one per external system, each with its fake.
7. **Security and access** — the role/capability model.
8. **Deployment** — a simple diagram.
9. **ADRs** — one decision per block: context, decision, consequence, **cost accepted**.
10. **What this architecture deliberately does not do** — and why.

An ADR without an accepted cost is not an ADR, it is advertising.

---

## 6. `Eval-Results.md`

The evaluator's run. Structure:

- Overall score and score per persona.
- Criterion-by-criterion table: `ID | verdict | evidence (§ and quote) | gap`.
- The criticals, separately and first.
- Iteration history: what was fixed between runs.

If the score comes back low, **do not soften the criterion** — fix the spec. A criterion
watered down so it passes is exactly what the exercise is designed to catch.

---

## 7. Order of work

1. `README.md`, the domain context (30–40 min).
2. The list of personas → the `Users/*.md` files. The pains fall out of these, and the
   pains are the raw material for §2 and §11.
3. Skeleton of `Eval-Spec.md`: §1–4 and §6, quickly.
4. **§5.1, the parameter table.** Everything gets decided here. Every question that
   surfaces is either a question for the professor or an assumption for §12.
5. §5.2, the model plus the worked reference case. Recompute it.
6. §7 decisions, §8 UX, §9 flows, §10 stages.
7. §11 criteria, one block per persona, anchored to the pains.
8. §12 assumptions and risks.
9. Run the evaluator → `Eval-Results.md` → fix → repeat until ≥8/10.

---

## 8. Mistakes that cost points

| Mistake | How it looks | Antidote |
|---|---|---|
| Prose with no numbers | "the system validates the amount" | §5.1 pins the amount |
| "Exactly N", miscounted | Spec says 10 tiles, the system has 7 | Count before writing "exactly" |
| Uncheckable criterion | "that the UX is intuitive" | Phrase it as something readable in the document |
| Figures that disagree | Two sections publish different totals | One reference case, recomputed |
| Everything is critical | Twenty criteria marked **(Critical)** | One to three per persona, maximum |
| A block with no owner | "the user cannot continue" | "…— owner: Treasury" |
| States with no transitions | A list of five states and nothing else | Transition table plus explained absences |
| Flat scope | Everything in a single delivery | §10 by stages, Stage 0 minimal and demonstrable |
| Exclusions with no reason | A bare list under Out of Scope | Every exclusion carries its motive |
| A document you cannot navigate | IDs that are never referenced again | `§5.1`, `V5`, `D8` cited in both directions |

---

## 9. Opening prompts for a fresh chat

**To build the context:**
> Here is the assignment and the professor's screenshots. Follow the attached playbook.
> Start with `README.md`: context, purpose, scope, actors (integrations included),
> entities, processes, rules, and the glossary. When you're done, give me the open
> questions before touching the spec.

**For the spec:**
> With `README.md` settled, write `Eval-Spec.md` in the section order the playbook gives.
> Start with §5.1: anchor every vague concept to a concrete value and tell me which ones
> you had to assume. Don't write §11 until §5.1 and §5.2 are closed.

**For the criteria:**
> Write §11: one block per persona, each criterion checking what the *document* specifies
> (not what the software does), anchored to a pain from §2, with at most two marked
> (Critical) per persona. Add the Scoring Guidance at the end.

**To review:**
> Walk `Eval-Spec.md` as if you were the evaluator and fill in `Eval-Results.md`:
> criterion, verdict, evidence with § and quote, gap. Do not soften any criterion to make
> it pass; if something is absent from the document, record it as a gap.

---

## 10. Final pass before submitting

- [ ] Every number in the document appears in §5.1 or derives from §5.2
- [ ] Every "exactly N" has been counted
- [ ] Every §11 criterion can be checked by reading the document
- [ ] Every criterion anchors to a pain or a stated need
- [ ] Every block names the role that owns it
- [ ] Every state machine has a transition table
- [ ] Every §4 exclusion carries its reason
- [ ] The §12 assumptions are the ones that would change the design if false
- [ ] The reference case closes to the smallest unit
- [ ] No fact lives in two files

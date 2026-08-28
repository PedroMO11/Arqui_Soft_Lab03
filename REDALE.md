# SendIt - R.E.D.A.L.E.

Case Study #3, remittance service.

Model user: the Sender. He is the one who decides to use SendIt. The Recipient and the
Branch Agent use the system but do not choose it.

How the service works: the Sender goes in person to a SendIt branch and pays cash or card
at the counter. SendIt keeps the funds, charges a fee, applies an exchange rate and
delivers the money to the Recipient in the destination country. There are two payout
channels: cash pickup at a branch, or deposit into a bank account. SendIt acts as the
intermediary for the whole operation and does not delegate the payout to a partner.

---

## R - Requirements

Problems to solve.

Sender:
- His bank does not allow transfers to banks in other countries.
- Going through several intermediaries makes the transfer slower and more expensive.
- He does not know how much will arrive, or how much he is being charged, before paying.
- After handing over the money he cannot tell what happened to it.

Recipient:
- He does not know if the money arrived and has no account to check it with.
- The money takes too long to become available.
- Another person could present himself as him and collect the money.

SendIt at the counter:
- Money can be taken before knowing who the sender is, leaving the agent holding cash for
  a remittance that was not valid.
- Cash can be given to the wrong person if identity is not checked against the data
  registered in the remittance.
- The same remittance can be paid twice if two branches try to pay it at the same time.
- There may be no money available in the destination country when the recipient arrives.
- Money that nobody collects stays open with no end date.
- Without a record that cannot be modified there is no way to prove what was collected and
  what was paid.

Constraints:
- The money collected has to match the money payable. It cannot be paid twice or lost.
- A slow external provider cannot block the counter.
- Quote and status lookup have to respond fast enough for someone waiting at a counter.
- Information has to be protected in transit and at rest.

---

## E - Estimate

Inputs. Western Union is used as the reference for scale.

| Input | Value | Where it comes from |
|---|---|---|
| Agent locations | 500,000 | Published by Western Union, in 200 countries and territories |
| Customers served | 150M | Published by Western Union. They are not accounts: at the counter a customer does not register |
| Remittances per day | 1M | Western Union reported 75M transactions in Q4 2024, around 300M per year, which is 820,000 per day. Rounded up for headroom |
| Status checks per remittance | 10 | Assumption. Sender and recipient both check until the money is collected |
| Peak factor | 5 | Assumption, for paydays and weekends |

Load:
- Writes: 1M/day = 12 tx/s average, 60 tx/s peak.
- Status reads: 10M/day = 116 req/s average, 580 req/s peak.
- Counter operations (login, quote, identity check): 60 req/s peak.
- Total: 700 req/s peak.

Servers, taking 1 core at 5 req/s and a server of 32 cores at 160 req/s:
- 700 / 160 = 4.4, so 5 application servers.
- With N+1 redundancy in 3 regions: 30 application servers.
- The limit is the database, not the CPU. There are only 60 writes/s but they have to be
  serialized.

Storage, calculated from the data model in section A. One remittance and everything it
generates:

| Record | Per remittance | Size each | Subtotal |
|---|---|---|---|
| Remittance | 1 | 500 B | 500 B |
| Quote | 1.5, counting abandoned quotes | 100 B | 150 B |
| Payment | 1 | 100 B | 100 B |
| Reservation | 1 | 100 B | 100 B |
| Payout order | 1 | 100 B | 100 B |
| Receipt | 1 | 100 B | 100 B |
| Identity check | 2, sender and recipient | 400 B | 800 B |
| Audit event | 6, one per state change | 450 B | 2.7 KB |
| Total | | | 5 KB |

- 1M/day x 5 KB = 5 GB/day, 1.8 TB/year.
- 10 years with 3 copies: 55 TB.
- Reference data is small: 500k agents and branches is around 170 MB, and corridors are a
  few dozen records.

---

## D - Design the services

Access and identity. Only branch agents have an account; senders and recipients do not.

| Service | What it does |
|---|---|
| Register Service | Registers a branch agent |
| Login Service | Authenticates the agent at the counter |
| Security Service | Session security, permissions and what each agent is allowed to do |

Creating the remittance.

| Service | What it does |
|---|---|
| Registration Service | Registers the remittance: amount, corridor, recipient and payout channel |
| Validation Service | Validates the remittance data: active corridor, complete recipient data, limits |
| Money Laundering Service | Checks who the two parties are. Verifies the identity of sender and recipient with the KYC provider and takes its verdict. Payment capture is blocked until it approves |
| Money exchange Service | Gets the rate from the FX provider and locks the quote for its validity window |
| Transaction Code Service | Generates the tracking code |
| Payment Capture Service | Takes cash or card at the counter. Until it succeeds there is no money |

Moving the money.

| Service | What it does |
|---|---|
| Transfer Service | Moves the remittance from the origin country to the destination country |
| Money Administration Service | Controls the money: what came in, what is payable and what was paid |
| Money reservation Service | Registers the reservation of money for a remittance: which funds, from which source and against which remittance. The reservation is a record, not a transfer |
| Reserve Service | The pool of funds SendIt holds in each country and how much of it is already reserved |
| Self-fund Service | SendIt's own cash, money already collected from other senders in that country. First source used |
| Corp-fund Service | SendIt's corporate account. Used when the own cash does not cover the amount |
| Bank Service | Bank partners. Used when the amount is too large, or when neither the own cash nor the corporate account cover it |

Paying out.

| Service | What it does |
|---|---|
| Recipient Service | Handles the recipient on the destination side |
| Recipient identification service | Verifies the recipient's identity against the data registered in that remittance. Blocks the release if it does not match |
| Withdraw Service | Releases the money, as cash at the branch or as a deposit. Guarantees a single payout even if two branches try at the same time |
| Receipt Service | Issues the proof of the paid remittance to both parties |

Status and closing.

| Service | What it does |
|---|---|
| View Transaction Service | Shows the status to sender or recipient, without an account |
| Transaction Code verification Service | Validates the tracking code used to look up or to collect |
| Notification Service | Notifies sender and recipient on each status change |
| Expiration Service | Expires remittances that pass the collection window. The funds stay with SendIt and the reservation returns to the reserve |

---

## A - Data model

**Agent**
| Attribute | Data type |
|---|---|
| id | integer |
| branch_id | integer |
| name | string(120) |
| role | enum: agent, supervisor |
| password_hash | string(60) |
| status | enum: active, inactive |

**Branch**
| Attribute | Data type |
|---|---|
| id | integer |
| country | string(2) |
| city | string(80) |
| currency | string(3) |
| status | enum: open, closed |

**Corridor**
| Attribute | Data type |
|---|---|
| id | integer |
| origin_country | string(2) |
| origin_currency | string(3) |
| destination_country | string(2) |
| destination_currency | string(3) |
| status | enum: active, inactive |

**Quote**
| Attribute | Data type |
|---|---|
| id | integer |
| corridor_id | integer |
| origin_amount | decimal(14,2) |
| exchange_rate | decimal(12,6) |
| fee | decimal(12,2) |
| destination_amount | decimal(14,2) |
| expires_at | datetime |

**Remittance**
| Attribute | Data type |
|---|---|
| id | integer |
| tracking_code | string(10), unique |
| quote_id | integer |
| corridor_id | integer |
| sender_name | string(120) |
| sender_document | string(20) |
| recipient_name | string(120) |
| recipient_document | string(20) |
| origin_amount | decimal(14,2) |
| fee | decimal(12,2) |
| exchange_rate | decimal(12,6) |
| destination_amount | decimal(14,2) |
| payout_channel | enum: cash, deposit |
| state | enum: created, collected, ready_for_pickup, paid, rejected, expired |
| created_at | datetime |
| collected_at | datetime, empty until the payment is captured |
| ready_at | datetime, empty until the funds are reserved |
| paid_at | datetime, empty until it is paid |
| expires_at | datetime |
| created_by_agent | integer |
| paid_by_agent | integer, empty until it is paid |

**Payment**
| Attribute | Data type |
|---|---|
| id | integer |
| remittance_id | integer |
| method | enum: cash, card |
| amount | decimal(14,2) |
| captured_at | datetime |
| terminal_reference | string(40), empty when cash |

**Reserve**
| Attribute | Data type |
|---|---|
| id | integer |
| country | string(2) |
| currency | string(3) |
| available_amount | decimal(18,2) |
| reserved_amount | decimal(18,2) |

**Reservation**
| Attribute | Data type |
|---|---|
| id | integer |
| remittance_id | integer |
| reserve_id | integer |
| amount | decimal(14,2) |
| source | enum: self_fund, corp_fund, bank |
| state | enum: held, released, consumed |
| created_at | datetime |
| released_at | datetime, empty while held |

**Payout order**
| Attribute | Data type |
|---|---|
| id | integer |
| remittance_id | integer |
| channel | enum: cash, deposit |
| bank_account | string(40), empty when cash |
| state | enum: pending, executed, failed |
| executed_at | datetime |

**Receipt**
| Attribute | Data type |
|---|---|
| id | integer |
| remittance_id | integer |
| issued_at | datetime |
| document_reference | string(60) |

**Identity check**, written once and not modified afterwards
| Attribute | Data type |
|---|---|
| id | integer |
| remittance_id | integer |
| subject | enum: sender, recipient |
| subject_name | string(120) |
| subject_document | string(20) |
| document_type | string(30) |
| provider | string(30) |
| result | enum: approved, rejected |
| checked_at | datetime |

**Audit event**, written once and not modified afterwards
| Attribute | Data type |
|---|---|
| sequence | integer |
| remittance_id | integer |
| actor | string(30) |
| action | string(30) |
| state_before | string(20) |
| state_after | string(20) |
| occurred_at | datetime |

### Remittance states

Created, Collected, Ready for pickup, Paid, and the final states Rejected and Expired.

| From | To | Trigger |
|---|---|---|
| Created | Collected | Payment captured at the counter |
| Created | Rejected | Identity check fails or payment is declined. No money was taken |
| Collected | Ready for pickup | Funds reserved in the destination and tracking code issued |
| Ready for pickup | Paid | Recipient identity validated and money released |
| Ready for pickup | Expired | The collection window passes. The funds stay with SendIt |

There is no transition out of Paid. A paid remittance cannot be reversed. A later dispute
is registered as a new remittance and not as a change to this one.

---

## L - Components

### Iteration 1

```mermaid
flowchart LR
    S([Sender]) --> INT([Intermediary])
    INT --> LOGIN[Login Service]
    REG[Register Service] --> LOGIN
    LOGIN --> SEC[Security Service]

    LOGIN --> RS[Registration Service]
    RS --> VAL[Validation Service]
    VAL --> AML[Money Laundering Service]
    AML --> FX[Money exchange Service]
    RS --> TC[Transaction Code Service]

    RS --> TR[Transfer Service]
    TR --> MA[Money Administration Service]
    MA --> MR[Money reservation Service]
    MR --> CORP[Corp-fund Service]
    MR --> SELF[Self-fund Service]
    MR --> BANK[Bank Service]
    CORP --> RES[Reserve Service]
    SELF --> RES
    BANK --> RES
    WD[Withdraw Service] --> RES

    MA --> RCP[Recipient Service]
    RCP --> RID[Recipient identification service]
    RID --> WD
    RCP --> R([Recipient])

    S --> VT[View Transaction Service]
    R --> VT
    VT --> TCV[Transaction Code verification Service]
```

Order enforced in this iteration: the sender's identity is verified before the remittance
is created, the amount is reserved in the destination before the recipient is told the
money is available, and the recipient's identity is verified before the money is released.

Only the path where every step works is drawn. Failures are not covered yet.

---

## E - Scale

| Load | What fails first | Response |
|---|---|---|
| One branch | Nothing | One server and one database |
| One country | Status lookups are more frequent than anything else | Read replicas and a cache for the lookup |
| Several countries | Reserve per country and cross-border latency | One reserve and one database per country. The corridor is the unit of partitioning |
| Full scale, 150M users | Serialization of the payout | 30 application servers, from section E. Each remittance stays in a single database so the payout can be controlled there |

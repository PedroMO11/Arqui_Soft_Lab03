# Juan Rodríguez — Recipient

## Role
A person who receives a remittance sent by family or a friend from another country. He
collects it either as cash at a SendIt branch or as a deposit into his bank account.

## A day in their work
- Juan is notified that a remittance is waiting for him, with the tracking code.
- Cash channel: he goes to a SendIt branch with his ID and the tracking code. The agent
  runs a KYC check on his identity against the sender's data and hands him the cash.
- Bank deposit channel: SendIt runs a KYC check on his identity with the KYC provider and
  sends the money to his account. No branch visit needed.
- If he does not collect within 30 days, the remittance expires. SendIt keeps the funds.
  The sender is not refunded and Juan is not paid.
- If the sender cancels before Juan collects, the remittance shows as `Cancelled` and the
  sender gets the money back.

## Needs
- To know the status of the money a family member or friend sent him.
- To check that status with the tracking code alone, without creating an account first.
- To get the money as soon as possible.
- To know exactly how many days he has to collect before the remittance expires.
- To choose whether he collects cash at a branch or a deposit to his bank account.

## Current pains
- "The money takes too long to reach me."
- "I'm worried someone could pretend to be me and collect it first."

## What the system gives them
- Real-time status and the tracking code, with no account or registration.
- A KYC check (at the counter, or with the provider for deposits) that blocks
  impersonation.
- A clear collection deadline shown before the remittance expires.
- Two ways to collect: cash at a branch, or a deposit to his bank account.

## Associated requirements
`Eval-Spec.md` §11: R4, R5, R7, R10, R11, R17. Coverage for this persona is scored in
`Eval-Results.md`.

# atm

A test target for a requirements exercise for teaching testing.

## Assignment

An ATM allows a daily withdraw limit of 300€

You can’t ask the stakeholder any questions about the requirement, what tests would you run to understand the requirement better and test this functionality?

## Model

The requirement names one rule and one number. Behind them sit several interacting concepts.

### Concepts

**Cardholder** — the person making the withdrawal; identified by a Card.

**Card** — the physical or virtual card presented at the ATM; linked to one or more Accounts.

**Account** — holds the customer's funds; has a balance and optionally its own daily withdrawal limit independent of any ATM limit.

**ATM** — the machine; has its own daily withdrawal limit and a finite cash inventory of denominated bills (€10, €20, €50, €100).

**Withdrawal** — a single attempt to take money; carries an amount, an outcome (approved / declined), a decline reason if applicable, and a timestamp that determines which Day it belongs to.

**Day** — the time window that resets the daily counters; its definition is not given by the requirement (see oracle tests: calendar midnight, rolling 24 hours, which timezone, DST).

**Daily limit** — a cap on the cumulative approved amount within a Day; may exist at the ATM level, the Account level, or both, and may be scoped to a Card, an Account, or a Cardholder.

### Relationships

```
Cardholder
  └─ uses ──────────────► Card
                            └─ linked to ──► Account
                                              ├─ balance
                                              └─ account daily limit
  └─ initiates ─────────► Withdrawal
                            ├─ amount
                            ├─ outcome
                            └─ timestamp (→ Day)
                            │
                            └─ at ─────────► ATM
                                              ├─ ATM daily limit
                                              └─ Cash inventory
                                                  ├─ €100 ×n
                                                  ├─ €50  ×n
                                                  ├─ €20  ×n
                                                  └─ €10  ×n
```

### Effective limit rule

The amount available for any single withdrawal is:

```
available = min(ATM remaining, Account remaining, Account balance)

ATM remaining     = ATM daily limit     − sum of approved withdrawals at this ATM today
Account remaining = Account daily limit − sum of approved withdrawals across all ATMs today
```

A withdrawal is approved only when `requested amount ≤ available` **and** the ATM cash inventory can make exact change for that amount.

### Where the model is underspecified

Each of the following is a branching point — a different answer gives a different system, and each branch needs its own tests:

| Dimension                 | Options                                                           |
| ------------------------- | ----------------------------------------------------------------- |
| Scope of "daily"          | Calendar day (midnight) · Rolling 24-hour window                  |
| Clock authority           | ATM local time · Bank server time · Customer home-branch timezone |
| Limit scope               | Per card · Per account · Per cardholder                           |
| ATM limit                 | This machine only · Shared across all ATMs                        |
| Joint/multi-card accounts | Shared pool · Independent limits per card                         |
| Failed transaction        | Counts toward daily total · Does not count                        |
| Reversed transaction      | Subtracted from daily total · Not subtracted                      |
| Exact change impossible   | Decline · Dispense only when request fits limits                  |

---

## Solution

Tests split into two categories: **behavior tests** confirm what should happen given the rule; **oracle tests** expose what the requirement leaves undefined — each one is a question the requirement doesn’t answer.

---

### Behavior tests

These have a knowable expected outcome.

**The 300€ boundary**

**Cumulative tracking across transactions**

**Declined attempts do not consume the limit**

**Edge inputs**

**Denomination and change interactions**

**Feedback after transactions**

### Oracle tests

These reveal ambiguity. The requirement is silent; the test result defines the answer.

**What does “daily” mean?**

**Which ATM does the limit apply to?**

**Whose limit is it?**

**What counts toward the total?**

**Currency and denomination edge cases**

**Limit changes and overrides**

**Security and abuse**

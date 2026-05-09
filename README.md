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

| Dimension | Options |
|---|---|
| Scope of "daily" | Calendar day (midnight) · Rolling 24-hour window |
| Clock authority | ATM local time · Bank server time · Customer home-branch timezone |
| Limit scope | Per card · Per account · Per cardholder |
| ATM limit | This machine only · Shared across all ATMs |
| Joint/multi-card accounts | Shared pool · Independent limits per card |
| Failed transaction | Counts toward daily total · Does not count |
| Reversed transaction | Subtracted from daily total · Not subtracted |
| Exact change impossible | Decline · Offer nearest lower dispensable amount |

---

## Solution

Tests split into two categories: **behavior tests** confirm what should happen given the rule; **oracle tests** expose what the requirement leaves undefined — each one is a question the requirement doesn’t answer.

---

### Behavior tests

These have a knowable expected outcome.

**The 300€ boundary**
- Withdraw exactly €300 — succeeds
- Withdraw €301 — declined
- Withdraw €299 — succeeds
- Withdraw €300 in one transaction vs. €150 + €150 — same result

**Cumulative tracking across transactions**
- €100 + €100 + €100 — all three succeed (total €300)
- €100 + €100 + €100 + €10 — fourth declined (would exceed €300)
- €200 + €200 — second declined outright, not partially dispensed

**Declined attempts do not consume the limit**
- Withdraw €250, then attempt €300 — declined; can still withdraw €50
- Attempt €400 from the start — declined; full €300 still available

**Edge inputs**
- Withdraw €0 — error, not processed
- Withdraw a negative amount — rejected
- Withdraw €300.50 (non-integer) — rejected; smallest unit is €10
- Withdraw €300 when account balance is €0 — declined (balance check)
- Withdraw a very large amount (e.g., €999999) — declined

**Denomination and change interactions**
- ATM has only €50 bills — €300 dispensed as six €50s
- ATM has only €20 bills — €300 cannot be made exactly; declined (€290 or €300?)
- ATM has 25 × €10 bills (€250 total) — request for €300 declined; not partial

**Feedback after transactions**
- After a successful €300 withdrawal, balance display reflects the deduction
- Declined transaction: error message states how much remains today, not just “declined”
- Declined transaction: message says “limit reached”, not “insufficient funds”

---

### Oracle tests

These reveal ambiguity. The requirement is silent; the test result defines the answer.

**What does “daily” mean?**
- Withdraw €300 at 23:59, then €10 at 00:00 midnight — does the second succeed?
- Is “day” a calendar day (midnight reset) or a rolling 24-hour window from first transaction?
- Whose clock determines the day: the ATM’s local time, the bank server’s time, or the customer’s home branch time zone?
- What happens to the counter during a daylight saving time transition (a 23-hour or 25-hour day)?

**Which ATM does the limit apply to?**
- Withdraw €200 at ATM A, then €200 at ATM B same day — total €400 allowed or blocked?
- Withdraw €300 at ATM A — can the same card withdraw €300 at ATM B the same day?
- ATM is offline — how does it know the day’s running total?
- Concurrent withdrawal at two ATMs at the same instant — both approved for €300?
- System restart mid-day — does the counter survive?

**Whose limit is it?**
- Is the €300 limit per card, per account, or per customer?
- Joint account: do both cardholders share one €300, or does each have their own?
- Two cards linked to the same account — can each card withdraw €300?
- Business account vs. personal account — same limit or different?
- ATM has its own €300 daily limit and the account has a separate €500 daily limit — which applies? Both?
- Account limit (€200) is lower than ATM limit (€300) — is the effective ceiling €200?

**What counts toward the total?**
- A transaction is approved but cash is not dispensed (machine jams) — does it count?
- Network disconnects after approval but before dispensing — does it count?
- A reversed or refunded transaction — is it subtracted from the day’s total?
- A declined transaction — does it touch the counter at all?

**Currency and denomination edge cases**
- ATM dispenses only €20 bills — is €290 accepted when €300 cannot be made exactly?
- Withdrawal at a foreign ATM in a different currency — is the €300 limit applied before or after conversion? At which exchange rate?

**Limit changes and overrides**
- Bank changes the limit from €300 to €500 mid-day — does the new limit apply immediately?
- Fraud hold placed on the account mid-day — does it interact with the daily counter?
- Is the limit configurable per account, or fixed for everyone?

**Security and abuse**
- Rapid repeated attempts just below the limit — are they tracked or flagged?
- Two ATMs used simultaneously by the same card (race condition) — can the limit be exceeded?
- Client-side time manipulation — does moving the clock forward trigger a reset?

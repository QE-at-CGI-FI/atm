# atm
A test target for a requirements exercise for teaching testing. 

## Assignment

An ATM allows a daily withdraw limit of $300

You can’t ask the stakeholder any questions about the requirement, what tests would you run to understand the requirement better and test this functionality?

## Solution

Here's what I'd test — organized to expose the ambiguities hiding in that one sentence:

---

**What counts as "a day"?**
- Withdraw $300 at 11:59 PM, then $10 at 12:00 AM midnight — does the second succeed?
- Withdraw $300 on Monday, close browser, reopen Tuesday — is the counter reset?
- Is "day" calendar day (midnight) or rolling 24-hour window?

**Is $300 the ceiling or an example?**
- Withdraw exactly $300 — succeeds
- Withdraw $301 — declined
- Withdraw $299 — succeeds
- Withdraw $300 in one go vs. $150 + $150 — same result?

**Cumulative across transactions:**
- Withdraw $100, $100, $100 — all three succeed (total $300)
- Withdraw $100, $100, $100, $10 — fourth declined (would exceed $300)
- Withdraw $200, then attempt $200 — second declined, not partial

**This ATM or all ATMs?**
- Withdraw $200 from this ATM, then $200 from a different ATM — is total $400 allowed or blocked at $101?
- Withdraw $300 from ATM A — can you withdraw $300 from ATM B same day?

**Whose limit is it — ATM or account?**
- Account balance is $200, daily limit is $300 — is the effective ceiling $200?
- Account has its own $500 daily limit, ATM limit is $300 — which one applies? Both?
- Can the account limit be lower than the ATM limit ($200 account limit, $300 ATM limit)?

**Does a declined attempt consume the limit?**
- Withdraw $250, then attempt $300 (would exceed) — after the decline, can you still withdraw $50?
- Attempt $400 (over limit from the start) — does this touch the $300 counter at all?

**Exact change and denomination interactions:**
- ATM only has $50 bills — can you withdraw exactly $300?
- ATM only has $10 bills — you request $300, ATM has only 25 bills ($250) — declined or partial?

**Edge inputs:**
- Withdraw $0 — allowed or error?
- Withdraw a negative amount
- Withdraw $300.50 (non-integer)
- Withdraw $300 when account balance is $0

**After limit is reached:**
- Try to withdraw $1 after $300 spent — declined
- The error message: does it say how much is left today, or just "declined"?
- Withdraw $300, then check balance — does balance display reflect the withdrawal?

**The currency:**
- Requirement says `$`, app shows `€` — is that a bug or intentional?

---

These 30+ tests fall into roughly two buckets: **behavior tests** (verifying the rule works) and **oracle tests** (revealing what the requirement actually means where it's silent). The second bucket is the more interesting one for a requirements exercise — each test there represents a question the requirement doesn't answer.

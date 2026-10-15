# Nia Fineract 1.9.0-nia.1: twice-weekly repayments for bi-weekly products

**Status:** implemented and tested on a Fineract 1.9.0 test instance (the same change was first built and tested on 1.10.1). Not deployed to production yet.
**Base:** Apache Fineract 1.9.0, plus a backport of the Apache fix FINERACT-1971 (first repayment date with a minimum number of days), which 1.9.0 lacks. This is an internal Nia Financial Services change, not an Apache Software Foundation release.
**Date:** 13 October 2026

---

Dear Nia team,

Nia IT plans to release Nia Fineract 1.9.0-nia.1. It makes Nia's bi-weekly loan products repay **twice a week, on fixed pairs of days, with Sunday off**. Today these products repay once a week, so a "4 Weeks (Bi-Weekly)" loan actually runs for 8 weeks.

## What's new

- **Twice-weekly repayment schedule** for the 14 bi-weekly products listed below. The Hilton product and all weekly products are not affected.
- **Correct first repayment date.** The first repayment follows the same rule, whether the loan has just been applied for or has been disbursed.
- **On/off setting.** The new rule has its own global setting, which is off when the update is installed. Turning it off restores today's behaviour for new schedules immediately.

## How it works

Each repayment falls **3 working days** after the previous one. Monday to Saturday are working days; Sunday is not. This gives each weekday a fixed partner, and the loan alternates between the two:

| Disbursed on | Repayments |
|---|---|
| Monday | Thu, Mon, Thu, Mon, Thu, Mon, Thu, Mon |
| Tuesday | Fri, Tue, Fri, Tue, Fri, Tue, Fri, Tue |
| Wednesday | Sat, Wed, Sat, Wed, Sat, Wed, Sat, Wed |
| Thursday | Mon, Thu, Mon, Thu, Mon, Thu, Mon, Thu |
| Friday | Tue, Fri, Tue, Fri, Tue, Fri, Tue, Fri |
| Saturday | Wed, Sat, Wed, Sat, Wed, Sat, Wed, Sat |

- **First repayment:** 3 working days after disbursement. For a product with a minimum gap (for example 7 days), the first repayment is the later of the two, and never on a Sunday. The pair then follows from that day.
- **Sundays:** loans can't be disbursed on a Sunday, and no repayment is ever scheduled on one.
- **Stable dates:** the same disbursement date and holidays always give the same dates. Re-running the schedule (disbursement, reschedule, preview) does not move the pattern.
- **Number of repayments:** unchanged. 8 repayments × 3 working days = 4 weeks, so "4 Weeks (Bi-Weekly)" now really runs 4 weeks.

## Holidays

Holidays work as they do today in Fineract. Nia will use the **"reschedule to next repayment date"** type:

- **Repayment on a holiday:** it moves to the next date in the pair, and the following repayments move with it. For example, a Thursday holiday on a Mon/Thu loan moves that payment to Monday, then Thursday, Monday as before. The pair is kept, nobody pays twice on one day, and the loan ends one slot (3–4 days) later.
- **Disbursement on a holiday:** it is rejected ("The expected disbursement date cannot be on a holiday").
- **Holidays added later:** they are applied to existing loans by the daily holiday job. Enter holidays **before** the date. A holiday entered afterwards can re-date installments that have already been paid.
- **Offices:** a holiday only applies to the offices ticked on it.

**Tested on the test server (stock 1.10.1, weekly products)** with Independence Day, Saturday 18 April 2026, as a one-day holiday for all 10 offices:

| Loan | Result |
|---|---|
| WC4W disbursed Sat 11 Apr (first repayment on the holiday) | Repayments 25 Apr, 2 May, 9 May, 16 May: moved one week, then weekly |
| WC4W disbursed Sat 4 Apr (2nd repayment on the holiday) | 11 Apr, 25 Apr, 2 May, 9 May: one 14-day gap |
| WC4W disbursed Mon 13 Apr | 20 Apr, 27 Apr, 4 May, 11 May: unaffected |
| WC4W disbursed Sat 18 Apr | Rejected: disbursement on a holiday |
| Interest | Unchanged by the holiday (WC4W 20.00; WC4B 36.92, same as on non-holiday dates) |

## Products affected

Each product changes from "every 1 week" to "every 3 days" (counted as working days). Its number of repayments stays the same.

| ID | Short name | Repayments | Name |
|---|---|---|---|
| 2 | HP4B | 8 | 15% HP 4 Weeks (bi-weekly) |
| 4 | 25wb | 10 | 25% WC 5 Weeks (Bi-weekly) |
| 5 | W6WB | 12 | 20% WC 6 Weeks (Bi-weekly) |
| 9 | H6BW | 12 | 20 % HP 6 Weeks (Bi Weekly) |
| 10 | WC4B | 8 | 20% WC 4 Weeks (Bi-Weekly) |
| 11 | 2W5B | 10 | 20% WC 5 Weeks (Bi-Weekly) |
| 12 | G4WB | 8 | 20% GWC 4 Weeks (Bi-Weekly) |
| 13 | G5WB | 10 | 20% GWC 5 Weeks (Bi-Weekly) |
| 16 | SFBW | 20 | 20% SF 10 Weeks (Bi- Weekly) |
| 18 | 2W6B | 12 | 25% WC 6 Weeks (Bi-Weekly) |
| 19 | 5G5B | 10 | 25% GWC 5 Weeks (Bi-Weekly) |
| 24 | 2HB4 | 8 | 20% HP 4 Weeks (Bi-Weekly) |
| 27 | 2H8B | 16 | 20% HP 8 Weeks (Bi-Weekly) |
| 28 | 5H8B | 16 | 15% HP 8 Weeks (Bi-Weekly) |

**Not affected:** HBW4 "20% 4 Weeks (Bi_Weekly) Hilton" (stays every 2 weeks) and the 15 weekly products.

**Existing loans** keep their current weekly schedules and run to the end as booked. Only loans created after a product is converted use the new rhythm.

## Installation (for DevOps)

1. Back up the database and keep the current `fineract-provider.war`.
2. Deploy the new WAR. The new setting is off, so behaviour is unchanged.
3. Turn the setting on.
4. Convert one product (`repaymentFrequencyType` = days, `repaymentEvery` = 3). Book a test loan and compare it with the table above.
5. Convert the remaining 13 products.

Working days need no change: production already runs Monday to Saturday with "move to next working day".

## Rollback

- **First step:** turn the setting off. New schedules return to today's behaviour at once.
- **Full rollback:** restore the previous WAR. Nothing in the database has to be undone.
- **Loans booked meanwhile:** they keep their stored schedules. Correct them before disbursement, or with a loan reschedule.

## Open points before release

1. **G4WB, G5WB and 5G5B** require 7 days before the first repayment, which makes a 4-week loan about 5 weeks. Keep or reduce to 3?
2. **WC4B and 2H8B** charge interest "per month". Their totals must be checked with the schedule preview after conversion.
3. **Production holidays:** the four active holidays are "specific date" with no target date. They should be recreated as "next repayment date". No holidays are defined after 18 April 2026; enter the rest of 2026 ahead of time.
4. **Sunday transactions** are blocked today. Confirm that no repayments (e.g. mobile money) need to be posted on Sundays.

## Technical details

- `WorkingDaysUtil`: new working-day stepping helpers.
- `DefaultScheduledDateGenerator`: next due date and ideal disbursement date use working days for day-based products when the setting is on.
- `LoanScheduleAssembler.deriveFirstRepaymentDate`: first repayment uses the same rule.
- One new global-configuration row for the on/off setting (no new tables).
- Full design and code-path analysis: https://claude.ai/artifact/EeXM1LE3ziF95qaC3XUQqM

Regards,
Nia IT

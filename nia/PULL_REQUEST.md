# Twice-weekly repayments for bi-weekly products (Fineract 1.9.0)

**Branch:** `nia/twice-weekly-repayments` → base: Apache Fineract `1.9.0`
**Commits:** 3 (FINERACT-1971 backport, the Nia change, a build fix)
**Status:** ready for review. Unit tests and end-to-end tests pass on a local 1.9.0 test instance; not deployed to production.
**Same change for 1.10.1:** https://github.com/Tashinga4/nia-fineract-1.10.1/pull/1

## Description

Nia's 14 bi-weekly loan products must repay **twice a week on fixed day pairs, with Sunday off**:

| First repayment on | Repayments alternate |
|---|---|
| Monday / Thursday | Mon ↔ Thu |
| Tuesday / Friday | Tue ↔ Fri |
| Wednesday / Saturday | Wed ↔ Sat |

Stock Fineract can't produce this. With Sunday as a non-working day, Fineract only moves a due date that lands on Sunday, and the next date is still counted from the unmoved date. A product repaying every 3 days therefore cycles through the whole week (Thu, Mon, Wed, Sat, Tue, Fri…) instead of alternating between two days.

## The commits

### 1. FINERACT-1971: Fix wrong due date calculation when loan got submitted (Apache backport)

Upstream Apache commit `039bca440b` (April 2024, in Fineract 1.10.0 and later), application code only (`LoanApplicationTerms`, `LoanScheduleAssembler`). Its integration tests are left out because they depend on later test helpers.

**Why it's needed:** in stock 1.9.0 a product with a **minimum number of days before the first repayment** (G4WB, G5WB, 5G5B, WC4W, G4WW, 5G5W, 15H8 and others) shows the right first date when the loan is applied for, but **loses the minimum when the schedule is regenerated at disbursement**. Example on the 1.9.0 test instance, G4WB disbursed Mon 5 Oct: application said Mon 12 Oct, after disbursement the first repayment was Thu 8 Oct. With the backport both say Mon 12 Oct. This affects weekly products too, independent of the twice-weekly change.

### 2. Count daily repayment steps in working days when enabled (the Nia change)

The same commit as on 1.10.1 (10 files, +467 / −15). With the backport underneath, `LoanScheduleAssembler` applies exactly as on 1.10.1.

New global configuration **`repayment-steps-in-working-days`**, disabled by default. When it is enabled, loans that repay **every N days** (and have no meeting calendar) count those N days in **working days** taken from Admin → Working Days. Every 3 working days with Monday to Saturday working gives the pairs above.

- `WorkingDaysUtil`: new `plusWorkingDays`, `minusWorkingDays`, `nextWorkingDayOnOrAfter`.
- `DefaultScheduledDateGenerator`: next due date and ideal disbursement date step in working days.
- `LoanScheduleAssembler.deriveFirstRepaymentDate`: the first repayment uses the same rule, so the application schedule equals the one kept after disbursement.
- `HolidayDetailDTO`: carries the flag (set in `LoanScheduleAssembler` and `LoanUtilService`).
- `ConfigurationDomainService(Jpa)`: `isRepaymentStepsInWorkingDaysEnabled()`.
- Liquibase `0141_add_configuration_repayment_steps_in_working_days.xml`: adds the setting row, disabled. The file name is the same as in the 1.10.1 build, so a later upgrade to Nia's 1.10.1 build sees it as already applied.

Differences from the 1.10.1 commit: only the unit-test helper (1.9.0's `LoanApplicationTerms.assembleFrom` takes its arguments in a different order) and the position of the Liquibase include.

**Not affected:** weekly, monthly and yearly products, loans linked to a meeting calendar, and everything else when the setting is off.

### 3. Disable the asciidoctor documentation build

1.9.0 no longer builds as released: the documentation plugin needs `com.burgstaller:okhttp-digest:1.10`, which was only on JCenter and can't be downloaded any more. Apache did the same in 1.10.1: the asciidoctor plugins are commented out and `fineract-doc` isn't included. The application is not affected.

## Tests

**Unit tests** (JDK 17, Gradle 8.6):
- New `DefaultScheduledDateGeneratorWorkingDayStepsTest`: 20 tests pass.
- All unit tests in `fineract-core`, `fineract-loan` and `fineract-provider`: 688 tests, 0 failures.

**End-to-end on a 1.9.0 test instance** (Docker image built from this branch, empty database, the 30 Nia products imported with Nia's import script, working days and calendar settings as in production). Scripts and logs in [`e2e/`](e2e/).

`e2e-190-run2.log`: **20/20 checks pass**:
- 2W5B disbursed on every day Monday to Saturday: 10 repayments each, correct pair, application schedule equal to the disbursed one.
- Down-payment product 2HB4: regular repayments on the pair.
- G4WB (7-day minimum): first repayment a week after disbursement, then the pair.
- Disbursed a day later than expected: schedule regenerated on the new pair.
- A repayment doesn't move any due date.
- Setting off gives stock behaviour; setting on again gives the pairs.
- Holiday Thu 22 Oct ("reschedule to next repayment date"): the payment moves to the next date in the pair; the holiday job applies the same to existing loans with no doubled dates and no Sundays.

`e2e-190-run1.log`: 18/26. Every schedule in it follows the pair and the holiday rules. The 8 failures all come from the down-payment product 2HB4, where that script expects 8 regular repayments. Fineract creates 6, and **stock 1.9.0 does the same with the setting off** (checked on the test instance), so this is the known down-payment issue below, not this change.

## How to apply

**From this repository:** merge the pull request from `nia/twice-weekly-repayments` into `main`.

**Onto the server's own 1.9.0 source**, take all three commits, in order:

```bash
git fetch https://github.com/Tashinga4/nia-fineract-1.9.0.git nia/twice-weekly-repayments
git cherry-pick FETCH_HEAD~2 FETCH_HEAD~1 FETCH_HEAD
```

If the server's 1.9.0 has its own changes in `LoanScheduleAssembler`, `LoanApplicationTerms` or `DefaultScheduledDateGenerator`, expect conflicts there.

Build with **JDK 17**: `./gradlew :fineract-provider:bootJar` (or `jibDockerBuild` for a Docker image).

## Deployment

1. Back up the database and keep the current build/image.
2. Deploy the new build. The setting is off, so twice-weekly behaviour is unchanged. Note that the FINERACT-1971 backport is active immediately: products with a minimum number of days now keep it after disbursement.
3. Enable **Admin → System → Configurations → `repayment-steps-in-working-days`**.
4. Change one bi-weekly product to repay every **3 Days** and book a test loan. Then convert the rest:
   HP4B, 25wb, W6WB, H6BW, WC4B, 2W5B, G4WB, G5WB, SFBW, 2W6B, 5G5B, 2HB4, 2H8B, 5H8B. **Do not convert Hilton (HBW4).**
5. Loan term = 3 × number of repayments, in days (for example 24 days for 8 repayments).

**Rollback:** disable the setting (new schedules revert immediately), or redeploy the previous build.

## Reviewer notes

- Existing active loans keep their stored schedules. Only new loans on converted products use the new rhythm.
- Holidays: use **"Reschedule to next repayment date"**, then run **Scheduler Jobs → "Apply Holidays To Loans"** straight away. A loan booked between activating a holiday and the job keeps a payment on the holiday (stock behaviour, weekly products too).
- Stock Fineract, not fixed here: products with a **down payment** generate fewer regular repayments than configured (2HB4: 6 of 8), which lowers "whole term" interest. Worth checking in production.
- "Per month" interest products (WC4B, 2H8B) charge less once the term is shorter. That's a business decision before conversion.

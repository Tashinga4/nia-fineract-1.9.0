# Nia Fineract 1.9.0: twice-weekly repayments

Internal Nia Financial Services work on Apache Fineract 1.9.0, the version the live server runs, behind the Mifos web-app.

**Status:** implemented and tested on a Fineract 1.9.0 test instance (unit tests and end-to-end tests pass). **Not deployed to production.**

This is the 1.9.0 version of the change in [nia-fineract-1.10.1](https://github.com/Tashinga4/nia-fineract-1.10.1).

## The problem

Nia's bi-weekly products (for example **20% WC 4 Weeks (Bi-Weekly)**) must repay **twice a week on fixed pairs of days, with Sunday off**: Mon/Thu, Tue/Fri or Wed/Sat.

Stock Fineract can't produce this:

- Set up as **weekly**, the products repay once a week, so a "4 Weeks (Bi-Weekly)" loan runs for 8 weeks.
- Set up as **every 3 days**, Fineract counts calendar days. The repayments wander through the whole week (Thu, Mon, Wed, Sat, Tue, Fri…).

## The fix

A new global setting, **`repayment-steps-in-working-days`** (off by default). When it is on, loans that repay **every N days** count those N days in **working days**. With Monday to Saturday as working days, "every 3 days" gives exactly the pairs above, and a "4 Weeks" loan with 8 repayments really runs 4 weeks.

On 1.9.0 this also needs an Apache fix that only arrived in 1.10.0 (**FINERACT-1971**). Without it, products with a minimum number of days before the first repayment (for example G4WB, 7 days) lose that minimum when the loan is disbursed. The fix is backported here.

The pull request from `nia/twice-weekly-repayments` into `main` has three commits. [PULL_REQUEST.md](PULL_REQUEST.md) describes each one.

## Tests

- **Unit tests:** the 20 new twice-weekly tests and all 688 unit tests in `fineract-core`, `fineract-loan` and `fineract-provider` pass.
- **End-to-end** on a 1.9.0 test instance with the 30 Nia products and real loans: **20/20 checks pass** (`e2e/e2e-190-run2.log`). See [PULL_REQUEST.md](PULL_REQUEST.md) for what was checked and for the notes on `e2e-190-run1.log`.
- `e2e/setup_test_instance.py` prepares an empty test instance (working days, placeholder accounts, product import, test client). It needs Nia's product import script next to it and refuses anything other than a localhost URL. The scripts use Fineract's default test login. Don't point them at production.

## Repository layout

| Branch | Contents |
|---|---|
| `main` | Apache Fineract 1.9.0 source (tag `1.9.0`, without upstream history) and this `nia/` folder |
| `nia/twice-weekly-repayments` | `main` + the three commits (pull request into `main`) |

## Deploying

See the "Deployment" section of [PULL_REQUEST.md](PULL_REQUEST.md) and [release-notes.md](release-notes.md). In short: back up the database, deploy the build (the setting is off), enable the setting, convert one bi-weekly product to "every 3 days" and test it, then convert the rest. **Do not convert Hilton (HBW4).**

**Rollback:** turn the setting off, or redeploy the previous build.

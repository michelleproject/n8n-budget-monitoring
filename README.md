# Budget Monitoring Workflow

An n8n workflow that reads transactions from Google Sheets, aggregates spending by
category, compares each category against its limit, and emails a summary that
flags every overage.

Scheduled to run every Monday at 8 am. The alert is pushed to your inbox rather than pulled from a dashboard.



## What it does

1. Reads every row from the **Transactions** sheet
2. Groups by category, summing amounts and counting transactions
3. Writes the totals to a **Summary** sheet, updating existing category rows
rather than duplicating them
4. Compares each category total against its budget limit
5. Sends an email listing every category, with `[OVER]` and the exact overage on
any that breached

## Example output

```
Total spending: 4.170.000

[OVER] Health: 702.000 of 500.000 - over by 202.000 (2 transactions)
[OVER] Shopping: 695.000 of 500.000 - over by 195.000 (3 transactions)
[OVER] Subscriptions: 481.000 of 400.000 - over by 81.000 (4 transactions)
Groceries: 736.000 of 800.000 (3 transactions)
Transport: 598.000 of 600.000 (5 transactions)
Utilities: 568.000 of 600.000 (2 transactions)
Food & Drink: 390.000 of 500.000 (5 transactions)

Over budget: Health, Shopping, Subscriptions
```

The subject line changes based on the result, so an alert is distinguishable from
a routine summary without opening the email.

A category with no matching budget limit is reported as `(NO LIMIT SET)` rather
than passing silently. Without that, a typo or a capitalisation mismatch between
the sheet and the limits would leave a category permanently unmonitored while
still appearing in the email with a correct total.

## Sheet structure

**Transactions** tab:

```
Date | Description | Amount | Category
```

Dates in ISO format (`2026-07-01`). Amounts as plain numbers with no currency
symbol or thousands separator, so `65000` rather than `Rp 65.000`.

**Summary** tab:

```
Category | Total | Transaction Count
```

Written by the workflow. Uses append-or-update matched on Category, so rerunning
updates the existing rows instead of appending duplicates.

## Budget limits

Set at the top of the **Check budget limits** node:

```javascript
const budgets = {
  "Groceries": 800000,
  "Health": 500000,
  "Shopping": 500000,
  "Transport": 600000,
  "Utilities": 600000,
  "Subscriptions": 400000,
  "Food & Drink": 500000
};
```

Names must match the Category column exactly. A mismatch produces a
`(NO LIMIT SET)` line in the email, which is the intended signal to fix it.

## Setup

**Requires:** n8n, Google Sheets OAuth credential, Gmail OAuth credential.

1. Import `budgeting_workflow.json` into n8n
2. Reconnect the Google Sheets and Gmail credentials
3. Create a spreadsheet with the two tabs described above
4. Point both Google Sheets nodes at your spreadsheet
5. Set `sendTo` in the Gmail node to your own address. It ships as
`your-email@example.com` to keep a real address out of this repo
6. Edit the budget limits in the Check budget limits node
7. Adjust the Schedule Trigger if Monday 8am does not suit

## Known limitations

* **No date filter.** Aggregation covers every row in the Transactions sheet.
Tested against a single month (July 2026), where this is correct. With two
months of data the totals accumulate across both while the limits still read as
monthly, so categories would show as over budget when they are not. A period
filter is needed before this handles more than one month.
* **Weekly schedule, monthly-shaped limits.** The trigger runs weekly but the
limits are sized like monthly budgets. Reconciling the two is part of the same
fix as the date filter.
* **Amounts must be plain numbers.** `Number()` on a string like `Rp 65.000`
yields `NaN`, which falls back to `0` and drops that transaction from the total
with no error.
* **Limits live in code.** Moving them to their own Sheets tab would let a
non-technical user adjust budgets without editing JavaScript.

## Notes

* Both a Manual Trigger and a Schedule Trigger feed the same chain, so the
workflow can be tested on demand and still run unattended.
* Aggregation and limit-checking are separate nodes. Changing how budgets are
evaluated does not touch the aggregation logic.
* Figures are formatted with `toLocaleString('id-ID')` for Indonesian thousands
separators.
* The spreadsheet referenced in this export contains dummy data.

